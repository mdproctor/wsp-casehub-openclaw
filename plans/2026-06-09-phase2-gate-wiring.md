# Phase 2 Gate Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `CommitmentTools.done()` through `ActionRiskClassifier` → when `GateRequired`, dispatch a COMMAND to the oversight channel instead of closing the commitment; update `fulfill()` to close the agent's commitment (DONE/DECLINE) on gate resolution.

**Architecture:** `OversightGateService.openGate()` (new) classifies the proposed action via `@RiskClassifier Instance<ActionRiskClassifier>`, opens a gate COMMAND on the oversight channel embedding original-commitment context as Properties-format content, and returns a `GateDecision` sealed result. `CommitmentTools.channelBacked_done()` switches on that result — `Autonomous` proceeds with DONE, `GatePending` returns a "pending oversight" JSON to the agent. `OversightGateService.fulfill()` parses the gate COMMAND content to recover the original commitment and dispatches DONE/DECLINE to the work channel atomically via the updated `OversightGateDispatcher`.

**Tech Stack:** Java 21, Quarkus CDI, casehub-engine-api (`ActionRiskClassifier`, `PlannedAction`, `RiskDecision`, `@RiskClassifier`), casehub-qhorus-api (`MessageService`, `MessageType`), Mockito 5, AssertJ.

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/GateDecision.java` | Public sealed return type of `openGate()` |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java` | Package-private record: original commitment context for gate resolution |
| Modify | `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java` | Add `Optional<GateContext>` param; DONE/DECLINE on work channel when context present |
| Modify | `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java` | Add `openGate()`, `classifyMostRestrictive()`, `serializeGateContent()`, `parseGateContent()`; update `fulfill()` |
| Modify | `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java` | Update existing 6-arg stubs; add 3 new tests for context path |
| Modify | `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java` | Update setUp stub; add 10 new tests |
| Modify | `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java` | Inject `OversightGateService`; update `channelBacked_done()` |
| Modify | `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java` | Update setUp; add 3 new tests |
| Modify | `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java` | Update stale comment about gate entry point |

---

### Task 1: Create GateDecision and GateContext types

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/GateDecision.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java`

These are pure data types — no tests needed.

- [ ] **Step 1: Create GateDecision.java**

```java
package io.casehub.openclaw.casehub;

import java.util.UUID;

public sealed interface GateDecision permits GateDecision.Autonomous, GateDecision.GatePending {
    record Autonomous() implements GateDecision {}
    record GatePending(UUID gateId, String reason) implements GateDecision {}
}
```

- [ ] **Step 2: Create GateContext.java**

```java
package io.casehub.openclaw.casehub;

import java.util.UUID;

record GateContext(String originalCommitmentId, UUID workChannelId, long commandMessageId) {}
```

- [ ] **Step 3: Compile check**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl casehub -am -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/GateDecision.java casehub/src/main/java/io/casehub/openclaw/casehub/GateContext.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): GateDecision and GateContext types for Phase 2 gate wiring

Refs #30"
```

---

### Task 2: Update OversightGateDispatcher — new 7-arg signature with context-aware dispatch

`OversightGateDispatcher.dispatch()` gains `Optional<GateContext>` as the 7th parameter. When context is present and approved: dispatches DONE (not STATUS) to the work channel to close the agent's commitment. When rejected with context: dispatches DECLINE to the work channel. When context is absent (restart / content parse error): falls back to STATUS on work channel (existing behaviour).

All three dispatch pairs remain atomic in the single `@Transactional` boundary.

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java`

- [ ] **Step 1: Write failing tests for the new context path — and update existing tests to the new signature**

Replace the entire content of `OversightGateDispatcherTest.java` with:

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
    UUID workChannelId = UUID.randomUUID();
    UUID gateId = UUID.randomUUID();
    String originalCommitmentId = UUID.randomUUID().toString();
    long originalCommandMsgId = 99L;
    GateContext gateContext = new GateContext(originalCommitmentId, workChannelId, originalCommandMsgId);

    @BeforeEach
    void setup() {
        messageService = mock(MessageService.class);
        dispatcher = new OversightGateDispatcher(messageService);
        when(messageService.dispatch(any())).thenReturn(dispatchResult(1L));
    }

    // ── no-context fallback (existing behaviour) ────────────────────────────

    @Test
    void dispatch_approved_noContext_sendsResponseToOversightAndStatusToWork() {
        dispatcher.dispatch(true, oversightChannelId, workChannelId, 42L, gateId, "approved",
                Optional.empty());

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getAllValues().get(0);
        assertThat(oversight.channelId()).isEqualTo(oversightChannelId);
        assertThat(oversight.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(oversight.sender()).isEqualTo(OversightGateService.GATE_SENDER);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());
        assertThat(oversight.inReplyTo()).isEqualTo(42L);
        assertThat(oversight.content()).isEqualTo("approved");
        assertThat(oversight.actorType()).isEqualTo(ActorType.AGENT);

        MessageDispatch work = captor.getAllValues().get(1);
        assertThat(work.channelId()).isEqualTo(workChannelId);
        assertThat(work.type()).isEqualTo(MessageType.STATUS);
        assertThat(work.sender()).isEqualTo(OversightGateService.GATE_SENDER);
        assertThat(work.actorType()).isEqualTo(ActorType.AGENT);
    }

    @Test
    void dispatch_rejected_noContext_sendsDeclineToOversightAndStatusToWork() {
        dispatcher.dispatch(false, oversightChannelId, workChannelId, 42L, gateId, "rejected",
                Optional.empty());

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getAllValues().get(0);
        assertThat(oversight.type()).isEqualTo(MessageType.DECLINE);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());
        assertThat(oversight.inReplyTo()).isEqualTo(42L);

        MessageDispatch work = captor.getAllValues().get(1);
        assertThat(work.type()).isEqualTo(MessageType.STATUS);
    }

    @Test
    void dispatch_approved_noContext_nullOutput_contentDefaultsToApproved() {
        dispatcher.dispatch(true, oversightChannelId, workChannelId, 42L, gateId, null,
                Optional.empty());

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());
        assertThat(captor.getAllValues().get(0).content()).isEqualTo("approved");
    }

    @Test
    void dispatch_rejected_noContext_nullOutput_contentDefaultsToRejected() {
        dispatcher.dispatch(false, oversightChannelId, workChannelId, 42L, gateId, null,
                Optional.empty());

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());
        assertThat(captor.getAllValues().get(0).content()).isEqualTo("rejected");
    }

    // ── with context (new behaviour) ────────────────────────────────────────

    @Test
    void dispatch_approved_withContext_sendsResponseToOversightAndDoneToWork() {
        dispatcher.dispatch(true, oversightChannelId, workChannelId, 42L, gateId, "approved",
                Optional.of(gateContext));

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getAllValues().get(0);
        assertThat(oversight.channelId()).isEqualTo(oversightChannelId);
        assertThat(oversight.type()).isEqualTo(MessageType.RESPONSE);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());
        assertThat(oversight.inReplyTo()).isEqualTo(42L);

        MessageDispatch work = captor.getAllValues().get(1);
        assertThat(work.channelId()).isEqualTo(workChannelId);
        assertThat(work.type()).isEqualTo(MessageType.DONE);
        assertThat(work.correlationId()).isEqualTo(originalCommitmentId);
        assertThat(work.inReplyTo()).isEqualTo(originalCommandMsgId);
        assertThat(work.sender()).isEqualTo(OversightGateService.GATE_SENDER);
        assertThat(work.actorType()).isEqualTo(ActorType.AGENT);
    }

    @Test
    void dispatch_rejected_withContext_sendsDeclineToOversightAndDeclineToWork() {
        dispatcher.dispatch(false, oversightChannelId, workChannelId, 42L, gateId, "rejected",
                Optional.of(gateContext));

        ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
        verify(messageService, times(2)).dispatch(captor.capture());

        MessageDispatch oversight = captor.getAllValues().get(0);
        assertThat(oversight.type()).isEqualTo(MessageType.DECLINE);
        assertThat(oversight.correlationId()).isEqualTo(gateId.toString());

        MessageDispatch work = captor.getAllValues().get(1);
        assertThat(work.type()).isEqualTo(MessageType.DECLINE);
        assertThat(work.correlationId()).isEqualTo(originalCommitmentId);
        assertThat(work.inReplyTo()).isEqualTo(originalCommandMsgId);
    }

    @Test
    void dispatch_approved_withContext_twoDispatchCallsTotal() {
        dispatcher.dispatch(true, oversightChannelId, workChannelId, 42L, gateId, "approved",
                Optional.of(gateContext));
        verify(messageService, times(2)).dispatch(any());
    }

    private DispatchResult dispatchResult(Long messageId) {
        return new DispatchResult(messageId, oversightChannelId, OversightGateService.GATE_SENDER,
                MessageType.RESPONSE, null, null, null, null, null, null, null, 0);
    }
}
```

