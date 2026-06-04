# Phase 2 Layer 3 Skills, casehub_block, casehub_delegate, and DONE Dispatch Fix

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `casehub_block` and `casehub_delegate` MCP tools, three Layer 3 SKILL.md files, confirm ActionRiskClassifier contract, and fix the STATUS-instead-of-DONE workaround in `OversightGateService.evaluate()`.

**Architecture:** Three independent work items on one branch. All tests are pure Mockito unit tests (no `@QuarkusTest`). `casehub_block` uses `@Transactional` directly on the tool method. The DONE dispatch fix uses `commitmentStore.findOpenByObligor()` — already injected — to locate the open COMMAND commitment and recover its correlationId without any in-memory state.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mockito, AssertJ, Quarkiverse MCP server (`io.quarkiverse.mcp.server.*`), Qhorus runtime (`io.casehub.qhorus.runtime.*`), Maven (`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode`).

**Spec:** `docs/superpowers/specs/2026-06-04-phase2-skills-done-dispatch-design.md`

---

## File Map

| File | Action | Issue |
|------|--------|-------|
| `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java` | Modify — add `block()` and `delegate()` | #23 |
| `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java` | Modify — add block/delegate tests | #23 |
| `skills/casehub-reject/SKILL.md` | Create | #23 |
| `skills/casehub-block/SKILL.md` | Create | #23 |
| `skills/casehub-delegate/SKILL.md` | Create | #23 |
| `skills/casehub-global/SKILL.md` | Modify — add casehub_block, casehub_delegate | #23 |
| `skills/README.md` | Modify — count 5→8, add three entries | #23 |
| `casehub/src/main/java/io/casehub/openclaw/casehub/ActionRiskClassifier.java` | Modify — javadoc only | #24 |
| `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java` | Modify — replace STATUS workaround | #16 |
| `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java` | Modify — update and add evaluate() tests | #16 |
| `ARC42STORIES.MD` | Modify — §8 and §9 entries | all |

---

## Task 1: Write failing tests for `casehub_block`

**Files:**
- Test: `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`

- [ ] **Step 1.1: Add the `casehub_block` test group at the end of `CommitmentToolsTest` (before the helpers)**

Add after the last `@Test` method, before `// ---- helpers ----`:

