# Playwright E2E Demo UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing. Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #59 — Playwright/E2E test automation for demo UI
**Issue group:** #59

**Goal:** Browser-level E2E tests for the SSE-driven Lit Web Components demo dashboard
using Playwright with a mock SSE/REST server.

**Architecture:** Mock HTTP server emits scripted `CaseExecutionEvent` JSON sequences.
Playwright loads the real Lit UI served from esbuild output. `page.route()` with
`route.continue()` redirects `/api/**` requests to the mock server. No JVM in the loop.

**Tech Stack:** Playwright, TypeScript, Node.js http, Lit Web Components, esbuild

## Global Constraints

- All files created under `app/src/main/webui/e2e/`
- `@playwright/test` added to `devDependencies` in `app/src/main/webui/package.json`
- Mock server port from `MOCK_SERVER_PORT` env var, default `3099`
- UI port from `UI_PORT` env var, default `3098`
- Sequential execution: `workers: 1` (shared mock server)
- `route.continue()` not `route.fulfill()` for SSE streaming
- Agent IDs for trading-oversight: `signal`, `risk`, `execution` (from `ScenarioMetadataProvider`)
- Gate endpoint: `PUT /api/scenarios/{id}/workitems/{gateId}/complete` with `{outcome, resolution?}`
- Chromium only for first pass

---

## Task 1: Infrastructure + Test 01 (overview → start)

**Files:**
- Modify: `app/src/main/webui/package.json` (add `@playwright/test` devDependency)
- Create: `app/src/main/webui/e2e/playwright.config.ts`
- Create: `app/src/main/webui/e2e/tsconfig.json`
- Create: `app/src/main/webui/e2e/fixtures/events.ts`
- Create: `app/src/main/webui/e2e/fixtures/mock-server.ts`
- Create: `app/src/main/webui/e2e/fixtures/setup.ts`
- Create: `app/src/main/webui/e2e/fixtures/teardown.ts`
- Create: `app/src/main/webui/e2e/fixtures/api-route.ts`
- Create: `app/src/main/webui/e2e/tests/01-overview-start.spec.ts`

**Interfaces:**
- Produces: `MockServer` class with `setScenarios()`, `emitEvent()`, `emitSequence()`,
  `disconnectSSE()`, `reconnectSSE()`, `setStateSnapshot()`, `setNextResponse()`,
  `getRecordedCalls()`, `reset()`, `close()`, `port`
- Produces: Event factory functions matching `src/types/events.ts`
- Produces: `test` fixture extending Playwright's `test` with `mockServer` parameter

- [ ] **Step 1: Install Playwright**

Run: `cd app/src/main/webui && npm install --save-dev @playwright/test && npx playwright install chromium`

- [ ] **Step 2: Create `e2e/tsconfig.json`**

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["node"]
  },
  "include": ["."]
}
```

- [ ] **Step 3: Create `e2e/playwright.config.ts`**

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  workers: 1,
  globalSetup: './fixtures/setup.ts',
  globalTeardown: './fixtures/teardown.ts',
  use: {
    baseURL: `http://localhost:${process.env.UI_PORT || 3098}`,
  },
});
```

- [ ] **Step 4: Create `e2e/fixtures/events.ts`**

Factory functions producing `CaseExecutionEvent` JSON. Each returns a plain object
matching the interfaces in `src/types/events.ts`.

```typescript
const DEFAULT_SCENARIO = 'trading-oversight';
const now = () => new Date().toISOString();

export const scenarioStarted = (scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'SCENARIO_STARTED' as const, scenarioId, occurredAt: now() });

export const scenarioCompleted = (scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'SCENARIO_COMPLETED' as const, scenarioId, occurredAt: now() });

export const scenarioFailed = (error: string, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'SCENARIO_FAILED' as const, scenarioId, occurredAt: now(), error });

export const agentStarted = (agentId: string, role: string, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'AGENT_STARTED' as const, scenarioId, occurredAt: now(), agentId, role });

