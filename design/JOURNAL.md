# Design Journal — issue-4-casehub-spi-implementations

### 2026-05-29 · §Architecture

Epic 4 implements the four CaseHub SPI interfaces that complete the bidirectional Qhorus ↔ OpenClaw integration. The central architectural decision was how to coordinate the `ChannelContextWindowService`'s association mapping between agents and channels. The original `associate(agentId, Set<channelIds>)` API required both pieces of information simultaneously, but they arrive from independent SPIs (`WorkerProvisioner` knows the agentId; `CaseChannelProvider` knows the channelIds). This created a notification protocol between the two SPIs with a provable race condition.

The resolution was a two-phase bind API: `bindAgent(agentId, caseId)` called by `WorkerProvisioner`, `bindChannel(caseId, channelId)` called by `CaseChannelProvider`, joined at query time by `query(agentId)` which walks `agentId → caseId → channelIds`. Neither SPI knows about the other. No shared registry. No race. A side effect of this design is that messages arriving on a channel after `bindChannel()` but before `bindAgent()` are captured and available at the first query — a net improvement.

Cleanup is symmetric: `WorkerStatusListener.onWorkerCompleted()` calls both `registry.deregister()` and `service.unbindAgent()`. The `caseChannels` map entry is intentionally retained after unbind — TTL eviction handles ring buffer cleanup, and in-flight messages can still write to the buffer. See openclaw#13 for the deferred `closeCase()` method.

### 2026-05-29 · §Module Structure

`casehub/pom.xml` uses `casehub-engine-api` (the pure-Java SPI interface module) instead of `casehub-engine` (the full Quarkus runtime). The engine runtime pulls in CDI beans with unsatisfied persistence SPIs (`CaseInstanceRepository`, `EventLogRepository`, `JobScheduler` etc.) that have no implementations in the casehub/ module. Using only the API module keeps the dependency graph clean and avoids a proliferation of `quarkus.arc.exclude-types` entries to suppress ambiguous CDI resolutions.

The `quarkus.arc.exclude-types` list in `application.properties` excludes only the three blocking no-op SPI beans that casehub-openclaw replaces (`NoOpWorkerProvisioner`, `NoOpCaseChannelProvider`, `NoOpWorkerStatusListener`). The reactive variants and `EmptyWorkerContextProvider` are intentionally kept — they remain the active implementations for those SPIs since casehub-openclaw does not implement reactive counterparts in this epic.

### 2026-05-29 · §Key Integration Patterns

`OpenClawChannelBackend` self-registers with Qhorus `ChannelGateway` by observing `ChannelInitialisedEvent`. This event fires both at channel creation and at Quarkus startup (Qhorus's startup recovery fires it for all persisted channels). The backend filters for `case-*` channel names and registers itself as the "agent" backend for each. This eliminates the need for any separate startup recovery logic — Qhorus's own recovery mechanism drives it. The pattern was validated against Claudony's implementation which uses the same event for its `ClaudonyChannelBackend`.

The `OpenClawCaseChannelProvider` channel layout (work/observe/oversight with APPEND semantic) deliberately matches Claudony's `NormativeChannelLayout` rather than the platform spec's idealized table, which has different `allowedTypes` values. Claudony's implementation is the deployed ground truth. The spec discrepancy is noted in the channel provider's source comment and will be resolved in parent#93 (consolidation of the normative layout).

`OpenClawDeliveryResource` (`POST /openclaw/delivery/channel/{channelId}`) completes the OpenClaw → Qhorus write-back path. It always returns 200 — OpenClaw's retry behavior on non-2xx responses is unverified (openclaw#11). On dispatch failure, the case step hangs until Watchdog recovery. This is an acknowledged trade-off for Phase 1; it becomes revisable once openclaw#11 resolves the retry contract.
