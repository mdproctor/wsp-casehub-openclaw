# Design Journal — issue-003-channel-context-window

## §ChannelContextWindow service — 2026-05-28

### Key decisions

**agentId → channels mapping: explicit `associate()` rather than auto-discovery or global window.**
Three alternatives were evaluated and rejected:
- Auto-discovery (senderId): agents are receivers of cross-channel context, not just senders — grocery-agent seeing finance-agent's posts is the entire point. senderId-based discovery breaks this by construction.
- Channel name convention: creates a hidden unenforced contract between `CaseChannelProvider` naming and context filtering; doesn't generalise to case-scoped shared channels.
- Global window: correct for single-case POC; breaks at scale (cross-case leakage) and forecloses auth retrofit.

`ChannelContextWindowService.associate(agentId, channelIds)` is the authoritative API. Called by `WorkerProvisioner` (Epic 4). Additive — never removes channels.

**`windowSeq` cursor: internal `AtomicLong`, not Qhorus sequenceNumber.**
`MessageReceivedEvent` carries no timestamp and no sequenceNumber. Qhorus sequenceNumber is per-channel (restarts at 1 per channel) — unusable as a cross-channel cursor. `ChannelContextWindowService` maintains its own global monotonic counter (`AtomicLong windowSeq`). `WindowContent.currentWindowSeq` enables the Python SDK to detect service restarts (`since > currentWindowSeq` → reset cursor to 0).

**`lastEvictionWindowSeq` instead of cumulative `evictionCount`.**
A cumulative count would cause the Python SDK to inject the overflow notice forever after any historical eviction. `lastEvictionWindowSeq` (the windowSeq of the most recently evicted message) lets the SDK gate the notice correctly: inject only if `lastEvictionWindowSeq > since`.

**`agentChannels` stored as immutable `Set` (code quality fix).**
The initial implementation stored a plain `HashSet` — iterating it in `query()` while `associate()` mutated it concurrently could throw `ConcurrentModificationException`. Fixed: the merge function now stores `Collections.unmodifiableSet(merged)`, making concurrent reads in `query()` safe without additional synchronisation.

**`EvictionScheduler` in `app/`, not `core/`.**
Keeps `quarkus-scheduler` off the library module. Scheduler runs at the TTL interval — memory reclaim is best-effort (expired entries may linger up to one additional TTL period), but query correctness is unaffected (TTL filtering applied lazily at every `query()` call).

### Module placement
- `ContextMessage`, `WindowContent`, `ChannelRingBuffer`, `ChannelContextWindowService` → `core/`
- `ChannelContextWindowObserver` (implements `MessageObserver` SPI) → `casehub/`
- `ChannelContextWindowResource`, `EvictionScheduler` → `app/`

### Test counts (Epic 3)
- `ChannelRingBufferTest`: 11 tests (pure JUnit 5)
- `ChannelContextWindowServiceTest`: 16 tests (pure JUnit 5, includes concurrency smoke tests)
- `ChannelContextWindowObserverTest`: 11 tests (pure JUnit 5 + Mockito)
- `ChannelContextWindowResourceTest`: 4 tests (@QuarkusTest + @InjectMock)
