# Parallel COMMAND Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing. Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #70 — ChannelBackend parallel COMMAND routing for multi-agent cases
**Issue group:** openclaw#70, qhorus#370, engine#758

**Goal:** Flow agent targeting information from engine provisioning through Qhorus
dispatch to ChannelBackend delivery, enabling correct COMMAND routing in 1:N agent cases.

**Architecture:** Four gaps in the dispatch chain are closed bottom-up: `OutboundMessage`
gains `target` (qhorus), `ProvisionResult` returns `workerId` and `postToChannel()` gains
`target` (engine), and `OpenClawChannelBackend.post()` reads `target` with validated fallback
(openclaw). See `docs/specs/2026-07-19-parallel-command-routing-design.md` for the reviewed design.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny (reactive), Qhorus messaging

## Global Constraints

- Pre-release platform — breaking changes are acceptable and preferred over shims.
- `MessageService.dispatch()` is the sole channel write gate (PP-20260523-a08b97).
- Delivery endpoints always return 200 (PP-20260603-d52060).
- Normative channel layout unchanged (PP-20260622-normative-layout).
- All three repos use `0.2-SNAPSHOT`. Build order: qhorus → engine → openclaw.

## Slot Context

All work happens in **slot 6** (`/Users/mdproctor/claude/casehub/worktrees/6/`).
Each repo has its own worktree on branch `issue-70-channelbackend-parallel-routing`.

| Repo | Slot path | Issue |
|------|-----------|-------|
| qhorus | `worktrees/6/qhorus` | qhorus#370 |
| engine | `worktrees/6/engine` | engine#758 |
| openclaw | `worktrees/6/openclaw` | openclaw#70 |

Each repo needs its own: work-end, ARC42STORIES update, blog entry, issue closure, HANDOFF.

---

## Task 1: `OutboundMessage` gains `target` (qhorus-api)

**Repo:** `worktrees/6/qhorus`
**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`

**Interfaces:**
- Produces: `OutboundMessage(UUID messageId, String sender, MessageType type, String content, String correlationId, Long inReplyTo, ActorType senderActorType, List<ArtefactRef> artefactRefs, String target)` — new 9-field record

- [ ] **Step 1: Add `target` field to the record**

```java
public record OutboundMessage(
        UUID messageId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        ActorType senderActorType,
        java.util.List<io.casehub.qhorus.api.message.ArtefactRef> artefactRefs,
        String target) {}
```

Use `ide_edit_member` on `OutboundMessage.java`, member=`OutboundMessage`, to replace the record declaration.

- [ ] **Step 2: Find and update ALL test construction sites**

Search: `ide_search_text` for `new OutboundMessage(` across `api/src/test/` and `runtime/src/test/`. Every call site needs the additional `null` parameter appended. This is mechanical — add `, null)` before the closing paren on each.

Also search in `testing/` module if it exists.

- [ ] **Step 3: Build qhorus-api module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api -f worktrees/6/qhorus/pom.xml`

