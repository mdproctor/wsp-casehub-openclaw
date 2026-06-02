# Handoff — 2026-06-02

**Head commit (project):** 48e190c — docs(claude-md): add ARC42STORIES.MD to Project Artifacts; note LAYER-LOG superseded
**Head commit (workspace):** 540cf5f — docs: add project blog entry 2026-06-02-arc42stories-integration-tier

## What Changed This Session

ARC42STORIES.MD bootstrapped and merged to main (issue #21 closed). Full §1–§13 with custom L1–L6 module-tier taxonomy (integration tier — not a harness app, not a foundation module). LAYER-LOG.md updated with migration note and corrected Epic statuses. CLAUDE.md updated to reference ARC42STORIES.MD. Blog entry written.

Quality sweep caught: `parent#118` and `parent#129` both closed — removed from What's Left. Epic 3 issue (`openclaw#3`) still open despite code being complete — flagged in §12, added to What's Left.

## Immediate Next Step

Close `openclaw#3` (Epic 3 epic issue — ChannelContextWindow code complete since 2026-05-28; issue left open after Flyway migration was dropped from scope). Then run `/work` to begin Epic 8 or pick up What's Left items.

## What's Left

- `openclaw#3` — close the Epic 3 GitHub issue (code complete; never closed after Flyway migration dropped) · XS · Low
- `openclaw#11` — verify OpenClaw webhook payload field names against live API (`@JsonAlias` still unverified) · S · Low
- `openclaw#15` — two-step dispatch atomicity in OversightGateService.fulfill() (`@Transactional` fix) · S · Low
- `openclaw#16` — proper DONE dispatch (COMMAND's Long messageId; STATUS workaround in place) · M · Med
- `openclaw#17` — Epic 6 code review minor findings · S · Low
- `openclaw#18` — track OpenClaw upstream bug: `after_tool_call` not firing in embedded runs · XS · Low
- `openclaw#19` — update casehub-openclaw.md deep-dive for Epic 7 MCP layer (peer repo — parent) · S · Low
- `openclaw#20` — store `channelId` on Commitment entity to survive Quarkus restart · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#8** | **Epic 8: Speech act classification Phase 2–3** | M | High | openclaw#10; gates on openclaw#16 |
| #9 | Wire ActionRiskClassifier to engine-api SPI | S | Low | Gates on casehubio/engine#402 shipping |
| #10 | Oversight channel allowedTypes final decision | S | Low | Gates on casehubio/claudony#142 |
| Phase 2 | casehub-reject, casehub-block, casehub-delegate SKILL.md | M | Low | Extend Layer 3 skill pack |
| Phase 3 | Multi-agent coordination skills (broadcast, vote, handoff) | L | High | Needs Phase 2 first |

## References

- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
- Blog: `blog/2026-06-02-mdp01-arc42stories-integration-tier.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
