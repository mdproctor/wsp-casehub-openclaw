# Epic 5 — TypeScript Plugin + Python SDK Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the OpenClaw `before_prompt_build` context injection plugin (TypeScript) and companion Python client library, so OpenClaw agents automatically receive recent Qhorus channel activity in their system prompt each turn.

**Architecture:** A TypeScript in-process OpenClaw plugin (`plugin/`) registers a `before_prompt_build` hook via `api.on()`, fetches from the Java ChannelContextWindow REST endpoint, formats the result, and returns `{ appendSystemContext: "..." }`. A separate Python PyPI package (`python/`) provides a synchronous HTTP client and typed Pydantic models for use in Python skill scripts that want to query channel context explicitly.

**Tech Stack:** TypeScript 5.4, Vitest 1.6, Node.js 26 (fetch global built-in), Python 3.14, Pydantic v2, httpx, respx, pytest

**Spec:** `docs/specs/2026-05-29-epic5-python-sdk-design.md`

---

## File Map

**Created:**
- `plugin/openclaw.plugin.json` — plugin manifest
- `plugin/package.json` — npm config (name: `casehub-openclaw-plugin`)
- `plugin/tsconfig.json` — TypeScript config (NodeNext, strict)
- `plugin/vitest.config.ts` — Vitest config
- `plugin/src/types.ts` — all TypeScript interfaces (WindowContent, ContextMessage, plugin API)
- `plugin/src/formatters.ts` — `formatMessages()`, `formatIdle()` (extracted for testability)
- `plugin/src/channel-client.ts` — `ChannelClient.getContext()` (fetch + AbortController)
- `plugin/src/index.ts` — `ChannelContextPlugin` class + `register()` entry point
- `plugin/tests/formatters.test.ts` — 7 formatter unit tests
- `plugin/tests/channel-client.test.ts` — 5 HTTP client tests
- `plugin/tests/index.test.ts` — 10 plugin hook integration tests
- `python/src/casehub_openclaw/models.py` — Pydantic v2 `WindowContent`, `ContextMessage`
- `python/tests/conftest.py` — shared pytest fixtures
- `python/tests/test_channel_client.py` — 9 tests (6 model + 3 client)

**Modified:**
- `python/pyproject.toml` — remove `pytest-asyncio`, `httpx[http2]`; add `respx`, `build`
- `python/src/casehub_openclaw/channel_client.py` — implement sync httpx client with URL encoding
- `python/src/casehub_openclaw/__init__.py` — export `ChannelClient`, `WindowContent`, `ContextMessage`
- `python/README.md` — update installation instructions for TypeScript plugin

**Deleted:**
- `python/src/casehub_openclaw/context_hook.py` — hook logic moved to TypeScript

---

## Task 1: TypeScript project scaffold

**Files:** Create `plugin/openclaw.plugin.json`, `plugin/package.json`, `plugin/tsconfig.json`, `plugin/vitest.config.ts`

- [ ] **Step 1: Create plugin/ directory and manifest**

```json
// plugin/openclaw.plugin.json
{
  "name": "casehub-openclaw",
  "version": "0.2.0",
  "description": "Injects recent Qhorus channel activity as appendSystemContext before each OpenClaw agent turn",
  "entry": "dist/index.js"
}
```

- [ ] **Step 2: Create package.json**

```json
// plugin/package.json
{
  "name": "casehub-openclaw-plugin",
  "version": "0.2.0",
  "type": "module",
  "description": "CaseHub ChannelContextWindow plugin for OpenClaw",
  "main": "dist/index.js",
  "exports": { ".": "./dist/index.js" },
  "engines": { "node": ">=18" },
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "devDependencies": {
    "@types/node": "^20",
    "typescript": "^5.4",
    "vitest": "^1.6"
  }
}
```

- [ ] **Step 3: Create tsconfig.json**

```json
// plugin/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "declaration": true
  },
  "include": ["src"],
  "exclude": ["tests", "dist", "vitest.config.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
// plugin/vitest.config.ts
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: { environment: "node" },
});
```

- [ ] **Step 5: Install dependencies**

```bash
npm --prefix plugin install
```

Expected: `node_modules/` created in `plugin/`, no errors.

- [ ] **Step 6: Commit scaffold**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "chore(plugin): scaffold TypeScript plugin project structure

Refs #5"
```

---

## Task 2: TypeScript types

**Files:** Create `plugin/src/types.ts`

No runtime tests needed — TypeScript interfaces are compile-time only. Verified by `tsc`.

- [ ] **Step 1: Create src/types.ts**

```typescript
// plugin/src/types.ts

// ChannelContextWindow REST response types — mirror Java WindowContent / ContextMessage records

export interface ContextMessage {
  windowSeq: number;
  channelId: string;
  channelName: string | null;
  messageType: string;           // Qhorus MessageType name: "EVENT", "COMMAND", "STATUS", etc.
  senderId: string;
  correlationId: string | null;
  content: string;
  receivedAt: string;            // ISO-8601
}

