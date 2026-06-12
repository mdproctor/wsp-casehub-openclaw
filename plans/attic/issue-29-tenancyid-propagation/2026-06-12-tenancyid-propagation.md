# tenancyId Propagation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Propagate `tenancyId` through all casehub-openclaw components so that multi-tenant deployments correctly scope agent registry entries, context window lookups, Qhorus message dispatches, and delivery webhook routing.

**Architecture:** Tenancy flows from `CurrentPrincipal` at provision time into the `OpenClawAgentRegistry` (caseId → tenancyId) and `ChannelContextWindowService` (composite AgentKey). Delivery webhook paths recover tenancyId without a principal: the channel delivery endpoint uses `CrossTenantChannelStore.findById()`, and gate fulfillment reads tenancyId from `GateContext` (persisted in Qhorus COMMAND content) via `CrossTenantMessageStore.scan()`. No qhorus changes required — all cross-tenant infrastructure shipped in qhorus#260.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JUnit 5, Mockito, AssertJ. Maven test commands use `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test`. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`.

---

## File Map

**New files:**
- `core/src/main/java/io/casehub/openclaw/context/AgentKey.java` — package-private record keying the context window map

**Modified (main):**
- `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`
- `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`
- `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java`
- `app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java`

**Modified (tests):**
- `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`
- `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`
- `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`
- `app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java`
- `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java`

**New tests:**
- `app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java`

---

## Task 1: `AgentKey` + `ChannelContextWindowService` signature change

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/context/AgentKey.java`
- Modify: `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`
- Test: `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`

> After this task `core/` compiles and its tests pass. `casehub/` and `app/` will have compilation errors fixed in Tasks 3–8.

- [ ] **Step 1: Write the failing tests**

Add these methods to `ChannelContextWindowServiceTest`. The existing `bindAgent("agent-1", caseId)` calls (2-arg) will be updated in this same step alongside the new tests:

```java
// ── tenancyId isolation ───────────────────────────────────────────────────

@Test
void bindAgent_sameAgentId_differentTenants_independentWindows() {
    UUID caseA = UUID.randomUUID(), caseB = UUID.randomUUID();
    UUID channelA = UUID.randomUUID(), channelB = UUID.randomUUID();
    service.bindAgent("bot", "tenant-A", caseA);
    service.bindAgent("bot", "tenant-B", caseB);
    service.bindChannel(caseA, channelA);
    service.bindChannel(caseB, channelB);
    service.add(event(channelA, "case-x/work", MessageType.STATUS));

    assertThat(service.query("bot", "tenant-A", 0L).messages()).hasSize(1);
    assertThat(service.query("bot", "tenant-B", 0L).messages()).isEmpty();
}

@Test
void query_wrongTenant_returnsNoAssociation() {
    UUID caseId = UUID.randomUUID();
    service.bindAgent("bot", "tenant-A", caseId);
    assertThat(service.query("bot", "tenant-B", 0L).agentHasAssociation()).isFalse();
}

@Test
void unbindAgent_oneTenant_doesNotAffectOther() {
    UUID caseA = UUID.randomUUID(), caseB = UUID.randomUUID();
    service.bindAgent("bot", "tenant-A", caseA);
    service.bindAgent("bot", "tenant-B", caseB);
    service.unbindAgent("bot", "tenant-A");
    assertThat(service.query("bot", "tenant-B", 0L).agentHasAssociation()).isTrue();
}

