# ProvisionerConfigRegistry Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `AgentProviderConfigSource` with `OpenClawAgentConfigResolver` that delegates to the platform `ProvisionerConfigRegistry` SPI with local-config fallback.

**Architecture:** A concrete `@ApplicationScoped` resolver in `casehub/` wraps the untyped `ProvisionerConfigRegistry` and exposes typed `AgentConfig` records. Union semantics: local config + registry merged, registry wins per-agent. Startup validation catches malformed registry data at pod boot.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api (`ProvisionerConfigRegistry` SPI)

## Global Constraints

- Java 21 source level, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Use `mvn` not `./mvnw`
- All commits reference `Refs #56`
- Test scoping: `mvn test -pl <module> -am -Dsurefire.failIfNoSpecifiedTests=false`
- Spec: `docs/specs/2026-06-29-provisioner-config-registry-design.md`

---

### Task 1: Create OpenClawAgentConfigResolver with tests

**Files:**
- Create: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolver.java`
- Create: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolverTest.java`

**Interfaces:**
- Consumes: `io.casehub.api.spi.ProvisionerConfigRegistry` (engine-api), `io.casehub.openclaw.casehub.OpenClawCasehubConfig` (casehub/)
- Produces: `OpenClawAgentConfigResolver` bean with `allAgents()`, `configFor(String)`, `agentIds()`, nested `AgentConfig` record

- [ ] **Step 1: Write the test class with registry-populated tests**

Create `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolverTest.java`.

Tests to write (each as a separate `@Test` method):

1. `registryPopulated_allAgents_returnsTypedConfigs` — mock registry returns two agents with `Map.of("sessionKey", "sk1", "capabilities", List.of("finance"))`. Assert `allAgents()` returns `AgentConfig("sk1", List.of("finance"))`.
2. `registryEmpty_allAgents_fallsBackToLocalConfig` — mock registry returns empty. Mock `OpenClawCasehubConfig` with one agent entry. Assert `allAgents()` returns config from local.
3. `unionSemantics_agentIds_returnsBothSources` — registry declares `["agent-a"]`, local config has `["agent-b"]`. Assert `agentIds()` returns `Set.of("agent-a", "agent-b")`.
4. `unionSemantics_allAgents_registryOverridesLocal` — both sources declare `"agent-a"` with different sessionKeys. Assert `allAgents().get("agent-a").sessionKey()` matches registry value.
5. `configFor_registryFirst_thenLocalFallback` — registry has `"agent-a"`, local has `"agent-b"`. Assert `configFor("agent-a")` returns registry value, `configFor("agent-b")` returns local value.
6. `configFor_unknownAgent_throwsIllegalArgument` — neither source has `"ghost"`. Assert `assertThrows(IllegalArgumentException.class, () -> resolver.configFor("ghost"))`.
7. `fromRaw_missingSessionKey_throwsWithDescriptiveMessage` — registry returns `Map.of("capabilities", List.of("x"))` for `"bad-agent"`. Assert exception message contains `"bad-agent"` and `"sessionKey"`.
8. `fromRaw_capabilitiesSingleString_coercedToList` — registry returns `Map.of("sessionKey", "sk", "capabilities", "single-cap")`. Assert `configFor("agent").capabilities()` equals `List.of("single-cap")`.
9. `fromRaw_capabilitiesAbsent_defaultsToEmptyList` — registry returns `Map.of("sessionKey", "sk")`. Assert `configFor("agent").capabilities()` equals `List.of()`.
10. `fromRaw_capabilitiesWrongType_throwsWithTypeName` — registry returns `Map.of("sessionKey", "sk", "capabilities", 42)`. Assert exception message contains `"Integer"`.
11. `fromRaw_sessionKeyNonString_coercedViaValueOf` — registry returns `Map.of("sessionKey", 123, "capabilities", List.of())`. Assert `configFor("agent").sessionKey()` equals `"123"`.
12. `errorMessages_includeAgentId` — trigger a missing-sessionKey error. Assert message contains the specific agent ID string.

