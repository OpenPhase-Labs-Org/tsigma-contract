# 0005. Multi-contract binaries allowed; contracts stay separate; host orchestrates

- Status: Accepted
- Date: 2026-06-27

## Context

A vendor that owns both a device and its log format may want to ship one binary
serving both the `method` and `decoder` contracts. The risk is a merged
"poll-and-decode" black box that fetches and decodes internally and routes around
the host pipeline.

## Decision

A single binary MAY register multiple subsystem contracts (a `go-plugin` plugin
set). But the contracts **stay separate** even inside a combined binary: the host
still orchestrates `fetch -> decode -> validate -> persist`, and there is **never
a merged poll-and-decode path**. A decoder registration still does bytes -> events
only; a method registration still does transport only.

## Rationale

- Convenience for a vendor who owns device + format, **without** sacrificing the
  poison-aware integrity spine that sits between decode and persist (see
  [0010](0010-integrity-spine-ownership.md)).
- Decoder reusability: a decoder must work on a payload from a different method,
  or an uploaded file. A black-box poll-and-decode kills that reuse.
- The guardrail is load-bearing for data integrity - a merged path would let an
  untrusted vendor binary launder data around validate-and-flag.

## Consequences

- Plugin-set registration is supported; the host composes the pipeline from
  separate contract calls.
- Realized concretely as method raw-bytes-only
  ([0014](0014-method-raw-bytes-only.md)) and decoder pure-transform
  ([0010](0010-integrity-spine-ownership.md)).
- Related: [0003](0003-per-subsystem-contract-set.md),
  [0010](0010-integrity-spine-ownership.md),
  [0014](0014-method-raw-bytes-only.md).
