# Architecture

**Analysis Date:** 2026-02-21

## Pattern Overview

**Overall:** Domain-Driven Micro-Services with Protobuf-Based Integration (Sebuf)

**Key Characteristics:**
- **Domain Decomposition**: 18 independent intelligence domains (aviation, military, economic, climate, etc.), each with isolated business logic
- **Protocol Buffers + Code Generation**: Sebuf framework auto-generates service definitions, client/server interfaces, and route bindings from proto schemas
- **Layered Frontend**: Single-Page Application (SPA) with React/TypeScript consuming multiple domain services through typed gRPC-like clients
- **Edge Function Aggregation**: Vercel serverless functions orchestrate across domains and proxy external APIs
- **Variant Architecture**: Same codebase supports 3 different business variants (full/tech/finance) via environment configuration

## Layers

**Backend API Layer (Sebuf Services):**
- Purpose: Domain-specific RPC handlers implementing business logic for intelligence collection, processing, and analysis
- Location: `api/` (Vercel edge functions) and `server/worldmonitor/{domain}/v1/handler.ts`
- Contains: 18 domain handlers (seismology, wildfire, climate, prediction, displacement, aviation, research, unrest, conflict, maritime, cyber, economic, infrastructure, market, news, intelligence, military)
- Depends on: External data sources (APIs, RSS feeds, databases), proto-generated service definitions, shared utilities
- Used by: Frontend clients via HTTP POST to `/api/{domain}/v1/{rpc}`

**Frontend Application Layer:**
- Purpose: Real-time global intelligence dashboard UI with interactive map, panels, and data visualization
- Location: `src/`
- Contains: React components (`src/components/`), services (`src/services/`), configuration (`src/config/`), utilities (`src/utils/`), styling (`src/styles/`)
- Depends on: Domain service clients, MapLibre GL, Deck.gl, D3, i18n, local storage, service workers
- Used by: End users via web browser and Tauri desktop application

**Service Layer (Data Processing):**
- Purpose: Domain-agnostic and cross-cutting data processing (clustering, aggregation, ML, timeline generation)
- Location: `src/services/{name}.ts` (e.g., clustering.ts, country-instability.ts, geo-convergence.ts)
- Contains: Business logic for signal generation, data fusion, trend analysis, risk scoring
- Depends on: Raw data from domain APIs, domain service clients, utilities
- Used by: App.ts (main orchestrator), components, other services

**Client Generation Layer (Proto-Generated):**
- Purpose: Typed API client stubs and server routing auto-generated from proto definitions
- Location: `src/generated/client/worldmonitor/{domain}/v1/service_client.ts` and `src/generated/server/worldmonitor/{domain}/v1/service_server.ts`
- Contains: Service class definitions, type definitions, route descriptors
- Depends on: Protobuf compiler, sebuf codegen
- Used by: Handlers (server-side) and service consumers (client-side)

**Configuration Layer:**
- Purpose: Data-driven configuration for feeds, geographies, markets, military bases, etc.
- Location: `src/config/`
- Contains: Feed lists, geographic locations (bases, pipelines, cables), market symbols, intelligence hotspots
- Depends on: None (static data)
- Used by: Services, components, and API handlers

**Worker/Async Layer:**
- Purpose: Background processing and compute-intensive operations (ML inference, clustering)
- Location: `src/workers/` and `src/services/ml-worker.ts`, `src/services/analysis-worker.ts`
- Contains: Web Workers for ML model execution, signal analysis
- Depends on: ONNX Runtime, transformers.js, local computation
- Used by: Main application thread for non-blocking processing

**Desktop Runtime Layer (Tauri):**
- Purpose: Native desktop application wrapper with local sidecar backend
- Location: `src-tauri/`, `src/services/runtime.ts`, `src/services/runtime-config.ts`
- Contains: Rust backend (local API server), TypeScript bridge code, desktop-specific logic
- Depends on: Tauri framework, local Node.js sidecar, system APIs
- Used by: Desktop variant of application

## Data Flow

**Real-Time Intelligence Pipeline:**

1. **Data Ingestion** → External sources (RSS, APIs, WebSockets) are polled by domain handlers
   - Example: `server/worldmonitor/seismology/v1/handler.ts` calls USGS Earthquake API
   - Example: `src/services/aviation/index.ts` calls FAA NASSTATUS and Eurocontrol APIs

