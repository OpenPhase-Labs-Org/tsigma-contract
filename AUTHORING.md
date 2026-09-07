# Writing a plugin

What a plugin author builds, in order. `PROTOCOL.md` specifies the envelope and
`TYPES.md` the payloads; this assembles both into the sequence you implement.

A plugin is **one executable** that serves gRPC. Any language with a gRPC and an
Arrow library will do. You are not required to use a helper library from anyone -
generate stubs from the `.proto` files in this repository and implement the
services below.

## 1. Startup sequence

Your executable does these five things, in this order, before it is useful:

1. **Check the magic cookie.** Read env `TSIGMA_PLUGIN_MAGIC`. If it is not
   `tsigma-plugin-v1`, write a line to stderr and exit non-zero. This stops a
   plugin being run as an ordinary program and an ordinary program being loaded
   as a plugin. It is a guard, not a secret.
2. **Bind a listener** on `127.0.0.1:0` (or a unix socket) and note the address
   the OS gave you.
3. **Start the gRPC server** with every service in section 2 registered.
4. **Print the handshake line to stdout, then flush**:
   `1|1|tcp|127.0.0.1:PORT|grpc`
   Fields are `CORE|APP|NETWORK|ADDRESS|PROTOCOL`. `CORE` is `1`. `APP` is the
   plugin wire-protocol version, currently `1`. `NETWORK` is `tcp` or `unix`.
   `PROTOCOL` is `grpc`. One line, no leading whitespace, flushed - a host that
   does not see it before its timeout kills you.
5. **Serve until shut down.** Do not exit after the handshake unless you are a
   cron-model plugin, which runs once and exits.

Anything else you write to stdout or stderr is drained by the host and logged;
it will not wedge you, but the FIRST line of stdout must be the handshake.

## 2. Services you must serve

Three, whatever your subsystem:

| Service | Where it is defined | Why |
|---|---|---|
| your subsystem service | `proto/tsigma/<subsystem>/v1/` | the work |
| `grpc.health.v1.Health` | standard gRPC health proto | the host polls it; unhealthy or unreachable means unavailable |
| `plugin.GRPCController` | `proto/vendor/go-plugin/grpc_controller.proto` | graceful shutdown; without it the host kills your process |

`grpc.health.v1` is not vendored here - take it from your gRPC distribution.

## 3. Your subsystem

Implement the RPCs in your subsystem's `.proto`. Every subsystem has a
`Describe` that the host calls once at registration; what it returns is how the
host knows what you can do.

**Decoder** - `proto/tsigma/decoder/v1/decoder.proto`
- `Describe` - your name, the file extensions you claim, your `output_kind`, and
  your `time_semantics` (see section 4).
- `CanDecode` - given a bounded head of a file, say whether it is yours.
- `Decode` - bidirectional stream. The host sends byte chunks; you send Arrow
  record batches, optionally one `FileMetadata`, and exactly one terminal
  `DecodeStatus`. A stream that ends without a status is a FAILURE.

**Method** (poller / listener) - `proto/tsigma/method/v1/method.proto`
- `Describe`, `HealthCheck`, then `Poll` for a scheduled fetch or `Listen` for a
  long-lived receiver, per your `execution_mode`.
- You do not decode or persist. You fetch bytes and hand them to the host via
  `EventSinkService.DecodeAndPersist`; the host decodes, validates and stores.

**Report** - `proto/tsigma/report/v1/report.proto`
- `Describe` and `Generate`. `Generate` streams exactly one `ViewModel` first,
  then zero or more Arrow batches of rows.

**Notify** - `proto/tsigma/notify/v1/notify.proto`
- `Describe` and `Send`. `Send` is advisory: a non-OK result is logged by the
  host and never blocks ingestion.

## 4. Payload rules that are easy to get wrong

- **Arrow field names and types are fixed.** See `TYPES.md`. A decoder emits
  `event_time` / `event_code` / `event_param`, not names of your choosing.
- **Every event batch carries `tsigma.time_semantics`** in its Arrow *schema
  metadata*, valued `utc` or `naive_local`. It is not a column and not a
  per-event flag. A batch without it is a contract violation and the host flags
  it - it will never guess, because guessing moves every timestamp in the batch
  by a whole timezone offset.
