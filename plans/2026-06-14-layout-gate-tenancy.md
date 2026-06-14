# Layout Extraction, Gate Recovery, and Channel Context TenancyId-Free Query

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close issues #32 (extract shared LAYOUT constant), #34 (gate crash recovery for pre-#29 gates), and #33 (remove AgentKey from ChannelContextWindowService so the channel context endpoint works without a principal).

**Architecture:** Three independent changes sequenced by module dependency: #32 is a pure refactor in `casehub/`; #34 adds a new injection and recovery branch in `casehub/`; #33 simplifies `core/` first (service API change + AgentKey deletion), then updates all callers in `casehub/`, then fixes `app/`.

**Tech Stack:** Java 21/Quarkus 3.32.2, Maven multi-module (`core/`, `casehub/`, `app/`), JUnit 5, Mockito, AssertJ. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode`.

---

## File Map

**Created:**
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawNormativeLayout.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawNormativeLayoutTest.java`

**Deleted:**
- `core/src/main/java/io/casehub/openclaw/context/AgentKey.java`
- `app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java`

**Modified:**
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProvider.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`
- `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`
- `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`
- `app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java`
- `app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java`

---

## Task 1: #32 — Create `OpenClawNormativeLayout` with value-assertion tests

**Files:**
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawNormativeLayoutTest.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawNormativeLayout.java`

- [ ] **Step 1.1: Write the test**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawNormativeLayoutTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.MessageType;

import static org.assertj.core.api.Assertions.assertThat;

class OpenClawNormativeLayoutTest {

    @Test
    void layout_containsExactlyThreePurposes() {
        assertThat(OpenClawNormativeLayout.LAYOUT.keySet())
                .containsExactlyInAnyOrder("work", "observe", "oversight");
    }

    @Test
    void layout_work_isUnrestricted() {
        OpenClawNormativeLayout.ChannelSpec spec = OpenClawNormativeLayout.LAYOUT.get("work");
        assertThat(spec).isNotNull();
        assertThat(spec.allowedTypes()).isNull();
        assertThat(spec.deniedTypes()).isNull();
    }

    @Test
    void layout_observe_allowsOnlyEvent() {
        OpenClawNormativeLayout.ChannelSpec spec = OpenClawNormativeLayout.LAYOUT.get("observe");
        assertThat(spec).isNotNull();
        assertThat(spec.allowedTypes()).isEqualTo(Set.of(MessageType.EVENT));
        assertThat(spec.deniedTypes()).isNull();
    }

    @Test
    void layout_oversight_deniesEvent() {
        OpenClawNormativeLayout.ChannelSpec spec = OpenClawNormativeLayout.LAYOUT.get("oversight");
        assertThat(spec).isNotNull();
        assertThat(spec.allowedTypes()).isNull();
        assertThat(spec.deniedTypes()).isEqualTo(Set.of(MessageType.EVENT));
    }
}
```

- [ ] **Step 1.2: Run test — expect FAIL (class doesn't exist yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OpenClawNormativeLayoutTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compilation error — `OpenClawNormativeLayout` cannot be found.

- [ ] **Step 1.3: Create `OpenClawNormativeLayout.java`**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawNormativeLayout.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Map;
import java.util.Set;

import io.casehub.qhorus.api.message.MessageType;

/**
 * Normative 3-channel layout for CaseHub/OpenClaw integration.
 * Source of truth: PLATFORM.md §Agent Communication Mesh.
 * Consolidation of NormativeChannelLayout: parent#93.
 */
final class OpenClawNormativeLayout {

    /**
     * @param allowedTypes message types permitted on this channel; null = unrestricted
     * @param deniedTypes  message types blocked on this channel; null = unrestricted
     */
    record ChannelSpec(
            String description,
            Set<MessageType> allowedTypes,
            Set<MessageType> deniedTypes
    ) {}

    static final Map<String, ChannelSpec> LAYOUT = Map.of(
            "work",     new ChannelSpec(
                    "Primary coordination — all obligation-carrying message types", null, null),
            "observe",  new ChannelSpec(
                    "Telemetry — EVENT only, no obligations created", Set.of(MessageType.EVENT), null),
            "oversight",new ChannelSpec(
                    "Human governance — agent actions pending human approval", null, Set.of(MessageType.EVENT))
    );

    private OpenClawNormativeLayout() {}
}
```

- [ ] **Step 1.4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OpenClawNormativeLayoutTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS, 4 tests pass.

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawNormativeLayout.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawNormativeLayoutTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "feat(casehub): add OpenClawNormativeLayout — single source of truth for 3-channel layout — Refs #32"
```

---

## Task 2: #32 — Update `OpenClawCaseChannelProvider`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`