export interface WindowContent {
  messages: ContextMessage[];
  lastEvictionWindowSeq: number; // -1 if no eviction
  lastWindowSeq: number;
  currentWindowSeq: number;
  agentHasAssociation: boolean;
  lastChannelActivity: string;   // ISO-8601; epoch sentinel = "1970-01-01T00:00:00Z"
}

// OpenClaw Plugin SDK interfaces — structurally assumed from documented plugin examples.
// api.config field name is provisional; verify against upstream types when available.

export interface PluginConfig {
  baseUrl?: string;
  timeoutMs?: number;
}

export interface PluginHookContext {
  agentId: string;
  sessionKey: string;
  channelId?: string;
}

export interface HookResult {
  appendSystemContext?: string;
}

export interface OpenClawPluginApi {
  on(
    event: "before_prompt_build",
    handler: (ctx: PluginHookContext) => Promise<HookResult>,
  ): void;
  // PROVISIONAL: "config" field name assumed from existing plugin examples.
  // If OpenClaw publishes a types package, verify this field name.
  config?: PluginConfig;
}
```

- [ ] **Step 2: Create src/.gitkeep placeholder for channel-client and index (needed for tsc)**

Actually, skip tsc at this point — we have no entry point yet. Verify via test run in Task 5 instead.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/src/types.ts
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(plugin): add TypeScript type interfaces for ChannelContextWindow and OpenClaw Plugin API

Refs #5"
```

---

## Task 3: TypeScript formatters (TDD)

**Files:** Create `plugin/tests/formatters.test.ts`, create `plugin/src/formatters.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// plugin/tests/formatters.test.ts
import { describe, it, expect, vi, afterEach } from "vitest";
import { formatMessages, formatIdle } from "../src/formatters.js";
import type { ContextMessage } from "../src/types.js";

const MESSAGE: ContextMessage = {
  windowSeq: 1,
  channelId: "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  channelName: "household/observe",
  messageType: "EVENT",
  senderId: "finance-agent",
  correlationId: null,
  content: "Monthly budget exhausted.",
  receivedAt: "2026-05-29T10:15:00Z",
};

describe("formatMessages", () => {
  it("formats a single message with UTC time, channel name, and speech act type", () => {
    const result = formatMessages([MESSAGE]);
    expect(result).toBe(
      "**finance-agent** on `household/observe` [EVENT] at 10:15Z:\nMonthly budget exhausted."
    );
  });

  it("joins multiple messages with double newline", () => {
    const second: ContextMessage = { ...MESSAGE, windowSeq: 2, content: "Second message." };
    const result = formatMessages([MESSAGE, second]);
    expect(result).toContain("\n\nMonthly budget exhausted.");
    // Both messages present
    expect(result.split("\n\n")).toHaveLength(2);
  });

  it("falls back to channelId when channelName is null", () => {
    const noName: ContextMessage = { ...MESSAGE, channelName: null };
    const result = formatMessages([noName]);
    expect(result).toContain("`3fa85f64-5717-4562-b3fc-2c963f66afa6`");
  });

  it("renders receivedAt as UTC HH:MMZ", () => {
    const result = formatMessages([MESSAGE]);
    expect(result).toContain("10:15Z");
    // Verify the format — not local time
    expect(result).not.toMatch(/\d{1,2}:\d{2}(?!Z)/); // no time without trailing Z
  });
});

describe("formatIdle", () => {
  afterEach(() => { vi.useRealTimers(); });

  it("returns 'no activity recorded' for epoch sentinel string", () => {
    expect(formatIdle("1970-01-01T00:00:00Z")).toBe(
      "No channel activity recorded for this agent yet."
    );
  });

  it("returns 'no activity recorded' for epoch with millisecond precision", () => {
    // Jackson may produce either format depending on version
    expect(formatIdle("1970-01-01T00:00:00.000Z")).toBe(
      "No channel activity recorded for this agent yet."
    );
  });

  it("returns elapsed minutes from real lastChannelActivity", () => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2026-05-29T12:30:00Z"));
    // 90 minutes before the fixed "now"
    const result = formatIdle("2026-05-29T11:00:00Z");
    expect(result).toBe("No channel activity in the last 90 minute(s).");
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```bash
npm --prefix plugin test
```

Expected: `Cannot find module '../src/formatters.js'`

- [ ] **Step 3: Implement formatters.ts**

```typescript
// plugin/src/formatters.ts
import type { ContextMessage } from "./types.js";

export function formatMessages(messages: ContextMessage[]): string {
  return messages.map((m) => {
    const channel = m.channelName ?? m.channelId;
    const time = new Date(m.receivedAt).toISOString().slice(11, 16) + "Z"; // UTC HH:MMZ
    return `**${m.senderId}** on \`${channel}\` [${m.messageType}] at ${time}:\n${m.content}`;
  }).join("\n\n");
}

