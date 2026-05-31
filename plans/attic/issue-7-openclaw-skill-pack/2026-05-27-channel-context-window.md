# ChannelContextWindow Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the in-memory ChannelContextWindow service — a ring buffer of Qhorus channel activity exposed as a REST endpoint for the Python SDK `before_prompt_build` hook.

**Architecture:** Three Maven modules: `core/` holds the ring buffer data structures and service logic (pure Java, no Quarkus lifecycle); `casehub/` holds the `MessageObserver` SPI implementation that feeds the ring buffer; `app/` holds the REST endpoint and scheduled eviction runner.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, AssertJ, REST-assured, `casehub-qhorus-api` (MessageObserver SPI).

**Design spec:** `docs/specs/2026-05-27-channel-context-window-design.md`

---

## File Map

```
CREATE core/src/main/java/io/casehub/openclaw/context/ContextMessage.java
CREATE core/src/main/java/io/casehub/openclaw/context/WindowContent.java
CREATE core/src/main/java/io/casehub/openclaw/context/ChannelRingBuffer.java
CREATE core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java
CREATE core/src/test/java/io/casehub/openclaw/context/ChannelRingBufferTest.java
CREATE core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java

MODIFY casehub/pom.xml                      — add mockito-core test dep
CREATE casehub/src/main/java/io/casehub/openclaw/casehub/ChannelContextWindowObserver.java
CREATE casehub/src/test/java/io/casehub/openclaw/casehub/ChannelContextWindowObserverTest.java

MODIFY app/pom.xml                          — add quarkus-scheduler + quarkus-junit5-mockito
MODIFY app/src/main/resources/application.properties  — context-window config + jackson dates
CREATE app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java
CREATE app/src/main/java/io/casehub/openclaw/app/EvictionScheduler.java
CREATE app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java
```

---

## Task 1: Data Types — ContextMessage and WindowContent

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/context/ContextMessage.java`
- Create: `core/src/main/java/io/casehub/openclaw/context/WindowContent.java`

No behaviour to unit-test (pure data). Both are tested implicitly by all subsequent tasks. Compile verification is in Task 2.

- [ ] **Step 1: Create ContextMessage record**

```java
package io.casehub.openclaw.context;

import java.time.Instant;
import java.util.UUID;

import io.casehub.qhorus.api.message.MessageType;

/**
 * One message entry in a ChannelContextWindow ring buffer.
 *
 * {@code windowSeq} is a global monotonic counter assigned by
 * {@link ChannelContextWindowService} — not Qhorus's per-channel sequenceNumber.
 *
 * {@code receivedAt} is stamped at ingestion time ({@code Instant.now()}) inside
 * {@link ChannelContextWindowService#add} — not taken from Qhorus (MessageReceivedEvent
 * carries no timestamp). Used only for TTL eviction; {@code windowSeq} governs ordering.
 *
 * {@code correlationId} is nullable: COMMAND and QUERY originate correlations; other
 * types carry the correlation they were issued with; EVENT has none.
 */
public record ContextMessage(
        long windowSeq,
        UUID channelId,
        String channelName,
        MessageType messageType,
        String senderId,
        String correlationId,
        String content,
        Instant receivedAt) {}
```

- [ ] **Step 2: Create WindowContent record**

```java
package io.casehub.openclaw.context;

import java.time.Instant;
import java.util.List;