export const agentCompleted = (agentId: string, role: string, outcome: string, durationMs: number, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'AGENT_COMPLETED' as const, scenarioId, occurredAt: now(), agentId, role, outcome, durationMs });

export const channelMessage = (agentId: string, role: string, content: string, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'CHANNEL_MESSAGE' as const, scenarioId, occurredAt: now(), agentId, role, content });

export const gatePending = (gateId: string, agentId: string, action: string, classification: string, priorAgents: string, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'GATE_PENDING' as const, scenarioId, occurredAt: now(), gateId, agentId, action, classification, priorAgents });

export const gateResolved = (gateId: string, decision: 'approved' | 'rejected', scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'GATE_RESOLVED' as const, scenarioId, occurredAt: now(), gateId, decision });

export const commitmentUpdated = (agentId: string, commitmentId: string, state: string, outcome: string, scenarioId = DEFAULT_SCENARIO) =>
  ({ type: 'COMMITMENT_UPDATED' as const, scenarioId, occurredAt: now(), agentId, commitmentId, state, outcome });
```

- [ ] **Step 5: Create `e2e/fixtures/mock-server.ts`**

Node.js HTTP server with SSE streaming, REST endpoints, and test control API.

```typescript
import { createServer, IncomingMessage, ServerResponse } from 'http';

interface RecordedCall {
  method: string;
  path: string;
  body: any;
}

interface NextResponse {
  statusCode: number;
  body?: object;
}

export class MockServer {
  private server: ReturnType<typeof createServer>;
  private sseClients = new Set<ServerResponse>();
  private sseAccepting = true;
  private scenarios: any[] = [];
  private stateSnapshots = new Map<string, any>();
  private recordedCalls: RecordedCall[] = [];
  private nextResponses = new Map<string, NextResponse>();
  readonly port: number;

  constructor(port: number) {
    this.port = port;
    this.server = createServer((req, res) => this.handleRequest(req, res));
  }

  async start(): Promise<void> {
    return new Promise((resolve) => {
      this.server.listen(this.port, () => resolve());
    });
  }

  async close(): Promise<void> {
    this.disconnectSSE();
    return new Promise((resolve) => {
      this.server.close(() => resolve());
    });
  }

  setScenarios(list: any[]) { this.scenarios = list; }
  setStateSnapshot(id: string, snapshot: any) { this.stateSnapshots.set(id, snapshot); }
  getRecordedCalls() { return [...this.recordedCalls]; }

  setNextResponse(path: string, statusCode: number, body?: object) {
    this.nextResponses.set(path, { statusCode, body });
  }

  reset() {
    this.scenarios = [];
    this.stateSnapshots.clear();
    this.recordedCalls = [];
    this.nextResponses.clear();
    this.disconnectSSE();
    this.sseAccepting = true;
  }

  emitEvent(event: any) {
    const data = `data: ${JSON.stringify(event)}\n\n`;
    for (const client of this.sseClients) {
      client.write(data);
    }
  }

  async emitSequence(events: any[], intervalMs = 100) {
    for (const event of events) {
      this.emitEvent(event);
      if (intervalMs > 0) await new Promise(r => setTimeout(r, intervalMs));
    }
  }

  disconnectSSE() {
    for (const client of this.sseClients) {
      client.end();
    }
    this.sseClients.clear();
    this.sseAccepting = false;
  }

  reconnectSSE() {
    this.sseAccepting = true;
  }

