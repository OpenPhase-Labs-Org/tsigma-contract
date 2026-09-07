# 0015. Broker queries are parameterized; raw SQL never crosses to a plugin

- Status: Accepted
- Date: 2026-06-27
- Amended: 2026-08-31 - no vendor-composed query crosses the boundary at all; the
  rule stands for anything that would

## Context

On the consume-side broker ([0006](0006-broker-consume-side-data-plane.md)), a
plugin constructs the queries it sends to the host. A query interface is easy to
build with a SQL escape hatch in it - a `where` fragment on an event fetch, a
conditional-aggregate filter - guarded only by the convention that callers are
in-house code that will not interpolate untrusted input. Across a plugin
boundary that convention does not hold: the caller is a vendor binary.

## Decision

Broker query services are **parameterized**; **raw SQL never crosses to a plugin**.
No field carries a SQL fragment. A conditional aggregate takes a **structured
predicate** the host compiles to SQL under its own session - never a
vendor-supplied SQL string.

## Rationale

An untrusted-vendor-OK report plugin with a raw-SQL field would hold arbitrary SQL
against the host session - the exact opposite of the broker containment in
[0006](0006-broker-consume-side-data-plane.md) and the trust model in
[0007](0007-trust-tiers.md). Typed parameters - an explicit value set, a
membership test, a split-query RPC, a structured predicate - cover the cases a
fragment would have been reached for; any further need is served by adding a
parameterized field, never a raw-SQL escape hatch.

## Amendment (2026-08-31): the case it guarded no longer arises

The rule stands. The case it was written against has gone: with report reads on
the host API ([0006](0006-broker-consume-side-data-plane.md), amended), a report
composes no query that crosses the boundary, so there is no vendor-supplied
predicate for the host to compile and no field a raw-SQL escape hatch could
occupy.

The rule keeps live subjects. Consume-side **reads** remain - a checkpoint load
keyed by `(method, device_type, device_id)`, decoder resolution by name or by
filename extension, the device list for a method - alongside consume-side
**writes**: checkpoint save, error record, persist, and the auth provider's
session, provisioning and audit callbacks. Every one of them takes typed keys or
typed facts that the host acts on under its own session. None takes a query, and
this rule is why none may.

This is a stronger form of what the decision protected, not a relaxation of it.

## Consequences

- No consume-side service carries a SQL string or a vendor-composed predicate.
  The reads take typed keys; the writes take typed facts.
- A consume-side service added later that does accept a query is bound by this
  rule: the need is served by adding a parameterized field, never a raw-SQL
  escape hatch.
- Related: [0006](0006-broker-consume-side-data-plane.md),
  [0007](0007-trust-tiers.md).
