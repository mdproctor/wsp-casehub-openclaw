# Design: Epic 4 — CaseHub SPI Implementations

**Issue:** casehubio/openclaw#4  
**Branch:** issue-4-casehub-spi-implementations  
**Date:** 2026-05-29  
**Status:** Approved

---

## 1. Scope

Implements all CaseHub SPI interfaces in `casehub-openclaw`, completing the integration tier:

- `WorkerProvisioner` — registers OpenClaw agents as CaseHub workers
- `CaseChannelProvider` — creates and manages Qhorus channels per case
- `ChannelBackend` — delivers Qhorus COMMANDs to OpenClaw agents (Qhorus → OpenClaw direction)
- `WorkerStatusListener` — reacts to engine lifecycle events
- `OpenClawDeliveryResource` — receives OpenClaw webhook results and writes back to Qhorus (OpenClaw → Qhorus direction)

Also updates `ChannelContextWindowService` (core/) to replace `associate()` with a split `bindAgent()` / `bindChannel()` API that eliminates the coordination race between provisioner and channel provider.

---

## 2. Key Design Decision: Service-Level Join

The original `associate(agentId, Set<channelIds>)` API requires both agentId and channelIds to be known simultaneously. In the actual flow, they're produced independently by different components — the provisioner knows the agentId, the channel provider knows the channelIds. Forcing one side to gather the other's data creates a notification protocol between SPIs that has a provable (if rare) race condition.

**Fix:** Split into two independent writes. The service joins at query time.

```
WorkerProvisioner  → service.bindAgent(agentId, caseId)
CaseChannelProvider → service.bindChannel(caseId, channelId)
query(agentId) → joins agentId→caseId→channelIds internally
```

No coordination between SPIs. No shared registry. No race. This also enables a net improvement: messages that arrive on a channel after `bindChannel()` but before `bindAgent()` are captured and available on the first query.

---

## 3. Component Overview

```
core/
  ChannelContextWindowService   (modified — new bindAgent/bindChannel API)

casehub/
  OpenClawAgentRegistry         (new — caseId↔agentId↔sessionKey; used by ChannelBackend + provisioner)
  OpenClawWorkerProvisioner     (new — implements WorkerProvisioner)
  OpenClawCaseChannelProvider   (new — implements CaseChannelProvider)
  OpenClawChannelBackend        (new — implements ChannelBackend)
  OpenClawWorkerStatusListener  (new — implements WorkerStatusListener)

app/
  OpenClawDeliveryResource      (new — POST /openclaw/delivery/channel/{channelId})
```

**Dependency graph (no cycles):**
```
OpenClawWorkerProvisioner     → ChannelContextWindowService, OpenClawAgentRegistry, OpenClawConfig
OpenClawCaseChannelProvider   → ChannelContextWindowService, ChannelService (Qhorus), MessageService (Qhorus)
OpenClawChannelBackend        → OpenClawAgentRegistry, OpenClawHookClient, OpenClawConfig, ChannelGateway (Qhorus)
OpenClawWorkerStatusListener  → OpenClawAgentRegistry
OpenClawDeliveryResource      → ChannelService (Qhorus), MessageService (Qhorus), WorkerStatusListener
```

---

## 4. ChannelContextWindowService Changes (core/)

### 4.1 API change

Remove `associate(String agentId, Set<UUID> channelIds)`.  
Add:

```java
void bindAgent(String agentId, UUID caseId)
void bindChannel(UUID caseId, UUID channelId)
```

No other public method signatures change.

### 4.2 Internal state additions

```java
ConcurrentHashMap<String, UUID> agentToCase    // agentId → caseId
ConcurrentHashMap<UUID, Set<UUID>> caseChannels // caseId → Set<channelId> (newKeySet())
```

Existing `buffers: ConcurrentHashMap<UUID, ChannelRingBuffer>` is unchanged.

### 4.3 `bindChannel()` behaviour

Creates the ring buffer for the channelId at bind time (previously done in `associate()`):

```java
caseChannels.computeIfAbsent(caseId, id -> ConcurrentHashMap.newKeySet()).add(channelId);
buffers.putIfAbsent(channelId, new ChannelRingBuffer(config.maxMessages(), config.ttl()));
```

