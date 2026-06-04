# Handoff — 2026-06-04

*Updated: openclaw#9 closed — removed from backlog. parent#147 closed — removed from backlog.*

**Head commit (project):** b10e665 — docs(claude-md): add casehub_block, casehub_delegate tools and Layer 3 lifecycle skills
**Head commit (workspace):** beca6f2 — archive(issue-23-layer3-lifecycle-skills): move plans to attic

## What Changed This Session

Phase 2 lifecycle skills shipped. Three new MCP tools: `casehub_block` (@Transactional direct expiresAt mutation — safe because no try/catch in block(); deferred to qhorus#250 for proper extendDeadline()), `casehub_delegate` (HANDOFF with required toAgent — makes delegation machine-readable vs escalation). Three new SKILL.md files: casehub-reject, casehub-block, casehub-delegate. Fixed openclaw#16: OversightGateService.evaluate() now dispatches DONE (not STATUS) via commitmentStore.findOpenByObligor() — Qhorus-native, restart-resilient. ActionRiskClassifier javadoc confirms contract identical to engine#402. Garden: 1 entry (GE-20260604-d08c9f — @Transactional + @Tool technique). PR: casehubio/openclaw#26.

Closed: #23 (lifecycle skills), #24 (ActionRiskClassifier), #16 (DONE dispatch). Filed: qhorus#250 (CommitmentService.extendDeadline).

## Immediate Next Step

Speech act classification Phase 2/3 (openclaw#10) is the next meaningful M-scale work. Or pick up #22 (QuarkusTest regression for OversightGateDispatcher — S · Low, no gates). Run `/work` for either.

## What's Left

- `openclaw#20` — store `channelId` on Commitment entity to survive Quarkus restart · S · Med
- `openclaw#22` — @QuarkusTest atomicity regression test for OversightGateDispatcher · S · Low
- `qhorus#250` — CommitmentService.extendDeadline() to remove casehub_block direct mutation workaround · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#8** | **Epic 8: Speech act classification Phase 2/3** | M | High | openclaw#10 — SpeechActClassifier prefix detection + structured JSON; no longer gated (openclaw#16 resolved) |
| #22 | @QuarkusTest atomicity regression for OversightGateDispatcher | S | Low | No gates — pick up anytime |
| Phase 3 | Multi-agent coordination skills (casehub-broadcast, casehub-vote, casehub-handoff) | L | High | Needs Phase 2 speech act work first |

## References

- Blog: `blog/2026-06-04-mdp01-layer3-lifecycle-skills-done-dispatch.md`
- PR: https://github.com/casehubio/openclaw/pull/26
- Integration spec: `proj/docs/specs/openclaw-integration.md`
- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
