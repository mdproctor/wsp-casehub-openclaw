# Handoff — 2026-06-22

**Head commit (project):** `7d1a851` — refactor(#38): replace OpenClawNormativeLayout with NormativeChannelLayout from casehub-engine-api

## What Changed This Session

Implemented and closed openclaw#38 as part of parent#93 (extract normative channel layout).

**Deleted:**
- `OpenClawNormativeLayout` (replaced by `NormativeChannelLayout` from engine-api)
- `OpenClawNormativeLayoutTest`
- `docs/protocols/casehub/normative-layout-single-source.md` (promoted to garden as platform protocol)

**Updated:**
- `OpenClawCaseChannelProvider` — `layout.channelsFor(caseId, null)` + `orElseThrow` for unknown purposes (was silent null fallback). New test: `openChannel_unknownPurpose_throwsIllegalArgumentException`.
- `ReactiveOpenClawCaseChannelProvider` — `initializeLayout` refactored to iterate `List<ChannelSpec>` directly; `openOrCreate(UUID, String)` → `openOrCreate(UUID, CaseChannelLayout.ChannelSpec)` (eliminates second map lookup).

150 tests green.

## Immediate Next Step

Start a new issue — top candidates:
- **openclaw#39** — update ARC42STORIES.MD for OpenClawNormativeLayout removal
- **openclaw#31** — extract OversightGateService to casehub-engine-api (L/High, design required)

Run `/work` to begin.

## What's Left

- `qhorus#250` — CommitmentService.extendDeadline() · S · Low · peer repo
- `casehubio/parent#215` — reactive SPI doc sync for casehub-openclaw.md · XS · Low
- `casehubio/parent#262` — docs/repos/casehub-openclaw.md needs examples/ documentation · XS · Low
- `openclaw#39` — ARC42STORIES.MD update for mesh SPI migration · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #39 | Update ARC42STORIES.MD for OpenClawNormativeLayout removal | XS | Low | Quick doc update |
| #31 | Extract OversightGateService to casehub-engine-api | L | High | Brainstorm + engine coordination required |