### 4.4 `query()` change

Two-level join:
1. `caseId = agentToCase.get(agentId)` → null → `noAssociation()`
2. `channelIds = caseChannels.getOrDefault(caseId, Set.of())` → empty → empty window (not `noAssociation()`)
3. Read from buffers for each channelId (unchanged)

### 4.5 `agentChannels` removal

The existing `agentChannels: ConcurrentHashMap<String, Set<UUID>>` field (used by the old `associate()`) is removed.

### 4.6 Test rewrite

All existing tests for `ChannelContextWindowService` that call `associate()` are rewritten to use `bindAgent()` + `bindChannel()` pairs. New test cases added:
- `bindChannel` before `bindAgent` → messages captured and returned once agent binds
- `bindAgent` without channels → empty window (not `noAssociation()`)

---

## 5. OpenClawAgentRegistry (casehub/)

`@ApplicationScoped`. Maintains bidirectional mapping for routing.

```java
void register(String agentId, UUID caseId, String sessionKey)
void deregister(String agentId)
Optional<String> findAgentId(UUID caseId)       // for ChannelBackend routing
Optional<String> findSessionKey(String agentId) // for invoke() calls
```

Internal: `ConcurrentHashMap<String, UUID> agentToCase` and `ConcurrentHashMap<UUID, String> caseToAgent` and `ConcurrentHashMap<String, String> agentToSessionKey`. All three updated atomically in `register()` and `deregister()`.

This registry is separate from `ChannelContextWindowService`'s internal `agentToCase` map. Both maintain a copy of the agentId↔caseId relation for their own consumers. This duplication is acceptable: the service owns the query join; the registry owns routing concerns.

---

## 6. OpenClawWorkerProvisioner (casehub/)

Implements blocking `WorkerProvisioner` (not reactive — nothing in `provision()` is async).

### 6.1 `provision(Set<String> capabilities, ProvisionContext context)`

1. Resolve `agentId` from `capabilities` via config (see §10)
2. `registry.register(agentId, context.caseId(), sessionKey)`
3. `service.bindAgent(agentId, context.caseId())`
4. Return `Worker(agentId, capabilityList, ctx -> Map.of())`

The `Worker` uses a no-op function holder — OpenClaw is invoked via `ChannelBackend.post()`, not the worker function. The no-op satisfies `Worker.build()`'s non-null requirement.

### 6.2 `terminate(String agentId)`

`registry.deregister(agentId)`. No HTTP call to OpenClaw — termination is declarative (the agent stops receiving commands).

### 6.3 `getCapabilities()`

Returns the union of all configured agent capabilities.

### 6.4 Error handling

Throws `ProvisioningException` if no agent is configured for the requested capabilities.

---

## 7. OpenClawCaseChannelProvider (casehub/)

Implements blocking `CaseChannelProvider`. Mirrors Claudony's implementation but uses a fixed normative layout rather than a configurable one (consolidation deferred to parent#93).

### 7.1 Channel layout

Three channels per case, hardcoded:

| Purpose | Semantic | Allowed Types |
|---|---|---|
| `work` | `AGENT_TO_AGENT` | `COMMAND,RESPONSE,DONE,DECLINE,EXPIRED` |
| `observe` | `OBSERVER` | `EVENT,QUERY,STATUS` |
| `oversight` | `HUMAN_PARTICIPANT` | `COMMAND,RESPONSE` |

### 7.2 `openChannel(UUID caseId, String purpose)`

1. `channelName = CaseChannel.channelName(caseId, purpose)` → `"case-{caseId}/{purpose}"`
2. `channel = channelService.createOrGet(channelName, purpose, semantic, allowedTypes)` — idempotent per SPI contract
3. `service.bindChannel(caseId, channel.id)` — registers the channel with ChannelContextWindow
4. Return `CaseChannel(channel.id, channel.name, purpose, "qhorus", Map.of("qhorus-name", channel.name))`

### 7.3 `postToChannel(...)`

