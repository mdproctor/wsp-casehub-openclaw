# Handoff — 2026-05-26

**Head commit (project):** 9cec110 — ci: add repository_dispatch trigger
**Head commit (workspace):** 14e81d1 — init: workspace scaffold

## What Changed This Session

Session was entirely workspace infrastructure recovery. Discovered openclaw and
life workspaces had no isolated git repos — both were sharing the parent casehub
workspace repo. Audited all casehub workspaces (11 correct, 2 wrong). Fixed both:
`wsp-casehub-openclaw` and `wsp-casehub-life` now exist as isolated GitHub repos.
Parent workspace restored (life's stale branch deleted, dirs untracked + gitignored).
casehubio/parent#73 filed for checklist fix — implemented same day by parent session.

Epic #2 never started. Workspace is now correctly initialized and ready.

## Immediate Next Step

Run `work-start` referencing issue #2 (Epic 2 — OpenClaw hook API client).
This is the first time work-start will run against the correctly initialized workspace.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #2 | Epic 2: OpenClaw hook API client | M | Med | Core integration — `core/` module. Start here. |
| #3 | Epic 3: ChannelContextWindow service | M | Med | Ring buffer + REST API |
| #4 | Epic 4: CaseHub SPI implementations | L | High | WorkerProvisioner, ChannelBackend |
| #5 | Epic 5: Python SDK component | M | Med | `before_prompt_build` hook |
| #6 | Epic 6: Bidirectional wiring end-to-end | M | High | |
| #7 | Epic 7: casehub OpenClaw skill pack | M | Med | |
| #8 | Epic 8: Speech act classification | M | High | Research-heavy |

## References

- Spec: `specs/2026-05-25-workspace-git-isolation-fix.md`
- Plan: `plans/2026-05-25-workspace-git-isolation-fix.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