export function formatIdle(lastChannelActivity: string): string {
  // Use Date.parse() not string comparison — Jackson may add millisecond precision
  // to "1970-01-01T00:00:00Z" depending on version; Date.parse() returns 0 for both.
  const ts = Date.parse(lastChannelActivity);
  if (ts === 0) return "No channel activity recorded for this agent yet.";
  const elapsedMin = Math.floor((Date.now() - ts) / 60_000);
  return `No channel activity in the last ${elapsedMin} minute(s).`;
}
```

- [ ] **Step 4: Run and confirm PASS**

```bash
npm --prefix plugin test
```

Expected: 7 tests pass, 0 fail.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/src/formatters.ts plugin/tests/formatters.test.ts
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(plugin): implement formatMessages and formatIdle with tests

Refs #5"
```

---

## Task 4: TypeScript HTTP client (TDD)

**Files:** Create `plugin/tests/channel-client.test.ts`, create `plugin/src/channel-client.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// plugin/tests/channel-client.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { ChannelClient } from "../src/channel-client.js";

const BASE_CONTENT = {
  messages: [],
  lastEvictionWindowSeq: -1,
  lastWindowSeq: 0,
  currentWindowSeq: 100,
  agentHasAssociation: true,
  lastChannelActivity: "2026-05-29T10:00:00Z",
};

const mockFetch = vi.fn();
vi.stubGlobal("fetch", mockFetch);

beforeEach(() => { mockFetch.mockReset(); });

describe("ChannelClient.getContext", () => {
  it("returns parsed WindowContent on successful response", async () => {
    mockFetch.mockResolvedValueOnce(
      new Response(JSON.stringify(BASE_CONTENT), { status: 200 })
    );
    const client = new ChannelClient("http://localhost:8080", 3000);
    const result = await client.getContext("home-agent", 0);
    expect(result.agentHasAssociation).toBe(true);
    expect(result.currentWindowSeq).toBe(100);
  });

  it("throws on non-2xx response", async () => {
    mockFetch.mockResolvedValueOnce(new Response(null, { status: 503 }));
    const client = new ChannelClient("http://localhost:8080", 3000);
    await expect(client.getContext("home-agent", 0)).rejects.toThrow("HTTP 503");
  });

  it("throws AbortError on timeout", async () => {
    // Simulate AbortController.abort() firing
    mockFetch.mockRejectedValueOnce(new DOMException("Aborted", "AbortError"));
    const client = new ChannelClient("http://localhost:8080", 1); // 1ms timeout
    await expect(client.getContext("home-agent", 0)).rejects.toThrow();
  });

  it("forwards the since query parameter", async () => {
    mockFetch.mockResolvedValueOnce(
      new Response(JSON.stringify(BASE_CONTENT), { status: 200 })
    );
    const client = new ChannelClient("http://localhost:8080", 3000);
    await client.getContext("home-agent", 42);
    const calledUrl = mockFetch.mock.calls[0][0] as string;
    expect(calledUrl).toContain("since=42");
  });

  it("URL-encodes agentId containing special characters", async () => {
    mockFetch.mockResolvedValueOnce(
      new Response(JSON.stringify(BASE_CONTENT), { status: 200 })
    );
    const client = new ChannelClient("http://localhost:8080", 3000);
    await client.getContext("agent/with/slash", 0);
    const calledUrl = mockFetch.mock.calls[0][0] as string;
    expect(calledUrl).toContain("agent%2Fwith%2Fslash");
    expect(calledUrl).not.toContain("agent/with/slash");
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```bash
npm --prefix plugin test
```

Expected: `Cannot find module '../src/channel-client.js'`

- [ ] **Step 3: Implement channel-client.ts**

```typescript
// plugin/src/channel-client.ts
import type { WindowContent } from "./types.js";

export class ChannelClient {
  constructor(private readonly baseUrl: string, private readonly timeoutMs: number) {}

  async getContext(agentId: string, since: number): Promise<WindowContent> {
    const url =
      `${this.baseUrl}/channel-context/${encodeURIComponent(agentId)}?since=${since}`;
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), this.timeoutMs);
    try {
      const res = await fetch(url, { signal: controller.signal });
      if (!res.ok) throw new Error(`HTTP ${res.status} from ${url}`);
      return (await res.json()) as WindowContent;
    } finally {
      clearTimeout(timer);
    }
  }
}
```

- [ ] **Step 4: Run and confirm PASS**

```bash
npm --prefix plugin test
```

Expected: 12 tests pass (7 formatter + 5 client), 0 fail.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/src/channel-client.ts plugin/tests/channel-client.test.ts
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(plugin): implement ChannelClient with URL encoding and timeout support

Refs #5"
```

---

## Task 5: TypeScript plugin core (TDD)

