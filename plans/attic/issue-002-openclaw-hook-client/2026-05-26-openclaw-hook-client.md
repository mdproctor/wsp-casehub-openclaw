# OpenClaw Hook API Client — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `OpenClawHookClient` and supporting types in `core/` — a typed, session-aware HTTP client for the OpenClaw hook API with bearer-token auth and a WireMock integration test.

**Architecture:** `OpenClawGatewayClient` is the raw `@RegisterRestClient` interface; `OpenClawHookClient` wraps it, owns the agentId→session registry, and is the single class `casehub/` SPIs will import for agent invocation. `BearerTokenRequestFilter` injects `Authorization: Bearer <token>` on every outbound request via `@RegisterProvider`.

**Tech Stack:** Java 21, Quarkus 3.32.2, MicroProfile REST Client (`quarkus-rest-client-jackson`), Quarkus CDI (`quarkus-arc`), Mockito (`quarkus-junit5-mockito`), WireMock 3.x (`org.wiremock:wiremock`), AssertJ.

---

## File Structure

```
core/pom.xml                                                   ← add Mockito + WireMock test deps; quarkus-maven-plugin for generate-code-tests

core/src/main/java/io/casehub/openclaw/client/
  OpenClawInvocationException.java                            ← unchecked exception
  OpenClawClientConfig.java                                   ← @ConfigMapping (gateway, delivery, agent sections)
  OpenClawSession.java                                        ← record(agentId, sessionKey, webhookUrl)
  AgentInvocationRequest.java                                 ← record for POST /hooks/agent body + forWebhook() factory
  AgentWakeRequest.java                                       ← record for POST /hooks/wake body
  BearerTokenRequestFilter.java                               ← @ApplicationScoped ClientRequestFilter
  OpenClawGatewayClient.java                                  ← @RegisterRestClient interface
  OpenClawHookClient.java                                     ← @ApplicationScoped session registry + invocation

core/src/test/java/io/casehub/openclaw/client/
  OpenClawWireMockResource.java                               ← QuarkusTestResourceLifecycleManager (dynamic port)
  OpenClawHookClientTest.java                                 ← pure unit tests (Mockito, no Quarkus)
  OpenClawGatewayClientIT.java                                ← @QuarkusTest + WireMock (HTTP + auth)

core/src/test/resources/
  application.properties                                      ← test config defaults
```

---

## Task 1: Test dependencies and Quarkus test support

**Files:**
- Modify: `core/pom.xml`

- [ ] **Step 1: Add test dependencies and Quarkus plugin to core/pom.xml**

In `core/pom.xml`, add inside `<dependencies>`:
```xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5-mockito</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock</artifactId>
      <version>3.13.0</version>
      <scope>test</scope>
    </dependency>
```

Add inside `<build><plugins>` (after the existing jandex plugin):
```xml
      <plugin>
        <groupId>io.quarkus.platform</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${version.quarkus.platform}</version>
        <executions>
          <execution>
            <goals>
              <goal>generate-code</goal>
              <goal>generate-code-tests</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

The `generate-code-tests` goal enables `@QuarkusTest` augmentation in this library module without producing a deployable application (no `build` goal).

- [ ] **Step 2: Verify the pom compiles cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode validate -pl core -am
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add core/pom.xml
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "build(core): add WireMock + Mockito test deps, Quarkus generate-code-tests goal

Refs #2"
```

---

## Task 2: Exception class and session record

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/client/OpenClawInvocationException.java`
- Create: `core/src/main/java/io/casehub/openclaw/client/OpenClawSession.java`

- [ ] **Step 1: Create `OpenClawInvocationException`**

```java
// core/src/main/java/io/casehub/openclaw/client/OpenClawInvocationException.java
package io.casehub.openclaw.client;

public class OpenClawInvocationException extends RuntimeException {

    public OpenClawInvocationException(String message) {
        super(message);
    }