- [ ] **Step 2.1: Replace the local `ChannelSpec` record and `LAYOUT` field**

In `OpenClawCaseChannelProvider.java`, delete lines 44–54 (the `ChannelSpec` record and `LAYOUT` field):

```java
// DELETE these lines:
private record ChannelSpec(String description, String allowedTypes, String deniedTypes) {}

// Normative channel layout. Source of truth: PLATFORM.md §Agent Communication Mesh.
// oversight uses deniedTypes=EVENT (no telemetry on governance channel; all obligation
// types allowed). observe uses allowedTypes=EVENT (telemetry only, no obligations).
// work is unrestricted. Consolidation of NormativeChannelLayout: parent#93.
private static final Map<String, ChannelSpec> LAYOUT = Map.of(
        "work",     new ChannelSpec("Primary coordination — all obligation-carrying message types", null, null),
        "observe",  new ChannelSpec("Telemetry — EVENT only, no obligations created", "EVENT", null),
        "oversight",new ChannelSpec("Human governance — agent actions pending human approval", null, "EVENT")
);
```

- [ ] **Step 2.2: Update `openChannel()` to use `OpenClawNormativeLayout`**

Replace the call-site parsing block in `openChannel()`. The old code (lines 75–81):

```java
ChannelSpec spec = LAYOUT.get(purpose);
String description = spec != null ? spec.description() : purpose;
// ChannelService.create() requires Set<MessageType> — parse from ChannelSpec strings
java.util.Set<MessageType> allowedSet = spec != null && spec.allowedTypes() != null
        ? java.util.Set.of(MessageType.valueOf(spec.allowedTypes())) : null;
java.util.Set<MessageType> deniedSet = spec != null && spec.deniedTypes() != null
        ? java.util.Set.of(MessageType.valueOf(spec.deniedTypes())) : null;
```

Replace with:

```java
OpenClawNormativeLayout.ChannelSpec spec = OpenClawNormativeLayout.LAYOUT.get(purpose);
String description = spec != null ? spec.description() : purpose;
Set<MessageType> allowedSet = spec != null ? spec.allowedTypes() : null;
Set<MessageType> deniedSet = spec != null ? spec.deniedTypes() : null;
```

Also remove any now-unused `java.util.Map` import if needed (the field `LAYOUT` is gone; the `Map` import may now be unused — check and remove if so).

- [ ] **Step 2.3: Run provider tests — expect PASS (behaviour unchanged)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OpenClawCaseChannelProviderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS.

- [ ] **Step 2.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "refactor(casehub): OpenClawCaseChannelProvider uses OpenClawNormativeLayout — Refs #32"
```

---

## Task 3: #32 — Update `ReactiveOpenClawCaseChannelProvider`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProvider.java`

- [ ] **Step 3.1: Replace the local `ChannelSpec` record and `LAYOUT` field**

Delete the private `ChannelSpec` record (lines 55–56) and the `LAYOUT` field (lines 57–62):

```java
// DELETE these lines:
private record ChannelSpec(String description, String allowedTypes, String deniedTypes) {}

// Normative channel layout — same as OpenClawCaseChannelProvider. Source of truth: PLATFORM.md §Agent Communication Mesh.
private static final Map<String, ChannelSpec> LAYOUT = Map.of(
        "observe",   new ChannelSpec("Telemetry — EVENT only, no obligations created", "EVENT", null),
        "oversight", new ChannelSpec("Human governance — agent actions pending human approval", null, "EVENT"),
        "work",      new ChannelSpec("Primary coordination — all obligation-carrying message types", null, null)
);
```

- [ ] **Step 3.2: Update `openOrCreate()` and `initializeLayout()` to use `OpenClawNormativeLayout`**

In `initializeLayout()`, replace `LAYOUT.keySet()`:

```java
// OLD:
List<String> purposes = LAYOUT.keySet().stream().sorted().toList();
// NEW:
List<String> purposes = OpenClawNormativeLayout.LAYOUT.keySet().stream().sorted().toList();
```

In `openOrCreate()`, replace the `ChannelSpec` lookup and parsing (lines 163–174):