Expected: Compilation failures in `runtime/` (production sites not yet updated — that's Task 2).
The `api/` module itself should compile cleanly.

- [ ] **Step 4: Commit**

```
feat(#370): add target field to OutboundMessage for agent routing

The target field carries the intended recipient agentId through the
Qhorus dispatch pipeline. Previously dropped when converting
MessageDispatch → OutboundMessage.

Refs qhorus#370
```

---

## Task 2: Populate `target` at all 6 construction sites (qhorus runtime)

**Repo:** `worktrees/6/qhorus`
**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (lines ~257, ~382)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/ChannelGateway.java` (line ~433)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java` (line ~168)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/ReactiveMessageService.java` (lines ~388, ~449)

**Interfaces:**
- Consumes: `OutboundMessage` 9-field record from Task 1
- Consumes: `MessageDispatch.target()` (already exists on the record)
- Consumes: `Message.target()` (already exists on the entity)

- [ ] **Step 1: Update MessageService — LAST_WRITE fanOut (site 1, line ~257)**

The construction builds from a `MessageDispatch`. Add `dispatch.target()` as the 9th argument.

- [ ] **Step 2: Update MessageService — normal fanOut (site 2, line ~382)**

Same pattern — add `dispatch.target()`.

- [ ] **Step 3: Update ChannelGateway.deliverRemote() (site 3, line ~433)**

This constructs from a stored `Message`. Add `msg.target()` as the 9th argument:

```java
OutboundMessage outbound = new OutboundMessage(UUID.randomUUID(),
    msg.sender(), msg.messageType(), msg.content(), msg.correlationId(),
    msg.inReplyTo(), msg.actorType(), msg.artefactRefs(), msg.target());
```

- [ ] **Step 4: Update DeliveryBatchExecutor.toOutbound() (site 4, line ~168)**

This constructs from a stored `Message`. Add `msg.target()`. This is the primary
AT_LEAST_ONCE delivery path — if this site omits target, OpenClaw never sees it.

- [ ] **Step 5: Update ReactiveMessageService — OverwriteResult fanOut (site 5, line ~388)**

Constructs from `MessageDispatch`. Add `dispatch.target()`.

- [ ] **Step 6: Update ReactiveMessageService — FullResult fanOut (site 6, line ~449)**

Constructs from `MessageDispatch`. Add `dispatch.target()`.

- [ ] **Step 7: Update remaining test sites in runtime**

Search `runtime/src/test/` for `new OutboundMessage(` and append `, null)` to each.

- [ ] **Step 8: Build full qhorus**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -f worktrees/6/qhorus/pom.xml`

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 9: Commit**

```
feat(#370): populate target at all OutboundMessage construction sites

Six production sites updated to flow MessageDispatch.target() or
Message.target() into the new OutboundMessage.target field.

Closes qhorus#370
```

---

## Task 3: `ProvisionResult.workerId` + `postToChannel()` target (engine-api)

**Repo:** `worktrees/6/engine`
**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/ProvisionResult.java`
- Modify: `api/src/main/java/io/casehub/api/spi/CaseChannelProvider.java`
- Modify: `api/src/main/java/io/casehub/api/spi/ReactiveCaseChannelProvider.java`

**Interfaces:**
- Produces: `ProvisionResult(UUID causedByEntryId, String workerId)` with `empty()` and `withWorker(String)` factories
- Produces: `CaseChannelProvider.postToChannel(CaseChannel, String, String, MessageType, String, String, String)` — 7-param abstract
- Produces: `ReactiveCaseChannelProvider.postToChannel(...)` — 7-param abstract returning `Uni<Void>`

- [ ] **Step 1: Add `workerId` to ProvisionResult**

```java
public record ProvisionResult(UUID causedByEntryId, String workerId) {

  public static ProvisionResult empty() {
    return new ProvisionResult(null, null);
  }

  public static ProvisionResult withWorker(String workerId) {
    return new ProvisionResult(null, workerId);
  }
}
```

Use `ide_edit_member` on `ProvisionResult.java`, member=`ProvisionResult`, to replace the full record declaration.

- [ ] **Step 2: Add `target` to CaseChannelProvider.postToChannel()**

Replace the 6-param abstract with a 7-param abstract. Update the 3-param default to delegate to 7-param:

```java
void postToChannel(
    CaseChannel channel,
    String from,
    String content,
    MessageType type,
    String correlationId,
    String deadline,
    String target);

default void postToChannel(CaseChannel channel, String from, String content) {
    postToChannel(channel, from, content, null, null, null, null);
}
```

Use `ide_edit_member` for both the `postToChannel` methods.

- [ ] **Step 3: Add `target` to ReactiveCaseChannelProvider.postToChannel()**

Same change — 7-param abstract returning `Uni<Void>`, 3-param default delegates to 7-param:

```java
Uni<Void> postToChannel(
    CaseChannel channel,
    String from,
    String content,
    MessageType type,
    String correlationId,
    String deadline,
    String target);

default Uni<Void> postToChannel(CaseChannel channel, String from, String content) {
    return postToChannel(channel, from, content, null, null, null, null);
}
```

- [ ] **Step 4: Build engine-api**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api -f worktrees/6/engine/pom.xml -Dsurefire.failIfNoSpecifiedTests=false`

Expected: Compilation failures in `runtime/` (NoOp providers, WorkerScheduleEventHandler — that's Task 4).
The `api/` module should compile cleanly.

- [ ] **Step 5: Commit**

```
feat(#758): add workerId to ProvisionResult and target to postToChannel

ProvisionResult now returns the physical agent identity via workerId().
CaseChannelProvider.postToChannel() gains a 7th parameter (target) so
the engine can pass the intended recipient to the provider.

Breaking: 6-param postToChannel() is removed. All implementations must
update to the 7-param signature.

Refs engine#758
```

---

## Task 4: Engine runtime — NoOp providers + dispatch threading (engine runtime)

**Repo:** `worktrees/6/engine`
**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCaseChannelProvider.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveCaseChannelProvider.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Search and update: all `WorkerScheduleEvent` construction sites in `runtime/` and `common/`
- Search and update: all `new ProvisionResult(` and `ProvisionResult.empty()` call sites
- Search and update: all `postToChannel(` call sites (6-param → 7-param)

**Interfaces:**
- Consumes: 7-param `CaseChannelProvider.postToChannel()` from Task 3
- Consumes: `ProvisionResult.workerId()` from Task 3

- [ ] **Step 1: Update NoOpCaseChannelProvider — add `target` parameter**

```java
@Override
public void postToChannel(
    CaseChannel channel,
    String from,
    String content,
    MessageType type,
    String correlationId,
    String deadline,
    String target) {
  // intentional no-op
}
```

- [ ] **Step 2: Update NoOpReactiveCaseChannelProvider — add `target` parameter**

```java
@Override
public Uni<Void> postToChannel(
    CaseChannel channel,
    String from,
    String content,
    MessageType type,
    String correlationId,
    String deadline,
    String target) {
  return Uni.createFrom().voidItem();
}
```

- [ ] **Step 3: Add `target` to WorkerScheduleEvent**

Add `String target` as a new field on the record. Update all convenience constructors to pass `null`:

```java
public record WorkerScheduleEvent(
    CaseInstance caseInstance,
    Worker worker,
    Capability capability,
    String bindingName,
    String inputProjectionOverride,
    UUID signalId,
    ExecutionOrigin origin,
    List<RetrievedExperience> experiences,
    String target) {
  // ... compact constructor and convenience constructors updated
}
```

- [ ] **Step 4: Update WorkerScheduleEventHandler.dispatchCommand() — pass target**

Read `event.target()` (or the threaded `target` value) and pass it as the 7th param:

```java
caseChannelProvider.postToChannel(
    channel,
    "casehub-engine:orchestrator",
    serialize(command),
    MessageType.COMMAND,
    String.valueOf(eventLogId),
    deadline,
    target);
```

The `target` value comes from the `WorkerScheduleEvent`. Thread it through `submitIfNeeded()` → `dispatchCommand()`.

- [ ] **Step 5: Find and update all WorkerScheduleEvent construction sites**

Search for `new WorkerScheduleEvent(` across the engine codebase. Each needs the additional
`null` target parameter (or the actual `ProvisionResult.workerId()` where provisioning context
is available). The callers that create events after provisioning should pass the workerId.

- [ ] **Step 6: Find and update all ProvisionResult construction sites**

Search for `new ProvisionResult(` and `ProvisionResult.empty()` across the engine codebase.
`ProvisionResult(UUID)` no longer compiles — update to `ProvisionResult(UUID, null)` or
use `ProvisionResult.empty()`.

- [ ] **Step 7: Find and update all 6-param postToChannel() call sites**

Search for `postToChannel(` in the engine runtime. Any call with 6 args needs a 7th
(the target — `null` for non-targeted dispatches, or the workerId for provisioned workers).

- [ ] **Step 8: Build full engine**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -f worktrees/6/engine/pom.xml`

Expected: BUILD SUCCESS. All tests pass. If tests construct `OutboundMessage` from
the qhorus dependency, they need the new `target` param — install qhorus from Task 2 first.

- [ ] **Step 9: Commit**

```
feat(#758): thread target through engine dispatch path

WorkerScheduleEvent gains target field. WorkerScheduleEventHandler
passes it to postToChannel(). NoOp providers updated to 7-param.

Closes engine#758
```

---

## Task 5: OpenClaw provider, provisioner, and backend routing (openclaw)

**Repo:** `worktrees/6/openclaw`
**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProvider.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProviderTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawCaseChannelProviderTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java`

**Interfaces:**
- Consumes: `OutboundMessage.target()` from Task 1
- Consumes: `ProvisionResult.withWorker(String)` from Task 3
- Consumes: 7-param `CaseChannelProvider.postToChannel()` from Task 3

### Sub-task 5a: Provisioner returns workerId

- [ ] **Step 1: Write failing test — OpenClawWorkerProvisionerTest**

```java
@Test
void provision_returnsProvisionResultWithAgentIdAsWorkerId() {
    ProvisionResult result = provisioner.provision(
        Set.of("research"), provisionContext());
    assertThat(result.workerId()).isEqualTo("research-agent");
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -f worktrees/6/openclaw/pom.xml -Dtest=OpenClawWorkerProvisionerTest#provision_returnsProvisionResultWithAgentIdAsWorkerId -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — `ProvisionResult.empty()` has null workerId.

- [ ] **Step 3: Update OpenClawWorkerProvisioner.provision()**

Change `return ProvisionResult.empty();` to `return ProvisionResult.withWorker(agentId);`

- [ ] **Step 4: Run test — verify it passes**

- [ ] **Step 5: Same for ReactiveOpenClawWorkerProvisioner**

Write the parallel test and implementation.

- [ ] **Step 6: Commit**

```
feat(#70): provisioner returns agentId as workerId in ProvisionResult

Enables the engine to pass the physical agent identity as the target
when dispatching COMMANDs.

Refs #70
```

### Sub-task 5b: Provider passes target to MessageDispatch

- [ ] **Step 1: Write failing test — OpenClawCaseChannelProviderTest**

```java
@Test
void postToChannel_7param_setsTargetOnDispatch() {
    provider.postToChannel(caseChannel, "engine", "content",
        MessageType.COMMAND, "corr-1", null, "finance-agent");

    ArgumentCaptor<MessageDispatch> captor =
        ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().target()).isEqualTo("finance-agent");
}

@Test
void postToChannel_7param_nullTarget_dispatchHasNullTarget() {
    provider.postToChannel(caseChannel, "engine", "content",
        MessageType.COMMAND, "corr-1", null, null);

    ArgumentCaptor<MessageDispatch> captor =
        ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().target()).isNull();
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Update OpenClawCaseChannelProvider.postToChannel()**

Replace the 6-param method with the 7-param signature. Set `.target(target)` on the builder:

```java
@Override
public void postToChannel(CaseChannel channel, String from, String content,
                           MessageType type, String correlationId,
                           String deadline, String target) {
    MessageType effectiveType = type != null ? type : MessageType.STATUS;
    messageService.dispatch(MessageDispatch.builder()
            .channelId(UUID.fromString(channel.id()))
            .sender(from)
            .type(effectiveType)
            .content(content)
            .correlationId(correlationId)
            .deadline(deadline != null ? Instant.parse(deadline) : null)
            .target(target)
            .actorType(ActorType.AGENT)
            .build());
}
```

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Same for ReactiveOpenClawCaseChannelProvider**

- [ ] **Step 6: Commit**

```
feat(#70): provider passes target to MessageDispatch

OpenClawCaseChannelProvider and ReactiveOpenClawCaseChannelProvider
now set .target() on the MessageDispatch builder from the 7-param
postToChannel() signature.

Refs #70
```

### Sub-task 5c: Backend reads target with validated routing

- [ ] **Step 1: Write failing tests — OpenClawChannelBackendTest**

Update the `command()` and `commandWithCorrelationId()` helpers to include `target`:

```java
private OutboundMessage commandWithTarget(String content, String target) {
    return new OutboundMessage(UUID.randomUUID(), "engine", MessageType.COMMAND,
            content, null, null, ActorType.AGENT, null, target);
}
```

Tests:

```java
@Test
void post_command_withTarget_routesToTargetAgent() {
    registry.register("agent-a", "t", caseId, "sk-a");
    registry.register("agent-b", "t", caseId, "sk-b");
    ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");

    backend.post(ref, commandWithTarget("task", "agent-a"));

    verify(hookClient).invoke(eq("agent-a"), any(), any(), anyInt());
}

@Test
void post_command_withTarget_notRegisteredForCase_noOp() {
    registry.register("agent-a", "t", caseId, "sk-a");
    ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");

    backend.post(ref, commandWithTarget("task", "agent-x"));

    verify(hookClient, never()).invoke(any(), any(), any(), anyInt());
}

@Test
void post_command_nullTarget_singleAgent_fallsBack() {
    registry.register("agent-a", "t", caseId, "sk-a");
    ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");

    backend.post(ref, commandWithTarget("task", null));

    verify(hookClient).invoke(eq("agent-a"), any(), any(), anyInt());
}

@Test
void post_command_nullTarget_multipleAgents_dropsCommand() {
    registry.register("agent-a", "t", caseId, "sk-a");
    registry.register("agent-b", "t", caseId, "sk-b");
    ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");

    backend.post(ref, commandWithTarget("task", null));

    verify(hookClient, never()).invoke(any(), any(), any(), anyInt());
}

@Test
void post_command_withTarget_crossCaseAgent_noOp() {
    UUID otherCase = UUID.randomUUID();
    registry.register("agent-a", "t", otherCase, "sk-a");
    ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");

    backend.post(ref, commandWithTarget("task", "agent-a"));

    verify(hookClient, never()).invoke(any(), any(), any(), anyInt());
}
```

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement target-based routing in OpenClawChannelBackend.post()**

```java
@Override
public void post(final ChannelRef channel, final OutboundMessage message) {
    if (message.type() != MessageType.COMMAND) return;

    final UUID caseId = extractCaseId(channel.name());
    if (caseId == null) return;

    String agentId = message.target();

    if (agentId != null) {
        if (!registry.findAgentIds(caseId).contains(agentId)) {
            log.warnf("Target agent %s not registered for caseId=%s — "
                    + "ignoring COMMAND on %s", agentId, caseId, channel.name());
            return;
        }
    } else {
        Set<String> agents = registry.findAgentIds(caseId);
        if (agents.isEmpty()) {
            log.debugf("No OpenClaw agent for caseId=%s — ignoring COMMAND on %s",
                       caseId, channel.name());
            return;
        }
        if (agents.size() > 1) {
            log.errorf("Multiple agents registered for caseId=%s but COMMAND has "
                    + "no target — cannot route. Engine must set target for "
                    + "multi-agent cases (openclaw#70).", caseId);
            return;
        }
        agentId = agents.iterator().next();
    }

    final String sessionKey = registry.findSessionKey(agentId).orElse(null);
    if (sessionKey == null) {
        log.warnf("No session key found for agentId=%s — ignoring COMMAND", agentId);
        return;
    }

    String webhookUrl = config.delivery().baseUrl() + "/channel/" + channel.id();
    webhookUrl = appendDeliveryToken(webhookUrl);
    hookClient.registerSession(agentId, sessionKey, webhookUrl);

    try {
        final UUID correlationId = message.correlationId() != null
                ? UUID.fromString(message.correlationId())
                : null;
        hookClient.invoke(agentId, buildPrompt(message.content(), agentId, correlationId),
                config.agent().defaultModel(), config.agent().defaultTimeoutSeconds());
        log.debugf("Invoked OpenClaw agent: agentId=%s caseId=%s target=%s",
                   agentId, caseId, message.target());
    } catch (OpenClawInvocationException e) {
        log.errorf("OpenClaw invocation failed for agentId=%s: %s", agentId, e.getMessage());
    }
}
```

- [ ] **Step 4: Update existing test helpers to include target parameter**

All existing `command()` and `commandWithCorrelationId()` helpers need `, null` appended
for the new `target` field on `OutboundMessage`. Also update any other tests that construct
`OutboundMessage` directly.

- [ ] **Step 5: Run all tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -f worktrees/6/openclaw/pom.xml`

- [ ] **Step 6: Commit**

```
feat(#70): target-based COMMAND routing in OpenClawChannelBackend

Routes COMMANDs to the agent specified in OutboundMessage.target().
Validates target is registered for the case (prevents cross-case
misrouting). Falls back to single-agent lookup when target is null.
Drops COMMANDs with no target when multiple agents are registered.

Closes #70
```

---

## Task 6: Cross-repo build verification

- [ ] **Step 1: Install qhorus to local .m2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -f worktrees/6/qhorus/pom.xml`

- [ ] **Step 2: Install engine to local .m2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -f worktrees/6/engine/pom.xml`

- [ ] **Step 3: Build and test openclaw**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -f worktrees/6/openclaw/pom.xml`

Expected: BUILD SUCCESS across all three repos.

---

## Task 7: Per-repo work-end

Each repo needs its own closure cycle. The slot session should run work-end
for each repo independently, in dependency order:

1. **qhorus** — work-end closes qhorus#370, updates qhorus ARC42STORIES.MD, writes blog, HANDOFF
2. **engine** — work-end closes engine#758, updates engine ARC42STORIES.MD, writes blog, HANDOFF
3. **openclaw** — work-end closes openclaw#70, updates openclaw ARC42STORIES.MD, writes blog, HANDOFF

Each work-end should reference the cross-repo nature of the change and link to the companion issues.
