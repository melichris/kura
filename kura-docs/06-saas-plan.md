# Kura — SaaS Plan

> Phase 4 work. Nothing in this document gets built until Phase 1–3
> gates are fully closed. Documented now so architecture decisions
> in Phase 0–3 don't create blockers later.

---

## 0. SaaS Model Summary

| Dimension | Decision |
|---|---|
| Multi-tenancy | Subdomain per user: `username.kura.dev` |
| Custom domains | Pro plan only: user brings their own domain |
| Auth | Supabase Auth — same system, scoped by RLS |
| Billing | Stripe — subscriptions, webhooks, portal |
| Plan enforcement | API layer (not DB constraints) |
| Template system | User's portfolio design = Template 001 |

---

## 1. Plans

### Free

| Limit | Value |
|---|---|
| Projects | 3 |
| Sections | 6 (all built-in types) |
| Design tokens | 3 stored (1 active) |
| Custom domain | ✗ |
| Analytics | ✗ |
| CV uploads | 1 active |
| Portfolio subdomain | `username.kura.dev` |
| Branding | "Built with Kura" footer badge |

### Pro

| Limit | Value |
|---|---|
| Projects | Unlimited |
| Sections | Unlimited |
| Design tokens | Unlimited stored |
| Custom domain | ✓ (1 domain) |
| Analytics | ✓ (page views, section clicks) |
| CV uploads | Unlimited (version history) |
| Portfolio subdomain | `username.kura.dev` (or custom) |
| Branding | Badge removed |
| Price | $9/month or $90/year |

---

## 2. Multi-Tenant Routing

### How it works

All traffic hits the same Next.js deployment. The `middleware.ts` reads
the hostname and injects a header that scopes all data fetching.

```
username.kura.dev  →  X-Kura-User: username
melichris.dev      →  X-Kura-User: melichris  (custom domain lookup)
kura.dev           →  serve marketing site
kura.dev/admin     →  serve admin (scoped to auth session)
```

### Middleware logic

```typescript
// middleware.ts

import { NextRequest, NextResponse } from 'next/server'

const ROOT_DOMAIN = 'kura.dev'

export async function middleware(req: NextRequest) {
  const hostname = req.headers.get('host') ?? ''
  const url = req.nextUrl.clone()

  // 1. Admin routes — handled by Supabase Auth, no subdomain logic needed
  if (url.pathname.startsWith('/admin')) {
    return handleAdminAuth(req)
  }

  // 2. Root domain — marketing site
  if (hostname === ROOT_DOMAIN || hostname === `www.${ROOT_DOMAIN}`) {
    return NextResponse.next()
  }

  // 3. Subdomain — username.kura.dev
  if (hostname.endsWith(`.${ROOT_DOMAIN}`)) {
    const username = hostname.replace(`.${ROOT_DOMAIN}`, '')
    const res = NextResponse.next()
    res.headers.set('X-Kura-User', username)
    return res
  }

  // 4. Custom domain — look up in portfolios table
  const supabase = createServerClient()
  const { data: portfolio } = await supabase
    .from('portfolios')
    .select('slug')
    .eq('custom_domain', hostname)
    .single()

  if (portfolio) {
    const res = NextResponse.next()
    res.headers.set('X-Kura-User', portfolio.slug)
    return res
  }

  // 5. Unknown domain
  return NextResponse.redirect(new URL('/', `https://${ROOT_DOMAIN}`))
}
```

### Portfolio layout reads the header

```typescript
// app/(portfolio)/layout.tsx

import { headers } from 'next/headers'