Calls `MessageService.dispatch()` with all 6 params mapped directly. Delegates `correlationId` (String) and `deadline` (ISO-8601 String → `Instant.parse()`) per the SPI contract.

### 7.4 `closeChannel(CaseChannel channel)`

No-op. Qhorus channels are persistent.

### 7.5 `listChannels(UUID caseId)`

`channelService.findByNamePrefix("case-{caseId}/")`

---

## 8. OpenClawChannelBackend (casehub/)

Implements `ChannelBackend`. Self-registers with Qhorus via `ChannelInitialisedEvent` observation — the designed extension point for external backends (per Qhorus deep-dive). This also handles startup recovery automatically: Qhorus fires `ChannelInitialisedEvent` for all existing channels at startup, so the backend re-registers without implementing its own recovery logic.

### 8.1 Identity

```java
String backendId() → "openclaw"
ActorType actorType() → ActorType.AGENT
```

### 8.2 Backend registration

```java
@Inject ChannelGateway gateway;

void onChannelInitialised(@Observes ChannelInitialisedEvent event) {
    if (!event.channelName().startsWith(CaseChannel.CASE_CHANNEL_PREFIX)) return;
    gateway.registerBackend(event.channelId(), this, "agent");
}
```

Only registers for channels whose name starts with `"case-"` (the `CASE_CHANNEL_PREFIX` constant). Non-case channels are ignored.

### 8.3 `open()` / `close()`

Both no-op. Channel lifecycle is managed by `OpenClawCaseChannelProvider`. Registration is via `ChannelInitialisedEvent`, not via `open()`.

### 8.4 `post(ChannelRef channel, OutboundMessage message)`

**Filter:** Only `MessageType.COMMAND` triggers agent invocation. All other types are silently ignored — the message is already stored in the ring buffer by `ChannelContextWindowObserver` and in the Qhorus ledger.

**Routing:**
1. `caseId = extractCaseId(channel.name())` — parses `"case-{caseId}/{purpose}"` → UUID
2. `agentId = registry.findAgentId(caseId)` → not found → log debug, return
3. `sessionKey = registry.findSessionKey(agentId).orElseThrow()` (should always be present if agentId found)
4. `webhookUrl = config.deliveryUrlTemplate().replace("{channelId}", channel.id().toString())`
5. `hookClient.registerSession(agentId, sessionKey, webhookUrl)` — idempotent
6. `hookClient.invoke(agentId, message.content(), config.defaultModel(), config.defaultTimeout())`

**Non-throwing contract:** `OpenClawInvocationException` is caught, logged, and swallowed. `ChannelGateway.fanOut()` is non-fatal — propagated exceptions abort the entire fan-out.

### 8.5 `extractCaseId(String channelName)`

Parses `"case-{caseId}/{purpose}"` by stripping the `"case-"` prefix and taking the segment before the first `/`. Returns `null` if the format doesn't match (non-case channels should not be registered with this backend, but defensive handling is correct).

---

## 9. OpenClawWorkerStatusListener (casehub/)

Implements blocking `WorkerStatusListener`. The engine injects and calls this SPI; our implementation reacts to lifecycle notifications.

```java
void onWorkerStarted(String workerId, Map<String,String> sessionMeta):
  log.infof("OpenClaw agent started: agentId=%s", workerId)

void onWorkerCompleted(String workerId, WorkResult result):
  log.infof("OpenClaw agent completed: agentId=%s status=%s", workerId, result.status())
  registry.deregister(workerId)

void onWorkerStalled(String workerId):
  log.warnf("OpenClaw agent stalled: agentId=%s", workerId)
  events.fire(new OpenClawWorkerStalledEvent(workerId))
```