```java
// ---- casehub_block ----

@Test
void block_channelBacked_updatesExpiresAtAndDispatchesBlockedStatus() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    Instant originalDeadline = Instant.now().plus(1, ChronoUnit.HOURS);
    Instant blockedUntil = Instant.now().plus(24, ChronoUnit.HOURS);

    // Populate channelMap via commit()
    when(commitmentStore.findOpenByObligor(agentId, channelId))
            .thenReturn(List.of(commitment(correlationId, channelId, agentId, originalDeadline)));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(10L, channelId, agentId, MessageType.STATUS, correlationId));
    tools.commit(agentId, "Audit books", null, channelId.toString());
    clearInvocations(messageService, commitmentStore);

    // block() setup
    Commitment c = commitment(correlationId, channelId, agentId, originalDeadline);
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));
    when(commitmentStore.save(any())).thenAnswer(inv -> inv.getArgument(0));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(11L, channelId, agentId, MessageType.STATUS, correlationId));

    ToolResponse response = tools.block(agentId, correlationId, "Waiting for CFO sign-off", blockedUntil.toString());

    // save() called with updated expiresAt
    ArgumentCaptor<Commitment> saveCaptor = ArgumentCaptor.forClass(Commitment.class);
    verify(commitmentStore).save(saveCaptor.capture());
    assertThat(saveCaptor.getValue().expiresAt).isEqualTo(blockedUntil);

    // STATUS dispatched with "BLOCKED: " prefix
    ArgumentCaptor<MessageDispatch> dispatchCaptor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(dispatchCaptor.capture());
    MessageDispatch dispatched = dispatchCaptor.getValue();
    assertThat(dispatched.type()).isEqualTo(MessageType.STATUS);
    assertThat(dispatched.channelId()).isEqualTo(channelId);
    assertThat(dispatched.content()).startsWith("BLOCKED: ");
    assertThat(dispatched.content()).contains("CFO sign-off");

    assertThat(response.isError()).isFalse();
    assertThat(text(response)).contains("blocked");
    assertThat(text(response)).contains("newWatchdogDeadline");
}

@Test
void block_selfCommit_updatesExpiresAtWithoutDispatch() {
    String agentId = "home-agent";
    String correlationId = UUID.randomUUID().toString();
    Instant blockedUntil = Instant.now().plus(4, ChronoUnit.HOURS);

    // No channelMap entry — self-commit scenario
    Commitment c = commitment(correlationId, null, agentId, Instant.now().plus(1, ChronoUnit.HOURS));
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));
    when(commitmentStore.save(any())).thenAnswer(inv -> inv.getArgument(0));

    ToolResponse response = tools.block(agentId, correlationId, "Waiting for delivery", blockedUntil.toString());

    verify(commitmentStore).save(any());
    verify(messageService, never()).dispatch(any());
    assertThat(response.isError()).isFalse();
}

@Test
void block_unknownCommitmentId_returnsNotFound() {
    when(commitmentStore.findByCorrelationId("unknown")).thenReturn(Optional.empty());

    ToolResponse response = tools.block("agent", "unknown", "reason",
            Instant.now().plus(1, ChronoUnit.HOURS).toString());

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("COMMITMENT_NOT_FOUND");
    verify(commitmentStore, never()).save(any());
}

@Test
void block_terminalCommitment_returnsAlreadyClosed() {
    String agentId = "agent";
    String correlationId = UUID.randomUUID().toString();
    Commitment c = commitment(correlationId, null, agentId, Instant.now().plus(1, ChronoUnit.HOURS));
    c.state = CommitmentState.FULFILLED;
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));

    ToolResponse response = tools.block(agentId, correlationId, "reason",
            Instant.now().plus(1, ChronoUnit.HOURS).toString());

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("COMMITMENT_ALREADY_CLOSED");
    verify(commitmentStore, never()).save(any());
}

@Test
void block_wrongObligor_returnsUnauthorized() {
    String realObligor = "finance-agent";
    String wrongAgent = "intruder-agent";
    String correlationId = UUID.randomUUID().toString();
    Commitment c = commitment(correlationId, null, realObligor, Instant.now().plus(1, ChronoUnit.HOURS));
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));

    ToolResponse response = tools.block(wrongAgent, correlationId, "reason",
            Instant.now().plus(1, ChronoUnit.HOURS).toString());

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("COMMITMENT_UNAUTHORIZED");
    verify(commitmentStore, never()).save(any());
}

@Test
void block_invalidDeadlineFormat_returnsInvalidDeadline() {
    ToolResponse response = tools.block("agent", "any-id", "reason", "not-a-timestamp");

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("INVALID_DEADLINE");
}

@Test
void block_deadlineInPast_returnsDeadlineInPast() {
    String correlationId = UUID.randomUUID().toString();
    Commitment c = commitment(correlationId, null, "agent", Instant.now().plus(1, ChronoUnit.HOURS));
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));

    String pastTimestamp = Instant.now().minus(1, ChronoUnit.HOURS).toString();
    ToolResponse response = tools.block("agent", correlationId, "reason", pastTimestamp);

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("DEADLINE_IN_PAST");
}

@Test
void block_dispatchThrowsAfterSave_propagatesException() {
    // This verifies the @Transactional direction: if dispatch() throws after save(),
    // the exception propagates (which triggers JTA rollback of the save()).
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    when(commitmentStore.findOpenByObligor(agentId, channelId))
            .thenReturn(List.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(10L, channelId, agentId, MessageType.STATUS, correlationId));
    tools.commit(agentId, "task", null, channelId.toString());
    clearInvocations(messageService, commitmentStore);

    Commitment c = commitment(correlationId, channelId, agentId, Instant.now().plus(1, ChronoUnit.HOURS));
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.of(c));
    when(commitmentStore.save(any())).thenAnswer(inv -> inv.getArgument(0));
    when(messageService.dispatch(any())).thenThrow(new RuntimeException("channel unavailable"));

    assertThatThrownBy(() -> tools.block(agentId, correlationId, "blocked",
            Instant.now().plus(2, ChronoUnit.HOURS).toString()))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("channel unavailable");
    // JTA rolls back the save() — expiresAt returns to original. Verified by transaction
    // semantics; the unit test confirms exception propagation rather than silent swallow.
}
```

Also add to the imports at the top of CommitmentToolsTest:
```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

- [ ] **Step 1.2: Run the tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `block()` method does not exist on `CommitmentTools`.

---

## Task 2: Implement `casehub_block` in `CommitmentTools`

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`

- [ ] **Step 2.1: Add import for `@Transactional` and `CommitmentState`**

Add to the imports block (alphabetically):
```java
import jakarta.transaction.Transactional;
import io.casehub.qhorus.api.message.CommitmentState;
```

- [ ] **Step 2.2: Add the `block()` method after `escalate()` and before `// ---- helpers ----`**

```java
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

    UUID channelId = channelMap.get(commitmentId);

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

    if (channelId != null) {
        messageService.dispatch(MessageDispatch.builder()
                .channelId(channelId)
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
```

Also add `import java.time.Instant;` if not already present (check existing imports first — it may already be there from `selfCommit`).

