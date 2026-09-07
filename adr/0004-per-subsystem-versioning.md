# 0004. Per-subsystem version negotiation, not a single global app-protocol version

- Status: Accepted
- Date: 2026-06-27

## Context

The handshake carries an app-protocol version. With a per-subsystem contract set
([0003](0003-per-subsystem-contract-set.md)), compatibility could be gated on one
global number or negotiated per subsystem.

## Decision

Two independent version axes:

- **Wire-protocol version** - the handshake `APP-PROTOCOL-VERSION` gates the
  envelope only (handshake / health / controller / broker).
- **Per-subsystem contract version** - each subsystem `.proto` is independently
  versioned (`tsigma.decoder.v1`, `tsigma.report.v1`, ...). A plugin declares **at
  registration** which subsystem contracts and versions it implements; the host
  accepts each at a version it supports and rejects only the mismatched ones.

A breaking `decoder/v2` therefore does **not** invalidate anyone's `report`
plugins.

## Rationale

A multi-vendor ecosystem must not gate every vendor's plugins on one global
number. The proto packages were already per-subsystem; only the handshake gate
needed reconciling to match.

## Consequences

- Registration carries per-subsystem version metadata.
- Within a subsystem major, message evolution is additive (add fields, never
  renumber or repurpose a tag), so a newer host loads an older plugin and vice
  versa within that subsystem's major.
- Related: [0002](0002-freeze-wire-protocol.md),
  [0003](0003-per-subsystem-contract-set.md).
