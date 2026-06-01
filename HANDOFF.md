# Handoff — 2026-05-31

**Head commit (project):** 7f480f3 — docs: sync CLAUDE.md and design journal to Epic 7 implementation
**Head commit (workspace):** fbbe264 — archive(issue-7-openclaw-skill-pack): move plans to attic

## What Changed This Session

Epic 7 (CaseHub OpenClaw skill pack) designed and implemented. Four-layer architecture:

- **Layer 0:** Quarkus MCP endpoint (mcp4j `quarkus-mcp-server-http:1.11.1`) — 9 tools
  (`casehub_commit`, `casehub_done`, `casehub_reject`, `casehub_checkpoint`, `casehub_escalate`,
  `casehub_create_workitem`, `casehub_queue`, `casehub_status`) + 2 resources
- **Layer 1:** TypeScript plugin hooks — `before_tool_call` (auto-commit), `agent_end`
  (auto-done), `session_start` (open commitment injection); `commitment-manager.ts`
- **Layer 2:** `casehub-global` SKILL.md (`always: true`)
- **Layer 3:** 4 stateless SKILL.md files (workitem, case, queue, status)
- **PluginCommitResource:** REST endpoints `/openclaw/plugin/{commit,done,commitments}` for
  plugin auto-commit hooks

86 Java + 34 TypeScript tests, all green. ADR-0002 written (Quarkus-embedded MCP).
Key design decisions: per-turn commitment flag (not stack), CaseHub lifecycle tools excluded
from auto-commit, plugin calls REST directly (not MCP). Epic closed, issue #7 closed,
upstream pushed.

4 garden entries (mcp4j @ResourceTemplate, sed macOS, auto-commit per-turn, OpenClaw
before_tool_call payload). 1 protocol (PP-20260531-43de8e: plugin-hooks-call-rest-not-mcp).

## Immediate Next Step

Start Epic 8 or pick up What's Left items. Run `/work` to begin.

## What's Left

- `openclaw#11` — verify OpenClaw webhook payload field names against live API (`@JsonAlias` still unverified) · S · Low
- `openclaw#15` — two-step dispatch atomicity in OversightGateService.fulfill() (`@Transactional` fix) · S · Low
- `openclaw#16` — proper DONE dispatch (COMMAND's Long messageId; STATUS workaround in place) · M · Med
- `openclaw#17` — Epic 6 code review minor findings · S · Low
- `openclaw#18` — track OpenClaw upstream bug: `after_tool_call` not firing in embedded runs (upgrade plugin when fixed) · XS · Low
- `openclaw#19` — update casehub-openclaw.md deep-dive for Epic 7 MCP layer (peer repo — parent) · S · Low
- `openclaw#20` — store `channelId` on Commitment entity to survive Quarkus restart · S · Med
- `parent#118` — sync casehub-openclaw.md deep-dive for Epics 5+6 · S · Low
- `parent#129` — add casehub-openclaw MCP layer to PLATFORM.md Capability Ownership · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| **#8** | **Epic 8: Speech act classification Phase 2–3** | M | High | openclaw#10; gates on openclaw#16 |
| #9 | Wire ActionRiskClassifier to engine-api SPI | S | Low | Gates on casehubio/engine#402 shipping |
| #10 | Oversight channel allowedTypes final decision | S | Low | Gates on casehubio/claudony#142 |
| Phase 2 | casehub-reject, casehub-block, casehub-delegate SKILL.md | M | Low | Extend Layer 3 skill pack |
| Phase 3 | Multi-agent coordination skills (broadcast, vote, handoff) | L | High | Needs Phase 2 first |

## References

- Epic 7 spec: `proj/docs/specs/2026-05-31-epic7-skill-pack-design.md`
- ADR-0002: `proj/docs/adr/0002-mcp-server-host-process.md`
- Blog: `blog/2026-05-31-mdp01-openclaw-skill-pack.md`
- Integration spec: `proj/docs/specs/openclaw-integration.md`
