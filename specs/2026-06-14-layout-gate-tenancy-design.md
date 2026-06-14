# Design: Layout extraction, gate crash recovery, channel context tenancyId-free query

**Branch:** `issue-32-layout-gate-tenancy`
**Covers:** #32, #34, #33
**Date:** 2026-06-14

---

## Issue #32 — `OpenClawNormativeLayout` extraction

### Problem

`OpenClawCaseChannelProvider` and `ReactiveOpenClawCaseChannelProvider` each define identical
`private record ChannelSpec(String description, String allowedTypes, String deniedTypes)` and
`private static final Map<String, ChannelSpec> LAYOUT` constants. Both carry a comment
pointing at `PLATFORM.md §Agent Communication Mesh` as the source of truth.

Two additional defects:
- `allowedTypes`/`deniedTypes` are stored as `String` and parsed to `Set<MessageType>` at every
  `openChannel()` call site — duplicated parsing in both providers.
- No compile-time or test-time guard detects divergence between the two `LAYOUT` maps.

### Design

New package-private `OpenClawNormativeLayout` in `io.casehub.openclaw.casehub`:

```java
final class OpenClawNormativeLayout {
    record ChannelSpec(
        String description,
        Set<MessageType> allowedTypes,   // null = unrestricted
        Set<MessageType> deniedTypes     // null = unrestricted
    ) {}

    static final Map<String, ChannelSpec> LAYOUT = Map.of(
        "work",     new ChannelSpec("Primary coordination — all obligation-carrying message types", null, null),
        "observe",  new ChannelSpec("Telemetry — EVENT only, no obligations created", Set.of(MessageType.EVENT), null),
        "oversight",new ChannelSpec("Human governance — agent actions pending human approval", null, Set.of(MessageType.EVENT))
    );

    private OpenClawNormativeLayout() {}
}
```

`ChannelSpec` holds `Set<MessageType>` directly. Both providers:
- Delete their local `ChannelSpec` record and `LAYOUT` field
- Replace with `OpenClawNormativeLayout.LAYOUT` and `OpenClawNormativeLayout.ChannelSpec`
- Delete all string-to-Set parsing code at call sites

### Test

`OpenClawNormativeLayoutTest`:
- Asserts `LAYOUT` contains exactly `{"work", "observe", "oversight"}`
- Asserts both providers call `openChannel()` only with purposes present in `LAYOUT`
  (divergence guard — no new purpose can be added to one provider without updating `LAYOUT`)

---

## Issue #34 — Gate crash recovery for pre-#29 persisted gates

### Problem

`OversightGateService.fulfill()` reads `GateContext.tenancyId` from the serialized gate
COMMAND content. Gates opened before #29 have no `tenancyId` key → `parseGateContent()`
returns `Optional.of(GateContext(oci, wci, cmi, null))` → `tenancyId = null`.

`gateDispatcher.dispatch(..., null)` resolves tenancyId from `CurrentPrincipal`, which in
webhook context is `MockCurrentPrincipal` → `DEFAULT_TENANT_ID`. For non-default tenants,
all dispatches in `fulfill()` (oversight RESPONSE/DECLINE + work channel DONE) write to the
wrong tenant's message store. The commitment never closes for the affected tenant.

### Recovery design

`CrossTenantChannelStore` already exists in the Qhorus jar and already exposes
`findById(UUID) → Optional<Channel>`. `Channel.tenancyId` is non-null (enforced at the DB
level). `fulfill()` already has `UUID oversightChannelId = gateCmd.channelId`.

Add `@CrossTenant CrossTenantChannelStore crossTenantChannelStore` to `OversightGateService`
constructor (consistent with the existing `@CrossTenant CrossTenantMessageStore` injection).

In `fulfill()`, after:
```java
String tenancyId = gateContext.map(GateContext::tenancyId).orElse(null);
```
insert:
```java
if (tenancyId == null) {
    tenancyId = crossTenantChannelStore.findById(oversightChannelId)
        .map(ch -> ch.tenancyId)
        .orElse(null);
    if (tenancyId != null)
        log.infof("fulfill(): recovered tenancyId from channel for pre-#29 gate %s", gateId);
    else
        log.warnf("fulfill(): cannot recover tenancyId for gate %s — oversight channel not found", gateId);
}
```

