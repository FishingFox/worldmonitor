# Codebase Concerns

**Analysis Date:** 2026-02-21

## Tech Debt

### Monolithic App Component

**Files:** `src/App.ts`

**Issue:** Single component file at 4,570 lines — massive god-class managing all lifecycle, panels, event handlers, URL state, idle detection, animations, and 50+ imported services.

**Impact:**
- Difficult to debug panel interactions
- Hard to test individual features in isolation
- Painful to add new panels or modify existing ones
- Risk of unexpected side effects when changing any behavior

**Fix approach:**
1. Extract panel system into `PanelManager` class
2. Split lifecycle hooks into separate concerns (idle detection → `IdleDetector`, URL sync → `UrlStateManager`, etc.)
3. Create `EventBus` for cross-panel communication instead of direct prop drilling
4. Refactor panel instantiation from hardcoded list into registry pattern

### API Catch-All Handler Size

**Files:** `api/[[...path]].js` (10,446 lines generated)

**Issue:** The compiled catch-all API handler is massive (~10k lines) because it contains all 17 domain handlers in a single file.

**Impact:**
- Slower cold starts for serverless functions
- Entire API gateway re-deploys on any domain handler change
- Difficult to debug routing logic when file is this large
- No granular code splitting opportunities

**Fix approach:**
1. Leverage Vercel's dynamic route imports to create per-domain edge functions
2. Move to pattern: `/api/[domain]/route.ts` instead of catch-all
3. Share common utilities (CORS, auth, error mapping) via separate modules
4. Measure cold start improvement after split

### Type Casting Proliferation

**Files:** Multiple — `src/workers/ml.worker.ts`, `src/services/weather.ts`, `src/services/runtime.ts`, `src/components/DeckGLMap.ts`, `src/App.ts` (+2 more)

**Pattern:** 289 occurrences of `as any`, `as unknown as`, type assertions on geometry/API responses

**Issue:** Type casts bypass TypeScript compiler safety; they hide:
- API response schema changes that should fail at compile time
- Data shape mismatches in worker messages
- Browser API assumptions (e.g., `globalThis.ort` in ML worker)

**Examples:**
```typescript
const ort = (globalThis as any).ort;  // ML worker assumes ONNX global
const coords = geometry.coordinates as unknown as number[][][];  // Weather API coord shape
const obj = info.object as any;  // DeckGL picking result shape
```

**Fix approach:**
1. Create proper type definitions for external APIs (weather geometry, ONNX runtime interface)
2. Use type guards instead of assertions: `if (isValidCoordinates(coords)) { ... }`
3. Generate types from API responses using `json-schema-to-typescript` if possible
4. Enable `noImplicitAny` rule in tsconfig if not already set

## Known Bugs

### Empty Catch Blocks

**Files:** `src/App.ts:2847`, `src/services/trending-keywords.ts:168`

**Pattern:** Silent error suppression
```typescript
try { el.webkitRequestFullscreen(); } catch {}  // Swallows fullscreen errors
} catch {}  // Trending keywords fetch failure hidden
```

**Issue:**
- Errors are invisible to monitoring/logging
- Difficult to diagnose why features silently fail
- User has no feedback that an operation failed

**Fix approach:**
1. Replace with proper error logging: `catch (err) { console.warn('[component] Failed to...', err); }`
2. Add user-facing error handling for critical operations
3. Set up error boundary for component failures
4. Use Sentry for production error tracking

### Unhandled Promise Rejections

**Count:** 167 stub returns (`return null`, `return []`, `return {}`) across 60 files

**Issue:** Service functions return empty collections on failure without:
- Throwing errors that callers can catch
- Logging what failed
- Distinguishing "no data" from "error"

**Example patterns:**
- Fetch fails → returns `null` or `[]` → UI treats as "no data available"
- Component continues as if request succeeded
- No way for parent to know a required data source failed

**Fix approach:**
1. Create result type wrapper: `type Result<T> = { ok: true; data: T } | { ok: false; error: string }`
2. Return explicit errors from all fetch functions
3. Add telemetry to track which data sources are frequently failing
4. UI should handle error state distinctly from empty state

## Security Considerations

### Missing API Key Validation in Development

**Files:** `api/_api-key.js`, `api/[domain]/v1/[rpc].ts`

**Issue:** API key is optional for certain origins/environments, but:
- No clear documentation on which keys are required where
- Falling back to unauthenticated access on localhost/desktop could leak during testing
- No rate limiting visible in middleware

**Current behavior (from code):**
```typescript
const keyCheck = validateApiKey(request);
if (keyCheck.required && !keyCheck.valid) {  // Only enforced if "required"
```

**Recommendations:**
1. Add API key requirement rules documentation
2. Implement per-origin rate limiting (use Upstash Redis rate limiter)
3. Log all API calls (especially unauthenticated ones) for audit trail
4. Add header validation for all external integrations

### Middleware Bot Detection Bypass Risk

**Files:** `middleware.ts`

