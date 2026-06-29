*Updated: life#38 closed — removed from Immediate Next Step.*

# Handoff — 2026-06-28

**Head commit (project):** aeee484 — fix(#55): align with platform#111 rename
**Head commit (workspace):** workspace main

## What Changed This Session

Closed branch `issue-42-plugin-tenant-and-cleanup` via work-end. #42, #47, #48 all closed. Squashed 11 → 3 commits, rebased onto main, pushed. Full build green (126 tests, 0 failures).

Key implementation: `PluginTokenBridgeMechanism` (custom `HttpAuthenticationMechanism` with path guard + null credential transport — framework constraints documented in 3 garden entries), `OpenClawCurrentPrincipal @Priority(150)` (prevents `MissingTenancyException` for non-OIDC SecurityIdentity), `@RolesAllowed(PLUGIN)` on `PluginCommitResource`, TypeScript bearer token support.

ARC42STORIES.MD §10 synced. Design spec committed. Blog entry published. 3 garden entries submitted (auth-mechanism/@TestSecurity conflict, Bearer transport ambiguity, production-only tenancyId regression).

## Immediate Next Step

Pick next work from What's Next.

## What's Left

- `openclaw#31` — parked, superseded by parent#310 (casehub-blocks extraction) · M · Med
- `parent#310` — Epic: casehub-blocks repo creation + pattern extraction · L · High
- `openclaw#51` — protect `/channel-context/{agentId}` with plugin auth · S · Low
- `openclaw#52` — migrate plugin auth from bridge token to OIDC · M · Med · blocked by upstream
- `openclaw#53` — PluginTokenBridgeMechanism quality nits (timing-safe, constructor injection) · XS · Low
- `platform#121` — OidcCurrentPrincipal handle non-OIDC SecurityIdentity types · S · Med
- `parent#318` — sync casehub-openclaw deep-dive for plugin auth changes · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #44 | Delivery endpoint hardening — webhook signing | S | Med | Blocked by upstream OpenClaw |
| #51 | Channel-context endpoint auth | S | Low | Reuse bridge mechanism pattern |
| #53 | PluginTokenBridgeMechanism quality nits | XS | Low | Timing-safe comparison, constructor injection |
| ops#12 | Add agentIds() to DeploymentProviderConfigStore | XS | Low | Unblocks deployment adapter |

**Paused:** `issue-31-extract-oversight-gate-service` on pause stack (superseded by parent#310).

## References

- Spec: `docs/specs/2026-06-27-plugin-auth-cleanup-design.md`
- Blog: `blog/2026-06-28-mdp01-auth-mechanism-traps.md`
- Garden: `GE-20260628-04a38c`, `GE-20260628-f4177d`, `GE-20260628-919f9f`
