# Handoff — 2026-06-09

**Head commit (project):** 473d22f — fix(test): update MessageReceivedEvent constructor calls for tenancyId — qhorus API update
**Head commit (workspace):** workspace main

## What Changed This Session

#30 closed — Phase 2 oversight gate wiring. When an agent calls `casehub_done`, the action is now classified by `@RiskClassifier ActionRiskClassifier` CDI beans. If `GateRequired`, a COMMAND is opened on the Qhorus oversight channel with gate context serialized as Java Properties in the message content (persists across JVM restarts). On gate approval, `fulfill()` dispatches DONE to the work channel (closing the agent commitment); on rejection, DECLINE. The `commandMessageId = -1L` sentinel bug was caught in code review and fixed. Two protocols captured: gate fail-open asymmetry rule and sentinel guard rule.

Also fixed pre-existing `MessageReceivedEvent` constructor mismatch (qhorus added `tenancyId` parameter) in `ChannelContextWindowServiceTest` and `ChannelContextWindowObserverTest`.

## Immediate Next Step

Run `/work` to start a new issue. Top candidates:
- **#31** — Extract OversightGateService to casehub-engine-api (known placement violation)
- **#29** — Multi-tenancy tenancyId propagation (blocked on Qhorus multi-tenancy)

## What's Left

- `qhorus#250` — CommitmentService.extendDeadline() to remove casehub_block direct mutation · S · Low (peer repo — file issue on qhorus session)

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #31 | Extract OversightGateService to casehub-engine-api (known placement violation) | L | High | Coordinate with engine session |
| #29 | Multi-tenancy — tenancyId propagation | M | Med | Blocked on Qhorus multi-tenancy |

## References

- Blog: `blog/2026-06-09-mdp01-phase2-gate-wiring.md`
- Spec: `proj/docs/superpowers/specs/2026-06-09-phase2-gate-wiring-design.md`
- Protocols: `proj/docs/protocols/casehub/gate-fail-open-asymmetry.md`, `gate-context-sentinel-guard.md`
