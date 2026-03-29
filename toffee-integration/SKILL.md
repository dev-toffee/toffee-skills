---
name: toffee-integration
description: Use when integrating the toffee SDK (@toffee-at/sdk) into a website, setting up toffee for a new project, or answering questions about toffee's AI agent detection, tracking, configuration, or API. Triggers on "toffee", "agent detection", "agent-radar", "bot detection SDK", "add toffee", "set up toffee".
---

# Toffee SDK Integration

## Overview

Toffee detects AI agents, bots, and humans visiting your website using 6-layer client-side Bayesian detection + server-side ML classification (SAINT). It classifies visitors into 3 classes: **human**, **bot**, **agent**.

**Package:** `@toffee-at/sdk` on npm
**Docs:** https://docs.toffee.at
**Dashboard:** https://db.toffee.at
**API (prod):** https://api.toffee.at

## Quick Reference

| Export | Purpose |
|--------|---------|
| `init(config)` | Initialize SDK (takes `AgentRadarConfig`), returns instance with `.destroy()` |
| `identify(metadata)` | Tag session with user metadata (module-level, targets last instance) |
| `getDetection()` | Get latest `DetectionOutput` synchronously or `null` (module-level) |

```typescript
interface AgentRadarConfig {
  siteId: string;            // Required. From dashboard site settings
  apiKey: string;            // Required. Starts with ar_, from dashboard
  endpoint?: string;         // Default: '/api/v1/events' (use absolute URL if API is on different domain)
  debug?: boolean;           // Logs to console as [agent-radar] (default: false)
  onDetection?: (result: DetectionOutput) => void;
}
```

### ToffeeInstance methods

| Method | Purpose |
|--------|---------|
| `toffee.identify(props)` | Attach custom key-value metadata to the session |
| `toffee.getDetection()` | Get current `DetectionOutput` or `null` if not yet run |
| `toffee.destroy()` | Flush pending events, remove listeners, clean up timers |

`init()` is a **singleton** — calling it twice returns the existing instance. Call `.destroy()` before re-initializing.

## Integration: Existing Project

### 1. Get an API Key

Go to https://db.toffee.at, create a team and site. Copy the API key (starts with `ar_`). It's shown only once.

### 2. Install

**NPM:**
```bash
npm install @toffee-at/sdk   # NOT @toffee/sdk — that's a different package
```

**Script tag (auto-initializes):**
```html
<script src="https://cdn.toffee.at/sdk.js" data-api-key="YOUR_API_KEY"></script>
```

**Tracking pixel (non-JS environments — curl, wget, Python requests, CLI agents):**
```html
<img src="https://api.toffee.at/api/v1/pixel?k=ar_YOUR_KEY&url=https://yoursite.com/page"
     alt="" width="1" height="1" style="position:absolute;left:-9999px" />
```
The pixel does server-side classification from HTTP headers only. API key is visible in the URL (write-only, scoped to site).

### 3. Initialize

**React / Next.js (App Router):**

```tsx
'use client';
import { useEffect, useRef } from 'react';
import { init } from '@toffee-at/sdk';

export function ToffeeProvider() {
  const initialized = useRef(false);
  useEffect(() => {
    if (initialized.current) return; // Guard against strict mode double-mount
    initialized.current = true;
    const toffee = init({
      siteId: 'YOUR_SITE_ID',
      apiKey: 'ar_YOUR_KEY',
      endpoint: 'https://api.toffee.at/api/v1/events',
      onDetection: (result) => {
        console.log(result.classification.classification); // 'human' | 'bot' | 'agent'
        console.log(result.riskTier);       // 'definite-human' to 'definite-bot'
        console.log(result.probability);    // 0.0 - 1.0
      },
    });
    return () => { toffee.destroy(); initialized.current = false; };
  }, []);
  return null;
}
```

Mount in root layout:
```tsx
// app/layout.tsx
<body>{children}<ToffeeProvider /></body>
```

**Vue:**
```vue
<script setup>
import { onMounted, onUnmounted } from 'vue';
import { init } from '@toffee-at/sdk';
let toffee;
onMounted(() => { toffee = init({ siteId: '...', apiKey: 'ar_...', endpoint: '...' }); });
onUnmounted(() => toffee?.destroy());
</script>
```

### 4. Track Custom Elements