/**
 * Query result returned by {@link ChannelContextWindowService#query}.
 *
 * <p><b>Python SDK usage:</b>
 * <ol>
 *   <li>{@code agentHasAssociation = false} → skip injection silently (not yet wired)</li>
 *   <li>{@code since > currentWindowSeq} → service restarted; SDK resets cursor to 0
 *       and skips this turn</li>
 *   <li>{@code lastEvictionWindowSeq > since} → inject overflow notice (additive with messages)</li>
 *   <li>{@code messages} non-empty → format and inject as context</li>
 *   <li>{@code messages} empty AND {@code lastChannelActivity} older than TTL → inject idle notice</li>
 * </ol>
 *
 * <p><b>Overflow and messages are additive</b> — both injected when overflow occurred but
 * newer messages still exist. Never use if/elif to choose between them.
 *
 * @param lastEvictionWindowSeq  {@code -1} if no eviction has occurred; otherwise the
 *                               {@code windowSeq} of the most recently evicted message
 *                               across all associated channels
 * @param lastWindowSeq          max windowSeq of returned messages; {@code since} if none returned
 * @param currentWindowSeq       service's {@code AtomicLong} value at query time — used by
 *                               SDK to detect a service restart (since > currentWindowSeq)
 * @param lastChannelActivity    {@code Instant.EPOCH} if no messages ever received on any
 *                               associated channel; never null
 */