- [ ] **Step 2: Run tests — expect compilation failures because OversightGateDispatcher still has 6-arg dispatch()**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateDispatcherTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: compilation error — `dispatch()` method with 7 args not found.

- [ ] **Step 3: Update OversightGateDispatcher.java**

Replace the entire file:

```java
package io.casehub.openclaw.casehub;

import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.MessageService;

/**
 * Owns the atomic dispatch boundary for oversight gate decisions.
 *
 * <p>Both dispatches in an approve/reject decision commit in the same transaction,
 * so a crash between them cannot leave the case step hanging with a resolved
 * Commitment but no closure on the work channel (openclaw#15).
 *
 * <p>When {@code gateContext} is present (Phase 2 gate path), the work-channel
 * dispatch closes the agent's original commitment: DONE on approval, DECLINE on
 * rejection. When absent (restart recovery / content parse error), the work channel
 * receives STATUS only (no commitment closure — logged by caller).
 *
 * <p>Accepts primitives only — no JPA entities cross a @Transactional boundary.
 * Package-private: only OversightGateService may call this.
 */
@ApplicationScoped
class OversightGateDispatcher {

    private final MessageService messageService;

    @Inject
    OversightGateDispatcher(MessageService messageService) {
        this.messageService = messageService;
    }

    @Transactional
    void dispatch(boolean approved,
                  UUID oversightChannelId,
                  UUID workChannelId,
                  long commandMessageId,
                  UUID gateId,
                  String rawOutput,
                  Optional<GateContext> gateContext) {
        if (approved) {
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(oversightChannelId)
                    .sender(OversightGateService.GATE_SENDER)
                    .type(MessageType.RESPONSE)
                    .content(rawOutput != null ? rawOutput : "approved")
                    .correlationId(gateId.toString())
                    .inReplyTo(commandMessageId)
                    .actorType(ActorType.AGENT)
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
                        .build());
            } else {
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(workChannelId)
                        .sender(OversightGateService.GATE_SENDER)
                        .type(MessageType.STATUS)
                        .content("Gate approved")
                        .actorType(ActorType.AGENT)
                        .build());
            }
        } else {
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(oversightChannelId)
                    .sender(OversightGateService.GATE_SENDER)
                    .type(MessageType.DECLINE)
                    .content(rawOutput != null ? rawOutput : "rejected")
                    .correlationId(gateId.toString())
                    .inReplyTo(commandMessageId)
                    .actorType(ActorType.AGENT)
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
                        .build());
            } else {
                messageService.dispatch(MessageDispatch.builder()
                        .channelId(workChannelId)
                        .sender(OversightGateService.GATE_SENDER)
                        .type(MessageType.STATUS)
                        .content("Human rejected the proposed action via oversight gate")
                        .actorType(ActorType.AGENT)
                        .build());
            }
        }
    }
}
```

- [ ] **Step 4: Run the dispatcher tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateDispatcherTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -15
```

Expected: all 7 tests pass.

- [ ] **Step 5: Also run OversightGateServiceTest to confirm it now fails (setUp stub uses old 6-arg signature — this is expected)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: compilation failure — `doNothing().when(gateDispatcher).dispatch(anyBoolean(), any(), any(), anyLong(), any(), any())` no longer matches the 7-arg signature.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OversightGateDispatcher — context-aware DONE/DECLINE dispatch

Refs #30"
```

---

### Task 3: Implement OversightGateService.openGate()

This is the core of Phase 2. `openGate()` classifies the proposed action via `@RiskClassifier Instance<ActionRiskClassifier>` and either returns `Autonomous` (proceed with DONE) or opens an oversight gate COMMAND on the oversight channel and returns `GatePending`.

The method must also fix the broken `OversightGateServiceTest` setUp stub (6-arg → 7-arg).

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`

- [ ] **Step 1: Add failing openGate() tests and fix the broken setUp stub**

In `OversightGateServiceTest.java`:

1. Add these imports:
```java
import java.io.StringReader;
import java.util.List;
import java.util.Properties;

import jakarta.enterprise.inject.Instance;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskDecision;
import io.casehub.openclaw.casehub.GateDecision;
```

2. Add these fields after `OversightGateService service;`:
```java
@SuppressWarnings("unchecked")
Instance<ActionRiskClassifier> classifiers = mock(Instance.class);
ActionRiskClassifier mockClassifier = mock(ActionRiskClassifier.class);

String agentId = "test-agent";
String commitmentId = UUID.randomUUID().toString();
long commandMsgId = 7L;
```

3. Replace the `setUp()` method:
```java
@BeforeEach
void setup() {
    channelService = mock(ChannelService.class);
    messageService = mock(MessageService.class);
    commitmentStore = mock(CommitmentStore.class);

    workChannel = new Channel();
    workChannel.id = workChannelId;
    workChannel.name = "case-" + caseId + "/work";

    oversightChannel = new Channel();
    oversightChannel.id = oversightChannelId;
    oversightChannel.name = "case-" + caseId + "/oversight";

    when(channelService.findById(workChannelId)).thenReturn(Optional.of(workChannel));
    when(channelService.findById(oversightChannelId)).thenReturn(Optional.of(oversightChannel));
    when(channelService.findByName("case-" + caseId + "/oversight"))
            .thenReturn(Optional.of(oversightChannel));
    when(channelService.findByName("case-" + caseId + "/work"))
            .thenReturn(Optional.of(workChannel));

    gateDispatcher = mock(OversightGateDispatcher.class);
    // Updated to 7-arg signature
    doNothing().when(gateDispatcher).dispatch(
            anyBoolean(), any(), any(), anyLong(), any(), any(), any());

    when(messageService.dispatch(any())).thenReturn(dispatchResult(42L));

    // Default: no classifiers (isUnsatisfied = true)
    when(classifiers.isUnsatisfied()).thenReturn(true);

    service = new OversightGateService(channelService, messageService, commitmentStore,
            gateDispatcher, classifiers);
}
```

4. Add these new test methods after the existing `fulfill_*` tests:

```java
// ── openGate() — no classifiers ───────────────────────────────────────────

@Test
void openGate_noClassifiers_returnsAutonomous() {
    when(classifiers.isUnsatisfied()).thenReturn(true);
    stubOpenGateCommitment();

    GateDecision result = service.openGate(agentId, commitmentId, "analysis done");

    assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    verify(messageService, never()).dispatch(any());
}

// ── openGate() — Autonomous decision ─────────────────────────────────────

