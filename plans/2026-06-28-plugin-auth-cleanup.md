# Plugin Auth, CurrentPrincipal Cleanup, Qhorus MCP Entry — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Secure plugin endpoints with bearer token auth, clean up CurrentPrincipal CDI config, add Qhorus mcpServers entry.

**Architecture:** Custom `HttpAuthenticationMechanism` (bridge to OIDC) creates a `SecurityIdentity` for plugin callers; `OpenClawCurrentPrincipal @Alternative @Priority(150)` handles the bridge identity's tenancyId to avoid `MissingTenancyException` in production; `PluginCommitResource` switches from `@PermitAll` to `@RolesAllowed`. Config cleanups remove redundant CDI exclusions and add Qhorus MCP server documentation.

**Tech Stack:** Java 21 / Quarkus 3.32.2 / `quarkus-vertx-http` security SPI / TypeScript (plugin)

## Global Constraints

- Java 21 source on Java 26 JVM. `JAVA_HOME=$(/usr/libexec/java_home -v 26)`.
- Use `mvn` not `./mvnw`. Always `--batch-mode`.
- Test single class: `mvn --batch-mode test -pl app -am -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`
- All commits reference an issue: `Refs #N` or `Closes #N`.
- Default tenancy UUID: `"278776f9-e1b0-46fb-9032-8bddebdcf9ce"` (hardcoded — matches `OidcCurrentPrincipal`, `FixedCurrentPrincipal`, `Commitment` entity).
- IntelliJ MCP for all class searches and navigation — never bash grep/find for code symbols.
- TDD: write the failing test first, verify it fails, implement, verify it passes.

---

### Task 1: #48 — CurrentPrincipal exclude-types cleanup + config

**Files:**
- Modify: `app/src/main/resources/application.properties:45-64`
- Modify: `app/src/test/resources/application.properties:34-38`

**Interfaces:**
- Consumes: nothing
- Produces: clean CDI config — no exclude-types for CurrentPrincipal, no selected-alternatives for FixedCurrentPrincipal

- [ ] **Step 1: Run the full build to establish baseline**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am
```

Expected: all tests pass (green baseline before any changes).

- [ ] **Step 2: Edit main application.properties — remove CurrentPrincipal exclusions**

In `app/src/main/resources/application.properties`, replace lines 45–64 with:

```properties
# CDI: exclude casehub-engine no-op SPI beans that collide with our blocking implementations.
# Only exclude the 3 blocking SPIs we implement — do NOT exclude reactive or WorkerContextProvider
# (we don't implement those; excluding them would leave them unsatisfied).
# See GE-20260428-9311f8 for root cause (symptom: classLoader=null on startup, CDI debug needed).
quarkus.arc.exclude-types=\
  io.casehub.engine.internal.worker.NoOpWorkerProvisioner,\
  io.casehub.engine.internal.worker.NoOpCaseChannelProvider,\
  io.casehub.engine.internal.worker.NoOpWorkerStatusListener
```

This removes `MockCurrentPrincipal` and `QhorusInboundCurrentPrincipal` from the exclude list and removes the entire stale comment block about them. `OidcCurrentPrincipal @Alternative @Priority(100)` (platform#111) displaces both via standard CDI resolution.

- [ ] **Step 3: Edit test application.properties — remove selected-alternatives + fix comment**

In `app/src/test/resources/application.properties`, replace lines 34–38 with:

```properties
# FixedCurrentPrincipal @Alternative @Priority(200) (platform#112) is the winning
# CurrentPrincipal in tests — displaces OidcCurrentPrincipal @Priority(100) automatically.
# No selected-alternatives config needed.
```

This removes the `quarkus.arc.selected-alternatives` line and fixes the stale comment (which said `@Priority(1)` — bytecode shows `@Priority(200)`).

- [ ] **Step 4: Run tests to verify CDI resolution is unchanged**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am
```

