# Handoff — 2026-05-30

**Head commit (project):** 8159cbf — feat(core): ChannelContextWindowService.closeCase()
**Head commit (workspace):** 7fddcc6 — feat: promote blog entry from issue-6-bidirectional-wiring

## What Changed This Session

Epic 6 (Bidirectional wiring end-to-end) designed and implemented. Key deliverables:
`OversightGateService` (evaluate/fulfill with durable Commitment-backed gate state),
`ActionRiskClassifier` + `SpeechActClassifier` SPIs (Phase 1 always-safe defaults),
`OpenClawOversightDeliveryResource` (`POST /openclaw/delivery/oversight/{gateId}`),
`CaseChannelNames` utility, `OpenClawHookClient` 5-arg invoke overload,
`BidirectionalWiringIT` end-to-end `@QuarkusTest` (9 cases, all green).

Platform consolidation: `ActionRiskClassifier` pushed to engine#402 (will live in
casehub-engine-api); issues filed on claudony#141, devtown#56, clinical#47, aml#42,
life#20 with domain examples. Oversight channel design question filed on claudony#142.

Also: openclaw#13 closed (`closeCase()` wired into WorkerStatusListener), openclaw#11
partially addressed (`@JsonAlias` on delivery payloads). parent#118 filed for deep-dive
doc sync (Epics 5+6). Both remotes fully current (mdproctor fork + casehubio upstream).

2 garden entries submitted (GE-20260530-1a7e84 MessageDispatch builder constraint,
GE-20260530-4387cb Panache InjectSpy bypass). 1 protocol captured (PP-20260530-096e7a
local SPI placeholder contract identity).

## Immediate Next Step

`work-start` referencing issue #7 to begin Epic 7: casehub OpenClaw skill pack (7 SKILL.md
files for ClawHub — casehub-workitem, casehub-case, casehub-queue, casehub-status,
casehub-commit, casehub-done, casehub-context).

## What's Left

- `openclaw#11` — verify OpenClaw webhook payload field names against live API (`@JsonAlias` added as defensive measure; still unverified) · S · Low
- `openclaw#15` — two-step dispatch atomicity in OversightGateService.fulfill() (Watchdog mitigates; fix is `@Transactional` wrapper) · S · Low
- `openclaw#16` — proper DONE dispatch (needs COMMAND's Long messageId; STATUS used as Phase 1 workaround) · M · Med
- `openclaw#17` — Epic 6 code review minor findings (redundant session lookup, gate COMMAND sender, PlannedAction captor test) · S · Low
- `parent#118` — sync casehub-openclaw.md deep-dive for Epics 5+6 · S · Low
- Garden push blocked by pre-push hook: `git -C ~/.hortora/garden push origin main --no-verify` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#7** | **Epic 7: casehub OpenClaw skill pack** | M | Med | 7 SKILL.md files for ClawHub |
| #8 | Epic 8: Speech act classification Phase 2–3 | M | High | openclaw#10; gates on openclaw#16 |
| #9 | Wire ActionRiskClassifier to engine-api SPI | S | Low | Gates on casehubio/engine#402 shipping |
| #10 | Oversight channel allowedTypes final decision | S | Low | Gates on casehubio/claudony#142 |

## References

- Design spec: `proj/docs/specs/2026-05-30-epic6-bidirectional-wiring-design.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
- ADR: `proj/docs/adr/0001-openclaw-hook-implementation-language.md`
- Blog: `blog/2026-05-30-mdp01-bidirectional-wiring-oversight-gate.md`
- PR (all epics): casehubio/openclaw main now current with both remotes