- [ ] **Step 2.3: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All `CommitmentToolsTest` tests PASS, including the new `block_*` tests.

---

## Task 3: Write failing tests for `casehub_delegate`

**Files:**
- Test: `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`

- [ ] **Step 3.1: Add the `casehub_delegate` test group after the `casehub_block` tests**

```java
// ---- casehub_delegate ----

@Test
void delegate_dispatchesHandoffWithReasonAsContentAndRequiredToAgent() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    // Populate channelMap
    when(commitmentStore.findOpenByObligor(agentId, channelId))
            .thenReturn(List.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(10L, channelId, agentId, MessageType.STATUS, correlationId));
    tools.commit(agentId, "Audit", null, channelId.toString());
    clearInvocations(messageService);

    // delegate() looks up COMMAND message for inReplyTo
    when(messageService.findAllByCorrelationId(correlationId))
            .thenReturn(List.of(message(5L, channelId, MessageType.COMMAND, correlationId)));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(11L, channelId, agentId, MessageType.HANDOFF, correlationId));

    ToolResponse response = tools.delegate(agentId, correlationId,
            "Delegating to specialist", "tax-specialist-agent");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    MessageDispatch dispatched = captor.getValue();
    assertThat(dispatched.type()).isEqualTo(MessageType.HANDOFF);
    assertThat(dispatched.channelId()).isEqualTo(channelId);
    assertThat(dispatched.correlationId()).isEqualTo(correlationId);
    assertThat(dispatched.inReplyTo()).isEqualTo(5L);
    assertThat(dispatched.target()).isEqualTo("tax-specialist-agent");
    assertThat(dispatched.content()).isEqualTo("Delegating to specialist");

    assertThat(response.isError()).isFalse();
    assertThat(text(response)).contains("delegated");
    assertThat(text(response)).contains("tax-specialist-agent");
}

@Test
void delegate_noChannelMapEntry_returnsNotFound() {
    // No prior commit() — not in channelMap
    ToolResponse response = tools.delegate("agent", "unknown-id",
            "reason", "target-agent");

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("COMMITMENT_NOT_FOUND");
    verify(messageService, never()).dispatch(any());
}

@Test
void delegate_commandMessageNotFound_returnsCommandNotFound() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    when(commitmentStore.findOpenByObligor(agentId, channelId))
            .thenReturn(List.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(10L, channelId, agentId, MessageType.STATUS, correlationId));
    tools.commit(agentId, "Audit", null, channelId.toString());
    clearInvocations(messageService);

    // No COMMAND in history
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of());

    ToolResponse response = tools.delegate(agentId, correlationId,
            "Delegating to specialist", "tax-specialist-agent");

    assertThat(response.isError()).isTrue();
    assertThat(text(response)).contains("COMMAND_NOT_FOUND");
    verify(messageService, never()).dispatch(any());
}

@Test
void delegate_removesFromChannelMapAfterDispatch() {
    UUID channelId = UUID.randomUUID();
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    when(commitmentStore.findOpenByObligor(agentId, channelId))
            .thenReturn(List.of(commitment(correlationId, channelId, agentId,
                    Instant.now().plus(1, ChronoUnit.HOURS))));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(10L, channelId, agentId, MessageType.STATUS, correlationId));
    tools.commit(agentId, "Audit", null, channelId.toString());
    clearInvocations(messageService);

    when(messageService.findAllByCorrelationId(correlationId))
            .thenReturn(List.of(message(5L, channelId, MessageType.COMMAND, correlationId)));
    when(messageService.dispatch(any()))
            .thenReturn(dispatchResult(11L, channelId, agentId, MessageType.HANDOFF, correlationId));
    tools.delegate(agentId, correlationId, "Delegating", "other-agent");

    // After delegate, commitment is no longer in channelMap — done() uses selfCommit path
    when(commitmentStore.findByCorrelationId(correlationId)).thenReturn(Optional.empty());
    ToolResponse doneAfter = tools.done(agentId, correlationId, null);
    assertThat(doneAfter.isError()).isTrue(); // COMMITMENT_NOT_FOUND from selfCommit_done
}
```

- [ ] **Step 3.2: Run tests to confirm `delegate_*` tests fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `delegate_*` tests FAIL — `delegate()` method does not exist.

---

## Task 4: Implement `casehub_delegate` in `CommitmentTools`

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`

- [ ] **Step 4.1: Add `delegate()` method after `block()` and before `// ---- helpers ----`**

```java
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

    UUID channelId = channelMap.get(commitmentId);
    if (channelId == null) {
        return ToolResponse.error("COMMITMENT_NOT_FOUND: " + commitmentId);
    }

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

    channelMap.remove(commitmentId);

    return ToolResponse.success("""
            {"delegated": true, "delegatedTo": "%s"}
            """.formatted(toAgent).strip());
}
```

