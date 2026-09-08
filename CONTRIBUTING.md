# Contributing to the TSIGMA Contract

This repository is a **contract**, not an application. It defines the boundary
between a TSIGMA host and the plugins it loads, and it does nothing by itself.
A change here is a change to everyone's interface at once — every host, and
every vendor plugin already built against it.

That makes the bar different from ordinary code review. The question is not
"does this work," it is **"can an implementer read this and get it right without
asking anyone."**

> Writing a plugin rather than changing the contract? You want
> [AUTHORING.md](AUTHORING.md). This document is for changing the contract
> itself.

This supplements the OpenPhase Labs
[organization contributing guide](https://github.com/OpenPhase-Labs-Org/.github/blob/main/CONTRIBUTING.md);
where the two differ, this file wins.

---

## The three gates

Every change to a `.proto` must pass all three before you open a pull request
([0019](adr/0019-buf-as-verification-toolchain.md)):

```
buf build                                    # the set parses and resolves
buf lint                                     # naming and layout conform
buf breaking --against '.git#branch=main'    # no wire- or JSON-breaking change
```

`buf breaking` is the one that matters. `protoc` proves a file compiles; it
cannot tell you that a field number was reused or a message renamed — which is
exactly the failure this contract forbids and exactly the failure that is
invisible in review, because a renumbered tag looks like a trivial diff and
silently reinterprets every message already on the wire.

**While the contract is pre-release, a deliberate breaking change is still
legitimate.** The gate exists to make breakage *intentional*, not impossible. If
`buf breaking` fires and you meant it, say so in the pull request and explain
what implementers have to do. If it fires and you did not mean it, you just
avoided shipping a silent incompatibility.

---

## Decisions are recorded, not just made

Design decisions live in [`adr/`](adr/). If your change alters a decision, or
makes one, it needs a record — a new ADR, or an amendment to the existing one.
Read [`adr/README.md`](adr/README.md) for the format before writing either.

Three conventions that catch people out:

- **Amending keeps the original Context and Decision** and adds a dated
  `## Amendment` section. But **Consequences are rewritten in place** to describe
  the surface as it now stands. A Consequences section still describing the
  pre-amendment surface is a defect, not history.
- **Records must be self-contained.** A record's reasoning has to hold for
  someone who has only this repository. Do not justify a decision by pointing at
  an implementation — implementations get replaced, and the record goes stale
  with them.
- **Name an external artifact, never locate it.** No repository URLs, hosts,
  accounts or filesystem paths; they break the moment the thing moves.

The `.proto` files and the prose docs cite ADRs. ADRs never cite them back.

---

## Rules the contract already committed to

These are settled. A change that contradicts one is not a pull request, it is an
ADR proposing to reverse a decision — which is welcome, but it is a different
conversation and it comes first.

- **No message approaches the gRPC cap, and the cap is never raised**
  ([0009](adr/0009-never-send-max.md)). Any RPC carrying file bytes or returning
  rows streams instead: bounded chunks in, bounded Arrow batches out. If you are
  adding an RPC that could carry an unbounded payload, it streams.
- **Arrow for bulk and tabular, protobuf for structured and small, and no
  language-native serialization on the wire**
  ([0008](adr/0008-wire-types-arrow-protobuf.md)). A host-native table object
  never crosses the boundary.
- **Subsystems are versioned independently**
  ([0004](adr/0004-per-subsystem-versioning.md)). A breaking `decoder/v2` must
  not invalidate anyone's `report` plugins. No `.proto` here imports another
  subsystem's `.proto` — that coupling is what the per-subsystem split exists to
  prevent, so duplicate the enum rather than import it.
- **A plugin gets no database credentials and no schema**
  ([0006](adr/0006-broker-consume-side-data-plane.md)). Anything a plugin needs
  it asks the host for, and the host answers under its own session.
- **No vendor-composed query crosses the boundary**
  ([0015](adr/0015-broker-queries-parameterized.md)). Consume-side services take
  typed keys and typed facts. If you find yourself adding a field that carries a
  fragment of a query, stop.
- **Validation and the integrity spine are host-core**
  ([0010](adr/0010-integrity-spine-ownership.md),
  [0011](adr/0011-validation-is-core.md)). Do not add a surface that lets an
  untrusted plugin supply decoded events, identity, or validation metadata.

---

## Trust tier matters for what you are touching

The wire boundary is uniform; the trust posture is not
([0007](adr/0007-trust-tiers.md)).

- **Untrusted-vendor-OK** — `decoder`, `method`, `report`, `notify`. Assume an
  arbitrary vendor binary implements this and ask what it could do with the
  surface you are adding.
- **Privileged** — `auth` and `storage`. These are trust roots that mediation
  cannot contain: auth asserts identity, storage holds the data. A change here
  needs its security argument made explicitly in the pull request, not left
  implied.

---

## Documentation is part of the change

The prose is normative here, not decoration.

- [`PROTOCOL.md`](PROTOCOL.md) — the envelope: handshake, health, lifecycle,
  broker
- [`TYPES.md`](TYPES.md) — the payloads: Arrow schemas, presence and nullability
- [`AUTHORING.md`](AUTHORING.md) — the plugin author's sequence
- [`adr/README.md`](adr/README.md) — the decision index

If you change a `.proto`, update whichever of these describes it, **in the same
change**. A document describing the previous shape is worse than no document,
because it is believed. If you add or amend an ADR, add its row to the index —
and an index that lists a record that does not exist, or omits one that does, is
its own defect.

---

## The vendored protos are not ours

[`proto/vendor/go-plugin/`](proto/vendor/go-plugin/) holds byte-for-byte
upstream copies, pinned ([0002](adr/0002-freeze-wire-protocol.md)). Do not edit
them, do not restyle them, and do not bring them into lint. They are excluded in
`buf.yaml` deliberately. If they need to move, that is a pin change with its own
justification — see [`PIN.md`](proto/vendor/go-plugin/PIN.md).

---

## Pull request checklist

- [ ] `buf build`, `buf lint`, and `buf breaking` all pass — or the break is
      deliberate and the description says what implementers must do
- [ ] Field numbers unchanged; nothing renumbered, nothing reused
- [ ] No subsystem `.proto` imports another subsystem's `.proto`
- [ ] Prose docs updated in the same change
- [ ] An ADR added or amended if this alters a decision; the index row matches
- [ ] For `auth` or `storage`, the security argument is stated explicitly
- [ ] The description explains the *why*