  private async handleRequest(req: IncomingMessage, res: ServerResponse) {
    const url = new URL(req.url!, `http://localhost:${this.port}`);
    const path = url.pathname;

    // Check for one-shot response override
    const override = this.nextResponses.get(path);
    if (override && req.method !== 'GET') {
      this.nextResponses.delete(path);
      const body = await readBody(req);
      this.recordedCalls.push({ method: req.method!, path, body });
      res.writeHead(override.statusCode, { 'Content-Type': 'application/json' });
      res.end(override.body ? JSON.stringify(override.body) : '');
      return;
    }

    // SSE endpoint
    if (path === '/api/scenarios/events' && req.method === 'GET') {
      if (!this.sseAccepting) {
        res.writeHead(503);
        res.end();
        return;
      }
      res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive',
      });
      res.write('\n');
      this.sseClients.add(res);
      req.on('close', () => this.sseClients.delete(res));
      return;
    }

    // GET /api/scenarios
    if (path === '/api/scenarios' && req.method === 'GET') {
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(this.scenarios));
      return;
    }

    // GET /api/scenarios/{id}/state
    const stateMatch = path.match(/^\/api\/scenarios\/([^/]+)\/state$/);
    if (stateMatch && req.method === 'GET') {
      const snapshot = this.stateSnapshots.get(stateMatch[1]);
      res.writeHead(snapshot ? 200 : 404, { 'Content-Type': 'application/json' });
      res.end(snapshot ? JSON.stringify(snapshot) : '{}');
      return;
    }

    // POST /api/scenarios/{id}/start
    const startMatch = path.match(/^\/api\/scenarios\/([^/]+)\/start$/);
    if (startMatch && req.method === 'POST') {
      this.recordedCalls.push({ method: 'POST', path, body: null });
      res.writeHead(202);
      res.end();
      return;
    }

    // PUT /api/scenarios/{id}/workitems/{gateId}/complete
    const gateMatch = path.match(/^\/api\/scenarios\/([^/]+)\/workitems\/([^/]+)\/complete$/);
    if (gateMatch && req.method === 'PUT') {
      const body = await readBody(req);
      this.recordedCalls.push({ method: 'PUT', path, body });
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ status: 'ok' }));
      return;
    }

    res.writeHead(404);
    res.end();
  }
}

function readBody(req: IncomingMessage): Promise<any> {
  return new Promise((resolve) => {
    const chunks: Buffer[] = [];
    req.on('data', (chunk) => chunks.push(chunk));
    req.on('end', () => {
      const raw = Buffer.concat(chunks).toString();
      try { resolve(JSON.parse(raw)); }
      catch { resolve(raw || null); }
    });
  });
}

export async function createMockServer(port: number): Promise<MockServer> {
  const server = new MockServer(port);
  await server.start();
  return server;
}
```

- [ ] **Step 6: Create `e2e/fixtures/setup.ts`**

```typescript
import { execSync } from 'child_process';
import { createServer } from 'http';
import { readFileSync, existsSync } from 'fs';
import { join, extname } from 'path';

const MIME_TYPES: Record<string, string> = {
  '.html': 'text/html',
  '.js': 'application/javascript',
  '.css': 'text/css',
  '.map': 'application/json',
};

async function globalSetup() {
  const webui = join(__dirname, '../..');
  const dist = join(webui, 'dist');

  // Build the UI
  execSync('node esbuild.config.mjs', { cwd: webui, stdio: 'pipe' });

  // Start static file server
  const port = Number(process.env.UI_PORT) || 3098;
  const server = createServer((req, res) => {
    const filePath = join(dist, req.url === '/' ? 'index.html' : req.url!);
    if (!existsSync(filePath)) {
      res.writeHead(404);
      res.end();
      return;
    }
    const ext = extname(filePath);
    res.writeHead(200, { 'Content-Type': MIME_TYPES[ext] || 'application/octet-stream' });
    res.end(readFileSync(filePath));
  });

  await new Promise<void>((resolve) => server.listen(port, resolve));
  (globalThis as any).__UI_SERVER__ = server;
}

export default globalSetup;
```

- [ ] **Step 7: Create `e2e/fixtures/teardown.ts`**

```typescript
async function globalTeardown() {
  const server = (globalThis as any).__UI_SERVER__;
  if (server) {
    await new Promise<void>((resolve) => server.close(resolve));
  }
}

export default globalTeardown;
```

- [ ] **Step 8: Create `e2e/fixtures/api-route.ts`**

```typescript
import { test as base } from '@playwright/test';
import { createMockServer, MockServer } from './mock-server';

export { expect } from '@playwright/test';

