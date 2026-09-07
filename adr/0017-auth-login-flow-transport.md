# 0017. Auth login-flow transport over the gRPC wire (HTTP-over-gRPC)

- Status: Proposed (leaning shape a)
- Date: 2026-06-27

## Context

An auth provider owns a login flow that is inherently **HTTP-shaped**: a local
form post (`POST /login`), and for OIDC a redirect out to the IdP
(`GET /oidc/login`) plus the callback the IdP returns the browser to
(`GET /oidc/callback`). These are browser-facing routes, mounted by the host
under `/api/v1/auth/`. The frozen wire protocol is **gRPC**, and the plugin is a
separate process - so the host cannot simply mount the provider's routes.
Mapping an HTTP-handler boundary onto the gRPC wire is a genuine open modeling
decision.

## Decision

**Proposed - leaning shape (a)**, a generic HTTP tunnel:
`HandleHttp(HandleHttpRequest) -> HandleHttpResponse`. The host forwards the raw browser
request (method, path, query, headers, body) for any route under `/api/v1/auth/`,
and the provider returns status/headers/body plus the resolved `Principal` on
success. The host still mints the session (`SessionService`) and owns `Set-Cookie`.

Not final. Two alternatives are retained as flagged drafts:

- **(b)** semantic per-step RPCs (`BeginLogin` / `CompleteLogin`) with the host
  owning the routes - cleaner, but it **relocates route ownership** from the
  plugin to the host, which changes the boundary rather than choosing a
  transport for it.
- **(c)** the provider listens on its own host:port and the host reverse-proxies -
  closest to a native HTTP-handler shape, but partly steps outside the gRPC-only
  wire model.

## Rationale

Shape (a) keeps route ownership with the provider, which is what makes an auth
provider replaceable: a provider that brings its own login flow can be swapped in
without the host learning that flow's steps. (b) would require the host to model
every provider's flow. Shape (a) may change during host development as the
implementation is built; it is not set in stone. Under (a), the consume-side
broker callbacks
(Session / Provision / Audit) are needed, and the provider validates CSRF via the
host.

## Consequences

- `HandleHttp` + `HandleHttpRequest` / `HandleHttpResponse` are provisionally **active** in
  `auth.proto`; shapes (b)/(c) are retained as flagged drafts.
- To be confirmed or revised during host development.
- Related: [0006](0006-broker-consume-side-data-plane.md),
  [0007](0007-trust-tiers.md).