2. **Domain Processing** → Handler transforms and enriches raw data using service layer
   - Handlers invoke domain service (e.g., `src/services/aviation/index.ts`) to parse, validate, map proto fields
   - Domain service applies circuit breakers, caching, deduplication

3. **Response Serialization** → Proto codec serializes to Protobuf wire format
   - Auto-generated service_server.ts creates HTTP response with Content-Type: application/protobuf

4. **Client Reception** → Browser receives proto bytes and deserializes via generated client
   - Client invokes `new AviationServiceClient({fetch: fetch.bind(globalThis)})`
   - Generated client stub deserializes response into typed TS objects

5. **State Management & Aggregation** → Frontend services aggregate across domains
   - `src/services/signal-aggregator.ts` merges signals from multiple domains into unified risk indicators
   - `src/services/geo-convergence.ts` detects multi-domain events at same location
   - `src/services/country-instability.ts` fuses news, military, conflict, outages into country-level CII score

6. **Visualization** → Components render data on map and in panels
   - MapLibre GL renders base layers (tiles, satellite)
   - Deck.gl renders 3D overlays (flights, vessels, heatmaps)
   - React panels display tabular data, charts (D3), timelines

7. **Persistence** → Local IndexedDB caches responses and computed state
   - `src/services/storage.ts` manages IndexedDB schema and transactions
   - Used for offline support, playback control, baseline trending

**State Management:**

- **Distributed State**: Each domain service manages its own fetch state, caching, retry logic
- **Component Local State**: React components use hooks (useState) for UI state (selected hotspot, time range, panel visibility)
- **Global Config**: Shared via `src/config/` and module-level constants
- **LocalStorage**: User preferences (theme, API keys, language) persisted to localStorage
- **IndexedDB**: Historical data for trending, baselines, snapshots

**Authentication & Authorization:**

- API Key validation happens at edge function level (`api/_api-key.js`)
- Keys are optional for read-only public endpoints, required for premium data
- Desktop mode stores keys in encrypted config via Tauri
- Browser mode accepts keys via settings panel

## Key Abstractions

**Domain Service Pattern:**

- Purpose: Encapsulates all logic for a single intelligence domain
- Examples: `src/services/aviation/index.ts`, `src/services/military-flights.ts`, `src/services/conflict/index.ts`
- Pattern: Export factory function that returns service object with typed methods
  ```typescript
  export async function fetchFlightDelays(): Promise<AirportDelayAlert[]>
  export async function initMilitaryVesselStream(): Promise<void>
  ```
- Maps proto types to consumer-friendly types (e.g., proto `FLIGHT_DELAY_SEVERITY_MAJOR` → `'major'`)
- Includes error handling via circuit breakers and fallback values

**Signal Abstraction:**

- Purpose: Represents a detected event/change in one or more domains
- Location: `src/services/signal-aggregator.ts`, `src/services/geo-convergence.ts`
- Pattern: Objects with timestamp, location, source domain, confidence, and action
  ```typescript
  {
    id: string;
    timestamp: number;
    lat: number; lon: number;
    source: 'aviation' | 'military' | 'economic' | ...;
    confidence: number; // 0-1
    description: string;
  }
  ```

**Panel Component Pattern:**

- Purpose: Reusable UI component for displaying domain-specific data
- Location: `src/components/*Panel.ts` (NewsPanel, MarketPanel, MonitorPanel, etc.)
- Pattern: Takes time range, map bounds, and config; returns rendered DOM
  ```typescript
  class NewsPanel extends Panel {
    async render(timeRange: TimeRange, bounds?: MapBounds): Promise<void>
  }
  ```
- Integrates with MapContainer for lifecycle management

**Proto-to-TS Mapping:**

- Purpose: Bridge between proto wire format and consumer-friendly TypeScript types
- Pattern: Service layer includes reverse mapping functions (e.g., `toDisplayAlert()`)
- Ensures proto enums match consumer types before exposure to UI

**Circuit Breaker Pattern:**

- Purpose: Prevent cascading failures when external APIs are unavailable
- Location: `src/utils/circuit-breaker.ts`, used in domain services
- Pattern: Counts failures, temporarily stops calling service, returns cached/fallback data

**Geo-Indexing & Clustering:**

- Purpose: Efficient spatial queries and grouping of events
- Location: `src/services/focal-point-detector.ts`, `src/services/clustering.ts`
- Pattern: H3 hexagonal grids for clustering, KD-trees for spatial indexing