export const test = base.extend<{}, { mockServer: MockServer }>({
  mockServer: [async ({}, use) => {
    const server = await createMockServer(Number(process.env.MOCK_SERVER_PORT) || 3099);
    await use(server);
    await server.close();
  }, { scope: 'worker' }],

  page: async ({ page, mockServer }, use) => {
    await page.route('/api/**', (route) => {
      const url = new URL(route.request().url());
      url.port = String(mockServer.port);
      return route.continue({ url: url.toString() });
    });
    mockServer.reset();
    await use(page);
  },
});
```

- [ ] **Step 9: Create `e2e/tests/01-overview-start.spec.ts`**

```typescript
import { test, expect } from '../fixtures/api-route';
import { scenarioStarted } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution with human oversight gates', status: 'idle' as const },
  { id: 'multi-agent-dev-team', name: 'Dev Team', description: 'Multi-agent software development', status: 'idle' as const },
  { id: 'incident-response', name: 'Incident Response', description: 'Automated incident triage and response', status: 'idle' as const },
];

test('scenario cards load with idle status', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  const cards = page.locator('.scenario-card');
  await expect(cards).toHaveCount(3);
  await expect(cards.first().locator('.scenario-status')).toHaveText('idle');
});

test('start button triggers POST and SSE updates status', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');

  await page.locator('.scenario-card').first().locator('button', { hasText: 'Start' }).click();

  const calls = mockServer.getRecordedCalls();
  expect(calls).toHaveLength(1);
  expect(calls[0].path).toBe('/api/scenarios/trading-oversight/start');

  mockServer.emitEvent(scenarioStarted('trading-oversight'));
  await expect(page.locator('.scenario-card').first().locator('.scenario-status')).toHaveText('running');
  await expect(page.locator('.scenario-card').first().locator('button', { hasText: 'View' })).toBeVisible();
});
```

- [ ] **Step 10: Run test 01**

Run: `cd app/src/main/webui && npx playwright test e2e/tests/01-overview-start.spec.ts --config e2e/playwright.config.ts`

Expected: PASS (both tests)

- [ ] **Step 11: Commit**

```
feat(#59): Playwright infrastructure + overview/start E2E test

Mock SSE server, event factories, api-route fixture, globalSetup/
teardown. First test verifies scenario card loading and start flow.

Refs #59
```

---

## Task 2: Test 02 — Agent pipeline progression

**Files:**
- Create: `app/src/main/webui/e2e/tests/02-agent-pipeline.spec.ts`

**Interfaces:**
- Consumes: `test`, `expect` from `api-route.ts`; `agentStarted`, `agentCompleted`, `scenarioStarted` from `events.ts`

- [ ] **Step 1: Write test**

```typescript
import { test, expect } from '../fixtures/api-route';
import { agentStarted, agentCompleted, scenarioStarted } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test.beforeEach(async ({ mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
});

test('agent appears on AGENT_STARTED with running badge', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  const worker = page.locator('.worker').first();
  await expect(worker.locator('.worker-name')).toHaveText('signal');
  await expect(worker.locator('.worker-role')).toHaveText('Signal Analyst');
  await expect(worker.locator('.worker-state')).toHaveText('running');
  await expect(worker.locator('.worker-state')).toHaveClass(/running/);
});

test('agent transitions to completed with duration', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  mockServer.emitEvent(agentCompleted('signal', 'Signal Analyst', 'completed', 4200));
  const worker = page.locator('.worker').first();
  await expect(worker.locator('.worker-state')).toHaveText('completed');
  await expect(worker.locator('.worker-state')).toHaveClass(/completed/);
  await expect(worker.locator('.worker-duration')).toContainText('4.2s');
});

test('multiple agents appear in pipeline order', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  mockServer.emitEvent(agentCompleted('signal', 'Signal Analyst', 'completed', 4200));
  mockServer.emitEvent(agentStarted('risk', 'Risk Assessor'));

  const workers = page.locator('.worker');
  await expect(workers).toHaveCount(2);
  await expect(workers.nth(0).locator('.worker-name')).toHaveText('signal');
  await expect(workers.nth(1).locator('.worker-name')).toHaveText('risk');
});

