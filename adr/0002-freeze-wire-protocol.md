# 0002. Freeze the go-plugin gRPC wire protocol as TSIGMA Plugin Protocol v1

- Status: Accepted
- Date: 2026-06-27

## Context

The contract needs a concrete plugin transport: handshake, health, graceful
lifecycle, and a bidirectional broker. HashiCorp `go-plugin` already provides a
documented, language-neutral gRPC wire protocol for exactly this (magic-cookie +
handshake line on stdout, `grpc.health.v1`, `GRPCController`, `GRPCBroker`), and
a host built on that library gets all of it natively. Two questions: adopt
go-plugin's protocol
or build a bespoke one; and if we adopt it, do we track "whatever go-plugin ships"
or pin our own version.

## Decision

Adopt the `go-plugin` gRPC wire protocol, but **freeze it as TSIGMA Plugin
Protocol v1**, documented in PROTOCOL.md as **our own spec**. Every host conforms
to **our** spec: a host built on the `go-plugin` library gets it natively; any
other host implements the same spec over its own gRPC stack. The spec is the
authority; the library is one implementation of it.

## Rationale

- `go-plugin` provides handshake, process supervision, health, graceful shutdown,
  and the bidirectional broker already, battle-tested. Reimplementing that
  machinery from scratch would rebuild HashiCorp's protocol for no gain.
- Pinning **our own** version insulates the contract from `go-plugin` upstream
  drift: if `go-plugin` changes its handshake/broker, each host pins or shims to
  keep conforming to **this** spec.
- Honest caveat: insulation is at the **spec** level. Each host still reaches the
  protocol through an implementation, so a `go-plugin` upstream change can require
  a pin on a host that uses the library natively, not only a shim on one that
  reimplements the protocol.

## Consequences

- PROTOCOL.md is normative for handshake/health/controller/broker at protocol
  version 1. `go-plugin`'s own service protos are referenced and pinned, not
  forked.
- A host that does not use the `go-plugin` library implements the same protocol
  over its own gRPC stack.
- A future `go-plugin` change is absorbed by a host-side pin, never by changing
  the frozen spec out from under existing plugins.
- Related: [0001](0001-one-shared-contract-owned-and-published.md),
  [0006](0006-broker-consume-side-data-plane.md).
