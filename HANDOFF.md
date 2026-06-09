# Handoff — 2026-06-09

**Head commit (project):** ef7e9fb — docs: sync ARC42STORIES.MD — stale scan at session wrap (close #12, #19)
**Head commit (workspace):** workspace main

## What Changed This Session

#12 closed — Reactive SPI variants (`ReactiveOpenClawWorkerProvisioner`, `ReactiveOpenClawCaseChannelProvider`) implemented and landed on main. Both gated on `casehub.qhorus.reactive.enabled=true` (same flag as `ReactiveChannelService`/`ReactiveMessageService` — eliminates co-deployment trap). Also fixed a latent silent-data-loss bug in the existing blocking `OpenClawCaseChannelProvider.openChannel()`: channels created after startup never had `gateway.initChannel()` called, so `OpenClawChannelBackend` was never registered for them and COMMANDs were silently dropped. `ReactiveOpenClawCaseChannelProvider` uses a memoized layout cache (`ConcurrentHashMap + Uni.memoize().indefinitely()`) to eliminate the concurrent-create race.

#19 closed (already resolved — parent repo doc was already updated from Epic 7).

Filed: openclaw#32 (LAYOUT map duplication between blocking and reactive channel providers — tracking issue for when engine adds a 4th channel type), casehubio/parent#215 (reactive SPI doc sync for casehub-openclaw.md deep-dive).

## Immediate Next Step

Run `/work` to start a new issue. Top candidates:
- **#31** — Extract `OversightGateService` to casehub-engine-api (known placement violation) · L · High
- **#29** — Multi-tenancy `tenancyId` propagation · M · Med · blocked on Qhorus

## What's Left

- `qhorus#250` — `CommitmentService.extendDeadline()` to remove `casehub_block` direct mutation · S · Low · peer repo — needs a qhorus session
- casehubio/parent#215 — reactive SPI doc sync for casehub-openclaw.md · XS · Low · filed, awaiting parent session

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Extract OversightGateService to casehub-engine-api | L | High | Coordinate with engine session; known PLATFORM.md violation |
| #29 | Multi-tenancy — tenancyId propagation | M | Med | Blocked on Qhorus multi-tenancy |
| #32 | Extract shared LAYOUT constant (blocking + reactive channel providers) | S | Low | Trigger: engine adding 4th normative channel type |

## References

- Blog: `blog/2026-06-09-mdp02-silent-failure-blocking-code.md`
- Spec: `proj/docs/superpowers/specs/2026-06-09-reactive-spi-variants-design.md`
- Protocols: `proj/docs/protocols/casehub/reactive-harness-gate-matches-infrastructure.md`, `channel-create-requires-init-channel.md` (garden: casehubio/garden)
