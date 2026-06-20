# Design Journal — issue-31-extract-oversight-gate-service

§ Oversight Gate SPI extraction design — June 17–20 2026

The core design challenge was where to place the `OversightGateService` SPI and how to
handle classification delegation across the module boundary.

**SPI placement:** `GateDecision`, `OversightGateService`, `ReactiveOversightGateService`
go to `io.casehub.api.spi` in `casehub-engine-api`. `GateContext` and
`OversightGateDispatcher` stay package-private in openclaw — implementation details, not
SPI surface. `evaluate()` (webhook archiving) stays on `OpenClawOversightGateService`
concrete class only — OpenClaw-specific, has no platform equivalent.

**Classification delegation:** The existing `classifyMostRestrictive()` private method in
`OversightGateService` duplicated `ChainedReactiveActionRiskClassifier` logic (blocking
classifiers only, silently dropped reactive classifiers). The correct fix: delete the
private method and delegate to the injected `ReactiveActionRiskClassifier`
(`.classify(action).await().indefinitely()` — safe in the `@Blocking` MCP context).
This required moving `ChainedReactiveActionRiskClassifier` from engine runtime to
`io.casehub.api.classification` in engine-api so openclaw-casehub (which depends only on
engine-api, not engine runtime) can inject it.

**Reactive variant:** `ReactiveOpenClawOversightGateService` uses the thin delegate
pattern (not split-class). A full reactive re-implementation would require
`@IfBuildProperty`-gated reactive Qhorus services, coupling the reactive gate to the
Qhorus reactive deployment flag. The thin delegate avoids this:
`Uni.createFrom().item(() -> delegate.openGate(...)).runSubscriptionOn(workerPool)`.
No `@IfBuildProperty` needed — always safe to activate.

**Thread constraint:** `OpenClawOversightGateService.openGate()` calls
`await().indefinitely()` on the classifier result, making it unsafe on Vert.x IO threads.
Reactive callers must use `ReactiveOversightGateService`. This is the concrete
justification for the reactive interface (not just parity).

**NoOp defaults:** `NoOpOversightGateService` logs WARN at startup via
`@Observes StartupEvent` — fires only when the NoOp is the active CDI bean, making
misconfigured deployments observable. `NoOpReactiveOversightGateService` is silent (one
WARN is sufficient). Both are `@DefaultBean @ApplicationScoped` in engine runtime.

**CaseChannelNames removal:** `CaseChannelNames` duplicated `CaseChannel` static methods
already in engine-api. Deleted; callers migrated to `CaseChannel.parseCaseId()`,
`CaseChannel.oversightChannelName()`, `CaseChannel.channelName(caseId, "work")`.
`workChannelName()` had no production callers (confirmed with `ide_find_references`).

**CI fix:** engine#530 added `String taskType` at position 3 of the `ProvisionContext`
record. openclaw's 6-arg test constructors compiled against the cached snapshot but failed
when CI picked up the new snapshot. Fixed by adding `null` at position 3; cherry-picked
to main separately from the feature branch.