    public OpenClawInvocationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

- [ ] **Step 2: Create `OpenClawSession`**

```java
// core/src/main/java/io/casehub/openclaw/client/OpenClawSession.java
package io.casehub.openclaw.client;

public record OpenClawSession(String agentId, String sessionKey, String webhookUrl) {
}
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/client/OpenClawInvocationException.java \
  core/src/main/java/io/casehub/openclaw/client/OpenClawSession.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): add OpenClawInvocationException and OpenClawSession record

Refs #2"
```

---

## Task 3: Configuration mapping

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/client/OpenClawClientConfig.java`

- [ ] **Step 1: Create `OpenClawClientConfig`**

```java
// core/src/main/java/io/casehub/openclaw/client/OpenClawClientConfig.java
package io.casehub.openclaw.client;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.openclaw")
public interface OpenClawClientConfig {

    Gateway gateway();

    Delivery delivery();

    Agent agent();

    interface Gateway {
        // SmallRye @ConfigMapping auto-converts bearerToken() → bearer-token
        String url();
        String bearerToken();
    }

    interface Delivery {
        // No consumers in this epic — placed here to establish the config boundary
        // for WorkerProvisioner (casehub/ module, later epic), which constructs:
        //   webhookUrl = baseUrl() + "/channel/" + qhorusChannelId
        String baseUrl();
    }

    interface Agent {
        @WithDefault("claude-opus-4-5")
        String defaultModel();

        @WithDefault("120")
        int defaultTimeoutSeconds();
    }
}
```

`@WithName` is omitted throughout — SmallRye Config auto-converts Java camelCase method names to kebab-case config keys (`bearerToken` → `bearer-token`, `defaultModel` → `default-model`, etc.).

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/client/OpenClawClientConfig.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): add OpenClawClientConfig @ConfigMapping

Refs #2"
```

---

## Task 4: Request record types

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/client/AgentInvocationRequest.java`
- Create: `core/src/main/java/io/casehub/openclaw/client/AgentWakeRequest.java`

- [ ] **Step 1: Create `AgentInvocationRequest`**

```java
// core/src/main/java/io/casehub/openclaw/client/AgentInvocationRequest.java
package io.casehub.openclaw.client;

import com.fasterxml.jackson.annotation.JsonInclude;

public record AgentInvocationRequest(
        String message,
        String agentId,
        String deliver,
        String to,
        String model,
        int timeoutSeconds,
        @JsonInclude(JsonInclude.Include.NON_NULL) String sessionName,
        @JsonInclude(JsonInclude.Include.NON_NULL) String wakeMode
) {
    /**
     * Factory for the only delivery mode casehub-openclaw uses. Callers must not
     * construct AgentInvocationRequest directly — use this factory to prevent
     * accidental use of a different deliver value.
     *
     * sessionName: maps to OpenClaw's session_name (Python SDK). JSON field name is
     * "sessionName" (camelCase), consistent with other OpenClaw fields (agentId,
     * timeoutSeconds). If OpenClaw's HTTP API uses snake_case instead, add
     * @JsonProperty("session_name") to the component — to be verified against the
     * live API before casehub/ SPI implementations are built.
     *
     * wakeMode: how the agent is woken. Null uses OpenClaw's default (appropriate for
     * the direct-call pattern). Values are undocumented in the current spec — pass
     * null until verified. If OpenClaw documents required values for webhook delivery
     * mode, update accordingly.
     *
     * /hooks/wake body format: {agentId, message} is assumed based on "lightweight
     * nudge — wakes agent with a text event" in the spec. No body schema is specified.
     * Verify against the live API before relying on wake() in production.
     */
    static AgentInvocationRequest forWebhook(
            String message,
            String agentId,
            String to,
            String model,
            int timeoutSeconds,
            String sessionName,
            String wakeMode) {
        return new AgentInvocationRequest(
                message, agentId, "webhook", to, model, timeoutSeconds, sessionName, wakeMode);
    }
}
```

Both `sessionName` and `wakeMode` carry `@JsonInclude(NON_NULL)` — null values are omitted from the serialized JSON body. OpenClaw deployments that do not support these fields are unaffected.

- [ ] **Step 2: Create `AgentWakeRequest`**

```java
// core/src/main/java/io/casehub/openclaw/client/AgentWakeRequest.java
package io.casehub.openclaw.client;

public record AgentWakeRequest(String agentId, String message) {
}
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/client/AgentInvocationRequest.java \
  core/src/main/java/io/casehub/openclaw/client/AgentWakeRequest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): add AgentInvocationRequest and AgentWakeRequest records

AgentInvocationRequest.forWebhook() factory enforces deliver=webhook.
wakeMode and sessionName are optional/nullable per @JsonInclude(NON_NULL).

