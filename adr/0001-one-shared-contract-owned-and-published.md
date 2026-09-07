# 0001. One shared contract, owned by OpenPhase Labs, LLC and published under MPL-2.0

- Status: Accepted
- Date: 2026-06-27

## Context

A TSIGMA host loads plugins - decoders, ingestion methods, reports,
notifications, auth, storage - written by parties other than whoever wrote the
host. The ecosystem value is in a vendor building **one** plugin against a
published interface rather than against a particular host's internals.

The failure mode to avoid: the contract lives inside a host codebase, so each
host carries its own copy, the copies drift, and "one vendor plugin everywhere"
quietly becomes "two dialects, port your plugin twice." A second failure mode: a
host extends the standard privately, and the interface a vendor targets stops
being the published one.

## Decision

The plugin contract is a **single, standalone, published artifact** (this
repository), **owned by OpenPhase Labs, LLC** and licensed **MPL-2.0**. It is governed
here, independently of any host codebase: a change is made once, in this
repository, and consumed from it. Every host **codegens from it**; none keeps a
private copy or a fork. The artifact is language-neutral - it is the standard,
and hosts are implementations of the standard.

This is enforced **by construction** (one `.proto` source, many codegens), not by
vigilance.

## Rationale

- **One plugin, any conforming host.** A vendor builds one plugin against this
  artifact and it runs on any host that conforms. The contract is the neutral
  ground: whatever a host differs in, it does not differ in the interface a
  plugin targets.
- **The open-data-layer promise made real.** Openness lives in the data and
  contract layer. An agency's choice of host does not determine which vendor
  plugins are available to it.
- **Drift is structurally impossible** when every host generates its stubs from
  one source.
- **Governance survives a future where implementations diverge or change hands.**
  A standard published from one owned repository cannot be quietly forked by
  whichever implementation holds the most leverage; MPL-2.0 keeps modifications
  to the contract files themselves open, so a divergent copy cannot be closed.

## Consequences

- The artifact lives in its own repository, separate from any host's codebase,
  and is published under MPL-2.0.
- Every contract change is a change to the **standard**: made once here, consumed
  by every host and every vendor. Message evolution follows proto rules (add
  fields, never renumber or repurpose a tag).
- The artifact must be **self-contained**: a normative rule belongs in the
  contract (PROTOCOL.md / TYPES.md / the `.proto` files), not in a host's private
  docs. Implementers read only this repo.
- Codegen tooling (buf vs raw protoc, lockfiles, CI publish) is a follow-up
  decision, intentionally not pinned at draft time.
- Related: [0002](0002-freeze-wire-protocol.md) (the wire protocol this contract
  freezes), [0003](0003-per-subsystem-contract-set.md) (how the contract is
  partitioned).
