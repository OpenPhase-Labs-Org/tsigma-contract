# 0016. Privileged-tier enforcement mechanism

- Status: Open
- Date: 2026-06-27

## Context

The privileged tier (auth, storage; [0007](0007-trust-tiers.md)) must be enforced
by a **host-side gate** - a binary physically cannot register as `auth` or
`storage` without authorization - **not** by operator discipline ("should install
only vetted ones"). A trust root protected only by "should" is not protected. But
**who authorizes** depends on **who operates the deployment**: a controller-unit
vendor, a third-party integrator, or a DOT/Agency itself - each implies a different
trust anchor.

## Decision

**OPEN.** The **requirement** is fixed: a host-side registration gate keyed to the
subsystem a binary registers for. The **mechanism** is not decided. Options on the
table:

- **Signed plugins** - the host verifies a publisher key. Open sub-question: whose
  key (OpenPhase first-party, a DOT-sanctioned signer, the integrator)?
- **Operator allowlist** - binary hash/identity keyed to the permitted subsystem.
- **Explicit per-binary approval** at registration.

These may combine, and may differ by operator type.

## Rationale (for leaving it open)

The operator-identity question (CU vendor / integrator / DOT) genuinely changes the
trust anchor and must be resolved before a mechanism can be chosen. This is more
urgent than the deferred vendor-licensing / sandboxing questions, because it guards
the identity and data trust roots.

## Consequences

- Until decided, PROTOCOL.md section 6 carries the **requirement** and flags the
  mechanism as open.
- The contract **shape** of `auth`/`storage` is unaffected - the gate is host
  policy, not service shape - so this does not block authoring or implementing
  those subsystems.
- Related: [0007](0007-trust-tiers.md).