```java
// OLD:
ChannelSpec spec = LAYOUT.get(purpose);
String description = spec != null ? spec.description() : purpose;
// ReactiveChannelService.create() requires Set<MessageType> — parse from ChannelSpec strings
java.util.Set<io.casehub.qhorus.api.message.MessageType> allowedSet =
        spec != null && spec.allowedTypes() != null
        ? java.util.Set.of(io.casehub.qhorus.api.message.MessageType.valueOf(spec.allowedTypes()))
        : null;
java.util.Set<io.casehub.qhorus.api.message.MessageType> deniedSet =
        spec != null && spec.deniedTypes() != null
        ? java.util.Set.of(io.casehub.qhorus.api.message.MessageType.valueOf(spec.deniedTypes()))
        : null;

// NEW:
OpenClawNormativeLayout.ChannelSpec spec = OpenClawNormativeLayout.LAYOUT.get(purpose);
String description = spec != null ? spec.description() : purpose;
Set<MessageType> allowedSet = spec != null ? spec.allowedTypes() : null;
Set<MessageType> deniedSet = spec != null ? spec.deniedTypes() : null;
```

- [ ] **Step 3.3: Run provider tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=ReactiveOpenClawCaseChannelProviderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS.

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProvider.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "refactor(casehub): ReactiveOpenClawCaseChannelProvider uses OpenClawNormativeLayout — Refs #32"
```

---

## Task 4: #34 — Write new `OversightGateServiceTest` tests

**Files:**
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`

- [ ] **Step 4.1: Add import and field for `CrossTenantChannelStore`**

At the top of `OversightGateServiceTest.java`, add import (alongside existing `CrossTenantMessageStore` import):

```java
import io.casehub.qhorus.runtime.store.CrossTenantChannelStore;
```

Add field declaration after `CrossTenantMessageStore crossTenantMessageStore;`:

```java
CrossTenantChannelStore crossTenantChannelStore;
```

- [ ] **Step 4.2: Update `setup()` — add mock and update constructor call**

In `setup()`, after the line that mocks `crossTenantMessageStore`, add:

```java
crossTenantChannelStore = mock(CrossTenantChannelStore.class);
when(crossTenantChannelStore.findById(any())).thenReturn(Optional.empty()); // default: not found
```

Update the constructor call from 6 args to 7 args:

```java
// OLD:
service = new OversightGateService(channelService, messageService, commitmentStore,
        gateDispatcher, classifiers, crossTenantMessageStore);
// NEW:
service = new OversightGateService(channelService, messageService, commitmentStore,
        gateDispatcher, classifiers, crossTenantMessageStore, crossTenantChannelStore);
```

- [ ] **Step 4.3: Run existing fulfill tests to confirm setup change compiles and passes**

The constructor change will fail to compile until the production class is updated (Task 5). Run to see the compilation error now:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: compilation error — `OversightGateService` constructor has 6 parameters.

- [ ] **Step 4.4: Add the two new test methods (below the existing `fulfill_*` tests)**

In `OversightGateServiceTest.java`, add these two tests in the `// ── fulfill() — with gate context` section:

```java
@Test
void fulfill_missingTenancyIdInGateContent_recoversFromChannel() {
    UUID gateId = UUID.randomUUID();
    // Build gate command content WITHOUT tenancyId key
    java.util.Properties props = new java.util.Properties();
    props.setProperty("originalCommitmentId", commitmentId);
    props.setProperty("workChannelId", workChannelId.toString());
    props.setProperty("commandMessageId", String.valueOf(commandMsgId));
    props.setProperty("reason", "test risk");
    // deliberately omit tenancyId
    java.io.StringWriter sw = new java.io.StringWriter();
    try { props.store(sw, null); } catch (Exception e) { throw new RuntimeException(e); }

    Message cmd = new Message();
    cmd.id = 42L;
    cmd.channelId = oversightChannelId;
    cmd.messageType = MessageType.COMMAND;
    cmd.content = sw.toString();
    cmd.correlationId = gateId.toString();
    when(crossTenantMessageStore.scan(any())).thenReturn(List.of(cmd));

    // Recovery: channel lookup returns a channel with tenancyId = "tenant-A"
    Channel oversight = new Channel();
    oversight.id = oversightChannelId;
    oversight.tenancyId = "tenant-A";
    when(crossTenantChannelStore.findById(oversightChannelId)).thenReturn(Optional.of(oversight));

    service.fulfill(gateId, "approved");

    // Recovered tenancyId passed to dispatcher
    verify(crossTenantChannelStore).findById(oversightChannelId);
    verify(gateDispatcher).dispatch(
            eq(true), any(), anyLong(), any(), any(), any(), eq("tenant-A"));
}

@Test
void fulfill_missingTenancyIdAndChannelNotFound_dispatchesWithNullTenancyId() {
    UUID gateId = UUID.randomUUID();
    java.util.Properties props = new java.util.Properties();
    props.setProperty("originalCommitmentId", commitmentId);
    props.setProperty("workChannelId", workChannelId.toString());
    props.setProperty("commandMessageId", String.valueOf(commandMsgId));
    props.setProperty("reason", "test risk");
    java.io.StringWriter sw = new java.io.StringWriter();
    try { props.store(sw, null); } catch (Exception e) { throw new RuntimeException(e); }

    Message cmd = new Message();
    cmd.id = 42L;
    cmd.channelId = oversightChannelId;
    cmd.messageType = MessageType.COMMAND;
    cmd.content = sw.toString();
    cmd.correlationId = gateId.toString();
    when(crossTenantMessageStore.scan(any())).thenReturn(List.of(cmd));
    // crossTenantChannelStore already returns Optional.empty() by default in setup()

    service.fulfill(gateId, "approved");

    verify(crossTenantChannelStore).findById(oversightChannelId);
    // dispatcher still called, with null tenancyId — fail-open behaviour
    verify(gateDispatcher).dispatch(
            eq(true), any(), anyLong(), any(), any(), any(), (String) isNull());
}
```