Refs #2"
```

---

## Task 5: Gateway client interface and bearer-token filter

**Files:**
- Create: `core/src/main/java/io/casehub/openclaw/client/BearerTokenRequestFilter.java`
- Create: `core/src/main/java/io/casehub/openclaw/client/OpenClawGatewayClient.java`

- [ ] **Step 1: Create `BearerTokenRequestFilter`**

```java
// core/src/main/java/io/casehub/openclaw/client/BearerTokenRequestFilter.java
package io.casehub.openclaw.client;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.client.ClientRequestContext;
import jakarta.ws.rs.client.ClientRequestFilter;

@ApplicationScoped
public class BearerTokenRequestFilter implements ClientRequestFilter {

    private final String token;

    @Inject
    public BearerTokenRequestFilter(OpenClawClientConfig config) {
        this.token = config.gateway().bearerToken();
    }

    BearerTokenRequestFilter(String token) {
        this.token = token;
    }

    @Override
    public void filter(ClientRequestContext requestContext) {
        requestContext.getHeaders().putSingle("Authorization", "Bearer " + token);
    }
}
```

`@ApplicationScoped` is required — Quarkus uses CDI to instantiate the filter when it is registered via `@RegisterProvider`. `@Provider` (JAX-RS server-side annotation) must NOT appear here. The package-private constructor is for direct unit testing.

- [ ] **Step 2: Create `OpenClawGatewayClient`**

```java
// core/src/main/java/io/casehub/openclaw/client/OpenClawGatewayClient.java
package io.casehub.openclaw.client;

import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.rest.client.annotation.RegisterProvider;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;

@RegisterRestClient(configKey = "openclaw-gateway")
@RegisterProvider(BearerTokenRequestFilter.class)
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface OpenClawGatewayClient {

    @POST
    @Path("/hooks/agent")
    Response invokeAgent(AgentInvocationRequest request);

    @POST
    @Path("/hooks/wake")
    Response wakeAgent(AgentWakeRequest request);
}
```

Config key `openclaw-gateway` maps to `quarkus.rest-client.openclaw-gateway.url`. `@RegisterProvider` on the interface — not via `RestClientBuilder` — is the only pattern that reliably applies the filter (GE-20260415-dfa8ba).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/client/BearerTokenRequestFilter.java \
  core/src/main/java/io/casehub/openclaw/client/OpenClawGatewayClient.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): add OpenClawGatewayClient @RegisterRestClient and BearerTokenRequestFilter

Refs #2"
```

---

## Task 6: `OpenClawHookClient` — TDD (session registry + invocation)

**Files:**
- Create: `core/src/test/java/io/casehub/openclaw/client/OpenClawHookClientTest.java`
- Create: `core/src/main/java/io/casehub/openclaw/client/OpenClawHookClient.java`

- [ ] **Step 1: Write the failing session registry and invocation tests**