**Files:** Create `plugin/tests/index.test.ts`, create `plugin/src/index.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// plugin/tests/index.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { ChannelContextPlugin } from "../src/index.js";
import type { OpenClawPluginApi, PluginHookContext, HookResult } from "../src/types.js";

// ---------------------------------------------------------------------------
// Test infrastructure: capture the registered handler by mocking the plugin API
// ---------------------------------------------------------------------------
function makeApi(config?: Record<string, unknown>): {
  api: OpenClawPluginApi;
  callHook: (ctx: PluginHookContext) => Promise<HookResult>;
} {
  let capturedHandler: ((ctx: PluginHookContext) => Promise<HookResult>) | undefined;
  const api: OpenClawPluginApi = {
    on(_event, handler) { capturedHandler = handler; },
    config: config as OpenClawPluginApi["config"],
  };
  return {
    api,
    callHook: (ctx) => {
      if (!capturedHandler) throw new Error("handler not registered");
      return capturedHandler(ctx);
    },
  };
}

// ---------------------------------------------------------------------------
// Helpers for mocking fetch
// ---------------------------------------------------------------------------
const mockFetch = vi.fn();
vi.stubGlobal("fetch", mockFetch);
beforeEach(() => mockFetch.mockReset());
afterEach(() => vi.useRealTimers());

function mockOkResponse(body: object): void {
  mockFetch.mockResolvedValueOnce(new Response(JSON.stringify(body), { status: 200 }));
}

const CTX: PluginHookContext = { agentId: "home-agent", sessionKey: "main" };

const BASE = {
  messages: [] as object[],
  lastEvictionWindowSeq: -1,
  lastWindowSeq: 0,
  currentWindowSeq: 100,
  agentHasAssociation: true,
  lastChannelActivity: "2026-05-29T10:00:00Z",
};

const ONE_MESSAGE = {
  windowSeq: 42,
  channelId: "uuid-1",
  channelName: "household/observe",
  messageType: "EVENT",
  senderId: "finance-agent",
  correlationId: null,
  content: "Budget exhausted.",
  receivedAt: "2026-05-29T10:00:00Z",
};

// ---------------------------------------------------------------------------
// Tests
// ---------------------------------------------------------------------------
describe("ChannelContextPlugin._inject", () => {
  it("returns {} when agentHasAssociation is false", async () => {
    mockOkResponse({ ...BASE, agentHasAssociation: false });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result).toEqual({});
  });

  it("returns {} and resets cursor on service restart (since > currentWindowSeq)", async () => {
    // Prime the cursor: first turn sets cursor to 50
    mockOkResponse({ ...BASE, messages: [ONE_MESSAGE], lastWindowSeq: 50, currentWindowSeq: 100 });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    await callHook(CTX); // cursor = 50

    // Second turn: service restarted — currentWindowSeq (5) < since (50)
    mockOkResponse({ ...BASE, currentWindowSeq: 5, lastWindowSeq: 5 });
    const result = await callHook(CTX);
    expect(result).toEqual({});

    // Third turn: cursor was reset to 0, so fetch uses since=0
    mockOkResponse({ ...BASE, messages: [ONE_MESSAGE], lastWindowSeq: 42 });
    const result3 = await callHook(CTX);
    expect(result3.appendSystemContext).toBeDefined();
    const fetchedUrl = mockFetch.mock.calls[2][0] as string;
    expect(fetchedUrl).toContain("since=0");
  });

  it("injects formatted messages when messages are present", async () => {
    mockOkResponse({ ...BASE, messages: [ONE_MESSAGE], lastWindowSeq: 42 });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result.appendSystemContext).toContain("## Channel Context");
    expect(result.appendSystemContext).toContain("**finance-agent**");
    expect(result.appendSystemContext).toContain("Budget exhausted.");
  });

  it("injects overflow notice AND messages (additive) when both apply", async () => {
    // lastEvictionWindowSeq (10) > since (0): overflow occurred
    // messages also present
    mockOkResponse({
      ...BASE,
      messages: [ONE_MESSAGE],
      lastEvictionWindowSeq: 10,
      lastWindowSeq: 42,
    });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result.appendSystemContext).toContain("evicted");
    expect(result.appendSystemContext).toContain("Budget exhausted.");
  });

  it("injects overflow notice only when eviction occurred but messages is empty", async () => {
    mockOkResponse({
      ...BASE,
      messages: [],
      lastEvictionWindowSeq: 10,
      lastWindowSeq: 0,
    });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result.appendSystemContext).toContain("evicted");
    // No idle notice (would contradict the overflow signal)
    expect(result.appendSystemContext).not.toContain("No channel activity");
  });

  it("injects idle notice with elapsed time when no messages and no eviction", async () => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date("2026-05-29T12:30:00Z"));
    mockOkResponse({
      ...BASE,
      messages: [],
      lastEvictionWindowSeq: -1,
      lastChannelActivity: "2026-05-29T11:00:00Z", // 90 minutes ago
    });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result.appendSystemContext).toContain("No channel activity in the last 90 minute(s).");
  });

  it("injects 'no activity recorded' idle notice when lastChannelActivity is epoch", async () => {
    mockOkResponse({
      ...BASE,
      messages: [],
      lastEvictionWindowSeq: -1,
      lastChannelActivity: "1970-01-01T00:00:00Z",
    });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result.appendSystemContext).toContain("No channel activity recorded for this agent yet.");
  });

  it("returns {} and logs warning on HTTP error (fail open)", async () => {
    mockFetch.mockResolvedValueOnce(new Response(null, { status: 503 }));
    const warnSpy = vi.spyOn(console, "warn").mockImplementation(() => undefined);
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result).toEqual({});
    expect(warnSpy).toHaveBeenCalledWith(expect.stringContaining("[casehub-openclaw]"));
    warnSpy.mockRestore();
  });

  it("returns {} and logs warning on network timeout (fail open)", async () => {
    mockFetch.mockRejectedValueOnce(new DOMException("Aborted", "AbortError"));
    const warnSpy = vi.spyOn(console, "warn").mockImplementation(() => undefined);
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    const result = await callHook(CTX);
    expect(result).toEqual({});
    expect(warnSpy).toHaveBeenCalled();
    warnSpy.mockRestore();
  });

  it("resets cursor to 0 when sessionKey changes (new session for same agent)", async () => {
    // Turn 1: session-A, cursor advances to 42
    mockOkResponse({ ...BASE, messages: [ONE_MESSAGE], lastWindowSeq: 42 });
    const { api, callHook } = makeApi();
    const plugin = new ChannelContextPlugin("http://localhost:8080", 3000);
    plugin.register(api);
    await callHook({ agentId: "home-agent", sessionKey: "session-A" });

    // Turn 2: session-B (daily reset) — cursor should reset to 0
    mockOkResponse({ ...BASE, messages: [ONE_MESSAGE], lastWindowSeq: 10 });
    await callHook({ agentId: "home-agent", sessionKey: "session-B" });

    const secondCallUrl = mockFetch.mock.calls[1][0] as string;
    expect(secondCallUrl).toContain("since=0");
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```bash
npm --prefix plugin test
```

Expected: `Cannot find module '../src/index.js'`

- [ ] **Step 3: Implement index.ts**

```typescript
// plugin/src/index.ts
import { ChannelClient } from "./channel-client.js";
import { formatMessages, formatIdle } from "./formatters.js";
import type {
  ContextMessage,
  HookResult,
  OpenClawPluginApi,
  PluginHookContext,
  WindowContent,
} from "./types.js";

