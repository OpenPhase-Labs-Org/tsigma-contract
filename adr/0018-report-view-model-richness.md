# 0018. Report view-model richness: minimal vs structured

- Status: Accepted (provisional minimal)
- Date: 2026-06-27

## Context

TYPES.md pins reports to a protobuf **view-model** plus Arrow **rows**
([0008](0008-wire-types-arrow-protobuf.md)). A report's result is tabular -
columns and rows. The open question is whether the view-model carries anything
beyond what is needed to render that table.

## Decision

**Minimal for now.** `ViewModel = {columns, empty, preferred_http_status}`; tabular
rows stream as Arrow batches. No report-specific structured fields (chart axes,
series descriptors, summary-stat blocks). The view-model may be **enriched
additively later** if a host UI requires more.

## Rationale

Chart axes and series descriptors describe how a host **renders** a result, not
what the result **is**. Fixing a presentation model in a language-neutral
contract commits every host to it, ahead of any host that has asked for it - and
a guess frozen into the wire is carried forever, since a tag can be added but
never repurposed ([0001](0001-one-shared-contract-owned-and-published.md)).

The asymmetry decides it. Proto evolution is additive: a richer view-model can
be introduced later without breaking a plugin built against this one. Deferring
therefore costs a later additive change; guessing early costs a permanent field
that may be wrong.

## Consequences

- `ViewModel` stays minimal; enrichment is deferred and additive.
- Confirmed for now; may change.
- Related: [0008](0008-wire-types-arrow-protobuf.md).