`OpenClawWorkerStalledEvent` is a public record nested in the listener (matching Claudony's pattern). Observers can react — a future health monitor or alerting component can subscribe without modifying the listener.

---

## 10. OpenClawDeliveryResource (app/)

REST endpoint receiving the OpenClaw → Qhorus write-back.

### 10.1 Endpoint

```
POST /openclaw/delivery/channel/{channelId}
Content-Type: application/json
```

### 10.2 Payload

```java
public record OpenClawDeliveryPayload(
    String agentId,   // WARNING: field name unverified against live API — see openclaw#11
    String output     // WARNING: field name unverified against live API — see openclaw#11
) {}
```

Both fields may need renaming to snake_case (`agent_id`, `output`) depending on what the live OpenClaw API sends. This must be verified before production use (openclaw#11).

### 10.3 Handler

1. `channelService.find(channelId)` → not found → 404
2. `messageService.dispatch(MessageDispatch.builder()` with:
   - `channelId` from path param
   - `sender` = `payload.agentId()`
   - `type` = `MessageType.DONE` (Phase 1 — always DONE; graduation tracked in openclaw#10)
   - `content` = `payload.output()`
   - `actorType` = `ActorType.AGENT`
3. On dispatch success → 200 OK
4. On dispatch failure → log error, still return 200 (OpenClaw must not retry due to our processing errors)

### 10.4 Auth

None. Follows the gateway topology: auth is Claudony's concern. The delivery endpoint is internal — network policy or mTLS provides isolation.

---

## 11. Configuration (app/src/main/resources/application.properties)

### 11.1 Agent capability mapping

```properties
# Map agent IDs to their capability strings
casehub.openclaw.agents.finance-agent.capabilities=finance,banking
casehub.openclaw.agents.finance-agent.session-key=finance-agent
casehub.openclaw.agents.code-review-agent.capabilities=code-review,security-review
casehub.openclaw.agents.code-review-agent.session-key=code-review-agent
```

Resolved via `@ConfigMapping(prefix = "casehub.openclaw")`. The `agents` key is a `Map<String, AgentConfig>` where each value has `capabilities: List<String>` and `sessionKey: String`.

### 11.2 Delivery URL template

```properties
casehub.openclaw.delivery.url-template=http://localhost:8080/openclaw/delivery/channel/{channelId}
```

### 11.3 CDI collision fix (GE-20260428-9311f8)

Required in `app/src/main/resources/application.properties`:

```properties
quarkus.arc.exclude-types=\
  io.casehub.engine.internal.worker.NoOpWorkerStatusListener,\
  io.casehub.engine.internal.worker.NoOpReactiveWorkerStatusListener,\
  io.casehub.engine.internal.worker.EmptyWorkerContextProvider,\
  io.casehub.engine.internal.worker.EmptyReactiveWorkerContextProvider,\
  io.casehub.engine.internal.worker.NoOpWorkerProvisioner,\
  io.casehub.engine.internal.worker.NoOpReactiveWorkerProvisioner,\
  io.casehub.engine.internal.worker.NoOpCaseChannelProvider,\
  io.casehub.engine.internal.worker.NoOpReactiveCaseChannelProvider
```

### 11.4 pom.xml changes (casehub/)

Remove `<scope>provided</scope>` from `casehub-engine` and `casehub-qhorus` — actual SPI beans are now present.

---

## 12. Testing Strategy

### 12.1 core/ — ChannelContextWindowServiceTest (rewrite)

- `bindAgent` + `bindChannel` in both orders → `query()` returns messages correctly
- `bindChannel` before `bindAgent` → messages captured during the window, returned once agent binds
- `bindAgent` without `bindChannel` → `query()` returns empty window (not `noAssociation()`)
- Neither bind → `query()` returns `noAssociation()`
- Overflow eviction — `lastEvictionWindowSeq` set correctly (unchanged behaviour)
- TTL expiry — unchanged behaviour
- `evictExpired()` — unchanged behaviour
- Concurrent `bindAgent` and `bindChannel` calls — no exception, eventual consistency

### 12.2 casehub/ — unit tests (pure JUnit 5 + Mockito)

**OpenClawWorkerProvisionerTest:**
- `provision()` calls `bindAgent()` with correct `(agentId, caseId)`
- `provision()` calls `registry.register()` with correct params
- `provision()` with unknown capabilities → `ProvisioningException`
- `terminate()` calls `registry.deregister(agentId)`
- `getCapabilities()` returns union of all configured capabilities

**OpenClawCaseChannelProviderTest:**
- `openChannel()` calls `bindChannel()` with `(caseId, channelId)`
- `openChannel()` idempotent — same channel name returns same result without error
- `postToChannel()` calls `dispatch()` with correct params including 6-param overload
- `postToChannel()` with null `type` / `correlationId` / `deadline` → builds dispatch without those fields

**OpenClawChannelBackendTest:**
- `onChannelInitialised` with case channel → `gateway.registerBackend()` called
- `onChannelInitialised` with non-case channel → `gateway.registerBackend()` NOT called
- COMMAND message → `hookClient.invoke()` called with correct agentId and content
- Non-COMMAND message (STATUS, EVENT, DONE, RESPONSE) → `hookClient.invoke()` NOT called
- Channel with no registered agent → no-op, no exception
- `hookClient.invoke()` throws → caught, no exception propagated from `post()`
- `extractCaseId` parses `"case-{uuid}/work"` → correct UUID
- `extractCaseId` with non-case channel name → null (no-op)

**OpenClawWorkerStatusListenerTest:**
- `onWorkerCompleted()` → calls `registry.deregister(workerId)`
- `onWorkerStalled()` → fires `OpenClawWorkerStalledEvent`

**OpenClawAgentRegistryTest:**
- `register()` → `findAgentId(caseId)` and `findSessionKey(agentId)` return correct values
- `deregister()` → both lookups return empty
- `findAgentId()` for unknown caseId → empty
- `findSessionKey()` for unknown agentId → empty

### 12.3 app/ — @QuarkusTest

**OpenClawDeliveryResourceTest:**
- Valid channelId, valid payload → `dispatch()` called with DONE type, returns 200
- Unknown channelId → 404
- `dispatch()` throws → still returns 200 (fail-open)
- CDI startup check — implicit via `@QuarkusTest` startup with all SPI impls present (verifies no `AmbiguousResolutionException`)

---

## 13. End-to-End Message Flow

```
Engine calls openChannel(caseId, "work")
  → OpenClawCaseChannelProvider creates Qhorus channel "case-{caseId}/work"
  → ChannelGateway.initChannel() fires ChannelInitialisedEvent
  → OpenClawChannelBackend.onChannelInitialised() → gateway.registerBackend(channelId, this, "agent")
  → service.bindChannel(caseId, channelId)  [ring buffer initialized]

Engine calls provision(capabilities, context)
  → OpenClawWorkerProvisioner resolves agentId
  → registry.register(agentId, caseId, sessionKey)
  → service.bindAgent(agentId, caseId)

Engine calls postToChannel(channel, "engine", "Analyse this PR", COMMAND, correlationId, deadline)
  → OpenClawCaseChannelProvider calls MessageService.dispatch()
  → Qhorus writes message to ledger
  → ChannelContextWindowObserver.onMessage() → service.add() → ring buffer updated
  → ChannelGateway.fanOut() calls OpenClawChannelBackend.post()
  → COMMAND detected → registry.findAgentId(caseId) → agentId
  → hookClient.invoke(agentId, "Analyse this PR", model, timeout)

OpenClaw processes the prompt, runs code-review skill
Python SDK before_prompt_build fires → GET /channel-context/{agentId}?since=0
  → service.query(agentId, 0) → joins agentToCase → reads ring buffer → returns context window
  → context injected into system prompt

OpenClaw completes, POSTs result to webhook URL
  → POST /openclaw/delivery/channel/{channelId}
  → OpenClawDeliveryResource receives payload
  → MessageService.dispatch(DONE, agentOutput, to channel "case-{caseId}/work")
  → QhorusMessageSignalBridge bridges DONE to CaseHubRuntime.signal()
  → Engine resumes case
```

---

## 14. Out of Scope

- Speech act classification Phases 2–3 (openclaw#10)
- OpenClaw webhook payload format verification (openclaw#11)
- Reactive SPI variants (openclaw#12)
- Channel layout consolidation with Claudony (parent#93)
- `WorkerContextProvider` SPI — no OpenClaw-specific lineage context needed at this stage
- Heartbeat mode — not applicable for in-case direct call provisioning