Note: `crossTenantChannelStore.findById(any())` defaults to `Optional.empty()` from setup — the second test relies on this default.

---

## Task 5: #34 — Update `OversightGateService` with recovery logic

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`

- [ ] **Step 5.1: Add import and field**

Add import (alongside existing `CrossTenantMessageStore` import):

```java
import io.casehub.qhorus.runtime.store.CrossTenantChannelStore;
```

Add field after `crossTenantMessageStore`:

```java
private final CrossTenantChannelStore crossTenantChannelStore;
```

- [ ] **Step 5.2: Update constructor**

Update the `@Inject` constructor to accept `@CrossTenant CrossTenantChannelStore crossTenantChannelStore` as a 7th parameter (after `crossTenantMessageStore`):

```java
@Inject
public OversightGateService(final ChannelService channelService,
                             final MessageService messageService,
                             final CommitmentStore commitmentStore,
                             final OversightGateDispatcher gateDispatcher,
                             @RiskClassifier final Instance<ActionRiskClassifier> classifiers,
                             @CrossTenant final CrossTenantMessageStore crossTenantMessageStore,
                             @CrossTenant final CrossTenantChannelStore crossTenantChannelStore) {
    this.channelService = channelService;
    this.messageService = messageService;
    this.commitmentStore = commitmentStore;
    this.gateDispatcher = gateDispatcher;
    this.classifiers = classifiers;
    this.crossTenantMessageStore = crossTenantMessageStore;
    this.crossTenantChannelStore = crossTenantChannelStore;
}
```

- [ ] **Step 5.3: Add recovery logic in `fulfill()`**

In `fulfill()`, after this existing line:

```java
String tenancyId = gateContext.map(GateContext::tenancyId).orElse(null);
```

Insert:

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

- [ ] **Step 5.4: Run all `OversightGateServiceTest` tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am \
  -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS. All existing fulfill tests pass (they use `buildGateCommand` which includes `tenancyId` in content → recovery path not triggered). Two new tests also pass.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "fix(casehub): recover tenancyId from CrossTenantChannelStore for pre-#29 gates in fulfill() — Refs #34"
```

---

## Task 6: #33 — Update `ChannelContextWindowServiceTest`

**Files:**
- Modify: `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`

This task updates the test file to the new API. It will fail to compile until Task 7 updates the service.

- [ ] **Step 6.1: Delete the four AgentKey-isolation tests**

Delete these four complete test methods from the file:
- `bindAgent_sameAgentId_differentTenants_independentWindows` (lines 332–343)
- `query_wrongTenant_returnsNoAssociation` (lines 346–350)
- `unbindAgent_oneTenant_doesNotAffectOther` (lines 353–359)
- `unbindAgent_nullTenancyId_logsAndNoOps` (lines 362–368)

Also delete the section comment `// ── tenancyId isolation ───────────────────────────────────────────────────` (line 329).

- [ ] **Step 6.2: Update all `bindAgent` calls — drop tenancyId (3-arg → 2-arg)**

Search for `service.bindAgent(` and `restarted.bindAgent(`. Remove the second argument (tenancyId string) from every call. Examples:

```java
// OLD → NEW
service.bindAgent("agent-1", "test-tenant", caseId)  →  service.bindAgent("agent-1", caseId)
restarted.bindAgent("agent-1", "test-tenant", caseId) →  restarted.bindAgent("agent-1", caseId)
```

This affects approximately 15 call sites spread across the test class.

- [ ] **Step 6.3: Update all `unbindAgent` calls — drop tenancyId (2-arg → 1-arg)**