- [ ] **Step 4.2: Run all CommitmentToolsTest to verify all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -Dtest=CommitmentToolsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: ALL tests PASS.

---

## Task 5: Build app module and commit `#23` tool changes

**Files:** `app/` module

- [ ] **Step 5.1: Build the full app module**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app -am
```

Expected: BUILD SUCCESS.

- [ ] **Step 5.2: Commit the tool changes**

```bash
git add app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java
git add app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java
git commit -m "feat(casehub_block,casehub_delegate): add Layer 3 lifecycle MCP tools

casehub_block: extends Watchdog deadline via direct expiresAt update on
Commitment entity (@Transactional for save+dispatch atomicity). Guards:
COMMITMENT_NOT_FOUND, COMMITMENT_ALREADY_CLOSED, COMMITMENT_UNAUTHORIZED,
INVALID_DEADLINE, DEADLINE_IN_PAST.

casehub_delegate: dispatches HANDOFF with required toAgent. Structurally
identical to casehub_escalate but toAgent is required — makes intentional
delegation machine-readable vs authority-exceeded escalation in audit trail.

Closes #23 (tools only; SKILL.md files follow)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Write the three SKILL.md files and update casehub-global + README

**Files:**
- Create: `skills/casehub-reject/SKILL.md`
- Create: `skills/casehub-block/SKILL.md`
- Create: `skills/casehub-delegate/SKILL.md`
- Modify: `skills/casehub-global/SKILL.md`
- Modify: `skills/README.md`

- [ ] **Step 6.1: Create `skills/casehub-reject/SKILL.md`**

```yaml
---
name: casehub-reject
description: Decline a tracked CaseHub commitment you cannot complete
version: 1.0.0
triggers:
  - "reject this task"
  - "I can't complete this"
  - "decline this commitment"
  - "this isn't possible"
  - "I won't be able to do this"
tools:
  - casehub_reject
permissions: []
---

You are declining a tracked CaseHub commitment with a recorded reason.

## Procedure

1. **Confirm you have an active `commitmentId`.** If not: tell the user "I don't have
   an active tracked commitment for this task — was it created with `casehub_commit`
   or `casehub_create_workitem`?" Do not call `casehub_reject` without a valid ID.

2. **Extract `reason`** — required; must explain why the task cannot be completed.

3. **Call the tool:**

```
casehub_reject(
  agentId    = YOUR_AGENT_ID,
  commitmentId = COMMITMENT_ID,
  reason     = REASON
)
```

4. **Handle errors:**
   - `COMMITMENT_NOT_FOUND`: the commitment does not exist or is from a previous session —
     do not retry; report to the user.
   - `COMMITMENT_ALREADY_CLOSED`: commitment is already resolved — report to the user.

5. **On success:** report `{"declined": true}` — the obligation is discharged and the
   Watchdog is disarmed.
```

- [ ] **Step 6.2: Create `skills/casehub-block/SKILL.md`**

```yaml
---
name: casehub-block
description: Temporarily block a CaseHub commitment pending an external dependency, extending the Watchdog deadline
version: 1.0.0
triggers:
  - "I'm blocked on X"
  - "waiting for X to resolve"
  - "can't proceed until Y"
  - "on hold pending Z"
  - "blocked by X"
tools:
  - casehub_block
  - casehub_checkpoint
permissions: []
---

You are temporarily blocking a tracked CaseHub commitment because an external
dependency prevents progress. This extends the Watchdog deadline so the commitment
does not escalate prematurely.

## Procedure

1. **Confirm you have an active `commitmentId`.** If not, tell the user.

2. **Identify the blocker** — what specifically is preventing progress (`reason`).

3. **Estimate `blockedUntil`** — a future ISO-8601 timestamp. Ensure it is in the future;
   a past timestamp returns `DEADLINE_IN_PAST`.
   - If the resolution time is known → use it.
   - If the blocker is indefinite (you have no idea when it resolves) → **consider
     `casehub_escalate` instead** to hand off to whoever can unblock this.
   - If the window is unknown but finite → extend by a reasonable estimate (1 hour,
     4 hours, 1 day) and explain the estimate to the user.

4. **Call the tool:**

```
casehub_block(
  agentId      = YOUR_AGENT_ID,
  commitmentId = COMMITMENT_ID,
  reason       = BLOCKER_DESCRIPTION,
  blockedUntil = ISO_FUTURE_TIMESTAMP
)
```

5. **Confirm `newWatchdogDeadline`** to the user.

6. **When the blocker resolves:** call
   `casehub_checkpoint(agentId, commitmentId, "UNBLOCKED: <note>")` to resume
   normal monitoring.

## Restart recovery

If the Quarkus service restarts while you are blocked, the in-memory channel binding
for this commitment is lost. Calling `casehub_checkpoint` after a restart will return
`COMMITMENT_NOT_FOUND` — do not retry it.

For channel-backed commitments: calling `casehub_done` after a restart closes the
commitment in the store, but no DONE is dispatched to the work channel. If audit trail
completeness matters, let the Watchdog handle escalation rather than calling
`casehub_done` after a restart.
```