public record WindowContent(
        List<ContextMessage> messages,
        long lastEvictionWindowSeq,
        long lastWindowSeq,
        long currentWindowSeq,
        boolean agentHasAssociation,
        Instant lastChannelActivity) {

    public static WindowContent noAssociation() {
        return new WindowContent(List.of(), -1L, 0L, 0L, false, Instant.EPOCH);
    }
}
```

---

## Task 2: ChannelRingBuffer — TDD

**Files:**
- Create: `core/src/test/java/io/casehub/openclaw/context/ChannelRingBufferTest.java`
- Create: `core/src/main/java/io/casehub/openclaw/context/ChannelRingBuffer.java`

`ChannelRingBuffer` is package-private — test is in the same package.

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import io.casehub.qhorus.api.message.MessageType;

import static org.assertj.core.api.Assertions.assertThat;

class ChannelRingBufferTest {

    static final int MAX_SIZE = 3;
    static final Duration TTL = Duration.ofMinutes(30);

    ChannelRingBuffer buffer;

    @BeforeEach
    void setup() {
        buffer = new ChannelRingBuffer(MAX_SIZE, TTL);
    }

    private ContextMessage msg(long seq) {
        return new ContextMessage(seq, UUID.randomUUID(), "test/channel",
                MessageType.STATUS, "sender-1", "corr-1", "content", Instant.now());
    }

    private ContextMessage expiredMsg(long seq) {
        Instant past = Instant.now().minus(Duration.ofHours(1));
        return new ContextMessage(seq, UUID.randomUUID(), "test/channel",
                MessageType.STATUS, "sender-1", "corr-1", "old content", past);
    }

    @Test
    void initialState_lastActivityIsEpoch() {
        assertThat(buffer.lastActivity()).isEqualTo(Instant.EPOCH);
    }

    @Test
    void initialState_lastEvictionWindowSeqIsMinusOne() {
        assertThat(buffer.lastEvictionWindowSeq()).isEqualTo(-1L);
    }

    @Test
    void query_emptyBuffer_returnsEmpty() {
        assertThat(buffer.query(0, Instant.now())).isEmpty();
    }

    @Test
    void add_withinCapacity_allReturned() {
        buffer.add(msg(1));
        buffer.add(msg(2));
        assertThat(buffer.query(0, Instant.now())).hasSize(2);
    }

    @Test
    void add_overflow_oldestEvicted_lastEvictionSeqUpdated() {
        buffer.add(msg(1));
        buffer.add(msg(2));
        buffer.add(msg(3));
        buffer.add(msg(4)); // overflows: seq=1 is evicted

        assertThat(buffer.lastEvictionWindowSeq()).isEqualTo(1L);
        assertThat(buffer.query(0, Instant.now()))
                .extracting(ContextMessage::windowSeq)
                .containsExactly(2L, 3L, 4L);
    }

    @Test
    void add_multipleOverflows_lastEvictionSeqIsNewest() {
        buffer.add(msg(1));
        buffer.add(msg(2));
        buffer.add(msg(3));
        buffer.add(msg(4)); // evicts seq=1, lastEviction=1
        buffer.add(msg(5)); // evicts seq=2, lastEviction=2

        assertThat(buffer.lastEvictionWindowSeq()).isEqualTo(2L);
    }

    @Test
    void query_withSince_returnsOnlyNewer() {
        buffer.add(msg(1));
        buffer.add(msg(2));
        buffer.add(msg(3));

        assertThat(buffer.query(2, Instant.now()))
                .extracting(ContextMessage::windowSeq)
                .containsExactly(3L);
    }

    @Test
    void query_ttlFilter_expiredMessagesExcluded() {
        buffer.add(expiredMsg(1));
        buffer.add(msg(2));

        assertThat(buffer.query(0, Instant.now()))
                .extracting(ContextMessage::windowSeq)
                .containsExactly(2L);
    }

    @Test
    void evictExpired_removesStaleEntries() {
        buffer.add(expiredMsg(1));
        buffer.add(msg(2));

        buffer.evictExpired(Instant.now());

        // After eviction, query without TTL filter should still work
        assertThat(buffer.query(0, Instant.now())).hasSize(1)
                .extracting(ContextMessage::windowSeq).containsExactly(2L);
    }

    @Test
    void lastActivity_updatedOnAdd() {
        Instant before = Instant.now().minusSeconds(1);
        buffer.add(msg(1));
        assertThat(buffer.lastActivity()).isAfter(before);
    }

    @Test
    void lastActivity_notUpdatedByEvictExpired() {
        buffer.add(msg(1));
        Instant afterAdd = buffer.lastActivity();
        buffer.evictExpired(Instant.now());
        assertThat(buffer.lastActivity()).isEqualTo(afterAdd);
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -Dtest=ChannelRingBufferTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD FAILURE — `ChannelRingBuffer` class does not exist.

- [ ] **Step 3: Implement ChannelRingBuffer**

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;

/**
 * Per-channel bounded ring buffer of {@link ContextMessage} entries.
 * Package-private — accessed only by {@link ChannelContextWindowService}.
 *
 * <p>All methods are synchronized on {@code this}. Contention is per-channel;
 * concurrent writes to different channels proceed without blocking each other.
 */
final class ChannelRingBuffer {

    private final int maxSize;
    private final Duration ttl;
    private final Deque<ContextMessage> messages = new ArrayDeque<>();

    private long lastEvictionWindowSeq = -1L;
    private Instant lastActivity = Instant.EPOCH;

    ChannelRingBuffer(int maxSize, Duration ttl) {
        this.maxSize = maxSize;
        this.ttl = ttl;
    }

    synchronized void add(ContextMessage message) {
        if (messages.size() >= maxSize) {
            ContextMessage evicted = messages.pollFirst();
            if (evicted != null) {
                lastEvictionWindowSeq = evicted.windowSeq();
            }
        }
        messages.addLast(message);
        lastActivity = message.receivedAt();
    }

    synchronized List<ContextMessage> query(long since, Instant now) {
        Instant ttlBoundary = now.minus(ttl);
        return messages.stream()
                .filter(m -> m.windowSeq() > since)
                .filter(m -> !m.receivedAt().isBefore(ttlBoundary))
                .toList();
    }

    synchronized void evictExpired(Instant now) {
        Instant ttlBoundary = now.minus(ttl);
        messages.removeIf(m -> m.receivedAt().isBefore(ttlBoundary));
    }

    synchronized long lastEvictionWindowSeq() {
        return lastEvictionWindowSeq;
    }

    synchronized Instant lastActivity() {
        return lastActivity;
    }
}
```

- [ ] **Step 4: Run tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -Dtest=ChannelRingBufferTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS — all `ChannelRingBufferTest` tests pass.

- [ ] **Step 5: Commit**

```bash
git add core/src/main/java/io/casehub/openclaw/context/ core/src/test/java/io/casehub/openclaw/context/ChannelRingBufferTest.java
git commit -m "feat(core): ChannelRingBuffer — per-channel bounded ring buffer with TTL and overflow tracking

Refs #3"
```

---

## Task 3: ChannelContextWindowService — TDD

**Files:**
- Create: `core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java`
- Create: `core/src/main/java/io/casehub/openclaw/context/ChannelContextWindowService.java`

