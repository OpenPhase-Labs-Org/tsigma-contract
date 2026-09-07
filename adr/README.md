# Architecture Decision Records - TSIGMA Contract

These ADRs record the **contract-specific** decisions behind the TSIGMA Contract
and **why** - the rationale that is otherwise lost once it is implicit in the
`.proto` and the host code. They are the standing record, not ephemeral planning.

Scope: **contract-only.** Host-specific and broader architecture decisions live in
their own repos, not here (the contract is language-neutral and so are its ADRs).

## Format

One decision per record, `adr/NNNN-kebab-title.md`:

```
# NNNN. Title

- Status: Accepted | Proposed | Open | Superseded by NNNN
- Date: YYYY-MM-DD
- Amended: YYYY-MM-DD - one line on what changed        (only if amended)

## Context
## Decision
## Rationale
## Consequences
```

An ADR may record an **undecided** decision (Status: Open) so the open question
and its options are on the record, not just in someone's head.

A status may carry a short parenthetical qualifier - `Proposed (leaning shape
a)`, `Accepted (provisional minimal)` - where the record is settled enough to act
on but the ADR itself says it may be revised. The qualifier narrows the base
status; it never replaces it.

An amended record keeps its original Context and Decision and adds a dated
`## Amendment (YYYY-MM-DD)` section. **Consequences are rewritten in place** to
describe the surface as it now stands - a Consequences section that still
describes the pre-amendment surface is a defect, not history.

## Index

| ADR | Title | Status | Amended |
|-----|-------|--------|---------|
| [0001](0001-one-shared-contract-owned-and-published.md) | One shared contract, owned by OpenPhase Labs, LLC and published under MPL-2.0 | Accepted | - |
| [0002](0002-freeze-wire-protocol.md) | Freeze the go-plugin gRPC wire protocol as TSIGMA Plugin Protocol v1 (spec is authority, not the library) | Accepted | - |
| [0003](0003-per-subsystem-contract-set.md) | Per-subsystem contract set, not one monolithic contract | Accepted | - |
| [0004](0004-per-subsystem-versioning.md) | Per-subsystem version negotiation, not a single global app-protocol version | Accepted | - |
| [0005](0005-multi-contract-binaries-stay-separate.md) | Multi-contract binaries allowed; contracts stay separate; host orchestrates (no merged black box) | Accepted | - |
| [0006](0006-broker-consume-side-data-plane.md) | Broker consume-side data-plane: no DB credentials/schema to plugins; host serves tenant-scoped | Accepted | 2026-08-31 |
| [0007](0007-trust-tiers.md) | Trust tiers: untrusted-vendor-OK vs privileged (auth/storage) | Accepted | 2026-08-31 |
| [0008](0008-wire-types-arrow-protobuf.md) | Wire types: Arrow IPC for bulk/tabular, protobuf for structured/small; no language-native serialization | Accepted | - |
| [0009](0009-never-send-max.md) | never-send-max: batch/stream, never raise the gRPC cap | Accepted | - |
| [0010](0010-integrity-spine-ownership.md) | Integrity-spine ownership: host attaches signal_id/device_id/validation_metadata; transport/transform plugins are pure | Accepted | - |
| [0011](0011-validation-is-core.md) | Validation is core, never a wire plugin | Accepted | - |
| [0012](0012-time-semantics-declared-at-describe.md) | Time semantics declared at decoder Describe + carried as an Arrow schema-metadata key | Accepted | - |
| [0013](0013-decoder-output-arrow-streamed.md) | Decoder output is Arrow record batches, streamed (not protobuf-repeated) | Accepted | - |
| [0014](0014-method-raw-bytes-only.md) | Ingestion method is raw-bytes-only (no decoded-events-in path) | Accepted | - |
| [0015](0015-broker-queries-parameterized.md) | Broker queries are parameterized; raw SQL never crosses to a plugin | Accepted | 2026-08-31 |
| [0016](0016-privileged-tier-enforcement.md) | Privileged-tier enforcement mechanism | Open | - |
| [0017](0017-auth-login-flow-transport.md) | Auth login-flow transport over the gRPC wire (HTTP-over-gRPC) | Proposed (leaning shape a) | - |
| [0018](0018-report-view-model-richness.md) | Report view-model richness (minimal vs structured) | Accepted (provisional minimal) | - |

## Self-containment

A record stands on its own. Its Context, Decision and Rationale must hold for a
reader who has only this repository
([0001](0001-one-shared-contract-owned-and-published.md)). An external codebase,
document or ADR set cited as evidence for a decision may not exist later, and a
record whose reasoning rests on one goes stale the moment it is gone - or reads
as an argument from a thing the reader cannot see.

Naming the **subject** of a decision is different from citing **evidence** for
it. A record about an adopted dependency must name that dependency; a record may
not justify itself by pointing at an implementation. Where an external artifact
is named, **name it, never locate it** - no repository URLs, hosts, accounts or
filesystem paths, all of which break the moment the thing moves.

An Open record states its options rather than a conclusion.
