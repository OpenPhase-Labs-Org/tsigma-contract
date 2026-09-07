# 0003. Per-subsystem contract set, not one monolithic contract

- Status: Accepted
- Date: 2026-06-27

## Context

TSIGMA loads several kinds of plugin: decoder, ingestion method, report,
notification, auth provider, storage backend. The contract could be one large
service surface or one contract per subsystem.

## Decision

One `.proto` package per **wire** subsystem: `decoder`, `method`, `report`,
`notify`, `auth`, `storage` (six). Validation is **core**, not a wire subsystem,
and has no `.proto` (see [0011](0011-validation-is-core.md)). Each subsystem
contract is self-contained and independent.

## Rationale

- A vendor implements only the contract(s) it builds; it does not pull an
  unrelated surface.
- Subsystems evolve independently (see [0004](0004-per-subsystem-versioning.md));
  a change to one does not churn the others.
- Trust posture differs per subsystem (see [0007](0007-trust-tiers.md)) - a
  monolith would blur that boundary.

## Consequences

- Layout is `proto/tsigma/<subsystem>/v1/`.
- A single binary may still serve several contracts as a plugin set, with the
  contracts kept separate (see [0005](0005-multi-contract-binaries-stay-separate.md)).
- Related: [0004](0004-per-subsystem-versioning.md),
  [0005](0005-multi-contract-binaries-stay-separate.md),
  [0007](0007-trust-tiers.md), [0011](0011-validation-is-core.md).