Auto-tracked: `h1-h6, p, a, button, form, input, img`. For anything else:

```html
<div data-toffee-track="hero-banner">...</div>
<button data-toffee-track="signup-cta">Sign Up</button>
```

Dynamically added elements are picked up automatically via `MutationObserver`.

### 5. Identify Users (Optional)

```typescript
import { identify } from '@toffee-at/sdk';
identify({ userId: '123', email: 'user@example.com', plan: 'pro' });
```

Custom properties appear in the dashboard for filtering.

## Tracking & Sessions

### Automatic Events

| Event | Trigger | Data |
|-------|---------|------|
| `pageview` | Page load + SPA route changes | URL, referrer, page title |
| `detection` | Detection probability changes | Score, isAgent flag, contributing signals |
| `session_end` | User leaves or hides tab | Duration (ms), page count, interaction count → triggers server-side ML |

### Tracked Interactions

- **Clicks**: element details, trusted status
- **Scroll depth**: milestone percentages
- **Form fields**: selector and timing only (no values captured)

### Session Management

Sessions use random IDs in `sessionStorage`. A new session starts when a new tab opens or sessionStorage is cleared. Sessions persist across navigations within a single tab.

### Event Batching

Events are batched and transmitted periodically, on session end, or when buffer thresholds are reached. On page unload, SDK uses `sendBeacon` with `?apiKey=` query param instead of header. The API accepts both.

## Detection Output

```typescript
interface DetectionOutput {
  score: number;          // 0-100 legacy
  probability: number;    // 0.0-1.0 Bayesian posterior
  riskTier: 'definite-bot' | 'likely-bot' | 'suspicious' | 'likely-human' | 'definite-human';
  isAgent: boolean;       // probability >= 0.50
  signals: { detector: string; fired: boolean; llr: number; evidence: string[] }[];
  classification: {
    classification: 'human' | 'bot' | 'agent';
    probabilities: { human: number; bot: number; agent: number };
    source: 'model' | 'heuristic';
  };
}
```

### Risk Tiers

| Tier | Probability | Recommended Action |
|------|------------|-------------------|
| `definite-bot` | >= 0.95 | Block or challenge |
| `likely-bot` | >= 0.80 | Block or challenge |
| `suspicious` | >= 0.50 | Soft challenge or log |
| `likely-human` | >= 0.20 | Allow |
| `definite-human` | < 0.20 | Allow |

## Detection Timeline

`onDetection` fires **multiple times** as detection refines:

| Stage | When | What's included |
|-------|------|-----------------|
| instant | 0ms | UA, headless, automation, navigator, fingerprint |
| early | 3s | + behavioral analysis |
| session | 10s | + accumulated behavioral data |
| extended | 30s | Final scheduled assessment |
| continuous | every 15s | Catches late/extension agents |
| interaction | 1s after click/scroll | Re-score on user action |
| model | async from server | ML 3-class probabilities |

### Six Detector Categories

1. **User-agent analysis** — Known bot/agent UA strings
2. **Headless browser detection** — Missing browser features
3. **Automation framework detection** — Selenium, Puppeteer, Playwright markers
4. **Navigator inconsistencies** — Mismatched properties
5. **Browser fingerprint anomalies** — Canvas, WebGL, font anomalies
6. **Behavioral analysis** — Mouse, scroll, keyboard patterns

## API Reference

**Base URL:** `https://api.toffee.at`

### Authentication

| Method | Usage | How |
|--------|-------|-----|
| Session cookie | Dashboard, team management | Via `/api/auth/*` (BetterAuth) |
| API key (header) | SDK event ingestion | `X-Api-Key: ar_YOUR_KEY` |
| API key (query) | Tracking pixel, sendBeacon | `?apiKey=ar_YOUR_KEY` or `?k=ar_YOUR_KEY` |

### SDK Endpoints (API key auth)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/events` | Ingest event batch. Returns `{ accepted, classifications }` |
| GET | `/api/v1/pixel?k={key}&url={url}` | Tracking pixel (1x1 GIF) |

### Analytics Endpoints (Session auth)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/stats?range=24` | Aggregate stats |
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/agents?range=24` | Detected agents |
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/pages?range=24` | Per-page analytics |
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/elements?page={url}` | Element interactions |
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/visitors?range=24` | Visitor sessions |
| GET | `/api/v1/teams/{teamSlug}/analytics/{siteId}/session/{sessionId}` | Session events |
| GET | `/api/v1/live?siteId={id}` | Real-time SSE stream |