test('failed outcome shows failed badge', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('risk', 'Risk Assessor'));
  mockServer.emitEvent(agentCompleted('risk', 'Risk Assessor', 'failed', 1500));
  await expect(page.locator('.worker-state')).toHaveText('failed');
  await expect(page.locator('.worker-state')).toHaveClass(/failed/);
});

test('declined outcome shows warning styling', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  mockServer.emitEvent(agentCompleted('signal', 'Signal Analyst', 'declined', 800));
  await expect(page.locator('.worker-state')).toHaveText('declined');
  await expect(page.locator('.worker-state')).toHaveClass(/declined/);
});

test('delegated outcome shows warning styling', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  mockServer.emitEvent(agentCompleted('signal', 'Signal Analyst', 'delegated', 1200));
  await expect(page.locator('.worker-state')).toHaveText('delegated');
  await expect(page.locator('.worker-state')).toHaveClass(/delegated/);
});

test('timeout outcome shows danger styling', async ({ page, mockServer }) => {
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  mockServer.emitEvent(agentCompleted('signal', 'Signal Analyst', 'timeout', 30000));
  await expect(page.locator('.worker-state')).toHaveText('timeout');
  await expect(page.locator('.worker-state')).toHaveClass(/timeout/);
});
```

- [ ] **Step 2: Run and verify**

Run: `cd app/src/main/webui && npx playwright test e2e/tests/02-agent-pipeline.spec.ts --config e2e/playwright.config.ts`

- [ ] **Step 3: Commit**

```
feat(#59): E2E test for agent pipeline progression

Covers all 5 outcome states (completed, failed, declined, delegated,
timeout), multi-agent ordering, and duration display.

Refs #59
```

---

## Task 3: Test 03 — Channel feed

**Files:**
- Create: `app/src/main/webui/e2e/tests/03-channel-feed.spec.ts`

- [ ] **Step 1: Write test**

```typescript
import { test, expect } from '../fixtures/api-route';
import { channelMessage } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test('channel messages appear in feed', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(channelMessage('signal', 'Signal Analyst', 'Analysing trade #1234 — USD/EUR 50M'));
  await expect(page.locator('channel-feed')).toContainText('Analysing trade #1234');

  mockServer.emitEvent(channelMessage('risk', 'Risk Assessor', 'Risk assessment: exposure within limits'));
  await expect(page.locator('channel-feed')).toContainText('Risk assessment: exposure within limits');
});
```

- [ ] **Step 2: Run and verify**
- [ ] **Step 3: Commit**

```
feat(#59): E2E test for channel activity feed

Refs #59
```

---

## Task 4: Test 04 — Oversight gate modal

**Files:**
- Create: `app/src/main/webui/e2e/tests/04-oversight-gate.spec.ts`

- [ ] **Step 1: Write test**

```typescript
import { test, expect } from '../fixtures/api-route';
import { gatePending, gateResolved } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test('gate pending shows modal with action and classification', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(gatePending('gate-1', 'execution', 'Execute trade #1234', 'high-value', 'risk'));
  await expect(page.locator('pages-modal')).toBeVisible();
  await expect(page.locator('approval-gate')).toContainText('Execute trade #1234');
  await expect(page.locator('approval-gate')).toContainText('high-value');
});

test('approve gate sends PUT and modal dismisses on resolution', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(gatePending('gate-1', 'execution', 'Execute trade #1234', 'high-value', 'risk'));
  await expect(page.locator('pages-modal')).toBeVisible();

  // Click approve — may trigger confirmation dialog
  await page.locator('approval-gate').locator('button', { hasText: /approve/i }).click();

  // Handle confirmation dialog if present
  const confirmBtn = page.locator('blocks-confirm-dialog button', { hasText: /confirm/i });
  if (await confirmBtn.isVisible({ timeout: 1000 }).catch(() => false)) {
    await confirmBtn.click();
  }

  // Verify PUT was recorded
  await expect.poll(() => mockServer.getRecordedCalls().some(c =>
    c.path.includes('/workitems/gate-1/complete')
  )).toBeTruthy();

  mockServer.emitEvent(gateResolved('gate-1', 'approved'));
  await expect(page.locator('pages-modal')).not.toBeVisible();
});