```java
// OLD → NEW
service.unbindAgent("agent-1", "test-tenant")  →  service.unbindAgent("agent-1")
restarted.unbindAgent("agent-1", "test-tenant") → restarted.unbindAgent("agent-1") (if present)
```

In the concurrency test (line 297), the lambda becomes:
```java
executor.submit(() -> { try { for (int i=0;i<50;i++) { service.unbindAgent("agent-1"); service.bindAgent("agent-1",caseId); } } finally { latch.countDown(); } });
```

- [ ] **Step 6.4: Update all `query` calls — drop tenancyId (3-arg → 2-arg)**

```java
// OLD → NEW
service.query("agent-1", "test-tenant", 0L)  →  service.query("agent-1", 0L)
service.query("agent-1", "test-tenant", firstSeq)  →  service.query("agent-1", firstSeq)
service.query("bot", "tenant-A", 0L)  →  service.query("bot", 0L)   (if still present after deletions)
restarted.query("agent-1", "test-tenant", 3L)  →  restarted.query("agent-1", 3L)
service.query("any-agent", "test-tenant", 0L)  →  service.query("any-agent", 0L)
```

- [ ] **Step 6.5: Add the last-write-wins test**

Add this test to the file (e.g., after `closeCase_unknownCase_noOp`):

```java
// ── last-write-wins (AgentKey removed) ───────────────────────────────────────

@Test
void bindAgent_sameAgentId_secondCallOverwritesFirst() {
    UUID caseA = UUID.randomUUID(), caseB = UUID.randomUUID();
    UUID channelA = UUID.randomUUID(), channelB = UUID.randomUUID();
    service.bindAgent("bot", caseA);
    service.bindChannel(caseA, channelA);
    service.add(event(channelA, "case-a/work", MessageType.STATUS)); // seq=1

    service.bindAgent("bot", caseB);  // overwrites — caseA binding gone
    service.bindChannel(caseB, channelB);
    service.add(event(channelB, "case-b/work", MessageType.COMMAND)); // seq=2

    WindowContent result = service.query("bot", 0L);
    assertThat(result.agentHasAssociation()).isTrue();
    // Only caseB's channel content visible — caseA's channel buffer still exists
    // but caseA's agentToCase entry is gone, so it is not reachable via query.
    assertThat(result.messages()).hasSize(1);
    assertThat(result.messages().get(0).messageType()).isEqualTo(MessageType.COMMAND);
}
```