export class ChannelContextPlugin {
  private readonly client: ChannelClient;

  // Cursor keyed by agentId only — bounded by distinct agents, not sessions.
  // Stores sessionKey alongside cursor; resets to 0 when sessionKey changes
  // (OpenClaw resets sessions daily and on idle timeout).
  private readonly cursors = new Map<string, { cursor: number; sessionKey: string }>();

  constructor(baseUrl: string, timeoutMs: number) {
    this.client = new ChannelClient(baseUrl, timeoutMs);
  }

  // Synchronous — OpenClaw snapshots hooks at plugin registration time.
  // Registering from start() (async, post-gateway) silently misses the window.
  register(api: OpenClawPluginApi): void {
    api.on("before_prompt_build", (ctx) => this._inject(ctx));
  }

  private async _inject(ctx: PluginHookContext): Promise<HookResult> {
    const entry = this.cursors.get(ctx.agentId);
    // Reset cursor when sessionKey changes — new session gets full window from buffer
    const since = (entry?.sessionKey === ctx.sessionKey) ? (entry?.cursor ?? 0) : 0;

    let result: WindowContent;
    try {
      result = await this.client.getContext(ctx.agentId, since);
    } catch (err) {
      // Fail open — agent turn must never be blocked by context unavailability
      console.warn(`[casehub-openclaw] context fetch failed for ${ctx.agentId}: ${err}`);
      return {};
    }

    // agentHasAssociation=false covers both: agent not yet wired AND service restarted
    // before re-registration. After re-registration, agentHasAssociation=true and
    // currentWindowSeq will be low, triggering the restart detection below.
    if (!result.agentHasAssociation) return {};

    // Service restart: currentWindowSeq reset below our cursor.
    // Skip this turn; next turn will call with since=0 and get a fresh window.
    if (since > result.currentWindowSeq) {
      this.cursors.set(ctx.agentId, { cursor: 0, sessionKey: ctx.sessionKey });
      return {};
    }

    const parts: string[] = [];

    // Overflow notice — additive with messages, not exclusive.
    // Edge case: if eviction occurred AND all remaining messages also expired (messages=[]),
    // the overflow notice still fires but no idle notice is injected — the empty messages
    // array implicitly signals no current content. A combined notice is not warranted.
    if (result.lastEvictionWindowSeq > since) {
      parts.push(
        "⚠️ Some channel messages were evicted before this turn (high volume). " +
        "Full history is available in the CaseHub audit ledger.",
      );
    }

    // Available messages — always injected when present
    if (result.messages.length > 0) {
      parts.push(formatMessages(result.messages as ContextMessage[]));
    }

    // Idle notice — only when no messages AND no relevant eviction.
    // Injecting idle alongside overflow would be contradictory.
    if (result.messages.length === 0 && result.lastEvictionWindowSeq <= since) {
      parts.push(formatIdle(result.lastChannelActivity));
    }

    // Advance cursor. All non-early-return paths produce at least one part
    // (overflow OR messages OR idle are mutually covering), so parts.length===0
    // is unreachable in practice. Guard kept for defensive correctness.
    this.cursors.set(ctx.agentId, { cursor: result.lastWindowSeq, sessionKey: ctx.sessionKey });

    if (parts.length === 0) return {};
    return { appendSystemContext: "## Channel Context\n\n" + parts.join("\n\n") };
  }
}