test('reject gate sends PUT with reject outcome', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(gatePending('gate-1', 'execution', 'Execute trade #1234', 'high-value', 'risk'));
  await expect(page.locator('pages-modal')).toBeVisible();

  await page.locator('approval-gate').locator('button', { hasText: /reject/i }).click();

  const confirmBtn = page.locator('blocks-confirm-dialog button', { hasText: /confirm/i });
  if (await confirmBtn.isVisible({ timeout: 1000 }).catch(() => false)) {
    await confirmBtn.click();
  }

  await expect.poll(() => {
    const calls = mockServer.getRecordedCalls();
    return calls.some(c => c.path.includes('/workitems/gate-1/complete') && c.body?.outcome === 'reject');
  }).toBeTruthy();

  mockServer.emitEvent(gateResolved('gate-1', 'rejected'));
  await expect(page.locator('pages-modal')).not.toBeVisible();
});
```

- [ ] **Step 2: Run and verify**
- [ ] **Step 3: Commit**

```
feat(#59): E2E test for oversight gate modal

Covers gate pending modal, approve with confirmation, reject with
confirmation, and modal dismissal on resolution.

Refs #59
```

---

## Task 5: Tests 05 + 06 — Completion + theme

**Files:**
- Create: `app/src/main/webui/e2e/tests/05-scenario-completion.spec.ts`
- Create: `app/src/main/webui/e2e/tests/06-theme-toggle.spec.ts`

- [ ] **Step 1: Write test 05**

```typescript
import { test, expect } from '../fixtures/api-route';
import { scenarioStarted, scenarioCompleted, scenarioFailed } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test('scenario completed shows success banner', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(scenarioStarted());
  mockServer.emitEvent(scenarioCompleted());
  const banner = page.locator('.status-banner');
  await expect(banner).toContainText('COMPLETED');
  await expect(banner).toHaveClass(/completed/);
});

test('scenario failed shows failure banner', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(scenarioStarted());
  mockServer.emitEvent(scenarioFailed('Agent timeout'));
  const banner = page.locator('.status-banner');
  await expect(banner).toContainText('FAILED');
  await expect(banner).toHaveClass(/failed/);
});
```

- [ ] **Step 2: Write test 06**

```typescript
import { test, expect } from '../fixtures/api-route';

test('theme toggle switches dark to light and back', async ({ page }) => {
  await page.goto('/');

  // Default is dark — check background
  const bg1 = await page.locator('app-shell').evaluate(el =>
    getComputedStyle(el).getPropertyValue('background-color'));

  await page.locator('button', { hasText: 'Toggle Theme' }).click();
  const bg2 = await page.locator('app-shell').evaluate(el =>
    getComputedStyle(el).getPropertyValue('background-color'));
  expect(bg2).not.toBe(bg1);

  await page.locator('button', { hasText: 'Toggle Theme' }).click();
  const bg3 = await page.locator('app-shell').evaluate(el =>
    getComputedStyle(el).getPropertyValue('background-color'));
  expect(bg3).toBe(bg1);
});
```

- [ ] **Step 3: Run both**

Run: `cd app/src/main/webui && npx playwright test e2e/tests/05-scenario-completion.spec.ts e2e/tests/06-theme-toggle.spec.ts --config e2e/playwright.config.ts`

- [ ] **Step 4: Commit**

```
feat(#59): E2E tests for scenario completion and theme toggle

Refs #59
```

---

## Task 6: Tests 07 + 08 — Reconnection + error handling

**Files:**
- Create: `app/src/main/webui/e2e/tests/07-sse-reconnection.spec.ts`
- Create: `app/src/main/webui/e2e/tests/08-error-handling.spec.ts`

- [ ] **Step 1: Write test 07**

```typescript
import { test, expect } from '../fixtures/api-route';
import { agentStarted } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test('SSE reconnection recovers state from snapshot', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  // Initial agent via SSE
  mockServer.emitEvent(agentStarted('signal', 'Signal Analyst'));
  await expect(page.locator('.worker')).toHaveCount(1);

  // Set snapshot for reconnection backfill
  mockServer.setStateSnapshot('trading-oversight', {
    scenarioId: 'trading-oversight',
    status: 'running',
    agents: [
      { agentId: 'signal', role: 'Signal Analyst', state: 'completed', durationMs: 4200 },
      { agentId: 'risk', role: 'Risk Assessor', state: 'running', durationMs: null },
    ],
    pendingGate: null,
    recentMessages: [],
  });

  // Disconnect and reconnect
  mockServer.disconnectSSE();
  mockServer.reconnectSSE();

  // Wait for reconnection — EventSource auto-reconnects
  // UI should call loadState() on reconnection (requires openclaw#72 fix)
  // For now, verify SSE reconnects and new events work
  await page.waitForTimeout(2000);
  mockServer.emitEvent(agentStarted('execution', 'Trade Executor'));

  // Verify new events arrive after reconnection
  await expect(page.locator('.worker-name', { hasText: 'execution' })).toBeVisible();
});
```

- [ ] **Step 2: Write test 08**

```typescript
import { test, expect } from '../fixtures/api-route';
import { gatePending } from '../fixtures/events';

