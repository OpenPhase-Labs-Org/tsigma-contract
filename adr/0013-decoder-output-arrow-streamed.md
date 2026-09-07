# 0013. Decoder output is Arrow record batches, streamed (not protobuf-repeated)

- Status: Accepted
- Date: 2026-06-27

## Context

A decoder returns decoded events. Two candidate wire shapes: a protobuf `repeated
DecodedEvent` field, or Apache Arrow IPC record batches.

## Decision

Decoder output is **Apache Arrow record batches, server-streamed** over the
bidirectional `Decode` RPC, each batch carrying the time-semantics schema-metadata
key. Not protobuf-repeated.

## Rationale

Once `Decode` must stream - uploaded files exceed the gRPC cap
([0009](0009-never-send-max.md)) - Arrow record batches are unambiguously right.
The "a single-file decode is a bounded list, so use `repeated DecodedEvent`"
argument was only attractive under a unary framing that
[0009](0009-never-send-max.md) drops. Arrow is the neutral wire form a host maps
to its own native types after the boundary
([0008](0008-wire-types-arrow-protobuf.md)).

## Consequences

- `DecodeResponse` carries `events_arrow_ipc`; `Decode` is bidirectional-streaming;
  `CanDecode` takes a bounded head (not the whole file).
- Related: [0008](0008-wire-types-arrow-protobuf.md),
  [0009](0009-never-send-max.md),
  [0012](0012-time-semantics-declared-at-describe.md).
