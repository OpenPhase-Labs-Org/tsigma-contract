# 0009. never-send-max: batch/stream, never raise the gRPC cap

- Status: Accepted
- Date: 2026-06-27

## Context

gRPC has a default maximum message size (~4 MB). Decoders and ingestion methods
must handle uploaded files - a multi-day controller log easily exceeds that - and
reports can return large row sets. Raising the cap is a deployment-breaking
ceiling that simply moves with the data size.

## Decision

**No message approaches the gRPC cap, and the cap is never raised.** Any RPC that
carries a file/byte payload or returns rows **streams** instead:

- File-bearing inputs are **client-streamed** as bounded byte chunks.
- Row / event-returning outputs are **server-streamed** as bounded Apache Arrow
  record batches.
- Stateful payloads (a decoder needs the header before the body) are handled by
  the plugin stream-parsing its input; the host does not pre-chunk semantically.

## Rationale

Streaming bounds memory on both sides and removes the cap as a failure mode. A
decoder running on an uploaded file cannot fit it in one message, and raising the
limit just defers the break to the next larger file.

## Consequences

- `Decode` is bidirectional-streaming; `CanDecode` takes a bounded head.
- `EventSinkService.DecodeAndPersist` client-streams bytes.
- Report `Generate` server-streams Arrow batches.
- `ConfigService.ListDevicesForMethod` server-streams one `DevicePair` per
  message, so listing a large fleet never approaches the cap.
- Related: [0008](0008-wire-types-arrow-protobuf.md),
  [0013](0013-decoder-output-arrow-streamed.md),
  [0014](0014-method-raw-bytes-only.md).
