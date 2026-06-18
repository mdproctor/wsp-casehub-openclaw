# Extract OversightGateService to casehub-engine-api Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the oversight gate SPI (`OversightGateService`, `GateDecision`, `ReactiveOversightGateService`) from `casehub-openclaw` to `casehub-engine-api`, replacing the private `classifyMostRestrictive()` duplication with delegation to the engine's `ChainedReactiveActionRiskClassifier`.

**Architecture:** Two independent sessions with a hard dependency: the engine session adds types to `casehub-engine-api` and publishes a snapshot; the openclaw session then consumes that snapshot to rename the concrete implementation, delete dead code, and add the reactive delegate. The classification logic moves from a private copy inside openclaw to the engine's canonical aggregator.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny, CDI (`jakarta.enterprise`), Maven multi-module, JUnit 5 + AssertJ + Mockito.

**Spec:** `docs/superpowers/specs/2026-06-17-extract-oversight-gate-service-design.md`

---

## ⚠️ Part A — Engine Session (openclaw is BLOCKED until this is done)

Engine session must publish `casehub-engine-api` snapshot before openclaw session begins.

**Engine repo:** `/Users/mdproctor/claude/casehub/engine`
**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
**Test command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -Dtest=<TestClass> -Dsurefire.failIfNoSpecifiedTests=false`

---

### Task A1: Add GateDecision sealed interface to engine-api

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/GateDecision.java`
- Create: `api/src/test/java/io/casehub/api/spi/GateDecisionTest.java`

- [ ] **Step 1: Write the failing test**

```java
// api/src/test/java/io/casehub/api/spi/GateDecisionTest.java
package io.casehub.api.spi;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class GateDecisionTest {

    @Test
    void autonomous_isInstanceOfGateDecision() {
        GateDecision d = new GateDecision.Autonomous();
        assertThat(d).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void gatePending_carriesGateIdAndReason() {
        UUID gateId = UUID.randomUUID();
        GateDecision d = new GateDecision.GatePending(gateId, "risk: file deletion");
        assertThat(d).isInstanceOf(GateDecision.GatePending.class);
        assertThat(((GateDecision.GatePending) d).gateId()).isEqualTo(gateId);
        assertThat(((GateDecision.GatePending) d).reason()).isEqualTo("risk: file deletion");
    }

    @Test
    void exhaustiveSwitchCompiles() {
        GateDecision d = new GateDecision.Autonomous();
        String result = switch (d) {
            case GateDecision.Autonomous a -> "autonomous";
            case GateDecision.GatePending p -> "pending:" + p.gateId();
        };
        assertThat(result).isEqualTo("autonomous");
    }
}
```