```java
// core/src/test/java/io/casehub/openclaw/client/OpenClawHookClientTest.java
package io.casehub.openclaw.client;

import jakarta.ws.rs.core.Response;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class OpenClawHookClientTest {

    private OpenClawGatewayClient gatewayClient;
    private OpenClawClientConfig config;
    private OpenClawHookClient hookClient;

    @BeforeEach
    void setup() {
        gatewayClient = mock(OpenClawGatewayClient.class);
        config = mockConfig("claude-opus-4-5", 120);
        hookClient = new OpenClawHookClient(gatewayClient, config);
    }

    // ── Session registry ─────────────────────────────────────────────────────

    @Test
    void registerSession_thenFindSession_returnsSession() {
        hookClient.registerSession("finance-agent", "key-abc", "http://webhook/channel/1");
        assertThat(hookClient.findSession("finance-agent"))
                .isPresent()
                .hasValueSatisfying(s -> {
                    assertThat(s.agentId()).isEqualTo("finance-agent");
                    assertThat(s.sessionKey()).isEqualTo("key-abc");
                    assertThat(s.webhookUrl()).isEqualTo("http://webhook/channel/1");
                });
    }

    @Test
    void findSession_noRegistration_returnsEmpty() {
        assertThat(hookClient.findSession("unknown-agent")).isEmpty();
    }

    @Test
    void deregisterSession_removesMapping() {
        hookClient.registerSession("finance-agent", "key-abc", "http://webhook/channel/1");
        hookClient.deregisterSession("finance-agent");
        assertThat(hookClient.findSession("finance-agent")).isEmpty();
    }

    @Test
    void registerSession_secondRegistration_overwritesFirst() {
        hookClient.registerSession("finance-agent", "key-1", "http://webhook/channel/1");
        hookClient.registerSession("finance-agent", "key-2", "http://webhook/channel/2");
        assertThat(hookClient.findSession("finance-agent"))
                .hasValueSatisfying(s -> assertThat(s.sessionKey()).isEqualTo("key-2"));
    }

    // ── invoke ───────────────────────────────────────────────────────────────

    @Test
    void invoke_noRegisteredSession_throwsInvocationException() {
        assertThatThrownBy(() -> hookClient.invoke("unknown-agent", "hello", null, 0))
                .isInstanceOf(OpenClawInvocationException.class)
                .hasMessageContaining("unknown-agent");
    }

    @Test
    void invoke_registeredSession_callsGatewayWithCorrectRequest() {
        Response ok = mockResponse(200);
        when(gatewayClient.invokeAgent(any())).thenReturn(ok);

        hookClient.registerSession("finance-agent", "session-key-1",
                "http://casehub.test/delivery/channel/abc");
        hookClient.invoke("finance-agent", "Pull transactions", "claude-haiku-4-5-20251001", 45);

        ArgumentCaptor<AgentInvocationRequest> captor =
                ArgumentCaptor.forClass(AgentInvocationRequest.class);
        verify(gatewayClient).invokeAgent(captor.capture());
        AgentInvocationRequest req = captor.getValue();
        assertThat(req.agentId()).isEqualTo("finance-agent");
        assertThat(req.message()).isEqualTo("Pull transactions");
        assertThat(req.deliver()).isEqualTo("webhook");
        assertThat(req.to()).isEqualTo("http://casehub.test/delivery/channel/abc");
        assertThat(req.model()).isEqualTo("claude-haiku-4-5-20251001");
        assertThat(req.timeoutSeconds()).isEqualTo(45);
        assertThat(req.sessionName()).isEqualTo("session-key-1");
    }

    @Test
    void invoke_nullModel_usesConfigDefault() {
        when(gatewayClient.invokeAgent(any())).thenReturn(mockResponse(200));
        hookClient.registerSession("finance-agent", "key", "http://webhook");
        hookClient.invoke("finance-agent", "msg", null, 30);

        ArgumentCaptor<AgentInvocationRequest> captor =
                ArgumentCaptor.forClass(AgentInvocationRequest.class);
        verify(gatewayClient).invokeAgent(captor.capture());
        assertThat(captor.getValue().model()).isEqualTo("claude-opus-4-5");
    }

    @Test
    void invoke_blankModel_usesConfigDefault() {
        when(gatewayClient.invokeAgent(any())).thenReturn(mockResponse(200));
        hookClient.registerSession("finance-agent", "key", "http://webhook");
        hookClient.invoke("finance-agent", "msg", "  ", 30);

        ArgumentCaptor<AgentInvocationRequest> captor =
                ArgumentCaptor.forClass(AgentInvocationRequest.class);
        verify(gatewayClient).invokeAgent(captor.capture());
        assertThat(captor.getValue().model()).isEqualTo("claude-opus-4-5");
    }

    @Test
    void invoke_zeroTimeout_usesConfigDefault() {
        when(gatewayClient.invokeAgent(any())).thenReturn(mockResponse(200));
        hookClient.registerSession("finance-agent", "key", "http://webhook");
        hookClient.invoke("finance-agent", "msg", "claude-opus-4-5", 0);

        ArgumentCaptor<AgentInvocationRequest> captor =
                ArgumentCaptor.forClass(AgentInvocationRequest.class);
        verify(gatewayClient).invokeAgent(captor.capture());
        assertThat(captor.getValue().timeoutSeconds()).isEqualTo(120);
    }

    @Test
    void invoke_gatewayReturns5xx_throwsInvocationException() {
        when(gatewayClient.invokeAgent(any())).thenReturn(mockResponse(503));
        hookClient.registerSession("finance-agent", "key", "http://webhook");
        assertThatThrownBy(() -> hookClient.invoke("finance-agent", "msg", null, 0))
                .isInstanceOf(OpenClawInvocationException.class)
                .hasMessageContaining("503");
    }

    // ── wake ─────────────────────────────────────────────────────────────────

    @Test
    void wake_callsGatewayWithoutRequiringSession() {
        when(gatewayClient.wakeAgent(any())).thenReturn(mockResponse(200));
        hookClient.wake("home-agent", "Check boiler");

        ArgumentCaptor<AgentWakeRequest> captor =
                ArgumentCaptor.forClass(AgentWakeRequest.class);
        verify(gatewayClient).wakeAgent(captor.capture());
        assertThat(captor.getValue().agentId()).isEqualTo("home-agent");
        assertThat(captor.getValue().message()).isEqualTo("Check boiler");
    }

    @Test
    void wake_gatewayReturns5xx_throwsInvocationException() {
        when(gatewayClient.wakeAgent(any())).thenReturn(mockResponse(500));
        assertThatThrownBy(() -> hookClient.wake("home-agent", "msg"))
                .isInstanceOf(OpenClawInvocationException.class)
                .hasMessageContaining("500");
    }

    // ── helpers ──────────────────────────────────────────────────────────────

    private static Response mockResponse(int status) {
        Response r = mock(Response.class);
        when(r.getStatus()).thenReturn(status);
        return r;
    }

    private static OpenClawClientConfig mockConfig(String model, int timeoutSecs) {
        OpenClawClientConfig config = mock(OpenClawClientConfig.class);
        OpenClawClientConfig.Agent agent = mock(OpenClawClientConfig.Agent.class);
        when(config.agent()).thenReturn(agent);
        when(agent.defaultModel()).thenReturn(model);
        when(agent.defaultTimeoutSeconds()).thenReturn(timeoutSecs);
        return config;
    }
}
```

