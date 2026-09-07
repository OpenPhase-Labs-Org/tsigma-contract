# Wire-Type Conventions

**No language-native serialization ever crosses the wire** - no dataframe
pickle, no language's own object-graph encoding. Each host maps to its native
types **after** the boundary. This is what makes one plugin binary work against
any conforming host, and it settles the "native table object vs view-model"
question: the wire carries neither - it carries Arrow or protobuf, and each host
builds its own shape from that.

## The split

| Payload kind | Wire form | Examples |
|--------------|-----------|----------|
| **Bulk / tabular** | **Apache Arrow record batches**, server-streamed (each with a declared Arrow schema) | decoded event batches, dataset rows, report row sets |
| **Structured / small** | **protobuf** message | config, control, describe/health, file metadata, report **view-models** |

Bulk payloads are **streamed as a sequence of Arrow record batches**, never a
single blob - so no message approaches the gRPC cap and neither side buffers a
whole file/result in memory (see PROTOCOL.md "Message size invariant").

Arrow has independent, interoperating implementations that read the **same**
buffer without conversion, and protobuf is already language-neutral. After the
boundary a host may materialize a batch into a dataframe, a slice of structs, or
anything else - that mapping is the host's private business and never appears on
the wire.

## Canonical Arrow schemas

### Time semantics (pinned)

A decoder's timestamps are anchored one of two ways, and **which one is a static
property of the vendor format**, not a per-event or per-batch choice:

- **`utc`** - absolute UTC (epoch-based or already-UTC formats, e.g. SEPAC,
  OpenPhase, Peek epoch).
- **`naive_local`** - a controller-LOCAL wall-clock instant with no zone. The
  decoder emits it as-is; the **host** resolves the signal's zone (geo-derived
  from lat/long by default, admin override, deployment default) and converts to
  UTC at ingest. The decoder never converts - it does not know the signal's zone.

This is declared **once, at `Describe`** (`DescribeResponse.time_semantics`), so a
vendor cannot forget a per-batch flag. It is **carried on the wire** as the Arrow
schema-metadata key **`tsigma.time_semantics`** = `"utc"` | `"naive_local"` on
every emitted event batch. A batch whose key is missing or `unspecified` is a
contract violation (the host flags it; it never silently assumes UTC).

### `DecodedEvent` (controller event)

| Field | Arrow type | Notes |
|-------|-----------|-------|
| `event_time` | `timestamp[us]` | Interpreted per the batch's `tsigma.time_semantics` schema-metadata key (above). For `naive_local` the wall-clock fields are taken with no zone; for `utc` it is absolute UTC. |
| `event_code` | `int32` | 0-255 in practice (Indiana Hi-Res), int32 on the wire |
| `event_param` | `int32` | 0-255 in practice, int32 on the wire |

### `SensorDetection` (roadside sensor)

`timestamp[us]` (UTC) + `vendor_tag` (utf8) + `event_type` (int32:
SPEED/CLASSIFICATION/QUEUE/OCCUPANCY) + nullable `mph`/`kph`/`length_feet`/
`vehicle_class`/`lane_number`/`direction_id` (int32) + nullable `occupancy_pct`/
`queue_length_feet` (float64). A null is **absent**, not zero: a sensor that did
not report occupancy is distinct from one that reported 0%, and collapsing the
two would fabricate a reading.

## Canonical persisted event (host-side, for reference only)

A decoder emits only `event_time / event_code / event_param`. The **host**
attaches identity and integrity metadata before persisting; this row is **not** a
plugin output, shown here so the contract's event shape is unambiguous:

`controller_event_log( signal_id, event_time TIMESTAMPTZ, event_code, event_param,
device_id, validation_metadata JSONB )`. `signal_id` / `device_id` /
`validation_metadata` are host-resolved (signal lookup, decoder resolution,
validate-and-flag), never supplied by the decoder plugin.

## Report view-models

Report results cross the wire as **protobuf view-model messages** (structured),
optionally accompanied by an **Arrow** payload for bulk row data. Never a
host-native table object on the wire. Each host renders the view-model with its
own UI layer.

## Presence and nullability

How a host represents these in its own types is its business; what the contract
pins is the **distinction each one carries**, which an implementation must not
flatten away.

| Contract form | What it means |
|---------------|---------------|
| `google.protobuf.Timestamp`, message presence | Unset = **absent**, distinct from the zero instant. |
| Arrow `timestamp[us]` | Absolute UTC, except on a decoded event batch, where the `tsigma.time_semantics` key decides (see Time semantics above). |
| Arrow nullable `int32` / `float64` | Null = **not reported**, distinct from `0`. |
| protobuf scalar | No presence. A zero value and an unset field are indistinguishable - do not use a bare scalar where "absent" must be expressible. |
| Arrow record batch | An ordered row set with a declared schema; the batch boundary is a transport artifact and carries no meaning. |

## Resolved

- **Arrow vs protobuf-repeated for decoder output -> Arrow, streamed.** Once Decode
  is bidirectional-streaming (it must be, to run on files larger than the gRPC cap),
  Arrow record batches are unambiguously right; the "bounded per-file list" argument
  was only attractive under the dropped unary framing.
- **Naive-vs-UTC encoding -> pinned.** Declared at `Describe`
  (`time_semantics`, decoder-level static) and carried as the Arrow schema-metadata
  key `tsigma.time_semantics` (see Time semantics above). No per-row/per-batch
  footgun.
