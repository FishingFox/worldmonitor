# External Integrations

**Analysis Date:** 2026-02-21

## APIs & External Services

**Economic Data:**
- Federal Reserve Economic Data (FRED) - Time series economic indicators
  - SDK/Client: Direct HTTP (no SDK)
  - Endpoint: `https://api.stlouisfed.org/fred`
  - Auth: `FRED_API_KEY` (env var, optional, returns graceful empty on missing)
  - Handler: `server/worldmonitor/economic/v1/get-fred-series.ts`
  - Usage: Macro economic signals

**Financial Markets:**
- Yahoo Finance API - Stock market data
  - Proxy: `/api/yahoo` (dev Vite proxy to `https://query1.finance.yahoo.com`)
  - Auth: None
  - Usage: Stock prices and market data

- Finnhub - Stock, options, crypto data
  - Auth: `FINNHUB_API_KEY` (env var)
  - Usage: Market context for Tech/Finance variants

- Polymarket - Prediction markets data
  - Proxy: `/api/polymarket` (dev plugin at `vite.config.ts` lines 159-204)
  - Endpoint: `https://gamma-api.polymarket.com`
  - Auth: None (Cloudflare blocks server-side TLS, returns empty array)
  - Usage: Market predictions visualization

**Geopolitical & Intelligence:**
- GDELT Project - Global event database
  - Proxy: `/api/gdelt` (to `https://api.gdeltproject.org`)
  - Usage: Geopolitical event tracking
  - Handler: `src/services/gdelt-intel.ts`

- ACLED - Armed Conflict Location & Event Database
  - Auth: `ACLED_ACCESS_TOKEN` (env var)
  - Usage: Conflict zone tracking

- UCDP - Uppsala Conflict Data Program
  - Auth: `UC_DP_KEY` (env var)
  - Usage: Conflict and displacement data
  - Handler: `server/worldmonitor/displacement/v1/handler.ts`

- OpenAI/Anthropic/Google AI Blogs - News source
  - Proxy: `/rss/openai`, `/rss/anthropic`, `/rss/googleai` (RSS feeds)
  - Usage: Tech news aggregation

**Military & Aviation:**
- OpenSky Network - Aircraft tracking
  - Proxy: `/api/opensky` (to `https://opensky-network.org/api`)
  - Auth: `OPENSKY_CLIENT_ID`, `OPENSKY_CLIENT_SECRET` (env vars)
  - Usage: Military flight detection
  - Handler: `server/worldmonitor/aviation/v1/handler.ts`

- ADS-B Exchange - Military aircraft tracking (backup)
  - Proxy: `/api/adsb-exchange` (to `https://adsbexchange.com/api`)
  - Auth: None (free tier)
  - Usage: Supplement OpenSky with additional military aircraft data

- Wingbits - Military aircraft identification
  - Auth: `WINGBITS_API_KEY` (env var)
  - Usage: Aircraft type classification
  - Handler: `src/services/wingbits.ts`

- FAA NASSTATUS - Airport delays and closures
  - Proxy: `/api/faa` (to `https://nasstatus.faa.gov`)
  - Auth: None
  - Usage: Aviation operations monitoring

**Maritime:**
- AISStream - Live vessel tracking
  - WebSocket Proxy: `/ws/aisstream` (to `wss://stream.aisstream.io`)
  - Auth: `AISSTREAM_API_KEY` (env var)
  - Usage: Real-time ship position tracking
  - Handler: `server/worldmonitor/maritime/v1/handler.ts`

- NGA Maritime Safety Information - Navigation warnings
  - Proxy: `/api/nga-msi` (to `https://msi.nga.mil`)
  - Auth: None
  - Usage: Undersea cable advisories, maritime hazards

**Infrastructure & Outages:**
- Cloudflare Radar - Internet outage tracking
  - Proxy: `/api/cloudflare-radar` (to `https://api.cloudflare.com`)
  - Auth: `CLOUDFLARE_API_TOKEN` (env var)
  - Usage: Global Internet outage detection
  - Handler: `server/worldmonitor/infrastructure/v1/handler.ts`

**Natural Disasters & Climate:**
- USGS Earthquake API - Seismic data
  - Proxy: `/api/earthquake` (to `https://earthquake.usgs.gov`)
  - Auth: None
  - Usage: Earthquake detection and visualization
  - Handler: `server/worldmonitor/seismology/v1/handler.ts`

