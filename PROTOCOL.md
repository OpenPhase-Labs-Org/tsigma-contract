# Plugin Wire Protocol (frozen)

The transport every TSIGMA plugin speaks is the HashiCorp `go-plugin` **gRPC wire
protocol**, pinned and frozen here as **TSIGMA Plugin Protocol version 1** - the
standard is "this document," not "whatever `go-plugin` currently ships."

Every host conforms to this spec, by whatever route its language affords: a host
built on the `go-plugin` library gets it natively, and any other host implements
the same handshake, health, lifecycle and broker behavior over its own gRPC
stack.

A plugin built once against this protocol therefore loads on any conforming host.

> **Insulation caveat.** Pinning the version freezes the *contract*, but every
> host still reaches it through some implementation. If `go-plugin` upstream
> changes its handshake or broker behavior, a host that uses the library natively
> pins or shims to keep conforming to this spec, exactly as one that reimplements
> the protocol would. The insulation is at the spec level, not the library level.
> The spec is the authority; the library is one implementation of it.

## 1. Launch and handshake

The host launches the plugin as a subprocess and sets a magic-cookie environment
variable; the plugin proves it is a real TSIGMA plugin by emitting a single
handshake line on **stdout**, then serving gRPC.

- **Magic cookie:** env `TSIGMA_PLUGIN_MAGIC` must equal the pinned cookie value
  `tsigma-plugin-v1`. It is a handshake guard, not a secret.
  A process started without it exits immediately (prevents a plugin being run as a
  normal program, and a normal program being loaded as a plugin).
- **Handshake line** (stdout, one line):
  `CORE-PROTOCOL-VERSION | APP-PROTOCOL-VERSION | NETWORK | ADDRESS | PROTOCOL`
  - `CORE-PROTOCOL-VERSION` = `1` (the `go-plugin` envelope version; pinned).
  - `APP-PROTOCOL-VERSION` = the **TSIGMA Plugin Protocol** version (this
    document; currently `1`) - the wire/envelope version, NOT the Contract SemVer
    and NOT any subsystem's contract version. It gates the **envelope** at
    handshake; per-subsystem **contract** versions are negotiated separately at
    registration (see section 5 and ADR-0004). A wire-protocol-major mismatch is
    rejected at handshake.
  - `NETWORK` = `tcp` or `unix`.
  - `ADDRESS` = host:port or socket path the plugin is listening on.
  - `PROTOCOL` = `grpc` (the only supported value; `net/rpc` is not used).

### Transport security

The handshake carries no TLS field: whether a connection is encrypted is a
property of the deployment, not of the wire protocol, and both ends learn it from
their own configuration.

A launched plugin listens on loopback, where the kernel is the boundary. A plugin
reached across a network SHOULD serve TLS, and a host SHOULD refuse to dial a
non-loopback address without credentials - the broker means a plugin connection
also carries a callback into the host, so an unauthenticated one is not merely an
unencrypted read. Self-signed certificates are expected; trusting one issuer
directly is stricter than trusting a public bundle. Mutual TLS lets each end
identify the other.

## 2. Health

The plugin serves the standard **`grpc.health.v1.Health`** service. The host polls
it; an `UNHEALTHY`/unreachable plugin is marked unavailable and supervised
(restart on next discovery/reload). Health is how crash-isolation is observed:
a plugin that dies takes only its own process down.

## 3. Lifecycle / graceful shutdown

The host stops a plugin via **`GRPCController.Shutdown`** (graceful), falling back
to killing the subprocess. The host owns the plugin lifecycle: discover -> launch
-> handshake -> register -> supervise -> shut down.

## 4. Bidirectional broker (the consume-side data-plane)

The single host<->plugin connection carries a **`GRPCBroker`** that can broker
**additional gRPC connections in both directions**. This is the mechanism for the
**consume-side** of every subsystem contract:

- **Implement-side:** the host calls the plugin (e.g. `Decoder.Decode`,
  `Method.Poll`).
- **Consume-side:** the plugin dials **back** to **host-served** broker services
  (e.g. `CheckpointService.LoadCheckpoint`, `EventSinkService.DecodeAndPersist`) over
  the same connection.

**The plugin never receives database credentials or schema.** It asks a host
broker service for what it needs; the host answers **from its own session, under
the request's tenant context**. This host-mediated data plane - not a sandbox - is
what contains an untrusted plugin: it can compute, but it can only reach data the
host hands it, tenant-scoped.

Broker service definitions live **per subsystem**, defined only where a subsystem
needs them. Pure transforms (e.g. most decoders: bytes -> events) use no broker
services at all; a method is the typical consumer.

A **report** does not use broker query services. It reads through the host's own
API under a short-lived credential the host supplies for one invocation, so the
host answers under the requesting user's own permissions rather than serving a
parallel data path that bypasses them (see ADR-0006, amended).