@Test
void openGate_classifierReturnsAutonomous_returnsAutonomous() {
    stubSingleClassifier(new RiskDecision.Autonomous());
    stubOpenGateCommitment();

    GateDecision result = service.openGate(agentId, commitmentId, "analysis done");

    assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    verify(messageService, never()).dispatch(any());
}

// ── openGate() — GateRequired decision ───────────────────────────────────

@Test
void openGate_classifierReturnsGateRequired_dispatchesCommandToOversightAndReturnsPending() {
    stubSingleClassifier(new RiskDecision.GateRequired("risk: file deletion", true, null, null, null));
    stubOpenGateCommitment();

    GateDecision result = service.openGate(agentId, commitmentId, "deleting old reports");

    assertThat(result).isInstanceOf(GateDecision.GatePending.class);
    GateDecision.GatePending pending = (GateDecision.GatePending) result;
    assertThat(pending.reason()).isEqualTo("risk: file deletion");
    assertThat(pending.gateId()).isNotNull();

    // Gate COMMAND dispatched to oversight channel
    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    MessageDispatch cmd = captor.getValue();
    assertThat(cmd.channelId()).isEqualTo(oversightChannelId);
    assertThat(cmd.type()).isEqualTo(MessageType.COMMAND);
    assertThat(cmd.sender()).isEqualTo(OversightGateService.GATE_SENDER);
    assertThat(cmd.correlationId()).isEqualTo(pending.gateId().toString());
    assertThat(cmd.content()).isNotBlank();
}

@Test
void openGate_gateCommandContentIsPropertiesFormatContainingOriginalCommitmentId()
        throws Exception {
    stubSingleClassifier(new RiskDecision.GateRequired("risky", true, null, null, null));
    stubOpenGateCommitment();

    service.openGate(agentId, commitmentId, "outcome");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    String content = captor.getValue().content();

    Properties props = new Properties();
    props.load(new StringReader(content));
    assertThat(props.getProperty("originalCommitmentId")).isEqualTo(commitmentId);
    assertThat(props.getProperty("workChannelId")).isEqualTo(workChannelId.toString());
    assertThat(props.getProperty("commandMessageId")).isEqualTo(String.valueOf(commandMsgId));
    assertThat(props.getProperty("reason")).isEqualTo("risky");
}

// ── openGate() — fail-safe on classifier exception ────────────────────────

@Test
void openGate_classifierThrows_appliesFailSafeGateRequired() {
    stubSingleClassifier_throws(new RuntimeException("classifier crashed"));
    stubOpenGateCommitment();

    GateDecision result = service.openGate(agentId, commitmentId, "outcome");

    // Fail-safe: exception → GateRequired → gate opened
    assertThat(result).isInstanceOf(GateDecision.GatePending.class);
    GateDecision.GatePending pending = (GateDecision.GatePending) result;
    assertThat(pending.reason()).contains("Classifier error");
}

// ── openGate() — fail-open on infrastructure errors ───────────────────────

@Test
void openGate_oversightChannelMissing_returnsAutonomous() {
    stubSingleClassifier(new RiskDecision.GateRequired("risky", true, null, null, null));
    stubOpenGateCommitment();
    when(channelService.findByName("case-" + caseId + "/oversight"))
            .thenReturn(Optional.empty());

    GateDecision result = service.openGate(agentId, commitmentId, "outcome");

    assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    verify(messageService, never()).dispatch(any());
}

@Test
void openGate_dispatchThrows_failsOpenAndReturnsAutonomous() {
    stubSingleClassifier(new RiskDecision.GateRequired("risky", true, null, null, null));
    stubOpenGateCommitment();
    when(messageService.dispatch(any())).thenThrow(new RuntimeException("channel unavailable"));

    GateDecision result = service.openGate(agentId, commitmentId, "outcome");

    assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
}

@Test
void openGate_noChannelBackedCommitment_returnsAutonomous() {
    stubSingleClassifier(new RiskDecision.GateRequired("risky", true, null, null, null));
    // commitment has no channelId (self-commit)
    Commitment c = commitment(commitmentId, null, agentId, null);
    when(commitmentStore.findByCorrelationId(commitmentId)).thenReturn(Optional.of(c));

    GateDecision result = service.openGate(agentId, commitmentId, "outcome");

    assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    verify(messageService, never()).dispatch(any());
}

@Test
void openGate_mostRestrictive_picksSmallerCandidateGroupsAsMoreRestrictive() {
    // Two classifiers: narrow (1 group) vs broad (2 groups) — narrow wins
    ActionRiskClassifier narrowClassifier = mock(ActionRiskClassifier.class);
    ActionRiskClassifier broadClassifier = mock(ActionRiskClassifier.class);
    when(narrowClassifier.classify(any()))
            .thenReturn(new RiskDecision.GateRequired("narrow", true, List.of("admin"), null, null));
    when(broadClassifier.classify(any()))
            .thenReturn(new RiskDecision.GateRequired("broad", true, List.of("admin", "member"), null, null));
    when(classifiers.isUnsatisfied()).thenReturn(false);
    when(classifiers.iterator()).thenReturn(List.of(narrowClassifier, broadClassifier).iterator());
    stubOpenGateCommitment();

    GateDecision result = service.openGate(agentId, commitmentId, "outcome");

    assertThat(result).isInstanceOf(GateDecision.GatePending.class);
    assertThat(((GateDecision.GatePending) result).reason()).isEqualTo("narrow");
}

// ── openGate() helpers ────────────────────────────────────────────────────

private void stubOpenGateCommitment() {
    Commitment c = commitment(commitmentId, workChannelId, agentId,
            java.time.Instant.now().plusSeconds(3600));
    when(commitmentStore.findByCorrelationId(commitmentId)).thenReturn(Optional.of(c));
    when(messageService.findAllByCorrelationId(commitmentId))
            .thenReturn(List.of(commandMessage(commitmentId, commandMsgId)));
}

private void stubSingleClassifier(RiskDecision decision) {
    when(mockClassifier.classify(any())).thenReturn(decision);
    when(classifiers.isUnsatisfied()).thenReturn(false);
    when(classifiers.iterator()).thenReturn(List.of(mockClassifier).iterator());
}

private void stubSingleClassifier_throws(RuntimeException ex) {
    when(mockClassifier.classify(any())).thenThrow(ex);
    when(classifiers.isUnsatisfied()).thenReturn(false);
    when(classifiers.iterator()).thenReturn(List.of(mockClassifier).iterator());
}
```

5. Add these two helpers at the bottom of the helpers section:

```java
private Commitment commitment(String correlationId, UUID channelId,
                               String obligor, java.time.Instant expiresAt) {
    Commitment c = new Commitment();
    c.id = UUID.randomUUID();
    c.correlationId = correlationId;
    c.channelId = channelId;
    c.obligor = obligor;
    c.state = io.casehub.qhorus.api.message.CommitmentState.OPEN;
    c.expiresAt = expiresAt;
    return c;
}

private Message commandMessage(String correlationId, long messageId) {
    Message m = new Message();
    m.id = messageId;
    m.messageType = MessageType.COMMAND;
    m.correlationId = correlationId;
    return m;
}
```

- [ ] **Step 2: Run tests — expect compilation failure (openGate() not yet on OversightGateService, new constructor arg missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: compilation error.

- [ ] **Step 3: Implement openGate() and classifier composition in OversightGateService.java**

Replace the entire file:

```java
package io.casehub.openclaw.casehub;

import java.io.IOException;
import java.io.StringReader;
import java.io.StringWriter;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Properties;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.CommitmentStore;

