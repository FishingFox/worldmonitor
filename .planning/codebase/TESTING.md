# Testing Patterns

**Analysis Date:** 2026-02-21

## Test Framework

**Runner:**
- Node.js built-in `node:test` module (ESM-based)
- Playwright for E2E browser tests (`@playwright/test@^1.52.0`)

**Assertion Library:**
- Node built-in: `node:assert/strict`
- Playwright assertions: `expect()` API

**Run Commands:**
```bash
npm run test:data                    # Run all Node test files (tests/*.test.mjs)
npm run test:sidecar                 # Run sidecar/API tests (api/*.test.mjs)
npm run test:e2e:runtime             # E2E: desktop runtime routing guardrails
npm run test:e2e:full                # E2E: full variant visual + interaction tests
npm run test:e2e:tech                # E2E: tech variant tests
npm run test:e2e:finance             # E2E: finance variant tests
npm run test:e2e                     # Run all E2E suites in sequence
npm run test:e2e:visual:full         # E2E: golden screenshot comparisons (full variant)
npm run test:e2e:visual:update:full  # Update golden screenshots
```

## Test File Organization

**Location:**
- Node tests: `tests/*.test.mjs` (data validation, handler correctness)
- Sidecar tests: `api/*.test.mjs`, `api/youtube/*.test.mjs` (API endpoint validation)
- E2E tests: `e2e/*.spec.ts` (browser integration, visual regressions)

**Naming:**
- `.test.mjs` for Node test files
- `.spec.ts` for Playwright E2E tests
- Test harness files: `e2e/*-harness.spec.ts`

**Structure:**
```
tests/
├── server-handlers.test.mjs    # Handler correctness assertions
├── deploy-config.test.mjs      # Deployment configuration
├── gulf-fdi-data.test.mjs      # Data fixture validation
└── countries-geojson.test.mjs  # Geospatial data

api/
├── _cors.test.mjs              # CORS header validation
├── youtube/
│   └── embed.test.mjs          # YouTube embed API
└── ...

e2e/
├── runtime-fetch.spec.ts       # Desktop runtime routing
├── map-harness.spec.ts         # Map layer visualization
├── investments-panel.spec.ts    # Panel interaction
└── mobile-map-popup.spec.ts    # Mobile responsiveness
```

## Test Structure

**Suite Organization:**
```typescript
// Node test pattern (from tests/server-handlers.test.mjs)
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';

describe('getHumanitarianSummary handler', () => {
  const src = readSrc('server/worldmonitor/conflict/v1/get-humanitarian-summary.ts');

  it('returns undefined when country has no ISO3 mapping', () => {
    assert.match(src, /pattern/);
  });
});
```

```typescript
// Playwright pattern (from e2e/runtime-fetch.spec.ts)
import { expect, test } from '@playwright/test';

test.describe('desktop runtime routing guardrails', () => {
  test('detectDesktopRuntime covers packaged tauri hosts', async ({ page }) => {
    await page.goto('/tests/runtime-harness.html');
    const result = await page.evaluate(async () => {
      // Test logic in browser context
    });
    expect(result.tauriHost).toBe(true);
  });
});
```

**Patterns:**
- Setup: File reading, context initialization, mock setup
- Assertions: Direct assertions, pattern matching on source code, or browser state
- Teardown: Implicit cleanup via test isolation (no explicit cleanup in most tests)

## Mocking

**Framework:** Manual fetch stubbing and source code inspection

**Patterns:**
```typescript
// Fetch mocking (from e2e/runtime-fetch.spec.ts)
const calls: string[] = [];
window.fetch = (async (input: RequestInfo | URL) => {
  const url = typeof input === 'string' ? input : input instanceof URL ? input.toString() : input.url;
  calls.push(url);

  if (url.includes('127.0.0.1:46123/api/fred-data')) {
    return new Response(JSON.stringify({ error: 'missing local api key' }), { status: 500 });
  }
  // ... handler logic
});
```

```typescript
// Source code pattern matching (from tests/server-handlers.test.mjs)
const src = readFileSync(resolve(root, 'server/worldmonitor/conflict/v1/get-humanitarian-summary.ts'), 'utf-8');
assert.match(src, /if\s*\(\s*!iso3\s*\)\s*return\s+undefined/);
```

**What to Mock:**
- Upstream API responses via fetch stubbing
- Browser globals (`window.fetch`, `localStorage`)
- Time via `setInterval`/`setTimeout` controls
- Not mocked: actual service layer clients (use real sebuf clients in E2E)

**What NOT to Mock:**
- Service handler implementations (test with real handler code)
- Type definitions (assert on source pattern instead)
- Proto-generated clients (test real client behavior)
- DOM structure (inspect actual rendered output)

