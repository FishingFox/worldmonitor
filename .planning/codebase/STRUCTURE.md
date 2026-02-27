# Codebase Structure

**Analysis Date:** 2026-02-21

## Directory Layout

```
worldmonitor.nosync/
├── src/                          # Frontend application (TypeScript/React)
│   ├── App.ts                    # Main app orchestrator
│   ├── main.ts                   # Browser entry point
│   ├── settings-main.ts          # Settings page entry point
│   ├── components/               # React UI components (54+ panels and views)
│   ├── services/                 # Domain and cross-cutting services (79+ files)
│   ├── config/                   # Static configuration (feeds, geographies, markets)
│   ├── utils/                    # Shared utilities (export, theme, proxy, etc.)
│   ├── styles/                   # CSS stylesheets
│   ├── types/                    # TypeScript type definitions
│   ├── generated/                # Proto-generated client/server stubs
│   ├── locales/                  # i18n translation JSON files (30+ languages)
│   ├── workers/                  # Web Workers for background tasks
│   └── bootstrap/                # Initialization code (chunk reload guard, etc.)
│
├── server/                       # Backend server code (Sebuf handlers)
│   ├── router.ts                 # Route matcher for sebuf services
│   ├── cors.ts                   # CORS header logic
│   ├── error-mapper.ts           # Error → HTTP response mapping
│   ├── _shared/                  # Shared backend utilities
│   └── worldmonitor/
│       └── {domain}/v1/          # 18 domains: seismology, wildfire, climate, etc.
│           └── handler.ts        # Domain RPC handler
│
├── api/                          # Vercel edge functions (catch-all gateway + helpers)
│   ├── [[...path]].js            # Main sebuf API router (compiled from vite build)
│   ├── _api-key.js               # API key validation
│   ├── _cors.js                  # CORS helper
│   ├── story.js                  # Story page rendering
│   ├── og-story.js               # Open Graph meta tag generation
│   ├── download.js               # Data export handler
│   ├── rss-proxy.js              # RSS feed proxy
│   ├── fwdstart.js               # Forward/start event handler
│   ├── version.js                # Version info endpoint
│   ├── register-interest.js       # Newsletter signup
│   ├── [domain]/v1/[rpc].ts      # Legacy file-based API routes (deprecated)
│   ├── data/                     # Static data (city coordinates)
│   ├── eia/                      # Energy Information Admin proxy
│   ├── youtube/                  # YouTube utilities (live detection, embed)
│   └── test files                # _cors.test.mjs, usni-fleet.test.mjs, etc.
│
├── proto/                        # Protocol Buffer definitions
│   ├── worldmonitor/
│   │   ├── core/v1/              # Base types (Location, Timestamp, etc.)
│   │   └── {domain}/v1/          # 18 domain protos
│   │       └── service.proto      # RPC definitions
│   └── sebuf/http/               # Sebuf HTTP framework types
│
├── src-tauri/                    # Desktop application (Rust + TypeScript)
│   ├── src/                      # Rust backend
│   ├── sidecar/                  # Node.js sidecar backend (local API server)
│   ├── tauri.conf.json           # Desktop app config
│   ├── Cargo.toml                # Rust dependencies
│   └── icons/                    # App icons
│
├── convex/                       # Convex backend (optional realtime DB)
│   └── (configuration)
│
├── tests/                        # Node.js test files for data pipelines
│   └── *.test.mjs                # Jest/Node test runners
│
├── e2e/                          # End-to-end tests (Playwright)
│   └── *.spec.ts                 # Playwright test suites
│
├── public/                       # Static assets (images, favicons, manifests)
│   ├── favico/                   # Favicon variants
│   └── (other assets)
│
├── deploy/                       # Deployment configuration
│   └── nginx/                    # Nginx reverse proxy config
│
├── scripts/                      # Build and utility scripts
│   ├── build-sidecar-sebuf.mjs   # Compile sidecar handlers
│   ├── desktop-package.mjs       # Package desktop builds
│   └── sync-desktop-version.mjs  # Keep version in sync
│
├── docs/                         # Documentation (design, API, etc.)
│
├── .planning/                    # GSD planning and state documents
│   ├── ROADMAP.md                # Phase breakdown
│   ├── STATE.md                  # Current migration status
│   └── codebase/                 # This directory (ARCHITECTURE.md, STRUCTURE.md, etc.)
│
├── vite.config.ts                # Frontend build config (dev server, plugins, chunking)
├── tsconfig.json                 # TypeScript compiler config
├── package.json                  # npm dependencies and scripts
├── index.html                    # Main SPA entry point
├── settings.html                 # Settings page entry point
└── .env.example                  # Environment variables template
```