- **`naive_local` means you do not convert.** Emit the controller's wall-clock
  reading as-is. The host knows the signal's zone; you do not.
- **Stream, never accumulate.** Send a sequence of bounded Arrow batches. Do not
  build one large message - see the message-size invariant in `PROTOCOL.md`.
- **Send your terminal status.** `SUCCESS`, `PARTIAL` or `FAILURE`. If you
  decoded some rows and dropped others, that is `PARTIAL`, and you send the rows
  you did decode along with it. The host persists them and flags the remainder
  for review; it does not discard data you handed it.

## 4a. If a host dials you over a network

A plugin launched as a child process listens on loopback and needs no transport
security - nothing leaves the machine. A plugin an orchestrator runs (systemd,
k8s) is dialled across a network, and then two things are true: the ingestion
data crosses a wire, and the broker hands you a callback INTO the host. Serve
TLS on that listener.

- **A self-signed certificate is fine**, and is the normal case. An operator
  points their host at your certificate, or at the CA that issued it. That
  trusts exactly one issuer, which is stricter than a public bundle.
- **Issue it for the name the host will dial** - the service name, usually. If
  the host dials a pod IP instead, the operator overrides the name their end
  checks against; the certificate is still verified.
- **Mutual TLS is worth supporting.** Ask for a client certificate and you can
  tell whether the thing calling you is the host. On a connection that is about
  to hand you a data-plane callback, that matters in both directions.

A host may refuse to dial a non-loopback address in the clear. Do not rely on
plaintext working off the local machine.

## 5. What you do not get

- **No database credentials, no DSN, no schema.** Ever. If you need host data,
  ask for it - see section 6.
- **No filesystem contract.** Paths the host gives you are the host's business;
  do not assume a shared filesystem.
- **No guarantee you are the only instance.** Write nothing to a fixed path.

## 6. Getting data back from the host

Only if your subsystem needs it. A pure transform - most decoders - needs
nothing here.

The single connection carries a `GRPCBroker` that brokers connections in both
directions, so you can dial back to host-served services. Which services exist is
per subsystem and defined in that subsystem's `.proto`; a method uses
`CheckpointService` and `EventSinkService`.

A report does **not** use broker query services. It reads through the host's own
HTTP API, using a short-lived credential the host supplies for that one
invocation. The host answers under the requesting user's own permissions, so a
report sees exactly what that user sees.

## 7. Shutting down

Implement `GRPCController.Shutdown`: stop accepting work, finish what is in
flight, close your listener, exit zero. If you do not exit promptly the host
kills the process, and anything you had buffered is lost.

## 8. Declaring yourself to a host

A host needs to know your name, how to run you, and which subsystems you serve.
That declaration is host-specific - it is not part of this contract - but every
host needs the same four facts:

- a name, unique among that host's plugins
- how you are run: launched as a child process, started on a schedule, or
  already running under an orchestrator
- the command to launch you, or the address to dial if you are orchestrated
- which subsystems you serve

Check your host's documentation for the format it expects.

## Minimal decoder, in pseudocode

```
if env["TSIGMA_PLUGIN_MAGIC"] != "tsigma-plugin-v1":
    stderr("magic cookie missing or wrong"); exit(1)

port   = bind("127.0.0.1:0")
server = grpc_server()
server.add(DecoderService(impl))
server.add(HealthService())          # grpc.health.v1
server.add(GRPCControllerService())  # graceful shutdown
server.start()

stdout("1|1|tcp|127.0.0.1:" + port + "|grpc"); flush()
server.wait()
```

`Decode`, in the same shape:

```
chunks = []
for request in stream:                  # host sends bounded byte chunks
    chunks.append(request.chunk)

events = my_parser(join(chunks))        # your format, your code

batch = arrow_batch(
    event_time  = [e.timestamp for e in events],   # timestamp[us], no tz
    event_code  = [e.code      for e in events],   # int32
    event_param = [e.param     for e in events],   # int32
)
batch.schema.metadata["tsigma.time_semantics"] = "naive_local"   # or "utc"

send(DecodeResponse(events_arrow_ipc = batch))
send(DecodeResponse(status = DecodeStatus(outcome = SUCCESS,
                                        events_emitted = len(events))))
```