/**
 * Owns the oversight gate lifecycle.
 *
 * <p>{@link #evaluate(UUID, String, String)} archives the agent's webhook output as a
 * non-resolving STATUS message on the work channel.
 *
 * <p>{@link #openGate(String, String, String)} classifies the proposed action via
 * {@code @RiskClassifier} CDI beans. If {@code GateRequired}, dispatches a COMMAND to
 * the oversight channel and returns {@link GateDecision.GatePending}. If {@code Autonomous}
 * (or no classifiers registered), returns {@link GateDecision.Autonomous} so the caller
 * can proceed with normal DONE dispatch. Fail-open on infrastructure errors.
 *
 * <p>{@link #fulfill(UUID, String)} processes human responses to oversight gates. Parses
 * the original commitment context from the gate COMMAND message content (persisted in Qhorus)
 * and dispatches DONE or DECLINE to close the agent's work commitment.
 *
 * <p>All three methods catch and log all exceptions — none propagate to callers.
 */
@ApplicationScoped
public class OversightGateService {

    private static final Logger log = Logger.getLogger(OversightGateService.class);
    static final String GATE_SENDER = "openclaw-gate";

    private final ChannelService channelService;
    private final MessageService messageService;
    private final CommitmentStore commitmentStore;
    private final OversightGateDispatcher gateDispatcher;
    private final Instance<ActionRiskClassifier> classifiers;

    @Inject
    public OversightGateService(final ChannelService channelService,
                                 final MessageService messageService,
                                 final CommitmentStore commitmentStore,
                                 final OversightGateDispatcher gateDispatcher,
                                 @RiskClassifier final Instance<ActionRiskClassifier> classifiers) {
        this.channelService = channelService;
        this.messageService = messageService;
        this.commitmentStore = commitmentStore;
        this.gateDispatcher = gateDispatcher;
        this.classifiers = classifiers;
    }

    /**
     * Archives the agent's webhook output as a non-resolving STATUS on the work channel.
     */
    public void evaluate(final UUID workChannelId, final String agentId, final String output) {
        try {
            if (output == null || output.isBlank()) return;
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(workChannelId)
                    .sender(agentId)
                    .type(MessageType.STATUS)
                    .content(output)
                    .actorType(ActorType.AGENT)
                    .build());
        } catch (Exception e) {
            log.errorf("evaluate() failed to archive webhook output for channel=%s agent=%s: %s",
                    workChannelId, agentId, e.getMessage());
        }
    }

    /**
     * Classifies the proposed action and either opens an oversight gate or returns Autonomous.
     *
     * <p>When {@code GateRequired}: dispatches a COMMAND to the case oversight channel with gate
     * context serialized in the content (persisted by Qhorus for crash-safe fulfillment).
     *
     * <p>Fail-open: infrastructure failures (channel not found, dispatch error) → Autonomous.
     * Classifier exception → GateRequired fail-safe (not Autonomous — failure ≠ safe).
     */
    public GateDecision openGate(final String agentId, final String commitmentId,
                                  final String outcome) {
        try {
            Optional<Commitment> cOpt = commitmentStore.findByCorrelationId(commitmentId);
            if (cOpt.isEmpty() || cOpt.get().channelId == null) {
                log.warnf("openGate: no channel-backed commitment for correlationId=%s — failing open",
                        commitmentId);
                return new GateDecision.Autonomous();
            }
            UUID workChannelId = cOpt.get().channelId;

            Channel workChannel = channelService.findById(workChannelId).orElse(null);
            if (workChannel == null) {
                log.warnf("openGate: work channel %s not found — failing open", workChannelId);
                return new GateDecision.Autonomous();
            }

            UUID caseId = CaseChannelNames.extractCaseId(workChannel.name);
            if (caseId == null) {
                log.warnf("openGate: cannot extract caseId from channel name '%s' — failing open",
                        workChannel.name);
                return new GateDecision.Autonomous();
            }

            PlannedAction action = new PlannedAction(agentId, caseId, outcome, "COMPLETION", Map.of());
            RiskDecision decision = classifyMostRestrictive(action);

            if (decision instanceof RiskDecision.Autonomous) {
                return new GateDecision.Autonomous();
            }

            RiskDecision.GateRequired gate = (RiskDecision.GateRequired) decision;

            Channel oversightChannel = channelService
                    .findByName(CaseChannelNames.oversightChannelName(caseId)).orElse(null);
            if (oversightChannel == null) {
                log.warnf("openGate: oversight channel not found for caseId=%s — failing open " +
                        "(oversight not configured)", caseId);
                return new GateDecision.Autonomous();
            }

            long commandMessageId = messageService.findAllByCorrelationId(commitmentId).stream()
                    .filter(m -> m.messageType == MessageType.COMMAND)
                    .mapToLong(m -> m.id)
                    .findFirst()
                    .orElse(-1L);

            UUID gateId = UUID.randomUUID();
            GateContext ctx = new GateContext(commitmentId, workChannelId, commandMessageId);

            messageService.dispatch(MessageDispatch.builder()
                    .channelId(oversightChannel.id)
                    .sender(GATE_SENDER)
                    .type(MessageType.COMMAND)
                    .content(serializeGateContent(ctx, gate.reason()))
                    .correlationId(gateId.toString())
                    .actorType(ActorType.AGENT)
                    .build());

            log.infof("Gate opened: gateId=%s agentId=%s commitmentId=%s caseId=%s reason=%s",
                    gateId, agentId, commitmentId, caseId, gate.reason());

            return new GateDecision.GatePending(gateId, gate.reason());
        } catch (Exception e) {
            log.errorf("openGate() failed for agentId=%s commitmentId=%s: %s — failing open",
                    agentId, commitmentId, e.getMessage());
            return new GateDecision.Autonomous();
        }
    }

    /**
     * Processes the oversight agent's response to a gate.
     */
    public void fulfill(final UUID gateId, final String rawOutput) {
        try {
            Optional<Message> gateCmdOpt = messageService.findAllByCorrelationId(gateId.toString())
                    .stream()
                    .filter(m -> m.messageType == MessageType.COMMAND)
                    .findFirst();
            if (gateCmdOpt.isEmpty()) {
                log.warnf("fulfill() called for gateId=%s — no COMMAND message found via correlationId; " +
                        "possible restart or duplicate delivery; failing open", gateId);
                return;
            }
            long commandMessageId = gateCmdOpt.get().id;

            Optional<GateContext> gateContext = parseGateContent(gateCmdOpt.get().content);
            if (gateContext.isEmpty()) {
                log.warnf("fulfill() gateId=%s — gate context missing or unparseable; " +
                        "work commitment will not be closed (falling back to STATUS)", gateId);
            }

            boolean approved = parseApproval(gateId, rawOutput);

            Optional<Commitment> commitmentOpt = commitmentStore.findByCorrelationId(gateId.toString());
            if (commitmentOpt.isEmpty()) {
                log.warnf("fulfill() called for unknown gateId=%s — possible duplicate delivery, ignoring",
                        gateId);
                return;
            }

            Commitment commitment = commitmentOpt.get();
            Channel oversightChannel = channelService.findById(commitment.channelId).orElse(null);
            if (oversightChannel == null) {
                log.errorf("Oversight channel %s not found for gateId=%s — failing open",
                        commitment.channelId, gateId);
                return;
            }

            UUID caseId = CaseChannelNames.extractCaseId(oversightChannel.name);
            if (caseId == null) {
                log.errorf("Could not extract caseId from oversight channel '%s' for gateId=%s — failing open",
                        oversightChannel.name, gateId);
                return;
            }

            Channel workChannel = channelService.findByName(CaseChannelNames.workChannelName(caseId))
                    .orElse(null);
            if (workChannel == null) {
                log.errorf("Work channel not found for caseId=%s gateId=%s — failing open", caseId, gateId);
                return;
            }

            gateDispatcher.dispatch(approved, oversightChannel.id, workChannel.id,
                    commandMessageId, gateId, rawOutput, gateContext);
            log.infof("Gate %s: gateId=%s caseId=%s", approved ? "approved" : "rejected", gateId, caseId);
        } catch (Exception e) {
            log.errorf("OversightGateService.fulfill() failed for gateId=%s: %s", gateId, e.getMessage());
        }
    }

    private RiskDecision classifyMostRestrictive(PlannedAction action) {
        if (classifiers.isUnsatisfied()) return new RiskDecision.Autonomous();
        RiskDecision result = new RiskDecision.Autonomous();
        for (ActionRiskClassifier classifier : classifiers) {
            try {
                result = mostRestrictive(result, classifier.classify(action));
            } catch (Exception e) {
                log.warnf("ActionRiskClassifier %s threw for action '%s': %s — applying fail-safe GateRequired",
                        classifier.getClass().getSimpleName(), action.description(), e.getMessage());
                return new RiskDecision.GateRequired(
                        "Classifier error — manual review required before proceeding",
                        true, null, null, null);
            }
        }
        return result;
    }

    private RiskDecision mostRestrictive(RiskDecision a, RiskDecision b) {
        if (!(b instanceof RiskDecision.GateRequired)) return a;
        if (!(a instanceof RiskDecision.GateRequired gA)) return b;
        return narrower(gA, (RiskDecision.GateRequired) b);
    }

    private RiskDecision.GateRequired narrower(RiskDecision.GateRequired a, RiskDecision.GateRequired b) {
        int sizeA = a.candidateGroups() == null ? Integer.MAX_VALUE : a.candidateGroups().size();
        int sizeB = b.candidateGroups() == null ? Integer.MAX_VALUE : b.candidateGroups().size();
        if (sizeA != sizeB) return sizeA < sizeB ? a : b;
        if (a.expiresIn() != null && b.expiresIn() != null) {
            return a.expiresIn().compareTo(b.expiresIn()) <= 0 ? a : b;
        }
        return a.expiresIn() != null ? a : b;
    }

    private String serializeGateContent(GateContext ctx, String reason) {
        Properties props = new Properties();
        props.setProperty("originalCommitmentId", ctx.originalCommitmentId());
        props.setProperty("workChannelId", ctx.workChannelId().toString());
        props.setProperty("commandMessageId", String.valueOf(ctx.commandMessageId()));
        props.setProperty("reason", reason != null ? reason : "");
        StringWriter sw = new StringWriter();
        try {
            props.store(sw, null);
        } catch (IOException e) {
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
            if (oci == null || wci == null || cmi == null) return Optional.empty();
            return Optional.of(new GateContext(oci, UUID.fromString(wci), Long.parseLong(cmi)));
        } catch (Exception e) {
            log.warnf("parseGateContent: failed to parse gate content: %s", e.getMessage());
            return Optional.empty();
        }
    }

    private boolean parseApproval(final UUID gateId, final String rawOutput) {
        if (rawOutput == null || rawOutput.isBlank()) {
            log.warnf("fulfill() received null/blank output for gateId=%s — treating as rejected", gateId);
            return false;
        }
        String firstToken = rawOutput.trim().toLowerCase().split("\\s+")[0].replaceAll("[^a-z]+$", "");
        return firstToken.equals("approved");
    }
}
```

- [ ] **Step 4: Run openGate() tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: all tests pass (existing evaluate/fulfill tests + 10 new openGate tests).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OversightGateService.openGate() — classifier composition + gate COMMAND dispatch

Refs #30"
```

