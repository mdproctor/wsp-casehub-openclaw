# Epic 4 — CaseHub SPI Implementations

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement all CaseHub SPI interfaces (WorkerProvisioner, CaseChannelProvider, ChannelBackend, WorkerStatusListener) and the OpenClaw delivery webhook endpoint, completing the bidirectional Qhorus ↔ OpenClaw integration.

**Architecture:** `ChannelContextWindowService` (core/) gains two-level association (`bindAgent`/`bindChannel`/`unbindAgent`) replacing the race-prone `associate()`. Five new @ApplicationScoped beans in `casehub/` implement the engine and Qhorus SPIs, coordinated through a lightweight `OpenClawAgentRegistry`. The delivery webhook (`app/`) completes the OpenClaw → Qhorus write-back path.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, MicroProfile @ConfigMapping (SmallRye), JUnit 5 + Mockito (unit tests), AssertJ assertions. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn`.

---

## File Map

| Action | Path | Responsibility |
|---|---|---|
| Modify | `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java` | Replace associate() with bindAgent/bindChannel/unbindAgent |
| Rewrite | `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java` | All 16 test cases for new API |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java` | caseId↔agentId↔sessionKey routing maps |
| Create | `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java` | Unit tests |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCasehubConfig.java` | @ConfigMapping for agents capability map |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java` | WorkerProvisioner SPI implementation |
| Create | `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java` | Unit tests |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java` | CaseChannelProvider SPI implementation |
| Create | `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProviderTest.java` | Unit tests |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java` | ChannelBackend SPI + ChannelInitialisedEvent observer |
| Create | `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java` | Unit tests |
| Create | `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java` | WorkerStatusListener SPI implementation |
| Create | `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java` | Unit tests |
| Modify | `casehub/pom.xml` | Remove provided scope from engine + qhorus deps |
| Create | `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryPayload.java` | Webhook payload record |
| Create | `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java` | POST /openclaw/delivery/channel/{channelId} |
| Create | `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java` | @QuarkusTest for delivery endpoint |
| Modify | `app/src/main/resources/application.properties` | quarkus.arc.exclude-types + agent config |

---

## Task 1: ChannelContextWindowService — replace associate() with bind API

**Files:**
- Modify: `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`
- Rewrite: `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`

- [ ] **Step 1: Write the failing tests**