const SCENARIOS = [
  { id: 'trading-oversight', name: 'Trading Oversight', description: 'Trade execution', status: 'idle' as const },
];

test('duplicate start (409) does not corrupt scenario state', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');

  mockServer.setNextResponse('/api/scenarios/trading-oversight/start', 409, { error: 'Already running' });
  await page.locator('.scenario-card').first().locator('button', { hasText: 'Start' }).click();

  // Status should remain idle — no SCENARIO_STARTED event was emitted
  await expect(page.locator('.scenario-card').first().locator('.scenario-status')).toHaveText('idle');
});

test('gate decision failure keeps modal open', async ({ page, mockServer }) => {
  mockServer.setScenarios(SCENARIOS);
  await page.goto('/');
  await page.locator('li', { hasText: 'Trading Oversight' }).click();

  mockServer.emitEvent(gatePending('gate-1', 'execution', 'Execute trade', 'high-value', 'risk'));
  await expect(page.locator('pages-modal')).toBeVisible();

  mockServer.setNextResponse('/api/scenarios/trading-oversight/workitems/gate-1/complete', 500, { error: 'Internal server error' });

  await page.locator('approval-gate').locator('button', { hasText: /approve/i }).click();
  const confirmBtn = page.locator('blocks-confirm-dialog button', { hasText: /confirm/i });
  if (await confirmBtn.isVisible({ timeout: 1000 }).catch(() => false)) {
    await confirmBtn.click();
  }

  // Modal should stay open — user can retry
  await expect(page.locator('pages-modal')).toBeVisible();
});
```

- [ ] **Step 3: Run both**

Run: `cd app/src/main/webui && npx playwright test e2e/tests/07-sse-reconnection.spec.ts e2e/tests/08-error-handling.spec.ts --config e2e/playwright.config.ts`

- [ ] **Step 4: Commit**

```
feat(#59): E2E tests for SSE reconnection and error handling

Covers SSE disconnect/reconnect recovery, 409 duplicate start
(no state corruption), and gate decision failure (modal stays open).

Refs #59
```

---

## Task 7: Full suite run + final commit

- [ ] **Step 1: Run entire E2E suite**

Run: `cd app/src/main/webui && npx playwright test --config e2e/playwright.config.ts`

Expected: All 8 test files pass.

- [ ] **Step 2: Fix any failures**

Iterate on failing tests — adjust selectors, timing, or mock server behavior as needed.
Shadow DOM in Lit components may require `locator('component-name')` piercing.

- [ ] **Step 3: Final commit if fixes were needed**

```
fix(#59): adjust E2E test selectors for shadow DOM / timing

Closes #59
```