Mock setup helper:
```java
private static ProvisionerConfigRegistry mockRegistry(Map<String, Map<String, Object>> agents) {
    return new ProvisionerConfigRegistry() {
        @Override public Map<String, Object> configFor(String prov, String id) {
            return agents.getOrDefault(id, Map.of());
        }
        @Override public Set<String> declaredAgentIds(String prov) {
            return agents.keySet();
        }
    };
}

private static OpenClawCasehubConfig mockConfig(Map<String, OpenClawCasehubConfig.AgentEntry> agents) {
    // Return a mock OpenClawCasehubConfig with the given agents map
    // and empty Oversight (Optional.empty() for agentId)
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -am -Dtest=OpenClawAgentConfigResolverTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: compilation failure — `OpenClawAgentConfigResolver` does not exist.

- [ ] **Step 3: Implement OpenClawAgentConfigResolver**

Create `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolver.java` per the spec. Copy the implementation verbatim from the design spec (lines 36–132 of `docs/specs/2026-06-29-provisioner-config-registry-design.md`). The class includes:

- `PROVIDER = "openclaw"`, `KEY_SESSION_KEY`, `KEY_CAPABILITIES` constants
- `AgentConfig` record
- `onStartup(@Observes StartupEvent)` — validates all registry agents at boot
- `allAgents()` — union semantics, local first then registry overlay
- `configFor(String agentId)` — registry-first per-agent with local fallback
- `agentIds()` — union of both sources
- `fromRaw(String agentId, Map<String, Object>)` — typed boundary with validation
- `fromLocalEntry(AgentEntry)` — static helper

Required imports: `io.casehub.api.spi.ProvisionerConfigRegistry`, `io.quarkus.runtime.StartupEvent`, `jakarta.enterprise.context.ApplicationScoped`, `jakarta.enterprise.event.Observes`, `jakarta.inject.Inject`, `org.jboss.logging.Logger`, plus `java.util.*`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -am -Dtest=OpenClawAgentConfigResolverTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: all 12 tests PASS.

- [ ] **Step 5: Add startup validation tests**

Add three more tests to `OpenClawAgentConfigResolverTest`:

13. `startupValidation_malformedAgent_throwsOnStartup` — registry declares `"bad"` with `Map.of("capabilities", List.of())` (no sessionKey). Construct resolver, call `onStartup(null)`. Assert `IllegalArgumentException` thrown.
14. `startupValidation_noOpRegistry_noError` — empty registry. Call `onStartup(null)`. No exception.
15. `startupValidation_allValid_logsSuccessMessage` — registry with two valid agents. Call `onStartup(null)`. No exception thrown (log verification optional).

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -am -Dtest=OpenClawAgentConfigResolverTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: all 15 tests PASS.

- [ ] **Step 7: Commit**

```bash
git add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolver.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawAgentConfigResolverTest.java
git commit -m "feat(#56): add OpenClawAgentConfigResolver — typed ProvisionerConfigRegistry adapter — Refs #56"
```

---

### Task 2: Migrate consumers from AgentProviderConfigSource to OpenClawAgentConfigResolver

**Files:**
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java`
- Modify: `app/src/main/java/io/casehub/openclaw/app/example/ExampleController.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java`
- Modify: `casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java`

**Interfaces:**
- Consumes: `OpenClawAgentConfigResolver` from Task 1
- Produces: updated provisioners and controller using the resolver

- [ ] **Step 1: Update OpenClawWorkerProvisioner**

In `OpenClawWorkerProvisioner.java`:
1. Change field `AgentProviderConfigSource configSource` → `OpenClawAgentConfigResolver resolver`
2. Update constructor parameter accordingly
3. Update `provision()`: `configSource.allAgents().get(agentId).sessionKey()` → `resolver.configFor(agentId).sessionKey()`
4. Update `getCapabilities()`: `configSource.allAgents()` → `resolver.allAgents()`
5. Update `resolveAgentId()`: `configSource.allAgents()` → `resolver.allAgents()`
6. Update import: remove `AgentProviderConfigSource`, add `OpenClawAgentConfigResolver`

- [ ] **Step 2: Update ReactiveOpenClawWorkerProvisioner**

Same changes as Step 1, applied to `ReactiveOpenClawWorkerProvisioner.java`:
1. Field: `AgentProviderConfigSource configSource` → `OpenClawAgentConfigResolver resolver`
2. Constructor parameter updated
3. `provision()`: `configSource.allAgents().get(agentId).sessionKey()` → `resolver.configFor(agentId).sessionKey()`
4. `getCapabilities()`: `configSource.allAgents()` → `resolver.allAgents()`
5. `resolveAgentId()`: `configSource.allAgents()` → `resolver.allAgents()`
6. Imports updated

- [ ] **Step 3: Update ExampleController**

In `ExampleController.java`:
1. Field: `AgentProviderConfigSource configSource` → `OpenClawAgentConfigResolver resolver`
2. Constructor parameter updated
3. Line ~126: `AgentProviderConfigSource.AgentConfig agentConfig = configSource.allAgents().get(agentId)` → `OpenClawAgentConfigResolver.AgentConfig agentConfig = resolver.allAgents().get(agentId)` — preserves existing null-check pattern on the next line
4. Update import