`range` parameter = hours (default 24, e.g. `range=168` for 7 days).

### Team & Site Management (Session auth)

| Method | Path | Purpose |
|--------|------|---------|
| GET/POST | `/api/v1/teams` | List / create teams |
| GET/PATCH/DELETE | `/api/v1/teams/{teamSlug}` | Get / update / delete team |
| GET/POST | `/api/v1/teams/{teamSlug}/members` | List / add members |
| PATCH/DELETE | `/api/v1/teams/{teamSlug}/members/{userId}` | Update role / remove member |
| GET/POST | `/api/v1/teams/{teamSlug}/sites` | List / create sites (returns API key on create) |
| GET/PATCH/DELETE | `/api/v1/teams/{teamSlug}/sites/{siteId}` | Get / update / delete site |
| GET/POST | `/api/v1/teams/{teamSlug}/sites/{siteId}/keys` | List / create API keys |
| DELETE | `/api/v1/teams/{teamSlug}/sites/{siteId}/keys/{keyId}` | Revoke API key |

### Error Format

```json
{ "message": "Description of the error" }
```
Status codes: 400 (invalid params), 401 (unauthenticated), 403 (insufficient permissions), 404 (not found), 500 (server error).

## Integration: Fresh Project (Self-Hosted)

**Prerequisites:** PostgreSQL installed and running locally.

```bash
git clone https://github.com/VishiATChoudhary/toffee.git toffee-sdk
cd toffee-sdk
make setup    # Install deps, create PostgreSQL DB 'toffee', build shared packages
make dev      # Start API (port 3001) + Dashboard (port 5173)
```

**Manual test harness:**
```bash
make humantest  # API + Dashboard + test page (port 4000)
```
Then: Dashboard (localhost:5173) → sign up → create team & site → copy API key → Test page (localhost:4000) → paste key → Init SDK.

**Environment variables** (override via `make VAR=value`):
- `DATABASE_URL` — PostgreSQL connection (default: `postgresql://postgres:password@localhost:5432/toffee`)
- `API_PORT` — default 3001
- `MODEL_SERVICE_URL` — optional, for ML classification (falls back to heuristics)

**Deploy:**
```bash
make deploy-api      # Fly.io
make deploy-model    # Model service to Fly.io
make deploy-website  # GCP Cloud Run
```

## Deployment: Environment Variables

The SDK needs `NEXT_PUBLIC_TOFFEE_API_KEY` (or equivalent public env var) available at **build time** since it runs client-side. The `NEXT_PUBLIC_` prefix is Next.js-specific — other frameworks use different prefixes.

### Required Variables

| Variable | Value | Notes |
|----------|-------|-------|
| `NEXT_PUBLIC_TOFFEE_API_KEY` | `ar_...` | Public, safe to expose (write-only, scoped to site) |
| `NEXT_PUBLIC_TOFFEE_SITE_ID` | Site ID from dashboard | **Required** — `siteId` is not optional |
| `NEXT_PUBLIC_TOFFEE_ENDPOINT` | `https://api.toffee.at/api/v1/events` | Only needed if not using default |

Then reference in code:
```typescript
const toffee = init({
  siteId: process.env.NEXT_PUBLIC_TOFFEE_SITE_ID!,   // required — use ! assertion
  apiKey: process.env.NEXT_PUBLIC_TOFFEE_API_KEY!,    // required — use ! assertion
  endpoint: process.env.NEXT_PUBLIC_TOFFEE_ENDPOINT,
});
```

### Vercel

**Dashboard:** Project Settings → Environment Variables → add each variable for Production/Preview/Development.

**CLI:**
```bash
vercel env add NEXT_PUBLIC_TOFFEE_API_KEY        # prompts for value and environments
vercel env add NEXT_PUBLIC_TOFFEE_SITE_ID
```

Or in `vercel.json`:
```json
{
  "env": {
    "NEXT_PUBLIC_TOFFEE_API_KEY": "@toffee-api-key"
  }
}
```
Where `@toffee-api-key` is a Vercel secret created with `vercel secrets add toffee-api-key ar_YOUR_KEY`.