Pure JUnit 5 — no Quarkus container. Config fields are set directly (package-private, same package).

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Set;
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

    @Test
    void query_unknownAgent_returnsNoAssociation() {
        WindowContent result = service.query("unknown-agent", 0L);
        assertThat(result.agentHasAssociation()).isFalse();
        assertThat(result.messages()).isEmpty();
    }

    @Test
    void add_unassociatedChannel_silentlyIgnored() {
        UUID channelId = UUID.randomUUID();
        // no associate() called — add() must be a silent no-op
        service.add(event(channelId, "unregistered/work", MessageType.STATUS));

        assertThat(service.query("any-agent", 0L).agentHasAssociation()).isFalse();
    }

    @Test
    void associate_add_query_roundTrip() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "test/work", MessageType.COMMAND));

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.agentHasAssociation()).isTrue();
        assertThat(result.messages()).hasSize(1);
        assertThat(result.messages().get(0).messageType()).isEqualTo(MessageType.COMMAND);
    }

    @Test
    void query_withSince_filtersCorrectly() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "test/work", MessageType.COMMAND));
        service.add(event(channelId, "test/work", MessageType.STATUS));
        service.add(event(channelId, "test/work", MessageType.DONE));

        // get the first message's seq
        long firstSeq = service.query("agent-1", 0L).messages().get(0).windowSeq();

        WindowContent result = service.query("agent-1", firstSeq);
        assertThat(result.messages()).hasSize(2);
        assertThat(result.messages())
                .extracting(ContextMessage::messageType)
                .containsExactly(MessageType.STATUS, MessageType.DONE);
    }

    @Test
    void multipleChannels_mergedSortedByWindowSeq() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        service.associate("agent-1", Set.of(ch1, ch2));

        service.add(event(ch1, "work", MessageType.COMMAND));
        service.add(event(ch2, "observe", MessageType.STATUS));
        service.add(event(ch1, "work", MessageType.DONE));

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.messages()).hasSize(3);

        List<Long> seqs = result.messages().stream()
                .map(ContextMessage::windowSeq).toList();
        assertThat(seqs).isSorted();
    }

    @Test
    void associate_twice_sameAgent_channelsAddedNotReplaced() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        service.associate("agent-1", Set.of(ch1));
        service.associate("agent-1", Set.of(ch2));

        service.add(event(ch1, "work", MessageType.STATUS));
        service.add(event(ch2, "observe", MessageType.STATUS));

        assertThat(service.query("agent-1", 0L).messages()).hasSize(2);
    }

    @Test
    void currentWindowSeq_reflectsGlobalCounter() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "work", MessageType.STATUS));
        service.add(event(channelId, "work", MessageType.STATUS));

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.currentWindowSeq()).isEqualTo(2L);
    }

    @Test
    void lastWindowSeq_isMaxReturnedSeq() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=2

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.lastWindowSeq()).isEqualTo(2L);
    }

    @Test
    void lastWindowSeq_isSince_whenNoMessagesReturned() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=1

        // query with since=1 — no messages with seq > 1
        WindowContent result = service.query("agent-1", 1L);
        assertThat(result.messages()).isEmpty();
        assertThat(result.lastWindowSeq()).isEqualTo(1L); // cursor stays
    }

    @Test
    void restartDetection_staleClientCursor() {
        // Simulates: service restarted (new instance, AtomicLong reset to 0).
        // Client holds since=50 from before restart.
        // Service has only processed 2 messages (seq 1 and 2).
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS)); // seq=2

        WindowContent result = service.query("agent-1", 50L);
        // currentWindowSeq=2; since=50 > 2 → SDK must detect restart and reset
        assertThat(result.currentWindowSeq()).isEqualTo(2L);
        assertThat(result.messages()).isEmpty(); // nothing with seq > 50
        assertThat(result.agentHasAssociation()).isTrue();
    }

    @Test
    void windowSeq_isGloballyMonotonic_acrossChannels() {
        UUID ch1 = UUID.randomUUID();
        UUID ch2 = UUID.randomUUID();
        service.associate("agent-1", Set.of(ch1, ch2));

        for (int i = 0; i < 5; i++) {
            service.add(event(i % 2 == 0 ? ch1 : ch2, "chan", MessageType.STATUS));
        }

        List<Long> seqs = service.query("agent-1", 0L).messages().stream()
                .map(ContextMessage::windowSeq).toList();
        for (int i = 1; i < seqs.size(); i++) {
            assertThat(seqs.get(i)).isGreaterThan(seqs.get(i - 1));
        }
    }

    @Test
    void lastEvictionWindowSeq_reflectsOverflow() {
        service.maxMessagesPerChannel = 2; // tiny buffer
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));

        service.add(event(channelId, "work", MessageType.COMMAND)); // seq=1
        service.add(event(channelId, "work", MessageType.STATUS));  // seq=2
        service.add(event(channelId, "work", MessageType.DONE));    // seq=3 → evicts seq=1

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.lastEvictionWindowSeq()).isEqualTo(1L);
    }

    @Test
    void lastChannelActivity_isEpoch_whenNoMessages() {
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.lastChannelActivity()).isEqualTo(Instant.EPOCH);
    }

    @Test
    void concurrency_noExceptionAndBounded() throws InterruptedException {
        service.maxMessagesPerChannel = 10;
        UUID channelId = UUID.randomUUID();
        service.associate("agent-1", Set.of(channelId));

        int threads = 10;
        int messagesPerThread = 50;
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(threads);

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                try {
                    for (int j = 0; j < messagesPerThread; j++) {
                        service.add(event(channelId, "work", MessageType.STATUS));
                    }
                } finally {
                    latch.countDown();
                }
            });
        }
        latch.await(10, TimeUnit.SECONDS);
        executor.shutdown();

        WindowContent result = service.query("agent-1", 0L);
        assertThat(result.messages()).hasSizeLessThanOrEqualTo(10);
        assertThat(result.agentHasAssociation()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -Dtest=ChannelContextWindowServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD FAILURE — `ChannelContextWindowService` does not exist.

- [ ] **Step 3: Implement ChannelContextWindowService**

```java
package io.casehub.openclaw.context;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashSet;
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
 * <p>{@link #associate} is called by {@code WorkerProvisioner} (Epic 4) when an
 * OpenClaw agent is provisioned for a case. Until then, {@link #query} returns
 * {@link WindowContent#noAssociation()} — the correct fail-open state.
 *
 * <p>{@link #add} is called by {@code ChannelContextWindowObserver} on every
 * dispatched Qhorus message. It is intentionally a no-op for unassociated channels.
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
    private final ConcurrentHashMap<String, Set<UUID>> agentChannels = new ConcurrentHashMap<>();

    /**
     * Registers channel associations for an agent. Additive — never removes channels.
     * Pre-creates ring buffers so {@link #add} can use a lock-free get().
     * Called by WorkerProvisioner (Epic 4).
     */
    public void associate(String agentId, Set<UUID> channelIds) {
        agentChannels.merge(agentId, new HashSet<>(channelIds), (existing, added) -> {
            existing.addAll(added);
            return existing;
        });
        channelIds.forEach(id ->
                buffers.computeIfAbsent(id, k -> new ChannelRingBuffer(maxMessagesPerChannel, ttl)));
    }

    /**
     * Ingests a Qhorus message into the ring buffer for its channel.
     * Silent no-op if the channel is not associated with any agent.
     * Called by ChannelContextWindowObserver.
     */
    public void add(MessageReceivedEvent event) {
        ChannelRingBuffer buffer = buffers.get(event.channelId());
        if (buffer == null) return; // unassociated channel — common case, O(1)
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
     * <p>Returns {@link WindowContent#noAssociation()} if the agent has no registered
     * channels — the correct fail-open state before Epic 4 wires associate().
     */
    public WindowContent query(String agentId, long since) {
        Set<UUID> channelIds = agentChannels.get(agentId);
        if (channelIds == null || channelIds.isEmpty()) {
            return WindowContent.noAssociation();
        }

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

    /**
     * Eagerly evicts TTL-expired entries from all buffers.
     * Called by {@code EvictionScheduler} (app module) at the TTL interval.
     */
    public void evictExpired() {
        Instant now = Instant.now();
        buffers.values().forEach(b -> b.evictExpired(now));
    }
}
```

- [ ] **Step 4: Run all core tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS — all `ChannelRingBufferTest` and `ChannelContextWindowServiceTest` pass.

- [ ] **Step 5: Commit**

```bash
git add core/src/main/java/io/casehub/openclaw/context/ core/src/test/java/io/casehub/openclaw/context/ChannelContextWindowServiceTest.java
git commit -m "feat(core): ChannelContextWindowService — in-memory ring buffer with associate/add/query

Refs #3"
```

---

## Task 4: ChannelContextWindowObserver — TDD

**Files:**
- Modify: `casehub/pom.xml` — add `mockito-core` test dep
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/ChannelContextWindowObserverTest.java`
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/ChannelContextWindowObserver.java`

Pure JUnit 5 + Mockito — no Quarkus container. The `service` field on the observer is package-private; test is in the same package and sets it directly.

- [ ] **Step 1: Add mockito-core test dep to casehub/pom.xml**

Open `casehub/pom.xml`. In the `<dependencies>` section, add after the assertj entry:

```xml
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-core</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.openclaw.casehub;

import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;
import io.casehub.qhorus.api.message.MessageType;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;

class ChannelContextWindowObserverTest {

    ChannelContextWindowService mockService;
    ChannelContextWindowObserver observer;

    @BeforeEach
    void setup() {
        mockService = mock(ChannelContextWindowService.class);
        observer = new ChannelContextWindowObserver();
        observer.service = mockService; // package-private field injection
    }

    private MessageReceivedEvent event(MessageType type) {
        String content = (type == MessageType.EVENT) ? null : "content";
        return new MessageReceivedEvent(
                "test/channel", UUID.randomUUID(), type, "sender", "corr-1", content);
    }

    @Test
    void eventMessages_notPassedToService() {
        observer.onMessage(event(MessageType.EVENT));
        verify(mockService, never()).add(any());
    }

    @ParameterizedTest
    @EnumSource(value = MessageType.class,
                names = {"COMMAND", "RESPONSE", "STATUS", "DONE",
                         "FAILURE", "DECLINE", "HANDOFF", "QUERY"})
    void agentVisibleTypes_passedToService(MessageType type) {
        MessageReceivedEvent e = event(type);
        observer.onMessage(e);
        verify(mockService).add(e);
    }

    @Test
    void serviceException_caughtNotPropagated() {
        doThrow(new RuntimeException("simulated failure")).when(mockService).add(any());
        assertThatCode(() -> observer.onMessage(event(MessageType.STATUS)))
                .doesNotThrowAnyException();
    }

    @Test
    void scope_returnsLocal() {
        assertThat(observer.scope()).isEqualTo(MessageObserver.Scope.LOCAL);
    }
}
```

- [ ] **Step 3: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=ChannelContextWindowObserverTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD FAILURE — `ChannelContextWindowObserver` does not exist.

- [ ] **Step 4: Implement ChannelContextWindowObserver**

```java
package io.casehub.openclaw.casehub;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.qhorus.api.gateway.MessageObserver;
import io.casehub.qhorus.api.gateway.MessageReceivedEvent;

/**
 * Feeds the ChannelContextWindow ring buffer from every Qhorus message dispatch.
 *
 * <p>EVENT messages are excluded — {@link io.casehub.qhorus.api.message.MessageType#isAgentVisible()}
 * returns false for EVENT; their content is null per PP-20260508-90428f.
 *
 * <p>Per the MessageObserver SPI contract: never query Qhorus state inside this method
 * (the dispatcher fires before the enclosing transaction commits). Never propagate exceptions.
 */
@ApplicationScoped
public class ChannelContextWindowObserver implements MessageObserver {

    private static final Logger log = Logger.getLogger(ChannelContextWindowObserver.class);

    @Inject
    ChannelContextWindowService service;

    @Override
    public void onMessage(MessageReceivedEvent event) {
        if (!event.messageType().isAgentVisible()) return;
        try {
            service.add(event);
        } catch (Exception e) {
            log.errorf(e, "ChannelContextWindow write failed for channel %s — ignoring",
                    event.channelName());
        }
    }
}
```

- [ ] **Step 5: Run tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=ChannelContextWindowObserverTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS — all `ChannelContextWindowObserverTest` tests pass.

- [ ] **Step 6: Commit**

```bash
git add casehub/pom.xml casehub/src/main/java/io/casehub/openclaw/casehub/ casehub/src/test/java/io/casehub/openclaw/casehub/
git commit -m "feat(casehub): ChannelContextWindowObserver — MessageObserver SPI feeding ring buffer

Refs #3"
```

---

## Task 5: App Module — Deps and Config

**Files:**
- Modify: `app/pom.xml` — add `quarkus-scheduler` and `quarkus-junit5-mockito`
- Modify/Create: `app/src/main/resources/application.properties`

- [ ] **Step 1: Add quarkus-scheduler to app/pom.xml**

Open `app/pom.xml`. In the `<dependencies>` section, after the `quarkus-rest-jackson` entry:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-scheduler</artifactId>
    </dependency>
```

- [ ] **Step 2: Add quarkus-junit5-mockito test dep to app/pom.xml**

In the test dependencies section of `app/pom.xml`, after `quarkus-junit`:

```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5-mockito</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 3: Update application.properties**

Check if `app/src/main/resources/application.properties` exists. If not, create the directory and file. Add these entries (do not remove any existing entries):

```properties
# ChannelContextWindow ring buffer configuration
casehub.openclaw.context-window.max-messages-per-channel=100
casehub.openclaw.context-window.ttl=PT30M

# Jackson: serialize dates as ISO-8601 strings (not timestamps)
quarkus.jackson.write-dates-as-timestamps=false
```

- [ ] **Step 4: Verify app module compiles**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app -am
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add app/pom.xml app/src/main/resources/
git commit -m "chore(app): add quarkus-scheduler + mockito deps, context-window config

Refs #3"
```

---

## Task 6: ChannelContextWindowResource and EvictionScheduler — TDD

**Files:**
- Create: `app/src/test/java/io/casehub/openclaw/app/ChannelContextWindowResourceTest.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/ChannelContextWindowResource.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/EvictionScheduler.java`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.openclaw.app;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.openclaw.context.ContextMessage;
import io.casehub.openclaw.context.WindowContent;
import io.casehub.qhorus.api.message.MessageType;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.hasSize;
import static org.hamcrest.Matchers.is;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@QuarkusTest
class ChannelContextWindowResourceTest {

    @InjectMock
    ChannelContextWindowService service;

    private WindowContent contentWithOneMessage() {
        ContextMessage msg = new ContextMessage(
                1L, UUID.randomUUID(), "test/work", MessageType.STATUS,
                "finance-agent", "corr-1", "Budget warning", Instant.now());
        return new WindowContent(
                List.of(msg), -1L, 1L, 1L, true,
                Instant.now().minus(Duration.ofMinutes(5)));
    }

    @Test
    void get_knownAgent_returns200WithMessages() {
        when(service.query("test-agent", 0L)).thenReturn(contentWithOneMessage());

        given()
                .when().get("/channel-context/test-agent")
                .then()
                .statusCode(200)
                .body("agentHasAssociation", is(true))
                .body("messages", hasSize(1))
                .body("messages[0].senderId", is("finance-agent"))
                .body("messages[0].messageType", is("STATUS"))
                .body("currentWindowSeq", is(1))
                .body("lastEvictionWindowSeq", is(-1));
    }

    @Test
    void get_unknownAgent_returns200NotAssociated() {
        when(service.query(anyString(), anyLong()))
                .thenReturn(WindowContent.noAssociation());

        given()
                .when().get("/channel-context/ghost-agent")
                .then()
                .statusCode(200)
                .body("agentHasAssociation", is(false))
                .body("messages", hasSize(0));
    }

    @Test
    void get_sinceParamPassedThrough() {
        when(service.query("agent-1", 42L)).thenReturn(WindowContent.noAssociation());

        given()
                .when().get("/channel-context/agent-1?since=42")
                .then()
                .statusCode(200);

        verify(service).query("agent-1", 42L);
    }

    @Test
    void get_sinceDefaultsToZero() {
        when(service.query("agent-1", 0L)).thenReturn(WindowContent.noAssociation());

        given()
                .when().get("/channel-context/agent-1")
                .then()
                .statusCode(200);

        verify(service).query("agent-1", 0L);
    }
}
```

- [ ] **Step 2: Run tests — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ChannelContextWindowResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD FAILURE — `ChannelContextWindowResource` and `EvictionScheduler` do not exist, `@InjectMock` not wired.

- [ ] **Step 3: Implement ChannelContextWindowResource**

```java
package io.casehub.openclaw.app;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.DefaultValue;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.QueryParam;
import jakarta.ws.rs.core.MediaType;

import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.openclaw.context.WindowContent;

/**
 * REST endpoint for ChannelContextWindow queries.
 *
 * <p>Always returns 200. An unknown agentId is not an error — it is the fail-open state
 * ({@link WindowContent#noAssociation()}). The Python SDK treats agentHasAssociation=false
 * as a silent skip. 404 would be semantically wrong.
 *
 * <p>{@code since} defaults to 0 when omitted — returns all buffered messages.
 */
@Path("/channel-context")
@Produces(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class ChannelContextWindowResource {

    @Inject
    ChannelContextWindowService service;

    @GET
    @Path("/{agentId}")
    public WindowContent query(
            @PathParam("agentId") String agentId,
            @QueryParam("since") @DefaultValue("0") long since) {
        return service.query(agentId, since);
    }
}
```

- [ ] **Step 4: Implement EvictionScheduler**

```java
package io.casehub.openclaw.app;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.openclaw.context.ChannelContextWindowService;
import io.quarkus.scheduler.Scheduled;

/**
 * Eagerly evicts TTL-expired entries from ChannelContextWindow ring buffers.
 *
 * <p>Lives in app/ rather than core/ to keep quarkus-scheduler off the library module.
 * Runs at the TTL interval — not a separate config to avoid the misconfiguration risk
 * of an eviction interval that exceeds the TTL.
 */
@ApplicationScoped
public class EvictionScheduler {

    @Inject
    ChannelContextWindowService service;

    @Scheduled(every = "${casehub.openclaw.context-window.ttl:PT30M}")
    void evict() {
        service.evictExpired();
    }
}
```

- [ ] **Step 5: Run tests — expect green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ChannelContextWindowResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS — all `ChannelContextWindowResourceTest` tests pass.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/openclaw/app/ app/src/test/java/io/casehub/openclaw/app/
git commit -m "feat(app): ChannelContextWindowResource REST endpoint + EvictionScheduler

GET /channel-context/{agentId}?since={seq} — always 200, fail-open.
EvictionScheduler calls service.evictExpired() at the configured TTL interval.

Refs #3"
```

---

## Task 7: Full Build Validation

- [ ] **Step 1: Run full build across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS — all modules compile and all tests pass.

- [ ] **Step 2: Verify test counts**

In the Surefire output, confirm:
- `core` module: at minimum `ChannelRingBufferTest` (11 tests) + `ChannelContextWindowServiceTest` (13 tests) + existing hook client tests all pass
- `casehub` module: at minimum `ChannelContextWindowObserverTest` (10 tests) passes
- `app` module: at minimum `ChannelContextWindowResourceTest` (4 tests) passes

- [ ] **Step 3: If any test fails, fix before proceeding**

Do not proceed to code review with failing tests.