export function register(api: OpenClawPluginApi): void {
  const cfg = api.config ?? {};
  new ChannelContextPlugin(
    cfg.baseUrl ?? "http://localhost:8080",
    cfg.timeoutMs ?? 3000,
  ).register(api);
}
```

- [ ] **Step 4: Run and confirm all 22 tests pass**

```bash
npm --prefix plugin test
```

Expected: 22 tests pass (7 formatters + 5 client + 10 index), 0 fail.

- [ ] **Step 5: Run TypeScript compile to verify types**

```bash
npm --prefix plugin run build
```

Expected: `dist/` directory created, no type errors. If errors appear, fix before committing.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add plugin/src/index.ts plugin/tests/index.test.ts plugin/dist/
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(plugin): implement ChannelContextPlugin with before_prompt_build hook

Cursor keyed by agentId with sessionKey tracking — bounded Map, auto-resets on
session change. Fail-open on all error paths.

Refs #5"
```

---

## Task 6: Python package cleanup

**Files:** Modify `python/pyproject.toml`, delete `python/src/casehub_openclaw/context_hook.py`

- [ ] **Step 1: Delete context_hook.py**

```bash
rm /Users/mdproctor/claude/casehub/openclaw/python/src/casehub_openclaw/context_hook.py
```

- [ ] **Step 2: Update pyproject.toml dev dependencies**

Open `python/pyproject.toml`. Replace the `[project.optional-dependencies]` section with:

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "respx>=0.21",
    "build>=1.2",
]
```

(Removes: `pytest-asyncio>=0.23`, `httpx[http2]>=0.27`. Adds: `respx>=0.21`, `build>=1.2`.)

- [ ] **Step 3: Install updated dependencies**

```bash
pip install -e "python/[dev]"
```

Expected: installs respx and build packages, no errors.

- [ ] **Step 4: Verify existing tests still pass (none yet, but package installs cleanly)**

```bash
python -m pytest python/tests/ -v 2>/dev/null || echo "no tests yet — OK"
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add python/pyproject.toml
git -C /Users/mdproctor/claude/casehub/openclaw rm python/src/casehub_openclaw/context_hook.py
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "chore(python): remove context_hook.py stub; update dev deps (respx, build)

Hook registration moves to TypeScript plugin. context_hook.py was a placeholder
based on research spec pseudocode that turned out to be conceptually inaccurate.

Refs #5"
```

---

## Task 7: Python models (TDD)

**Files:** Create `python/src/casehub_openclaw/models.py`, `python/tests/conftest.py`, `python/tests/test_channel_client.py` (model tests)

- [ ] **Step 1: Create conftest.py with shared fixtures**

```python
# python/tests/conftest.py
import pytest

@pytest.fixture
def full_window_json():
    return {
        "messages": [
            {
                "windowSeq": 42,
                "channelId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
                "channelName": "household/observe",
                "messageType": "EVENT",
                "senderId": "finance-agent",
                "correlationId": None,
                "content": "Monthly budget exhausted.",
                "receivedAt": "2026-05-29T10:00:00Z",
            }
        ],
        "lastEvictionWindowSeq": -1,
        "lastWindowSeq": 42,
        "currentWindowSeq": 100,
        "agentHasAssociation": True,
        "lastChannelActivity": "2026-05-29T10:00:00Z",
    }

@pytest.fixture
def no_association_json():
    return {
        "messages": [],
        "lastEvictionWindowSeq": -1,
        "lastWindowSeq": 0,
        "currentWindowSeq": 0,
        "agentHasAssociation": False,
        "lastChannelActivity": "1970-01-01T00:00:00Z",
    }
```

- [ ] **Step 2: Write failing model tests**

```python
# python/tests/test_channel_client.py  (model tests only — client tests added in Task 8)
from datetime import datetime, timezone
from uuid import UUID

import pytest
from casehub_openclaw import ContextMessage, WindowContent


