# Kura — Deployment

> Two-stage deployment strategy:
> Phase 1–3: Vercel (zero-config, fast iteration)
> Phase 4–5: VPS via Docker + Coolify (cost control at scale)

---

## 0. Environment Strategy

Three environments. Each has its own Supabase project and env vars.

| Environment | Branch | URL | Supabase project |
|---|---|---|---|
| Development | local | `localhost:3000` | kura-dev |
| Preview | any PR | Vercel preview URL | kura-dev (shared) |
| Production | `main` | `kura.dev` / custom | kura-prod |

**Rule:** Never point local dev at the production Supabase project.

---

## 1. Environment Variables

### `.env.local` (development — never committed)

```env
# Supabase — dev project
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGci...   # server-only

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_ROOT_DOMAIN=localhost:3000
DEFAULT_PORTFOLIO_SLUG=melichris        # for local single-tenant testing
ADMIN_EMAIL=melichris.work@gmail.com   # locks /admin to this email in v1

# Stripe (use test keys locally)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### `.env.example` (committed — template for other devs / agents)

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# App
NEXT_PUBLIC_APP_URL=
NEXT_PUBLIC_ROOT_DOMAIN=
DEFAULT_PORTFOLIO_SLUG=
ADMIN_EMAIL=

# Stripe (Phase 4)
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
```

---

## 2. Vercel Setup (Phase 0)

### Steps

1. Push repo to GitHub: `github.com/melichris/kura`
2. Import project in Vercel dashboard
3. Framework preset: **Next.js** (auto-detected)
4. Root directory: `.` (monorepo root)
5. Add all production env vars from list above
6. Set build command: `next build` (default)
7. Set output: `standalone` (add to `next.config.ts`):

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  output: 'standalone',
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: '*.supabase.co' },
    ],
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
        ],
      },
    ]
  },
}

