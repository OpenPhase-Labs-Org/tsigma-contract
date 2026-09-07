# 0011. Validation is core, never a wire plugin

- Status: Accepted
- Date: 2026-06-27

## Context

Prose elsewhere lists "validators" among the plugin subsystems because they share
the registry shape. But the validate-and-flag spine is the host's poison-aware
integrity guarantee, and the host is accountable for it.

## Decision

Validation is **core**: in-process first-party code in the host, with **no
gRPC contract in this artifact**. Two systems, both core:

1. the **post-ingest layer pipeline** (the `Validator` interface - schema/range
   plus future ML layers), and
2. the **ingest-time integrity program** (temporal integrity, MAC-identity,
   config-phase drift, clock-health trending).

`validator` shares the registry *shape* of the wire subsystems but is **not** one
of them: it gets no `.proto` and loads no binary.

## Rationale

Exposing validation across the plugin boundary would let a third party omit or
override the checks the host is accountable for; a pass-everything validator
defeats the poison-aware thesis.
The integrity guarantee is "the host validates against references the plugin does
not hold" ([0010](0010-integrity-spine-ownership.md)), not "a plugin validates."

## Consequences

- Six wire subsystems carry `.proto` contracts; validation does not.
- Any custom validators are first-party / host-supervised, never untrusted vendor
  binaries.
- Related: [0007](0007-trust-tiers.md),
  [0010](0010-integrity-spine-ownership.md).
