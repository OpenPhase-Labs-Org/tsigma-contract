# 0014. Ingestion method is raw-bytes-only (no decoded-events-in path)

- Status: Accepted
- Date: 2026-06-27

## Context

An ingestion method fetches or receives bytes from a device. If it also **decodes**
them and hands the host finished events, it has performed decode, validation and
provenance itself, and the host's integrity spine
([0010](0010-integrity-spine-ownership.md)) never runs on that data. The contract
draft initially exposed both a raw-bytes path and a decoded-events-in path on the
method's consume-side `EventSinkService`.

## Decision

The untrusted method contract exposes **only the raw-bytes path**
(`EventSinkService.DecodeAndPersist`). The method forwards raw fetched/received bytes
(plus the decoder name and device scope); the host decodes, validates, and
persists. There is **no** decoded-events-in broker RPC - that path is host-internal
(core) and is dropped from the method surface.

## Rationale

This is the integrity guardrail ([0010](0010-integrity-spine-ownership.md))
realized on the wire: an untrusted poller/listener cannot supply decoded events,
because that would let it bypass the host's decode + validate + poison-aware spine.
When the method is out-of-process and untrusted, the whole decode-validate-persist
orchestration sits on the host side of the boundary; the method's job ends at
delivering the bytes it fetched.

## Consequences

- `EventSinkService` has a single RPC, `DecodeAndPersist` (client-streamed bounded
  bytes); the method never decodes.
- The host selects drift-check vs bare persist internally; the method does not
  choose persistence semantics.
- Related: [0005](0005-multi-contract-binaries-stay-separate.md),
  [0010](0010-integrity-spine-ownership.md), [0009](0009-never-send-max.md).
