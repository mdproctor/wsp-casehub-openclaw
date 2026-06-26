# Handoff — 2026-06-26

**Head commit (project):** 5dcc01e — docs: sync spec code block — toMillis precision
**Head commit (workspace):** d4323f7 — docs: add diary entry
**Branch:** `issue-50-hardening-config-auth` (covers #50, #36, #43)

## What Changed This Session

Implemented all three issues on branch `issue-50-hardening-config-auth`. Code complete, reviewed, full build green. **Branch is NOT merged — work-end not yet run.**

- **#50 DirectCallBridge hardening** — `orTimeout` + `whenComplete` self-eviction; `DirectCallDeliveryResource` moved from `casehub/` to `app/`; unused `agentId` removed from payload
- **#36 AgentProviderConfigSource SPI** — pluggable agent config; `@DefaultBean` reads from `application.properties`; both provisioners + ExampleController migrated
- **#43 MCP endpoint auth** — `quarkus.http.auth.permission.mcp` HTTP security policy; `@RolesAllowed` returns MCP `-32001` not HTTP 401
- **casehub-ops#12 filed** — `agentIds()` enumeration for DeploymentProviderConfigStore
- **openclaw#37 closed** — Worker import migration was already done during #49

## Immediate Next Step

Run `/work-end` on branch `issue-50-hardening-config-auth`. The branch is implementation-complete, code-reviewed, and build-verified. work-end will: rebase onto main, squash, push to fork, close #50/#36/#43, merge journal, promote artifacts, write final handover.

**Pre-close sweep already done this session:** forage (1 GE submitted), protocol sweep (nothing new), update-claude-md (already current), implementation-doc-sync (checked, nothing stale), ADR (none needed), diary written. Do NOT re-run these — go straight to work-end Step 2 (Flyway re-scan) after confirming the branch summary.

## What's Left

- `openclaw#31` — parked, superseded by parent#310 (casehub-blocks oversight gate extraction) · M · Med
- `parent#310` — Epic: casehub-blocks repo creation + pattern extraction · L · High
- `parent#311` — PLATFORM.md and deep-dive sync for DirectCallBridge · XS · Low
- `engine#563` — GateDecision → GateOutcome rename · S · Low · peer repo
- `openclaw#42` — plugin endpoint tenant isolation · M · High
- `openclaw#44` — delivery endpoint hardening — validate signed webhook payloads · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #42 | Plugin endpoint tenant isolation | M | High | Service-account token for /openclaw/plugin/* |
| #44 | Delivery endpoint hardening — webhook signing | S | Med | Open |
| ops#12 | Add agentIds() to DeploymentProviderConfigStore | XS | Low | Unblocks deployment adapter for #36's SPI |

**Paused:** `issue-31-extract-oversight-gate-service` on pause stack (superseded by parent#310).

## References

- Spec: `docs/specs/2026-06-26-hardening-config-auth-design.md`
- Blog: `blog/2026-06-26-mdp01-three-hardening-passes.md`
- Garden: `GE-20260626-5074cf` — CompletableFuture.orTimeout() toSeconds truncation