## Directory Purposes

**src/**
- Main TypeScript/React application code
- Builds to `dist/` after running `npm run build`
- Served by Vercel or local dev server

**src/components/**
- 54+ React UI components, organized by feature
- Naming convention: `{Feature}Panel.ts` for dashboard panels
- Examples: `NewsPanel.ts`, `MarketPanel.ts`, `MonitorPanel.ts`, `MapContainer.ts`, `DeckGLMap.ts`

**src/services/**
- Business logic organized by domain (aviation/, climate/, economic/, etc.)
- Cross-cutting services (clustering.ts, geo-convergence.ts, country-instability.ts)
- Utility services (storage.ts, rss.ts, market.ts, weather.ts)
- Each domain has index.ts exporting public interface

**src/config/**
- Static data files: feeds.ts (RSS sources), geo.ts (military bases, pipelines, cables), etc.
- Immutable after load time (used for dropdowns, map markers)
- Example: INTEL_HOTSPOTS array used by MapPopup.ts for clickable locations

**src/utils/**
- Shared helper functions (export.ts, circuit-breaker.ts, theme-manager.ts)
- No domain-specific logic

**src/generated/**
- Auto-generated from `proto/` by `buf generate`
- Directory: client/{domain}/v1/service_client.ts
- Directory: server/{domain}/v1/service_server.ts
- NEVER edit manually

**src/types/index.ts**
- Consumer-facing TypeScript interfaces (NewsItem, Hotspot, Signal, etc.)
- Separate from proto-generated types to allow mapping

**server/**
- Sebuf handler implementations (where RPC logic lives)
- One handler file per domain: `server/worldmonitor/{domain}/v1/handler.ts`
- Imports: domain service from `src/services/{domain}/`, proto types from `src/generated/`

**api/**
- Vercel edge function entry points
- `[[...path]].js` is the main router (compiled from TypeScript by vite build)
- Static JS files for utility endpoints (version.js, rss-proxy.js, etc.)
- Test files use `.test.mjs` or `.test.js` suffix

**proto/worldmonitor/**
- 18 domain directories, each with v1/service.proto
- Core types in core/v1/
- Run `cd proto && buf generate` to regenerate TypeScript code

**src-tauri/**
- Desktop-specific code
- Rust backend in src/, JavaScript bindings auto-generated by Tauri
- Sidecar runs local Node.js server on localhost:3001

**tests/** and **e2e/**
- Unit/integration tests use Node.js test runner (no Jest config)
- E2E tests use Playwright with visual snapshot testing

**public/**
- Static assets (never change after deploy)
- Service worker cache via vite-plugin-pwa

## Key File Locations

**Entry Points:**
- `index.html`: Main web app entry point
- `settings.html`: Settings/dashboard page
- `src/main.ts`: Main app initialization
- `src/settings-main.ts`: Settings app initialization
- `api/[[...path]].js`: Vercel API gateway (compiled from server/ + vite)

**Configuration:**
- `src/config/index.ts`: Central config exports
- `src/config/feeds.ts`: RSS feed list
- `src/config/geo.ts`: Geographic hotspots, conflict zones, military bases
- `src/config/markets.ts`: Stock/commodity symbols
- `vite.config.ts`: Frontend build config (dev server middleware, Vercel plugins)

**Core Logic:**
- `src/App.ts`: Main orchestrator (initializes panels, fetches data, manages lifecycle)
- `src/services/index.ts`: Barrel export of all domain services
- `src/services/{domain}/index.ts`: Individual domain service (fetch + transform)
- `server/worldmonitor/{domain}/v1/handler.ts`: RPC handler implementation
- `server/worldmonitor/{domain}/v1/list-{entity}.ts`: Individual RPC implementation

**Testing:**
- `e2e/`: Playwright test suites (*.spec.ts)
- `tests/`: Node.js test scripts (*.test.mjs)
- `api/_cors.test.mjs`: CORS middleware tests
- `api/cyber-threats.test.mjs`: Cyber threat fetcher tests

## Naming Conventions

**Files:**
- Component files: `{Name}Panel.ts` or `{Name}Modal.ts` or `{Name}.ts`
- Service files: lowercase with hyphens (e.g., `geo-convergence.ts`, `country-instability.ts`)
- Handler files: `handler.ts` in domain directories
- RPC implementations: `{action}-{entity}.ts` (e.g., `list-earthquakes.ts`, `list-airports.ts`)
- Config files: lowercase plural (e.g., `feeds.ts`, `bases.ts`)
- Test files: `.test.mjs` for Node.js, `.spec.ts` for Playwright

**Directories:**
- Domain services: `src/services/{domain}/` (all lowercase, no hyphens)
- Proto domains: `proto/worldmonitor/{domain}/` (matches service name)
- Handler directories: `server/worldmonitor/{domain}/v1/`
- Components: Grouped by feature, no nested directories (flat)

**Exported Functions/Classes:**
- camelCase for functions: `fetchAirportDelays()`, `calculateCII()`
- PascalCase for classes/React components: `NewsPanel`, `MapContainer`
- camelCase for types/interfaces: `newsItem`, `clusterEvent`

**Constants:**
- SCREAMING_SNAKE_CASE: `INTEL_HOTSPOTS`, `DEFAULT_PANELS`, `STORAGE_KEYS`

## Where to Add New Code

**New Feature (e.g., new intelligence domain):**
1. Create proto definition: `proto/worldmonitor/{domain}/v1/service.proto`
2. Generate code: `cd proto && buf generate`
3. Create service module: `src/services/{domain}/index.ts` with fetch + transform
4. Create handler: `server/worldmonitor/{domain}/v1/handler.ts` + individual RPC files
5. Register in `vite.config.ts` (sebufApiPlugin, lines 213-301)
6. Create UI component: `src/components/{Domain}Panel.ts`
7. Import in `src/App.ts`

**New Component/UI:**
- Implementation: `src/components/{Name}.ts` or `src/components/{Name}Panel.ts`
- Styling: Add to `src/styles/main.css` or co-locate in component directory
- Types: Add to `src/types/index.ts` if consumer-facing

**Utilities:**
- Shared helpers: `src/utils/{name}.ts`
- Domain-specific helpers: `src/services/{domain}/` subdirectory
- Shared backend: `server/_shared/` (e.g., shared validators, mappers)

**Tests:**
- E2E tests: `e2e/{feature}.spec.ts` (Playwright)
- Data tests: `tests/{feature}.test.mjs` (Node.js)
- Handler tests: `server/worldmonitor/{domain}/v1/{rpc}.test.ts` (if added)

## Special Directories

**src/generated/**
- Purpose: Auto-generated proto code (NEVER edit manually)
- Generated by: `cd proto && buf generate`
- Committed: YES (for reproducible builds without protoc)
- Contents: Client stubs, server interfaces, type definitions

**src/locales/**
- Purpose: i18n translations (30+ languages)
- Generated by: Maintained manually or via i18n tool
- Lazy-loaded after app init (see vite.config.ts line 579)
- Committed: YES

**.planning/**
- Purpose: GSD planning documents and roadmap
- Committed: YES
- Updated by: Orchestrator commands

**src-tauri/sidecar/**
- Purpose: Local Node.js server for desktop mode
- Built separately: `npm run build:sidecar-sebuf`
- Packaged: Inside .app/.exe bundle
- Committed: Source YES, binaries NO (in .gitignore)

**dist/ and .next/**
- Purpose: Build artifacts
- Generated by: `npm run build`
- Committed: NO (in .gitignore)

---

*Structure analysis: 2026-02-21*
