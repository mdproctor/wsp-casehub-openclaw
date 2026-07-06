# Design Journal — issue-58-demo-ui

## §1 Technology Pivot: casehub-pages DSL → Lit + blocks-ui (2026-07-06)

Original design (2026-06-30) used casehub-pages TypeScript DSL (`page()`, `table()`, `metric()`) with WebSocket wire messages. After discovering blocks-ui — the emerging shared UI layer for CaseHub — pivoted to Lit Web Components with blocks-ui design tokens, SSEManager, and event system.

**Why:** blocks-ui establishes the visual language for all CaseHub apps (work-item-inbox, queue-board). Building with a different design system would create visual inconsistency and double the migration work when promoting components.

**What changed:** WebSocket → SSE, WireMessage JSON → CaseExecutionEvent sealed interface, casehub-pages DSL → Lit @customElement, `--pages-*` tokens → `--blocks-*` OKLCH tokens.

**What stayed:** All design decisions from brainstorming (three scenarios, trading flagship, split view, gate modal, dark mode), backend data model (ScenarioMetadataProvider, ScenarioStateStore, ScenarioObserver), module boundaries (CDI beans in casehub/, JAX-RS in app/).

## §2 "Scenarios" Are CaseDefinitions (2026-07-06)

Explored whether "scenarios" should be a new foundation concept. Concluded: Scenario == CaseDefinition, which already exists in casehub-engine-api with full type hierarchy (namespace, workers, capabilities, bindings, milestones, goals, routing strategies). No new abstraction needed. `ScenarioMetadataProvider` is a demo convenience — a lightweight `ScenarioDef` record carrying only what the UI needs, mapped to engine CaseDefinitions by caseId.

## §3 Shared Component Strategy (2026-07-06)

Three components designed as blocks-ui promotion candidates: `case-worker-pipeline`, `channel-feed`, `gate-approval-modal`. Built with Lit extension points (render callbacks, slots) so domain-specific rendering is pluggable. OpenClaw provides `demo-launcher` and `app-shell` (openclaw-specific). Future: domain accelerators (FSI, clinical) provide pre-built renderers for common verticals.
