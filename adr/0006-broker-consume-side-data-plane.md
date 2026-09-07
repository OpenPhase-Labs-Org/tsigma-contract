# 0006. Broker consume-side data-plane: no DB credentials/schema to plugins; host serves tenant-scoped

- Status: Accepted
- Date: 2026-06-27
- Amended: 2026-08-31 - report reads move to the host API under a per-invocation
  credential; the broker keeps method-side writes

## Context

Plugins need data: a method needs device config and checkpoints; a report needs
events, aggregates, and historical config. Handing them DB credentials would make
every untrusted vendor binary a direct database client with whatever reach those
credentials carry.

## Decision

Each subsystem contract is **bidirectional**. The **implement-side** is host ->
plugin. The **consume-side** is a host-served **broker**: the plugin calls *back*
into host services (config lookups, event/aggregate queries, checkpoint, persist)
over the same `go-plugin` `GRPCBroker` connection. The plugin receives **no DB
credentials and no schema**; the host answers each call from its **own session,
under the request's tenant context**.

## Rationale

This host-mediated data plane - **not a sandbox** - is what contains an untrusted
plugin: it can compute, but it can only reach data the host hands it, tenant
scoped. It removes the need for a WASM / OS sandbox while still containing an
arbitrary vendor binary. Queries are parameterized, never raw SQL (see
[0015](0015-broker-queries-parameterized.md)).

## Amendment (2026-08-31): the carrier for report reads

The decision above stands unchanged. A plugin still receives no DB credentials and
no schema, and the host still answers under the request's tenant context. What
changes is which host surface carries a **report's reads**.

A report reads through the **host's own API**, authenticated with a short-lived
credential the host mints for the invocation, as the user the report is being run
for, and revokes when the invocation ends. The credential travels over the plugin
connection; it is never placed in the plugin's argv or environment.

The four report consume-side query services are withdrawn from
`tsigma/report/v1`. Method-side consume-side services are unaffected and remain
on the broker exactly as described above: checkpoint load/save, error recording,
persist, decoder resolution and device listing. These serve the plugin's own
operation - its checkpoint, its decoder name, the devices it is assigned - rather
than reading the tenant's stored traffic data, and they have no host-API
equivalent.

**Why the carrier changed.** A host API applies the deploying operator's access
policy per data category. A broker query service answering from the host session
does not consult that policy, so it grants a vendor binary a read path around a
control the operator set - while an unauthenticated human requesting the same data
goes through it. Minting the credential as the requesting user makes "the
request's tenant context" that user's actual entitlement rather than a second,
parallel notion of it maintained on the broker.

A plugin therefore reaches strictly less than before, not more: it is bounded by
the operator's policy for the user on whose behalf it runs, for as long as that
one invocation lasts.

## Consequences

- A report defines no consume-side broker services. Method defines checkpoint,
  persist, decoder resolution and device listing; auth defines session,
  provisioning and audit callbacks; pure transforms (most decoders) define none.
- The broker is `go-plugin`-native ([0002](0002-freeze-wire-protocol.md)).
- This is the containment that makes the untrusted tier safe
  ([0007](0007-trust-tiers.md)).
- Related: [0002](0002-freeze-wire-protocol.md), [0007](0007-trust-tiers.md),
  [0015](0015-broker-queries-parameterized.md).
