# 0010. Integrity-spine ownership: host attaches identity/integrity; transport/transform plugins are pure

- Status: Accepted
- Date: 2026-06-27

## Context

The poison-aware integrity spine - validate-and-flag between decode and persist,
with the host attaching identity and integrity metadata - is what makes stored
data trustworthy. If a plugin could supply identity (`signal_id`, `device_id`) or
integrity (`validation_metadata`), a vendor binary could route around the spine.

## Decision

The host owns the integrity spine. A **decoder is a pure transform** (bytes ->
events, plus optional file provenance) and **never** supplies `signal_id`,
`device_id`, or `validation_metadata`. A **method is pure transport** and **never**
decodes or validates. The host resolves identity (signal lookup, decoder
resolution), runs validate-and-flag, and attaches `validation_metadata` before
persist.

## Rationale

- The controller-untrusted axiom requires validate-and-flag to sit where no plugin
  can bypass it.
- Decoding inherently judges **format** validity (skip a malformed row, fail
  closed on a bad header) - that is intrinsic and unavoidable, and belongs to the
  decoder. The host judges the **data against independent references the decoder
  structurally lacks**: server-receive-time, the signal's geo-derived zone, the
  clock-offset trend, the NTCIP code table, neighboring signals. Keeping
  identity/integrity host-side keeps those judgments in the layer a plugin cannot
  reach.

## Consequences

- A decoder emits only `event_time` / `event_code` / `event_param` (plus optional
  provenance); the host attaches the rest.
- The method is raw-bytes-only ([0014](0014-method-raw-bytes-only.md)).
- Validation itself is core ([0011](0011-validation-is-core.md)).
- The guardrail holds even inside a multi-contract binary
  ([0005](0005-multi-contract-binaries-stay-separate.md)).
- Related: [0005](0005-multi-contract-binaries-stay-separate.md),
  [0011](0011-validation-is-core.md), [0014](0014-method-raw-bytes-only.md),
  [0015](0015-broker-queries-parameterized.md).
