# Handoff — 2026-06-23

**Head commit (project):** a6911ad — docs(arc42): §10 OIDC auth annotation strategy — openclaw#41
**Head commit (workspace):** workspace main

## What Changed This Session

Implemented and closed #41 — OIDC wiring for casehub-openclaw.

- **OIDC activation** — `casehub-platform-oidc` compile dep; `OidcCurrentPrincipal` is now the production `CurrentPrincipal`
- **Annotation sweep** — `@PermitAll` on 4 system-to-system REST resources; `@RolesAllowed(OpenClawGroups.ADMIN)` on `ExampleController`
- **CDI fix** — `QhorusInboundCurrentPrincipal` added to `quarkus.arc.exclude-types` (stale Javadoc claimed `@Priority(100)` — doesn't exist; qhorus#301 filed)
- **OIDC test config** — test app.properties now has `discovery-enabled=false` + `jwks-path` + `devservices.enabled=false` (dev profile doesn't reach @QuarkusTest)
- **OpenClawGroups** — `ADMIN = "openclaw-admin"` constants class
- **Protocol** — PP-20260623-c3244e: QhorusInboundCurrentPrincipal CDI exclusion rule
- **Garden** — 3 entries: stale Javadoc gotcha, %dev/%test isolation gotcha, @PermitAll explicit intent technique
- **Squash** — 6 commits → 3 clean commits on main

Also completed branch hygiene at session start: all 13 unstamped branches stamped and pushed; issue-31 properly paused (on stack).

## Immediate Next Step

Fix the pre-existing `PlannedAction` build failure (openclaw#46) — `io.casehub.api.spi.PlannedAction` was moved to `io.casehub.worker.api.PlannedAction`. Run `/work` to start a new branch.

## What's Left

- `openclaw#46` — PlannedAction import fix (`OversightGateService.java`, `DemoGateClassifier.java`) · XS · Low
- `openclaw#42` — plugin endpoint tenant isolation (service-account token from OpenClaw) · M · High · blocked upstream
- `openclaw#43` — MCP endpoint auth (Quarkus MCP server mechanism) · M · High
- `openclaw#44` — delivery endpoint webhook signing (OpenClaw upstream) · S · Med · blocked upstream
- `openclaw#45` — minor test polish from code review · XS · Low
- `qhorus#301` — stale QhorusInboundCurrentPrincipal Javadoc update · XS · Low · peer repo
- `parent#303` — remove stale deferred note in casehub-openclaw.md · XS · Low · peer repo
- `casehubio/parent#215` — reactive SPI doc sync for casehub-openclaw.md · XS · Low
- `casehubio/parent#262` — docs/repos/casehub-openclaw.md needs examples/ update · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #46 | Fix PlannedAction import (engine-api move) | XS | Low | Immediate — blocks full build |
| #31 | Extract OversightGateService to casehub-engine-api | L | High | Paused (on stack) — design spec at iter 7 |
| #37 | Migrate Worker imports to casehub-worker-api | S | Low | Open |
| #36 | Read agent provider config from casehub-ops DeploymentProviderConfigStore | M | Med | Open |

## References

- Blog: `blog/2026-06-23-mdp01-oidc-wiring-auth-retrofit.md`
- Spec: `docs/superpowers/specs/2026-06-23-oidc-wiring-design.md`
- Protocol: `docs/protocols/casehub/oidc-cdi-qhorus-exclusion.md`
- Garden: GE-20260623-22f1f7, GE-20260623-24ab25, GE-20260623-e9ac8d
