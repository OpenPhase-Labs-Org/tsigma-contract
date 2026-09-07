# 0008. Wire types: Arrow IPC for bulk/tabular, protobuf for structured/small; no language-native serialization

- Status: Accepted
- Date: 2026-06-27

## Context

A host and its plugins must exchange event batches, dataset rows, report rows,
config, control, and report results, and neither end can assume the other shares
its runtime. Putting a language-native serialization on the wire - a dataframe
pickle, a language's own object-graph encoding - would couple the boundary to
one runtime and break "one plugin binary, any conforming host."

## Decision

**No language-native serialization crosses the wire.** Split by payload kind:

- **Bulk / tabular** (decoded event batches, dataset rows, report row sets) ->
  **Apache Arrow IPC record batches**.
- **Structured / small** (config, control, describe/health, file metadata, report
  **view-models**) -> **protobuf**.

Each host maps to its native types **after** the boundary.

## Rationale

- Arrow has independent, interoperating implementations that read the **same**
  buffer without conversion, and protobuf is already language-neutral. One
  plugin binary works against any conforming host.
- It settles the dataframe-vs-view-model question directly: the wire carries
  neither. A report returns a protobuf **view-model** plus Arrow **rows**. What a
  host materializes from those rows - a dataframe, structs, anything else - is
  its own private business and never appears on the wire.

## Consequences

- Canonical Arrow schemas (`DecodedEvent`, `SensorDetection`) live in TYPES.md.
- Reports return a view-model plus Arrow rows, never a host-native table object
  (see [0018](0018-report-view-model-richness.md)).
- Event batches carry the time-semantics schema-metadata key (see
  [0012](0012-time-semantics-declared-at-describe.md)).
- Bulk payloads are streamed, never one large message (see
  [0009](0009-never-send-max.md)).
- Related: [0009](0009-never-send-max.md),
  [0012](0012-time-semantics-declared-at-describe.md),
  [0013](0013-decoder-output-arrow-streamed.md),
  [0018](0018-report-view-model-richness.md).