- NASA EONET - Natural disasters
  - SDK/Client: Direct HTTP
  - Usage: Natural disaster events

- NASA FIRMS - Satellite fire detection
  - Auth: `NASA_FIRMS_API_KEY` (env var)
  - Usage: VIIRS thermal hotspot detection

**Cybersecurity & Threats:**
- URLhaus - Malware URL database
  - Auth: `URLHAUS_AUTH_KEY` (env var)
  - Usage: Malicious URL IOCs
  - Handler: `server/worldmonitor/cyber/v1/handler.ts`

- AlienVault OTX - Cyber threat intelligence
  - Auth: `OTX_API_KEY` (env var)
  - Usage: Cyber threat indicators

- Abuseipdb - Malicious IP database
  - Auth: `ABUSEIPDB_API_KEY` (env var)
  - Usage: IP reputation and threat classification

**News & Content:**
- RSS Feeds (150+ aggregated):
  - BBC, Guardian, NPR, AP News, Al Jazeera, CNN, Hacker News, Ars Technica, The Verge, CNBC, MarketWatch
  - Defense sources: DefenseOne, War on the Rocks, Breaking Defense, Bellingcat
  - Tech: TechCrunch, VentureBeat, MIT Tech Review
  - Government: White House, State Dept, Defense Dept, Justice, CDC, FEMA, DHS, Federal Reserve, SEC, Treasury, CISA
  - Think Tanks: Brookings, CFR, CSIS
  - Finance: Yahoo Finance, Financial Times, Reuters
  - Diplomacy: The Diplomat, Foreign Policy
  - Proxies: `/rss/{source}` pattern in `vite.config.ts` (lines 707-971)

- YouTube Live Stream Detection
  - Proxy: `/api/youtube/live` (plugin at `vite.config.ts` lines 414-478)
  - Scrapes YouTube channel pages for live status
  - Auth: None
  - Usage: Live video embed detection for news channels

**PizzINT (Pentagon Pizza Index):**
- Pentagon Pizza Index - Canteen activity proxy for operations tempo
  - Proxy: `/api/pizzint` (to `https://www.pizzint.watch/api`)
  - Auth: None
  - Usage: Pentagon operational activity indicator

## Data Storage

**Databases:**
- Convex (Backend-as-a-Service)
  - Connection: Cloud-hosted via Convex SDK
  - Client: `convex` package (1.32.0)
  - Schema: `convex/schema.ts` (registrations table with email and metadata)
  - Purpose: User registrations, email collection for API key distribution

**Cache:**
- Upstash Redis (cloud)
  - Connection: REST API via `@upstash/redis` (1.36.1)
  - TTL: 24 hours for world brief summaries, 30+ days for temporal anomaly baseline
  - Purpose: LLM output caching, temporal baseline storage for anomaly detection
  - Auth: Upstash connection token (env var implied)

**Local Storage:**
- Browser IndexedDB - Map tiles, image caches
- Browser localStorage - Installation ID, user settings, language preference
- Desktop: OS keychain (macOS Keychain, Windows Credential Manager) - API keys via Tauri bridge

**File Storage:**
- None (stateless architecture)

## Authentication & Identity

**Backend Auth:**
- Convex Database - Built-in auth (optional)
- API Key Validation - Custom logic in `src/services/runtime-config.ts`
  - Desktop app gates cloud fallback on `WORLDMONITOR_API_KEY` (16+ chars minimum)
  - Validation: `installRuntimeFetchPatch()` checks key before cloud request

**Frontend Auth:**
- No user login required (public OSINT tool)
- Registration: Email collection via Convex for future API key distribution
  - Handler: `convex/registerInterest.ts`
- Installation ID: Pseudonymous UUID stored in localStorage (for analytics)

**OAuth/Third-party:**
- None implemented

## Monitoring & Observability

**Error Tracking:**
- Sentry (@sentry/browser 10.39.0)
  - Init: `src/main.ts`
  - DSN: `VITE_SENTRY_DSN` (env var, optional)
  - Enabled: Production only (excludes localhost, desktop app)
  - Sample rate: 10% trace sampling
  - Filtering: Extensive allowlist for WebGL, map lib, browser API errors
  - Before send hook: Suppresses non-actionable errors (maplibre internals, browser policy)