### Netlify

**Dashboard:** Site configuration → Environment variables → add variables.

**CLI:**
```bash
netlify env:set NEXT_PUBLIC_TOFFEE_API_KEY ar_YOUR_KEY
```

Or in `netlify.toml`:
```toml
[build.environment]
  NEXT_PUBLIC_TOFFEE_API_KEY = "ar_YOUR_KEY"
```

### Cloudflare Pages

**Dashboard:** Settings → Environment variables → add for Production and Preview.

**CLI (`wrangler`):**
```bash
wrangler pages project edit --binding NEXT_PUBLIC_TOFFEE_API_KEY=ar_YOUR_KEY
```

### AWS Amplify

**Dashboard:** App settings → Environment variables → Manage variables.

Or in `amplify.yml`:
```yaml
build:
  commands:
    - env | grep NEXT_PUBLIC_TOFFEE  # verify vars are set
    - npm run build
```

### Docker / Docker Compose

Pass at build time (since client-side code is bundled during build):
```dockerfile
ARG NEXT_PUBLIC_TOFFEE_API_KEY
ENV NEXT_PUBLIC_TOFFEE_API_KEY=$NEXT_PUBLIC_TOFFEE_API_KEY
RUN npm run build
```

```yaml
# docker-compose.yml
services:
  web:
    build:
      args:
        NEXT_PUBLIC_TOFFEE_API_KEY: ${NEXT_PUBLIC_TOFFEE_API_KEY}
```

### Fly.io

```bash
fly secrets set NEXT_PUBLIC_TOFFEE_API_KEY=ar_YOUR_KEY
```

Note: For client-side env vars with Fly.io, set them as build args in `fly.toml`:
```toml
[build.args]
  NEXT_PUBLIC_TOFFEE_API_KEY = "ar_YOUR_KEY"
```

### Framework Env Var Prefixes

| Framework | Public prefix | Example |
|-----------|--------------|---------|
| Next.js | `NEXT_PUBLIC_` | `NEXT_PUBLIC_TOFFEE_API_KEY` |
| Vite / SvelteKit | `VITE_` | `VITE_TOFFEE_API_KEY` |
| Create React App | `REACT_APP_` | `REACT_APP_TOFFEE_API_KEY` |
| Nuxt 3 | `NUXT_PUBLIC_` | `NUXT_PUBLIC_TOFFEE_API_KEY` |
| Astro | `PUBLIC_` | `PUBLIC_TOFFEE_API_KEY` |
| Remix | N/A (use `window.ENV`) | Pass via loader |
| Angular | N/A (use `environment.ts`) | Set in build config |

Adapt the variable name to match your framework's prefix. The API key is safe to expose publicly — it's write-only and scoped to your site.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Calling `init()` server-side (SSR) | SDK accesses `document`, `navigator`, `sessionStorage`. Always wrap in `useEffect` or `'use client'` component |
| Using relative `endpoint` with cross-domain API | Use absolute URL: `https://api.toffee.at/api/v1/events` |
| Making irreversible decisions on first `onDetection` | First callback has no behavioral data. Wait for `early` (3s) or `session` (10s) stage |
| Calling `init()` twice expecting fresh instance | `init()` is a singleton. Call `.destroy()` first if re-initializing |
| Missing CORS for custom API host | Toffee API has `cors()` built in. If self-hosting behind a proxy, ensure `X-Api-Key` header is allowed |
| Not handling `sendBeacon` fallback | On page unload, SDK uses `sendBeacon` with `?apiKey=` query param instead of header. API accepts both |

## Key Codebase Paths

| Path | Purpose |
|------|---------|
| `packages/sdk/src/core.ts` | SDK orchestration, config interface |
| `packages/sdk/src/detect/` | 6 detection layers + Bayesian fusion |
| `packages/sdk/src/detect/behavioral.ts` | Mouse/scroll/keyboard analysis |
| `packages/shared/src/agents.ts` | 50+ known agent patterns |
| `packages/shared/src/events.ts` | Event type definitions |
| `apps/api/src/routes/ingest.ts` | Event ingestion endpoint |
| `apps/api/src/routes/pixel.ts` | Tracking pixel endpoint |
| `test-page/index.html` | Interactive test harness |
| `Makefile` | All dev/deploy commands |
