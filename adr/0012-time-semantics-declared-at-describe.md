# 0012. Time semantics declared at decoder Describe + carried as an Arrow schema-metadata key

- Status: Accepted
- Date: 2026-06-27

## Context

Some decoders emit absolute **UTC** (SEPAC, OpenPhase, epoch-based); others emit
controller-**local naive** time (ASC/3, MaxTime, D4, Peek). The host must know
which, to resolve the signal's zone and convert to UTC at ingest. A per-event or
per-batch flag a vendor can forget is a silent-corruption-at-scale risk - it is the
per-signal-timezone thesis, and getting it wrong poisons data quietly.

## Decision

Time semantics are a **static property of the decoder** (`utc` | `naive_local`),
declared **once at `Describe`** (`DescribeResponse.time_semantics`) and carried on
the wire as the Arrow schema-metadata key **`tsigma.time_semantics`** on every
emitted event batch. A missing or unspecified key is a **contract violation the
host flags** - never a silent UTC assumption.

## Rationale

It is a property of the vendor format, not a per-event choice, so declaring it once
removes the footgun. The host resolves the signal's zone (geo-derived default,
admin override, deployment default) and converts `naive_local` instants to UTC at
ingest; the decoder never converts, because it does not know the signal's zone.

## Consequences

- `DescribeResponse` carries `time_semantics`; Arrow batches carry the metadata
  key; the host enforces its presence.
- Related: [0008](0008-wire-types-arrow-protobuf.md),
  [0013](0013-decoder-output-arrow-streamed.md),
  [0010](0010-integrity-spine-ownership.md).