**Issue:** Bot detection regex may be bypassed:
- Checks for `bot|crawl|spider|...` but sophisticated scrapers can spoof user agent
- Checks for `ua.length < 10` but easily bypassed
- No CAPTCHA or rate limiting fallback

**Risk:** Scraping of OSINT data, DDoS amplification, unauthorized bulk data extraction

**Recommendations:**
1. Add rate limiting per IP (Vercel Analytics can provide this)
2. Implement CAPTCHA for suspicious patterns
3. Add geo-blocking for high-risk origins if applicable
4. Log all rejected requests to detect attack patterns

### Secrets in Environment Configuration

**Files:** Entire codebase reads from `process.env` and `import.meta.env`

**Issue:** No evidence of secret rotation, key expiration, or access audit logs

**Current integrations requiring secrets:**
- Convex (backend)
- PostHog (analytics)
- Sentry (monitoring)
- ONNX Runtime (local ML models)
- Various feed APIs

**Recommendations:**
1. Document all required environment variables in `.env.example` (already present)
2. Implement secret rotation schedule for API keys
3. Add audit logging for secret access
4. Use managed secrets service (e.g., Vercel Secrets or AWS Secrets Manager)

## Performance Bottlenecks

### Large Component Rendering Chain

**Files:** `src/components/DeckGLMap.ts` (3,871 lines), `src/components/Map.ts` (3,496 lines)

**Issue:** Heavy deck.gl/maplibre visualization with 50+ layer types, clustering, filtering, and custom overlays. No evidence of memoization or rendering optimization.

**Problem:** On low-end devices (mobile, older laptops), re-rendering 50 layers on pan/zoom causes frame drops.

**Fix approach:**
1. Implement strict layer memoization using `useMemo`/`useCallback` patterns (if transitioning to a framework)
2. Use deck.gl's layer update diffing more aggressively
3. Lazy-load layers off-screen (only render visible viewport + margin)
4. Profile with DevTools to identify which layers are slowest
5. Consider pre-rendering static layers to canvas textures

### Unbounded Cache Growth

**Files:** `src/services/persistent-cache.ts`

**Issue:** Cache stores data with `updatedAt` timestamp but no visible cache eviction policy.

**Impact:**
- IndexedDB grows indefinitely
- Desktop sidecar cache grows indefinitely
- Over months, could fill disk/quota

**Current code:**
```typescript
const payload: CacheEnvelope<T> = { key, data, updatedAt: Date.now() };
// No TTL, no size limit, no expiration
```

**Fix approach:**
1. Add cache entry TTL (e.g., 7 days for feeds, 30 days for slower-changing data)
2. Implement LRU eviction when quota is exceeded
3. Add background cleanup task on app startup
4. Monitor cache size and log warnings when approaching limits

### Promise.all Avalanche

**Files:** `src/services/` (15+ places using `Promise.all`)

**Examples:**
- `fetchMultipleStocks` → Promise.all on all symbols
- `prediction/index.ts:257` → Promise.all on 20+ tags
- `economic/index.ts:114` → Promise.all on 20+ FRED series

**Issue:** If any single fetch times out or fails, entire batch fails. No backpressure/retry logic visible.

**Impact:**
- One slow API blocks entire dashboard load
- No timeout protection per request
- Cascading failures when one data source is degraded

**Fix approach:**
1. Switch critical batches to `Promise.allSettled()` (already done in some places)
2. Add per-request timeout: `Promise.race([fetch(...), timeout(5000)])`
3. Implement exponential backoff + jitter for retries
4. Add observable metrics for which requests are timing out

## Fragile Areas

### Military/Geopolitical Data Formatting

**Files:** `src/services/military-flights.ts` (561 lines), `src/services/military-vessels.ts` (642 lines), `src/services/military-surge.ts` (990 lines)

**Why fragile:**
- Parsing unstructured text feeds (OpenSky, MMSI formats)
- MMSI parsing: `MIDXXXXXX` format with conditional logic (lines 74+)
- Flight callsign matching against hardcoded military aircraft patterns
- Vessel flag state classification based on regex

**Safe modification:**
1. Create test cases for edge cases: unusual MMSI formats, spoofed callsigns
2. Add validation at ingestion boundary — log and skip malformed entries
3. Document pattern assumptions explicitly in code comments
4. Consider creating separate `MilitaryDataValidator` module

**Test coverage gaps:**
- No visible unit tests for MMSI parsing
- No tests for vessel flag classification accuracy
- No tests for callsign pattern edge cases

### Regional/Country Geometry Lookups

**Files:** `src/services/country-geometry.ts`, `src/services/geo-convergence.ts`

**Why fragile:**
- Coordinate-to-country matching is O(n) point-in-polygon for each signal
- Reverse geocoding uses external API with quota limits
- Country code mappings (ISO 3166) could be stale

**Safe modification:**
1. Add caching for reverse geocode results
2. Pre-load country geometries into fast spatial index (R-tree)
3. Add tolerance for coordinate precision issues (GPS noise)
4. Test boundary cases: points exactly on country borders

