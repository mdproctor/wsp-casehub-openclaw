# Handoff — 2026-06-05

**Head commit (project):** 0e35191 — docs: sync ARC42STORIES.MD — C8 complete, stale scan at session wrap
**Head commit (workspace):** 3d0c13b — archive(issue-10-c8-speech-act): move plans to attic

## What Changed This Session

C8 (speech act classification Phase 2+3) shipped. Three-tier detection pipeline: JSON envelope → bracket prefix → STATUS fallback. `SpeechActDetection` public utility, `SpeechActResult(type, content, tier)` record, `DetectionTier` enum. `OversightGateService` uses stripped content for dispatch and raw output for COMMAND audit record. `casehub-global` SKILL.md gains case step response protocol (agents must prefix responses). ADR-0003: STATUS fallback chosen over DONE — false completion is invisible, stuck commitment is recoverable. Deployment note in spec: code + skill updates must ship atomically.

Closed: #10 (speech act Phase 2/3), #8 (Epic 8). Filed: openclaw#27 (NliSpeechActClassifier, future), openclaw#28 (inject protocol via ChannelBackend), parent#172 (cross-dep table), parent#175 (audit-content-split protocol). ARC42STORIES.MD synced — C8 ✅, journey complete (C1–C8).

## Immediate Next Step

All epics complete. Pick up any What's Left item below, or run `/work` to start a new issue.

## What's Left

- `openclaw#20` — store `channelId` on Commitment entity to survive restart · S · Med
- `openclaw#22` — @QuarkusTest atomicity regression for OversightGateDispatcher · S · Low
- `openclaw#25` ⚡ — finalize oversight channel `allowedTypes` (claudony#142 closed, unblocked) · S · Low
- `qhorus#250` — CommitmentService.extendDeadline() to remove casehub_block direct mutation · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #22 | @QuarkusTest for OversightGateDispatcher atomicity | S | Low | No gates — quick win |
| #25 ⚡ | Finalize oversight channel allowedTypes | S | Low | Unblocked (claudony#142 closed) |
| #27 | NliSpeechActClassifier powered by casehub-inference-api | M | High | Gates on casehub-neural-text availability |
| #28 | Inject speech act protocol via OpenClawChannelBackend | S | Med | Removes casehub-global noise for non-case-step agents |
| Phase 3 | Multi-agent coordination skills (broadcast, vote, handoff) | L | High | No issue yet; post-C8 work |

## References

- Blog: `blog/2026-06-05-mdp01-c8-speech-act-classification.md`
- ADR: `proj/docs/adr/0003-speech-act-fallback-on-unrecognised-output.md`
- Spec: `proj/docs/specs/2026-06-05-c8-speech-act-classification-design.md`
- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
