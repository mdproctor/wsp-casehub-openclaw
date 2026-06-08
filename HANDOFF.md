# Handoff — 2026-06-09

*Updated: parent#199, parent#205, parent#206 closed — removed from What's Left.*

**Head commit (project):** 5854177 — docs: sync ARC42STORIES.MD — stale scan at session wrap
**Head commit (workspace):** b2a4f0a — feat: promote blog from issue-28-tool-call-first-completion

## What Changed This Session

#28 closed — tool-call-first completion architecture. What started as a narrow injection fix became a first-principles redesign: 10 production classes deleted (full speech act classification layer), `OversightGateService.evaluate()` collapsed from 140 lines to 12 (now archives webhook text as non-resolving STATUS), commitmentId injected into every case step COMMAND via `OpenClawChannelBackend.post()` (fully resolved — both agentId and commitmentId). ADR-0003 superseded by ADR-0004. #27 CLOSED (NliSpeechActClassifier superseded). 159 tests passing. 8 → 2 commits squashed to casehubio/openclaw main.

Also: 2 garden entries (GE-20260608-5087c8 @QuarkusTest correlationId fanOut gotcha; GE-20260608-c28c00 Qhorus STATUS without correlationId undocumented). ARC42STORIES stale scan fixed 4 items.

## Immediate Next Step

Run `/work` to start a new issue. Top candidates: `#29` (multi-tenancy tenancyId propagation, blocked on Qhorus multi-tenancy) or `#30` (Phase 2 gate wiring through CommitmentTools.done() — deferred from #28).

## What's Left

- `qhorus#250` — CommitmentService.extendDeadline() to remove casehub_block direct mutation · S · Low (peer repo — file issue on qhorus session)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #30 | Phase 2 gate wiring: CommitmentTools.done() → ActionRiskClassifier → OversightGateService.openGate() | M | Med | No gates until engine#402 SPI ships |
| #31 | Extract OversightGateService to casehub-engine-api (PLATFORM.md known violation) | L | High | Coordinate with engine session |
| #29 | Multi-tenancy — tenancyId propagation through provisioner and channel bridge | M | Med | Blocked on Qhorus multi-tenancy shipping |
| Phase 3 | Multi-agent coordination skills (broadcast, vote, handoff) | L | High | No issue yet; post-C8 work |

## References

- Blog: `blog/2026-06-08-mdp01-tool-call-first-completion.md`
- Spec: `proj/docs/superpowers/specs/2026-06-08-tool-call-first-completion-design.md`
- ADR: `proj/docs/adr/0004-tool-call-first-completion-signaling.md`
- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
