# Handoff — 2026-06-23

**Head commit (project):** e17f3bf — fix(casehub): PlannedAction SNAPSHOT break — compile against engine main HEAD
**Head commit (workspace):** workspace main

## What Changed This Session

Closed #46 and #45 on branch `issue-46-plannedaction-import-fix`.

- **SNAPSHOT investigation** — the published casehub-engine-api SNAPSHOT was built from an interim state that removed `PlannedAction` from `io.casehub.api.spi`. Local engine `main` still has the 5-field record. Local code already compiled correctly; the fix was diagnostic, not a code change.
- **engine#563** — GateDecision → GateOutcome rename proposal (naming distinction between policy decision and operational result)
- **engine#565** — filed to re-publish engine-api SNAPSHOT from local main HEAD; CI will stay red on #46 until this lands
- **#31 rescoped** — OversightGateService interface already exists in engine-api; S/Low job not L/High. Waiting on engine#563 for the GateOutcome rename before wiring.
- **Minor improvements** — `classifyMostRestrictive()` parameter marked `final`; DemoGateClassifierTest uses `PlannedAction.of()` factory
- **ARC42STORIES.MD §10** — decision record for engine-api local HEAD strategy

## Immediate Next Step

Trigger a CI run on engine `main` to re-publish the casehub-engine-api SNAPSHOT (engine#565). Once published, openclaw CI will go green on #46 without any further code changes.

## What's Left

- `engine#565` — re-publish engine-api SNAPSHOT from main · XS · Low · **peer repo — user action required**
- `engine#563` — GateDecision → GateOutcome rename · S · Low · peer repo
- `openclaw#31` — wire OversightGateService to implement engine-api interface; drop local GateDecision · S · Low · blocked on engine#563
- `openclaw#43` — MCP endpoint auth · M · High
- `openclaw#42` — plugin endpoint tenant isolation (blocked upstream) · M · High
- `openclaw#44` — delivery webhook signing (blocked upstream) · S · Med
- `qhorus#301` — stale QhorusInboundCurrentPrincipal Javadoc · XS · Low · peer repo
- `parent#303` — remove stale deferred note · XS · Low · peer repo

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Wire OversightGateService to engine-api interface | S | Low | Blocked on engine#563 |
| #37 | Migrate Worker imports to casehub-worker-api | S | Low | Open |
| #36 | Read agent provider config from DeploymentProviderConfigStore | M | Med | Open |
| #43 | MCP endpoint auth | M | High | Open |

**Paused:** `issue-31-extract-oversight-gate-service` on pause stack.

## References

- Blog: `blog/2026-06-23-mdp02-plannedaction-api-migration.md`
- Garden: GE-20260623-b460d4 (ide_refactor_rename on import statements)
