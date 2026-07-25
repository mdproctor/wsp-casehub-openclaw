# Hardening, Config, and Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden DirectCallBridge with self-evicting futures and correct module placement (#50), introduce AgentProviderConfigSource SPI for pluggable agent config (#36), and wire OIDC auth on the MCP endpoint (#43).

**Architecture:** Three independent changes on one branch, executed sequentially. §1 (DirectCallBridge) modifies `casehub/` and moves files to `app/`. §2 (config SPI) adds an interface in `casehub/`, a `@DefaultBean` impl in `app/`, and migrates all consumers. §3 (MCP auth) is config-only plus security tests.

**Tech Stack:** Java 21 / Quarkus 3.32.2 / quarkus-mcp-server-http 1.11.1 / JUnit 5 / AssertJ / Mockito

**Spec:** `docs/specs/2026-06-26-hardening-config-auth-design.md`

## Global Constraints

- Java 21 source (Java 26 JVM): `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn --batch-mode install` (system `mvn`, not `./mvnw`)
- Test scoping: `mvn test -pl <module> -am -Dsurefire.failIfNoSpecifiedTests=false`
- Every commit references an issue: `Refs #50`, `Refs #36`, or `Refs #43`
- IntelliJ MCP for all renames/moves (not bash)
- Protocols: `delivery-webhook-cross-tenant-reads`, `mcp-tool-no-instance-cache`, `auth-retrofit-readiness`

---

### Task 1: DirectCallBridge — self-evicting futures (#50)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallBridge.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallBridgeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `DirectCallBridge.submit(String correlationId, Duration timeout)` — returns `CompletableFuture<String>` with self-evicting cleanup via `whenComplete`

- [ ] **Step 1: Write failing test — submit with timeout self-evicts on timeout**

Add to `DirectCallBridgeTest.java`:

```java
import java.time.Duration;
import java.util.concurrent.TimeoutException;

@Test
void submit_withTimeout_selfEvictsOnTimeout() throws Exception {
    DirectCallBridge bridge = new DirectCallBridge();
    CompletableFuture<String> future = bridge.submit("corr-1", Duration.ofMillis(50));
    Thread.sleep(200);
    assertThat(future.isCompletedExceptionally()).isTrue();
    assertThatThrownBy(future::get)
            .hasCauseInstanceOf(TimeoutException.class);
    // Verify map cleanup: a new submit should NOT return the timed-out future
    CompletableFuture<String> fresh = bridge.submit("corr-1", Duration.ofSeconds(10));
    assertThat(fresh).isNotSameAs(future);
    assertThat(fresh.isDone()).isFalse();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=DirectCallBridgeTest#submit_withTimeout_selfEvictsOnTimeout -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — `submit(String, Duration)` does not exist.

- [ ] **Step 3: Implement self-evicting submit and update complete/cancel**

Replace the contents of `DirectCallBridge.java`:

```java
package io.casehub.openclaw.casehub;

import jakarta.enterprise.context.ApplicationScoped;

import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

@ApplicationScoped
public class DirectCallBridge {

    private static final Logger log = Logger.getLogger(DirectCallBridge.class);

    private final ConcurrentHashMap<String, CompletableFuture<String>> futures =
            new ConcurrentHashMap<>();

    public CompletableFuture<String> submit(String correlationId, Duration timeout) {
        CompletableFuture<String> future = new CompletableFuture<>();
        CompletableFuture<String> existing = futures.putIfAbsent(correlationId, future);
        if (existing != null) {
            log.warnf("Duplicate correlationId=%s — returning existing future", correlationId);
            return existing;
        }
        future.orTimeout(timeout.toSeconds(), TimeUnit.SECONDS);
        future.whenComplete((result, error) -> futures.remove(correlationId));
        return future;
    }

    public void complete(String correlationId, String responseText) {
        CompletableFuture<String> future = futures.get(correlationId);
        if (future != null) {
            future.complete(responseText);
        }
    }

