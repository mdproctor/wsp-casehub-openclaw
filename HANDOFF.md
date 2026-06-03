# Handoff — 2026-06-03

**Head commit (project):** f738ed8 — fix(oversight-gate): atomic dispatch, GATE_SENDER on COMMAND, hook client dedup
**Head commit (workspace):** b463f90 — feat: promote blog entry from issue-15-17-11-batch-cleanup

## What Changed This Session

All S/XS backlog items closed. Key code change: extracted OversightGateDispatcher @ApplicationScoped bean to make both dispatch() calls in OversightGateService.fulfill() atomic — @Transactional on the caller directly breaks fail-open (ArC marks tx rollback-only after swallowed exception; TransactionalException escapes the catch block). GATE_SENDER is now the COMMAND sender in openGate() for audit trail consistency. Hook client double-lookup eliminated. Garden: 2 entries (CDI @Transactional catch block gotcha + technique). Protocol: PP-20260603-d52060 (always-200 delivery endpoints). Blog: transactional-atomicity-cleanup.

Closed: #3 (Epic 3 admin), #11 (webhook field names — @JsonAlias permanent design), #15 (atomicity), #17 (code review findings). Tracked: #18 (upstream status noted), #19 → parent#147 filed.

## Immediate Next Step

Run `/work` to begin Epic 8 (speech act classification Phase 2–3, openclaw#10).

## What's Left

- `openclaw#16` — proper DONE dispatch (COMMAND's Long messageId; STATUS workaround in place) · M · Med
- `openclaw#20` — store `channelId` on Commitment entity to survive Quarkus restart · S · Med
- `openclaw#22` — @QuarkusTest atomicity regression test for OversightGateDispatcher · S · Low
- `parent#147` — update casehub-openclaw.md deep-dive for Epic 7 MCP layer (peer repo — open issue only)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#8** | **Epic 8: Speech act classification Phase 2–3** | M | High | openclaw#10; gates on openclaw#16 |
| #9 | Wire ActionRiskClassifier to engine-api SPI | S | Low | Gates on casehubio/engine#402 shipping |
| #10 | Oversight channel allowedTypes final decision | S | Low | Gates on casehubio/claudony#142 |
| Phase 2 | casehub-reject, casehub-block, casehub-delegate SKILL.md | M | Low | Extend Layer 3 skill pack |
| Phase 3 | Multi-agent coordination skills | L | High | Needs Phase 2 first |

## References

- Blog: `blog/2026-06-03-mdp01-transactional-atomicity-cleanup.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
