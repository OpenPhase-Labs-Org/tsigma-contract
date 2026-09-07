# 0019. `buf` is the contract's verification toolchain; codegen stays with the consumer

- Status: Accepted
- Date: 2026-09-07

## Context

[0001](0001-one-shared-contract-owned-and-published.md) makes two promises that
nothing in the artifact enforced. It says drift is impossible **by construction**
because every host generates from one source, and it says message evolution
follows proto rules - add fields, never renumber or repurpose a tag. Both were
prose. The `.proto` set had no configured tool, so it had never been shown to
parse, let alone checked for a breaking change, and a contract that has not been
compiled is unverified text.

[0001](0001-one-shared-contract-owned-and-published.md) deferred the choice of
tooling as a follow-up, which is the decision this record closes.

## Decision

**`buf` is the contract's verification toolchain**, configured at the repository
root, and the artifact is held to three checks: the set parses and resolves, it
conforms to the standard protobuf naming and layout rules, and it introduces no
wire- or JSON-breaking change against the previous commit.

**Codegen is not centralized here.** The artifact publishes no generation
templates and pins no per-language plugins. A consumer generates stubs with its
own toolchain, which is what [0001](0001-one-shared-contract-owned-and-published.md)'s
language-neutrality requires: the contract is the source, not the build system.

The vendored `go-plugin` protos are excluded from lint. They are byte-for-byte
upstream copies ([0002](0002-freeze-wire-protocol.md)); restyling them would fork
them.

## Rationale

The breaking check is the reason for the decision. `protoc` establishes only that
a file compiles; it cannot tell that a field number was reused or a type renamed,
which is precisely the failure
[0001](0001-one-shared-contract-owned-and-published.md) forbids and precisely the
failure that is invisible in review - a renumbered tag looks like a small diff
and silently reinterprets every message already on the wire. An evolution rule is
worth no more than the check that enforces it.

Lint earns its place for the same reason at a smaller scale. Naming is wire-
visible: a service name sits in the gRPC path and an enum value is scoped to its
package, not its enum, so both are frozen the moment anyone generates. Conforming
to the standard is free before that point and a breaking change after it.

Declining to own codegen follows from language-neutrality. Pinning generation
plugins would make the contract carry opinions about consumers' build systems and
would quietly privilege whichever languages were pinned.

## Consequences

- The repository root carries a `buf` configuration: the module is the `.proto`
  tree with the vendored directory excluded from lint, lint is the standard rule
  set, and the breaking check is wire-and-JSON compatibility.
- The three commands are documented in the artifact's README and are expected to
  pass before any `.proto` change is proposed.
- The contract conforms to the standard rule set as of this record; it was
  brought to lint-clean before its first commit, while renaming was still free
  ([0018](0018-report-view-model-richness.md) makes the same argument for
  deferring a guess rather than freezing one).
- While the contract is pre-release, a deliberate breaking change is still
  legitimate. The check exists to make breakage **intentional**, not impossible.
- No generation template ships with the artifact. Consumers generate with their
  own tooling.
- Release mechanics - publishing, tagging, running these checks in CI - remain
  open and are not decided here.
- Related: [0001](0001-one-shared-contract-owned-and-published.md),
  [0002](0002-freeze-wire-protocol.md).