Expected: all tests pass. CDI resolves CurrentPrincipal to FixedCurrentPrincipal in tests (Priority 200 wins) without `selected-alternatives`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/resources/application.properties app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "chore(#48): remove CurrentPrincipal exclude-types + stale selected-alternatives

OidcCurrentPrincipal @Alternative @Priority(100) (platform#111) displaces
MockCurrentPrincipal @DefaultBean and QhorusInboundCurrentPrincipal (plain
bean) via standard CDI resolution. FixedCurrentPrincipal @Priority(200)
(platform#112) wins in tests without selected-alternatives.

Closes #48"
```

---

### Task 2: #47 — Qhorus mcpServers entry in skills README

**Files:**
- Modify: `skills/README.md:81-101`

**Interfaces:**
- Consumes: nothing
- Produces: updated documentation — two mcpServers entries (casehub + qhorus)

- [ ] **Step 1: Update mcpServers config in README**

In `skills/README.md`, replace the JSON block in §3. Configure OpenClaw (lines 81–101) with:

```json
{
  "mcp": {
    "servers": {
      "casehub": {
        "transport": "streamable-http",
        "url": "http://localhost:8080/mcp"
      },
      "qhorus": {
        "transport": "streamable-http",
        "url": "http://localhost:8080/qhorus"
      }
    }
  },
  "plugins": {
    "casehub-openclaw": {
      "baseUrl": "http://localhost:8080",
      "timeoutMs": 3000,
      "casehub": {
        "autoCommit": false
      }
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add skills/README.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs(#47): add Qhorus mcpServers entry at /qhorus

qhorus#306 scoped Qhorus MCP tools to a named server via @McpServer(\"qhorus\").
Clients need a second mcpServers entry alongside the existing casehub entry.

Closes #47"
```

---

### Task 3: #42 — OpenClawGroups.PLUGIN constant + OpenClawCurrentPrincipal + unit test

**Files:**
- Modify: `app/src/main/java/io/casehub/openclaw/app/OpenClawGroups.java`
- Create: `app/src/main/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipal.java`
- Create: `app/src/test/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipalTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.identity.CurrentPrincipal` SPI, `io.casehub.platform.oidc.OidcCurrentPrincipal`, `io.quarkus.security.identity.SecurityIdentity`
- Produces: `OpenClawGroups.PLUGIN = "openclaw-plugin"` constant; `OpenClawCurrentPrincipal` CDI bean at `@Priority(150)`

- [ ] **Step 1: Add PLUGIN constant to OpenClawGroups**

In `app/src/main/java/io/casehub/openclaw/app/OpenClawGroups.java`:

```java
package io.casehub.openclaw.app;

public final class OpenClawGroups {

    public static final String ADMIN = "openclaw-admin";
    public static final String PLUGIN = "openclaw-plugin";

    private OpenClawGroups() {}
}
```

- [ ] **Step 2: Write failing unit test for OpenClawCurrentPrincipal**

Create `app/src/test/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipalTest.java`:

```java
package io.casehub.openclaw.app.security;

import java.security.Principal;
import java.util.Map;
import java.util.Set;

import org.junit.jupiter.api.Test;

import io.casehub.openclaw.app.OpenClawGroups;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.oidc.OidcCurrentPrincipal;
import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.security.runtime.QuarkusPrincipal;
import io.quarkus.security.runtime.QuarkusSecurityIdentity;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

class OpenClawCurrentPrincipalTest {

    private static final String DEFAULT_TENANCY = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";

    @Test
    void bridgeIdentity_tenancyId_returnsDefault() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("openclaw-plugin"))
                .addRole(OpenClawGroups.PLUGIN)
                .addAttribute("casehub.plugin.bridge", true)
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertEquals(DEFAULT_TENANCY, principal.tenancyId());
    }

    @Test
    void bridgeIdentity_actorId_returnsPrincipalName() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("openclaw-plugin"))
                .addRole(OpenClawGroups.PLUGIN)
                .addAttribute("casehub.plugin.bridge", true)
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertEquals("openclaw-plugin", principal.actorId());
    }

    @Test
    void bridgeIdentity_groups_returnsIdentityRoles() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("openclaw-plugin"))
                .addRole(OpenClawGroups.PLUGIN)
                .addAttribute("casehub.plugin.bridge", true)
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertEquals(Set.of(OpenClawGroups.PLUGIN), principal.groups());
    }

    @Test
    void bridgeIdentity_isCrossTenantAdmin_returnsFalse() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("openclaw-plugin"))
                .addRole(OpenClawGroups.PLUGIN)
                .addAttribute("casehub.plugin.bridge", true)
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertFalse(principal.isCrossTenantAdmin());
    }

    @Test
    void nonBridgeIdentity_delegatesToOidc() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("user@example.com"))
                .addRole("some-role")
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);
        when(oidc.tenancyId()).thenReturn("tenant-123");
        when(oidc.actorId()).thenReturn("user@example.com");
        when(oidc.groups()).thenReturn(Set.of("some-role"));
        when(oidc.isCrossTenantAdmin()).thenReturn(false);

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertEquals("tenant-123", principal.tenancyId());
        assertEquals("user@example.com", principal.actorId());
        assertEquals(Set.of("some-role"), principal.groups());
        assertFalse(principal.isCrossTenantAdmin());
        verify(oidc).tenancyId();
        verify(oidc).actorId();
    }

    @Test
    void anonymousIdentity_delegatesToOidc() {
        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setAnonymous(true)
                .setPrincipal(new QuarkusPrincipal("anonymous"))
                .build();
        OidcCurrentPrincipal oidc = mock(OidcCurrentPrincipal.class);
        when(oidc.tenancyId()).thenReturn(DEFAULT_TENANCY);
        when(oidc.actorId()).thenReturn("anonymous");

        OpenClawCurrentPrincipal principal = new OpenClawCurrentPrincipal(identity, oidc);

        assertEquals(DEFAULT_TENANCY, principal.tenancyId());
        verify(oidc).tenancyId();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawCurrentPrincipalTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `OpenClawCurrentPrincipal` class does not exist.

- [ ] **Step 4: Implement OpenClawCurrentPrincipal**

Create `app/src/main/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipal.java`:

```java
package io.casehub.openclaw.app.security;

import java.util.Set;

import jakarta.annotation.Priority;
import jakarta.enterprise.context.RequestScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.oidc.OidcCurrentPrincipal;
import io.quarkus.security.identity.SecurityIdentity;

@RequestScoped
@Alternative
@Priority(150)
public class OpenClawCurrentPrincipal implements CurrentPrincipal {

    private static final String DEFAULT_TENANCY = "278776f9-e1b0-46fb-9032-8bddebdcf9ce";
    private static final String BRIDGE_ATTR = "casehub.plugin.bridge";

    private final SecurityIdentity identity;
    private final OidcCurrentPrincipal oidcPrincipal;

    @Inject
    public OpenClawCurrentPrincipal(SecurityIdentity identity, OidcCurrentPrincipal oidcPrincipal) {
        this.identity = identity;
        this.oidcPrincipal = oidcPrincipal;
    }

    @Override
    public String actorId() {
        if (isBridgeIdentity()) {
            return identity.getPrincipal().getName();
        }
        return oidcPrincipal.actorId();
    }

    @Override
    public Set<String> groups() {
        if (isBridgeIdentity()) {
            return identity.getRoles();
        }
        return oidcPrincipal.groups();
    }

    @Override
    public String tenancyId() {
        if (isBridgeIdentity()) {
            return DEFAULT_TENANCY;
        }
        return oidcPrincipal.tenancyId();
    }

    @Override
    public boolean isCrossTenantAdmin() {
        if (isBridgeIdentity()) {
            return false;
        }
        return oidcPrincipal.isCrossTenantAdmin();
    }

    private boolean isBridgeIdentity() {
        Boolean bridge = identity.getAttribute(BRIDGE_ATTR);
        return bridge != null && bridge;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawCurrentPrincipalTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 6 tests PASS.

- [ ] **Step 6: Run full app test suite to verify no CDI regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am
```

Expected: all tests pass. `OpenClawCurrentPrincipal @Priority(150)` is now the winning `CurrentPrincipal` in production paths. `FixedCurrentPrincipal @Priority(200)` still wins in tests.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/OpenClawGroups.java app/src/main/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipal.java app/src/test/java/io/casehub/openclaw/app/security/OpenClawCurrentPrincipalTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#42): OpenClawCurrentPrincipal + PLUGIN role constant

OpenClawCurrentPrincipal @Alternative @Priority(150) handles bridge-
authenticated identities by returning the default tenancyId, preventing
MissingTenancyException when JpaCommitmentStore queries run under a non-
OIDC SecurityIdentity. Delegates to OidcCurrentPrincipal for OIDC/anonymous
paths. FixedCurrentPrincipal(200) still wins in tests.

Refs #42"
```

---

### Task 4: #42 — PluginTokenBridgeMechanism + config + security tests

**Files:**
- Create: `app/src/main/java/io/casehub/openclaw/app/security/PluginTokenBridgeMechanism.java`
- Modify: `app/src/main/resources/application.properties` (add plugin auth config)
- Modify: `app/src/test/resources/application.properties` (add test token)
- Modify: `app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/PluginCommitResource.java` (replace @PermitAll with @RolesAllowed)
- Modify: `app/src/test/java/io/casehub/openclaw/app/PluginCommitResourceTest.java` (add @TestSecurity)

**Interfaces:**
- Consumes: `OpenClawGroups.PLUGIN` from Task 3; `HttpAuthenticationMechanism` SPI from `quarkus-vertx-http`
- Produces: authenticated `/openclaw/plugin/*` endpoints; `PluginTokenBridgeMechanism` CDI bean

- [ ] **Step 1: Write failing security test — unauthenticated plugin returns 401**

In `app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java`, replace the `permitAll_pluginCommitments_noAuthRequired` test (line 82–87) with:

```java
    // ==============================
    // Plugin endpoints — @RolesAllowed(OpenClawGroups.PLUGIN) + plugin-token mechanism
    // ==============================

    @Test
    void unauthenticated_plugin_returns401() {
        given()
            .when().get("/openclaw/plugin/commitments/test-agent")
            .then().statusCode(401);
    }

    @Test
    @TestSecurity(user = "plugin", roles = {OpenClawGroups.PLUGIN})
    void authenticated_plugin_isNotForbidden() {
        given()
            .when().get("/openclaw/plugin/commitments/test-agent")
            .then().statusCode(not(in(List.of(401, 403))));
    }

    @Test
    void validBearerToken_plugin_passesAuth() {
        given()
            .header("Authorization", "Bearer test-plugin-token")
            .when().get("/openclaw/plugin/commitments/test-agent")
            .then().statusCode(not(in(List.of(401, 403))));
    }

    @Test
    void invalidBearerToken_plugin_returns401() {
        given()
            .header("Authorization", "Bearer wrong-token")
            .when().get("/openclaw/plugin/commitments/test-agent")
            .then().statusCode(401);
    }

    @Test
    void pluginToken_doesNotAuthenticateMcpEndpoint() {
        given()
            .header("Authorization", "Bearer test-plugin-token")
            .contentType(JSON)
            .body("{\"jsonrpc\":\"2.0\",\"method\":\"initialize\",\"id\":1}")
            .when().post("/mcp")
            .then().statusCode(401);
    }
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawRestSecurityTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: `unauthenticated_plugin_returns401` FAILS (currently @PermitAll, returns 200 not 401). Other new tests may also fail since the mechanism doesn't exist yet.

- [ ] **Step 3: Add test token to test application.properties**

Append to `app/src/test/resources/application.properties`:

```properties

# Plugin bridge mechanism test token (openclaw#42)
%test.casehub.openclaw.plugin.bearer-token=test-plugin-token
```

- [ ] **Step 4: Add plugin auth config to main application.properties**

Append to `app/src/main/resources/application.properties` (after the MCP auth block, before the dev profile section):

```properties

# Plugin endpoint auth — bridge mechanism validates pre-shared bearer token (openclaw#42).
# Migrate to OIDC client-credentials when available (openclaw#52).
casehub.openclaw.plugin.bearer-token=${OPENCLAW_PLUGIN_TOKEN}

# Plugin endpoints: require auth via bridge mechanism
quarkus.http.auth.permission.plugin.paths=/openclaw/plugin/*
quarkus.http.auth.permission.plugin.policy=authenticated
quarkus.http.auth.permission.plugin.auth-mechanism=plugin-token
```

Add to the dev profile section (alongside the existing `%dev` lines):

```properties
%dev.quarkus.http.auth.permission.plugin.policy=permit
%dev.casehub.openclaw.plugin.bearer-token=dev-unused
```

- [ ] **Step 5: Implement PluginTokenBridgeMechanism**

Create `app/src/main/java/io/casehub/openclaw/app/security/PluginTokenBridgeMechanism.java`:

```java
package io.casehub.openclaw.app.security;

import java.util.Set;

import jakarta.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import io.casehub.openclaw.app.OpenClawGroups;
import io.quarkus.security.AuthenticationFailedException;
import io.quarkus.security.identity.IdentityProviderManager;
import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.security.identity.request.AuthenticationRequest;
import io.quarkus.security.runtime.QuarkusPrincipal;
import io.quarkus.security.runtime.QuarkusSecurityIdentity;
import io.quarkus.vertx.http.runtime.security.ChallengeData;
import io.quarkus.vertx.http.runtime.security.HttpAuthenticationMechanism;
import io.quarkus.vertx.http.runtime.security.HttpCredentialTransport;
import io.smallrye.mutiny.Uni;
import io.vertx.ext.web.RoutingContext;

@ApplicationScoped
public class PluginTokenBridgeMechanism implements HttpAuthenticationMechanism {

    @ConfigProperty(name = "casehub.openclaw.plugin.bearer-token")
    String configuredToken;

    @Override
    public Uni<SecurityIdentity> authenticate(RoutingContext context,
                                              IdentityProviderManager identityProviderManager) {
        String authHeader = context.request().getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return Uni.createFrom().failure(
                    new AuthenticationFailedException("Bearer token required for plugin endpoints"));
        }

        String token = authHeader.substring(7).trim();
        if (!configuredToken.equals(token)) {
            return Uni.createFrom().failure(
                    new AuthenticationFailedException("Invalid plugin token"));
        }

        SecurityIdentity identity = QuarkusSecurityIdentity.builder()
                .setPrincipal(new QuarkusPrincipal("openclaw-plugin"))
                .addRole(OpenClawGroups.PLUGIN)
                .addAttribute("casehub.plugin.bridge", true)
                .build();

        return Uni.createFrom().item(identity);
    }

    @Override
    public Uni<ChallengeData> getChallenge(RoutingContext context) {
        return Uni.createFrom().item(
                new ChallengeData(401, "WWW-Authenticate", "Bearer realm=\"openclaw-plugin\""));
    }

    @Override
    public Uni<HttpCredentialTransport> getCredentialTransport(RoutingContext context) {
        return Uni.createFrom().item(
                new HttpCredentialTransport(
                        HttpCredentialTransport.Type.AUTHORIZATION, "bearer", "plugin-token"));
    }

    @Override
    public Set<Class<? extends AuthenticationRequest>> getCredentialTypes() {
        return Set.of();
    }
}
```

- [ ] **Step 6: Replace @PermitAll with @RolesAllowed on PluginCommitResource**

In `app/src/main/java/io/casehub/openclaw/app/PluginCommitResource.java`:

Replace the import:
```java
import jakarta.annotation.security.PermitAll;
```
with:
```java
import jakarta.annotation.security.RolesAllowed;
```

Replace the class-level annotation:
```java
@PermitAll
```
with:
```java
@RolesAllowed(OpenClawGroups.PLUGIN)
```

Update the Javadoc: replace the `@PermitAll` explanation paragraph with:
```java
 * <p>@RolesAllowed(OpenClawGroups.PLUGIN): authenticated by PluginTokenBridgeMechanism
 * via pre-shared bearer token. Migrate to OIDC client-credentials when available (openclaw#52).
```

- [ ] **Step 7: Add @TestSecurity to PluginCommitResourceTest**

In `app/src/test/java/io/casehub/openclaw/app/PluginCommitResourceTest.java`, add `@TestSecurity` at class level. Add the import:

```java
import io.quarkus.test.security.TestSecurity;
import io.casehub.openclaw.app.OpenClawGroups;
```

Add the annotation to the class:
```java
@QuarkusTest
@TestSecurity(user = "plugin", roles = {OpenClawGroups.PLUGIN})
class PluginCommitResourceTest {
```

- [ ] **Step 8: Run security tests to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=OpenClawRestSecurityTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all security tests pass — unauthenticated returns 401, valid token passes, invalid token returns 401, plugin token doesn't authenticate at /mcp.

- [ ] **Step 9: Run full app test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am
```

Expected: all tests pass including PluginCommitResourceTest (via @TestSecurity).

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add app/src/main/java/io/casehub/openclaw/app/security/PluginTokenBridgeMechanism.java app/src/main/java/io/casehub/openclaw/app/PluginCommitResource.java app/src/main/resources/application.properties app/src/test/resources/application.properties app/src/test/java/io/casehub/openclaw/app/security/OpenClawRestSecurityTest.java app/src/test/java/io/casehub/openclaw/app/PluginCommitResourceTest.java
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#42): PluginTokenBridgeMechanism + @RolesAllowed on plugin endpoints

Custom HttpAuthenticationMechanism validates pre-shared bearer token for
/openclaw/plugin/* via auth-mechanism=plugin-token policy binding.
PluginCommitResource switches from @PermitAll to @RolesAllowed(PLUGIN).
SecurityIdentity stamped with casehub.plugin.bridge attribute for
OpenClawCurrentPrincipal tenancyId handling.

Refs #42"
```

---

### Task 5: #42 — TypeScript plugin bearer token support + README update

**Files:**
- Modify: `plugin/src/commitment-manager.ts`
- Modify: `plugin/src/index.ts`
- Modify: `skills/README.md`

**Interfaces:**
- Consumes: `api.config.casehub.pluginToken` from OpenClaw plugin config
- Produces: `Authorization: Bearer <token>` header on all plugin HTTP calls

- [ ] **Step 1: Update CommitmentManager to accept and send token**

In `plugin/src/commitment-manager.ts`, add `pluginToken` parameter to constructor and send it in HTTP calls.

Replace the constructor (lines 60–64):

```typescript
  private readonly pluginToken: string | undefined;

  constructor(baseUrl: string, timeoutMs: number, autoCommit: boolean, pluginToken?: string) {
    this.baseUrl = baseUrl.replace(/\/$/, "");
    this.timeoutMs = timeoutMs;
    this.autoCommit = autoCommit;
    this.pluginToken = pluginToken;
  }
```

Replace the `post` method (lines 138–155):

```typescript
  private async post<T>(path: string, body: unknown): Promise<T> {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), this.timeoutMs);
    try {
      const headers: Record<string, string> = { "Content-Type": "application/json" };
      if (this.pluginToken) {
        headers["Authorization"] = `Bearer ${this.pluginToken}`;
      }
      const resp = await fetch(`${this.baseUrl}${path}`, {
        method: "POST",
        headers,
        body: JSON.stringify(body),
        signal: controller.signal,
      });
      if (!resp.ok) {
        throw new Error(`HTTP ${resp.status} from ${path}`);
      }
      return resp.json() as Promise<T>;
    } finally {
      clearTimeout(timer);
    }
  }
```

Replace the `get` method (lines 157–171):

```typescript
  private async get<T>(path: string): Promise<T> {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), this.timeoutMs);
    try {
      const headers: Record<string, string> = {};
      if (this.pluginToken) {
        headers["Authorization"] = `Bearer ${this.pluginToken}`;
      }
      const resp = await fetch(`${this.baseUrl}${path}`, {
        headers,
        signal: controller.signal,
      });
      if (!resp.ok) {
        throw new Error(`HTTP ${resp.status} from ${path}`);
      }
      return resp.json() as Promise<T>;
    } finally {
      clearTimeout(timer);
    }
  }
```

- [ ] **Step 2: Update index.ts to read pluginToken from config**

In `plugin/src/index.ts`, update the `register` function (lines 90–102):

```typescript
export function register(api: OpenClawPluginApi): void {
  const cfg = api.config ?? {};
  const baseUrl = cfg.baseUrl ?? "http://localhost:8080";
  const timeoutMs = cfg.timeoutMs ?? 3000;
  const autoCommit = cfg.casehub?.autoCommit ?? false;
  const pluginToken = cfg.casehub?.pluginToken as string | undefined;

  new ChannelContextPlugin(baseUrl, timeoutMs).register(api);

  const commitmentMgr = new CommitmentManager(baseUrl, timeoutMs, autoCommit, pluginToken);
  api.on("before_tool_call", (event) => commitmentMgr.onBeforeToolCall(event));
  api.on("agent_end", (event) => commitmentMgr.onAgentEnd(event));
  api.on("session_start", (event) => commitmentMgr.onSessionStart(event));
}
```

- [ ] **Step 3: Update skills/README.md plugin config example**

In `skills/README.md`, update the `plugins` section of the config example (already modified in Task 2) to include `pluginToken`. The full JSON block should now be:

```json
{
  "mcp": {
    "servers": {
      "casehub": {
        "transport": "streamable-http",
        "url": "http://localhost:8080/mcp"
      },
      "qhorus": {
        "transport": "streamable-http",
        "url": "http://localhost:8080/qhorus"
      }
    }
  },
  "plugins": {
    "casehub-openclaw": {
      "baseUrl": "http://localhost:8080",
      "timeoutMs": 3000,
      "casehub": {
        "autoCommit": false,
        "pluginToken": "<your-plugin-token>"
      }
    }
  }
}
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/src/commitment-manager.ts plugin/src/index.ts skills/README.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(#42): TypeScript plugin sends bearer token on plugin HTTP calls

CommitmentManager reads pluginToken from config and adds Authorization:
Bearer header to all /openclaw/plugin/* requests. ChannelClient unchanged —
/channel-context/* protection deferred to openclaw#51.

Refs #42"
```

---

### Task 6: Final verification — full build + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` (update OpenClawGroups and PluginCommitResource descriptions if needed)

**Interfaces:**
- Consumes: all prior tasks
- Produces: green full build, updated project docs

- [ ] **Step 1: Run full Maven build (all modules)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install
```

Expected: BUILD SUCCESS across all modules.

- [ ] **Step 2: Review CLAUDE.md for accuracy**

Check that the `## Architecture` section's description of `app/` still accurately describes the plugin endpoints. Update if the `@PermitAll` → `@RolesAllowed` change or the new security classes need to be reflected.

Specifically:
- `PluginCommitResource` description should reflect `@RolesAllowed` auth
- `app/` module should mention `PluginTokenBridgeMechanism` and `OpenClawCurrentPrincipal` in the security infrastructure

- [ ] **Step 3: Commit any CLAUDE.md updates**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs(#42): update CLAUDE.md for plugin auth + OpenClawCurrentPrincipal

Refs #42"
```