- [ ] **Step 6.6: Verify compilation fails (expected at this point)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test-compile -pl core 2>&1 | tail -10
```

Expected: compilation error because `ChannelContextWindowService` still has the old 3-arg signatures. This is expected — production code updated in Task 7.

---

## Task 7: #33 — Update `ChannelContextWindowService` and delete `AgentKey`

**Files:**
- Modify: `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`
- Delete: `core/src/main/java/io/casehub/openclaw/context/AgentKey.java`

- [ ] **Step 7.1: Change `agentToCase` map type**

In `ChannelContextWindowService.java`, replace:

```java
private final ConcurrentHashMap<AgentKey, UUID> agentToCase = new ConcurrentHashMap<>();
```

With:

```java
private final ConcurrentHashMap<String, UUID> agentToCase = new ConcurrentHashMap<>();
```

- [ ] **Step 7.2: Update `bindAgent` — drop tenancyId, simplify map put**

Replace the existing `bindAgent` method:

```java
public void bindAgent(String agentId, String tenancyId, UUID caseId) {
    agentToCase.put(new AgentKey(agentId, tenancyId), caseId);
}
```

With:

```java
public void bindAgent(String agentId, UUID caseId) {
    agentToCase.put(agentId, caseId);
}
```

- [ ] **Step 7.3: Update `unbindAgent` — drop tenancyId, drop null guard**

Replace the existing `unbindAgent` method:

```java
public void unbindAgent(String agentId, String tenancyId) {
    if (tenancyId == null) {
        log.warnf("unbindAgent called with null tenancyId for agentId=%s — agent was never provisioned; no-op", agentId);
        return;
    }
    agentToCase.remove(new AgentKey(agentId, tenancyId));
}
```

With:

```java
public void unbindAgent(String agentId) {
    agentToCase.remove(agentId);
}
```

- [ ] **Step 7.4: Update `query` — drop tenancyId, simplify map lookup**

Replace the existing `query` method signature and first lookup line:

```java
// OLD signature:
public WindowContent query(String agentId, String tenancyId, long since) {
    UUID caseId = agentToCase.get(new AgentKey(agentId, tenancyId));

// NEW signature:
public WindowContent query(String agentId, long since) {
    UUID caseId = agentToCase.get(agentId);
```

The rest of the method body (buffer aggregation, merge, sort) is unchanged.

- [ ] **Step 7.5: Remove the `AgentKey` import**

Delete the import for `AgentKey` from `ChannelContextWindowService.java` (it is no longer referenced).

- [ ] **Step 7.6: Delete `AgentKey.java`**

```bash
rm /Users/mdproctor/claude/casehub/openclaw/core/src/main/java/io/casehub/openclaw/context/AgentKey.java
git -C /Users/mdproctor/claude/casehub/openclaw rm \
  core/src/main/java/io/casehub/openclaw/context/AgentKey.java
```

- [ ] **Step 7.7: Run core tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core
```

Expected: BUILD SUCCESS. All tests pass including the new `bindAgent_sameAgentId_secondCallOverwritesFirst`.

- [ ] **Step 7.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java \
  core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "refactor(core): remove AgentKey — ChannelContextWindowService uses plain agentId key — Refs #33"
```

---

## Task 8: #33 — Update callers in `casehub/` module

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`

- [ ] **Step 8.1: Update `OpenClawWorkerProvisioner.provision()` and `terminate()`**

In `provision()`, change:
```java
service.bindAgent(agentId, tenancyId, caseId);
```
to:
```java
service.bindAgent(agentId, caseId);
```

In `terminate()`, change:
```java
service.unbindAgent(workerId, tenancyId);   // null-safe: service logs warn and no-ops
```
to:
```java
service.unbindAgent(workerId);
```

- [ ] **Step 8.2: Update `ReactiveOpenClawWorkerProvisioner.provision()` and `terminate()`**

In `provision()`, change:
```java
service.bindAgent(agentId, tenancyId, caseId);
```
to:
```java
service.bindAgent(agentId, caseId);
```

In `terminate()`, change:
```java
service.unbindAgent(workerId, tenancyId);   // null-safe: service logs warn and no-ops
```
to:
```java
service.unbindAgent(workerId);
```

Note: `final String tenancyId = currentPrincipal.tenancyId();` on line 61 of `ReactiveOpenClawWorkerProvisioner` **stays** — tenancyId is still used by `registry.register(agentId, tenancyId, caseId, sessionKey)` on the next line.

- [ ] **Step 8.3: Update `OpenClawWorkerStatusListener.onWorkerCompleted()`**

Change lines 52–56 from:

```java
// Capture caseId and tenancyId before deregistering — registry removes the mappings on deregister()
UUID caseId = registry.findCaseId(workerId).orElse(null);
String tenancyId = caseId != null ? registry.findTenancyId(caseId).orElse(null) : null;
registry.deregister(workerId);
service.unbindAgent(workerId, tenancyId);
```

to:

```java
// Capture caseId before deregistering — registry removes the mappings on deregister()
UUID caseId = registry.findCaseId(workerId).orElse(null);
registry.deregister(workerId);
service.unbindAgent(workerId);
```

- [ ] **Step 8.4: Verify `casehub/` compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl casehub -am 2>&1 | tail -5
```

Expected: BUILD SUCCESS (compilation only — tests still have old stubs).

---

## Task 9: #33 — Update caller tests in `casehub/`

**Files:**
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`

- [ ] **Step 9.1: Update `OpenClawWorkerProvisionerTest`**

Test `provision_callsBindAgent_onContextWindowService` (line 64): change:
```java
verify(mockService).bindAgent("code-review-agent", "test-tenant", caseId);
```
to:
```java
verify(mockService).bindAgent("code-review-agent", caseId);
```

Test `terminate_unbindsAgentWithTenancyId` (line 114): change:
```java
verify(mockService).unbindAgent("finance-agent", "test-tenant");
```
to:
```java
verify(mockService).unbindAgent("finance-agent");
```

Test `terminate_unknownAgent_stillCallsUnbindAgent` (line 121): change:
```java
verify(mockService).unbindAgent(eq("not-registered"), isNull());
```
to:
```java
verify(mockService).unbindAgent("not-registered");
```

Also remove now-unused imports `eq` and `isNull` from Mockito if they are only used in that one test.

`when(mockPrincipal.tenancyId()).thenReturn("test-tenant")` in `setup()` **stays** — `provision()` still calls `currentPrincipal.tenancyId()` for `registry.register()`.

- [ ] **Step 9.2: Update `ReactiveOpenClawWorkerProvisionerTest`**

Test `provision_callsBindAgent_onContextWindowService` (line 77): change:
```java
verify(mockService).bindAgent("code-review-agent", "test-tenant", caseId);
```
to:
```java
verify(mockService).bindAgent("code-review-agent", caseId);
```

Test `terminate_unbindsAgentWithTenancyId` (line 153): change:
```java
verify(mockService).unbindAgent("finance-agent", "test-tenant");
```
to:
```java
verify(mockService).unbindAgent("finance-agent");
```

Test `terminate_unknownAgent_stillCallsUnbindAgent` (line 162): change:
```java
verify(mockService).unbindAgent(eq("not-registered"), isNull());
```
to:
```java
verify(mockService).unbindAgent("not-registered");
```

`when(mockPrincipal.tenancyId()).thenReturn("test-tenant")` in `setup()` **stays** — same reason as blocking provisioner.

Remove now-unused Mockito imports `eq` and `isNull` if applicable.

- [ ] **Step 9.3: Update `OpenClawWorkerStatusListenerTest`**

Test `onWorkerCompleted_callsUnbindAgentOnContextWindowService` (line 55): change:
```java
verify(mockService).unbindAgent("agent-1", "test-tenant");
```
to:
```java
verify(mockService).unbindAgent("agent-1");
```

Test `onWorkerCompleted_readsTenancyIdBeforeDeregister` — **delete this test entirely** (lines 59–64). Its purpose was to verify tenancyId is read before deregister; that code is now gone from production.

Test `onWorkerCompleted_unknownAgent_unbindAgentWithNullTenancyId` (line 71): change:
```java
verify(mockService).unbindAgent(eq("unknown"), isNull());
```
to:
```java
verify(mockService).unbindAgent("unknown");
```
Rename the test method to `onWorkerCompleted_unknownAgent_callsUnbindAgent`.

Remove now-unused imports `eq` and `isNull` if applicable.

- [ ] **Step 9.4: Run all `casehub/` tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am
```

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 9.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java \
  casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java \
  casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "refactor(casehub): update provisioners and status listener to 1-arg unbindAgent/2-arg bindAgent — Refs #33"
```

---

## Task 10: #33 — Update `ChannelContextWindowResource` and its tests

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java`
- Delete: `app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java`

- [ ] **Step 10.1: Update `ChannelContextWindowResource`**

In `ChannelContextWindowResource.java`:
- Delete `import io.casehub.platform.api.identity.CurrentPrincipal;`
- Delete the `@Inject CurrentPrincipal currentPrincipal;` field
- Remove the Javadoc paragraph: `<p>tenancyId is resolved via {@link CurrentPrincipal} and passed to {@link ChannelContextWindowService#query(String, String, long)} so that the window is tenant-scoped (openclaw#29).`
- Change `query()` method body from:
  ```java
  return service.query(agentId, currentPrincipal.tenancyId(), since);
  ```
  to:
  ```java
  return service.query(agentId, since);
  ```

- [ ] **Step 10.2: Delete `CrossTenantContextIsolationTest`**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw rm \
  app/src/test/java/io/casehub/openclaw/app/CrossTenantContextIsolationTest.java
```

- [ ] **Step 10.3: Update `ChannelContextWindowResourceTest`**

Remove `@InjectMock CurrentPrincipal currentPrincipal;` field (and its import).

Remove all `when(currentPrincipal.tenancyId()).thenReturn(TEST_TENANCY_ID);` stubs.

Delete the `TEST_TENANCY_ID` constant (no longer used).

Update test method bodies — change all 3-arg service stubs/verifications to 2-arg:

Test `get_knownAgent_returns200WithMessages`:
```java
// OLD: when(service.query("test-agent", TEST_TENANCY_ID, 0L)).thenReturn(contentWithOneMessage());
// NEW:
when(service.query("test-agent", 0L)).thenReturn(contentWithOneMessage());
```

Test `get_unknownAgent_returns200NotAssociated`:
```java
// OLD: when(service.query(anyString(), anyString(), anyLong())).thenReturn(WindowContent.noAssociation());
// NEW:
when(service.query(anyString(), anyLong())).thenReturn(WindowContent.noAssociation());
```

Test `get_sinceParamPassedThrough`:
```java
// OLD: when(service.query("agent-1", TEST_TENANCY_ID, 42L)).thenReturn(WindowContent.noAssociation());
// OLD verify: verify(service).query("agent-1", TEST_TENANCY_ID, 42L);
// NEW:
when(service.query("agent-1", 42L)).thenReturn(WindowContent.noAssociation());
// ...
verify(service).query("agent-1", 42L);
```

Test `get_sinceDefaultsToZero`:
```java
// OLD: when(service.query("agent-1", TEST_TENANCY_ID, 0L)).thenReturn(WindowContent.noAssociation());
// OLD verify: verify(service).query("agent-1", TEST_TENANCY_ID, 0L);
// NEW:
when(service.query("agent-1", 0L)).thenReturn(WindowContent.noAssociation());
// ...
verify(service).query("agent-1", 0L);
```

Delete test `get_tenancyIdFromPrincipalPassedToService` entirely (behavior gone).

Add new test `get_queryCallsServiceWithAgentIdAndSince_noPrincipalInteraction`:
```java
@Test
void get_queryCallsServiceWithAgentIdAndSince_noPrincipalInteraction() {
    when(service.query("agent-1", 0L)).thenReturn(WindowContent.noAssociation());

    given()
            .when().get("/channel-context/agent-1")
            .then()
            .statusCode(200);

    verify(service).query("agent-1", 0L);
    // No CurrentPrincipal interaction — the resource no longer reads it
    verifyNoInteractions(currentPrincipal);  // remove this line if currentPrincipal field is gone
}
```

Note: with `@InjectMock CurrentPrincipal` removed, there's no `currentPrincipal` field to call `verifyNoInteractions` on. Instead, the test is simply: verify `service.query("agent-1", 0L)` is called, and the test passing without any principal configuration is itself the proof.

Simplified final form:
```java
@Test
void get_queryCallsServiceWithAgentIdAndSince_noPrincipalInteraction() {
    when(service.query("agent-1", 0L)).thenReturn(WindowContent.noAssociation());

    given()
            .when().get("/channel-context/agent-1")
            .then()
            .statusCode(200);

    verify(service).query("agent-1", 0L);
}
```

- [ ] **Step 10.4: Run app tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am
```

Expected: BUILD SUCCESS. All 4 remaining resource tests pass (5 old tests became 4 after deletion + 1 new = 4 total after `get_tenancyIdFromPrincipalPassedToService` deletion and addition of `get_queryCallsServiceWithAgentIdAndSince_noPrincipalInteraction`).

- [ ] **Step 10.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java \
  app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m \
  "refactor(app): ChannelContextWindowResource removes CurrentPrincipal — service resolves tenancyId internally — Refs #33"
```

---

## Task 11: Full build verification

- [ ] **Step 11.1: Run complete build and all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS. All modules compile and all tests pass.

- [ ] **Step 11.2: Confirm issue references appear in git log**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw log --oneline -10
```

Confirm commits contain `Refs #32`, `Refs #33`, `Refs #34`.

---

## Self-review against spec

- [x] **#32 LAYOUT.keySet() assertion** → Task 1 `OpenClawNormativeLayoutTest`
- [x] **#32 value assertions per entry** → Task 1 step 1.1 (4 test methods)
- [x] **#32 blocking provider updated** → Task 2
- [x] **#32 reactive provider updated** → Task 3
- [x] **#34 CrossTenantChannelStore added to constructor** → Task 5 step 5.2
- [x] **#34 recovery logic in fulfill()** → Task 5 step 5.3
- [x] **#34 test for successful recovery** → Task 4 step 4.4 (test 1)
- [x] **#34 test for recovery when channel not found** → Task 4 step 4.4 (test 2)
- [x] **#33 AgentKey deleted** → Task 7 step 7.6
- [x] **#33 agentToCase plain String key** → Task 7 step 7.1
- [x] **#33 bindAgent(agentId, caseId) — 2-arg** → Task 7 step 7.2
- [x] **#33 unbindAgent(agentId) — 1-arg, no null guard** → Task 7 step 7.3
- [x] **#33 query(agentId, since) — 2-arg** → Task 7 step 7.4
- [x] **#33 last-write-wins test** → Task 6 step 6.5
- [x] **#33 4 AgentKey isolation tests removed** → Task 6 step 6.1
- [x] **#33 resource removes CurrentPrincipal field + import** → Task 10 step 10.1
- [x] **#33 resource Javadoc paragraph removed** → Task 10 step 10.1
- [x] **#33 resource calls service.query(agentId, since)** → Task 10 step 10.1
- [x] **#33 CrossTenantContextIsolationTest deleted** → Task 10 step 10.2
- [x] **#33 resource test: 5 tests updated/removed, 1 new** → Task 10 step 10.3
- [x] **#33 provisioner tests: mockPrincipal stub retained** → Task 9 steps 9.1, 9.2
- [x] **#33 StatusListener dead code removed** → Task 8 step 8.3
- [x] **#33 StatusListener comment updated** → Task 8 step 8.3
- [x] **#33 stale comments in terminate() removed** → Task 8 steps 8.1, 8.2
- [x] **#33 readsTenancyIdBeforeDeregister test deleted** → Task 9 step 9.3