- [ ] **Step 4: Update OpenClawWorkerProvisionerTest**

1. Field: `AgentProviderConfigSource configSource` → `OpenClawAgentConfigResolver resolver` (line 28)
2. `setup()`: build resolver mock instead of configSource
3. Constructor call: pass `resolver` instead of `configSource`
4. `buildConfigSource()` helper → `buildResolver()` helper: return a mock `OpenClawAgentConfigResolver` that delegates to a `Map<String, OpenClawAgentConfigResolver.AgentConfig>`.  Use Mockito `mock(OpenClawAgentConfigResolver.class)` with `when(resolver.allAgents()).thenReturn(...)` and `when(resolver.configFor(anyString())).thenAnswer(inv -> agents.get(inv.getArgument(0)))`.
5. Update all `AgentProviderConfigSource.AgentConfig` references → `OpenClawAgentConfigResolver.AgentConfig`

- [ ] **Step 5: Update ReactiveOpenClawWorkerProvisionerTest**

Same pattern as Step 4, applied to `ReactiveOpenClawWorkerProvisionerTest.java`.

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl casehub -am -Dsurefire.failIfNoSpecifiedTests=false`

Expected: all casehub module tests PASS.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`

Expected: all app module tests PASS.

- [ ] **Step 7: Commit**

```bash
git add casehub/src/main/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisioner.java casehub/src/main/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisioner.java app/src/main/java/io/casehub/openclaw/app/example/ExampleController.java casehub/src/test/java/io/casehub/openclaw/casehub/OpenClawWorkerProvisionerTest.java casehub/src/test/java/io/casehub/openclaw/casehub/ReactiveOpenClawWorkerProvisionerTest.java
git commit -m "refactor(#56): migrate consumers to OpenClawAgentConfigResolver — Refs #56"
```

---

### Task 3: Delete AgentProviderConfigSource and ConfigFileAgentProviderConfigSource

**Files:**
- Delete: `casehub/src/main/java/io/casehub/openclaw/casehub/AgentProviderConfigSource.java`
- Delete: `app/src/main/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSource.java`
- Delete: `app/src/test/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSourceTest.java`

**Interfaces:**
- Consumes: Task 2 must be complete — no remaining references to the deleted types
- Produces: clean codebase with no orphaned SPI

- [ ] **Step 1: Verify no remaining references**

Use IntelliJ `ide_find_references` on `AgentProviderConfigSource` to confirm zero usages remain. If any references are found, go back to Task 2 and fix them first.

- [ ] **Step 2: Delete the three files**

Use IntelliJ `ide_refactor_safe_delete` on each file:
1. `casehub/src/main/java/io/casehub/openclaw/casehub/AgentProviderConfigSource.java`
2. `app/src/main/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSource.java`
3. `app/src/test/java/io/casehub/openclaw/app/ConfigFileAgentProviderConfigSourceTest.java`

- [ ] **Step 3: Full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`

Expected: BUILD SUCCESS, all tests pass (126+ tests, 0 failures).

- [ ] **Step 4: Commit**

```bash
git add -u
git commit -m "refactor(#56): delete AgentProviderConfigSource — replaced by OpenClawAgentConfigResolver — Refs #56"
```

---

### Task 4: Update CLAUDE.md and ARC42STORIES.MD

**Files:**
- Modify: `CLAUDE.md` — update Architecture section references to `AgentProviderConfigSource`
- Modify: `ARC42STORIES.MD` — update any references to the old SPI

**Interfaces:**
- Consumes: Tasks 1–3 complete
- Produces: documentation consistent with the new code

- [ ] **Step 1: Search for stale references in CLAUDE.md**

Search `CLAUDE.md` for `AgentProviderConfigSource` and `ConfigFileAgentProviderConfigSource`. Update any occurrences to reference `OpenClawAgentConfigResolver` and `ProvisionerConfigRegistry` instead.

Key sections to check:
- `## Architecture` → `casehub/` module description mentions `AgentProviderConfigSource`
- `## What This Project Is` → may reference the config source SPI

- [ ] **Step 2: Search for stale references in ARC42STORIES.MD**

Search `ARC42STORIES.MD` for `AgentProviderConfigSource` and `ConfigFileAgentProviderConfigSource`. Update references to reflect the new resolver pattern.

- [ ] **Step 3: Run full build to confirm nothing broke**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md ARC42STORIES.MD
git commit -m "docs(#56): update CLAUDE.md and ARC42STORIES.MD for ProvisionerConfigRegistry migration — Refs #56"
```
