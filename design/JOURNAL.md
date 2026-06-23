# Design Journal — issue-46-plannedaction-import-fix

### 2026-06-23 · §10

Coded against local HEAD of casehub-engine-api. PlannedAction remains in
io.casehub.api.spi as a 5-field record (workerId, caseId, description, actionType,
context). ActionRiskClassifier.classify() takes one PlannedAction arg with identity
fields populated by the caller. OversightGateService constructs PlannedAction with
agentId/caseId inline before passing to classifiers.

The published SNAPSHOT had an interim state that removed PlannedAction from engine-api
(engine #543 spec, not yet implemented locally). Filed engine#565 to re-publish from
local main HEAD. When #543 ships (ClassificationContext, worker-api migration),
openclaw will need to adapt again.
