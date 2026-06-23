# Design Journal — issue-41-wire-oidc

### 2026-06-23 · §10 · openclaw#41

casehub-openclaw's REST surface is predominantly system-to-system: OpenClaw webhook
callbacks, Python SDK calls, TypeScript plugin calls. None of these callers carry a
casehub OIDC token. This structural reality — not a policy choice — governs the auth
annotation strategy: `@PermitAll` on the four system-to-system resources (delivery×2,
channel-context, plugin) makes the deliberate absence of auth explicit rather than
implicit; `@RolesAllowed(OpenClawGroups.ADMIN)` on `ExampleController` is appropriate
because it is the only human-triggered endpoint.

The CDI exclusion of `QhorusInboundCurrentPrincipal` resolves an ambiguity introduced
by adding `casehub-platform-oidc`: both `QhorusInboundCurrentPrincipal @ApplicationScoped`
and `OidcCurrentPrincipal @RequestScoped` implement `CurrentPrincipal` with no qualifier
disambiguation → `AmbiguousResolutionException`. Excluding the Qhorus bean makes
`OidcCurrentPrincipal` the sole production winner. The sentinel values for anonymous paths
(`identity.isAnonymous()` → `DEFAULT_TENANT_ID = "278776f9-…"`) match the old
`QhorusInboundCurrentPrincipal` fallback exactly — no behavioural change for @PermitAll
endpoints.

This completes the auth retrofit for openclaw (parent#251). MCP endpoint auth (openclaw#43)
and plugin endpoint tenant isolation (openclaw#42) are deferred — they require a different
auth mechanism (Quarkus MCP server) and upstream OpenClaw support (service-account token)
respectively.