    public void cancel(String correlationId) {
        CompletableFuture<String> future = futures.get(correlationId);
        if (future != null) {
            future.cancel(true);
        }
    }
}
```

- [ ] **Step 4: Update existing tests to pass Duration to submit**

Update every `bridge.submit("corr-1")` call in `DirectCallBridgeTest.java` to `bridge.submit("corr-1", Duration.ofSeconds(30))`. The import `java.time.Duration` is already added in Step 1.

Tests to update: `submit_createsAndReturnsFuture`, `complete_resolvesFuture`, `cancel_cancelsFuture`, `complete_afterCancel_isNoOp`, `submit_duplicateCorrelationId_returnsExistingFuture`. Tests `complete_unknownCorrelationId_isNoOp` and `cancel_unknownCorrelationId_isNoOp` do not call `submit()`.

- [ ] **Step 5: Run all DirectCallBridge tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=DirectCallBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 6: Update OpenClawAgentProvider to pass timeout**

In `OpenClawAgentProvider.invoke()`, change line 36 from:

```java
var future = bridge.submit(correlationId);
```

to:

```java
Duration effectiveTimeout = config.timeout() != null
        ? config.timeout() : Duration.ofSeconds(120);
var future = bridge.submit(correlationId, effectiveTimeout);
```

Add `import java.time.Duration;` to the imports.

- [ ] **Step 7: Update OpenClawAgentProviderTest to pass Duration to submit**

Every test that constructs a `DirectCallBridge` and calls `bridge.submit(correlationId)` directly (the `doAnswer` mock in `invoke_emitsTextDeltaOnFutureCompletion`) must be updated. In that test, the `doAnswer` lambda calls `bridge.complete(corrId, ...)` which does not call `submit()` — so no change needed there. However, any test that calls `bridge.submit()` directly must add `Duration.ofSeconds(30)` as the second argument.

Check: in `OpenClawAgentProviderTest`, `bridge.submit()` is never called directly — all tests go through `provider.invoke(config)` which now calls `bridge.submit(correlationId, effectiveTimeout)` internally. No test changes needed.

- [ ] **Step 8: Run OpenClawAgentProvider tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=OpenClawAgentProviderTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallBridge.java casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentProvider.java casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallBridgeTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#50): self-evicting futures in DirectCallBridge — orTimeout + whenComplete cleanup

Refs #50"
```

---

### Task 2: Move DirectCallDeliveryResource to app/ and remove dead field (#50)

**Files:**
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallDeliveryResource.java`
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallDeliveryPayload.java`
- Delete: `casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallDeliveryResourceTest.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/DirectCallDeliveryResource.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/DirectCallDeliveryPayload.java`
- Create: `app/src/test/java/io/casehub/openclaw/app/DirectCallDeliveryResourceTest.java`
- Modify: `app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java`

**Interfaces:**
- Consumes: `DirectCallBridge.complete(String, String)` from Task 1
- Produces: `POST /openclaw/direct-call/{correlationId}` endpoint (unchanged URL)

- [ ] **Step 1: Create DirectCallDeliveryPayload in app/ with agentId removed**

Create `app/src/main/java/io/casehub/openclaw/app/DirectCallDeliveryPayload.java`:

```java
package io.casehub.openclaw.app;

public record DirectCallDeliveryPayload(String output) {}
```