export default async function PortfolioLayout({ children }) {
  const headersList = headers()
  const slug = headersList.get('X-Kura-User') ?? process.env.DEFAULT_PORTFOLIO_SLUG

  const portfolio = await getPortfolioBySlug(slug)
  if (!portfolio || !portfolio.is_published) notFound()

  const tokenCSS = portfolio.design_tokens?.generated_css ?? defaultTokenCSS

  return (
    <html>
      <head>
        <style id="kura-tokens" dangerouslySetInnerHTML={{ __html: tokenCSS }} />
      </head>
      <body>{children}</body>
    </html>
  )
}
```

---

## 3. Custom Domain Setup (Pro)

### User flow

1. User enters their domain in Settings → `melichris.dev`
2. Kura shows DNS instructions:
   ```
   Add a CNAME record:
   Name:  @  (or www)
   Value: cname.kura.dev
   ```
3. User adds the record at their DNS provider
4. Kura polls for DNS propagation (or user clicks "Verify")
5. On verify: check CNAME resolves to `cname.kura.dev`
6. Store `custom_domain = 'melichris.dev'` in portfolios table
7. Vercel automatically handles SSL via its domain API
   (or on VPS: Certbot + nginx config)

### On VPS (Phase 5)

nginx config is generated per custom domain:

```nginx
server {
  listen 80;
  server_name melichris.dev www.melichris.dev;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name melichris.dev www.melichris.dev;

  ssl_certificate     /etc/letsencrypt/live/melichris.dev/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/melichris.dev/privkey.pem;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

---

## 4. Stripe Integration

### Billing flow

```
User clicks "Upgrade to Pro"
  → POST /api/billing/checkout
      → stripe.checkout.sessions.create({
          customer: user.stripe_customer_id,
          line_items: [{ price: STRIPE_PRO_PRICE_ID, quantity: 1 }],
          mode: 'subscription',
          success_url: '/admin/settings?upgraded=true',
          cancel_url: '/admin/settings',
        })
      → return { url: checkoutUrl }
  → redirect to Stripe Checkout

User completes payment
  → Stripe fires webhook: checkout.session.completed
      → POST /api/billing/webhook
          → verify Stripe signature
          → extract customer_id, subscription_id
          → UPDATE users SET plan_id = 'pro'
          → INSERT INTO subscriptions (...)

User cancels / downgrades
  → Stripe fires webhook: customer.subscription.deleted
      → UPDATE users SET plan_id = 'free'
      → UPDATE subscriptions SET status = 'canceled'
      → enforce free plan limits on next API call
```

### Customer portal

```typescript
// POST /api/billing/portal
// Redirects user to Stripe Customer Portal for self-service billing management

const session = await stripe.billingPortal.sessions.create({
  customer: user.stripe_customer_id,
  return_url: `${process.env.NEXT_PUBLIC_APP_URL}/admin/settings`,
})

return NextResponse.redirect(session.url)
```

### Webhook events handled

| Event | Action |
|---|---|
| `checkout.session.completed` | Set plan = pro, insert subscription |
| `customer.subscription.updated` | Update subscription status/period |
| `customer.subscription.deleted` | Set plan = free, enforce limits |
| `invoice.payment_failed` | Mark subscription past_due, notify user |

---

## 5. Plan Enforcement

### Where limits are checked

Every mutating API route checks plan limits before INSERT.
Pattern is consistent — a single `enforceLimit` utility:

```typescript
// lib/plans/enforce.ts

import { createServerClient } from '@/lib/supabase/server'

type LimitKey = 'projects' | 'sections' | 'design_tokens' | 'custom_domain'

export async function enforceLimit(userId: string, limit: LimitKey) {
  const supabase = createServerClient()

  const { data: user } = await supabase
    .from('users')
    .select('plan_id')
    .eq('id', userId)
    .single()

  const { data: plan } = await supabase
    .from('plans')
    .select('*')
    .eq('id', user?.plan_id ?? 'free')
    .single()

  if (!plan) throw new Error('Plan not found')

  switch (limit) {
    case 'projects': {
      if (plan.max_projects === -1) return // unlimited
      const { count } = await supabase
        .from('projects')
        .select('id', { count: 'exact' })
        .eq('user_id', userId)
        .is('archived_at', null)
      if ((count ?? 0) >= plan.max_projects) {
        throw new PlanLimitError('projects', plan.max_projects)
      }
      break
    }
    case 'sections': {
      if (plan.max_sections === -1) return
      const { count } = await supabase
        .from('sections')
        .select('id', { count: 'exact' })
        .eq('user_id', userId)
      if ((count ?? 0) >= plan.max_sections) {
        throw new PlanLimitError('sections', plan.max_sections)
      }
      break
    }
    case 'custom_domain': {
      if (!plan.custom_domain) {
        throw new PlanLimitError('custom_domain', 0)
      }
      break
    }
  }
}

export class PlanLimitError extends Error {
  constructor(public readonly resource: string, public readonly limit: number) {
    super(`Plan limit reached for ${resource} (limit: ${limit})`)
    this.name = 'PlanLimitError'
  }
}
```

API routes catch `PlanLimitError` and return:

```json
{
  "error": "plan_limit",
  "resource": "projects",
  "limit": 3,
  "message": "Free plan allows 3 projects. Upgrade to Pro for unlimited.",
  "upgrade_url": "/admin/settings#billing"
}
```

HTTP status: `402 Payment Required`

---

## 6. Analytics (Pro only)

### What is tracked

- Portfolio page views (per day)
- Section visibility events (which sections users scroll to)
- Link clicks (project links, contact links)
- CV downloads

### How it works

Client-side: a lightweight `analytics.ts` script on the public site
fires POST requests to `/api/analytics/event`.

```typescript
// No third-party analytics — first-party only

type AnalyticsEvent =
  | { type: 'page_view' }
  | { type: 'section_view'; section_type: string }
  | { type: 'link_click'; label: string; url: string }
  | { type: 'cv_download' }

async function track(portfolioId: string, event: AnalyticsEvent) {
  // Check plan before sending — anon key read of portfolios.plan
  await fetch('/api/analytics/event', {
    method: 'POST',
    body: JSON.stringify({ portfolio_id: portfolioId, ...event }),
  })
}
```

Storage: a simple `analytics_events` table (portfolio_id, type, metadata JSONB,
created_at). Aggregated in admin dashboard for Pro users.

---

## 7. Template Gallery (Phase 4+)

Meli's portfolio design becomes **Template 001: Dark Lime**.

### How templates work

A template is a bundle of:
- A set of default section `content` values
- A design token JSON file
- A preview screenshot

When a new user signs up and picks a template:
1. Sections are seeded from the template's content defaults
2. The template's token file is uploaded as their first design token
3. The token is activated immediately
4. User sees their portfolio pre-populated with placeholder copy

### Template registry (future)

```typescript
// lib/templates/registry.ts

export const TEMPLATES = [
  {
    id: 'dark-lime',
    name: 'Dark Lime',
    description: 'High-contrast dark theme with lime accent. Clean, technical.',
    token_file: 'kura-dark-lime.token.json',
    preview_url: '/templates/dark-lime-preview.png',
  },
  // future templates added here
]
```

---

## 8. Marketing Site (`kura.dev`)

Built as separate pages under `app/(marketing)/`:

| Route | Content |
|---|---|
| `/` | Hero, features, pricing, testimonials, CTA |
| `/pricing` | Plan comparison table, FAQ |
| `/templates` | Template gallery with live previews |
| `/docs` | Token format docs, getting started guide |
| `/blog` | (future) Engineering posts |

The marketing site uses its own fixed design — not user-configurable.