Dispatch proceeds with the recovered tenancyId. No separate upgrade note — the migration
path handles pre-#29 gates silently and automatically.

### Test

`OversightGateServiceTest`:
- New test: gate COMMAND message with content missing `tenancyId` key → after recovery,
  `crossTenantChannelStore.findById(oversightChannelId)` called → tenancyId used for dispatch
- New test: gate COMMAND with no `tenancyId` AND channel not found → `fulfill()` logs warn,
  dispatch proceeds with null (existing behavior, now explicitly tested)

---

## Issue #33 — ChannelContextWindow tenancyId-free query

### Problem

`ChannelContextWindowResource` reads `currentPrincipal.tenancyId()` to scope
`service.query(agentId, tenancyId, since)`. In plugin/SDK context (no casehub principal),
`CurrentPrincipal` resolves to `MockCurrentPrincipal` → `DEFAULT_TENANT_ID`. For non-default
tenants, `agentToCase.get(new AgentKey(agentId, DEFAULT_TENANT_ID))` returns null →
`WindowContent.noAssociation()` → no context injected.

### Root cause

The resource shouldn't need a principal. The service already knows each agent's tenancyId —
`bindAgent(agentId, tenancyId, caseId)` registers it at provision time. The tenancyId is
derivable from `agentId` without the caller supplying it.

### Design

**`ChannelContextWindowService`** — add internal reverse lookup:

```java
private final ConcurrentHashMap<String, String> agentIdToTenancyId = new ConcurrentHashMap<>();
```

- `bindAgent(agentId, tenancyId, caseId)`: also `agentIdToTenancyId.put(agentId, tenancyId)`
- `unbindAgent(agentId, tenancyId)`: also `agentIdToTenancyId.remove(agentId)`
- Remove `query(agentId, tenancyId, since)` from public API
- Replace with `query(agentId, since)`:
  ```java
  public WindowContent query(String agentId, long since) {
      String tenancyId = agentIdToTenancyId.get(agentId);
      if (tenancyId == null) return WindowContent.noAssociation();
      UUID caseId = agentToCase.get(new AgentKey(agentId, tenancyId));
      if (caseId == null) return WindowContent.noAssociation();
      // ... existing buffer aggregation logic unchanged ...
  }
  ```

`AgentKey` isolation is preserved: the internal `agentToCase` lookup still uses
`AgentKey(agentId, tenancyId)` — multi-tenant isolation is maintained. Only the caller
no longer needs to supply the tenancyId; the service supplies it from its own map.

**`ChannelContextWindowResource`**:
- Remove `@Inject CurrentPrincipal`
- Change `service.query(agentId, currentPrincipal.tenancyId(), since)` → `service.query(agentId, since)`

**Plugin/SDK**: no changes. Both already pass `agentId` correctly and fail open on error.

**Auth retrofit**: this endpoint stays unauthenticated and principal-free permanently.
OIDC auth is not needed here — `agentId` is the effective scoping key. The window is on
the intelligence path (best-effort, not correctness), so the security tradeoff is sound.

### Test

`ChannelContextWindowServiceTest`:
- After `bindAgent("agent-1", "tenant-A", caseId)` + `bindChannel(caseId, channelId)` +
  `add(event for channelId)` → `query("agent-1", 0)` returns messages
- After `bindAgent("agent-1", "tenant-A", ...)` + `bindAgent("agent-2", "tenant-B", ...)`:
  `query("agent-1", 0)` does not see tenant-B's content (isolation via `AgentKey`)
- `query("unknown-agent", 0)` returns `noAssociation()` (no `agentIdToTenancyId` entry)

`ChannelContextWindowResourceTest`:
- `CurrentPrincipal` is no longer injected into the resource — verify no principal-dependent
  behavior in the resource layer

---

## Platform coherence

- All changes are fully scoped to `casehub-openclaw`. No cross-repo API changes.
- `#32`: `OpenClawNormativeLayout` is package-private — no risk of consumers depending on it.
  The Known Placement Violation for `CaseChannelLayout` (parent#93, Claudony) is unchanged.
- `#34`: `CrossTenantChannelStore` injection follows existing pattern in the same class.
- `#33`: `ChannelContextWindowService` query signature change is an internal-only break
  (service lives in `core/`, the only callers are `ChannelContextWindowResource` in `app/`
  and tests). Breaking callers is intentional — it forces them to the correct API.
  No callers in peer repos.