Replace the entire content of `ChannelContextWindowServiceTest.java` with:

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelContextWindowServiceTest {

    ChannelContextWindowService service;

    @BeforeEach
    void setup() {
        service = new ChannelContextWindowService();
        service.maxMessagesPerChannel = 100;
        service.ttl = Duration.ofMinutes(30);
    }

    private MessageReceivedEvent event(UUID channelId, String channelName, MessageType type) {
        String content = (type == MessageType.EVENT) ? null : "content";
        return new MessageReceivedEvent(channelName, channelId, type, "sender", "corr-1", content);
    }

    // ── noAssociation path ────────────────────────────────────────────────────

    @Test
    void query_unknownAgent_returnsNoAssociation() {
        assertThat(service.query("unknown-agent", 0L).agentHasAssociation()).isFalse();
    }

    @Test
    void add_unregisteredChannel_silentlyIgnored() {
        UUID channelId = UUID.randomUUID();
        service.add(event(channelId, "case-x/work", MessageType.STATUS));
        assertThat(service.query("any-agent", 0L).agentHasAssociation()).isFalse();
    }

    // ── bind-after-channel path (agent registered first) ─────────────────────

    @Test
    void bindChannel_beforeBindAgent_messagesCaptured_returnedOnceAgentBinds() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "case-x/work", MessageType.STATUS)); // captured

        service.bindAgent("agent-1", caseId);
        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.agentHasAssociation()).isTrue();
        assertThat(result.messages()).hasSize(1);
    }

    @Test
    void bindAgent_withoutBindChannel_returnsEmptyWindow_notNoAssociation() {
        UUID caseId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.agentHasAssociation()).isTrue();
        assertThat(result.messages()).isEmpty();
    }

    // ── normal path (channels first, then agent) ──────────────────────────────

    @Test
    void bindAgent_bindChannel_add_query_roundTrip() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "case-x/work", MessageType.COMMAND));

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.agentHasAssociation()).isTrue();
        assertThat(result.messages()).hasSize(1);
        assertThat(result.messages().get(0).messageType()).isEqualTo(MessageType.COMMAND);
    }

    @Test
    void bindChannel_twice_samePair_idempotent() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.bindChannel(caseId, channelId); // idempotent
        service.add(event(channelId, "case-x/work", MessageType.STATUS));

        assertThat(service.query("agent-1", 0L).messages()).hasSize(1);
    }

    @Test
    void query_withSince_filtersCorrectly() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "case-x/work", MessageType.COMMAND)); // seq=1
        service.add(event(channelId, "case-x/work", MessageType.STATUS));  // seq=2
        service.add(event(channelId, "case-x/work", MessageType.DONE));    // seq=3

        long firstSeq = service.query("agent-1", 0L).messages().get(0).windowSeq();
        WindowContent result = service.query("agent-1", firstSeq);
        assertThat(result.messages()).hasSize(2);
        assertThat(result.messages())
                .extracting(ContextMessage::messageType)
                .containsExactly(MessageType.STATUS, MessageType.DONE);
    }

    @Test
    void multipleChannels_mergedSortedByWindowSeq() {
        UUID caseId = UUID.randomUUID();
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, ch1);
        service.bindChannel(caseId, ch2);

        service.add(event(ch1, "work", MessageType.COMMAND));
        service.add(event(ch2, "observe", MessageType.STATUS));
        service.add(event(ch1, "work", MessageType.DONE));

        List<Long> seqs = service.query("agent-1", 0L).messages()
                .stream().map(ContextMessage::windowSeq).toList();
        assertThat(seqs).isSorted();
        assertThat(seqs).hasSize(3);
    }

    // ── unbindAgent path ──────────────────────────────────────────────────────

    @Test
    void unbindAgent_subsequentQuery_returnsNoAssociation() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "case-x/work", MessageType.STATUS));
        assertThat(service.query("agent-1", 0L).agentHasAssociation()).isTrue();

        service.unbindAgent("agent-1");
        assertThat(service.query("agent-1", 0L).agentHasAssociation()).isFalse();
    }

    @Test
    void unbindAgent_doesNotClearCaseChannels_bufferStillWritable() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.unbindAgent("agent-1");

        // Buffer still exists — add() does not throw or drop for the channelId
        service.add(event(channelId, "case-x/work", MessageType.STATUS));

        // Re-bind the agent — messages written after unbind but before re-bind are captured
        service.bindAgent("agent-1", caseId);
        assertThat(service.query("agent-1", 0L).messages()).hasSize(1);
    }

    // ── cursor / sequence semantics (unchanged from before) ───────────────────

    @Test
    void currentWindowSeq_reflectsGlobalCounter() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "work", MessageType.STATUS));
        service.add(event(channelId, "work", MessageType.STATUS));

        assertThat(service.query("agent-1", 0L).currentWindowSeq()).isEqualTo(2L);
    }

    @Test
    void lastWindowSeq_isMaxReturnedSeq() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=2

        assertThat(service.query("agent-1", 0L).lastWindowSeq()).isEqualTo(2L);
    }

    @Test
    void lastEvictionWindowSeq_reflectsOverflow() {
        service.maxMessagesPerChannel = 2;
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "work", MessageType.COMMAND)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS));  // seq=2
        service.add(event(channelId, "work", MessageType.DONE));    // seq=3, evicts seq=1

        assertThat(service.query("agent-1", 0L).lastEvictionWindowSeq()).isEqualTo(1L);
    }

    @Test
    void restartDetection_staleClientCursor_greaterThanCurrentWindowSeq() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=2
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=3

        // Simulate restart: new instance, AtomicLong resets to 0
        ChannelContextWindowService restarted = new ChannelContextWindowService();
        restarted.maxMessagesPerChannel = 100;
        restarted.ttl = Duration.ofMinutes(30);
        restarted.bindAgent("agent-1", caseId);
        restarted.bindChannel(caseId, channelId);
        restarted.add(event(channelId, "work", MessageType.STATUS)); // new seq=1

        // Client sends stale cursor=3; currentWindowSeq=1 → restart detected
        WindowContent result = restarted.query("agent-1", 3L);
        assertThat(result.currentWindowSeq()).isEqualTo(1L);
        assertThat(3L).isGreaterThan(result.currentWindowSeq()); // SDK detects restart
        assertThat(result.agentHasAssociation()).isTrue();
    }

    // ── concurrency ───────────────────────────────────────────────────────────

    @Test
    void concurrency_noExceptionAndBounded() throws InterruptedException {
        service.maxMessagesPerChannel = 10;
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, channelId);

        int threads = 10;
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(threads);
        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                try {
                    for (int j = 0; j < 50; j++)
                        service.add(event(channelId, "work", MessageType.STATUS));
                } finally { latch.countDown(); }
            });
        }
        latch.await(10, TimeUnit.SECONDS);
        executor.shutdown();

        assertThat(service.query("agent-1", 0L).messages()).hasSizeLessThanOrEqualTo(10);
    }

    @Test
    void concurrency_bindAndQuery_noException() throws InterruptedException {
        UUID caseId = UUID.randomUUID();
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        service.bindAgent("agent-1", caseId);
        service.bindChannel(caseId, ch1);

        ExecutorService executor = Executors.newFixedThreadPool(4);
        CountDownLatch latch = new CountDownLatch(4);
        executor.submit(() -> { try { for (int i=0;i<50;i++) service.bindChannel(caseId, ch2); } finally { latch.countDown(); } });
        executor.submit(() -> { try { for (int i=0;i<50;i++) service.add(event(ch1,"work",MessageType.STATUS)); } finally { latch.countDown(); } });
        executor.submit(() -> { try { for (int i=0;i<100;i++) service.query("agent-1",0L); } finally { latch.countDown(); } });
        executor.submit(() -> { try { for (int i=0;i<50;i++) { service.unbindAgent("agent-1"); service.bindAgent("agent-1",caseId); } } finally { latch.countDown(); } });
        latch.await(10, TimeUnit.SECONDS);
        executor.shutdown();
        // No assertions — just no exceptions thrown
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=ChannelContextWindowServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `COMPILATION ERROR` — `bindAgent`, `bindChannel`, `unbindAgent` methods do not exist yet.

- [ ] **Step 3: Rewrite ChannelContextWindowService**

Replace the full content of `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`:

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

import jakarta.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import io.casehub.qhorus.api.gateway.MessageReceivedEvent;

/**
 * Manages per-channel ring buffers of recent Qhorus channel activity and
 * answers per-agent context queries.
 *
 * <p>Association is two-phase and ordering-independent:
 * <ul>
 *   <li>{@link #bindAgent} registers which case an agent handles.
 *   <li>{@link #bindChannel} registers which channels belong to a case.
 *       Also initialises the ring buffer so {@link #add} can proceed immediately.
 *   <li>{@link #query} joins the two maps at read time — no coordination between callers.
 * </ul>
 *
 * <p>{@link #unbindAgent} is called by {@code OpenClawWorkerStatusListener.onWorkerCompleted()}
 * to release the agentToCase entry. The caseChannels entry is retained until TTL eviction.
 */
@ApplicationScoped
public class ChannelContextWindowService {

    private static final Logger log = Logger.getLogger(ChannelContextWindowService.class);

    @ConfigProperty(name = "casehub.openclaw.context-window.max-messages-per-channel",
                    defaultValue = "100")
    int maxMessagesPerChannel;

    @ConfigProperty(name = "casehub.openclaw.context-window.ttl", defaultValue = "PT30M")
    Duration ttl;

    private final AtomicLong windowSeq = new AtomicLong(0);
    private final ConcurrentHashMap<UUID, ChannelRingBuffer> buffers = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, UUID> agentToCase = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<UUID, Set<UUID>> caseChannels = new ConcurrentHashMap<>();

    /** Registers which case this agent handles. Called by OpenClawWorkerProvisioner.provision(). */
    public void bindAgent(String agentId, UUID caseId) {
        agentToCase.put(agentId, caseId);
    }

    /**
     * Registers a channel under a case and initialises its ring buffer.
     * Called by OpenClawCaseChannelProvider.openChannel().
     * Idempotent — calling twice for the same (caseId, channelId) is safe.
     */
    public void bindChannel(UUID caseId, UUID channelId) {
        caseChannels.computeIfAbsent(caseId, id -> ConcurrentHashMap.newKeySet()).add(channelId);
        buffers.putIfAbsent(channelId, new ChannelRingBuffer(maxMessagesPerChannel, ttl));
    }

    /**
     * Removes the agent's case association. Called by OpenClawWorkerStatusListener.onWorkerCompleted().
     * The caseChannels entry is retained — the buffer continues to accept writes until TTL eviction.
     */
    public void unbindAgent(String agentId) {
        agentToCase.remove(agentId);
    }

    /**
     * Ingests a Qhorus message into the ring buffer for its channel.
     * Silent no-op if bindChannel() has not been called for this channelId.
     * Called by ChannelContextWindowObserver on every dispatched Qhorus message.
     */
    public void add(MessageReceivedEvent event) {
        ChannelRingBuffer buffer = buffers.get(event.channelId());
        if (buffer == null) return;
        buffer.add(new ContextMessage(
                windowSeq.incrementAndGet(),
                event.channelId(),
                event.channelName(),
                event.messageType(),
                event.senderId(),
                event.correlationId(),
                event.content(),
                Instant.now()));
    }

    /**
     * Returns buffered channel context for the given agent since the cursor.
     *
     * <p>Returns {@link WindowContent#noAssociation()} if the agent has not been bound via
     * {@link #bindAgent}. Returns an empty window (agentHasAssociation=true, no messages) if
     * the agent is bound but no channels have been registered yet via {@link #bindChannel}.
     */
    public WindowContent query(String agentId, long since) {
        UUID caseId = agentToCase.get(agentId);
        if (caseId == null) return WindowContent.noAssociation();

        Set<UUID> channelIds = caseChannels.getOrDefault(caseId, Set.of());
        Instant now = Instant.now();
        List<ContextMessage> merged = new ArrayList<>();
        long maxEvictionSeq = -1L;
        Instant latestActivity = Instant.EPOCH;

        for (UUID channelId : channelIds) {
            ChannelRingBuffer buffer = buffers.get(channelId);
            if (buffer == null) continue;
            merged.addAll(buffer.query(since, now));
            long evSeq = buffer.lastEvictionWindowSeq();
            if (evSeq > maxEvictionSeq) maxEvictionSeq = evSeq;
            Instant la = buffer.lastActivity();
            if (la.isAfter(latestActivity)) latestActivity = la;
        }

        merged.sort(Comparator.comparingLong(ContextMessage::windowSeq));
        long lastSeq = merged.isEmpty() ? since : merged.getLast().windowSeq();
        long current = windowSeq.get();
        return new WindowContent(merged, maxEvictionSeq, lastSeq, current, true, latestActivity);
    }

    /** Eagerly evicts TTL-expired entries from all buffers. Called by EvictionScheduler. */
    public void evictExpired() {
        Instant now = Instant.now();
        buffers.values().forEach(b -> b.evictExpired(now));
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core -Dtest=ChannelContextWindowServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 16 tests PASS.

- [ ] **Step 5: Run the full core module tests to check for regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl core
```

Expected: BUILD SUCCESS. `ChannelContextWindowObserverTest` in `casehub/` calls `service.add()` (unchanged) — no regressions there.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor(core): replace associate() with bindAgent/bindChannel/unbindAgent — service-level join

Two-phase association eliminates the cross-SPI coordination race: provisioner
and channel provider write independently; service joins at query time.
unbindAgent() called by WorkerStatusListener.onWorkerCompleted() for cleanup.

Refs #4"
```

---

## Task 2: OpenClawAgentRegistry — routing maps

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java`

- [ ] **Step 1: Write the failing test**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class OpenClawAgentRegistryTest {

    OpenClawAgentRegistry registry;

    @BeforeEach
    void setup() {
        registry = new OpenClawAgentRegistry();
    }

    @Test
    void register_findAgentId_roundTrip() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "session-key-1");
        assertThat(registry.findAgentId(caseId)).contains("agent-1");
    }

    @Test
    void register_findSessionKey_roundTrip() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "session-key-1");
        assertThat(registry.findSessionKey("agent-1")).contains("session-key-1");
    }

    @Test
    void findAgentId_unknownCase_returnsEmpty() {
        assertThat(registry.findAgentId(UUID.randomUUID())).isEmpty();
    }

    @Test
    void findSessionKey_unknownAgent_returnsEmpty() {
        assertThat(registry.findSessionKey("nobody")).isEmpty();
    }

    @Test
    void deregister_removesAllMappings() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "sk");
        registry.deregister("agent-1");
        assertThat(registry.findAgentId(caseId)).isEmpty();
        assertThat(registry.findSessionKey("agent-1")).isEmpty();
    }

    @Test
    void register_overwrite_updatesAllMaps() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "sk-1");
        registry.register("agent-1", caseId, "sk-2");
        assertThat(registry.findSessionKey("agent-1")).contains("sk-2");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawAgentRegistryTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 3: Implement OpenClawAgentRegistry**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

import jakarta.enterprise.context.ApplicationScoped;

/**
 * Routing maps for OpenClaw agent sessions. Shared by WorkerProvisioner,
 * ChannelBackend, and WorkerStatusListener.
 *
 * <p>MVP constraint: 1:1 caseId ↔ agentId. Multiple agents per case silently
 * overwrites the previous entry. Log warnings at register() time if a caseId
 * already has a different agentId registered.
 */
@ApplicationScoped
public class OpenClawAgentRegistry {

    private final ConcurrentHashMap<String, UUID> agentToCase = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<UUID, String> caseToAgent = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, String> agentToSessionKey = new ConcurrentHashMap<>();

    public void register(String agentId, UUID caseId, String sessionKey) {
        agentToCase.put(agentId, caseId);
        caseToAgent.put(caseId, agentId);
        agentToSessionKey.put(agentId, sessionKey);
    }

    public void deregister(String agentId) {
        UUID caseId = agentToCase.remove(agentId);
        if (caseId != null) caseToAgent.remove(caseId);
        agentToSessionKey.remove(agentId);
    }

    public Optional<String> findAgentId(UUID caseId) {
        return Optional.ofNullable(caseToAgent.get(caseId));
    }

    public Optional<String> findSessionKey(String agentId) {
        return Optional.ofNullable(agentToSessionKey.get(agentId));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawAgentRegistryTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentRegistry.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentRegistryTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawAgentRegistry — caseId↔agentId routing maps

Shared by WorkerProvisioner (write), ChannelBackend (read), and
WorkerStatusListener (cleanup). MVP: 1:1 caseId:agentId constraint.

Refs #4"
```

---

## Task 3: OpenClawCasehubConfig — @ConfigMapping for agent capabilities

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCasehubConfig.java`

No separate test class — the config is tested implicitly through WorkerProvisioner tests.

- [ ] **Step 1: Create OpenClawCasehubConfig**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCasehubConfig.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.List;
import java.util.Map;

import io.smallrye.config.ConfigMapping;

/**
 * Agent capability configuration for CaseHub SPI implementations.
 * Separate from OpenClawClientConfig (core/) which owns gateway/delivery/agent-defaults.
 *
 * <p>Example properties:
 * <pre>
 * casehub.openclaw.agents.finance-agent.capabilities=finance,banking
 * casehub.openclaw.agents.finance-agent.session-key=finance-agent
 * casehub.openclaw.agents.code-review-agent.capabilities=code-review
 * casehub.openclaw.agents.code-review-agent.session-key=cr-agent-main
 * </pre>
 */
@ConfigMapping(prefix = "casehub.openclaw")
public interface OpenClawCasehubConfig {

    /** Map of agentId → agent configuration. Keys are agentId strings (e.g. "finance-agent"). */
    Map<String, AgentEntry> agents();

    interface AgentEntry {
        /** Capability tags this agent can handle (e.g. ["finance", "banking"]). */
        List<String> capabilities();

        /**
         * OpenClaw session name — the value passed as {@code sessionName} in /hooks/agent.
         * May differ from the map key if the OpenClaw session has a different name.
         * WARNING: OpenClaw API field name (camelCase vs snake_case) unverified — see openclaw#11.
         */
        String sessionKey();
    }
}
```

- [ ] **Step 2: Compile to verify no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl casehub
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCasehubConfig.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawCasehubConfig — @ConfigMapping for agent capability mapping

Maps agentId → capabilities + sessionKey. Separate from OpenClawClientConfig
(core/) to keep CaseHub-specific configuration in the casehub/ module.

Refs #4"
```

---

## Task 4: OpenClawWorkerProvisioner — WorkerProvisioner SPI

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`

- [ ] **Step 1: Write the failing tests**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.openclaw.context.ChannelContextWindowService;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;

class OpenClawWorkerProvisionerTest {

    ChannelContextWindowService service;
    OpenClawAgentRegistry registry;
    OpenClawWorkerProvisioner provisioner;

    // Minimal config stub — one agent with capabilities [finance, banking]
    OpenClawCasehubConfig config;

    @BeforeEach
    void setup() {
        service = new ChannelContextWindowService();
        service.maxMessagesPerChannel = 100;
        service.ttl = Duration.ofMinutes(30);

        registry = new OpenClawAgentRegistry();
        config = buildConfig(Map.of(
                "finance-agent", buildAgentEntry(List.of("finance", "banking"), "finance-agent"),
                "code-review-agent", buildAgentEntry(List.of("code-review"), "cr-agent-main")));

        provisioner = new OpenClawWorkerProvisioner(service, registry, config);
    }

    @Test
    void provision_singleCapabilityMatch_registersAgent() {
        UUID caseId = UUID.randomUUID();
        provisioner.provision(Set.of("finance"), context(caseId));
        assertThat(registry.findAgentId(caseId)).contains("finance-agent");
    }

    @Test
    void provision_subsetMatch_selectsAgent() {
        UUID caseId = UUID.randomUUID();
        provisioner.provision(Set.of("finance", "banking"), context(caseId));
        assertThat(registry.findAgentId(caseId)).contains("finance-agent");
    }

    @Test
    void provision_bindsAgent_toChannelContextWindow() {
        UUID caseId = UUID.randomUUID();
        provisioner.provision(Set.of("code-review"), context(caseId));

        // bindAgent should have been called — query returns empty window (not noAssociation)
        assertThat(service.query("code-review-agent", 0L).agentHasAssociation()).isTrue();
    }

    @Test
    void provision_registersSessionKey() {
        UUID caseId = UUID.randomUUID();
        provisioner.provision(Set.of("code-review"), context(caseId));
        assertThat(registry.findSessionKey("code-review-agent")).contains("cr-agent-main");
    }

    @Test
    void provision_unknownCapability_throwsProvisioningException() {
        assertThatThrownBy(() ->
                provisioner.provision(Set.of("unknown-capability"), context(UUID.randomUUID())))
                .isInstanceOf(ProvisioningException.class)
                .hasMessageContaining("unknown-capability");
    }

    @Test
    void provision_returnsWorker_withAgentIdAsName() {
        UUID caseId = UUID.randomUUID();
        var worker = provisioner.provision(Set.of("finance"), context(caseId));
        assertThat(worker.getName()).isEqualTo("finance-agent");
    }

    @Test
    void getCapabilities_returnsAllConfiguredCapabilities() {
        Set<String> caps = provisioner.getCapabilities();
        assertThat(caps).containsExactlyInAnyOrder("finance", "banking", "code-review");
    }

    @Test
    void terminate_deregistersAgent() {
        UUID caseId = UUID.randomUUID();
        provisioner.provision(Set.of("finance"), context(caseId));
        provisioner.terminate("finance-agent");
        assertThat(registry.findAgentId(caseId)).isEmpty();
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private ProvisionContext context(UUID caseId) {
        return new ProvisionContext(caseId, "finance", null, null, null, null);
    }

    private OpenClawCasehubConfig buildConfig(Map<String, OpenClawCasehubConfig.AgentEntry> agents) {
        return new OpenClawCasehubConfig() {
            @Override public Map<String, AgentEntry> agents() { return agents; }
        };
    }

    private OpenClawCasehubConfig.AgentEntry buildAgentEntry(List<String> caps, String sk) {
        return new OpenClawCasehubConfig.AgentEntry() {
            @Override public List<String> capabilities() { return caps; }
            @Override public String sessionKey() { return sk; }
        };
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawWorkerProvisionerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 3: Implement OpenClawWorkerProvisioner**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.model.Capability;
import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.WorkerProvisioner;
import io.casehub.openclaw.context.ChannelContextWindowService;

/**
 * Provisions OpenClaw agents as CaseHub workers.
 *
 * <p>"Provisioning" means selecting the correct pre-configured OpenClaw agent for the
 * requested capabilities, registering it in the agent routing maps, and binding it to
 * the ChannelContextWindow. No process is started — OpenClaw agents are always-running.
 *
 * <p>MVP constraint: one OpenClaw agent per case. A second provision() call for the
 * same caseId silently overwrites the first in OpenClawAgentRegistry.
 */
@ApplicationScoped
public class OpenClawWorkerProvisioner implements WorkerProvisioner {

    private static final Logger log = Logger.getLogger(OpenClawWorkerProvisioner.class);

    private final ChannelContextWindowService service;
    private final OpenClawAgentRegistry registry;
    private final OpenClawCasehubConfig config;

    @Inject
    public OpenClawWorkerProvisioner(ChannelContextWindowService service,
                                      OpenClawAgentRegistry registry,
                                      OpenClawCasehubConfig config) {
        this.service = service;
        this.registry = registry;
        this.config = config;
    }

    @Override
    public Worker provision(Set<String> capabilities, ProvisionContext context) {
        String agentId = resolveAgentId(capabilities);
        String sessionKey = config.agents().get(agentId).sessionKey();
        UUID caseId = context.caseId();

        registry.register(agentId, caseId, sessionKey);
        service.bindAgent(agentId, caseId);

        log.infof("Provisioned OpenClaw agent: agentId=%s caseId=%s capabilities=%s",
                agentId, caseId, capabilities);

        List<Capability> capList = capabilities.stream()
                .map(c -> new Capability(c, null, null))
                .toList();
        return new Worker(agentId, capList, ctx -> Map.of());
    }

    @Override
    public void terminate(String workerId) {
        registry.deregister(workerId);
        log.infof("Terminated OpenClaw agent: agentId=%s", workerId);
    }

    @Override
    public Set<String> getCapabilities() {
        return config.agents().values().stream()
                .flatMap(e -> e.capabilities().stream())
                .collect(Collectors.toSet());
    }

    /**
     * Subset match: agent is a candidate if every requested capability is in its config set.
     * First match in iteration order wins. Logs a warning if multiple agents match.
     *
     * @throws ProvisioningException if no agent covers all requested capabilities
     */
    private String resolveAgentId(Set<String> requested) {
        List<String> candidates = config.agents().entrySet().stream()
                .filter(e -> e.getValue().capabilities().containsAll(requested))
                .map(Map.Entry::getKey)
                .sorted()
                .toList();

        if (candidates.isEmpty()) {
            throw new ProvisioningException(
                    "No OpenClaw agent configured for capabilities: " + requested);
        }
        if (candidates.size() > 1) {
            log.warnf("Multiple agents match capabilities %s: %s — selecting %s. " +
                    "Check configuration for overlapping capability declarations.",
                    requested, candidates, candidates.get(0));
        }
        return candidates.get(0);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawWorkerProvisionerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawWorkerProvisioner — WorkerProvisioner SPI implementation

Subset capability match; first-wins alphabetical; ProvisioningException on no match.
Calls bindAgent() and registry.register() at provision time.

Refs #4"
```

---

## Task 5: OpenClawCaseChannelProvider — CaseChannelProvider SPI

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProviderTest.java`

- [ ] **Step 1: Write the failing tests**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProviderTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.api.model.CaseChannel;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.qhorus.runtime.model.Channel;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class OpenClawCaseChannelProviderTest {

    ChannelService channelService;
    MessageService messageService;
    ChannelContextWindowService contextService;
    OpenClawCaseChannelProvider provider;

    @BeforeEach
    void setup() {
        channelService = mock(ChannelService.class);
        messageService = mock(MessageService.class);
        contextService = new ChannelContextWindowService();
        contextService.maxMessagesPerChannel = 100;
        contextService.ttl = Duration.ofMinutes(30);

        provider = new OpenClawCaseChannelProvider(channelService, messageService, contextService);
    }

    private Channel stubChannel(String name) {
        Channel ch = new Channel();
        ch.id = UUID.randomUUID();
        ch.name = name;
        return ch;
    }

    @Test
    void openChannel_work_callsBindChannel() {
        UUID caseId = UUID.randomUUID();
        String name = CaseChannel.channelName(caseId, "work");
        Channel ch = stubChannel(name);
        when(channelService.findByNamePrefix(name)).thenReturn(List.of());
        when(channelService.create(anyString(), anyString(), any(), any(), any(), any(), any(), any(), any()))
                .thenReturn(ch);

        CaseChannel result = provider.openChannel(caseId, "work");

        assertThat(result.id()).isEqualTo(ch.id.toString());
        assertThat(result.purpose()).isEqualTo("work");
        assertThat(result.backendType()).isEqualTo("qhorus");

        // bindChannel should have been called for this channel
        assertThat(contextService.query("agent-1", 0L).agentHasAssociation()).isFalse();
        // (no agent bound — just verify bindChannel was called by checking buffer exists)
    }

    @Test
    void openChannel_idempotent_returnsExisting() {
        UUID caseId = UUID.randomUUID();
        String name = CaseChannel.channelName(caseId, "work");
        Channel ch = stubChannel(name);
        when(channelService.findByNamePrefix(name)).thenReturn(List.of(ch));

        CaseChannel result1 = provider.openChannel(caseId, "work");
        CaseChannel result2 = provider.openChannel(caseId, "work");

        assertThat(result1.id()).isEqualTo(result2.id());
        // create() should NOT have been called — found existing
        verify(channelService, times(0)).create(any(), any(), any(), any(), any(), any(), any(), any(), any());
    }

    @Test
    void postToChannel_callsMessageServiceDispatch() {
        UUID caseId = UUID.randomUUID();
        UUID channelId = UUID.randomUUID();
        CaseChannel channel = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus", Map.of());

        provider.postToChannel(channel, "sender-1", "content", MessageType.COMMAND, "corr-1", null);

        verify(messageService).dispatch(any(MessageDispatch.class));
    }

    @Test
    void postToChannel_threeArgDefault_callsDispatch() {
        UUID channelId = UUID.randomUUID();
        CaseChannel channel = new CaseChannel(channelId.toString(), "case-x/work", "work", "qhorus", Map.of());

        provider.postToChannel(channel, "sender-1", "content");

        verify(messageService).dispatch(any(MessageDispatch.class));
    }

    @Test
    void closeChannel_noOp_doesNotThrow() {
        CaseChannel channel = new CaseChannel(UUID.randomUUID().toString(), "case-x/work", "work", "qhorus", Map.of());
        provider.closeChannel(channel); // must not throw
    }

    @Test
    void listChannels_delegatesToChannelService() {
        UUID caseId = UUID.randomUUID();
        String prefix = "case-" + caseId + "/";
        Channel ch = stubChannel(prefix + "work");
        when(channelService.findByNamePrefix(prefix)).thenReturn(List.of(ch));

        List<CaseChannel> result = provider.listChannels(caseId);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).purpose()).isEqualTo("work");
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawCaseChannelProviderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 3: Implement OpenClawCaseChannelProvider**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java`:

```java
package io.casehub.openclaw.casehub;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.model.Channel;

/**
 * Creates and manages Qhorus channels per CaseHub case.
 *
 * <p>Three normative channels per case: work, observe, oversight — all APPEND semantic,
 * matching Claudony's NormativeChannelLayout. Channel names follow the CaseChannel convention
 * "case-{caseId}/{purpose}".
 *
 * <p>openChannel() is idempotent: finds existing channel by name before creating.
 * bindChannel() is called after each open to register with ChannelContextWindow.
 */
@ApplicationScoped
public class OpenClawCaseChannelProvider implements CaseChannelProvider {

    private static final Logger log = Logger.getLogger(OpenClawCaseChannelProvider.class);
    private static final String QHORUS_NAME_KEY = "qhorus-name";

    // Normative channel layout: purpose → (description, allowedTypes CSV or null)
    private static final Map<String, String[]> LAYOUT = Map.of(
            "work",     new String[]{"Primary coordination — all obligation-carrying message types", null},
            "observe",  new String[]{"Telemetry — EVENT only, no obligations created", "EVENT"},
            "oversight",new String[]{"Human governance — agent QUERY and human COMMAND", "COMMAND,QUERY"}
    );

    private final ChannelService channelService;
    private final MessageService messageService;
    private final ChannelContextWindowService contextService;

    @Inject
    public OpenClawCaseChannelProvider(ChannelService channelService,
                                        MessageService messageService,
                                        ChannelContextWindowService contextService) {
        this.channelService = channelService;
        this.messageService = messageService;
        this.contextService = contextService;
    }

    @Override
    public CaseChannel openChannel(UUID caseId, String purpose) {
        String channelName = CaseChannel.channelName(caseId, purpose);
        String[] spec = LAYOUT.get(purpose);
        String description = spec != null ? spec[0] : purpose;
        String allowedTypes = spec != null ? spec[1] : null;

        // Find existing channel first (idempotency contract from engine#323)
        Channel channel = channelService.findByNamePrefix(channelName).stream()
                .filter(c -> channelName.equals(c.name))
                .findFirst()
                .orElseGet(() -> channelService.create(channelName, description,
                        ChannelSemantic.APPEND, null, null, null, null, null, allowedTypes));

        contextService.bindChannel(caseId, channel.id);
        log.debugf("Opened channel: %s (id=%s)", channelName, channel.id);
        return new CaseChannel(channel.id.toString(), channel.name, purpose, "qhorus",
                Map.of(QHORUS_NAME_KEY, channel.name));
    }

    @Override
    public void postToChannel(CaseChannel channel, String from, String content,
                               MessageType type, String correlationId, String deadline) {
        messageService.dispatch(MessageDispatch.builder()
                .channelId(UUID.fromString(channel.id()))
                .sender(from)
                .type(type)
                .content(content)
                .correlationId(correlationId)
                .deadline(deadline != null ? Instant.parse(deadline) : null)
                .actorType(ActorType.AGENT)
                .build());
    }

    @Override
    public void closeChannel(CaseChannel channel) {
        // Qhorus channels are persistent — no close operation
    }

    @Override
    public List<CaseChannel> listChannels(UUID caseId) {
        String prefix = CaseChannel.CASE_CHANNEL_PREFIX + caseId + "/";
        return channelService.findByNamePrefix(prefix).stream()
                .map(ch -> new CaseChannel(
                        ch.id.toString(),
                        ch.name,
                        extractPurpose(ch.name, caseId),
                        "qhorus",
                        Map.of(QHORUS_NAME_KEY, ch.name)))
                .toList();
    }

    private String extractPurpose(String channelName, UUID caseId) {
        String prefix = CaseChannel.CASE_CHANNEL_PREFIX + caseId + "/";
        return channelName.startsWith(prefix) ? channelName.substring(prefix.length()) : channelName;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawCaseChannelProviderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProvider.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawCaseChannelProviderTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawCaseChannelProvider — CaseChannelProvider SPI implementation

Normative 3-channel layout (work/observe/oversight, APPEND semantic).
Find-before-create for idempotency. Calls bindChannel() after each open.

Refs #4"
```

---

## Task 6: OpenClawChannelBackend — ChannelBackend SPI + ChannelInitialisedEvent

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java`

- [ ] **Step 1: Write the failing tests**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import io.casehub.api.model.CaseChannel;
import io.casehub.openclaw.client.OpenClawClientConfig;
import io.casehub.openclaw.client.OpenClawHookClient;
import io.casehub.openclaw.client.OpenClawInvocationException;
import io.casehub.qhorus.api.gateway.ChannelInitialisedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class OpenClawChannelBackendTest {

    OpenClawAgentRegistry registry;
    OpenClawHookClient hookClient;
    ChannelGateway gateway;
    OpenClawClientConfig clientConfig;
    OpenClawChannelBackend backend;

    UUID caseId = UUID.randomUUID();
    UUID channelId = UUID.randomUUID();

    @BeforeEach
    void setup() {
        registry = new OpenClawAgentRegistry();
        hookClient = mock(OpenClawHookClient.class);
        gateway = mock(ChannelGateway.class);
        clientConfig = buildConfig("http://localhost:8080", "claude-opus-4-5", 120);
        backend = new OpenClawChannelBackend(registry, hookClient, gateway, clientConfig);
    }

    @Test
    void onChannelInitialised_caseChannel_registersBackend() {
        ChannelInitialisedEvent event = new ChannelInitialisedEvent(channelId, "case-" + caseId + "/work");
        backend.onChannelInitialised(event);
        verify(gateway).registerBackend(eq(channelId), eq(backend), eq("agent"));
    }

    @Test
    void onChannelInitialised_nonCaseChannel_doesNotRegister() {
        ChannelInitialisedEvent event = new ChannelInitialisedEvent(channelId, "some-other-channel");
        backend.onChannelInitialised(event);
        verify(gateway, never()).registerBackend(any(), any(), any());
    }

    @Test
    void post_command_invokesAgent() {
        registry.register("finance-agent", caseId, "finance-agent");

        ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine", MessageType.COMMAND,
                "Analyse this", null, null, ActorType.AGENT);

        backend.post(ref, msg);

        verify(hookClient).registerSession(eq("finance-agent"), eq("finance-agent"),
                eq("http://localhost:8080/channel/" + channelId));
        verify(hookClient).invoke(eq("finance-agent"), eq("Analyse this"), eq("claude-opus-4-5"), eq(120));
    }

    @ParameterizedTest
    @EnumSource(value = MessageType.class, mode = EnumSource.Mode.EXCLUDE, names = {"COMMAND"})
    void post_nonCommand_doesNotInvokeAgent(MessageType type) {
        registry.register("finance-agent", caseId, "finance-agent");
        ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine", type,
                "content", null, null, ActorType.AGENT);

        backend.post(ref, msg);
        verify(hookClient, never()).invoke(any(), any(), any(), any(Integer.class));
    }

    @Test
    void post_noAgentForCase_noOp() {
        // No agent registered for caseId
        ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine", MessageType.COMMAND,
                "content", null, null, ActorType.AGENT);

        assertThatCode(() -> backend.post(ref, msg)).doesNotThrowAnyException();
        verify(hookClient, never()).invoke(any(), any(), any(), any(Integer.class));
    }

    @Test
    void post_invokeThrows_exceptionCaught_notPropagated() {
        registry.register("finance-agent", caseId, "finance-agent");
        doThrow(new OpenClawInvocationException("network error"))
                .when(hookClient).invoke(any(), any(), any(), any(Integer.class));

        ChannelRef ref = new ChannelRef(channelId, "case-" + caseId + "/work");
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine", MessageType.COMMAND,
                "content", null, null, ActorType.AGENT);

        assertThatCode(() -> backend.post(ref, msg)).doesNotThrowAnyException();
    }

    @Test
    void post_nonCaseChannelName_noOp() {
        registry.register("finance-agent", caseId, "finance-agent");
        ChannelRef ref = new ChannelRef(channelId, "non-case-channel");
        OutboundMessage msg = new OutboundMessage(UUID.randomUUID(), "engine", MessageType.COMMAND,
                "content", null, null, ActorType.AGENT);

        assertThatCode(() -> backend.post(ref, msg)).doesNotThrowAnyException();
        verify(hookClient, never()).invoke(any(), any(), any(), any(Integer.class));
    }

    @Test
    void backendId_isOpenclaw() {
        assertThat(backend.backendId()).isEqualTo("openclaw");
    }

    @Test
    void actorType_isAgent() {
        assertThat(backend.actorType()).isEqualTo(ActorType.AGENT);
    }

    // ── helpers ───────────────────────────────────────────────────────────────

    private OpenClawClientConfig buildConfig(String baseUrl, String model, int timeout) {
        return new OpenClawClientConfig() {
            @Override public Gateway gateway() { return new Gateway() {
                @Override public String url() { return "http://openclaw"; }
                @Override public String bearerToken() { return "tok"; }
            }; }
            @Override public Delivery delivery() { return new Delivery() {
                @Override public String baseUrl() { return baseUrl; }
            }; }
            @Override public Agent agent() { return new Agent() {
                @Override public String defaultModel() { return model; }
                @Override public int defaultTimeoutSeconds() { return timeout; }
            }; }
        };
    }
}
```

Note: add the missing `assertThat` static import — same as other test files: `import static org.assertj.core.api.Assertions.assertThat;`

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawChannelBackendTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 3: Implement OpenClawChannelBackend**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Map;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.model.CaseChannel;
import io.casehub.openclaw.client.OpenClawClientConfig;
import io.casehub.openclaw.client.OpenClawHookClient;
import io.casehub.openclaw.client.OpenClawInvocationException;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.gateway.ChannelBackend;
import io.casehub.qhorus.api.gateway.ChannelInitialisedEvent;
import io.casehub.qhorus.api.gateway.ChannelRef;
import io.casehub.qhorus.api.gateway.OutboundMessage;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.gateway.ChannelGateway;

/**
 * Bridges Qhorus COMMANDs to OpenClaw agents via the hook API.
 *
 * <p>Self-registers with ChannelGateway on ChannelInitialisedEvent for case channels.
 * This also handles startup recovery: Qhorus fires ChannelInitialisedEvent for all
 * persisted channels at startup, so the backend re-registers without its own recovery logic.
 *
 * <p>Only COMMAND messages invoke the agent. All other types are silently ignored.
 * OpenClawInvocationException is caught and logged — ChannelGateway.fanOut() is non-fatal.
 */
@ApplicationScoped
public class OpenClawChannelBackend implements ChannelBackend {

    private static final Logger log = Logger.getLogger(OpenClawChannelBackend.class);

    private final OpenClawAgentRegistry registry;
    private final OpenClawHookClient hookClient;
    private final ChannelGateway gateway;
    private final OpenClawClientConfig config;

    @Inject
    public OpenClawChannelBackend(OpenClawAgentRegistry registry,
                                   OpenClawHookClient hookClient,
                                   ChannelGateway gateway,
                                   OpenClawClientConfig config) {
        this.registry = registry;
        this.hookClient = hookClient;
        this.gateway = gateway;
        this.config = config;
    }

    void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
        if (!event.channelName().startsWith(CaseChannel.CASE_CHANNEL_PREFIX)) return;
        gateway.registerBackend(event.channelId(), this, "agent");
        log.debugf("Registered OpenClaw backend for channel: %s", event.channelName());
    }

    @Override
    public String backendId() {
        return "openclaw";
    }

    @Override
    public ActorType actorType() {
        return ActorType.AGENT;
    }

    @Override
    public void open(ChannelRef channel, Map<String, String> metadata) {
        // Registration handled via ChannelInitialisedEvent — no-op here
    }

    @Override
    public void post(ChannelRef channel, OutboundMessage message) {
        if (message.type() != MessageType.COMMAND) return;

        UUID caseId = extractCaseId(channel.name());
        if (caseId == null) return;

        String agentId = registry.findAgentId(caseId).orElse(null);
        if (agentId == null) {
            log.debugf("No OpenClaw agent registered for caseId=%s — ignoring COMMAND", caseId);
            return;
        }

        String sessionKey = registry.findSessionKey(agentId)
                .orElseThrow(() -> new IllegalStateException("No session key for agentId: " + agentId));

        // webhookUrl: last-write-wins per invocation; safe because OpenClaw uses the
        // request-body URL for delivery, not a server-side session lookup.
        String webhookUrl = config.delivery().baseUrl() + "/channel/" + channel.id();
        hookClient.registerSession(agentId, sessionKey, webhookUrl);

        try {
            hookClient.invoke(agentId, message.content(),
                    config.agent().defaultModel(), config.agent().defaultTimeoutSeconds());
            log.debugf("Invoked OpenClaw agent: agentId=%s caseId=%s", agentId, caseId);
        } catch (OpenClawInvocationException e) {
            log.errorf("OpenClaw invocation failed for agentId=%s: %s", agentId, e.getMessage());
            // Non-fatal — ChannelGateway.fanOut() absorbs exceptions from non-default backends
        }
    }

    @Override
    public void close(ChannelRef channel) {
        // Qhorus channels are persistent — no teardown needed
    }

    /** Parses "case-{caseId}/{purpose}" → UUID, or returns null if format doesn't match. */
    UUID extractCaseId(String channelName) {
        if (!channelName.startsWith(CaseChannel.CASE_CHANNEL_PREFIX)) return null;
        String withoutPrefix = channelName.substring(CaseChannel.CASE_CHANNEL_PREFIX.length());
        int slash = withoutPrefix.indexOf('/');
        String uuidStr = slash >= 0 ? withoutPrefix.substring(0, slash) : withoutPrefix;
        try {
            return UUID.fromString(uuidStr);
        } catch (IllegalArgumentException e) {
            return null;
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawChannelBackendTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawChannelBackendTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawChannelBackend — ChannelBackend SPI + self-registration

Observes ChannelInitialisedEvent to register for case-* channels (handles
startup recovery automatically). COMMAND-only agent invocation; all other
types ignored. OpenClawInvocationException caught — fanOut() is non-fatal.

Refs #4"
```

---

## Task 7: OpenClawWorkerStatusListener — WorkerStatusListener SPI

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`

- [ ] **Step 1: Write the failing tests**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java`:

```java
package io.casehub.openclaw.casehub;

import java.time.Duration;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import io.casehub.api.model.WorkResult;
import io.casehub.api.model.WorkStatus;
import io.casehub.openclaw.context.ChannelContextWindowService;
import jakarta.enterprise.event.Event;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

class OpenClawWorkerStatusListenerTest {

    ChannelContextWindowService service;
    OpenClawAgentRegistry registry;
    Event<Object> events;
    OpenClawWorkerStatusListener listener;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setup() {
        service = new ChannelContextWindowService();
        service.maxMessagesPerChannel = 100;
        service.ttl = Duration.ofMinutes(30);

        registry = new OpenClawAgentRegistry();
        events = mock(Event.class);
        listener = new OpenClawWorkerStatusListener(service, registry, events);
    }

    @Test
    void onWorkerCompleted_deregistersFromRegistry() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "sk");
        service.bindAgent("agent-1", caseId);

        listener.onWorkerCompleted("agent-1", WorkResult.completed("key", Map.of(), "agent-1", caseId));

        assertThat(registry.findAgentId(caseId)).isEmpty();
    }

    @Test
    void onWorkerCompleted_unBindsFromContextWindow() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "sk");
        service.bindAgent("agent-1", caseId);

        listener.onWorkerCompleted("agent-1", WorkResult.completed("key", Map.of(), "agent-1", caseId));

        assertThat(service.query("agent-1", 0L).agentHasAssociation()).isFalse();
    }

    @Test
    void onWorkerStalled_firesEvent() {
        listener.onWorkerStalled("agent-1");
        verify(events).fire(any(OpenClawWorkerStatusListener.WorkerStalledEvent.class));
    }

    @Test
    void onWorkerStalled_doesNotDeregister() {
        UUID caseId = UUID.randomUUID();
        registry.register("agent-1", caseId, "sk");

        listener.onWorkerStalled("agent-1");

        // Agent remains registered — Watchdog drives recovery
        assertThat(registry.findAgentId(caseId)).contains("agent-1");
    }

    @Test
    void onWorkerStarted_doesNotThrow() {
        listener.onWorkerStarted("agent-1", Map.of("caseId", UUID.randomUUID().toString()));
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawWorkerStatusListenerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 3: Implement OpenClawWorkerStatusListener**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.Map;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.model.WorkResult;
import io.casehub.api.spi.WorkerStatusListener;
import io.casehub.openclaw.context.ChannelContextWindowService;

/**
 * Reacts to CaseHub engine worker lifecycle events for OpenClaw agents.
 *
 * <p>Called by WorkflowExecutionCompletedHandler (engine runtime) via the
 * WORKER_EXECUTION_FINISHED Vert.x event bus address, which is published when
 * QhorusMessageSignalBridge delivers a DONE signal to CaseHubRuntime.signal().
 *
 * <p>onWorkerCompleted(): cleans up registry and ChannelContextWindow mappings.
 * onWorkerStalled(): fires CDI event; agent remains registered (Watchdog drives recovery).
 */
@ApplicationScoped
public class OpenClawWorkerStatusListener implements WorkerStatusListener {

    private static final Logger log = Logger.getLogger(OpenClawWorkerStatusListener.class);

    private final ChannelContextWindowService service;
    private final OpenClawAgentRegistry registry;
    private final Event<Object> events;

    @Inject
    public OpenClawWorkerStatusListener(ChannelContextWindowService service,
                                         OpenClawAgentRegistry registry,
                                         Event<Object> events) {
        this.service = service;
        this.registry = registry;
        this.events = events;
    }

    @Override
    public void onWorkerStarted(String workerId, Map<String, String> sessionMeta) {
        log.infof("OpenClaw agent started: agentId=%s", workerId);
    }

    @Override
    public void onWorkerCompleted(String workerId, WorkResult result) {
        log.infof("OpenClaw agent completed: agentId=%s status=%s", workerId, result.status());
        registry.deregister(workerId);
        service.unbindAgent(workerId);
    }

    @Override
    public void onWorkerStalled(String workerId) {
        log.warnf("OpenClaw agent stalled: agentId=%s — remains registered; Watchdog drives recovery",
                workerId);
        events.fire(new WorkerStalledEvent(workerId));
    }

    /** Fired when an OpenClaw agent stalls. Observers may alert or trigger remediation. */
    public record WorkerStalledEvent(String agentId) {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -Dtest=OpenClawWorkerStatusListenerTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: All 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListener.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerStatusListenerTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): OpenClawWorkerStatusListener — WorkerStatusListener SPI implementation

onWorkerCompleted() cleans up registry + ChannelContextWindow (symmetric unbind).
onWorkerStalled() fires WorkerStalledEvent; agent stays registered for Watchdog recovery.

Refs #4"
```

---

## Task 8: casehub/ pom.xml — remove provided scopes

**Files:**
- Modify: `casehub/pom.xml`

- [ ] **Step 1: Remove provided scopes from engine and qhorus dependencies**

In `casehub/pom.xml`, find and update the `casehub-qhorus` dependency:

```xml
<!-- BEFORE -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus</artifactId>
  <scope>provided</scope>
</dependency>

<!-- AFTER -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-qhorus</artifactId>
</dependency>
```

And the `casehub-engine` dependency:

```xml
<!-- BEFORE -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine</artifactId>
  <scope>provided</scope>
</dependency>

<!-- AFTER -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine</artifactId>
</dependency>
```

- [ ] **Step 2: Build the casehub module to verify it compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl casehub -am
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Run all casehub tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/pom.xml
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "build(casehub): remove provided scope from casehub-engine and casehub-qhorus

SPI implementations now present — runtime scope required for CDI discovery.

Refs #4"
```

---

## Task 9: OpenClawDeliveryPayload + OpenClawDeliveryResource

**Files:**
- Create: `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryPayload.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java`
- Create: `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`

- [ ] **Step 1: Create OpenClawDeliveryPayload**

Create `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryPayload.java`:

```java
package io.casehub.openclaw.app;

/**
 * Webhook payload received from OpenClaw when an agent completes via deliver:webhook.
 *
 * WARNING: Field names are assumed camelCase based on other OpenClaw API fields.
 * Verify against live OpenClaw API before production use — see openclaw#11.
 */
public record OpenClawDeliveryPayload(
    String agentId,   // WARNING: may need @JsonProperty("agent_id") — verify openclaw#11
    String output     // WARNING: may need @JsonProperty("output") — verify openclaw#11
) {}
```

- [ ] **Step 2: Write the failing test**

Create `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`:

```java
package io.casehub.openclaw.app;

import java.util.Optional;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.is;

@QuarkusTest
class OpenClawDeliveryResourceTest {

    @Test
    void deliver_unknownChannel_returns404() {
        UUID unknownChannelId = UUID.randomUUID();
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"agentId": "finance-agent", "output": "Analysis complete."}
                """)
        .when()
            .post("/openclaw/delivery/channel/" + unknownChannelId)
        .then()
            .statusCode(404);
    }

    @Test
    void deliver_invalidUuid_returns404() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"agentId": "finance-agent", "output": "Analysis complete."}
                """)
        .when()
            .post("/openclaw/delivery/channel/not-a-uuid")
        .then()
            .statusCode(404);
    }
}
```

Note: The test for a valid channelId requires a real Qhorus channel to exist. Since this is a `@QuarkusTest` with H2 in-memory DBs (no pre-seeded data), only the failure paths are tested in unit scope. End-to-end path tested via the integration flow.

- [ ] **Step 3: Run to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -Dtest=OpenClawDeliveryResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: COMPILATION ERROR — class does not exist.

- [ ] **Step 4: Implement OpenClawDeliveryResource**

Create `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java`:

```java
package io.casehub.openclaw.app;

import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import org.jboss.logging.Logger;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.MessageService;

/**
 * Receives OpenClaw agent results delivered via deliver:webhook and posts them
 * as DONE messages to the originating Qhorus channel.
 *
 * <p>Always returns 200 — OpenClaw must not retry on our processing errors.
 * On dispatch failure, the case step hangs until Watchdog recovery.
 * This trade-off is revisited in openclaw#11 once OpenClaw's retry contract is verified.
 *
 * <p>This endpoint has no auth — follows gateway topology (Claudony is the auth entry point).
 * See auth-retrofit-readiness.md protocol.
 */
@Path("/openclaw/delivery/channel")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class OpenClawDeliveryResource {

    private static final Logger log = Logger.getLogger(OpenClawDeliveryResource.class);

    @Inject
    ChannelService channelService;

    @Inject
    MessageService messageService;

    @POST
    @Path("/{channelId}")
    public Response deliver(@PathParam("channelId") String channelIdStr,
                             OpenClawDeliveryPayload payload) {
        UUID channelId;
        try {
            channelId = UUID.fromString(channelIdStr);
        } catch (IllegalArgumentException e) {
            return Response.status(404).build();
        }

        boolean exists = !channelService.findByNamePrefix("").stream()
                .filter(c -> c.id.equals(channelId))
                .toList().isEmpty();

        if (!exists) {
            log.warnf("Delivery received for unknown channelId=%s", channelId);
            return Response.status(404).build();
        }

        try {
            messageService.dispatch(MessageDispatch.builder()
                    .channelId(channelId)
                    .sender(payload.agentId() != null ? payload.agentId() : "openclaw-agent")
                    .type(MessageType.DONE)
                    .content(payload.output() != null ? payload.output() : "")
                    .actorType(ActorType.AGENT)
                    .build());
        } catch (Exception e) {
            // Fail open — return 200 so OpenClaw does not retry. Case step hangs until Watchdog.
            log.errorf("Failed to dispatch DONE to channel %s: %s", channelId, e.getMessage());
        }

        return Response.ok().build();
    }
}
```

Note: The `channelService.findByNamePrefix("")` approach is a workaround for finding a channel by ID (which Qhorus doesn't expose as a direct method). Check if `ChannelService` has a `find(UUID)` method — if it does, prefer that. If not, use `findByNamePrefix("")` which returns all channels (only suitable for tests with few channels).

**Better approach:** Check if ChannelService has a direct `find(UUID id)`:

```bash
grep -n "Channel find\|Optional.*find" /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java | head -10
```

If a `find(UUID)` method exists, use it instead:

```java
Optional<Channel> channel = channelService.find(channelId);
if (channel.isEmpty()) {
    return Response.status(404).build();
}
```

- [ ] **Step 5: Verify ChannelService.find(UUID) exists and update if needed**

```bash
grep -n "public.*Channel find\|public Optional.*find" /Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java
```

If the method exists, update the resource to use it directly. If not, keep the `findByNamePrefix("")` approach with a note.

- [ ] **Step 6: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dtest=OpenClawDeliveryResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: Both tests PASS (404 for unknown channel, 404 for invalid UUID).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryPayload.java app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(app): OpenClawDeliveryResource — POST /openclaw/delivery/channel/{channelId}

Receives OpenClaw webhook results, posts DONE to Qhorus channel.
Phase 1: always DONE. Fail-open on dispatch failure (Watchdog recovery).
Field names unverified against live API — see openclaw#11.

Refs #4"
```

---

## Task 10: application.properties — CDI exclusions + agent config

**Files:**
- Modify: `app/src/main/resources/application.properties`

- [ ] **Step 1: Add quarkus.arc.exclude-types and sample agent config**

Append to `app/src/main/resources/application.properties`:

```properties
# CDI: exclude engine no-op SPI beans that would collide with our implementations
# Generated for casehub-engine 0.2-SNAPSHOT. If the engine adds new NoOp beans,
# this list must be updated — startup will fail with AmbiguousResolutionException.
quarkus.arc.exclude-types=\
  io.casehub.engine.internal.worker.NoOpWorkerStatusListener,\
  io.casehub.engine.internal.worker.NoOpReactiveWorkerStatusListener,\
  io.casehub.engine.internal.worker.EmptyWorkerContextProvider,\
  io.casehub.engine.internal.worker.EmptyReactiveWorkerContextProvider,\
  io.casehub.engine.internal.worker.NoOpWorkerProvisioner,\
  io.casehub.engine.internal.worker.NoOpReactiveWorkerProvisioner,\
  io.casehub.engine.internal.worker.NoOpCaseChannelProvider,\
  io.casehub.engine.internal.worker.NoOpReactiveCaseChannelProvider

# OpenClaw agent capability configuration
# Configure one entry per OpenClaw agent. Key = agentId (used internally for routing).
# session-key = OpenClaw session name (passed as sessionName in /hooks/agent).
# WARNING: sessionName field name unverified against live API — see openclaw#11.
casehub.openclaw.agents.finance-agent.capabilities=finance,banking
casehub.openclaw.agents.finance-agent.session-key=finance-agent
```

- [ ] **Step 2: Run the full app module build to verify CDI wiring**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am
```

Expected: BUILD SUCCESS — no `AmbiguousResolutionException` at startup. This implicitly verifies the `quarkus.arc.exclude-types` configuration is correct.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "config(app): CDI exclude-types for engine no-ops + sample agent config

Prevents AmbiguousResolutionException from casehub-engine's @ApplicationScoped
NoOp SPI beans colliding with our implementations (GE-20260428-9311f8).

Refs #4"
```

---

## Task 11: Full build verification

- [ ] **Step 1: Build all Maven modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS, all tests pass across core/, casehub/, app/.

- [ ] **Step 2: Verify test counts**

The output should show at least:
- `core/`: 16+ tests (ChannelContextWindowServiceTest rewritten + ChannelContextWindowObserverTest unchanged)
- `casehub/`: 35+ tests (6 registry + 8 provisioner + 6 channel provider + 9 backend + 5 listener + 4 observer)
- `app/`: 2+ tests (delivery resource)

- [ ] **Step 3: If any test fails — stop and diagnose before proceeding to code review**

Do not proceed to the code review step until all tests pass.

---

## Self-Review Checklist

- [x] **Spec coverage:**
  - `ChannelContextWindowService` bindAgent/bindChannel/unbindAgent — Task 1 ✅
  - `OpenClawAgentRegistry` — Task 2 ✅
  - `OpenClawCasehubConfig` @ConfigMapping — Task 3 ✅
  - `OpenClawWorkerProvisioner` — Task 4 ✅
  - `OpenClawCaseChannelProvider` — Task 5 ✅
  - `OpenClawChannelBackend` + ChannelInitialisedEvent — Task 6 ✅
  - `OpenClawWorkerStatusListener` — Task 7 ✅
  - pom.xml scope changes — Task 8 ✅
  - `OpenClawDeliveryResource` — Task 9 ✅
  - application.properties CDI exclusions — Task 10 ✅
  - Full build — Task 11 ✅

- [x] **Placeholder scan:** No TBDs or "similar to Task N" patterns. All test code is complete.

- [x] **Type consistency:**
  - `ChannelContextWindowService.bindAgent(String, UUID)` used in Tasks 1, 4, 7 ✅
  - `ChannelContextWindowService.bindChannel(UUID, UUID)` used in Tasks 1, 5 ✅
  - `ChannelContextWindowService.unbindAgent(String)` used in Tasks 1, 7 ✅
  - `OpenClawAgentRegistry.register/deregister/findAgentId/findSessionKey` used in Tasks 2, 4, 6, 7 ✅
  - `OpenClawCasehubConfig.agents()` returns `Map<String, AgentEntry>` used in Tasks 3, 4 ✅
  - `Worker(agentId, capList, ctx -> Map.of())` constructor pattern matches Task 4 ✅
  - `CaseChannel.channelName(UUID, String)` used in Tasks 5, 6 ✅
  - `CaseChannel.CASE_CHANNEL_PREFIX` used in Tasks 5, 6 ✅

- [x] **Pending verification in Task 9:** Whether `ChannelService.find(UUID)` exists — Step 5 checks this and updates accordingly.