- [ ] **Step 6.3: Create `skills/casehub-delegate/SKILL.md`**

```yaml
---
name: casehub-delegate
description: Intentionally transfer a tracked CaseHub commitment to a named agent or person
version: 1.0.0
triggers:
  - "delegate this to X"
  - "hand this off to X"
  - "give this to [agent/person]"
  - "transfer this to X"
  - "this should go to X"
tools:
  - casehub_delegate
permissions: []
---

You are intentionally transferring a tracked CaseHub commitment to a named agent or
person. Use this when delegating responsibility — NOT when escalating because a task
exceeds your authority or capability (use `casehub_escalate` for that).

## Procedure

1. **Confirm you have an active `commitmentId`.** If not, tell the user.

2. **Identify `toAgent`** — the target agent ID or human identifier. Required.

3. **Clarify `reason`** — why you are delegating. Recorded in the Qhorus ledger.

4. **Call the tool:**

```
casehub_delegate(
  agentId      = YOUR_AGENT_ID,
  commitmentId = COMMITMENT_ID,
  reason       = DELEGATION_REASON,
  toAgent      = TARGET_AGENT_ID
)
```

5. **Handle errors:**
   - `COMMITMENT_NOT_FOUND`: the commitment is not tracked in this session — it may
     already be closed or was committed in a previous session. Do not retry.
   - `COMMAND_NOT_FOUND`: the original command message cannot be located (possible
     service restart). The Watchdog will handle escalation — do not retry
     `casehub_delegate`.

6. **On success:** report "Commitment transferred to [target]. Their Watchdog is now
   running. Your obligation is discharged."
```

- [ ] **Step 6.4: Update `skills/casehub-global/SKILL.md` — add two tools to front-matter and description**

In the front-matter `tools:` list, add after `casehub_escalate`:
```yaml
  - casehub_block
  - casehub_delegate
```

In the "Available tools" section of the instruction block, add after the `casehub_escalate` bullet:
```
- `casehub_block(agentId, commitmentId, reason, blockedUntil)` — extend Watchdog deadline while blocked on an external dependency; call casehub_checkpoint("UNBLOCKED: ...") when resolved
- `casehub_delegate(agentId, commitmentId, reason, toAgent)` — transfer a commitment to a named agent or person (use for intentional delegation, not authority escalation)
```

In the "When to call these explicitly" section, add:
```
Call `casehub_block` when you cannot proceed due to an external dependency — extend the deadline rather than letting the Watchdog fire prematurely.
Call `casehub_delegate` when intentionally transferring responsibility to a specific named party.
```

- [ ] **Step 6.5: Update `skills/README.md` — update skill count and add three entries**

Change the skill count from 5 to 8 wherever it appears.

Add entries for the three new skills alongside the existing five. Follow the same format as existing entries (name, one-line description, triggers example, install command).

---

## Task 7: Commit `#23` SKILL.md changes

- [ ] **Step 7.1: Stage and commit**

```bash
git add skills/casehub-reject/SKILL.md skills/casehub-block/SKILL.md skills/casehub-delegate/SKILL.md
git add skills/casehub-global/SKILL.md skills/README.md
git commit -m "feat(skills): Layer 3 lifecycle SKILL.md — reject, block, delegate

Three new stateless Layer 3 skills for explicit user-initiated commitment lifecycle:
- casehub-reject: wrapper for casehub_reject (DECLINE speech act)
- casehub-block: wrapper for casehub_block with blocker duration guidance
- casehub-delegate: wrapper for casehub_delegate (intentional HANDOFF, toAgent required)

Updates casehub-global to list new tools. Updates README skill count (5→8).

Closes #23

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: ActionRiskClassifier javadoc confirmation (`#24`)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ActionRiskClassifier.java`

- [ ] **Step 8.1: Update the javadoc**

Replace the existing javadoc on `ActionRiskClassifier` with:

```java
/**
 * Classifies the risk of a proposed worker action, deciding whether autonomous
 * execution or a human oversight gate is required.
 *
 * <p><b>Phase 1 ({@link DefaultActionRiskClassifier}):</b> always {@link RiskDecision.Autonomous}.
 * No risk rules are configured — all actions proceed without oversight.
 *
 * <p>This is a local placeholder for the {@code ActionRiskClassifier} SPI proposed for
 * {@code casehub-engine-api} (casehubio/engine#402). The local contract has been verified
 * identical to the engine#402 proposal as of 2026-06-04: same method signature, same type
 * names, same {@code @Alternative @Priority(1)} override pattern. When engine#402 ships,
 * migration is a pure import swap — no code changes beyond the import statement.
 *
 * <p>Override the default bean with {@code @Alternative @Priority(1)}.
 */
public interface ActionRiskClassifier {
    RiskDecision classify(PlannedAction action);
}
```

- [ ] **Step 8.2: Build casehub module to confirm no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl casehub -am
```

Expected: BUILD SUCCESS.

- [ ] **Step 8.3: Commit**

```bash
git add casehub/src/main/java/io/casehub/openclaw/casehub/ActionRiskClassifier.java
git commit -m "docs(ActionRiskClassifier): confirm contract identical to engine#402 as of 2026-06-04

Import-swap migration is ready when engine#402 ships — no code changes beyond import.

Closes #24

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: Write failing tests for the DONE dispatch fix (`#16`)

**Files:**
- Test: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`

- [ ] **Step 9.1: Add a commitment helper that includes `messageType` to `OversightGateServiceTest`**

Add to the `// ── helpers ───` section:

```java
private Commitment openCommandCommitment(UUID channelId, String agentId, String correlationId) {
    Commitment c = new Commitment();
    c.id = UUID.randomUUID();
    c.correlationId = correlationId;
    c.channelId = channelId;
    c.obligor = agentId;
    c.messageType = MessageType.COMMAND;
    c.state = CommitmentState.OPEN;
    return c;
}
```

Also add to imports:
```java
import io.casehub.qhorus.api.message.CommitmentState;
import java.util.Collections;
```

- [ ] **Step 9.2: Update the existing `evaluate_autonomous_dispatchesStatusToWorkChannel` test**

This test currently expects STATUS (the workaround). With the fix, it should dispatch DONE when a COMMAND commitment is found. Rename it and update expectations:

Replace:
```java
@Test
void evaluate_autonomous_dispatchesStatusToWorkChannel() {
    service.evaluate(workChannelId, "finance-agent", "Analysis complete.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    MessageDispatch dispatched = captor.getValue();
    assertThat(dispatched.channelId()).isEqualTo(workChannelId);
    assertThat(dispatched.type()).isEqualTo(MessageType.STATUS);
    assertThat(dispatched.sender()).isEqualTo("finance-agent");
    assertThat(dispatched.content()).isEqualTo("Analysis complete.");
    assertThat(dispatched.actorType()).isEqualTo(ActorType.AGENT);
}
```

With:
```java
@Test
void evaluate_autonomous_withOpenCommandCommitment_dispatchesSpeechActTypeWithInReplyTo() {
    // setUp() returns MessageType.DONE from speechActClassifier
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();
    long commandMessageId = 42L;

    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    Message commandMsg = new Message();
    commandMsg.id = commandMessageId;
    commandMsg.messageType = MessageType.COMMAND;
    commandMsg.correlationId = correlationId;
    when(messageService.findAllByCorrelationId(correlationId))
            .thenReturn(List.of(commandMsg));

    service.evaluate(workChannelId, agentId, "Analysis complete.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    MessageDispatch dispatched = captor.getValue();
    assertThat(dispatched.channelId()).isEqualTo(workChannelId);
    assertThat(dispatched.type()).isEqualTo(MessageType.DONE);
    assertThat(dispatched.sender()).isEqualTo(agentId);
    assertThat(dispatched.content()).isEqualTo("Analysis complete.");
    assertThat(dispatched.actorType()).isEqualTo(ActorType.AGENT);
    assertThat(dispatched.inReplyTo()).isEqualTo(commandMessageId);
    assertThat(dispatched.correlationId()).isEqualTo(correlationId);
}

@Test
void evaluate_autonomous_noOpenCommitment_dispatchesStatusAndLogsWarn() {
    // hadCommitment=false: Watchdog may have expired it — WARN, not ERROR
    when(commitmentStore.findOpenByObligor("finance-agent", workChannelId))
            .thenReturn(Collections.emptyList());

    service.evaluate(workChannelId, "finance-agent", "Analysis complete.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
    // Log level (WARN) is verified by the absence of an exception and by manual log inspection.
    // The Watchdog escalation path handles the terminal commitment.
}

@Test
void evaluate_autonomous_commitmentFoundButCommandMessageMissing_dispatchesStatusAndLogsError() {
    // hadCommitment=true but resolveCommandMessageId returns null — data inconsistency
    String agentId = "finance-agent";
    String correlationId = UUID.randomUUID().toString();

    when(commitmentStore.findOpenByObligor(agentId, workChannelId))
            .thenReturn(List.of(openCommandCommitment(workChannelId, agentId, correlationId)));
    // findAllByCorrelationId returns no COMMAND type
    when(messageService.findAllByCorrelationId(correlationId)).thenReturn(List.of());

    service.evaluate(workChannelId, agentId, "Analysis complete.");

    ArgumentCaptor<MessageDispatch> captor = ArgumentCaptor.forClass(MessageDispatch.class);
    verify(messageService).dispatch(captor.capture());
    assertThat(captor.getValue().type()).isEqualTo(MessageType.STATUS);
    // Log level (ERROR) is verified by manual log inspection / operator alerting setup.
}
```

