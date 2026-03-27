---
name: toffee-integration
description: Use when integrating the toffee SDK (@agent-radar/sdk) into a website, setting up toffee for a new project, or answering questions about toffee's AI agent detection, tracking, configuration, or API. Triggers on "toffee", "agent detection", "agent-radar", "bot detection SDK", "add toffee", "set up toffee".
---

# Toffee SDK Integration

## Overview

Toffee detects AI agents, bots, and humans visiting your website using 6-layer client-side Bayesian detection + server-side ML classification (SAINT). It classifies visitors into 3 classes: **human**, **bot**, **agent**.

**Codebase:** `/Users/vishi/toffee/toffee-sdk` (pnpm monorepo)
**Package:** `@agent-radar/sdk` on npm
**Docs:** https://docs.toffee.at
**Dashboard:** https://db.toffee.at
**API (prod):** https://agent-radar-api.fly.dev

## Quick Reference

| Export | Purpose |
|--------|---------|
| `init(config)` | Initialize SDK, returns instance with `.destroy()` |
| `identify(metadata)` | Tag session with user metadata |
| `getDetection()` | Get latest `DetectionOutput` synchronously (or `null`) |

```typescript
interface AgentRadarConfig {
  siteId: string;           // From dashboard site settings
  apiKey: string;           // Starts with ar_, shown once on creation
  endpoint?: string;        // Default: '/api/v1/events' (use absolute URL if API is on different domain)
  debug?: boolean;          // Logs to console as [agent-radar]
  onDetection?: (result: DetectionOutput) => void;
}
```

`init()` is a **singleton** — calling it twice returns the existing instance. Call `.destroy()` before re-initializing.

## Integration: Existing Project

### 1. Get an API Key

Go to https://db.toffee.at, create a team and site. Copy the API key (starts with `ar_`). It's shown only once.

### 2. Install

```bash
npm install @agent-radar/sdk
```

### 3. Initialize

**React / Next.js (App Router):**

```tsx
'use client';
import { useEffect, useRef } from 'react';
import { init } from '@agent-radar/sdk';

export function ToffeeProvider() {
  const initialized = useRef(false);
  useEffect(() => {
    if (initialized.current) return; // Guard against strict mode double-mount
    initialized.current = true;
    const toffee = init({
      siteId: 'YOUR_SITE_ID',
      apiKey: 'ar_YOUR_KEY',
      endpoint: 'https://agent-radar-api.fly.dev/api/v1/events',
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
import { init } from '@agent-radar/sdk';
let toffee;
onMounted(() => { toffee = init({ siteId: '...', apiKey: 'ar_...', endpoint: '...' }); });
onUnmounted(() => toffee?.destroy());
</script>
```

**Script tag (IIFE):**
```html
<script src="https://cdn.toffee.at/sdk.js"></script>
<script>
  AgentRadar.init({ siteId: '...', apiKey: 'ar_...', endpoint: '...' });
</script>
```

### 4. Track Custom Elements

Auto-tracked: `h1-h6, p, a, button, form, input, img`. For anything else:

```html
<div data-ar-track="hero-banner">...</div>
<button data-ar-track="signup-cta">Sign Up</button>
```

Dynamically added elements are picked up automatically via `MutationObserver`.

### 5. Tracking Pixel (Non-JS Visitors)

For curl, wget, Python requests, CLI agents that don't execute JavaScript:

```html
<img src="https://agent-radar-api.fly.dev/api/v1/pixel?k=ar_YOUR_KEY&url=https://yoursite.com/page"
     alt="" width="1" height="1" style="position:absolute;left:-9999px" />
```

The pixel does server-side classification from HTTP headers only. Note: API key is visible in the URL (write-only, scoped to site).

### 6. Identify Users (Optional)

```typescript
import { identify } from '@agent-radar/sdk';
identify({ userId: '123', email: 'user@example.com', plan: 'pro' });
```

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

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Calling `init()` server-side (SSR) | SDK accesses `document`, `navigator`, `sessionStorage`. Always wrap in `useEffect` or `'use client'` component |
| Using relative `endpoint` with cross-domain API | Use absolute URL: `https://agent-radar-api.fly.dev/api/v1/events` |
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