---

### Task 4: Add fulfill() tests for gate context path

`fulfill()` is already implemented in Task 3. This task adds tests verifying that it correctly uses the parsed `GateContext` to close the agent's commitment.

**Files:**
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`

- [ ] **Step 1: Add fulfill-with-context tests**

Add these test methods after the existing `fulfill_*` tests:

```java
// ── fulfill() — with gate context (Phase 2 behaviour) ─────────────────────

@Test
void fulfill_approved_withContext_callsGateDispatcherWithContextPresent() throws Exception {
    UUID gateId = setupGateStubsWithContext();
    when(commitmentStore.findByCorrelationId(gateId.toString()))
            .thenReturn(Optional.of(commitment(gateId)));

    service.fulfill(gateId, "approved");

    ArgumentCaptor<Optional> contextCaptor = ArgumentCaptor.forClass(Optional.class);
    verify(gateDispatcher).dispatch(
            eq(true), any(), any(), anyLong(), any(), any(), contextCaptor.capture());
    assertThat(contextCaptor.getValue()).isPresent();
    GateContext ctx = (GateContext) contextCaptor.getValue().get();
    assertThat(ctx.originalCommitmentId()).isEqualTo(commitmentId);
    assertThat(ctx.workChannelId()).isEqualTo(workChannelId);
    assertThat(ctx.commandMessageId()).isEqualTo(commandMsgId);
}

@Test
void fulfill_rejected_withContext_callsGateDispatcherWithContextPresent() throws Exception {
    UUID gateId = setupGateStubsWithContext();
    when(commitmentStore.findByCorrelationId(gateId.toString()))
            .thenReturn(Optional.of(commitment(gateId)));

    service.fulfill(gateId, "rejected");

    ArgumentCaptor<Boolean> approvedCaptor = ArgumentCaptor.forClass(Boolean.class);
    ArgumentCaptor<Optional> contextCaptor = ArgumentCaptor.forClass(Optional.class);
    verify(gateDispatcher).dispatch(
            approvedCaptor.capture(), any(), any(), anyLong(), any(), any(), contextCaptor.capture());
    assertThat(approvedCaptor.getValue()).isFalse();
    assertThat(contextCaptor.getValue()).isPresent();
}

@Test
void fulfill_malformedGateContent_callsGateDispatcherWithEmptyContext() {
    UUID gateId = UUID.randomUUID();
    Message cmd = commandMessage(gateId.toString(), 42L);
    cmd.content = "not-properties-format-at-all";
    when(messageService.findAllByCorrelationId(gateId.toString()))
            .thenReturn(List.of(cmd));
    when(commitmentStore.findByCorrelationId(gateId.toString()))
            .thenReturn(Optional.of(commitment(gateId)));

    service.fulfill(gateId, "approved");

    ArgumentCaptor<Optional> contextCaptor = ArgumentCaptor.forClass(Optional.class);
    verify(gateDispatcher).dispatch(
            anyBoolean(), any(), any(), anyLong(), any(), any(), contextCaptor.capture());
    assertThat(contextCaptor.getValue()).isEmpty();
}
```

Add the `setupGateStubsWithContext()` helper after `setupGateStubs()`:

```java
private UUID setupGateStubsWithContext() throws Exception {
    UUID gateId = UUID.randomUUID();
    // Build valid Properties-format content containing the original commitment context
    Properties props = new Properties();
    props.setProperty("originalCommitmentId", commitmentId);
    props.setProperty("workChannelId", workChannelId.toString());
    props.setProperty("commandMessageId", String.valueOf(commandMsgId));
    props.setProperty("reason", "test reason");
    java.io.StringWriter sw = new java.io.StringWriter();
    props.store(sw, null);
    String content = sw.toString();

    Message cmd = commandMessage(gateId.toString(), 42L);
    cmd.content = content;
    when(messageService.findAllByCorrelationId(gateId.toString()))
            .thenReturn(List.of(cmd));
    return gateId;
}
```

Also add `content` field set in the existing `commandMessage()` helper and update the existing `setupGateStubs()` to set content to empty (so the existing tests pick the no-context path):

Update `setupGateStubs()`:
```java
private UUID setupGateStubs() {
    UUID gateId = UUID.randomUUID();
    Message cmd = commandMessage(gateId.toString(), 42L);
    cmd.content = "";  // empty = no gate context → fallback to STATUS
    when(messageService.findAllByCorrelationId(gateId.toString()))
            .thenReturn(List.of(cmd));
    return gateId;
}
```

Also update the existing `fulfill_*` test stubs that use `verify(gateDispatcher).dispatch(approvedCaptor.capture(), ...)` — they need to add `any()` for the new 7th argument. Find all `verify(gateDispatcher).dispatch(...)` calls and add `, any()` at the end:

```java
// Before (in fulfill_approved_callsGateDispatcherWithApprovedTrue etc.):
verify(gateDispatcher).dispatch(
    approvedCaptor.capture(), oversightCaptor.capture(), workCaptor.capture(),
    inReplyToCaptor.capture(), gateIdCaptor.capture(), outputCaptor.capture());