- [ ] **Step 2: Run tests — confirm they fail (class missing)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -am \
  -Dtest=OpenClawHookClientTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compilation error — `OpenClawHookClient` does not exist.

- [ ] **Step 3: Create `OpenClawHookClient`**

```java
// core/src/main/java/io/casehub/openclaw/client/OpenClawHookClient.java
package io.casehub.openclaw.client;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.rest.client.inject.RestClient;

import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class OpenClawHookClient {

    private final OpenClawGatewayClient gatewayClient;
    private final OpenClawClientConfig config;
    private final ConcurrentHashMap<String, OpenClawSession> sessions = new ConcurrentHashMap<>();

    @Inject
    public OpenClawHookClient(@RestClient OpenClawGatewayClient gatewayClient,
                               OpenClawClientConfig config) {
        this.gatewayClient = gatewayClient;
        this.config = config;
    }

    OpenClawHookClient(OpenClawGatewayClient gatewayClient, OpenClawClientConfig config) {
        this.gatewayClient = gatewayClient;
        this.config = config;
    }

    // ── Session registry ─────────────────────────────────────────────────────

    public void registerSession(String agentId, String sessionKey, String webhookUrl) {
        sessions.put(agentId, new OpenClawSession(agentId, sessionKey, webhookUrl));
    }

    public void deregisterSession(String agentId) {
        sessions.remove(agentId);
    }

    public Optional<OpenClawSession> findSession(String agentId) {
        return Optional.ofNullable(sessions.get(agentId));
    }

    // ── Invocation ───────────────────────────────────────────────────────────

    public void invoke(String agentId, String message, String model, int timeoutSeconds) {
        OpenClawSession session = sessions.get(agentId);
        if (session == null) {
            throw new OpenClawInvocationException(
                    "No session registered for agentId: " + agentId);
        }
        String effectiveModel = (model == null || model.isBlank())
                ? config.agent().defaultModel()
                : model;
        int effectiveTimeout = (timeoutSeconds > 0)
                ? timeoutSeconds
                : config.agent().defaultTimeoutSeconds();
        AgentInvocationRequest request = AgentInvocationRequest.forWebhook(
                message, agentId, session.webhookUrl(),
                effectiveModel, effectiveTimeout, session.sessionKey(), null);
        Response response = gatewayClient.invokeAgent(request);
        try {
            if (response.getStatus() / 100 != 2) {
                throw new OpenClawInvocationException(
                        "OpenClaw /hooks/agent returned HTTP " + response.getStatus()
                        + " for agentId: " + agentId);
            }
        } finally {
            response.close();
        }
    }

    public void wake(String agentId, String message) {
        Response response = gatewayClient.wakeAgent(new AgentWakeRequest(agentId, message));
        try {
            if (response.getStatus() / 100 != 2) {
                throw new OpenClawInvocationException(
                        "OpenClaw /hooks/wake returned HTTP " + response.getStatus()
                        + " for agentId: " + agentId);
            }
        } finally {
            response.close();
        }
    }
}
```