- [ ] **Step 2: Create DirectCallDeliveryResource in app/**

Create `app/src/main/java/io/casehub/openclaw/app/DirectCallDeliveryResource.java`:

```java
package io.casehub.openclaw.app;

import jakarta.annotation.security.PermitAll;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.openclaw.casehub.DirectCallBridge;
import io.smallrye.common.annotation.Blocking;

import org.jboss.logging.Logger;

@PermitAll
@Blocking
@ApplicationScoped
@Path("/openclaw/direct-call")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class DirectCallDeliveryResource {

    private static final Logger log = Logger.getLogger(DirectCallDeliveryResource.class);

    private final DirectCallBridge bridge;

    @Inject
    public DirectCallDeliveryResource(DirectCallBridge bridge) {
        this.bridge = bridge;
    }

    @POST
    @Path("/{correlationId}")
    public Response deliver(@PathParam("correlationId") String correlationId,
                             DirectCallDeliveryPayload payload) {
        try {
            String output = payload != null && payload.output() != null
                    ? payload.output() : "";
            bridge.complete(correlationId, output);
        } catch (Exception e) {
            log.errorf("direct-call delivery failed for correlationId=%s: %s",
                    correlationId, e.getMessage());
        }
        return Response.ok().build();
    }
}
```

- [ ] **Step 3: Create DirectCallDeliveryResourceTest in app/**

Create `app/src/test/java/io/casehub/openclaw/app/DirectCallDeliveryResourceTest.java`:

```java
package io.casehub.openclaw.app;

import io.casehub.openclaw.casehub.DirectCallBridge;
import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;

class DirectCallDeliveryResourceTest {

    @Test
    void deliver_completesTheBridge() {
        DirectCallBridge bridge = new DirectCallBridge();
        DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
        CompletableFuture<String> future = bridge.submit("corr-1", Duration.ofSeconds(30));

        Response response = resource.deliver("corr-1",
                new DirectCallDeliveryPayload("result text"));

        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(future.isDone()).isTrue();
        assertThat(future.getNow(null)).isEqualTo("result text");
    }

    @Test
    void deliver_unknownCorrelationId_returns200() {
        DirectCallBridge bridge = new DirectCallBridge();
        DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);

        Response response = resource.deliver("unknown",
                new DirectCallDeliveryPayload("text"));

        assertThat(response.getStatus()).isEqualTo(200);
    }

    @Test
    void deliver_nullPayload_returns200WithEmptyOutput() {
        DirectCallBridge bridge = new DirectCallBridge();
        DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
        CompletableFuture<String> future = bridge.submit("corr-1", Duration.ofSeconds(30));

        Response response = resource.deliver("corr-1", null);

        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(future.getNow(null)).isEqualTo("");
    }

    @Test
    void deliver_nullOutput_returns200WithEmptyString() {
        DirectCallBridge bridge = new DirectCallBridge();
        DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
        CompletableFuture<String> future = bridge.submit("corr-1", Duration.ofSeconds(30));

        Response response = resource.deliver("corr-1",
                new DirectCallDeliveryPayload(null));

        assertThat(response.getStatus()).isEqualTo(200);
        assertThat(future.getNow(null)).isEqualTo("");
    }

    @Test
    void deliver_exceptionInBridge_stillReturns200() {
        DirectCallBridge bridge = new DirectCallBridge();
        DirectCallDeliveryResource resource = new DirectCallDeliveryResource(bridge);
        CompletableFuture<String> future = bridge.submit("corr-1", Duration.ofSeconds(30));
        bridge.cancel("corr-1");

        Response response = resource.deliver("corr-1",
                new DirectCallDeliveryPayload("late response"));

        assertThat(response.getStatus()).isEqualTo(200);
    }
}
```

- [ ] **Step 4: Add security test for direct-call endpoint**

Add to `OpenClawRestSecurityTest.java`:

```java
@Test
void permitAll_directCallDelivery_noAuthRequired() {
    given().contentType(JSON).body("{\"output\":\"test\"}")
        .when().post("/openclaw/direct-call/" + UUID.randomUUID())
        .then().statusCode(not(in(List.of(401, 403))));
}
```

- [ ] **Step 5: Delete old files from casehub/**

Delete these files:
- `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallDeliveryResource.java`
- `casehub/src/main/java/io/casehub/openclaw/casehub/DirectCallDeliveryPayload.java`
- `casehub/src/test/java/io/casehub/openclaw/casehub/DirectCallDeliveryResourceTest.java`

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=DirectCallDeliveryResourceTest,OpenClawRestSecurityTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add -A
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "refactor(#50): move DirectCallDeliveryResource to app/; remove unused agentId from payload

Refs #50"
```

---

### Task 3: AgentProviderConfigSource SPI and @DefaultBean implementation (#36)

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/AgentProviderConfigSource.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSource.java`
- Create: `app/src/test/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSourceTest.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/example/ExampleController.java`

**Interfaces:**
- Consumes: `OpenClawCasehubConfig` (existing SmallRye config mapping — backing store for @DefaultBean)
- Produces: `AgentProviderConfigSource.allAgents()` — `Map<String, AgentConfig>` where `AgentConfig(String sessionKey, List<String> capabilities)`

- [ ] **Step 1: Create the SPI interface**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/AgentProviderConfigSource.java`:

```java
package io.casehub.openclaw.casehub;

import java.util.List;
import java.util.Map;

public interface AgentProviderConfigSource {
    record AgentConfig(String sessionKey, List<String> capabilities) {}
    Map<String, AgentConfig> allAgents();
}
```

- [ ] **Step 2: Write failing test for ConfigFileAgentProviderConfigSource**

Create `app/src/test/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSourceTest.java`:

```java
package io.casehub.openclaw.app;

import io.casehub.openclaw.casehub.AgentProviderConfigSource;
import io.casehub.openclaw.casehub.AgentProviderConfigSource.AgentConfig;
import io.casehub.openclaw.casehub.OpenClawCasehubConfig;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

class ConfigFileAgentProviderConfigSourceTest {

    @Test
    void allAgents_convertsConfigToAgentConfigMap() {
        OpenClawCasehubConfig config = buildConfig(Map.of(
                "finance-agent", entry(List.of("finance", "banking"), "fin-key"),
                "code-agent", entry(List.of("code-review"), "cr-key")));

        AgentProviderConfigSource source = new ConfigFileAgentProviderConfigSource(config);
        Map<String, AgentConfig> agents = source.allAgents();

        assertThat(agents).hasSize(2);
        assertThat(agents.get("finance-agent").sessionKey()).isEqualTo("fin-key");
        assertThat(agents.get("finance-agent").capabilities()).containsExactlyInAnyOrder("finance", "banking");
        assertThat(agents.get("code-agent").sessionKey()).isEqualTo("cr-key");
        assertThat(agents.get("code-agent").capabilities()).containsExactly("code-review");
    }

    @Test
    void allAgents_emptyConfig_returnsEmptyMap() {
        OpenClawCasehubConfig config = buildConfig(Map.of());
        AgentProviderConfigSource source = new ConfigFileAgentProviderConfigSource(config);
        assertThat(source.allAgents()).isEmpty();
    }

    private OpenClawCasehubConfig buildConfig(Map<String, OpenClawCasehubConfig.AgentEntry> agents) {
        return new OpenClawCasehubConfig() {
            @Override public Map<String, AgentEntry> agents() { return agents; }
            @Override public Oversight oversight() { return Optional::empty; }
        };
    }

    private OpenClawCasehubConfig.AgentEntry entry(List<String> caps, String sk) {
        return new OpenClawCasehubConfig.AgentEntry() {
            @Override public List<String> capabilities() { return caps; }
            @Override public String sessionKey() { return sk; }
        };
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ConfigFileAgentProviderConfigSourceTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — `ConfigFileAgentProviderConfigSource` does not exist.

- [ ] **Step 4: Implement ConfigFileAgentProviderConfigSource**

Create `app/src/main/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSource.java`:

```java
package io.casehub.openclaw.app;

import io.casehub.openclaw.casehub.AgentProviderConfigSource;
import io.casehub.openclaw.casehub.OpenClawCasehubConfig;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Map;
import java.util.stream.Collectors;

@DefaultBean
@ApplicationScoped
public class ConfigFileAgentProviderConfigSource implements AgentProviderConfigSource {

    private final OpenClawCasehubConfig config;

    @Inject
    public ConfigFileAgentProviderConfigSource(OpenClawCasehubConfig config) {
        this.config = config;
    }

    @Override
    public Map<String, AgentConfig> allAgents() {
        return config.agents().entrySet().stream()
                .collect(Collectors.toMap(
                        Map.Entry::getKey,
                        e -> new AgentConfig(
                                e.getValue().sessionKey(),
                                e.getValue().capabilities())));
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ConfigFileAgentProviderConfigSourceTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS

- [ ] **Step 6: Migrate OpenClawWorkerProvisioner to use AgentProviderConfigSource**

Replace `OpenClawCasehubConfig config` with `AgentProviderConfigSource configSource` in the constructor and all usages. Full replacement of `OpenClawWorkerProvisioner.java`:

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

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.spi.ProvisionResult;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.WorkerProvisioner;
import io.casehub.openclaw.context.ChannelContextWindowService;
import io.casehub.platform.api.identity.CurrentPrincipal;

@ApplicationScoped
public class OpenClawWorkerProvisioner implements WorkerProvisioner {

    private static final Logger log = Logger.getLogger(OpenClawWorkerProvisioner.class);

    private final ChannelContextWindowService service;
    private final OpenClawAgentRegistry registry;
    private final AgentProviderConfigSource configSource;
    private final CurrentPrincipal currentPrincipal;

    @Inject
    public OpenClawWorkerProvisioner(ChannelContextWindowService service,
                                      OpenClawAgentRegistry registry,
                                      AgentProviderConfigSource configSource,
                                      CurrentPrincipal currentPrincipal) {
        this.service = service;
        this.registry = registry;
        this.configSource = configSource;
        this.currentPrincipal = currentPrincipal;
    }

    @Override
    public ProvisionResult provision(Set<String> capabilities, ProvisionContext context) {
        String agentId = resolveAgentId(capabilities);
        String sessionKey = configSource.allAgents().get(agentId).sessionKey();
        UUID caseId = context.caseId();
        String tenancyId = currentPrincipal.tenancyId();

        registry.register(agentId, tenancyId, caseId, sessionKey);
        service.bindAgent(agentId, caseId);

        log.infof("Provisioned OpenClaw agent: agentId=%s caseId=%s tenancyId=%s capabilities=%s",
                agentId, caseId, tenancyId, capabilities);

        return ProvisionResult.empty();
    }

    @Override
    public void terminate(String workerId, String tenancyId) {
        registry.deregister(workerId);
        service.unbindAgent(workerId);
        log.infof("Terminated OpenClaw agent: agentId=%s tenancyId=%s", workerId, tenancyId);
    }

    @Override
    public Set<String> getCapabilities() {
        return configSource.allAgents().values().stream()
                .flatMap(e -> e.capabilities().stream())
                .collect(Collectors.toSet());
    }

    private String resolveAgentId(Set<String> requested) {
        List<String> candidates = configSource.allAgents().entrySet().stream()
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

- [ ] **Step 7: Update OpenClawWorkerProvisionerTest**

Replace `OpenClawCasehubConfig config` with `AgentProviderConfigSource configSource` in the test. Replace `buildConfig()` and `entry()` helpers. Full replacement of setup and helpers:

```java
// Replace fields:
AgentProviderConfigSource configSource;

// Replace setup():
@BeforeEach
void setup() {
    mockService = mock(ChannelContextWindowService.class);
    registry = new OpenClawAgentRegistry();
    configSource = buildConfigSource(Map.of(
            "finance-agent", new AgentProviderConfigSource.AgentConfig("finance-agent", List.of("finance", "banking")),
            "code-review-agent", new AgentProviderConfigSource.AgentConfig("cr-agent-main", List.of("code-review"))));
    mockPrincipal = mock(CurrentPrincipal.class);
    when(mockPrincipal.tenancyId()).thenReturn("test-tenant");
    provisioner = new OpenClawWorkerProvisioner(mockService, registry, configSource, mockPrincipal);
}

// Replace helpers:
private AgentProviderConfigSource buildConfigSource(Map<String, AgentProviderConfigSource.AgentConfig> agents) {
    return () -> agents;
}
```

Remove `buildConfig()` and `entry()` methods. Remove `OpenClawCasehubConfig` import, add `AgentProviderConfigSource` import.

- [ ] **Step 8: Run OpenClawWorkerProvisionerTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=OpenClawWorkerProvisionerTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 9: Migrate ReactiveOpenClawWorkerProvisioner**

Same mechanical swap: replace `OpenClawCasehubConfig config` with `AgentProviderConfigSource configSource` in constructor and all usages. The changes mirror Step 6 exactly — `config.agents()` → `configSource.allAgents()`, `e.getValue().capabilities()` → `e.getValue().capabilities()` (same), `config.agents().get(agentId).sessionKey()` → `configSource.allAgents().get(agentId).sessionKey()`.

- [ ] **Step 10: Update ReactiveOpenClawWorkerProvisionerTest**

Same mechanical swap as Step 7: replace config/helper pattern with `AgentProviderConfigSource`-based setup.

- [ ] **Step 11: Run ReactiveOpenClawWorkerProvisionerTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl casehub -am -Dtest=ReactiveOpenClawWorkerProvisionerTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 12: Migrate ExampleController to use AgentProviderConfigSource**

In `ExampleController.java`:
- Replace `private final OpenClawCasehubConfig config;` with `private final AgentProviderConfigSource configSource;`
- Update constructor: `OpenClawCasehubConfig config` → `AgentProviderConfigSource configSource`
- Update line 126: `config.agents().get(agentId)` → `configSource.allAgents().get(agentId)`
- Update the null check and `agentEntry.sessionKey()` → `agentConfig.sessionKey()`
- Replace import `io.casehub.openclaw.casehub.OpenClawCasehubConfig` with `io.casehub.openclaw.casehub.AgentProviderConfigSource`

- [ ] **Step 13: Run full app test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add -A
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#36): AgentProviderConfigSource SPI — pluggable agent config with @DefaultBean config-file adapter

Migrate OpenClawWorkerProvisioner, ReactiveOpenClawWorkerProvisioner, and
ExampleController from OpenClawCasehubConfig to AgentProviderConfigSource.

Refs #36"
```

---

### Task 4: MCP endpoint auth — HTTP security policy (#43)

**Files:**
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java`

**Interfaces:**
- Consumes: existing OIDC setup (`casehub-platform-oidc` dependency, OIDC properties)
- Produces: authenticated `/mcp` endpoint (HTTP 401 without bearer token)

- [ ] **Step 1: Write failing security test — unauthenticated MCP returns 401**

Add to `OpenClawRestSecurityTest.java`:

```java
// ==============================
// MCP endpoint — quarkus.http.auth.permission.mcp
// ==============================

@Test
void unauthenticated_mcp_returns401() {
    given().contentType(JSON)
        .body("{\"jsonrpc\":\"2.0\",\"method\":\"initialize\",\"id\":1}")
        .when().post("/mcp")
        .then().statusCode(401);
}

@Test
@TestSecurity(user = "agent", roles = {"openclaw-agent"})
void authenticated_mcp_isNotForbidden() {
    given().contentType(JSON)
        .body("{\"jsonrpc\":\"2.0\",\"method\":\"initialize\",\"id\":1}")
        .when().post("/mcp")
        .then().statusCode(not(in(List.of(401, 403))));
}
```

- [ ] **Step 2: Run test to verify unauthenticated_mcp_returns401 fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawRestSecurityTest#unauthenticated_mcp_returns401 -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — unauthenticated POST to `/mcp` does NOT return 401 (no auth policy configured yet).

- [ ] **Step 3: Add MCP auth policy to main application.properties**

Add after the OIDC section (after line 92) in `app/src/main/resources/application.properties`:

```properties
# MCP endpoint auth — require OIDC bearer token (openclaw#43)
# @RolesAllowed on @Tool methods returns MCP error -32001, not HTTP 401/403 —
# HTTP auth policy is the correct transport-level enforcement.
quarkus.http.auth.permission.mcp.paths=/mcp,/mcp/*
quarkus.http.auth.permission.mcp.policy=authenticated
```

- [ ] **Step 4: Run security tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawRestSecurityTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 5: Run full app test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`

Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/resources/application.properties app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#43): MCP endpoint auth — HTTP security policy for POST /mcp

Require authenticated OIDC bearer token for all MCP endpoint access.
Dev profile security already disabled; tests use @TestSecurity.

Refs #43"
```

---

### Task 5: Cross-repo issue and full build verification

**Files:**
- No code changes

**Interfaces:**
- Consumes: all prior tasks
- Produces: green full build, cross-repo issue filed

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`

Expected: BUILD SUCCESS

- [ ] **Step 2: File cross-repo issue on casehub-ops**

```bash
gh issue create --repo casehubio/casehub-ops --title "feat: add agentIds() to DeploymentProviderConfigStore for consumer enumeration" --body "$(cat <<'EOF'
## Context

casehub-openclaw#36 introduced `AgentProviderConfigSource` — a pluggable SPI for agent configuration. The @DefaultBean reads from `application.properties`. A future `DeploymentAgentProviderConfigSource` will read from `DeploymentProviderConfigStore`, but the store has no enumeration method — only `forAgent(agentId)`.

## What's needed

Add `Set<String> agentIds()` to `DeploymentProviderConfigStore` — trivially exposes `configs.keySet()`.

## Future direction

When multiple repos consume `ProviderConfig` (openclaw, claudony), consider extracting a `ProviderConfigSource` interface to `casehub-ops-api` so consumers depend on the API jar instead of the deployment module.

Refs casehubio/openclaw#36
EOF
)"
```

- [ ] **Step 3: Update CLAUDE.md with DirectCallDeliveryResource new location**

The CLAUDE.md `## Architecture` section describes `DirectCallDeliveryResource` as living in `casehub/`. Update to reflect its new location in `app/`.

- [ ] **Step 4: Commit doc updates**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs: update CLAUDE.md — DirectCallDeliveryResource moved to app/

Refs #50"
```
