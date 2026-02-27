# Technology Stack

**Analysis Date:** 2026-02-21

## Languages

**Primary:**
- TypeScript 5.7.2 - All source code (frontend, backend, proto handlers)
- JavaScript (ESM modules) - Build scripts and utility files

**Secondary:**
- Rust - Tauri desktop application (`src-tauri/src/main.rs`)
- Protocol Buffers (protobuf3) - API contracts and type definitions (`proto/` directory with 17 domains)

## Runtime

**Environment:**
- Node.js (version specified in `.nvmrc` if present, otherwise latest LTS expected)
- Browser: ES2020 target with DOM support

**Package Manager:**
- npm (lockfile: `package-lock.json` present)

## Frameworks

**Frontend:**
- React (via Vite SPA)
- MapLibre GL 5.16.0 - Interactive map rendering
- Deck.gl 9.2.6 - WebGL layer rendering (aggregation, geo, layers, mapbox integration)
- D3.js 7.9.0 - Data visualization and charting
- TopoJSON Client 3.1.0 - Geospatial data processing
- Tauri 2.10.0 - Desktop application (macOS, Windows, Linux)

**Build & Dev:**
- Vite 6.0.7 - Module bundler and dev server
- TypeScript 5.7.2 - Type checking and compilation
- esbuild 0.27.3 - Fast JavaScript bundler (used by Vite)

**Testing:**
- Playwright 1.52.0 - E2E testing (visual regression, runtime tests)
- Node.js `--test` runner - Data validation tests (`test:data` via Node's native test framework)

**Analytics & Observability:**
- Sentry (@sentry/browser 10.39.0) - Error tracking
- PostHog (posthog-js 1.352.0) - Product analytics with privacy-first design
- Vercel Analytics (@vercel/analytics 1.6.1) - Web vitals tracking

**Backend Runtime:**
- Convex (convex 1.32.0) - Backend-as-a-Service for database and auth
- Upstash Redis (@upstash/redis 1.36.1) - Caching and temporal anomaly detection

**ML & NLP:**
- Transformers.js (@xenova/transformers 2.17.2) - Browser-side language models (T5 fallback)
- ONNX Runtime (onnxruntime-web 1.23.2) - ML model inference in browser

**Utilities:**
- i18next 25.8.10 - Internationalization (14 languages)
- i18next-browser-languagedetector 8.2.1 - Auto language detection
- fast-xml-parser 5.3.7 - RSS feed parsing
- youtubei.js 16.0.1 - YouTube API client
- WebSocket (ws 8.19.0) - Real-time data streams

**PWA & Offline:**
- vite-plugin-pwa 1.2.0 - Service worker and PWA manifest generation
- Workbox (integrated via vite-plugin-pwa) - Cache strategies and offline support

## Key Dependencies

**Critical:**
- convex 1.32.0 - Database backend for registrations and config
- @sentry/browser 10.39.0 - Production error tracking (non-Tauri, non-localhost only)
- maplibre-gl 5.16.0 - Required for interactive map rendering
- @deck.gl/core + satellites - WebGL rendering for 35+ data layers

**Infrastructure & APIs:**
- @upstash/redis 1.36.1 - Redis client for caching and anomaly detection
- youtubei.js 16.0.1 - YouTube live stream detection

## Configuration

**Environment:**
- Config via `.env` file (dev) and runtime env vars (prod)
- Vite build-time variables: `VITE_*` prefix
- Runtime variables: `process.env.*` in Node.js contexts

**Key Environment Variables:**
- `FRED_API_KEY` - Federal Reserve Economic Data API
- `FINNHUB_API_KEY` - Stock market data
- `EIA_API_KEY` - Energy Information Administration
- `GROQ_API_KEY` - Cloud LLM provider
- `OPENROUTER_API_KEY` - Cloud LLM provider alternative
- `CLOUDFLARE_API_TOKEN` - Internet outage monitoring
- `ACLED_ACCESS_TOKEN` - Conflict data
- `URLHAUS_AUTH_KEY` - Malware URL database
- `OTX_API_KEY` - AlienVault cyber threats
- `ABUSEIPDB_API_KEY` - Malicious IP database
- `WINGBITS_API_KEY` - Military aircraft tracking
- `OPENSKY_CLIENT_ID` / `OPENSKY_CLIENT_SECRET` - OpenSky Network
- `AISSTREAM_API_KEY` - Maritime vessel tracking
- `NASA_FIRMS_API_KEY` - Satellite fire detection
- `UC_DP_KEY` - UCDP conflict database
- `OLLAMA_API_URL` / `OLLAMA_MODEL` - Local LLM integration
- `VITE_SENTRY_DSN` - Sentry error tracking (optional)
- `VITE_POSTHOG_KEY` - PostHog analytics (optional)
- `VITE_POSTHOG_HOST` - PostHog server hostname
- `WORLDMONITOR_API_KEY` - Gates cloud fallback in desktop app
- `VITE_VARIANT` - App variant selection (full/tech/finance)
- `VITE_DESKTOP_RUNTIME` - Desktop app flag for Tauri
- `VITE_MAP_INTERACTION_MODE` - Map mode (3D/flat)

**Build:**
- Vite config: `vite.config.ts`
- TypeScript config: `tsconfig.json` (strict mode, no emit)
- Convex schema: `convex/schema.ts` (registrations table)

## Platform Requirements

**Development:**
- Node.js with npm
- Rust toolchain (if building Tauri desktop app)
- Protobuf compiler tools: `buf`, `protoc-gen-ts-client`, `protoc-gen-ts-server` (for proto codegen)

**Production:**
- Deployed on Vercel (serverless functions via `api/[[...path]].ts`)
- Desktop distribution: macOS (arm64, x64), Windows (.exe), Linux (.AppImage)
- Browser: modern browsers with ES2020 support
- Service worker supported browsers for PWA features

**Storage & Runtime:**
- Convex database backend (cloud)
- Upstash Redis (cloud)
- Local browser storage (IndexedDB, localStorage) for cache and installation ID

---

*Stack analysis: 2026-02-21*