`Response.close()` is called in `finally` in both methods — `jakarta.ws.rs.core.Response` declares `close()` but does not implement `AutoCloseable`, so `try-with-resources` does not compile. The `finally` block ensures the connection is returned to the pool on both success and exception paths.

- [ ] **Step 4: Run tests — confirm all 12 pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -am \
  -Dtest=OpenClawHookClientTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `BUILD SUCCESS`, 12 tests green (4 session registry + 6 invoke + 2 wake).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/main/java/io/casehub/openclaw/client/OpenClawHookClient.java \
  core/src/test/java/io/casehub/openclaw/client/OpenClawHookClientTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(core): implement OpenClawHookClient with session registry and invocation

TDD: 12 unit tests pass — session registry, invoke defaults, error paths,
wake without session requirement. Response.close() in finally on both paths.

Refs #2"
```

---

## Task 7: Integration test — HTTP transport and auth filter

**Files:**
- Create: `core/src/test/resources/application.properties`
- Create: `core/src/test/java/io/casehub/openclaw/client/OpenClawWireMockResource.java`
- Create: `core/src/test/java/io/casehub/openclaw/client/OpenClawGatewayClientIT.java`

- [ ] **Step 1: Create test application.properties**

```properties
# core/src/test/resources/application.properties
casehub.openclaw.delivery.base-url=http://casehub.test/openclaw/delivery
casehub.openclaw.agent.default-model=claude-haiku-4-5-20251001
casehub.openclaw.agent.default-timeout-seconds=30

# bearer-token and gateway URL injected by OpenClawWireMockResource at test startup
# (values here are placeholders — overridden by the resource lifecycle manager)
casehub.openclaw.gateway.bearer-token=placeholder
quarkus.rest-client.openclaw-gateway.url=http://localhost:9999

# REST client timeouts — guard against a hung OpenClaw gateway during tests
quarkus.rest-client.openclaw-gateway.connect-timeout=2000
quarkus.rest-client.openclaw-gateway.read-timeout=10000
```

Production `application.properties` in `app/` should set these to appropriate values:
- `connect-timeout`: 5000 ms
- `read-timeout`: at least `agent.default-timeout-seconds * 1000 + 10000` ms (buffer above execution budget)

- [ ] **Step 2: Create `OpenClawWireMockResource`**

```java
// core/src/test/java/io/casehub/openclaw/client/OpenClawWireMockResource.java
package io.casehub.openclaw.client;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;
import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;

import java.util.Map;

public class OpenClawWireMockResource implements QuarkusTestResourceLifecycleManager {

    // Accessible from test class for stubbing and verification
    static WireMockServer INSTANCE;

    private WireMockServer server;

    @Override
    public Map<String, String> start() {
        server = new WireMockServer(WireMockConfiguration.options().dynamicPort());
        server.start();
        INSTANCE = server;
        return Map.of(
                "quarkus.rest-client.openclaw-gateway.url",
                "http://localhost:" + server.port(),
                "casehub.openclaw.gateway.bearer-token",
                "test-bearer-token"
        );
    }

    @Override
    public void stop() {
        if (server != null) server.stop();
    }
}
```

`QuarkusTestResourceLifecycleManager.start()` is called before Quarkus starts, so the config overrides are in place when the REST client is configured. Using `dynamicPort()` avoids port 9090 conflicts with other local services (Quarkus dev mode, Kubernetes sidecars, etc.).

- [ ] **Step 3: Write the integration test**

```java
// core/src/test/java/io/casehub/openclaw/client/OpenClawGatewayClientIT.java
package io.casehub.openclaw.client;