class TestWindowContentDeserialization:
    def test_full_window_parses_all_fields(self, full_window_json):
        result = WindowContent.model_validate(full_window_json)
        assert result.agent_has_association is True
        assert result.current_window_seq == 100
        assert result.last_window_seq == 42
        assert result.last_eviction_window_seq == -1
        assert len(result.messages) == 1

    def test_camelcase_aliases_map_to_snake_case(self, full_window_json):
        result = WindowContent.model_validate(full_window_json)
        msg = result.messages[0]
        assert msg.window_seq == 42
        assert msg.sender_id == "finance-agent"
        assert msg.message_type == "EVENT"
        assert str(msg.channel_id) == "3fa85f64-5717-4562-b3fc-2c963f66afa6"

    def test_null_correlation_id_and_channel_name_become_none(self):
        data = {
            "windowSeq": 1,
            "channelId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
            "channelName": None,
            "messageType": "EVENT",
            "senderId": "agent",
            "correlationId": None,
            "content": "hello",
            "receivedAt": "2026-05-29T10:00:00Z",
        }
        msg = ContextMessage.model_validate(data)
        assert msg.channel_name is None
        assert msg.correlation_id is None

    def test_epoch_last_channel_activity_parses_as_epoch_datetime(self, no_association_json):
        result = WindowContent.model_validate(no_association_json)
        epoch = datetime(1970, 1, 1, tzinfo=timezone.utc)
        assert result.last_channel_activity == epoch

    def test_empty_messages_list_is_valid(self, no_association_json):
        result = WindowContent.model_validate(no_association_json)
        assert result.messages == []
        assert result.agent_has_association is False

    def test_last_eviction_window_seq_minus_one_deserialises_correctly(self, full_window_json):
        result = WindowContent.model_validate(full_window_json)
        assert result.last_eviction_window_seq == -1
