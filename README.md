# TSIGMA Contract

The single, shared, language-neutral standard that **every** TSIGMA host and
**every** vendor plugin builds against.

A conforming host may be written in any language. Each is an **implementation of
this one contract**, not a contract of its own. The contract is owned by
**OpenPhase-Labs** and published under **MPL-2.0** as a versioned artifact,
governed here rather than inside any host's codebase: a change is made once, in
this repository, and consumed from it.

## Why it exists

A vendor builds one plugin against the `.proto` set in this repository. The
interface is governed here, versioned here, and published under MPL-2.0 - not
defined inside the software that happens to load it. A vendor targets a standard
that cannot be changed under them at an implementer's convenience.

That holds across builds. TSIGMA is MPL-2.0: anyone may run their own, extend it
privately, and offer it as a service. A plugin written against this contract
works on any of them, because conformance is to the contract and not to one
organisation's build. The ecosystem does not fragment when someone forks.

## What is in it

| Piece | File(s) | Purpose |
|-------|---------|---------|
| Plugin wire protocol (frozen) | [`PROTOCOL.md`](PROTOCOL.md) | handshake, health, lifecycle, bidirectional broker - pinned to TSIGMA's own version, not "whatever go-plugin ships" |
| Wire-type conventions | [`TYPES.md`](TYPES.md) | Apache Arrow IPC for bulk/tabular; protobuf for structured/small; no language-native serialization on the wire |
| Per-subsystem service contracts | [`proto/`](proto/) | one `.proto` package per **wire** plugin subsystem: decoder, method, report, auth, notify, storage (six). Validation is core to the host - see below. |
| Version pins | [`VERSION`](VERSION) | contract SemVer + the pinned integer plugin-protocol version |

**Validation is core, not a wire subsystem.** The poison-aware validation/integrity
spine (the post-ingest layer pipeline and the ingest-time integrity program) is
in-process first-party code in a conforming host and has **no contract here**: data
integrity is the host's own responsibility and is deliberately not delegated to a
plugin. It shares the registry shape of the wire subsystems, which is why prose
elsewhere may list "validator" among the subsystems, but it carries no `.proto` and
loads no binary. See PROTOCOL.md section 6.

## Single source of truth

Hosts and plugins **codegen from this artifact**; none keeps a private copy.

Generate service stubs with whatever protobuf/gRPC codegen your language provides
(`protoc-gen-go` / `protoc-gen-go-grpc`, `grpcio-tools`, and their equivalents
elsewhere). The frozen wire protocol is then satisfied either **natively**, by
building on the HashiCorp `go-plugin` library, or by a small shim that speaks the
same handshake, health, lifecycle and broker behavior over your own gRPC stack.

Interoperability is enforced **by construction** (one source, many codegens), not
by vigilance.

## Verifying a change

[`buf.yaml`](buf.yaml) configures the checks this contract is held to. From the
repository root:

```
buf build      # the .proto set parses and resolves
buf lint       # naming and layout conform to the protobuf standard
buf breaking --against '.git#branch=main'   # no wire- or JSON-breaking change
```

`buf breaking` is the one that matters most. Message evolution here is additive
by rule - add fields, never renumber or repurpose a tag - and that rule is worth
no more than the check that enforces it. Run all three before proposing a change
to any `.proto`.

The vendored `go-plugin` protos are excluded from lint: they are byte-for-byte
upstream copies, not ours to restyle.

## A vendor plugin

A vendor implements one subsystem contract (or several - a binary may serve more
than one), builds against the published stubs in any gRPC-capable language, and
ships a single executable. The host discovers it, handshakes over the frozen wire
protocol, and registers it.

**Start with [`AUTHORING.md`](AUTHORING.md)** - what you build, in the order you
build it. [`PROTOCOL.md`](PROTOCOL.md) is the envelope spec, [`TYPES.md`](TYPES.md)
the payload spec, and the per-subsystem `.proto` files are the authority for the
RPCs themselves.

## Governance and versioning

- The **contract** carries its own SemVer (see `VERSION`). Message evolution
  follows proto rules: add fields, never renumber or repurpose a tag.
- The **plugin wire protocol** carries a pinned integer version, frozen here
  independently of `go-plugin` upstream. If `go-plugin` ever changes its
  handshake/broker behavior, each host pins/shims to keep conforming to **this**
  spec - the spec is the authority, the library is not.
- This artifact lives as its **own standalone repository**, separate from any
  host's codebase, and is **published** under MPL-2.0 by OpenPhase-Labs.

## Status

**DRAFT.** The governance, the frozen base protocol, the wire-type conventions,
all **six wire subsystem contracts** (`decoder`, `method`, `report`, `notify`,
`auth`, `storage`), and the **19 contract ADRs** ([`adr/`](adr/)) are authored,
and the `.proto` set builds and lints clean (ADR-0019).

It stays draft until a TSIGMA host implements it. A contract is proven by a host
built against it, not by the completeness of its own documents - so the version
is not pinned to a release yet, and while it is draft a deliberate breaking
change is still legitimate. The checks above exist to make breakage
**intentional**, not impossible.

Open items are flagged in-file: the privileged-tier enforcement mechanism
(ADR-0016), the auth login-flow transport's final shape (ADR-0017, leaning shape
a), and release mechanics - publishing, tagging, CI (ADR-0019).

## License

Copyright (c) 2026 OpenPhase Labs, LLC.

Licensed under the Mozilla Public License, Version 2.0. See [`LICENSE`](LICENSE).

The `go-plugin` protos under [`proto/vendor/go-plugin/`](proto/vendor/go-plugin/)
are byte-for-byte upstream copies, also MPL-2.0, and retain their own copyright
notices - see [`PIN.md`](proto/vendor/go-plugin/PIN.md).
