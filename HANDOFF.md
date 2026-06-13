# Handoff — 2026-06-12

**Head commit (project):** 2a02902 — docs: add PP-20260612-520281 to Key Protocols table
**Head commit (workspace):** workspace main

## What Changed This Session

#29 closed — full multi-tenancy tenancyId propagation through casehub-openclaw. `AgentKey(agentId, tenancyId)` composite key in `ChannelContextWindowService` prevents cross-tenant context window collision. Delivery webhooks use `@CrossTenant CrossTenantChannelStore.findById()` for tenancyId recovery (no request principal in webhook context). Gate fulfillment uses `CrossTenantMessageStore.scan(MessageQuery)` — already existed in qhorus#260, no qhorus PR required. Fixed existing 404 protocol violation in `OpenClawDeliveryResource` (was calling tenant-scoped `channelService.findById()`, now cross-tenant). CDI ambiguity from two `@DefaultBean CurrentPrincipal` impls resolved: `quarkus.arc.exclude-types` in production, `quarkus.arc.selected-alternatives` in tests. SNAPSHOT compat fixes applied mid-close when engine#475 (terminate(workerId, tenancyId) SPI) and ChannelCreateRequest API changes landed.

Filed: openclaw#33 (TS plugin + Python SDK tenancyId propagation), openclaw#34 (pre-#29 gate crash recovery — upgrade note), engine#475 (terminate() SPI with tenancyId — already landed by the time this session ended).

## Immediate Next Step

Run `/work` to start a new issue. Top candidates:
- **#31** — Extract `OversightGateService` to casehub-engine-api · L · High
- **#33** — TS plugin + Python SDK tenancyId propagation · M · Med

## What's Left

- `qhorus#250` — `CommitmentService.extendDeadline()` to remove `casehub_block` direct mutation · S · Low · peer repo
- casehubio/parent#215 — reactive SPI doc sync for casehub-openclaw.md · XS · Low · awaiting parent session
- openclaw#34 — document upgrade note for pre-#29 persisted gates in multi-tenant deployments · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Extract OversightGateService to casehub-engine-api | L | High | Known PLATFORM.md violation; coordinate with engine session |
| #33 | TS plugin + Python SDK tenancyId propagation | M | Med | Requires auth retrofit; both L4 and L5 affected |
| #32 | Extract shared LAYOUT constant (blocking + reactive providers) | S | Low | Trigger: engine adding 4th normative channel type |

## References

- Blog: `blog/2026-06-12-mdp01-multi-tenant-tenancyid-propagation.md`
- Spec: `proj/docs/superpowers/specs/2026-06-12-tenancyid-propagation-design.md`
- Protocol: `proj/docs/protocols/casehub/delivery-webhook-cross-tenant-reads.md` (PP-20260612-520281)
