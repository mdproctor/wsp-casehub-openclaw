# Design Journal — issue-46-plannedaction-import-fix

### 2026-06-23 · §10

casehub-worker-api added as an explicit dependency of the casehub module.
PlannedAction moved from io.casehub.api.spi (engine-api) to io.casehub.worker.api
(worker-api), reflecting the platform's separation of orchestration contracts
from worker execution contracts. ActionRiskClassifier.classify() now takes a
separate ClassificationContext carrying identity (workerId, caseId, tenancyId),
decoupling action description from routing context. OversightGateService builds
ClassificationContext inline from its method parameters.