import com.github.tomakehurst.wiremock.WireMockServer;
import io.quarkus.test.common.QuarkusTestResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static com.github.tomakehurst.wiremock.client.WireMock.aResponse;
import static com.github.tomakehurst.wiremock.client.WireMock.equalTo;
import static com.github.tomakehurst.wiremock.client.WireMock.matchingJsonPath;
import static com.github.tomakehurst.wiremock.client.WireMock.notContaining;
import static com.github.tomakehurst.wiremock.client.WireMock.post;
import static com.github.tomakehurst.wiremock.client.WireMock.postRequestedFor;
import static com.github.tomakehurst.wiremock.client.WireMock.urlEqualTo;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
@QuarkusTestResource(OpenClawWireMockResource.class)
class OpenClawGatewayClientIT {

    static WireMockServer wireMock() {
        return OpenClawWireMockResource.INSTANCE;
    }

    @BeforeEach
    void resetWireMock() {
        wireMock().resetAll();
    }

    @Inject
    OpenClawHookClient hookClient;

    // ── POST /hooks/agent ────────────────────────────────────────────────────

    @Test
    void invokeAgent_sendsCorrectJsonBodyAndBearerToken() {
        wireMock().stubFor(post(urlEqualTo("/hooks/agent"))
                .willReturn(aResponse().withStatus(200)));

        hookClient.registerSession("finance-agent", "session-key-xyz",
                "http://casehub.test/openclaw/delivery/channel/ch-001");
        hookClient.invoke("finance-agent", "Pull this month's transactions", null, 0);

        wireMock().verify(postRequestedFor(urlEqualTo("/hooks/agent"))
                .withHeader("Authorization",  equalTo("Bearer test-bearer-token"))
                .withHeader("Content-Type",   equalTo("application/json"))
                .withRequestBody(matchingJsonPath("$.agentId",     equalTo("finance-agent")))
                .withRequestBody(matchingJsonPath("$.message",     equalTo("Pull this month's transactions")))
                .withRequestBody(matchingJsonPath("$.deliver",     equalTo("webhook")))
                .withRequestBody(matchingJsonPath("$.to",          equalTo("http://casehub.test/openclaw/delivery/channel/ch-001")))
                .withRequestBody(matchingJsonPath("$.sessionName", equalTo("session-key-xyz")))
                .withRequestBody(matchingJsonPath("$.model",       equalTo("claude-haiku-4-5-20251001")))
                // integer field — use JSONPath predicate, not equalTo(String)
                .withRequestBody(matchingJsonPath("$[?(@.timeoutSeconds == 30)]")));
    }

    @Test
    void invokeAgent_gatewayReturns500_throwsInvocationException() {
        wireMock().stubFor(post(urlEqualTo("/hooks/agent"))
                .willReturn(aResponse().withStatus(500)));

        hookClient.registerSession("finance-agent", "key", "http://webhook");
        assertThatThrownBy(() -> hookClient.invoke("finance-agent", "msg", null, 0))
                .isInstanceOf(OpenClawInvocationException.class)
                .hasMessageContaining("500");
    }

    @Test
    void invokeAgent_nullSessionKey_sessionNameOmittedFromJson() {
        wireMock().stubFor(post(urlEqualTo("/hooks/agent"))
                .willReturn(aResponse().withStatus(200)));

        hookClient.registerSession("home-agent", null, "http://webhook/channel/2");
        hookClient.invoke("home-agent", "run task", null, 0);

        // @JsonInclude(NON_NULL) on sessionName — the key must be absent from the body
        wireMock().verify(postRequestedFor(urlEqualTo("/hooks/agent"))
                .withRequestBody(notContaining("\"sessionName\"")));
    }

    // ── POST /hooks/wake ─────────────────────────────────────────────────────

    @Test
    void wakeAgent_sendsCorrectJsonBodyAndBearerToken() {
        wireMock().stubFor(post(urlEqualTo("/hooks/wake"))
                .willReturn(aResponse().withStatus(200)));

        hookClient.wake("home-agent", "Time to check the boiler");

        wireMock().verify(postRequestedFor(urlEqualTo("/hooks/wake"))
                .withHeader("Authorization", equalTo("Bearer test-bearer-token"))
                .withRequestBody(matchingJsonPath("$.agentId", equalTo("home-agent")))
                .withRequestBody(matchingJsonPath("$.message", equalTo("Time to check the boiler"))));
    }