### State Synchronization Between Panels

**Files:** `src/App.ts` (panel state management), individual panels (`src/components/*.ts`)

**Why fragile:**
- Panels refresh independently on different schedules (REFRESH_INTERVALS)
- No visible state lock/transaction mechanism
- Panel A reads state while Panel B is updating — potential race condition
- Country brief page state separate from main map state

**Safe modification:**
1. Create `PanelStateManager` with explicit read/write locks
2. Batch updates: collect all changes, apply atomically
3. Add state version/epoch to detect stale reads
4. Test concurrent panel updates with deterministic delays

## Scaling Limits

### Data Volume vs. Real-Time Rendering

**Current capacity:**
- ~500 military flights rendered simultaneously
- ~2,000 AIS ships rendered simultaneously
- ~10,000 news items in memory (clustered to ~500 visual clusters)

**Limit:** Desktop GPU can render ~50k objects efficiently; beyond that, frame rate degrades

**Scaling path:**
1. Implement dynamic detail level based on zoom: show fewer objects when zoomed out
2. Use deck.gl's GPU-accelerated clustering more aggressively
3. Stream data in viewport-relative chunks instead of loading all at once
4. Consider tiled rendering (divide world into regions, load on demand)

### API Request Concurrency

**Current pattern:** Services make requests in parallel via `Promise.all`

**Limit:** Vercel Edge Function has ~30-50 concurrent downstream connections before timeouts

**Scaling path:**
1. Add request queue per domain handler (currently unbounded)
2. Implement upstream HTTP/2 connection reuse
3. Cache frequently-requested data at edge (already using persistent-cache, but no edge cache)
4. Consider moving slow aggregation queries to background jobs (Vercel Cron)

### Browser Memory & Storage

**Current usage:**
- IndexedDB cache (see unbounded growth concern above)
- 113 localStorage references across codebase
- Web Workers for ML (TensorFlow.js, ONNX runtime in parallel)

**Limit:** Mobile devices (iOS especially) can evict storage at any time, causing silent data loss

**Scaling path:**
1. Graceful degradation when storage is unavailable
2. Compress large objects before storing (currently no compression)
3. Implement versioned migrations for storage schema changes
4. Monitor actual storage usage with Sentry

## Dependencies at Risk

### onnxruntime-web

**Risk:** ONNX Runtime v1.23.2 — heavy JavaScript library bundled in app

**Impact:** Adds ~2MB to bundle (compressed); loads models at runtime

**Migration plan:**
1. Monitor for v2.x releases (breaking changes expected)
2. Evaluate alternatives: Transformers.js (already in deps) vs ONNX
3. Consider deferring ML features for mobile (already skips on low-end devices)

### YouTubei.js

**Risk:** YouTube reverse-engineering library; APIs may break without notice when YouTube changes

**Impact:** Live webcam embeds could fail

**Migration plan:**
1. Add fallback to iframe embeds (more stable)
2. Monitor YouTube changes in analytics
3. Have backup data source for critical streams

### deck.gl

**Risk:** Version 9.2.6 — active development; some performance regressions reported in 9.3.x

**Current:** Pinned to 9.2.6 (good)

**Recommendation:** Test 9.3/10.x when released; current version is stable for scaling needs

### PostHog (Analytics)

**Risk:** Heavy JavaScript library; can impact page load on slow connections

**Current:** Deferred initialization, privacy-first design (from code inspection)

**Recommendation:** Continue monitoring bundle impact; already using lazy loading

## Test Coverage Gaps

### Missing E2E Tests for Military Data

**Untested area:** Military flight/vessel tracking, MMSI parsing, callsign classification

**Files:** `src/services/military-flights.ts`, `src/services/military-vessels.ts`

**Risk:** Silent data corruption (misidentified vessel flags, incorrect flight classification)

**Fix:** Add E2E tests with mock military data feeds; verify parsing accuracy

### No Tests for Concurrent Panel Updates

**Untested area:** What happens when two panels refresh simultaneously and modify shared state?

**Risk:** Race conditions, stale state, duplicated work

**Fix:** Add stress test that throttles multiple panel updates and verifies consistency

### Missing Data Freshness Verification

**Untested area:** `src/services/data-freshness.ts` — verifies data is recent

**Files:** `src/services/data-freshness.ts`

**Risk:** Stale data served without warning if freshness check breaks

**Fix:** Add test scenarios for timeout/late data and verify UI shows warning

### API Route Security Tests

**Untested area:** CORS, API key validation, bot detection

**Files:** `middleware.ts`, `api/[domain]/v1/[rpc].ts`, `api/_api-key.js`

**Risk:** Regressions in security rules could expose data

**Fix:** Add tests for:
1. Requests with invalid origins are rejected
2. Requests without required API keys are rejected
3. Bot user agents are blocked
4. OPTIONS preflight works correctly

---

*Concerns audit: 2026-02-21*
