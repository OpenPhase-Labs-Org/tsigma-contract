# 0007. Trust tiers: untrusted-vendor-OK vs privileged (auth/storage)

- Status: Accepted
- Date: 2026-06-27
- Amended: 2026-08-31 - a report is contained by the host API under a
  per-invocation credential, not by broker data
  ([0006](0006-broker-consume-side-data-plane.md))

## Context

The wire boundary is uniform across subsystems, but the blast radius of a
malicious or buggy implementation is not. A bad decoder/method/report/notification
is contained by host mediation; a bad auth provider (it **asserts identity**) or
storage backend (it **holds the data**) is a trust **root** that mediation cannot
contain.

## Decision

Two trust tiers across the same boundary:

- **Untrusted-vendor-OK:** decoder, method, report, notification. Contained by
  crash isolation + no DB credentials + host-mediated data access
  ([0006](0006-broker-consume-side-data-plane.md)): a method reaches only
  tenant-scoped broker services, and a report reads only through the host API,
  under a short-lived credential minted for that one invocation as the user the
  report runs for. An arbitrary vendor binary is safe to load.
- **Privileged - first-party or DOT-sanctioned only:** auth and storage. Same wire
  protocol, but the operator should install only vetted implementations, enforced
  by a host-side gate (see [0016](0016-privileged-tier-enforcement.md)).

The boundary is uniform; the trust posture is not.

## Rationale

- An untrusted decoder/method/report/notification can at worst produce bad data or
  carry a bug - recoverable, and for decoders **market-policed**: a vendor that
  fakes its own data sabotages the diagnostic its own customer uses against it and
  loses customers, so faithfulness is the vendor's own incentive. The
  controller-untrusted axiom is about the **controller in the field**, not the
  decoder; it does not extend to "decoder-untrusted."
- A third-party plugin in the untrusted tier is the **operator's** trust call -
  the same kind of decision as trusting OpenPhase to write TSIGMA correctly.
- Auth forges identity and storage exfiltrates or corrupts the record. Neither is
  recoverable, and neither can be contained by mediation, so both must be vetted.

## Consequences

- No certification apparatus is needed for the untrusted tier; provenance plus
  optional raw re-derivability cover the bug case (these are good engineering on
  their own merits, not trust controls).
- The privileged tier needs a host-side registration gate **by mechanism, not by
  "should"** - the mechanism is open ([0016](0016-privileged-tier-enforcement.md)).
- Related: [0006](0006-broker-consume-side-data-plane.md),
  [0016](0016-privileged-tier-enforcement.md).