    @Test
    void wakeAgent_gatewayReturns500_throwsInvocationException() {
        wireMock().stubFor(post(urlEqualTo("/hooks/wake"))
                .willReturn(aResponse().withStatus(500)));

        assertThatThrownBy(() -> hookClient.wake("home-agent", "msg"))
                .isInstanceOf(OpenClawInvocationException.class)
                .hasMessageContaining("500");
    }
}
```

- [ ] **Step 4: Run all core tests — confirm all pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl core -am
```

Expected: `BUILD SUCCESS` — `OpenClawHookClientTest` (12 tests) + `OpenClawGatewayClientIT` (5 tests) all green.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add \
  core/src/test/java/io/casehub/openclaw/client/OpenClawWireMockResource.java \
  core/src/test/java/io/casehub/openclaw/client/OpenClawGatewayClientIT.java \
  core/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "test(core): add WireMock integration test for OpenClawGatewayClient

Dynamic port via QuarkusTestResourceLifecycleManager. Asserts: JSON body shape,
Authorization header, timeoutSeconds as integer, sessionName NON_NULL omission,
HTTP 5xx -> OpenClawInvocationException.

Refs #2"
```

---

## Task 8: Full build verification

- [ ] **Step 1: Build all Maven modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: `BUILD SUCCESS` — all three modules (core, casehub, app) compile and test clean.

- [ ] **Step 2: Verify no IntelliJ errors**

Run `ide_diagnostics` (errors only) on each new file:
```
core/src/main/java/io/casehub/openclaw/client/OpenClawHookClient.java
core/src/main/java/io/casehub/openclaw/client/OpenClawGatewayClient.java
core/src/main/java/io/casehub/openclaw/client/BearerTokenRequestFilter.java
core/src/main/java/io/casehub/openclaw/client/AgentInvocationRequest.java
core/src/main/java/io/casehub/openclaw/client/OpenClawClientConfig.java
```

Expected: zero errors on all files.

---

## Self-Review

**Spec coverage:**

| Spec requirement | Task |
|---|---|
| `OpenClawGatewayClient` `@RegisterRestClient` for `/hooks/agent` and `/hooks/wake` | Task 5 |
| `BearerTokenRequestFilter` via `@RegisterProvider` | Task 5 |
| `OpenClawHookClient` session registry (`registerSession`, `deregisterSession`, `findSession`) | Task 6 |
| `invoke()` — session lookup, default model/timeout, blank model defaulting, error on missing session, error on 5xx, `Response.close()` | Task 6 |
| `wake()` — no session required, error on 5xx, `Response.close()` | Task 6 |
| `AgentInvocationRequest` with `sessionName` and `wakeMode` as `@JsonInclude(NON_NULL)` | Task 4 |
| `AgentInvocationRequest.forWebhook()` factory — enforce `deliver=webhook` | Task 4 |
| `AgentWakeRequest` | Task 4 |
| `OpenClawClientConfig` `@ConfigMapping` (no redundant `@WithName`) | Task 3 |
| `OpenClawInvocationException` unchecked | Task 2 |
| Integration test: assert request body shape, `Authorization: Bearer` header | Task 7 |
| Integration test: assert `timeoutSeconds` as integer JSONPath predicate | Task 7 |
| Integration test: assert `sessionName` omitted when null | Task 7 |
| REST client connect/read timeout config | Task 7 (application.properties) |
| Dynamic WireMock port via `QuarkusTestResourceLifecycleManager` | Task 7 |

All spec requirements covered. ✓

**Known deferred items (not in plan scope):**
- `sessionName` JSON field name (`sessionName` vs `session_name`) — documented in `forWebhook()` Javadoc; verify against live API before casehub/ SPI work begins
- `wakeMode` values — documented as "null = OpenClaw default"; verify against live API
- `/hooks/wake` body schema — documented as assumed `{agentId, message}`; verify against live API
- Concurrent same-agentId workers — last-write-wins; full fix requires workerId in WorkResult (upstream engine enhancement)

**Type consistency:** `AgentInvocationRequest.forWebhook()` used in Task 6 (`OpenClawHookClient`) with parameters `(message, agentId, to, model, timeoutSeconds, sessionName, wakeMode)` — consistent with definition in Task 4. All 8 record components named identically across test assertions and production code. ✓