## 5. Version negotiation (per-subsystem)

Two independent version axes (decided: **per-subsystem versioning** - a
multi-vendor ecosystem must not gate every vendor's plugins on one global number):

- **Wire-protocol version.** The handshake's `APP-PROTOCOL-VERSION` carries the
  TSIGMA **plugin protocol** version (this document; currently `1`). The host
  refuses a plugin whose wire-protocol major it does not support. This gates the
  envelope only (handshake / health / controller / broker), not any subsystem's
  payload.
- **Per-subsystem contract version.** Each subsystem `.proto` is independently
  versioned (`tsigma.decoder.v1`, `tsigma.report.v1`, ...). A plugin declares **at
  registration** which subsystem contracts and versions it implements (via the
  plugin-set registration metadata / `Describe`); the host accepts each subsystem
  at a version it supports and rejects only the mismatched ones. A breaking
  `decoder/v2` therefore does **not** invalidate anyone's `report` plugins.

Within a subsystem major, message evolution is additive (proto rules: add fields,
never renumber or repurpose a tag), so a newer host loads an older plugin and vice
versa within that subsystem's major.

## Message size invariant

**No message approaches the gRPC maximum, and the limit is never raised.** Any RPC
that carries file bytes or returns rows **streams** instead: bounded chunks /
record batches, sized well under the cap.

- File-bearing inputs (e.g. an uploaded controller log) are **client-streamed** as
  bounded byte chunks.
- Row-returning / event-returning outputs are **server-streamed** as Apache Arrow
  record batches (see `TYPES.md`).
- Stateful payloads (a decoder needs the header before the body) are handled by the
  plugin stream-parsing its input; the host does not pre-chunk semantically.

This keeps both sides from buffering a whole file/result in memory and removes the
cap as a deployment-breaking ceiling. Each subsystem `.proto` that moves bulk data
uses streaming RPCs accordingly.

## 6. Trust posture (uniform boundary, non-uniform posture)

The boundary (this protocol) is identical for every subsystem. The **trust
posture** is not (each `.proto` preamble in `proto/` states its own):

- **Untrusted-vendor-OK:** decoder, method (poller/listener), report,
  notification. Contained by host-mediated data access (crash-isolated process +
  no DB credentials): a method reaches only tenant-scoped broker services, and a
  report reads only through the host API under the short-lived per-invocation
  credential described in section 4. An arbitrary vendor binary is therefore safe
  to load.
- **Privileged - first-party or DOT-sanctioned only:** auth and storage. These are
  trust *roots* mediation cannot contain (auth **asserts identity**; storage
  **holds the data**).
- **Validation - host core, never a wire plugin (settled).** Both validation
  systems are in-process first-party code, never loaded from the plugin dir, with
  **no gRPC contract in this artifact**: (1) the post-ingest layer pipeline (the
  `Validator` interface - schema/range plus future ML layers) and (2) the
  ingest-time integrity program (temporal integrity, MAC-identity, config-phase
  drift, clock-health). They are the host's poison-aware integrity guarantee:
  exposing them across the plugin boundary would let a third party omit or
  override the checks the host is accountable for. `validator` shares the
  registry *shape* of the wire subsystems but is **not** one of them - it gets no
  `.proto`.

**The privileged tier must be enforced by a host-side gate, not by "should."** A
binary physically cannot be allowed to register as `auth` or `storage` unless it
clears an explicit operator authorization keyed to the subsystem it registers for.
A trust root protected only by operator discipline is not protected.

> **[DESIGN-CHECK] Privileged-tier enforcement mechanism - open (under
> deliberation).** The *requirement* (a host-side gate) is fixed; the *mechanism*
> is not, and it is complicated by **who operates the deployment** - a
> controller-unit vendor, a third-party integrator, or a DOT/Agency itself - since
> each implies a different trust anchor for "who may authorize an `auth`/`storage`
> binary." Options in play: signed plugins (whose publisher key - OpenPhase
> first-party, a DOT-sanctioned signer, the integrator?), an operator allowlist
> (hash/identity keyed to permitted subsystem), or an explicit per-binary approval
> at registration - possibly differing by operator type. More urgent than the
> deferred vendor-licensing/sandboxing questions because it guards the identity and
> data trust roots.

## What is NOT redefined here

The `grpc.health.v1.Health`, `GRPCController`, `GRPCBroker`, and `GRPCStdio`
service definitions are `go-plugin`'s own protos, pinned at protocol version 1.
This document **freezes and references** them as part of the standard; it does not
fork or rewrite them. The TSIGMA-specific contracts are the per-subsystem service
`.proto` files under [`proto/`](proto/).
