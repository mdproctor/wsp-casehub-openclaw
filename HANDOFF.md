# Handoff — 2026-05-28

**Head commit (project):** 5472692 — chore: branch closed (Epic 3 merged to main)
**Head commit (workspace):** c93a51d — docs: add blog entry 2026-05-28

## What Changed This Session

Epic 3 (ChannelContextWindow service) designed, implemented, reviewed, and merged.
Ring buffer in `core/`, `MessageObserver` SPI impl in `casehub/`, REST endpoint +
`EvictionScheduler` in `app/`. 59 tests across 4 test classes. Two spec compliance
passes, one code quality pass (caught `ConcurrentHashMap` stored-value race).
CLAUDE.md synced; design journal written; blog entry published.

`finishing-a-development-branch` skill removed from superpowers — it's the wrong
terminal step for casehub projects (single-repo only, misses `.meta` cleanup and
artifact promotion). Always use `work-end` instead.

5 forage entries attempted in parallel — all failed due to concurrent garden commits.
Re-capture next session before the context is gone.

## Immediate Next Step

`work-start` referencing issue #4 to begin Epic 4: CaseHub SPI implementations
(WorkerProvisioner, ChannelBackend, CaseChannelProvider, WorkerStatusListener).

## What's Left

- Re-capture 5 forage entries (all lost to garden write collision this session):
  1. `quarkus-junit5-mockito` → `quarkus-junit-mockito` rename (Quarkus 3.31+)
  2. `ConcurrentHashMap.merge()` bin-lock safe; stored value iteration is not
  3. `EnumSource.Mode.EXCLUDE` for future-proof enum parameterised tests
  4. Package-private `@ConfigProperty` fields enable pure JUnit 5 CDI bean tests
  5. `@Scheduled(every="...")` accepts ISO-8601 duration from config properties
- casehubio/parent#82 — sync `casehub-openclaw.md` deep-dive (in-memory vs persisted, current state)
- Verify `sessionName` JSON field name (camelCase vs snake_case) against live OpenClaw API before Epic 4

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#4** | **Epic 4: CaseHub SPI implementations** | L | High | WorkerProvisioner must call `associate()` — see issue #4 comment; verify sessionName first |
| #5 | Epic 5: Python SDK component | M | Med | `before_prompt_build` hook |
| #6 | Epic 6: Bidirectional wiring end-to-end | M | High | |
| #7 | Epic 7: casehub OpenClaw skill pack | M | Med | |
| #8 | Epic 8: Speech act classification | M | High | Research-heavy |

## References

- Design spec: `proj/docs/specs/2026-05-27-channel-context-window-design.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
- Blog: `blog/2026-05-28-mdp01-channel-context-window.md`
- Design journal: `design/JOURNAL.md`
