# Handoff — 2026-06-15

**Head commit (project):** bd7faa7 — refactor: remove AgentKey — ChannelContextWindowService uses plain agentId key — Closes #33
**Head commit (workspace):** workspace main

## What Changed This Session

Closed issues #32, #34, #33 on branch `issue-32-layout-gate-tenancy`.

- **#32** — `OpenClawNormativeLayout` extracted; both `CaseChannelProvider` implementations now reference a single source of truth for the 3-channel layout; `ChannelSpec.allowedTypes`/`deniedTypes` are `Set<MessageType>` (no more string parsing at call sites)
- **#34** — `OversightGateService.fulfill()` recovers `tenancyId` from `CrossTenantChannelStore` when `GateContext` has no `tenancyId` key (pre-#29 gates); silent for single-tenant, correct for multi-tenant
- **#33** — `AgentKey` removed from `ChannelContextWindowService`; service now uses plain `agentId` key consistent with `OpenClawAgentRegistry`; `GET /channel-context/{agentId}` and `casehub://channel/{agentId}/recent` require no principal; plugin/SDK work without changes

4 squashed commits on `upstream/main` (casehubio/openclaw). All tests green. Protocol `PP-20260615-11b9d2` added.

## Immediate Next Step

Run `/work` to start a new issue. Top candidates: **#31** (OversightGateService extraction to casehub-engine-api, L/High — design required before any code).

## What's Left

- `qhorus#250` — `CommitmentService.extendDeadline()` to remove `casehub_block` direct mutation · S · Low · peer repo
- `casehubio/parent#215` — reactive SPI doc sync for casehub-openclaw.md · XS · Low · awaiting parent session
- `casehubio/parent#239` — deep-dive `docs/repos/casehub-openclaw.md` needs multi-tenancy section · XS · Low · awaiting parent session
- `openclaw#34` — upgrade note for pre-#29 persisted gates — now resolved by code; consider closing the doc-note option

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Extract OversightGateService to casehub-engine-api | L | High | Known PLATFORM.md violation; coordinate with engine session; design brainstorm required |
| #32 | CLOSED | — | — | Closed this session |
| #33 | CLOSED | — | — | Closed this session |
| #34 | CLOSED | — | — | Closed this session |

## References

- Blog: `blog/2026-06-15-mdp01-layout-gate-tenancy.md`
- Protocol: `proj/docs/protocols/casehub/normative-layout-single-source.md` (PP-20260615-11b9d2)
- Integration spec: `proj/docs/specs/openclaw-integration.md` (updated — stale AgentKey ref removed)