## Entry Points

**Web Application:**

- Location: `index.html` → `src/main.ts` → `src/App.ts`
- Triggers: User navigates to https://worldmonitor.app
- Responsibilities:
  1. Initialize Sentry error tracking, Vercel analytics, PostHog product analytics
  2. Install runtime fetch patches for desktop (Tauri) or web (CORS)
  3. Initialize service worker (PWA) for offline support
  4. Create App instance and call `app.init()`
  5. Set up debug helpers and theme management

**Settings Application:**

- Location: `settings.html` → `src/settings-main.ts`
- Triggers: User opens settings panel or accesses `/settings.html`
- Responsibilities: API key management, language selection, theme preference, data export

**Desktop Application:**

- Location: `src-tauri/src/main.rs` (Rust) → Tauri webview → `src/main.ts`
- Triggers: User launches native executable
- Responsibilities:
  1. Spawn local Node.js sidecar backend (`src-tauri/sidecar/`)
  2. Route `/api/*` requests to localhost sidecar instead of https://worldmonitor.app
  3. Load system credentials and environment secrets

**API Router:**

- Location: `api/[[...path]].js` (Vercel catch-all)
- Triggers: Requests to `/api/{domain}/v1/{rpc}`
- Responsibilities:
  1. Apply CORS headers based on origin
  2. Route to appropriate domain handler
  3. Handle OPTIONS preflight
  4. Map errors to HTTP responses

**Batch Data Processing:**

- Location: `tests/*.test.mjs` (Node.js test runner)
- Triggers: `npm run test:data`
- Responsibilities: Validate data import pipelines, test external API integration

## Error Handling

**Strategy:** Graceful degradation with fallback to cached/mock data

**Patterns:**

- **API Errors**: Circuit breaker stops calling, returns cached result if available, else empty array/null
  - Example: Earthquake API down → use last known earthquakes from IndexedDB
  - Logged via Sentry for monitoring

- **Validation Errors**: Proto validators throw ValidationError, caught at edge, returned as 400
  - Includes violation details for debugging
  - Example: Missing required field in request body

- **Network Errors**: Fetch timeouts (8-30s depending on endpoint) trigger retry with backoff
  - Example: `vite.config.ts` timeout config for different API proxies
  - AIS WebSocket reconnects on disconnect

- **Client Errors**: Components show error modals or disable features gracefully
  - Example: Map tiles fail-over to CartoDB basemap
  - News feeds fail-over to alternative source

- **Unhandled Errors**: Caught by Sentry, filtered to reduce noise
  - `src/main.ts` defines 100+ ignored error patterns (maplibre internals, vendor scripts, etc.)
  - beforeSend hook suppresses low-value errors

- **TypeScript Errors**: Compile-time validation via `tsc` (build step)
  - Proto codegen ensures client/server types always match

## Cross-Cutting Concerns

**Logging:**

- Browser console via `console.log/warn/error`
- Sentry integration in production for error tracking
- PostHog for user behavior analytics
- Example: `src/services/data-freshness.ts` logs when data becomes stale

**Validation:**

- Proto validator codegen (required fields, enum values, numeric ranges)
- Domain service layer validates external API responses
- Circuit breaker validates response shape before returning
- Example: `src/services/aviation/index.ts` checks proto fields before mapping

**Authentication:**

- API key extracted from query params or Authorization header
- Validated at edge function `api/_api-key.js`
- Optional for public endpoints, required for premium data
- Desktop stores in `~/.config/worldmonitor/config.json` (encrypted by Tauri)

**Caching:**

- HTTP caching headers set by Vercel edge functions
- Service worker (PWA) caches assets and API responses per workbox rules
- Domain services use 2-tier cache: in-memory (Map) + IndexedDB (persistent)
- Example: `src/services/cached-theater-posture.ts` caches military posture analysis

**Rate Limiting:**

- External APIs have built-in rate limits (USGS earthquakes, Yahoo Finance, etc.)
- Circuit breaker enforces soft limit before hitting upstream limits
- Errors include `retryAfter` header for client backoff
- Example: `api/_api-key.js` returns 429 with retryAfter

**Internationalization (i18n):**

- Language selection via `src/components/LanguageSelector.ts`
- Locale files lazy-loaded from `src/locales/{lang}.json`
- Dynamic translation via `src/services/i18n.ts`
- Example: News titles auto-translated if required

---

*Architecture analysis: 2026-02-21*