// After:
verify(gateDispatcher).dispatch(
    approvedCaptor.capture(), oversightCaptor.capture(), workCaptor.capture(),
    inReplyToCaptor.capture(), gateIdCaptor.capture(), outputCaptor.capture(), any());
```

And for the shorter verify calls:
```java
// Before:
verify(gateDispatcher).dispatch(approvedCaptor.capture(), any(), any(), anyLong(), any(), any());
// After:
verify(gateDispatcher).dispatch(approvedCaptor.capture(), any(), any(), anyLong(), any(), any(), any());
```

Also need to add `import static org.mockito.ArgumentMatchers.eq;` to the imports.

And add `cmd.content` field setting in the existing `commandMessage()` helper so content is accessible:

```java
private Message commandMessage(String correlationId, long messageId) {
    Message m = new Message();
    m.id = messageId;
    m.messageType = MessageType.COMMAND;
    m.correlationId = correlationId;
    // content set per-test when needed
    return m;
}
```

- [ ] **Step 2: Run all casehub module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "test(casehub): fulfill() context path tests — DONE/DECLINE dispatch on gate resolution

Refs #30"
```

---

### Task 5: Update CommitmentTools.done() — inject OversightGateService and call openGate()

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`

- [ ] **Step 1: Write failing tests for the gated path in CommitmentToolsTest**

Add these imports to `CommitmentToolsTest.java`:
```java
import io.casehub.openclaw.casehub.GateDecision;
import io.casehub.openclaw.casehub.OversightGateService;
```

Add this field after `CommitmentTools tools;`:
```java
OversightGateService oversightGateService;
```

Update `setUp()` to mock `OversightGateService` and inject it:
```java
@BeforeEach
void setUp() {
    messageService = mock(MessageService.class);
    commitmentService = mock(CommitmentService.class);
    commitmentStore = mock(CommitmentStore.class);
    oversightGateService = mock(OversightGateService.class);
    // Default: Autonomous — normal done() path
    when(oversightGateService.openGate(any(), any(), any())).thenReturn(new GateDecision.Autonomous());
    tools = new CommitmentTools(messageService, commitmentService, commitmentStore, oversightGateService);
}
```

Add these new test methods after the existing `done_*` tests:

```java
// ── done(): gate decision paths ────────────────────────────────────────────

@Test
void done_channelBacked_autonomous_proceedsWithDone() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    when(commitmentStore.findByCorrelationId(correlationId))
            .thenReturn(Optional.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(messageService.findAllByCorrelationId(correlationId))
            .thenReturn(List.of(message(5L, channelId, MessageType.COMMAND, correlationId)));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(11L, channelId, agentId, MessageType.DONE, correlationId));
    when(oversightGateService.openGate(agentId, correlationId, "Report sent"))
            .thenReturn(new GateDecision.Autonomous());

    ToolResponse response = tools.done(agentId, correlationId, "Report sent");

    verify(messageService).dispatch(argThat(d -> MessageType.DONE == d.type()));
    assertThat(response.isError()).isFalse();
    assertThat(text(response)).contains("closed");
}

@Test
void done_channelBacked_gatePending_returnsPendingGateAndNoDispatch() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    UUID gateId = UUID.randomUUID();

    when(commitmentStore.findByCorrelationId(correlationId))
            .thenReturn(Optional.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(oversightGateService.openGate(agentId, correlationId, "Delete old files"))
            .thenReturn(new GateDecision.GatePending(gateId, "risk: file deletion"));

    ToolResponse response = tools.done(agentId, correlationId, "Delete old files");

    verify(messageService, never()).dispatch(any());
    assertThat(response.isError()).isFalse();
    String responseText = text(response);
    assertThat(responseText).contains("gated");
    assertThat(responseText).contains(gateId.toString());
    assertThat(responseText).contains("risk: file deletion");
}

@Test
void done_selfCommit_neverCallsOpenGate() {
    String agentId = "home-agent";
    String correlationId = UUID.randomUUID().toString();
    Commitment c = commitment(correlationId, null, agentId, Instant.now().plusSeconds(3600));

    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));
    when(commitmentService.fulfill(anyString())).thenReturn(Optional.of(c));

    tools.done(agentId, correlationId, null);

    verify(oversightGateService, never()).openGate(any(), any(), any());
}
```

- [ ] **Step 2: Run tests — expect compilation failure (CommitmentTools constructor not updated yet)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: compilation error.

- [ ] **Step 3: Update CommitmentTools.java**

Replace the entire file:

```java
package io.casehub.openclaw.app.mcp;

import java.time.Instant;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.jboss.logging.Logger;

import io.casehub.openclaw.casehub.GateDecision;
import io.casehub.openclaw.casehub.OversightGateService;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.DispatchResult;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.CommitmentService;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.quarkiverse.mcp.server.TextContent;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.quarkiverse.mcp.server.ToolResponse;

/**
 * MCP tools for the CaseHub commitment lifecycle.
 *
 * <p>Higher-level than Qhorus's {@code send_message} — the agent only needs its own
 * agentId and an optional channelId; this class resolves correlationId, inReplyTo, and
 * obligor automatically.
 *
 * <p>Commitment identity: for channel-backed commits, commitmentId == correlationId of
 * the open COMMAND commitment on that channel. For self-commits, a new UUID is generated.
 *
 * <p>Channel resolution: channelId is read from the {@link Commitment} entity on every
 * call to {@link #resolveChannelId(String)}, making the implementation crash-safe — no
 * in-memory map is needed. Terminal commitments (FULFILLED, DECLINED, DELEGATED, etc.)
 * are filtered out so that done()/reject() after escalate()/delegate() return
 * COMMITMENT_ALREADY_CLOSED rather than silently re-dispatching.
 *
 * <p>Phase 2 gate wiring (openclaw#30): {@link #channelBacked_done} calls
 * {@link OversightGateService#openGate} before dispatching DONE. If the result is
 * {@link GateDecision.GatePending}, the commitment stays open and the agent receives
 * a "pending oversight" response. If {@link GateDecision.Autonomous}, DONE is dispatched
 * normally.
 */
@ApplicationScoped
public class CommitmentTools {

    private static final Logger log = Logger.getLogger(CommitmentTools.class);

    private final MessageService messageService;
    private final CommitmentService commitmentService;
    private final CommitmentStore commitmentStore;
    private final OversightGateService oversightGateService;

    @Inject
    public CommitmentTools(MessageService messageService,
                           CommitmentService commitmentService,
                           CommitmentStore commitmentStore,
                           OversightGateService oversightGateService) {
        this.messageService = messageService;
        this.commitmentService = commitmentService;
        this.commitmentStore = commitmentStore;
        this.oversightGateService = oversightGateService;
    }

    // ---- casehub_commit ----