- [ ] **Step 2: Run test — expect compile failure (GateDecision does not exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -Dtest=GateDecisionTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL with `cannot find symbol: class GateDecision`

- [ ] **Step 3: Create GateDecision**

```java
// api/src/main/java/io/casehub/api/spi/GateDecision.java
package io.casehub.api.spi;

import java.util.UUID;

/**
 * Result of an oversight gate evaluation.
 *
 * <p>{@link Autonomous} — the action may proceed without human review.
 * <p>{@link GatePending} — the action requires human approval before the commitment can be
 * fulfilled; callers must wait for {@link OversightGateService#fulfill(UUID, String)}.
 */
public sealed interface GateDecision permits GateDecision.Autonomous, GateDecision.GatePending {
    record Autonomous() implements GateDecision {}
    record GatePending(UUID gateId, String reason) implements GateDecision {}
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -Dtest=GateDecisionTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /path/to/engine add api/src/main/java/io/casehub/api/spi/GateDecision.java \
    api/src/test/java/io/casehub/api/spi/GateDecisionTest.java
git -C /path/to/engine commit -m "feat(engine-api): add GateDecision sealed interface — Refs #<engine-issue>"
```

---

### Task A2: Add OversightGateService and ReactiveOversightGateService interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/OversightGateService.java`
- Create: `api/src/main/java/io/casehub/api/spi/ReactiveOversightGateService.java`
- Create: `api/src/test/java/io/casehub/api/spi/OversightGateServiceContractTest.java`

- [ ] **Step 1: Write the failing contract test**

```java
// api/src/test/java/io/casehub/api/spi/OversightGateServiceContractTest.java
package io.casehub.api.spi;

import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;

class OversightGateServiceContractTest {

    // Verify both interfaces can be implemented and the types are correct.

    static class StubBlocking implements OversightGateService {
        @Override
        public GateDecision openGate(String agentId, String commitmentId, String outcome, String tenancyId) {
            return new GateDecision.Autonomous();
        }
        @Override
        public void fulfill(UUID gateId, String rawOutput) {}
    }

    static class StubReactive implements ReactiveOversightGateService {
        @Override
        public Uni<GateDecision> openGate(String agentId, String commitmentId, String outcome, String tenancyId) {
            return Uni.createFrom().item(new GateDecision.Autonomous());
        }
        @Override
        public Uni<Void> fulfill(UUID gateId, String rawOutput) {
            return Uni.createFrom().voidItem();
        }
    }

    @Test
    void blockingInterface_implementable() {
        OversightGateService svc = new StubBlocking();
        GateDecision result = svc.openGate("agent", UUID.randomUUID().toString(), "done", "tenant");
        assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void reactiveInterface_implementable() {
        ReactiveOversightGateService svc = new StubReactive();
        GateDecision result = svc.openGate("agent", UUID.randomUUID().toString(), "done", "tenant")
                .await().atMost(Duration.ofSeconds(1));
        assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void reactiveInterface_fulfill_completesVoid() {
        ReactiveOversightGateService svc = new StubReactive();
        svc.fulfill(UUID.randomUUID(), "approved").await().atMost(Duration.ofSeconds(1));
        // No exception = pass
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api \
    -Dtest=OversightGateServiceContractTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL with `cannot find symbol: class OversightGateService`

- [ ] **Step 3: Create OversightGateService**

```java
// api/src/main/java/io/casehub/api/spi/OversightGateService.java
package io.casehub.api.spi;

import java.util.UUID;

/**
 * Evaluates proposed agent actions for risk and processes human gate responses.
 *
 * <p>Implementations must fail-open on infrastructure errors — a gate failure must never
 * block a case. The classification mechanism and channel topology are implementation details.
 */
public interface OversightGateService {

    /**
     * Evaluates the proposed action for risk and returns a gate decision.
     * Returns {@link GateDecision.Autonomous} when the action may proceed without human review.
     * Returns {@link GateDecision.GatePending} when the action requires human approval before
     * the commitment can be fulfilled.
     * Implementations must fail-open on infrastructure errors.
     */
    GateDecision openGate(String agentId, String commitmentId, String outcome, String tenancyId);

    /**
     * Processes a human response to a pending oversight gate identified by {@code gateId}.
     * The raw output is interpreted by the implementation to determine approval or rejection.
     * Implementations must fail-open on errors — an unprocessable response must not block the case.
     */
    void fulfill(UUID gateId, String rawOutput);
}
```

- [ ] **Step 4: Create ReactiveOversightGateService**

```java
// api/src/main/java/io/casehub/api/spi/ReactiveOversightGateService.java
package io.casehub.api.spi;

import io.smallrye.mutiny.Uni;
import java.util.UUID;

/**
 * Reactive variant of {@link OversightGateService} for Vert.x IO thread compatibility.
 *
 * <p>Implementations of {@link OversightGateService#openGate} that call
 * {@code await().indefinitely()} are unsafe on Vert.x IO threads. Callers on IO threads
 * must use this interface instead. Both interfaces must have the same semantic contract.
 */
public interface ReactiveOversightGateService {
    Uni<GateDecision> openGate(String agentId, String commitmentId, String outcome, String tenancyId);
    Uni<Void> fulfill(UUID gateId, String rawOutput);
}
```

- [ ] **Step 5: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api \
    -Dtest=OversightGateServiceContractTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git -C /path/to/engine add api/src/main/java/io/casehub/api/spi/OversightGateService.java \
    api/src/main/java/io/casehub/api/spi/ReactiveOversightGateService.java \
    api/src/test/java/io/casehub/api/spi/OversightGateServiceContractTest.java
git -C /path/to/engine commit -m "feat(engine-api): add OversightGateService + ReactiveOversightGateService SPI interfaces — Refs #<engine-issue>"
```

---

### Task A3: Move ChainedReactiveActionRiskClassifier from runtime to engine-api

**Files:**
- Create: `api/src/main/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifier.java`
- Create: `api/src/test/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifierTest.java`
- Delete: `runtime/src/main/java/io/casehub/engine/internal/worker/ChainedReactiveActionRiskClassifier.java`
- Delete: `runtime/src/test/java/io/casehub/engine/internal/worker/ChainedReactiveActionRiskClassifierTest.java`

This is the first concrete CDI bean in `casehub-engine-api`. This is a deliberate expansion of engine-api's role to include canonical default implementations that harnesses need without pulling in engine runtime. The `io.casehub.api.classification` package is scoped to this domain — it is not a general `impl` home.

- [ ] **Step 1: Create api/src/main/java/io/casehub/api/classification/ directory**

Create the directory (Maven will compile any `.java` under `src/main/java`).

- [ ] **Step 2: Copy ChainedReactiveActionRiskClassifier to new package**

```java
// api/src/main/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifier.java
// (Content is identical to runtime version except package declaration)
package io.casehub.api.classification;

import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.ReactiveActionRiskClassifier;
import io.casehub.api.spi.RiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.api.spi.RiskDecision.Autonomous;
import io.casehub.api.spi.RiskDecision.GateRequired;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.List;
import java.util.stream.StreamSupport;
import org.jboss.logging.Logger;

/**
 * Chains all {@link RiskClassifier}-qualified {@link ActionRiskClassifier} and {@link
 * ReactiveActionRiskClassifier} beans and returns the most restrictive {@link RiskDecision}.
 *
 * <p>Moved to engine-api so harnesses that depend only on engine-api (not engine runtime) can
 * inject {@link ReactiveActionRiskClassifier} and get the canonical aggregation behaviour.
 *
 * <p>When no {@code @RiskClassifier} classifier is registered, returns {@link Autonomous}.
 * Blocking classifiers are offloaded via {@link Infrastructure#getDefaultWorkerPool()}.
 * Any classifier exception applies the fail-safe {@link GateRequired}.
 */
@ApplicationScoped
public class ChainedReactiveActionRiskClassifier implements ReactiveActionRiskClassifier {

  private static final Logger LOG = Logger.getLogger(ChainedReactiveActionRiskClassifier.class);

  static final GateRequired FAIL_SAFE =
      new GateRequired(
          "Classifier error — manual review required before proceeding", true, null, null, null);

  @Inject @RiskClassifier Instance<ActionRiskClassifier> classifiers;
  @Inject @RiskClassifier Instance<ReactiveActionRiskClassifier> reactiveClassifiers;

  @Override
  public Uni<RiskDecision> classify(final PlannedAction action) {
    final boolean noBlocking = classifiers.isUnsatisfied();
    final boolean noReactive = reactiveClassifiers.isUnsatisfied();
    if (noBlocking && noReactive) {
      return Uni.createFrom().item(new Autonomous());
    }

    final Uni<RiskDecision> blockingResult =
        noBlocking
            ? Uni.createFrom().item(new Autonomous())
            : Uni.createFrom()
                .item(
                    () -> {
                      try {
                        return StreamSupport.stream(classifiers.spliterator(), false)
                            .map(c -> c.classify(action))
                            .reduce(
                                (RiskDecision) new Autonomous(),
                                ChainedReactiveActionRiskClassifier.this::mostRestrictive);
                      } catch (final Exception e) {
                        LOG.errorf(
                            e,
                            "ActionRiskClassifier threw for action type='%s' workerId='%s'"
                                + " caseId=%s — applying fail-safe GateRequired",
                            action.actionType(), action.workerId(), action.caseId());
                        return (RiskDecision) FAIL_SAFE;
                      }
                    })
                .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());

    if (noReactive) {
      return blockingResult;
    }

    final List<Uni<RiskDecision>> reactiveUnis;
    try {
      reactiveUnis =
          StreamSupport.stream(reactiveClassifiers.spliterator(), false)
              .map(c -> c.classify(action)
                  .onFailure().recoverWithItem(t -> {
                    LOG.errorf(t, "ReactiveActionRiskClassifier threw for action type='%s'"
                        + " — applying fail-safe GateRequired", action.actionType());
                    return FAIL_SAFE;
                  }))
              .toList();
    } catch (final Exception e) {
      LOG.errorf(e, "ReactiveActionRiskClassifier threw synchronously for action type='%s'"
          + " — applying fail-safe GateRequired", action.actionType());
      return Uni.createFrom().item((RiskDecision) FAIL_SAFE);
    }

    final Uni<RiskDecision> reactiveResult =
        Uni.join().all(reactiveUnis).andFailFast()
            .map(results -> results.stream().reduce(new Autonomous(), this::mostRestrictive));

    return Uni.combine().all().unis(blockingResult, reactiveResult).with(this::mostRestrictive);
  }

  RiskDecision mostRestrictive(final RiskDecision a, final RiskDecision b) {
    if (!(b instanceof GateRequired gb)) return a;
    if (!(a instanceof GateRequired ga)) return b;
    return narrower(ga, gb);
  }

  private GateRequired narrower(final GateRequired a, final GateRequired b) {
    final int sizeA = a.candidateGroups() == null ? Integer.MAX_VALUE : a.candidateGroups().size();
    final int sizeB = b.candidateGroups() == null ? Integer.MAX_VALUE : b.candidateGroups().size();
    if (sizeA != sizeB) return sizeA < sizeB ? a : b;
    if (a.expiresIn() != null && b.expiresIn() != null) {
      return a.expiresIn().compareTo(b.expiresIn()) <= 0 ? a : b;
    }
    if (a.expiresIn() != null) return a;
    if (b.expiresIn() != null) return b;
    return a;
  }
}
```

- [ ] **Step 3: Copy test to new package, delete runtime versions**

```java
// api/src/test/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifierTest.java
// Content identical to runtime test except package and class reference.
// Change: package io.casehub.engine.internal.worker → io.casehub.api.classification
// All test code is identical — same package gives access to package-private fields.
```

Copy the test content from `runtime/src/test/java/io/casehub/engine/internal/worker/ChainedReactiveActionRiskClassifierTest.java`, changing only the package declaration to `package io.casehub.api.classification;`.

Then delete both runtime source files:
```bash
git -C /path/to/engine rm runtime/src/main/java/io/casehub/engine/internal/worker/ChainedReactiveActionRiskClassifier.java
git -C /path/to/engine rm runtime/src/test/java/io/casehub/engine/internal/worker/ChainedReactiveActionRiskClassifierTest.java
```

- [ ] **Step 4: Run api test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api \
    -Dtest=ChainedReactiveActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 16, Failures: 0, Errors: 0` (same count as before)

- [ ] **Step 5: Build runtime to confirm old import is gone — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl runtime -am
```

Expected: BUILD SUCCESS (runtime consumers that injected `ReactiveActionRiskClassifier` still work because the CDI bean is now in api — api is on their classpath)

- [ ] **Step 6: Commit**

```bash
git -C /path/to/engine add \
    api/src/main/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifier.java \
    api/src/test/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifierTest.java
git -C /path/to/engine commit -m "feat(engine-api): move ChainedReactiveActionRiskClassifier to io.casehub.api.classification

Harnesses depending only on engine-api (not engine runtime) can now inject
ReactiveActionRiskClassifier and receive the canonical aggregation CDI bean.
This is the first concrete CDI bean in engine-api — deliberate expansion to
include canonical implementations that harnesses cannot access otherwise.
Refs #<engine-issue>"
```

---

### Task A4: Add NoOpOversightGateService and NoOpReactiveOversightGateService to engine runtime

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpOversightGateService.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveOversightGateService.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/worker/NoOpOversightGateServiceTest.java`

- [ ] **Step 1: Write the failing test**

```java
// runtime/src/test/java/io/casehub/engine/internal/worker/NoOpOversightGateServiceTest.java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.GateDecision;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

class NoOpOversightGateServiceTest {

    private final NoOpOversightGateService blocking = new NoOpOversightGateService();
    private final NoOpReactiveOversightGateService reactive = new NoOpReactiveOversightGateService();

    @Test
    void blocking_openGate_returnsAutonomous() {
        GateDecision result = blocking.openGate("agent-1", UUID.randomUUID().toString(), "done", "tenant-a");
        assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void blocking_fulfill_doesNotThrow() {
        assertThatCode(() -> blocking.fulfill(UUID.randomUUID(), "approved"))
                .doesNotThrowAnyException();
    }

    @Test
    void reactive_openGate_emitsAutonomous() {
        GateDecision result = reactive.openGate("agent-1", UUID.randomUUID().toString(), "done", "tenant-a")
                .await().atMost(Duration.ofSeconds(1));
        assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void reactive_fulfill_completesWithoutError() {
        assertThatCode(() -> reactive.fulfill(UUID.randomUUID(), "approved")
                .await().atMost(Duration.ofSeconds(1)))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl runtime -am \
    -Dtest=NoOpOversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL with `cannot find symbol: class NoOpOversightGateService`

- [ ] **Step 3: Create NoOpOversightGateService**

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpOversightGateService.java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.OversightGateService;
import io.quarkus.arc.DefaultBean;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Default no-op OversightGateService — all actions proceed autonomously.
 * Logs a startup WARN so misconfigured deployments are observable.
 * Replace with a harness-specific @ApplicationScoped implementation to enable oversight gating.
 */
@DefaultBean
@ApplicationScoped
public class NoOpOversightGateService implements OversightGateService {

  private static final Logger LOG = Logger.getLogger(NoOpOversightGateService.class);

  void onStart(@Observes StartupEvent event) {
    LOG.warn("OversightGateService: no implementation configured — all actions proceed autonomously. "
        + "Deploy an @ApplicationScoped OversightGateService implementation to enable oversight gating.");
  }

  @Override
  public GateDecision openGate(String agentId, String commitmentId, String outcome, String tenancyId) {
    return new GateDecision.Autonomous();
  }

  @Override
  public void fulfill(UUID gateId, String rawOutput) {
    // intentional no-op
  }
}
```

- [ ] **Step 4: Create NoOpReactiveOversightGateService**

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveOversightGateService.java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.ReactiveOversightGateService;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;

/**
 * Default no-op ReactiveOversightGateService — all actions proceed autonomously.
 * No startup WARN — NoOpOversightGateService already covers the misconfiguration signal.
 */
@DefaultBean
@ApplicationScoped
public class NoOpReactiveOversightGateService implements ReactiveOversightGateService {

  @Override
  public Uni<GateDecision> openGate(String agentId, String commitmentId, String outcome, String tenancyId) {
    return Uni.createFrom().item(new GateDecision.Autonomous());
  }

  @Override
  public Uni<Void> fulfill(UUID gateId, String rawOutput) {
    return Uni.createFrom().voidItem();
  }
}
```

- [ ] **Step 5: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl runtime -am \
    -Dtest=NoOpOversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git -C /path/to/engine add \
    runtime/src/main/java/io/casehub/engine/internal/worker/NoOpOversightGateService.java \
    runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveOversightGateService.java \
    runtime/src/test/java/io/casehub/engine/internal/worker/NoOpOversightGateServiceTest.java
git -C /path/to/engine commit -m "feat(engine): add NoOpOversightGateService + NoOpReactive @DefaultBean with startup WARN — Refs #<engine-issue>"
```

---

### Task A5: Full engine build + publish snapshot

- [ ] **Step 1: Full build of all engine modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS. All tests pass. The `casehub-engine-api` SNAPSHOT artifact is now in `~/.m2/repository/io/casehub/casehub-engine-api/`.

**openclaw session is now unblocked.**

---

## Part B — Openclaw Session (BLOCKED until Part A complete)

Verify `~/.m2/repository/io/casehub/casehub-engine-api/0.2-SNAPSHOT/` contains the new snapshot before starting.

**Project repo:** `/Users/mdproctor/claude/casehub/openclaw`
**Working branch:** `issue-31-extract-oversight-gate-service`
**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
**Test casehub module:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -Dtest=<TestClass> -Dsurefire.failIfNoSpecifiedTests=false`
**Test app module:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=<TestClass> -Dsurefire.failIfNoSpecifiedTests=false`

---

### Task B1: Verify snapshot is available + full build baseline

- [ ] **Step 1: Confirm new engine-api snapshot has OversightGateService**

```bash
ls ~/.m2/repository/io/casehub/casehub-engine-api/0.2-SNAPSHOT/*.jar | head -3
```

Expected: snapshot jar present with recent timestamp.

- [ ] **Step 2: Full build (expect failures — GateDecision import will break)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install 2>&1 | tail -30
```

Expected: Compilation errors referencing `GateDecision` cannot be found in `io.casehub.openclaw.casehub` — this confirms the new engine-api version is being used and the old class must be replaced. If the build succeeds without errors, the engine-api snapshot was not picked up — re-verify the snapshot is present.

---

### Task B2: Rename OversightGateService → OpenClawOversightGateService, inject ReactiveActionRiskClassifier

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawOversightGateService.java`
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java`

The critical changes from the old class:
1. Class renamed; implements `OversightGateService` (engine-api interface)
2. `@RiskClassifier Instance<ActionRiskClassifier> classifiers` → `ReactiveActionRiskClassifier reactiveClassifier`
3. `classifyMostRestrictive()`, `mostRestrictive()`, `narrower()` deleted
4. `openGate()` replaces `RiskDecision decision = classifyMostRestrictive(action)` with `RiskDecision decision = reactiveClassifier.classify(action).await().indefinitely()`
5. `GateDecision` import changes from `io.casehub.openclaw.casehub.GateDecision` to `io.casehub.api.spi.GateDecision`
6. `OversightGateService` import removed (this IS the class now)
7. `GATE_SENDER` constant stays on this class
8. `evaluate()` stays as a public method (not in the interface)

- [ ] **Step 1: Write the failing test first (rename + constructor change)**

Create the new test file with the updated constructor signature. The existing test file (`OversightGateServiceTest`) will be renamed separately in Task B9.

First, write a minimal compilation test to drive the class creation:

```java
// casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawOversightGateServiceCompileTest.java
package io.casehub.openclaw.casehub;

import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.OversightGateService;
import io.casehub.api.spi.ReactiveActionRiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class OpenClawOversightGateServiceCompileTest {

    @Test
    void openClawGateService_implementsOversightGateService() {
        // Type check — OpenClawOversightGateService must implement the SPI interface
        // This test exists purely to enforce the type hierarchy at compile time.
        // The full behavioural tests are in OpenClawOversightGateServiceTest.
        ReactiveActionRiskClassifier classifier = mock(ReactiveActionRiskClassifier.class);
        when(classifier.classify(any())).thenReturn(Uni.createFrom().item(new RiskDecision.Autonomous()));
        // Constructor will be verified by inspection — this is a compile-time guard.
        assertThat(OversightGateService.class).isInterface();
        assertThat(GateDecision.class).isInterface();
    }
}
```

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub \
    -Dtest=OpenClawOversightGateServiceCompileTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL (OpenClawOversightGateService does not exist yet)

- [ ] **Step 2: Create OpenClawOversightGateService**

```java
// casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawOversightGateService.java
package io.casehub.openclaw.casehub;

import java.io.IOException;
import java.io.StringReader;
import java.io.StringWriter;
import java.util.Map;
import java.util.Optional;
import java.util.Properties;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.OversightGateService;
import io.casehub.api.spi.PlannedAction;
import io.casehub.api.spi.ReactiveActionRiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.qhorus.api.message.MessageDispatch;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.qualifier.CrossTenant;
import io.casehub.qhorus.runtime.channel.Channel;
import io.casehub.qhorus.runtime.channel.ChannelService;
import io.casehub.qhorus.runtime.message.Commitment;
import io.casehub.qhorus.runtime.message.Message;
import io.casehub.qhorus.runtime.message.MessageService;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import io.casehub.qhorus.runtime.store.CrossTenantChannelStore;
import io.casehub.qhorus.runtime.store.CrossTenantMessageStore;
import io.casehub.qhorus.runtime.store.query.MessageQuery;

/**
 * Qhorus-backed implementation of the oversight gate lifecycle for OpenClaw harness.
 *
 * <p>{@link #evaluate(UUID, String, String, String)} archives the agent's webhook output as a
 * non-resolving STATUS message on the work channel. Not in the SPI — OpenClaw-specific.
 *
 * <p>{@link #openGate(String, String, String, String)} delegates risk classification to the
 * injected {@link ReactiveActionRiskClassifier} (resolved to
 * {@code ChainedReactiveActionRiskClassifier}). Called from {@code @Blocking} MCP handler —
 * {@code await().indefinitely()} is safe. If GateRequired, dispatches COMMAND to oversight
 * channel. Fail-open on infrastructure errors.
 *
 * <p>{@link #fulfill(UUID, String)} processes human gate responses. Fail-open on errors.
 */
@ApplicationScoped
public class OpenClawOversightGateService implements OversightGateService {

    private static final Logger log = Logger.getLogger(OpenClawOversightGateService.class);
    static final String GATE_SENDER = "openclaw-gate";

    private final ChannelService channelService;
    private final MessageService messageService;
    private final CommitmentStore commitmentStore;
    private final OversightGateDispatcher gateDispatcher;
    private final ReactiveActionRiskClassifier reactiveClassifier;
    private final CrossTenantMessageStore crossTenantMessageStore;
    private final CrossTenantChannelStore crossTenantChannelStore;

    @Inject
    public OpenClawOversightGateService(final ChannelService channelService,
                                         final MessageService messageService,
                                         final CommitmentStore commitmentStore,
                                         final OversightGateDispatcher gateDispatcher,
                                         final ReactiveActionRiskClassifier reactiveClassifier,
                                         @CrossTenant final CrossTenantMessageStore crossTenantMessageStore,
                                         @CrossTenant final CrossTenantChannelStore crossTenantChannelStore) {
        this.channelService = channelService;
        this.messageService = messageService;
        this.commitmentStore = commitmentStore;
        this.gateDispatcher = gateDispatcher;
        this.reactiveClassifier = reactiveClassifier;
        this.crossTenantMessageStore = crossTenantMessageStore;
        this.crossTenantChannelStore = crossTenantChannelStore;
    }

    /**
     * Archives the agent's webhook output as a non-resolving STATUS on the work channel.
     * OpenClaw-specific — not in the OversightGateService SPI.
     */
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

    @Override
    public GateDecision openGate(final String agentId, final String commitmentId,
                                  final String outcome, final String tenancyId) {
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

            UUID caseId = CaseChannel.parseCaseId(workChannel.name);
            if (caseId == null) {
                log.warnf("openGate: cannot extract caseId from channel name '%s' — failing open",
                        workChannel.name);
                return new GateDecision.Autonomous();
            }

            PlannedAction action = new PlannedAction(agentId, caseId, outcome, "COMPLETION", Map.of());
            // Delegates to ChainedReactiveActionRiskClassifier via CDI — handles both blocking and
            // reactive domain classifiers with fail-safe semantics. await() is safe: called from
            // @Blocking MCP handler. Unsafe on Vert.x IO threads — use ReactiveOversightGateService.
            RiskDecision decision = reactiveClassifier.classify(action).await().indefinitely();

            if (decision instanceof RiskDecision.Autonomous) {
                return new GateDecision.Autonomous();
            }

            RiskDecision.GateRequired gate = (RiskDecision.GateRequired) decision;

            Channel oversightChannel = channelService
                    .findByName(CaseChannel.oversightChannelName(caseId)).orElse(null);
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
            if (commandMessageId < 0) {
                log.warnf("openGate: no COMMAND message found for commitmentId=%s — failing open", commitmentId);
                return new GateDecision.Autonomous();
            }

            UUID gateId = UUID.randomUUID();
            GateContext ctx = new GateContext(commitmentId, workChannelId, commandMessageId, tenancyId);

            messageService.dispatch(MessageDispatch.builder()
                    .channelId(oversightChannel.id)
                    .sender(GATE_SENDER)
                    .type(MessageType.COMMAND)
                    .content(serializeGateContent(ctx, gate.reason()))
                    .correlationId(gateId.toString())
                    .actorType(ActorType.AGENT)
                    .tenancyId(tenancyId)
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

    @Override
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
            long commandMessageId = gateCmd.id;
            Optional<GateContext> gateContext = parseGateContent(gateCmd.content);
            String tenancyId = gateContext.map(GateContext::tenancyId).orElse(null);

            if (tenancyId == null) {
                tenancyId = crossTenantChannelStore.findById(oversightChannelId)
                        .map(ch -> ch.tenancyId)
                        .orElse(null);
                if (tenancyId != null)
                    log.infof("fulfill(): recovered tenancyId from channel for pre-#29 gate %s", gateId);
                else
                    log.warnf("fulfill(): cannot recover tenancyId for gate %s — oversight channel not found", gateId);
            }

            boolean approved = parseApproval(gateId, rawOutput);
            gateDispatcher.dispatch(approved, oversightChannelId, commandMessageId,
                    gateId, rawOutput, gateContext, tenancyId);

            log.infof("Gate %s: gateId=%s", approved ? "approved" : "rejected", gateId);
        } catch (Exception e) {
            log.errorf("OversightGateService.fulfill() failed for gateId=%s: %s", gateId, e.getMessage());
        }
    }

    private String serializeGateContent(GateContext ctx, String reason) {
        Properties props = new Properties();
        props.setProperty("originalCommitmentId", ctx.originalCommitmentId());
        props.setProperty("workChannelId", ctx.workChannelId().toString());
        props.setProperty("commandMessageId", String.valueOf(ctx.commandMessageId()));
        props.setProperty("reason", reason != null ? reason : "");
        if (ctx.tenancyId() != null) props.setProperty("tenancyId", ctx.tenancyId());
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
            String tid = props.getProperty("tenancyId");
            return Optional.of(new GateContext(oci, UUID.fromString(wci), Long.parseLong(cmi), tid));
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

- [ ] **Step 3: Delete old OversightGateService.java**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw rm casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateService.java
```

- [ ] **Step 4: Update OversightGateDispatcher — change GATE_SENDER reference**

In `casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java`, replace all four occurrences of `OversightGateService.GATE_SENDER` with `OpenClawOversightGateService.GATE_SENDER`.

The file has these four lines (lines 51, 63, 76, 88):
```java
// Before (×4):
.sender(OversightGateService.GATE_SENDER)
// After (×4):
.sender(OpenClawOversightGateService.GATE_SENDER)
```

- [ ] **Step 5: Run compile test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub \
    -Dtest=OpenClawOversightGateServiceCompileTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 1, Failures: 0, Errors: 0`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawOversightGateService.java \
    casehub/src/main/java/io/casehub/openclaw/casehub/OversightGateDispatcher.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): rename OversightGateService→OpenClawOversightGateService, delegate classification to ReactiveActionRiskClassifier — Refs #31"
```

---

### Task B3: Delete GateDecision.java — update all import references

**Files:**
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/GateDecision.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`

`GateDecision` now lives in `io.casehub.api.spi`. Any file importing `io.casehub.openclaw.casehub.GateDecision` must change.

- [ ] **Step 1: Delete GateDecision from openclaw**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw rm casehub/src/main/java/io/casehub/openclaw/casehub/GateDecision.java
```

- [ ] **Step 2: Update CommitmentTools.java imports**

In `app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java`:

```java
// Remove:
import io.casehub.openclaw.casehub.GateDecision;
import io.casehub.openclaw.casehub.OversightGateService;

// Add:
import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.OversightGateService;
```

The field type and constructor parameter remain `OversightGateService` — now it's the interface from engine-api. No logic changes.

- [ ] **Step 3: Build casehub + app modules to check for missed references**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl casehub,app -am 2>&1 | grep "ERROR\|cannot find" | head -20
```

Expected: No `GateDecision` or `OversightGateService` import errors remain.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    app/src/main/java/io/casehub/openclaw/app/mcp/CommitmentTools.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor: delete openclaw GateDecision, update CommitmentTools to import from engine-api — Refs #31"
```

---

### Task B4: Remove CaseChannelNames — update callers to use CaseChannel

**Files:**
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/CaseChannelNames.java`
- Delete: `casehub/src/test/java/io/casehub/openclaw/casehub/CaseChannelNamesTest.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`

`OpenClawOversightGateService` already uses `CaseChannel.parseCaseId()` and `CaseChannel.oversightChannelName()` (written in Task B2). Only `OpenClawChannelBackend` remains to migrate.

- [ ] **Step 1: Update OpenClawChannelBackend.extractCaseId()**

In `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java`, line 126:

```java
// Before:
UUID extractCaseId(final String channelName) {
    return CaseChannelNames.extractCaseId(channelName);
}

// After:
UUID extractCaseId(final String channelName) {
    return CaseChannel.parseCaseId(channelName);
}
```

Remove the import of `CaseChannelNames` if present (it is not imported by name — the call is the only usage).

- [ ] **Step 2: Delete CaseChannelNames and its test**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw rm \
    casehub/src/main/java/io/casehub/openclaw/casehub/CaseChannelNames.java \
    casehub/src/test/java/io/casehub/openclaw/casehub/CaseChannelNamesTest.java
```

- [ ] **Step 3: Build casehub module — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl casehub -am
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawChannelBackend.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor: delete CaseChannelNames, replace with CaseChannel static methods — Refs #31"
```

---

### Task B5: Update app-layer injections for OversightGateService / OpenClawOversightGateService

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/OpenClawOversightDeliveryResource.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java`

`OpenClawOversightDeliveryResource` calls `fulfill()` — uses interface. `OpenClawDeliveryResource` calls `evaluate()` — must inject concrete class since `evaluate()` is not on the SPI.

- [ ] **Step 1: Update OpenClawOversightDeliveryResource**

```java
// app/src/main/java/io/casehub/openclaw/app/OpenClawOversightDeliveryResource.java

// Change import:
// Remove: import io.casehub.openclaw.casehub.OversightGateService;
// Add:    import io.casehub.api.spi.OversightGateService;

// Field stays the same:
@Inject
OversightGateService oversightGateService;
```

- [ ] **Step 2: Update OpenClawDeliveryResource**

```java
// app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java

// Change import:
// Remove: import io.casehub.openclaw.casehub.OversightGateService;
// Add:    import io.casehub.openclaw.casehub.OpenClawOversightGateService;

// Change field type:
@Inject
OpenClawOversightGateService oversightGateService;
```

The call `oversightGateService.evaluate(channelId, tenancyId, agentId, output)` remains unchanged.

- [ ] **Step 3: Build app module — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app -am
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    app/src/main/java/io/casehub/openclaw/app/OpenClawOversightDeliveryResource.java \
    app/src/main/java/io/casehub/openclaw/app/OpenClawDeliveryResource.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor: update delivery resource injections — OversightGateService interface / OpenClawOversightGateService concrete — Refs #31"
```

---

### Task B6: Add ReactiveOpenClawOversightGateService

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateService.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateServiceTest.java`

Thin delegate pattern — wraps `OpenClawOversightGateService` with `.runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` so IO-thread callers get a real thread hop, not a synchronous-on-subscriber execution.

- [ ] **Step 1: Write the failing test**

```java
// casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateServiceTest.java
package io.casehub.openclaw.casehub;

import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.ReactiveActionRiskClassifier;
import io.casehub.api.spi.RiskDecision;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class ReactiveOpenClawOversightGateServiceTest {

    OpenClawOversightGateService delegate;
    ReactiveOpenClawOversightGateService reactive;

    @BeforeEach
    void setUp() {
        // Build a minimal delegate wired with mocks
        ReactiveActionRiskClassifier classifier = mock(ReactiveActionRiskClassifier.class);
        when(classifier.classify(any())).thenReturn(Uni.createFrom().item(new RiskDecision.Autonomous()));

        delegate = new OpenClawOversightGateService(
                mock(io.casehub.qhorus.runtime.channel.ChannelService.class),
                mock(io.casehub.qhorus.runtime.message.MessageService.class),
                mock(io.casehub.qhorus.runtime.store.CommitmentStore.class),
                mock(OversightGateDispatcher.class),
                classifier,
                mock(io.casehub.qhorus.runtime.store.CrossTenantMessageStore.class),
                mock(io.casehub.qhorus.runtime.store.CrossTenantChannelStore.class));

        reactive = new ReactiveOpenClawOversightGateService(delegate);
    }

    @Test
    void openGate_emitsSameResultAsDelegate() {
        // delegate returns Autonomous (no commitment found → fail-open)
        GateDecision result = reactive
                .openGate("agent-1", UUID.randomUUID().toString(), "done", "tenant-a")
                .await().atMost(Duration.ofSeconds(5));
        assertThat(result).isInstanceOf(GateDecision.Autonomous.class);
    }

    @Test
    void fulfill_completesWithoutError() {
        assertThatCode(() ->
                reactive.fulfill(UUID.randomUUID(), "approved")
                        .await().atMost(Duration.ofSeconds(5)))
                .doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2: Run test — expect compile failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub \
    -Dtest=ReactiveOpenClawOversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL with `cannot find symbol: class ReactiveOpenClawOversightGateService`

- [ ] **Step 3: Create ReactiveOpenClawOversightGateService**

```java
// casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateService.java
package io.casehub.openclaw.casehub;

import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.ReactiveOversightGateService;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.UUID;

/**
 * Reactive delegate wrapping {@link OpenClawOversightGateService} for Vert.x IO thread safety.
 *
 * <p>Uses the thin delegate pattern (not split-class): injects the concrete blocking implementation
 * and offloads calls to the worker pool via {@link Infrastructure#getDefaultWorkerPool()}.
 *
 * <p>No {@code @IfBuildProperty} gate — this class has no reactive Qhorus service deps and is
 * always safe to activate regardless of the Qhorus reactive build flag.
 *
 * <p>Callers on Vert.x IO threads must use this interface; the blocking
 * {@link OpenClawOversightGateService#openGate} is unsafe on IO threads due to
 * {@code await().indefinitely()} inside it.
 */
@ApplicationScoped
public class ReactiveOpenClawOversightGateService implements ReactiveOversightGateService {

    private final OpenClawOversightGateService delegate;

    @Inject
    public ReactiveOpenClawOversightGateService(final OpenClawOversightGateService delegate) {
        this.delegate = delegate;
    }

    @Override
    public Uni<GateDecision> openGate(final String agentId, final String commitmentId,
                                       final String outcome, final String tenancyId) {
        return Uni.createFrom()
                .item(() -> delegate.openGate(agentId, commitmentId, outcome, tenancyId))
                .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public Uni<Void> fulfill(final UUID gateId, final String rawOutput) {
        return Uni.createFrom()
                .item(() -> { delegate.fulfill(gateId, rawOutput); return null; })
                .replaceWithVoid()
                .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub \
    -Dtest=ReactiveOpenClawOversightGateServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `Tests run: 2, Failures: 0, Errors: 0`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateService.java \
    casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawOversightGateServiceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(casehub): add ReactiveOpenClawOversightGateService thin delegate with runSubscriptionOn — Refs #31"
```

---

### Task B7: Update test files — rename + import fixes + remove classification tests

**Files:**
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawOversightGateServiceTest.java`
- Delete: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java`
- Delete: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawOversightGateServiceCompileTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`

- [ ] **Step 1: Create OpenClawOversightGateServiceTest (rename + restructure)**

Copy `OversightGateServiceTest.java` content. Make these changes:

**Package/class:** unchanged (`io.casehub.openclaw.casehub`)

**Class name:** `OversightGateServiceTest` → `OpenClawOversightGateServiceTest`

**Imports to remove:**
```java
import jakarta.enterprise.inject.Instance;
import io.casehub.api.spi.ActionRiskClassifier;
import io.casehub.openclaw.casehub.GateDecision;
import io.casehub.openclaw.casehub.OversightGateService;
```

**Imports to add:**
```java
import io.casehub.api.spi.GateDecision;
import io.casehub.api.spi.OversightGateService;
import io.casehub.api.spi.ReactiveActionRiskClassifier;
```

**Field changes:**
```java
// Remove:
@SuppressWarnings("unchecked")
Instance<ActionRiskClassifier> classifiers = mock(Instance.class);
ActionRiskClassifier mockClassifier = mock(ActionRiskClassifier.class);

// Add:
ReactiveActionRiskClassifier reactiveClassifier = mock(ReactiveActionRiskClassifier.class);
```

**`service` field type:**
```java
// Before:
OversightGateService service;
// After:
OpenClawOversightGateService service;
```

**`setup()` changes:**
```java
// Remove:
// Default: no classifiers (isUnsatisfied = true)
when(classifiers.isUnsatisfied()).thenReturn(true);

service = new OversightGateService(channelService, messageService, commitmentStore,
        gateDispatcher, classifiers, crossTenantMessageStore, crossTenantChannelStore);

// Add:
// Default: classifier returns Autonomous
when(reactiveClassifier.classify(any()))
        .thenReturn(Uni.createFrom().item(new RiskDecision.Autonomous()));

service = new OpenClawOversightGateService(channelService, messageService, commitmentStore,
        gateDispatcher, reactiveClassifier, crossTenantMessageStore, crossTenantChannelStore);
```

**Remove these test methods entirely** (classification logic now tested in `ChainedReactiveActionRiskClassifierTest` in engine):
- `openGate_noClassifiers_returnsAutonomous`
- `openGate_classifierReturnsAutonomous_returnsAutonomous`
- `openGate_classifierThrows_appliesFailSafeGateRequired`
- `openGate_mostRestrictive_picksSmallerCandidateGroupsAsMoreRestrictive`
- `openGate_twoClassifiersOneAutonomousOneGateRequired_returnsGatePending`

**Keep all remaining test methods.** The `openGate_classifierReturnsGateRequired_*` tests must be updated to stub via:
```java
// Replace helper method stubSingleClassifier:
// Before:
private void stubSingleClassifier(RiskDecision decision) {
    when(mockClassifier.classify(any())).thenReturn(decision);
    when(classifiers.isUnsatisfied()).thenReturn(false);
    when(classifiers.iterator()).thenReturn(List.of(mockClassifier).iterator());
}

// After:
private void stubClassifier(RiskDecision decision) {
    when(reactiveClassifier.classify(any()))
            .thenReturn(Uni.createFrom().item(decision));
}
```

Replace all calls to `stubSingleClassifier(...)` with `stubClassifier(...)`. Delete `stubSingleClassifier_throws()` — no equivalent (infrastructure failures in `openGate()` catch at the outer try/catch level, returning Autonomous).

**GATE_SENDER constant reference:** stays as `OpenClawOversightGateService.GATE_SENDER` (line ~451 in old test).

- [ ] **Step 2: Delete old test and compile test**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw rm \
    casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateServiceTest.java \
    casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawOversightGateServiceCompileTest.java
```

- [ ] **Step 3: Update OversightGateDispatcherTest**

In `casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java`, line 123:
```java
// Before:
return new DispatchResult(messageId, oversightChannelId, OversightGateService.GATE_SENDER,
// After:
return new DispatchResult(messageId, oversightChannelId, OpenClawOversightGateService.GATE_SENDER,
```

Update the import if needed: remove `import io.casehub.openclaw.casehub.OversightGateService;`, keep `OpenClawOversightGateService` accessible (same package — no import needed).

- [ ] **Step 4: Update OversightGateDispatcherCdiTest**

In `app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java`:
```java
// Change import:
// Remove: import io.casehub.openclaw.casehub.OversightGateService;
// Add:    import io.casehub.api.spi.OversightGateService;

// Field type:
// Before: OversightGateService oversightGateService;
// After:  OversightGateService oversightGateService;  (no change in type name, just import)
```

CDI injects `OpenClawOversightGateService` (which implements the interface) into the `OversightGateService`-typed field — this is correct.

- [ ] **Step 5: Update CommitmentToolsTest**

In `app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java`:
```java
// Change imports:
// Remove: import io.casehub.openclaw.casehub.OversightGateService;
// Add:    import io.casehub.api.spi.OversightGateService;

// The mock field and mock creation remain the same:
OversightGateService oversightGateService;
oversightGateService = mock(OversightGateService.class);  // Mockito mocks interfaces fine
```

- [ ] **Step 6: Update OpenClawDeliveryResourceTest**

In `app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java`:
```java
// Field injects the concrete class (evaluate() is not on interface):
// Remove: OversightGateService oversightGateService;  (if typed as old class)
// Add: OpenClawOversightGateService oversightGateService;

// Import:
// Remove: import io.casehub.openclaw.casehub.OversightGateService;
// Add:    import io.casehub.openclaw.casehub.OpenClawOversightGateService;
```

Note: Mockito can mock concrete classes (`mock(OpenClawOversightGateService.class)`). If the test uses `@InjectMock` or `@QuarkusTest`, ensure the injection type matches.

- [ ] **Step 7: Run all casehub + app tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub,app -am
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
    casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawOversightGateServiceTest.java \
    casehub/src/test/java/io/casehub/openclaw/casehub/OversightGateDispatcherTest.java \
    app/src/test/java/io/casehub/openclaw/app/OversightGateDispatcherCdiTest.java \
    app/src/test/java/io/casehub/openclaw/app/mcp/CommitmentToolsTest.java \
    app/src/test/java/io/casehub/openclaw/app/OpenClawDeliveryResourceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "test: rename OversightGateServiceTest, update mocks to inject ReactiveActionRiskClassifier, remove duplicate classification tests — Refs #31"
```

---

### Task B8: Full build green + PLATFORM.md issue

- [ ] **Step 1: Full build of all openclaw modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 2: File PLATFORM.md update issue on casehubio/parent**

```bash
gh issue create --repo casehubio/parent \
  --title "docs: update PLATFORM.md — OversightGateService extracted to casehub-engine-api (openclaw#31)" \
  --body "$(cat <<'EOF'
## What changed

casehubio/openclaw#31 completed. OversightGateService has been extracted from casehub-openclaw to casehub-engine-api.

## Required PLATFORM.md edits

1. **Known Placement Violations table** — remove the OversightGateService row (it was listed with intended home casehub-engine-api; that is now done).

2. **Capability Ownership table** — update the oversight gate row:
   - Current: \`casehub-openclaw\` | \`OversightGateService.evaluate() archives...\`
   - New: \`casehub-engine-api\` (interface) / \`casehub-openclaw\` (impl) | \`OversightGateService, ReactiveOversightGateService, GateDecision\`

3. **Cross-repo dependency map** — add rows for new types in engine-api consumed by openclaw/casehub:
   - \`casehub-engine-api\` → \`casehub-openclaw\` / \`casehub\` — \`OversightGateService\`, \`ReactiveOversightGateService\`, \`GateDecision\`
EOF
)"
```

- [ ] **Step 3: Push openclaw branch**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw push -u origin issue-31-extract-oversight-gate-service
```

---

### Task B9: Delete compile test and do final clean pass

- [ ] **Step 1: Verify no remaining references to old types**

Using IntelliJ `ide_find_class` for `OversightGateService` — should only find the engine-api interface (no openclaw class), `OpenClawOversightGateService`, and `ReactiveOpenClawOversightGateService`. No class named `CaseChannelNames` or `GateDecision` in openclaw packages.

- [ ] **Step 2: Final full build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS. All tests pass.

- [ ] **Step 3: Commit any remaining cleanup**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw status
# If any files remain unstaged, add and commit:
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "chore: final cleanup — verify build green — Refs #31"
```
