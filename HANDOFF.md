# Handoff — 2026-05-29

**Head commit (project):** df7a213 — docs: promote Epic 4 design spec from workspace
**Head commit (workspace):** 54bc581 — docs(epic4): apply design journal + archive plan

## What Changed This Session

Epic 4 (CaseHub SPI implementations) designed, implemented, reviewed, and closed.
`ChannelContextWindowService` refactored to two-phase bind API (`bindAgent`/`bindChannel`/`unbindAgent`).
5 new SPI beans in `casehub/`, delivery webhook in `app/`, 108 tests total.
Key discovery: `casehub-engine-api` not `casehub-engine` in the casehub module — full runtime pulls in 31+ unsatisfied CDI beans.
PR #14 opened to casehubio/openclaw. Both repos on `main`. Branch marked closed.

6 garden entries submitted, 1 protocol captured (PP-20260529-ce2de0: engine-api scope rule).
DESIGN.md created from journal (3 sections: Architecture, Module Structure, Key Integration Patterns).

## Immediate Next Step

`work-start` referencing issue #5 to begin Epic 5: Python SDK component (`before_prompt_build` hook).

## What's Left

- openclaw#11 — verify OpenClaw webhook payload field names (`agentId`, `output` — camelCase vs snake_case) against live API before production use · S · Low
- casehubio/parent#95 — sync `casehub-openclaw.md` deep-dive for Epic 4 (Epic 4 status, engine-api dep) · S · Low
- openclaw#13 — `ChannelContextWindowService.closeCase(caseId)` cleanup for `caseChannels` map · S · Low
- Re-capture 5 forage entries from Epic 3 session that were lost to garden write collision (already in this session's 6 — confirmed captured) · ✅ Done

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#5** | **Epic 5: Python SDK component** | M | Med | `before_prompt_build` hook; calls `GET /channel-context/{agentId}?since={windowSeq}`; `appendSystemContext` for compaction safety |
| #6 | Epic 6: Bidirectional wiring end-to-end | M | High | |
| #7 | Epic 7: casehub OpenClaw skill pack | M | Med | |
| #8 | Epic 8: Speech act classification Phase 2–3 | M | High | openclaw#10 |

## References

- Design spec: `proj/docs/specs/2026-05-29-epic4-casehub-spi-design.md`
- Design journal merged: `DESIGN.md` (workspace root)
- Integration spec: `proj/docs/specs/openclaw-integration.md`
- Blog: `blog/2026-05-29-mdp01-casehub-spi-implementations.md`
- PR: casehubio/openclaw#14