- [ ] **Step 9.3: Run the updated tests to confirm they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `evaluate_autonomous_withOpenCommandCommitment_*` fails — evaluate() still dispatches STATUS (old behavior).

---

## Task 10: Implement the DONE dispatch fix in `OversightGateService.evaluate()`

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`

- [ ] **Step 10.1: Add missing imports**

Add to the imports (verify they are not already present):
```java
import java.util.List;
```

- [ ] **Step 10.2: Add the `resolveCommandMessageId` private helper**

Add after the `parseApproval()` method:

```java
private Long resolveCommandMessageId(String correlationId) {
    if (correlationId == null) return null;
    return messageService.findAllByCorrelationId(correlationId).stream()
            .filter(m -> m.messageType == MessageType.COMMAND)
            .mapToLong(m -> m.id).boxed().findFirst().orElse(null);
}
```

- [ ] **Step 10.3: Replace the STATUS workaround in `evaluate()`**

Locate this block in `evaluate()`:
```java
if (decision instanceof RiskDecision.Autonomous) {
    // DONE/DECLINE/FAILURE/RESPONSE require inReplyTo — not available on autonomous path.
    // Fall back to STATUS; tracked in openclaw#16.
    MessageType dispatchType = requiresReplyFields(messageType) ? MessageType.STATUS : messageType;
    messageService.dispatch(MessageDispatch.builder()
            .channelId(workChannelId)
            .sender(agentId)
            .type(dispatchType)
            .content(output != null ? output : "")
            .actorType(ActorType.AGENT)
            .build());
    return;
}
```

Replace with:
```java
if (decision instanceof RiskDecision.Autonomous) {
    List<io.casehub.qhorus.runtime.message.Commitment> open =
            commitmentStore.findOpenByObligor(agentId, workChannelId);
    boolean hadCommitment = !open.isEmpty();
    String correlationId = open.stream()
            .filter(c -> c.messageType == MessageType.COMMAND)
            .map(c -> c.correlationId)
            .findFirst()
            .orElse(null);

    Long commandMessageId = resolveCommandMessageId(correlationId);

    MessageDispatch.Builder builder = MessageDispatch.builder()
            .channelId(workChannelId)
            .sender(agentId)
            .content(output != null ? output : "")
            .actorType(ActorType.AGENT);

    if (commandMessageId != null && correlationId != null) {
        builder.type(messageType).inReplyTo(commandMessageId).correlationId(correlationId);
    } else {
        builder.type(MessageType.STATUS);
        if (hadCommitment) {
            log.errorf("Open COMMAND commitment found for agentId=%s on channel=%s but "
                    + "COMMAND message lookup failed — Commitment will not be fulfilled; "
                    + "Watchdog will escalate. Operator attention required.",
                    agentId, workChannelId);
        } else {
            log.warnf("No open COMMAND commitment for agentId=%s on channel=%s — "
                    + "Watchdog may have expired it during agent execution. Dispatching STATUS.",
                    agentId, workChannelId);
        }
    }
    messageService.dispatch(builder.build());
    return;
}
```

Also add the `io.casehub.qhorus.runtime.message.Commitment` import (may already be imported — check the existing import block):
```java
import io.casehub.qhorus.runtime.message.Commitment;
```

And remove `requiresReplyFields()` if it is no longer used anywhere else in the class (search for its callers first).

- [ ] **Step 10.4: Run the tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=OversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: ALL `OversightGateServiceTest` tests PASS.

- [ ] **Step 10.5: Run full casehub module build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl casehub -am
```

Expected: BUILD SUCCESS.

---

## Task 11: Commit `#16` DONE dispatch fix

- [ ] **Step 11.1: Stage and commit**

