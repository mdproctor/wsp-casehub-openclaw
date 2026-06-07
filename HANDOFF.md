# Handoff — 2026-06-07

**Head commit (project):** 52b8623 — protocol(PP-20260607-84b26d): mcp-tool-no-instance-cache
**Head commit (workspace):** 1a1e54d — archive(issue-20-channel-crash-recovery): move plans to attic

## What Changed This Session

S-item batch closed: #20, #22, #25. Dropped the `ConcurrentHashMap<String, UUID> channelMap` from `CommitmentTools` — replaced with `resolveChannelId()` reading from the Qhorus `Commitment` entity with a terminal-state filter. Added `selfCommit_done/reject` state guards (COMMITMENT_ALREADY_CLOSED on post-transfer done/reject). Oversight channel gains `deniedTypes=EVENT` via a new `private record ChannelSpec` replacing a `String[]` LAYOUT. `OversightGateDispatcherCdiTest` added for CDI wiring + fail-open verification. 35 tests passing.

Also: protocol PP-20260607-84b26d (no in-memory caches in @ApplicationScoped MCP beans), 2 garden entries (Mockito overload matcher silent failure GE-20260607-7d0aa8; clearInvocations spy sequencing GE-20260607-b63e95). Branch squashed 31→18 commits, pushed to casehubio/openclaw main.

## Immediate Next Step

All S-items done. Run `/work` to start a new issue. Next candidates: `#27` (NliSpeechActClassifier, M/High, gates on neural-text) or `#28` (inject speech act protocol via ChannelBackend, S/Med, no gates).

## What's Left

- `qhorus#250` — CommitmentService.extendDeadline() to remove casehub_block direct mutation · S · Low (peer repo — file issue on qhorus session)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #28 | Inject speech act protocol via OpenClawChannelBackend | S | Med | No gates — removes casehub-global noise for non-case-step agents |
| #27 | NliSpeechActClassifier powered by casehub-inference-api | M | High | Gates on casehub-neural-text availability |
| Phase 3 | Multi-agent coordination skills (broadcast, vote, handoff) | L | High | No issue yet; post-C8 work |

## References

- Blog: `blog/2026-06-07-mdp01-s-items-crash-recovery.md`
- Protocol: `proj/docs/protocols/casehub/mcp-tool-no-instance-cache.md`
- Spec: `proj/docs/superpowers/specs/2026-06-06-s-items-design.md`
- ARC42STORIES.MD: `proj/ARC42STORIES.MD`