    @Tool(description = "Register a CaseHub commitment and arm a Watchdog. "
            + "For case steps, commitmentId is provided in the COMMAND message — call casehub_done directly "
            + "when complete without calling this tool first. "
            + "Call this tool only when you need to send an early STATUS acknowledgment to reset the Watchdog "
            + "for a long-running task (when channelId is provided, dispatches STATUS to the channel). "
            + "Without channelId, creates a self-tracked commitment. "
            + "Returns commitmentId.")
    public ToolResponse commit(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "Task description") String task,
            @ToolArg(description = "Deadline in ISO-8601 format (e.g. 2026-06-01T17:00:00Z); "
                    + "used only for self-commits", required = false) String deadline,
            @ToolArg(description = "Qhorus channel UUID where the original COMMAND was received; "
                    + "omit for self-commits", required = false) String channelId) {

        if (channelId != null) {
            return channelBacked_commit(agentId, task, channelId);
        }
        return selfCommit(agentId, task, deadline);
    }

    private ToolResponse channelBacked_commit(String agentId, String task, String channelIdStr) {
        UUID channelId;
        try {
            channelId = UUID.fromString(channelIdStr);
        } catch (IllegalArgumentException e) {
            return ToolResponse.error("INVALID_CHANNEL_ID: " + channelIdStr);
        }

        List<Commitment> open = commitmentStore.findOpenByObligor(agentId, channelId);
        if (open.isEmpty()) {
            return ToolResponse.error("No open COMMAND commitment found for agent '"
                    + agentId + "' on channel " + channelId);
        }

        Commitment commitment = open.get(0);
        String correlationId = commitment.correlationId;
        Instant deadline = commitment.expiresAt;

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.STATUS)
                .content("Acknowledging: " + task)
                .correlationId(correlationId)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("""
                {"commitmentId": "%s", "watchdogDeadline": "%s"}
                """.formatted(correlationId, deadline != null ? deadline.toString() : "none").strip());
    }

    private ToolResponse selfCommit(String agentId, String task, String deadlineStr) {
        Instant deadline = deadlineStr != null ? Instant.parse(deadlineStr) : null;
        String correlationId = UUID.randomUUID().toString();

        commitmentService.open(
                UUID.randomUUID(),
                correlationId,
                null,
                MessageType.COMMAND,
                agentId,
                agentId,
                deadline);

        return ToolResponse.success("""
                {"commitmentId": "%s", "watchdogDeadline": "%s"}
                """.formatted(correlationId, deadline != null ? deadline.toString() : "none").strip());
    }

    // ---- casehub_done ----

    @Tool(description = "Close a CaseHub commitment. Dispatches DONE to the originating Qhorus "
            + "channel (if channel-backed) or calls CommitmentService.fulfill() directly "
            + "(self-commit). Disarms the Watchdog. Always call this when a task is complete. "
            + "If the action requires human oversight, returns a pending gate response instead of "
            + "closing immediately — the commitment closes when the human approves or rejects.")
    public ToolResponse done(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "Optional outcome description", required = false) String outcome) {

        return resolveChannelId(commitmentId)
                .map(channelId -> channelBacked_done(agentId, commitmentId, channelId, outcome))
                .orElseGet(() -> selfCommit_done(commitmentId));
    }

    private ToolResponse channelBacked_done(String agentId, String correlationId,
                                             UUID channelId, String outcome) {
        GateDecision gate = oversightGateService.openGate(agentId, correlationId, outcome);
        if (gate instanceof GateDecision.GatePending g) {
            return ToolResponse.success(
                    """
                    {"gated": true, "gateId": "%s", "pendingReason": "%s"}
                    """.formatted(g.gateId(), g.reason()).strip());
        }

        // GateDecision.Autonomous — proceed with normal DONE dispatch
        long commandMessageId = findCommandMessageId(correlationId);
        if (commandMessageId < 0) {
            return ToolResponse.error("COMMAND_NOT_FOUND: no COMMAND message found for correlationId '"
                    + correlationId + "' — cannot dispatch DONE (inReplyTo is required)");
        }

        DispatchResult result = messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.DONE)
                .content(outcome != null ? outcome : "Done")
                .correlationId(correlationId)
                .inReplyTo(commandMessageId)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("""
                {"closed": true, "ledgerSeq": %d}
                """.formatted(result.messageId()).strip());
    }

    private ToolResponse selfCommit_done(String correlationId) {
        Optional<Commitment> existing = commitmentStore.findByCorrelationId(correlationId);
        if (existing.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + correlationId);
        }
        if (existing.get().state.isTerminal()) {
            return ToolResponse.error("COMMITMENT_ALREADY_CLOSED: " + correlationId
                    + " is in state " + existing.get().state);
        }
        commitmentService.fulfill(correlationId);
        return ToolResponse.success("{\"closed\": true}");
    }

    // ---- casehub_reject ----

    @Tool(description = "Decline a CaseHub commitment — DECLINE speech act. Use when you cannot "
            + "complete the task. Reason is required and recorded in the ledger.")
    public ToolResponse reject(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "Reason for declining") String reason) {

        return resolveChannelId(commitmentId)
                .map(channelId -> channelBacked_reject(agentId, commitmentId, channelId, reason))
                .orElseGet(() -> selfCommit_reject(commitmentId, reason));
    }

    private ToolResponse channelBacked_reject(String agentId, String correlationId,
                                               UUID channelId, String reason) {
        long commandMessageId = findCommandMessageId(correlationId);
        if (commandMessageId < 0) {
            return ToolResponse.error("COMMAND_NOT_FOUND: no COMMAND message found for correlationId '"
                    + correlationId + "' — cannot dispatch DECLINE (inReplyTo is required)");
        }

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.DECLINE)
                .content(reason)
                .correlationId(correlationId)
                .inReplyTo(commandMessageId)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("{\"declined\": true}");
    }

    private ToolResponse selfCommit_reject(String correlationId, String reason) {
        Optional<Commitment> existing = commitmentStore.findByCorrelationId(correlationId);
        if (existing.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + correlationId);
        }
        if (existing.get().state.isTerminal()) {
            return ToolResponse.error("COMMITMENT_ALREADY_CLOSED: " + correlationId
                    + " is in state " + existing.get().state);
        }
        commitmentService.decline(correlationId);
        return ToolResponse.success("{\"declined\": true}");
    }

    // ---- casehub_checkpoint ----

    @Tool(description = "Report progress on a commitment. Dispatches STATUS to the originating "
            + "channel and resets the Watchdog TTL. Use for long-running tasks to prevent "
            + "false escalation.")
    public ToolResponse checkpoint(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "Progress note") String note) {

        Optional<UUID> channelOpt = resolveChannelId(commitmentId);
        if (channelOpt.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + commitmentId);
        }
        UUID channelId = channelOpt.get();

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.STATUS)
                .content(note)
                .correlationId(commitmentId)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("{\"watchdogReset\": true}");
    }

    // ---- casehub_escalate ----

    @Tool(description = "Escalate a commitment to a human or named agent. Dispatches HANDOFF "
            + "to the originating channel. The Watchdog continues running — the escalation "
            + "target is responsible for resolving the commitment.")
    public ToolResponse escalate(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "Reason for escalation") String reason,
            @ToolArg(description = "Target agent or human identifier", required = false) String toAgent) {

        Optional<UUID> channelOpt = resolveChannelId(commitmentId);
        if (channelOpt.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + commitmentId);
        }
        UUID channelId = channelOpt.get();

        long commandMessageId = findCommandMessageId(commitmentId);
        if (commandMessageId < 0) {
            return ToolResponse.error("COMMAND_NOT_FOUND: no COMMAND message found for correlationId '"
                    + commitmentId + "' — cannot dispatch HANDOFF (inReplyTo is required)");
        }

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.HANDOFF)
                .content(reason)
                .correlationId(commitmentId)
                .inReplyTo(commandMessageId)
                .target(toAgent)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("{\"escalated\": true, \"escalationId\": \"%s\"}"
                .formatted(commitmentId));
    }

    // ---- casehub_block ----

    @Transactional
    @Tool(description = "Temporarily block a commitment when an external dependency prevents progress. "
            + "Extends the Watchdog deadline (expiresAt) to prevent premature expiry. "
            + "Call casehub_checkpoint with 'UNBLOCKED: <note>' when the blocker resolves. "
            + "Only the obligor of the commitment may call this tool.")
    public ToolResponse block(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "What is blocking progress") String reason,
            @ToolArg(description = "Expected resolution time in ISO-8601 format; must be in the future") String blockedUntil) {

        Instant newDeadline;
        try {
            newDeadline = Instant.parse(blockedUntil);
        } catch (Exception e) {
            return ToolResponse.error("INVALID_DEADLINE: " + blockedUntil);
        }

        if (!newDeadline.isAfter(Instant.now())) {
            return ToolResponse.error("DEADLINE_IN_PAST: blockedUntil must be a future timestamp, got: " + blockedUntil);
        }

        Optional<Commitment> cOpt = commitmentStore.findByCorrelationId(commitmentId);
        if (cOpt.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + commitmentId);
        }

        Commitment commitment = cOpt.get();
        if (commitment.state.isTerminal()) {
            return ToolResponse.error("COMMITMENT_ALREADY_CLOSED: commitment " + commitmentId
                    + " is already in terminal state " + commitment.state);
        }
        if (!agentId.equals(commitment.obligor)) {
            return ToolResponse.error("COMMITMENT_UNAUTHORIZED: agentId '" + agentId
                    + "' is not the obligor for commitment " + commitmentId);
        }

        commitment.expiresAt = newDeadline;
        commitmentStore.save(commitment);

        if (commitment.channelId != null) {
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(commitment.channelId)
                    .sender(agentId)
                    .type(MessageType.STATUS)
                    .content("BLOCKED: " + reason)
                    .correlationId(commitmentId)
                    .actorType(ActorType.AGENT)
                    .build());
        }

        return ToolResponse.success("""
                {"blocked": true, "newWatchdogDeadline": "%s"}
                """.formatted(newDeadline).strip());
    }

    // ---- casehub_delegate ----

    @Tool(description = "Intentionally transfer a commitment to a named agent or person. "
            + "Dispatches HANDOFF to the originating channel. The Watchdog continues — "
            + "the delegatee is now responsible for fulfilling the commitment. "
            + "Use when delegating responsibility, NOT when escalating for authority or capability reasons "
            + "(use casehub_escalate for that).")
    public ToolResponse delegate(
            @ToolArg(description = "Your OpenClaw agentId") String agentId,
            @ToolArg(description = "commitmentId returned by casehub_commit") String commitmentId,
            @ToolArg(description = "Reason for delegation — recorded in the Qhorus ledger") String reason,
            @ToolArg(description = "Target agent ID or human identifier") String toAgent) {

        Optional<UUID> channelOpt = resolveChannelId(commitmentId);
        if (channelOpt.isEmpty()) {
            return ToolResponse.error("COMMITMENT_NOT_FOUND: " + commitmentId);
        }
        UUID channelId = channelOpt.get();

        long commandMessageId = findCommandMessageId(commitmentId);
        if (commandMessageId < 0) {
            return ToolResponse.error("COMMAND_NOT_FOUND: no COMMAND message found for correlationId '"
                    + commitmentId + "' — cannot dispatch HANDOFF (inReplyTo is required)");
        }

        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
                .sender(agentId)
                .type(MessageType.HANDOFF)
                .content(reason)
                .correlationId(commitmentId)
                .inReplyTo(commandMessageId)
                .target(toAgent)
                .actorType(ActorType.AGENT)
                .build());

        return ToolResponse.success("""
                {"delegated": true, "delegatedTo": "%s"}
                """.formatted(toAgent).strip());
    }

    // ---- helpers ----

    private Optional<UUID> resolveChannelId(String correlationId) {
        return commitmentStore.findByCorrelationId(correlationId)
                .filter(c -> !c.state.isTerminal())
                .map(c -> c.channelId)
                .filter(id -> id != null);
    }

    private long findCommandMessageId(String correlationId) {
        return messageService.findAllByCorrelationId(correlationId)
                .stream()
                .filter(m -> m.messageType == MessageType.COMMAND)
                .mapToLong(m -> m.id)
                .findFirst()
                .orElse(-1L);
    }
}
```

- [ ] **Step 4: Run CommitmentTools tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: all tests pass (existing + 3 new).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(app): CommitmentTools.done() — Phase 2 gate wiring via OversightGateService.openGate()

Refs #30"
```