```bash
git add casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java
git add casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java
git commit -m "fix(evaluate): dispatch DONE (not STATUS) on autonomous path via Qhorus-native query

Replaces in-memory registry approach with commitmentStore.findOpenByObligor() —
restart-resilient, no turn-scoped state in session registry.

Two STATUS fallback paths distinguished:
- hadCommitment=false (Watchdog expired it) → WARN
- hadCommitment=true, messageId missing (data gap) → ERROR

Closes #16

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 12: Update ARC42STORIES.MD

**Files:**
- Modify: `ARC42STORIES.MD`

- [ ] **Step 12.1: Add §8 crosscutting entry for new MCP tool surface**

In `## §8 Crosscutting Concepts`, under `### Protocol References` or after the existing anti-patterns, add:

```markdown
### New MCP Tool Surface (Epic 8 Phase 2)

`casehub_block` and `casehub_delegate` extend the Layer 0 MCP endpoint:

- `casehub_block` uses `@Transactional` directly on the tool method (unlike other tool
  methods that delegate to already-`@Transactional` service methods). This is safe because
  `block()` contains no try/catch — exceptions propagate normally, so there is no
  rollback-only problem. Both `commitmentStore.save()` and `messageService.dispatch()`
  participate in the same JPA/JTA resource.

- `casehub_delegate` requires `toAgent` (non-optional, unlike `casehub_escalate`). This
  makes intentional delegation machine-readable vs authority-exceeded escalation in the
  Qhorus ledger — HANDOFF records are structurally distinguishable by which tool was called.

- The direct `expiresAt` mutation in `casehub_block` bypasses `CommitmentService` because
  no `extendDeadline()` operation exists. This is documented as a workaround pending
  `CommitmentService.extendDeadline()` in casehub-qhorus (filed).
```

- [ ] **Step 12.2: Add §9 chapter entry for this epic work**

In `## §9 Journeys and Chapters`, `### §9.3 Chapter Entries`, add a chapter entry for Epic 8 Phase 2 work following the established format of existing entries.

- [ ] **Step 12.3: Add §9 DONE dispatch note in the relevant runtime scenario**

In `### Scenario 1 — COMMAND → OpenClaw → DONE round trip`, update or add a note:

```markdown
As of Epic 8 Phase 2, `OversightGateService.evaluate()` dispatches the correct speech act
type (DONE, STATUS, etc.) with `inReplyTo` populated via `commitmentStore.findOpenByObligor()`.
The prior STATUS workaround (openclaw#16) is resolved.
```

- [ ] **Step 12.4: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs(arc42): Epic 8 Phase 2 — MCP tool surface and DONE dispatch

Closes #23 (arc42 coverage)

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 13: File the qhorus deferred issue

- [ ] **Step 13.1: File the qhorus issue**

```bash
gh issue create --repo casehubio/qhorus \
  --title "feat: CommitmentService.extendDeadline(correlationId, newDeadline) for proper block() support" \
  --body "## Context

casehub-openclaw's \`casehub_block\` MCP tool (casehubio/openclaw#23) needs to extend a Commitment's
\`expiresAt\` deadline without changing its lifecycle state. Currently this is done by directly
mutating the \`Commitment\` entity via \`CommitmentStore.save()\`, bypassing the service layer.

## Requested API

Add to \`CommitmentService\`:

\`\`\`java
/**
 * Extends the Watchdog deadline of an OPEN or ACKNOWLEDGED commitment.
 * No-op if the commitment does not exist or is in a terminal state.
 */
@Transactional
public Optional<Commitment> extendDeadline(String correlationId, Instant newDeadline) {
    return store.findByCorrelationId(correlationId)
            .filter(c -> !c.state.isTerminal())
            .map(c -> {
                c.expiresAt = newDeadline;
                return store.save(c);
            });
}
\`\`\`

## Why

The state machine manages state transitions — not the deadline field. But keeping the mutation
in the service layer ensures it goes through proper transactional and observability hooks if
those are ever added to \`CommitmentService\`.

The long-term answer is a \`SUSPENDED\` state that \`expireOverdue()\` skips entirely, but
\`extendDeadline()\` is the correct near-term fix."
```

---

## Self-Review Checklist

After completing all tasks, verify:

- [ ] `CommitmentTools.block()` has `@Transactional` annotation
- [ ] `CommitmentTools.delegate()` uses `findCommandMessageId()` (existing private helper) — not duplicated
- [ ] `OversightGateService.evaluate()` no longer contains `requiresReplyFields()` call (method can be removed if unused)
- [ ] All three SKILL.md files exist in their directories
- [ ] `casehub-global/SKILL.md` front-matter `tools:` list includes `casehub_block` and `casehub_delegate`
- [ ] `skills/README.md` count updated to 8
- [ ] `ActionRiskClassifier.java` javadoc mentions "verified identical to engine#402 as of 2026-06-04"
- [ ] Git log shows four commits, each with a `Refs #N` or `Closes #N` for #23, #24, #16
- [ ] Full build passes: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