@Test
void unbindAgent_nullTenancyId_logsAndNoOps() {
    UUID caseId = UUID.randomUUID();
    service.bindAgent("bot", "tenant-A", caseId);
    assertThatCode(() -> service.unbindAgent("bot", null)).doesNotThrowAnyException();
    // tenant-A entry untouched
    assertThat(service.query("bot", "tenant-A", 0L).agentHasAssociation()).isTrue();
}
```

Also update all existing tests that call the old 2-arg signatures. Find every `bindAgent(`, `unbindAgent(`, `query(` call and add the tenancyId argument:

```java
// Old (line 60 area):  service.bindAgent("agent-1", caseId);
// New:
service.bindAgent("agent-1", "test-tenant", caseId);

// Old: service.query("unknown-agent", 0L)
// New:
service.query("unknown-agent", "test-tenant", 0L)

// Old: service.unbindAgent("agent-1");
// New:
service.unbindAgent("agent-1", "test-tenant");
```

Apply this pattern to every occurrence in the test file.

- [ ] **Step 2: Create `AgentKey.java`**

```java
package io.casehub.openclaw.context;

record AgentKey(String agentId, String tenancyId) {}
```

- [ ] **Step 3: Update `ChannelContextWindowService.java`**

Change the `agentToCase` map declaration and the three method signatures. Only these lines change — everything else stays:

```java
// Change field:
private final ConcurrentHashMap<AgentKey, UUID> agentToCase = new ConcurrentHashMap<>();

// Change bindAgent:
public void bindAgent(String agentId, String tenancyId, UUID caseId) {
    agentToCase.put(new AgentKey(agentId, tenancyId), caseId);
}

// Change unbindAgent:
public void unbindAgent(String agentId, String tenancyId) {
    if (tenancyId == null) {
        log.warnf("unbindAgent called with null tenancyId for agentId=%s — agent was never provisioned; no-op", agentId);
        return;
    }
    agentToCase.remove(new AgentKey(agentId, tenancyId));
}

// Change query:
public WindowContent query(String agentId, String tenancyId, long since) {
    UUID caseId = agentToCase.get(new AgentKey(agentId, tenancyId));
    if (caseId == null) return WindowContent.noAssociation();
    // rest of the method body unchanged
    ...
}
```

- [ ] **Step 4: Run core tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -Dtest=ChannelContextWindowServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS (casehub/ and app/ have compile errors — ignore for now, fixed in later tasks).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add core/
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): AgentKey composite key for ChannelContextWindowService — Refs #29"
```

---

## Task 2: `OpenClawAgentRegistry` — add `caseToTenancy`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java`

> After this task the registry compiles. Callers that pass 3 args to `register()` break — fixed in Task 3.

- [ ] **Step 1: Write failing tests**

Add to `OpenClawAgentRegistryTest`:

```java
@Test
void register_storesTenancyIdByCaseId() {
    UUID caseId = UUID.randomUUID();
    registry.register("agent-1", "tenant-A", caseId, "sk");
    assertThat(registry.findTenancyId(caseId)).contains("tenant-A");
}

@Test
void deregister_removesTenancyEntry() {
    UUID caseId = UUID.randomUUID();
    registry.register("agent-1", "tenant-A", caseId, "sk");
    registry.deregister("agent-1");
    assertThat(registry.findTenancyId(caseId)).isEmpty();
}

@Test
void findTenancyId_unknownCaseId_returnsEmpty() {
    assertThat(registry.findTenancyId(UUID.randomUUID())).isEmpty();
}
```

Also update existing tests that call `registry.register("agent-1", caseId, "sk")` (3-arg) → `registry.register("agent-1", "tenant-A", caseId, "sk")` (4-arg). Find every `register(` call in `OpenClawAgentRegistryTest` and `OpenClawWorkerStatusListenerTest` and add the tenancyId argument.

- [ ] **Step 2: Update `OpenClawAgentRegistry.java`**

```java
// Add field after existing maps:
private final ConcurrentHashMap<UUID, String> caseToTenancy = new ConcurrentHashMap<>();

// Update register():
public void register(String agentId, String tenancyId, UUID caseId, String sessionKey) {
    String existingAgent = caseToAgent.get(caseId);
    if (existingAgent != null && !existingAgent.equals(agentId)) {
        log.warnf("caseId=%s already mapped to agentId=%s; overwriting with agentId=%s. "
                + "Multiple OpenClaw agents per case violates the MVP 1:1 constraint.",
                caseId, existingAgent, agentId);
    }
    agentToCase.put(agentId, caseId);
    caseToAgent.put(caseId, agentId);
    agentToSessionKey.put(agentId, sessionKey);
    caseToTenancy.put(caseId, tenancyId);
}

// Update deregister():
public void deregister(String agentId) {
    UUID caseId = agentToCase.remove(agentId);
    if (caseId != null) {
        caseToAgent.remove(caseId);
        caseToTenancy.remove(caseId);
    }
    agentToSessionKey.remove(agentId);
}

// Add new method:
public Optional<String> findTenancyId(UUID caseId) {
    return Optional.ofNullable(caseToTenancy.get(caseId));
}
```

- [ ] **Step 3: Run registry tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OpenClawAgentRegistryTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawAgentRegistry — caseToTenancy map and findTenancyId() — Refs #29"
```

---

## Task 3: Provisioners + StatusListener — inject `CurrentPrincipal`, fix callers

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`

> This task fixes all compilation errors in casehub/ from Tasks 1–2.

- [ ] **Step 1: Update `OpenClawWorkerProvisionerTest`**

Add `CurrentPrincipal` mock. Update constructor call and verify:

```java
// Add field:
io.casehub.platform.api.identity.CurrentPrincipal mockPrincipal;

// In setup():
mockPrincipal = mock(io.casehub.platform.api.identity.CurrentPrincipal.class);
when(mockPrincipal.tenancyId()).thenReturn("test-tenant");
provisioner = new OpenClawWorkerProvisioner(mockService, registry, config, mockPrincipal);

// Update verify in provision_callsBindAgent_onContextWindowService:
verify(mockService).bindAgent("code-review-agent", "test-tenant", caseId);

// Add new test:
@Test
void provision_storesTenancyIdInRegistry() {
    UUID caseId = UUID.randomUUID();
    provisioner.provision(Set.of("finance"), ctx(caseId));
    assertThat(registry.findTenancyId(caseId)).contains("test-tenant");
}

@Test
void terminate_unbindsAgentWithTenancyId() {
    UUID caseId = UUID.randomUUID();
    provisioner.provision(Set.of("finance"), ctx(caseId));
    provisioner.terminate("finance-agent");
    verify(mockService).unbindAgent("finance-agent", "test-tenant");
}
```

- [ ] **Step 2: Update `OpenClawWorkerProvisioner.java`**

```java
// Add import:
import io.casehub.platform.api.identity.CurrentPrincipal;

// Add field and update constructor:
private final CurrentPrincipal currentPrincipal;

@Inject
public OpenClawWorkerProvisioner(ChannelContextWindowService service,
                                  OpenClawAgentRegistry registry,
                                  OpenClawCasehubConfig config,
                                  CurrentPrincipal currentPrincipal) {
    this.service = service;
    this.registry = registry;
    this.config = config;
    this.currentPrincipal = currentPrincipal;
}

// Update provision():
@Override
public ProvisionResult provision(Set<String> capabilities, ProvisionContext context) {
    String agentId = resolveAgentId(capabilities);
    String sessionKey = config.agents().get(agentId).sessionKey();
    UUID caseId = context.caseId();
    String tenancyId = currentPrincipal.tenancyId();

    registry.register(agentId, tenancyId, caseId, sessionKey);
    service.bindAgent(agentId, tenancyId, caseId);

    log.infof("Provisioned OpenClaw agent: agentId=%s caseId=%s tenancyId=%s capabilities=%s",
            agentId, caseId, tenancyId, capabilities);
    return ProvisionResult.empty();
}

// Update terminate():
@Override
public void terminate(String workerId) {
    Optional<UUID> caseId = registry.findCaseId(workerId);
    String tenancyId = caseId.flatMap(registry::findTenancyId).orElse(null);
    registry.deregister(workerId);
    if (tenancyId != null) {
        service.unbindAgent(workerId, tenancyId);
    }
    log.infof("Terminated OpenClaw agent: agentId=%s", workerId);
}
```

- [ ] **Step 3: Update `ReactiveOpenClawWorkerProvisionerTest`**

Same pattern as Step 1 — add `mockPrincipal`, update constructor call and verify assertions.

```java
mockPrincipal = mock(io.casehub.platform.api.identity.CurrentPrincipal.class);
when(mockPrincipal.tenancyId()).thenReturn("test-tenant");
provisioner = new ReactiveOpenClawWorkerProvisioner(mockService, registry, config, mockPrincipal);

// Update verify for bindAgent:
verify(mockService).bindAgent("code-review-agent", "test-tenant", caseId);
```

- [ ] **Step 4: Update `ReactiveOpenClawWorkerProvisioner.java`**

```java
// Add import + field + constructor arg (same as blocking provisioner).
// In provision():
String tenancyId = currentPrincipal.tenancyId();  // captured on request thread
return Uni.createFrom().item(() -> {
    registry.register(agentId, tenancyId, caseId, sessionKey);
    service.bindAgent(agentId, tenancyId, caseId);
    log.infof("Provisioned OpenClaw agent (reactive): agentId=%s caseId=%s tenancyId=%s capabilities=%s",
            agentId, caseId, tenancyId, capabilities);
    return ProvisionResult.empty();
});

// In terminate():
@Override
public Uni<Void> terminate(String workerId) {
    return Uni.createFrom().<Void>item(() -> {
        Optional<UUID> caseId = registry.findCaseId(workerId);
        String tenancyId = caseId.flatMap(registry::findTenancyId).orElse(null);
        registry.deregister(workerId);
        if (tenancyId != null) {
            service.unbindAgent(workerId, tenancyId);
        }
        log.infof("Terminated OpenClaw agent (reactive): agentId=%s", workerId);
        return null;
    });
}
```

- [ ] **Step 5: Update `OpenClawWorkerStatusListenerTest`**

```java
// In setup(): registry.register("agent-1", caseId, "sk") → register("agent-1", "test-tenant", caseId, "sk")
registry.register("agent-1", "test-tenant", caseId, "sk");

// Update verify:
verify(mockService).unbindAgent("agent-1", "test-tenant");

// Add new test:
@Test
void onWorkerCompleted_readstenancyIdBeforeDeregister() {
    UUID caseId = UUID.randomUUID();
    registry.register("agent-1", "tenant-X", caseId, "sk");
    listener.onWorkerCompleted("agent-1", WorkResult.completed("key", Map.of(), "agent-1", caseId));
    verify(mockService).unbindAgent("agent-1", "tenant-X");
}

@Test
void onWorkerCompleted_unknownAgent_unbindAgentWithNullTenancyId() {
    // Never provisioned — registry has no entry; tenancyId is null; unbindAgent must not throw
    assertThatCode(() ->
        listener.onWorkerCompleted("unknown", WorkResult.completed("key", Map.of(), "unknown", UUID.randomUUID()))
    ).doesNotThrowAnyException();
    verify(mockService).unbindAgent(eq("unknown"), isNull());
}
```

Add `import static org.mockito.ArgumentMatchers.isNull;` and `assertThatCode` import if not present.

- [ ] **Step 6: Update `OpenClawWorkerStatusListener.java`**

```java
@Override
public void onWorkerCompleted(String workerId, WorkResult result) {
    log.infof("OpenClaw agent completed: agentId=%s status=%s", workerId, result.status());
    UUID caseId = registry.findCaseId(workerId).orElse(null);
    String tenancyId = caseId != null ? registry.findTenancyId(caseId).orElse(null) : null;
    registry.deregister(workerId);
    service.unbindAgent(workerId, tenancyId);
    if (caseId != null) {
        service.closeCase(caseId);
    }
}
```

- [ ] **Step 7: Run all casehub/ unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest="OpenClawWorkerProvisionerTest,ReactiveOpenClawWorkerProvisionerTest,OpenClawWorkerStatusListenerTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): tenancyId in provisioners and status listener — Refs #29"
```

---

## Task 4: `GateContext` + `OversightGateDispatcher` — signature and behavior change

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java`

> `OversightGateService` will have a compile error on `gateDispatcher.dispatch()` — fixed in Task 5.

- [ ] **Step 1: Add `tenancyId` to `GateContext.java`**

```java
package io.casehub.openclaw.casehub;

import java.util.UUID;

record GateContext(String originalCommitmentId, UUID workChannelId,
                   long commandMessageId, String tenancyId) {}
```

- [ ] **Step 2: Rewrite `OversightGateDispatcherTest.java`**

Delete the 4 no-context tests (`dispatch_approved_noContext_*`, `dispatch_rejected_noContext_*`). Rewrite all 3 with-context tests. Add 2 replacement no-context tests. Full test file:

```java
package io.casehub.openclaw.casehub;

import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class OversightGateDispatcherTest {

    MessageService messageService;
    OversightGateDispatcher dispatcher;

    UUID oversightChannelId = UUID.randomUUID();
    UUID workChannelId      = UUID.randomUUID();
    UUID gateId             = UUID.randomUUID();
    String originalCommitmentId = UUID.randomUUID().toString();
    long originalCommandMsgId   = 99L;
    String tenancyId = "tenant-A";
    GateContext gateContext = new GateContext(originalCommitmentId, workChannelId,
                                              originalCommandMsgId, tenancyId);

    @BeforeEach
    void setup() {
        messageService = mock(MessageService.class);
        dispatcher = new OversightGateDispatcher(messageService);
        when(messageService.dispatch(any())).thenReturn(dispatchResult(1L));
    }

    // ── absent gateContext (pre-#29 gate / parse error): oversight channel only ──

    @Test
    void dispatch_approved_noContext_sendsOnlyResponseToOversight() {
        dispatcher.dispatch(true, oversightChannelId, 42L, gateId, "approved",
                Optional.empty(), null);

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(1)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getValue();
        assertThat(oversight.channelId()).isEqualTo(oversightChannelId);
        assertThat(oversight.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());
        assertThat(oversight.inReplyTo()).isEqualTo(42L);
        assertThat(oversight.content()).isEqualTo("approved");
        assertThat(oversight.actorType()).isEqualTo(ActorType.AGENT);
    }

    @Test
    void dispatch_rejected_noContext_sendsOnlyDeclineToOversight() {
        dispatcher.dispatch(false, oversightChannelId, 42L, gateId, "rejected",
                Optional.empty(), null);

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(1)).dispatch(captor.capture());
        assertThat(captor.getValue().type()).isEqualTo(MessageType.DECLINE);
        assertThat(captor.getValue().channelId()).isEqualTo(oversightChannelId);
    }

    // ── with gateContext: two dispatches, tenancyId set on all ───────────────

    @Test
    void dispatch_approved_withContext_sendsResponseToOversightAndDoneToWork() {
        dispatcher.dispatch(true, oversightChannelId, 42L, gateId, "approved",
                Optional.of(gateContext), tenancyId);

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getAllValues().get(0);
        assertThat(oversight.channelId()).isEqualTo(oversightChannelId);
        assertThat(oversight.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());
        assertThat(oversight.inReplyTo()).isEqualTo(42L);
        assertThat(oversight.tenancyId()).isEqualTo(tenancyId);

        MessageDispatch work = captor.getAllValues().get(1);
        assertThat(work.channelId()).isEqualTo(workChannelId);
        assertThat(work.type()).isEqualTo(MessageType.DONE);
        assertThat(work.correlationId()).isEqualTo(originalCommitmentId);
        assertThat(work.inReplyTo()).isEqualTo(originalCommandMsgId);
        assertThat(work.tenancyId()).isEqualTo(tenancyId);
    }

    @Test
    void dispatch_rejected_withContext_sendsDeclineToOversightAndDeclineToWork() {
        dispatcher.dispatch(false, oversightChannelId, 42L, gateId, "rejected",
                Optional.of(gateContext), tenancyId);

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        assertThat(captor.getAllValues().get(0).type()).isEqualTo(MessageType.DECLINE);
        assertThat(captor.getAllValues().get(0).tenancyId()).isEqualTo(tenancyId);
        assertThat(captor.getAllValues().get(1).type()).isEqualTo(MessageType.DECLINE);
        assertThat(captor.getAllValues().get(1).correlationId()).isEqualTo(originalCommitmentId);
        assertThat(captor.getAllValues().get(1).tenancyId()).isEqualTo(tenancyId);
    }

    @Test
    void dispatch_approved_withContext_twoDispatchCallsTotal() {
        dispatcher.dispatch(true, oversightChannelId, 42L, gateId, "approved",
                Optional.of(gateContext), tenancyId);
        verify(messageService, times(2)).dispatch(any());
    }

    private DispatchResult dispatchResult(Long messageId) {
        return new DispatchResult(messageId, oversightChannelId, OversightGateService.GATE_SENDER,
                MessageType.RESPONSE, null, null, null, null, null, null, null, 0);
    }
}
```

- [ ] **Step 3: Update `OversightGateDispatcher.java`**

Full replacement — remove `workChannelId` parameter, add `tenancyId`, fix absent-gateContext path:

```java
@Transactional
void dispatch(boolean approved,
              UUID oversightChannelId,
              long commandMessageId,
              UUID gateId,
              String rawOutput,
              Optional<GateContext> gateContext,
              String tenancyId) {
    if (approved) {
        messageService.dispatch(MessageDispatch.builder()
                .channelId(oversightChannelId)
                .sender(OversightGateService.GATE_SENDER)
                .type(MessageType.RESPONSE)
                .content(rawOutput != null ? rawOutput : "approved")
                .correlationId(gateId.toString())
                .inReplyTo(commandMessageId)
                .actorType(ActorType.AGENT)
                .tenancyId(tenancyId)
                .build());
        if (gateContext.isPresent()) {
            GateContext ctx = gateContext.get();
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(ctx.workChannelId())
                    .sender(OversightGateService.GATE_SENDER)
                    .type(MessageType.DONE)
                    .content("Action approved by oversight gate")
                    .correlationId(ctx.originalCommitmentId())
                    .inReplyTo(ctx.commandMessageId())
                    .actorType(ActorType.AGENT)
                    .tenancyId(tenancyId)
                    .build());
        }
        // absent gateContext: work channel dispatch is skipped (see openclaw#34)
    } else {
        messageService.dispatch(MessageDispatch.builder()
                .channelId(oversightChannelId)
                .sender(OversightGateService.GATE_SENDER)
                .type(MessageType.DECLINE)
                .content(rawOutput != null ? rawOutput : "rejected")
                .correlationId(gateId.toString())
                .inReplyTo(commandMessageId)
                .actorType(ActorType.AGENT)
                .tenancyId(tenancyId)
                .build());
        if (gateContext.isPresent()) {
            GateContext ctx = gateContext.get();
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(ctx.workChannelId())
                    .sender(OversightGateService.GATE_SENDER)
                    .type(MessageType.DECLINE)
                    .content("Action rejected by oversight gate")
                    .correlationId(ctx.originalCommitmentId())
                    .inReplyTo(ctx.commandMessageId())
                    .actorType(ActorType.AGENT)
                    .tenancyId(tenancyId)
                    .build());
        }
    }
}
```

- [ ] **Step 4: Run dispatcher tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateDispatcherTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): GateContext + OversightGateDispatcher tenancyId threading — Refs #29"
```

---

## Task 5: `OversightGateService` — `evaluate()`, `openGate()`, `fulfill()`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`

> After this task `casehub/` compiles fully. `CommitmentTools` (app/) and `OpenClawDeliveryResource` (app/) break on updated signatures — fixed in Tasks 6–7.

- [ ] **Step 1: Write the failing tests**

Open `OversightGateServiceTest.java` and add/update:

```java
// Add imports:
import io.casehub.qhorus.runtime.store.CrossTenantMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import static org.mockito.ArgumentMatchers.argThat;

// Add mock field:
@CrossTenant CrossTenantMessageStore crossTenantMessageStore;

// In setUp() — add mock:
crossTenantMessageStore = mock(CrossTenantMessageStore.class);
// Inject into OversightGateService constructor (update to match new 6-arg constructor below)

// Test for evaluate() with tenancyId:
@Test
void evaluate_withTenancyId_dispatchesWithTenancyId() {
    UUID channelId = UUID.randomUUID();
    // use ArgumentCaptor to verify tenancyId on the dispatch
    service.evaluate(channelId, "tenant-A", "agent-1", "output content");
    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().tenancyId()).isEqualTo("tenant-A");
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
}

@Test
void evaluate_nullTenancyId_skipsDispatch() {
    service.evaluate(UUID.randomUUID(), null, "agent-1", "output");
    verify(messageService, never()).dispatch(any());
}

// Test for openGate() with explicit tenancyId parameter:
@Test
void openGate_withTenancyId_serializesTenancyIdInGateContent() {
    // Setup commitment + oversight channel (existing pattern in test)
    // call: service.openGate(agentId, commitmentId, outcome, "tenant-A")
    // parse gate content from dispatched COMMAND message, verify tenancyId key present
    // (look for "tenancyId=tenant-A" in the serialized Properties content)
    ...
}

// Test for fulfill() using CrossTenantMessageStore:
@Test
void fulfill_usesGateContextTenancyId_forDispatch() {
    UUID gateId = UUID.randomUUID();
    UUID oversightChannelId = UUID.randomUUID();
    UUID workChannelId = UUID.randomUUID();
    String tenancyId = "tenant-A";

    // Build Properties-format gate content
    String gateContent = buildGateContent(UUID.randomUUID().toString(),
                                          workChannelId, 5L, tenancyId);
    Message gateCmd = new Message();
    gateCmd.id = 42L;
    gateCmd.channelId = oversightChannelId;
    gateCmd.messageType = MessageType.COMMAND;
    gateCmd.content = gateContent;
    gateCmd.correlationId = gateId.toString();

    when(crossTenantMessageStore.scan(argThat(q ->
            gateId.toString().equals(q.correlationId())
            && MessageType.COMMAND == q.messageType())))
        .thenReturn(List.of(gateCmd));

    service.fulfill(gateId, "approved");

    verify(gateDispatcher).dispatch(eq(true), eq(oversightChannelId),
            anyLong(), eq(gateId), eq("approved"), any(), eq(tenancyId));
}

@Test
void fulfill_gateCmdNotFound_noDispatch() {
    when(crossTenantMessageStore.scan(any())).thenReturn(List.of());
    service.fulfill(UUID.randomUUID(), "approved");
    verify(gateDispatcher, never()).dispatch(any(), any(), anyLong(), any(), any(), any(), any());
}

// Helper:
private String buildGateContent(String commitmentId, UUID workChannelId,
                                  long commandMessageId, String tenancyId) {
    java.util.Properties props = new java.util.Properties();
    props.setProperty("originalCommitmentId", commitmentId);
    props.setProperty("workChannelId", workChannelId.toString());
    props.setProperty("commandMessageId", String.valueOf(commandMessageId));
    props.setProperty("reason", "risk: test action");
    props.setProperty("tenancyId", tenancyId);
    java.io.StringWriter sw = new java.io.StringWriter();
    try { props.store(sw, null); } catch (Exception e) { throw new RuntimeException(e); }
    return sw.toString();
}
```

Note: look at the existing test file to understand what mocks are already set up before adding — build on the existing structure.

- [ ] **Step 2: Update `OversightGateService.java`**

Add import and new field, update constructor:

```java
import io.casehub.qhorus.runtime.store.CrossTenantMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;
import io.casehub.qhorus.api.annotation.CrossTenant;

// Add field:
private final CrossTenantMessageStore crossTenantMessageStore;

// Update constructor to accept it:
@Inject
public OversightGateService(final ChannelService channelService,
                             final MessageService messageService,
                             final CommitmentStore commitmentStore,
                             final OversightGateDispatcher gateDispatcher,
                             @RiskClassifier final Instance<ActionRiskClassifier> classifiers,
                             @CrossTenant final CrossTenantMessageStore crossTenantMessageStore) {
    this.channelService = channelService;
    this.messageService = messageService;
    this.commitmentStore = commitmentStore;
    this.gateDispatcher = gateDispatcher;
    this.classifiers = classifiers;
    this.crossTenantMessageStore = crossTenantMessageStore;
}
```

Update `evaluate()`:

```java
public void evaluate(final UUID workChannelId, final String tenancyId,
                     final String agentId, final String output) {
    try {
        if (output == null || output.isBlank()) return;
        if (tenancyId == null) {
            log.warnf("evaluate(): null tenancyId for channelId=%s — channel not found; skipping dispatch",
                    workChannelId);
            return;
        }
        messageService.dispatch(MessageDispatch.builder()
                .channelId(workChannelId)
                .sender(agentId)
                .type(MessageType.STATUS)
                .content(output)
                .actorType(ActorType.AGENT)
                .tenancyId(tenancyId)
                .build());
    } catch (Exception e) {
        log.errorf("evaluate() failed to archive webhook output for channel=%s agent=%s: %s",
                workChannelId, agentId, e.getMessage());
    }
}
```

Update `openGate()` signature:

```java
public GateDecision openGate(final String agentId, final String commitmentId,
                              final String outcome, final String tenancyId) {
```

Inside `openGate()`, update the GateContext construction to include tenancyId:

```java
GateContext ctx = new GateContext(commitmentId, workChannelId, commandMessageId, tenancyId);
```

And add `.tenancyId(tenancyId)` to the oversight channel MessageDispatch.

Update `fulfill()` — replace the bootstrap + channel-loading section with CrossTenantMessageStore:

```java
public void fulfill(final UUID gateId, final String rawOutput) {
    try {
        Message gateCmd = crossTenantMessageStore.scan(
                MessageQuery.builder()
                        .correlationId(gateId.toString())
                        .messageType(MessageType.COMMAND)
                        .build())
                .stream().findFirst().orElse(null);
        if (gateCmd == null) {
            log.warnf("fulfill(): no COMMAND message found for gateId=%s — ignoring", gateId);
            return;
        }
        UUID oversightChannelId = gateCmd.channelId;
        long commandMessageId   = gateCmd.id;
        Optional<GateContext> gateContext = parseGateContent(gateCmd.content);
        String tenancyId = gateContext.map(GateContext::tenancyId).orElse(null);

        boolean approved = parseApproval(gateId, rawOutput);
        gateDispatcher.dispatch(approved, oversightChannelId, commandMessageId,
                gateId, rawOutput, gateContext, tenancyId);

        log.infof("Gate %s: gateId=%s", approved ? "approved" : "rejected", gateId);
    } catch (Exception e) {
        log.errorf("OversightGateService.fulfill() failed for gateId=%s: %s", gateId, e.getMessage());
    }
}
```

Also update `serializeGateContent()` to include tenancyId and `parseGateContent()` to read it:

```java
private String serializeGateContent(GateContext ctx, String reason) {
    Properties props = new Properties();
    props.setProperty("originalCommitmentId", ctx.originalCommitmentId());
    props.setProperty("workChannelId", ctx.workChannelId().toString());
    props.setProperty("commandMessageId", String.valueOf(ctx.commandMessageId()));
    props.setProperty("reason", reason != null ? reason : "");
    props.setProperty("tenancyId", ctx.tenancyId() != null ? ctx.tenancyId() : "");
    StringWriter sw = new StringWriter();
    try { props.store(sw, null); } catch (IOException e) {
        throw new IllegalStateException("StringWriter never throws IOException", e);
    }
    return sw.toString();
}

private Optional<GateContext> parseGateContent(String content) {
    if (content == null || content.isBlank()) return Optional.empty();
    try {
        Properties props = new Properties();
        props.load(new StringReader(content));
        String oci = props.getProperty("originalCommitmentId");
        String wci = props.getProperty("workChannelId");
        String cmi = props.getProperty("commandMessageId");
        String tid = props.getProperty("tenancyId");
        if (oci == null || wci == null || cmi == null || tid == null || tid.isBlank()) {
            return Optional.empty();
        }
        return Optional.of(new GateContext(oci, UUID.fromString(wci), Long.parseLong(cmi), tid));
    } catch (Exception e) {
        log.warnf("parseGateContent: failed to parse gate content: %s", e.getMessage());
        return Optional.empty();
    }
}
```

- [ ] **Step 3: Run casehub/ unit tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest="OversightGateServiceTest,OversightGateDispatcherTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OversightGateService — tenancyId in evaluate/openGate/fulfill — Refs #29"
```

---

## Task 6: `CommitmentTools` — inject `CurrentPrincipal`, fix `openGate()` call

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`
- Test: `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`

- [ ] **Step 1: Surgery on `CommitmentToolsTest.java`**

Four changes — all mechanical arity fixes:

```java
// 1. Add field:
io.casehub.platform.api.identity.CurrentPrincipal currentPrincipal;

// 2. In setUp() — add mock and pass to constructor:
currentPrincipal = mock(io.casehub.platform.api.identity.CurrentPrincipal.class);
when(currentPrincipal.tenancyId()).thenReturn("test-tenant");
when(oversightGateService.openGate(any(), any(), any(), any()))
        .thenReturn(new GateDecision.Autonomous());
tools = new CommitmentTools(messageService, commitmentService, commitmentStore,
                             oversightGateService, currentPrincipal);

// 3. Line 234 — add 4th arg:
when(oversightGateService.openGate(agentId, correlationId, "Report sent", any()))
        .thenReturn(new GateDecision.Autonomous());
// Or use: when(oversightGateService.openGate(eq(agentId), eq(correlationId), eq("Report sent"), any()))

// 4. Line 254 — add 4th arg:
when(oversightGateService.openGate(agentId, correlationId, "Delete old files", any()))
        .thenReturn(new GateDecision.GatePending(gateId, "risk: file deletion"));

// 5. Line 278 — add 4th any():
verify(oversightGateService, never()).openGate(any(), any(), any(), any());
```

Note: the default stub in setUp() (originally line 53) is replaced by the new 4-arg stub shown in point 2.

- [ ] **Step 2: Update `CommitmentTools.java`**

```java
// Add import:
import io.casehub.platform.api.identity.CurrentPrincipal;

// Add field and update constructor:
private final CurrentPrincipal currentPrincipal;

@Inject
public CommitmentTools(MessageService messageService,
                       CommitmentService commitmentService,
                       CommitmentStore commitmentStore,
                       OversightGateService oversightGateService,
                       CurrentPrincipal currentPrincipal) {
    this.messageService = messageService;
    this.commitmentService = commitmentService;
    this.commitmentStore = commitmentStore;
    this.oversightGateService = oversightGateService;
    this.currentPrincipal = currentPrincipal;
}

// In channelBacked_done(), update the openGate() call:
String tenancyId = currentPrincipal.tenancyId();
GateDecision gate = oversightGateService.openGate(agentId, correlationId, outcome, tenancyId);
```

- [ ] **Step 3: Run CommitmentTools tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(app): CommitmentTools — inject CurrentPrincipal, pass tenancyId to openGate — Refs #29"
```

---

## Task 7: `OpenClawDeliveryResource` + `ChannelContextWindowResource`

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java`
- Test: `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`
- Test: `app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java`

- [ ] **Step 1: Update `OpenClawDeliveryResourceTest.java`**

Add tests for the new behavior:

```java
// Add to existing test class — confirm tenancyId flows and 200 on unknown channel:

@Test
void deliver_validChannel_passesChannelTenancyIdToEvaluate() {
    // Setup channel with tenancyId in CrossTenantChannelStore (mock or in-memory)
    // POST to /openclaw/delivery/channel/{channelId}
    // verify oversightGateService.evaluate(channelId, "tenant-A", ...) was called
}

@Test
void deliver_unknownChannel_returns200AndSkipsEvaluate() {
    // CrossTenantChannelStore returns empty
    // POST to /openclaw/delivery/channel/{unknownId}
    // assert response 200, evaluate called with null tenancyId, no dispatch
}
```

The exact test structure depends on whether this uses `@QuarkusTest` or mock. Follow the existing test pattern in `OpenClawDeliveryResourceTest.java`.

- [ ] **Step 2: Update `OpenClawDeliveryResource.java`**

Replace `ChannelService` injection with `@CrossTenant CrossTenantChannelStore`:

```java
import io.casehub.qhorus.api.annotation.CrossTenant;
import io.casehub.qhorus.runtime.store.CrossTenantChannelStore;
import io.casehub.qhorus.runtime.channel.Channel;

// Remove: @Inject ChannelService channelService;
// Add:
@Inject
@CrossTenant
CrossTenantChannelStore crossTenantChannelStore;

// Update deliver():
@POST
@Path("/{channelId}")
public Response deliver(@PathParam("channelId") String channelIdStr,
                         OpenClawDeliveryPayload payload) {
    UUID channelId;
    try {
        channelId = UUID.fromString(channelIdStr);
    } catch (IllegalArgumentException e) {
        return Response.status(400).build();
    }

    Optional<Channel> channel = crossTenantChannelStore.findById(channelId);
    if (channel.isEmpty()) {
        log.warnf("Delivery received for unknown channelId=%s — tenancyId unresolvable; skipping dispatch",
                channelId);
    }
    String tenancyId = channel.map(ch -> ch.tenancyId).orElse(null);

    String agentId = payload != null && payload.agentId() != null ? payload.agentId() : "openclaw-agent";
    String output  = payload != null && payload.output()  != null ? payload.output()  : "";

    oversightGateService.evaluate(channelId, tenancyId, agentId, output);
    return Response.ok().build();   // Always 200 — OpenClaw must not retry
}
```

- [ ] **Step 3: Update `ChannelContextWindowResource.java`**

```java
import io.casehub.platform.api.identity.CurrentPrincipal;

// Add field:
@Inject
CurrentPrincipal currentPrincipal;

// Update query():
@GET
@Path("/{agentId}")
public WindowContent query(
        @PathParam("agentId") String agentId,
        @QueryParam("since") @DefaultValue("0") long since) {
    return service.query(agentId, currentPrincipal.tenancyId(), since);
}
```

- [ ] **Step 4: Update `ChannelContextWindowResourceTest.java`**

Add or update tests to verify tenancyId scoping via mock or `FixedCurrentPrincipal`. Consult the existing test to understand its setup, then add:

```java
// Verify that query() passes tenancyId from CurrentPrincipal to service:
// If @QuarkusTest: use FixedCurrentPrincipal.setTenancyId("tenant-A") then assert scoping.
// If unit test: inject mock CurrentPrincipal, verify service.query("bot", "tenant-A", 0) called.
```

- [ ] **Step 5: Build `app/` module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compile PASS, tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(app): delivery resource + context window resource tenancyId — Refs #29"
```

---

## Task 8: `OversightGateDispatcherCdiTest` surgery + cross-tenant isolation test

**Files:**
- Modify: `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java`
- Create: `app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java`

- [ ] **Step 1: Rewrite `OversightGateDispatcherCdiTest.java`**

The test's purpose is unchanged: verify `@Transactional` atomicity (both dispatches commit or neither does) and that `fulfill()` is fail-open. The mechanism changes because `fulfill()` now uses `CrossTenantMessageStore.scan()` instead of `messageService.findAllByCorrelationId()`.

Key changes:
- Remove the `doAnswer` bridge for `messageService.findAllByCorrelationId()` (no longer needed)
- Write a Properties-format COMMAND (with full gate context including tenancyId) to `InMemoryMessageStore`
- Use `InMemoryCrossTenantMessageStore` which reads from the same backing store
- Assert 1 dispatch (RESPONSE) succeeded and work channel has no DONE (second dispatch threw)

```java
// Replace the entire test body:

@Test
void second_dispatch_failure_leaves_work_channel_no_done_and_fulfill_is_fail_open() {
    // 1. Build gate content with tenancyId and workChannelId
    String commitmentId = UUID.randomUUID().toString();
    java.util.Properties props = new java.util.Properties();
    props.setProperty("originalCommitmentId", commitmentId);
    props.setProperty("workChannelId", workChannel.id.toString());
    props.setProperty("commandMessageId", "99");
    props.setProperty("reason", "risk: test action");
    props.setProperty("tenancyId", "default");
    java.io.StringWriter sw = new java.io.StringWriter();
    try { props.store(sw, null); } catch (Exception e) { throw new RuntimeException(e); }

    // 2. Dispatch COMMAND with Properties-format gate content to oversight channel
    messageService.dispatch(MessageDispatch.builder()
            .channelId(oversightChannel.id)
            .sender("openclaw-gate")
            .type(MessageType.COMMAND)
            .content(sw.toString())
            .correlationId(gateId.toString())
            .actorType(ActorType.AGENT)
            .build());

    clearInvocations(messageService);

    // 3. Stub: 1st dispatch (RESPONSE to oversight) succeeds, 2nd (DONE to work) throws
    doCallRealMethod()
            .doThrow(new RuntimeException("simulated second-dispatch failure"))
            .when(messageService).dispatch(any());

    // 4. fulfill() — should not throw (fail-open)
    oversightGateService.fulfill(gateId, "approved");

    // 5. Work channel has no DONE from GATE_SENDER
    List<Message> workDone = messageStore.scan(MessageQuery.builder().build()).stream()
            .filter(m -> workChannel.id.equals(m.channelId)
                      && MessageType.DONE == m.messageType
                      && "openclaw-gate".equals(m.sender))
            .toList();
    assertThat(workDone).isEmpty();

    // 6. Exactly two fulfill-path dispatch calls attempted (RESPONSE + DONE attempted)
    verify(messageService, times(2)).dispatch(any());
}
```

Remove the old bridge `doAnswer().when(messageService).findAllByCorrelationId(any())` from `setUp()` entirely.

- [ ] **Step 2: Create `CrossTenantContextIsolationTest.java`**

```java
package io.casehub.openclaw.app;

import java.util.Set;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.openclaw.casehub.OpenClawAgentRegistry;
import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.platform.testing.FixedCurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Verifies that two tenants sharing the same agentId have independent
 * ChannelContextWindow entries — no cross-tenant leakage.
 */
@QuarkusTest
class CrossTenantContextIsolationTest {

    @Inject
    ChannelContextWindowService contextWindowService;

    @Inject
    FixedCurrentPrincipal principal;

    @Test
    void sameAgentId_differentTenants_independentContextWindows() {
        UUID caseA = UUID.randomUUID();
        UUID caseB = UUID.randomUUID();
        UUID channelA = UUID.randomUUID();

        // Bind agent "bot" for tenant-A
        contextWindowService.bindAgent("bot", "tenant-A", caseA);
        contextWindowService.bindChannel(caseA, channelA);

        // Bind agent "bot" for tenant-B — separate context window
        contextWindowService.bindAgent("bot", "tenant-B", caseB);

        // Query as tenant-A: association exists
        assertThat(contextWindowService.query("bot", "tenant-A", 0L).agentHasAssociation()).isTrue();

        // Query as tenant-B: association exists but different caseId, no channels yet → associated but empty
        assertThat(contextWindowService.query("bot", "tenant-B", 0L).agentHasAssociation()).isTrue();

        // Unbind tenant-A: tenant-B unaffected
        contextWindowService.unbindAgent("bot", "tenant-A");
        assertThat(contextWindowService.query("bot", "tenant-A", 0L).agentHasAssociation()).isFalse();
        assertThat(contextWindowService.query("bot", "tenant-B", 0L).agentHasAssociation()).isTrue();
    }
}
```

- [ ] **Step 3: Run app/ tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "test(app): rewrite CdiTest + cross-tenant isolation test — Refs #29"
```

---

## Task 9: Full build verification

- [ ] **Step 1: Full install**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS, zero failures across all modules.

- [ ] **Step 2: Verify git log**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw log --oneline origin/main..HEAD
```

Expected: 8 commits (Tasks 1–8 commits above).

- [ ] **Step 3: Commit if any final fixes needed**

If the build revealed any compilation issue not caught during individual module tests, fix and commit with `fix: ...` message referencing #29.

- [ ] **Step 4: Push**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw push --set-upstream origin issue-29-tenancyid-propagation
```

---

## Self-Review

**Spec coverage:**

| Spec section | Task |
|---|---|
| §1 `OpenClawAgentRegistry` caseToTenancy | Task 2 |
| §2 `ChannelContextWindowService` AgentKey | Task 1 |
| §3 `OpenClawWorkerProvisioner` inject CurrentPrincipal | Task 3 |
| §3 `ReactiveOpenClawWorkerProvisioner` pre-lambda capture | Task 3 |
| §4 `OpenClawWorkerStatusListener` read before deregister | Task 3 |
| §5 `OpenClawChannelBackend` webhook URL unchanged | No change needed — confirmed |
| §6 `OpenClawDeliveryResource` CrossTenantChannelStore, fix 404 | Task 7 |
| §7 `OversightGateService.evaluate()` tenancyId param | Task 5 |
| §8 `GateContext` + `openGate()` tenancyId param, CommitmentTools | Tasks 4 + 6 |
| §9 `OversightGateService.fulfill()` CrossTenantMessageStore.scan | Task 5 |
| §10 `OversightGateDispatcher` remove workChannelId, add tenancyId | Task 4 |
| §11 `ChannelContextWindowResource` inject CurrentPrincipal | Task 7 |
| OversightGateDispatcherTest surgery (4 delete, 3 rewrite, 2 new) | Task 4 |
| CommitmentToolsTest surgery (5 locations) | Task 6 |
| OversightGateDispatcherCdiTest rewrite | Task 8 |
| Cross-tenant isolation test | Task 8 |

All spec sections covered.