---

### Task 6: Update OversightGateDispatcherCdiTest comment

The CDI test's setup comment references openclaw#30 as future work. Update it to reflect the shipped implementation.

**Files:**
- Modify: `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java`

- [ ] **Step 1: Update the stale comment in the test**

Find this comment in `setUp()`:
```java
// 1. Dispatch setup COMMAND to oversight channel.
//    This simulates the gate COMMAND that OversightGateService.openGate() used to
//    dispatch (openclaw#30 will re-wire the gate entry). Qhorus InMemory auto-creates
//    a Commitment with channelId=oversightChannelId and correlationId=gateId —
//    this is what fulfill() looks up.
```

Replace with:
```java
// 1. Dispatch setup COMMAND to oversight channel (bare content — no Properties-format gate context).
//    This simulates a gate COMMAND from a pre-openclaw#30 environment or a post-restart scenario
//    where gate context is unavailable. Qhorus InMemory auto-creates a Commitment with
//    channelId=oversightChannelId and correlationId=gateId — this is what fulfill() looks up.
//    fulfill() will parse empty context and fall back to STATUS on the work channel.
```

- [ ] **Step 2: Run the CDI test to confirm it still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=OversightGateDispatcherCdiTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "test(app): update OversightGateDispatcherCdiTest comment — openclaw#30 shipped

Refs #30"
```

---

### Task 7: Full build verification

- [ ] **Step 1: Build and test all Maven modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install 2>&1 | tail -30
```

Expected: `BUILD SUCCESS`. All tests in `core`, `casehub`, and `app` pass.

- [ ] **Step 2: If any test fails, diagnose before proceeding**

Check the full test output:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD"
```

Common failures and fixes:
- `dispatch()` method ambiguity → ensure all call sites use 7 args
- `doNothing().when(gateDispatcher).dispatch(...)` stub arity mismatch → add `any()` for 7th arg
- Missing import for `eq()` in `OversightGateServiceTest` → add `import static org.mockito.ArgumentMatchers.eq;`
- `@RiskClassifier Instance<ActionRiskClassifier>` CDI ambiguity in app module → no `@RiskClassifier` beans present, `isUnsatisfied()` returns true; CDI does not fail on unsatisfied `Instance<T>` injection

- [ ] **Step 3: Confirm the commit log**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw log --oneline -8
```

Expected output (newest first):
```
<hash> test(app): update OversightGateDispatcherCdiTest comment — openclaw#30 shipped
<hash> feat(app): CommitmentTools.done() — Phase 2 gate wiring via OversightGateService.openGate()
<hash> test(casehub): fulfill() context path tests — DONE/DECLINE dispatch on gate resolution
<hash> feat(casehub): OversightGateService.openGate() — classifier composition + gate COMMAND dispatch
<hash> feat(casehub): OversightGateDispatcher — context-aware DONE/DECLINE dispatch
<hash> feat(casehub): GateDecision and GateContext types for Phase 2 gate wiring
```