## Fixtures and Factories

**Test Data:**
```javascript
// Simple fixture pattern (from api/_cors.test.mjs)
function makeRequest(origin) {
  const headers = new Headers();
  if (origin !== null) {
    headers.set('origin', origin);
  }
  return new Request('https://worldmonitor.app/api/test', { headers });
}

// Usage in test
const req = makeRequest('https://evil.example.com');
```

```typescript
// Scenario setup (from e2e/map-harness.spec.ts)
type LayerSnapshot = { id: string; dataCount: number };
type VisualScenarioSummary = {
  id: string;
  variant: 'both' | 'full' | 'tech' | 'finance';
};

const EXPECTED_FULL_DECK_LAYERS = [
  'cables-layer', 'pipelines-layer', 'conflict-zones-layer', ...
];
```

**Location:**
- Test fixtures defined inline in test files (no separate fixtures directory)
- Shared test data: `tests/` directory for larger fixtures
- Scenario definitions inline in E2E tests using TypeScript interfaces

## Coverage

**Requirements:** Not enforced

**View Coverage:** No coverage reporting configured in CI/package.json

**Current state:** Tests focus on:
- Handler correctness (regression prevention after proto migrations)
- Source code pattern validation (proto field naming, imports)
- E2E browser behavior (visual regressions, desktop runtime routing)
- API endpoint validation (CORS, request/response contracts)

## Test Types

**Unit Tests:**
- Scope: Individual functions, pure logic, adapters
- Approach: Direct function calls with assertions
- Example: Deduplication algorithm testing (from `tests/server-handlers.test.mjs`)
  ```javascript
  it('removes near-duplicate headlines', () => {
    const headlines = [ /* ... */ ];
    const result = deduplicateHeadlines(headlines);
    assert.equal(result.length, 2);
  });
  ```
- Location: `tests/*.test.mjs` for data transformation tests

**Integration Tests:**
- Scope: Handler implementations + upstream API interactions
- Approach: Source code pattern matching to verify proto field usage, import patterns
- Example: Handler contract verification (from `tests/server-handlers.test.mjs`)
  ```javascript
  it('returns ISO-2 country_code per proto contract', () => {
    const src = readSrc('server/worldmonitor/conflict/v1/get-humanitarian-summary.ts');
    assert.match(src, /countryCode:\s*countryCode.*\.toUpperCase\(\)/);
  });
  ```
- Location: `tests/server-handlers.test.mjs`, `api/*.test.mjs`

**E2E Tests:**
- Framework: Playwright (`@playwright/test`)
- Scope: Full browser behavior, map interactions, desktop integration
- Approach: Browser context evaluation, DOM inspection, visual regression snapshots
- Examples:
  - `e2e/runtime-fetch.spec.ts`: Desktop runtime routing fallback logic
  - `e2e/map-harness.spec.ts`: Map layer data, visual golden snapshots
  - `e2e/investments-panel.spec.ts`: Panel interaction workflows
- Visual regression: Screenshots stored in `e2e/*-snapshots/` directory
  - Update via: `npm run test:e2e:visual:update:full`
- Location: `e2e/*.spec.ts`

## Common Patterns

**Async Testing:**
```typescript
// Playwright async pattern
test('async operation', async ({ page }) => {
  const result = await page.evaluate(async () => {
    const runtime = await import('/src/services/runtime.ts');
    return await runtime.detectDesktopRuntime({ /* options */ });
  });

  expect(result).toBe(true);
});
```

```javascript
// Node async pattern
test('async handler', async () => {
  const response = await handler(makeRequest());
  assert.equal(response.status, 200);
});
```

**Error Testing:**
```javascript
// HTTP error codes
test('returns 400 for invalid input', async () => {
  const response = await handler(makeRequest('?videoId=bad'));
  assert.equal(response.status, 400);
});
```

```javascript
// Exception handling
test('gracefully handles missing config', () => {
  const result = deduplicateHeadlines([]); // No exception, returns []
  assert.equal(result.length, 0);
});
```

**Source Code Validation:**
```javascript
// Pattern matching for correctness assertions
it('uses renamed conflict-event proto fields', () => {
  assert.match(src, /conflictEventsTotal/);
  assert.match(src, /conflictPoliticalViolenceEvents/);
  assert.doesNotMatch(src, /populationAffected/);
});
```

**Browser State:**
```typescript
// Window global inspection
const result = await page.evaluate(() => {
  return {
    tauriHost: detectDesktopRuntime({ /* ... */ }),
    variant: window.__APP_VARIANT__,
    ready: window.__mapHarness?.ready,
  };
});
```

---

*Testing analysis: 2026-02-21*