export default config
```

### Vercel domain config

```
kura.dev          → production (main branch)
www.kura.dev      → redirect to kura.dev
*.kura.dev        → wildcard (for user subdomains — Phase 4)
```

Wildcard subdomain requires Vercel Pro plan. For Phase 1 (single user),
only `kura.dev` is needed. Add wildcard when Phase 4 begins.

### Auto-deploy rules

| Branch | Deploy | Environment |
|---|---|---|
| `main` | ✅ Auto-deploy | Production |
| `develop` | ✅ Auto-deploy | Preview (develop.kura.vercel.app) |
| `feature/*` | ✅ Auto-deploy | PR preview URL |

---

## 3. Supabase Setup (Phase 0)

### Steps

```bash
# Install Supabase CLI
npm install -g supabase

# Login
supabase login

# Link to remote project
supabase link --project-ref <project-ref>

# Run migrations on remote
supabase db push

# Generate TypeScript types from schema
supabase gen types typescript --linked > lib/types/database.types.ts
```

### Storage bucket setup

Run in Supabase dashboard → Storage → New bucket:

| Bucket | Public | Purpose |
|---|---|---|
| `avatars` | ✅ Public | Profile photos |
| `tokens` | ❌ Private | Design token JSON files |
| `cvs` | ❌ Private | CV PDFs |
| `assets` | ✅ Public | Section images, screenshots |

### Auth providers (Supabase dashboard → Auth → Providers)

Enable:
- **Email** (magic link) — enabled by default
- **GitHub** — requires GitHub OAuth app:
  - Homepage URL: `https://kura.dev`
  - Callback URL: `https://kura.dev/api/auth/callback`

---

## 4. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml

name: CI

on:
  pull_request:
    branches: [develop, main]

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx tsc --noEmit

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint

  build:
    runs-on: ubuntu-latest
    needs: [typecheck, lint]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
        env:
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
          NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}
          NEXT_PUBLIC_APP_URL: https://kura.dev
```

**PRs are blocked from merging if CI fails.**

---

## 5. VPS Migration (Phase 5)

### Why migrate

At SaaS scale, Vercel's pricing becomes significant.
A VPS gives full infrastructure control and predictable costs.

### Target stack

| Component | Technology |
|---|---|
| Server | Hetzner CX21 (2 vCPU, 4GB RAM) — ~€6/month |
| Container orchestration | Docker Compose |
| Deploy UI | Coolify (self-hosted Heroku alternative) |
| Reverse proxy | Traefik (built into Coolify) or nginx |
| SSL | Let's Encrypt (auto via Traefik/Certbot) |
| DB | Supabase remains hosted (don't self-host Postgres) |

### Docker setup

```dockerfile
# Dockerfile

FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package*.json ./
RUN npm ci

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS runner
ENV NODE_ENV=production
ENV PORT=3000

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml

version: '3.8'

services:
  kura:
    build: .
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_SUPABASE_URL=${NEXT_PUBLIC_SUPABASE_URL}
      - NEXT_PUBLIC_SUPABASE_ANON_KEY=${NEXT_PUBLIC_SUPABASE_ANON_KEY}
      - SUPABASE_SERVICE_ROLE_KEY=${SUPABASE_SERVICE_ROLE_KEY}
      - NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}
      - NEXT_PUBLIC_ROOT_DOMAIN=${NEXT_PUBLIC_ROOT_DOMAIN}
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.kura.rule=Host(`kura.dev`) || HostRegexp(`{subdomain:[a-z0-9-]+}.kura.dev`)"
      - "traefik.http.routers.kura.tls=true"
      - "traefik.http.routers.kura.tls.certresolver=letsencrypt"
```

### Zero-downtime deploy script

```bash
#!/bin/bash
# deploy.sh — run on VPS or via Coolify webhook

set -e

echo "→ Pulling latest image..."
docker compose pull

echo "→ Starting new container..."
docker compose up -d --no-deps --build kura

echo "→ Waiting for health check..."
sleep 5
curl -f http://localhost:3000/api/health || exit 1

echo "→ Deploy complete."
```

Add a health check route:

```typescript
// app/api/health/route.ts
export function GET() {
  return Response.json({ status: 'ok', ts: Date.now() })
}
```

### nginx config for custom domains (VPS)

Each Pro user domain gets an nginx server block.
In Phase 5, a background job generates and reloads nginx config
when a custom domain is verified:

```nginx
# /etc/nginx/sites-available/kura-custom-domains.conf
# Auto-generated — do not edit manually

server {
  listen 443 ssl;
  server_name melichris.dev;

  ssl_certificate     /etc/letsencrypt/live/melichris.dev/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/melichris.dev/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

---

## 6. Migration Checklist (Vercel → VPS)

Run this checklist before cutting DNS to VPS:

- [ ] VPS provisioned and SSH access working
- [ ] Docker + Docker Compose installed on VPS
- [ ] Coolify installed and dashboard accessible
- [ ] All env vars set in Coolify project
- [ ] Docker image builds successfully on VPS
- [ ] Container starts and health check passes
- [ ] Supabase Auth callback URL updated to new domain
- [ ] Stripe webhook URL updated to new domain
- [ ] Wildcard SSL cert issued for `*.kura.dev`
- [ ] nginx config generated for all existing custom domains
- [ ] DNS TTL lowered to 60s (24h before cutover)
- [ ] Smoke test on staging URL before DNS switch
- [ ] DNS switched, old Vercel deployment kept live for 48h as fallback
- [ ] Monitor error logs for 24h post-cutover
- [ ] Remove Vercel project after 7-day stability window

---

## 7. Monitoring (Phase 5)

| Tool | Purpose | Cost |
|---|---|---|
| Coolify built-in | Container health, restarts | Free |
| Uptime Kuma | External uptime monitoring | Free (self-hosted) |
| Loki + Grafana | Log aggregation + dashboards | Free (self-hosted) |
| Sentry | Error tracking (Next.js integration) | Free tier |
