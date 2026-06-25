# Handoff — 2026-06-25

**Head commit (project):** 47f1dda — feat(#49): add DirectCallBridge — synchronous webhook bridge for AgentExec workers
**Head commit (workspace):** workspace main

## What Changed This Session

Closed #49 on branch `issue-49-direct-call-bridge` — added DirectCallBridge infrastructure so AgentExec workers can call `/hooks/agent` and get structured results back synchronously.

- **DirectCallBridge** — `CompletableFuture` registry keyed by correlationId; `DirectCallDeliveryResource` receives webhook callbacks at `POST /openclaw/direct-call/{correlationId}`
- **OpenClawAgentProvider** — implements `AgentProvider` (platform SPI); fires `/hooks/agent` via `invokeDirect()`, emits `Multi<AgentEvent>` on webhook completion
- **OpenClawChatModel** — thin langchain4j bridge with schema-in-prompt serialization for structured output
- **API compatibility** — fixed pre-existing compilation breaks from upstream engine-api changes (PlannedAction → casehub-worker-api, ClassificationContext, ChannelCreateRequest)
- **casehub-blocks** — architectural discussion: identified need for a reusable building blocks repo (parent#310). Oversight gate lifecycle (#31) parked in favour of blocks extraction.
- **parent#311** — filed for PLATFORM.md and deep-dive doc sync

## Immediate Next Step

casehub-life#38 is now unblocked — add `casehub-openclaw-core` + `casehub-openclaw-casehub` as dependencies and wire `OpenClawChatModel` via a factory to convert 32 stub workers to real OpenClaw agents.

## What's Left

- `openclaw#50` — DirectCallBridge hardening: future eviction, module placement review · S · Low
- `openclaw#31` — parked, superseded by parent#310 (casehub-blocks oversight gate extraction) · M · Med
- `parent#310` — Epic: casehub-blocks repo creation + pattern extraction · L · High
- `parent#311` — PLATFORM.md and deep-dive sync for DirectCallBridge · XS · Low
- `engine#563` — GateDecision → GateOutcome rename · S · Low · peer repo
- `openclaw#43` — MCP endpoint auth · M · High
- `openclaw#42` — plugin endpoint tenant isolation · M · High
- `openclaw#44` — delivery webhook signing · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #50 | DirectCallBridge hardening | S | Low | Future eviction, module placement |
| #37 | Migrate Worker imports to casehub-worker-api | S | Low | Open |
| #36 | Read agent provider config from DeploymentProviderConfigStore | M | Med | Open |
| #43 | MCP endpoint auth | M | High | Open |

**Paused:** `issue-31-extract-oversight-gate-service` on pause stack (superseded by parent#310).

## References

- Blog: `blog/2026-06-23-mdp02-plannedaction-api-migration.md` (previous session)