**Product Analytics:**
- PostHog (posthog-js 1.352.0)
  - Init: `src/services/analytics.ts`
  - Host: `VITE_POSTHOG_HOST` (default: `https://us.i.posthog.com`)
  - API Key: `VITE_POSTHOG_KEY` (env var)
  - Privacy: No session recordings, no autocapture, typed event allowlists only
  - distinct_id: Random UUID (pseudonymous, not identifiable)
  - Features: Custom event schema validation, property sanitization
  - Tracked Events: 30+ (app load, panel view, summary generation, API key config, etc.)

**Web Vitals:**
- Vercel Analytics (@vercel/analytics 1.6.1)
  - Tracked metrics: Core Web Vitals (LCP, FID, CLS, INP, TTFB)

**Logging:**
- Console logging during development (dev proxy errors logged to console)
- No centralized logging service

## CI/CD & Deployment

**Hosting:**
- Vercel (serverless functions)
  - Catch-all API route: `api/[[...path]].ts`
  - Sebuf handlers: 17 domain services (seismology, wildfire, climate, prediction, displacement, aviation, research, unrest, conflict, maritime, cyber, economic, infrastructure, market, news, intelligence, military)
  - Each handler registered in route table via `vite.config.ts` sebufApiPlugin

**Desktop Distribution:**
- Tauri 2.10.0 (builds for macOS, Windows, Linux)
  - macOS: arm64 and x64 universal builds
  - Windows: .exe installers
  - Linux: .AppImage bundles
  - Desktop build: `npm run desktop:build:{variant}`
  - Sidecar: Local Node.js server runs handlers locally (no cloud fallback on sidecar-only mode)

**CI Pipeline:**
- GitHub Actions (inferred from git history)
- Build matrix: 3 variants (full, tech, finance) × 4 E2E test suites
- Testing: Playwright E2E tests with visual regression (golden screenshots)
- Build artifacts: Desktop installers, web builds, PWA manifests

## Environment Configuration

**Required env vars (development):**
- None strictly required (graceful degradation)
- Most API keys optional — missing keys return empty payloads

**Optional env vars (production):**
- `FRED_API_KEY` - Economic data (very commonly used)
- `VITE_SENTRY_DSN` - Error tracking
- `VITE_POSTHOG_KEY` - Analytics
- `WORLDMONITOR_API_KEY` - Desktop cloud fallback gate

**Secrets Location:**
- `.env` file (git-ignored, dev only)
- Environment variables (CI/CD platform secrets, Vercel env settings)
- Desktop: OS keychain integration via Tauri
- Convex: Backend env vars via Convex dashboard

## Webhooks & Callbacks

**Incoming:**
- None implemented

**Outgoing:**
- API key registration form → Convex database insert
  - Endpoint: Convex function `registerInterest()`
  - Data: Email, app version, timestamp

**Sentry Webhook:**
- Error events uploaded to Sentry DSN endpoint (if enabled)

**PostHog Event Submission:**
- Events submitted to PostHog ingest endpoint (if key configured)

## API Rate Limits & Quotas

**FRED:**
- Free tier: 120 calls/minute (respected by handler)

**OpenSky:**
- Free tier: 4 calls/second
- Auth required for premium tier

**Finnhub:**
- Free tier: 60 API calls/minute

**Cloudflare Radar:**
- No documented rate limit (public API)

**YouTube:**
- HTML scrape (no API quota consumed)

**Polymarket Gamma API:**
- No rate limit documented
- Cloudflare JA3 fingerprinting blocks server-side requests

## Data Transfer & Caching

**Service Worker Caching Strategy:**
- Map tiles: CacheFirst (30-day TTL)
- Fonts: StaleWhileRevalidate / CacheFirst (365-day TTL)
- Images: StaleWhileRevalidate (7-day TTL)
- API calls: NetworkOnly (no cache)
- RSS feeds: NetworkOnly (no cache)
- HTML navigation: NetworkFirst (3-second timeout)
- Locale files: CacheFirst (30-day TTL)

**Brotli Precompression:**
- Enabled for production builds
- Compresses: .js, .mjs, .css, .html, .svg, .json, .txt, .xml, .wasm
- Only files >1KB compressed

---

*Integration audit: 2026-02-21*