```

- [ ] **Step 3: Run and confirm FAIL**

```bash
python -m pytest python/tests/test_channel_client.py -v
```

Expected: `ImportError: cannot import name 'WindowContent' from 'casehub_openclaw'`

- [ ] **Step 4: Create models.py**

```python
# python/src/casehub_openclaw/models.py
from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class ContextMessage(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    window_seq: int = Field(alias="windowSeq")
    channel_id: UUID = Field(alias="channelId")
    channel_name: str | None = Field(alias="channelName")
    message_type: str = Field(alias="messageType")
    sender_id: str = Field(alias="senderId")
    correlation_id: str | None = Field(alias="correlationId")
    content: str
    received_at: datetime = Field(alias="receivedAt")


class WindowContent(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    messages: list[ContextMessage]
    last_eviction_window_seq: int = Field(alias="lastEvictionWindowSeq")
    last_window_seq: int = Field(alias="lastWindowSeq")
    current_window_seq: int = Field(alias="currentWindowSeq")
    agent_has_association: bool = Field(alias="agentHasAssociation")
    last_channel_activity: datetime = Field(alias="lastChannelActivity")
```

- [ ] **Step 5: Update __init__.py to export models (partial — ChannelClient added in Task 8)**

```python
# python/src/casehub_openclaw/__init__.py
from .models import ContextMessage, WindowContent

__all__ = ["ContextMessage", "WindowContent"]
```

- [ ] **Step 6: Run and confirm model tests PASS**

```bash
python -m pytest python/tests/test_channel_client.py -v
```

Expected: 6 tests pass, 0 fail.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add python/src/casehub_openclaw/models.py python/src/casehub_openclaw/__init__.py python/tests/conftest.py python/tests/test_channel_client.py
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(python): Pydantic v2 models for WindowContent and ContextMessage

camelCase aliases map to snake_case Python fields.
Epoch Instant.EPOCH parses correctly as datetime(1970, 1, 1, UTC).

Refs #5"
```

---

## Task 8: Python HTTP client (TDD)

**Files:** Modify `python/src/casehub_openclaw/channel_client.py`, modify `python/src/casehub_openclaw/__init__.py`, modify `python/tests/test_channel_client.py` (add client tests)

- [ ] **Step 1: Append failing client tests to test_channel_client.py**

Append these classes to the existing `python/tests/test_channel_client.py`:

```python
# Append to python/tests/test_channel_client.py

import httpx
import respx
from casehub_openclaw import ChannelClient


class TestChannelClientGetContext:
    @respx.mock
    def test_successful_response_returns_window_content(self, full_window_json):
        respx.get("http://localhost:8080/channel-context/home-agent").mock(
            return_value=httpx.Response(200, json=full_window_json)
        )
        client = ChannelClient("http://localhost:8080")
        result = client.get_context("home-agent", since=0)
        assert result.agent_has_association is True
        assert result.current_window_seq == 100

    @respx.mock
    def test_http_503_raises_http_status_error(self):
        respx.get("http://localhost:8080/channel-context/home-agent").mock(
            return_value=httpx.Response(503)
        )
        client = ChannelClient("http://localhost:8080")
        with pytest.raises(httpx.HTTPStatusError):
            client.get_context("home-agent", since=0)

    @respx.mock
    def test_agent_id_with_slash_is_url_encoded(self, full_window_json):
        # Raw slash in agent_id must be percent-encoded in the URL path
        respx.get("http://localhost:8080/channel-context/agent%2Fwith%2Fslash").mock(
            return_value=httpx.Response(200, json=full_window_json)
        )
        client = ChannelClient("http://localhost:8080")
        # This would raise a ConnectionError if URL encoding is wrong
        # (the mock only matches the encoded URL)
        result = client.get_context("agent/with/slash", since=0)
        assert result.agent_has_association is True
```

- [ ] **Step 2: Run and confirm FAIL**

```bash
python -m pytest python/tests/test_channel_client.py -v
```

Expected: 3 new tests fail with `ImportError: cannot import name 'ChannelClient'`

- [ ] **Step 3: Implement channel_client.py**

```python
# python/src/casehub_openclaw/channel_client.py
from urllib.parse import quote

import httpx

from .models import WindowContent


class ChannelClient:
    """HTTP client for the ChannelContextWindow REST endpoint.

    Uses top-level httpx.get() — one connection per call. For skill scripts
    that call get_context once per execution this is appropriate. For scripts
    making repeated calls, use httpx.Client as a context manager directly.

    Raises:
        httpx.HTTPStatusError: on non-2xx response.
        httpx.TimeoutException: when the request exceeds ``timeout`` seconds.
    """

    def __init__(self, base_url: str, timeout: float = 5.0) -> None:
        self._base_url = base_url.rstrip("/")
        self._timeout = timeout

    def get_context(self, agent_id: str, since: int = 0) -> WindowContent:
        url = f"{self._base_url}/channel-context/{quote(agent_id, safe='')}"
        resp = httpx.get(url, params={"since": since}, timeout=self._timeout)
        resp.raise_for_status()
        return WindowContent.model_validate(resp.json())
```

- [ ] **Step 4: Update __init__.py to export ChannelClient**

```python
# python/src/casehub_openclaw/__init__.py
from .channel_client import ChannelClient
from .models import ContextMessage, WindowContent

__all__ = ["ChannelClient", "ContextMessage", "WindowContent"]
```

- [ ] **Step 5: Run all 9 Python tests and confirm PASS**

```bash
python -m pytest python/tests/test_channel_client.py -v
```

Expected: 9 tests pass, 0 fail. Output should show all tests including URL encoding test.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add python/src/casehub_openclaw/channel_client.py python/src/casehub_openclaw/__init__.py python/tests/test_channel_client.py
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "feat(python): implement ChannelClient with URL encoding

agent_id path-encoded via urllib.parse.quote(safe=''). Raises standard httpx
exceptions; callers decide fail-open vs propagation per their own context.

Refs #5"
```

---

## Task 9: Update README

**Files:** Modify `python/README.md`

The existing README has "Installation (pending Epic 5)" placeholders. Replace with real content.

- [ ] **Step 1: Rewrite python/README.md**

```markdown
# casehub-openclaw Python SDK

Python client library for the CaseHub ChannelContextWindow REST API. Used by
OpenClaw Python skill scripts to query recent Qhorus channel activity.

> **Note:** Automatic `before_prompt_build` injection is handled by the
> **TypeScript plugin** (`casehub-openclaw-plugin` on npm), not this package.
> See the TypeScript plugin's README for automatic injection setup.

## What this package is for

OpenClaw skill scripts are Python scripts invoked as supporting resources
in SKILL.md files. This package provides a typed HTTP client so Python skill
scripts can explicitly query the ChannelContextWindow before acting.

## Installation

```bash
pip install casehub-openclaw
```

Requires a running `casehub-openclaw` Quarkus app instance.

## Usage

```python
from casehub_openclaw import ChannelClient, WindowContent
from datetime import datetime, timezone

client = ChannelClient("http://localhost:8080")
content: WindowContent = client.get_context("home-agent", since=0)

if not content.agent_has_association:
    # Agent not yet wired to any Qhorus channels
    pass
elif content.messages:
    for msg in content.messages:
        print(f"{msg.sender_id} [{msg.message_type}]: {msg.content}")
```

## Error handling

`get_context` raises `httpx.HTTPStatusError` on non-2xx and
`httpx.TimeoutException` on timeout. Callers decide whether to fail open.

```python
import httpx

try:
    content = client.get_context("home-agent", since=0)
except (httpx.HTTPStatusError, httpx.TimeoutException):
    content = None  # proceed without context
```

## Design

See `../docs/specs/openclaw-integration.md` §ChannelContextWindow for the full
design, failure modes, and reliability contract.
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/openclaw add python/README.md
git -C /Users/mdproctor/claude/casehub/openclaw commit -m "docs(python): update README for Epic 5 — clarify TypeScript plugin vs Python library

Closes #5"
```

---

## Post-Implementation Checklist

After all tasks complete, verify:

- [ ] `npm --prefix plugin test` — all 22 TypeScript tests pass
- [ ] `npm --prefix plugin run build` — TypeScript compiles with no errors, `dist/` present
- [ ] `python -m pytest python/tests/ -v` — all 9 Python tests pass
- [ ] No `context_hook.py` in the Python package
- [ ] `plugin/dist/index.js` exists (for OpenClaw to load)
- [ ] All commits reference `#5`
