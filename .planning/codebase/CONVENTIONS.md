# Coding Conventions

**Analysis Date:** 2026-02-21

## Naming Patterns

**Files:**
- kebab-case for file names: `circuit-breaker.ts`, `theme-manager.ts`, `runtime-config.ts`
- Service modules use domain names: `conflict/index.ts`, `military/index.ts`, `economic/index.ts`
- Shared utilities in `src/utils/`: `sanitize.ts`, `dom-utils.ts`, `export.ts`
- API handler files use action verbs: `list-acled-events.ts`, `get-humanitarian-summary.ts`, `get-aircraft-details.ts`

**Functions:**
- camelCase for function names: `createCircuitBreaker`, `fetchCableActivity`, `mapProtoEventType`, `sanitizeUrl`
- Async functions prefixed with `fetch`, `load`, or domain verb: `fetchAcledConflicts`, `loadDesktopSecrets`, `listAcledEvents`
- Private helpers prefixed with underscore: `_ctx` (unused context parameters)
- Exported public functions documented with JSDoc blocks

**Variables:**
- camelCase for local variables and constants: `breakerDataState`, `cachedFallback`, `eventType`
- UPPER_CASE for module-level constants: `DEFAULT_MAX_FAILURES`, `DEFAULT_COOLDOWN_MS`, `HTML_ESCAPE_MAP`, `ACLED_API_URL`
- Destructured imports preserve original names
- Type guard predicates use `is` prefix pattern: `isDesktopOfflineMode`, `isAllowedProtocol`, `Number.isFinite`

**Types:**
- PascalCase for interface and type names: `CircuitBreakerOptions`, `BreakerDataState`, `ConflictEvent`, `CacheEntry`
- Discriminated unions use string literal types: `'live' | 'cached' | 'unavailable'` for modes
- Type imports explicitly use `type` keyword: `import type { ConflictEvent } from '@/types'`
- Proto-generated types prefixed with `Proto` when aliased for clarity: `type ProtoAcledEvent`, `type ProtoUcdpEvent`
- Exported type aliases for re-export patterns: `export type { ThreatClassification, ThreatLevel, EventCategory }`

## Code Style

**Formatting:**
- No enforced Prettier/ESLint configuration in repo
- TypeScript strict mode enabled in `tsconfig.json`
- 2-space indentation (inferred from package.json and code samples)
- Line breaks after imports, before function definitions
- Destructuring used for multiple related imports

**Linting:**
- TypeScript strict compiler options:
  - `strict: true`
  - `noUnusedLocals: true`
  - `noUnusedParameters: true`
  - `noFallthroughCasesInSwitch: true`
  - `noUncheckedIndexedAccess: true`
- Unused context parameters marked with underscore: `_ctx: ServerContext`

## Import Organization

**Order:**
1. Node built-ins: `import { strict as assert } from 'node:assert';`
2. External packages: `import { createCircuitBreaker } from '@/utils';` (alphabetically by import name)
3. Type imports (grouped at top): `import type { ConflictEvent } from '@/types';`
4. Relative imports (domain-specific): `import { listAcledEvents } from './list-acled-events';`
5. Re-exports after declarations: `export { haversineDistanceKm };`

**Path Aliases:**
- `@/*` resolves to `src/*` per `tsconfig.json` baseUrl config
- Used consistently across all source files: `@/services`, `@/types`, `@/utils`, `@/config`, `@/generated`
- Generated code imports from `@/generated/client/...` and `@/generated/server/...`

## Error Handling

**Patterns:**
- Try-catch blocks with implicit error absorption in service handlers — no error logging on upstream failures (established pattern per "2F-01 pattern")
- Example: `try { ... } catch { return []; }` for graceful degradation
- Missing configuration returns empty/null values: `if (!token) return [];`
- Validation returns early: `if (!iso3) return undefined;`
- Frontend fetch errors logged to Sentry when configured: `Sentry.init()` in `main.ts`
- AbortSignal timeouts on fetch calls: `signal: AbortSignal.timeout(15000)`

**Recovery:**
- Circuit breaker pattern for resilience: retry after cooldown with fallback to cached data
- Cached fallback before returning defaults: `this.getCachedOrDefault(defaultValue)`
- No cascade error propagation — each layer handles independently

## Logging

**Framework:** Native console API (no logger framework)

**Patterns:**
- Use `console.log` for informational messages with context prefix: `console.log('[App] Variant check: stored="..."')`
- Use `console.warn` for degraded operations: `console.warn('[Storage] Snapshot cleanup failed:', e)`
- Use `console.error` for unhandled failures: `console.error('[App] PizzINT load failed:', error)`
- Use `console.info` for debug-level output: conditional via `? console.warn : console.info`
- Prefix all logs with `[ServiceName]` bracket notation for context: `[Beta]`, `[PWA]`, `[Circuit Breaker Name]`
- Debug helpers exposed on window globally: `window.geoDebug = { inject, cells, count }`

## Comments

**When to Comment:**
- JSDoc blocks for exported public functions and types — see `circular-breaker.ts` for pattern
- Inline comments for non-obvious algorithmic logic: e.g., deduplication similarity threshold explanation
- Section dividers for major logical blocks: `// ---- Client + Circuit Breakers ----`
- TODO/FIXME comments acceptable (not enforced to remove)

**JSDoc/TSDoc:**
- Block-level documentation for interfaces and functions with parameters/returns
- Example format from `circuit-breaker.ts`:
  ```typescript
  export class CircuitBreaker<T> {
    // Field documentation via inline comments above properties
    // Parameter types documented in constructor JSDoc
  }
  ```
- No strict JSDoc enforcement — present when added, not required

## Function Design

**Size:**
- Helpers extracted to module-level functions: `mapProtoEventType`, `toConflictEvent` as separate functions before main handler
- Single responsibility pattern: each handler file contains 1-2 exported async functions
- Adapter functions isolated: `toConflictEvent`, `toUcdpGeoEvent`, `toRiskScoresResponse`

**Parameters:**
- Context parameter pattern in handlers: `async function handler(_ctx: ServerContext, req: Request): Promise<Response>`
- Options objects for configuration: `CircuitBreakerOptions`, `BreakerDataState`
- Request/Response types from generated proto interfaces
- Destructuring for multiple params avoided — prefer single options object

**Return Values:**
- Async handlers return proto-generated response types: `Promise<ListAcledEventsResponse>`
- Service functions return domain types: `Promise<ConflictData>`
- Nullable returns explicit: `T | null`, `T | undefined`
- Empty collections preferred to null: `return []; return {};` for degradation

## Module Design

**Exports:**
- Named exports for utilities: `export function createCircuitBreaker(...)`
- Type exports separate: `export type BreakerDataMode = 'live' | 'cached' | ...`
- Single default export for handlers: `export const conflictHandler: ConflictServiceHandler = { ... }`
- Mixed exports in index files: re-export helper types alongside service implementations

**Barrel Files:**
- Service domain barrels: `src/services/{domain}/index.ts` exports all types and functions
- Example: `src/services/conflict/index.ts` exports `ConflictEvent`, `ConflictData`, client, adapters
- Prevents scattered imports — import from domain barrel, not internal files

**File Structure by Layer:**
- `server/worldmonitor/{domain}/v1/handler.ts`: Handler implementation
- `server/worldmonitor/{domain}/v1/{operation}.ts`: RPC implementation files
- `src/services/{domain}/index.ts`: Client wrapper, adapters, exported types
- `src/types/index.ts`: Shared domain types

---

*Convention analysis: 2026-02-21*
