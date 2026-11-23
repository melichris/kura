# Kura — Architecture

---

## Stack

| Layer        | Technology                        | Notes                                          |
|--------------|-----------------------------------|------------------------------------------------|
| Framework    | Next.js 15 (App Router)           | SSR + SSG + API Route Handlers                 |
| Language     | TypeScript                        | Strict mode                                    |
| Styling      | Tailwind CSS v4                   | + CSS custom properties for design tokens      |
| Auth         | Supabase Auth                     | Magic link + GitHub OAuth                      |
| Database     | Supabase Postgres                 | RLS enforced on all tables                     |
| Storage      | Supabase Storage                  | Buckets: avatars, tokens, cvs, assets          |
| ORM          | Supabase JS client (typed)        | No Prisma for now — Supabase types via CLI     |
| Validation   | Zod                               | API input + design token file validation       |
| Deployment   | Vercel (v1) → VPS/Coolify (final) |                                                |

---

## Folder Structure

```
kura/
├── app/
│   ├── (portfolio)/                  # Public portfolio — route group
│   │   ├── layout.tsx                # Injects design tokens CSS from DB
│   │   ├── page.tsx                  # Renders all visible sections in order
│   │   └── work/[slug]/page.tsx      # Individual case study pages
│   │
│   ├── admin/                        # CMS — auth-gated
│   │   ├── layout.tsx                # Auth guard + admin shell UI
│   │   ├── page.tsx                  # Dashboard: quick stats + recent activity
│   │   ├── profile/
│   │   │   └── page.tsx              # Name, bio, photo, social links
│   │   ├── projects/
│   │   │   ├── page.tsx              # Project list + reorder
│   │   │   └── [id]/page.tsx         # Edit individual project
│   │   ├── sections/
│   │   │   └── page.tsx              # Section manager + per-section content editor
│   │   ├── tokens/
│   │   │   └── page.tsx              # Upload token JSON, preview, activate
│   │   ├── cv/
│   │   │   └── page.tsx              # Upload CV PDF, version history
│   │   ├── experience/
│   │   │   └── page.tsx              # Work + education entries
│   │   └── settings/
│   │       └── page.tsx              # Domain, plan, danger zone
│   │
│   └── api/                          # Route Handlers (REST-style)
│       ├── auth/callback/route.ts    # Supabase OAuth callback
│       ├── portfolio/route.ts        # GET public portfolio data
│       ├── projects/
│       │   ├── route.ts              # GET list / POST create
│       │   └── [id]/route.ts         # GET / PATCH / DELETE
│       ├── sections/
│       │   ├── route.ts
│       │   └── [id]/route.ts
│       ├── tokens/
│       │   ├── route.ts              # POST upload + validate + store
│       │   └── [id]/activate/route.ts
│       ├── cv/route.ts
│       └── upload/route.ts           # Generic signed-URL generator
│
├── components/
│   ├── portfolio/                    # Public section components (presentational)
│   │   ├── NavBar.tsx
│   │   ├── HeroSection.tsx
│   │   ├── AboutSection.tsx
│   │   ├── WorkSection.tsx
│   │   ├── ExperienceSection.tsx
│   │   ├── ContactSection.tsx
│   │   ├── FooterSection.tsx
│   │   └── ProjectCard.tsx
│   │
│   ├── admin/                        # Admin UI components
│   │   ├── SectionEditor.tsx         # Per-section content form
│   │   ├── SectionList.tsx           # Drag-to-reorder section list
│   │   ├── TokenUploader.tsx         # Upload + validate + preview token JSON
│   │   ├── LivePreview.tsx           # iframe preview of public site
│   │   ├── ProjectForm.tsx
│   │   ├── ExperienceForm.tsx
│   │   └── CVUploader.tsx
│   │
│   └── ui/                           # Shared primitives
│       ├── Button.tsx
│       ├── Input.tsx
│       ├── Badge.tsx
│       ├── Modal.tsx
│       └── Toast.tsx
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts                 # Browser client
│   │   ├── server.ts                 # Server client (cookies)
│   │   └── middleware.ts             # Auth session refresh
│   │
│   ├── tokens/
│   │   ├── parser.ts                 # Reads uploaded token JSON
│   │   ├── validator.ts              # Zod schema for token format
│   │   └── generator.ts             # Outputs :root { --color-bg: ... } string
│   │
│   └── types/
│       ├── portfolio.ts
│       ├── tokens.ts
│       ├── sections.ts
│       └── user.ts
│
├── supabase/
│   ├── migrations/                   # Versioned SQL migration files
│   │   └── 0001_initial_schema.sql
│   └── seed.ts                       # Dev seed data (Meli's portfolio)
│
├── middleware.ts                     # Auth guard + (v2) subdomain routing
├── tailwind.config.ts
├── next.config.ts
└── tsconfig.json
```

---

## Data Flow

### Public portfolio page load

```
Browser → GET /
  → Next.js layout.tsx (server)
    → supabase.server: SELECT active design_token WHERE portfolio_id = X
    → generator.ts: build :root { } CSS string
    → inject as <style> into <head>
  → page.tsx (server)
    → supabase.server: SELECT sections ORDER BY order WHERE is_visible = true
    → SELECT projects WHERE is_featured = true
    → SELECT experience, profile
  → render section components with data
  → stream HTML to browser
```

### Admin saves a project

```
Admin form submit → POST /api/projects
  → middleware: verify Supabase session (JWT)
  → Zod: validate body
  → supabase.server: INSERT INTO projects (RLS enforces user_id match)
  → return { id, ... }
  → SWR/client revalidates project list
```

### Design token upload

```
Admin uploads token.json → POST /api/tokens
  → read file buffer
  → validator.ts: Zod parse against token schema
    → error? return 400 + field-level errors
  → parser.ts: normalize to internal TokenMap
  → generator.ts: produce :root { } CSS string
  → supabase.storage: upload raw JSON to tokens/ bucket
  → INSERT INTO design_tokens (raw_json, generated_css, is_active=false)
  → return { id, preview_css }
  → admin TokenUploader shows live preview (inject CSS into iframe)
  → user clicks "Activate" → PATCH /api/tokens/:id/activate
    → UPDATE design_tokens SET is_active = false WHERE portfolio_id = X
    → UPDATE design_tokens SET is_active = true WHERE id = :id
  → public site picks up new token on next request (or ISR revalidation)
```

---

## Auth Flow

```
/admin/* routes → middleware.ts
  → supabase.server: getSession()
  → no session → redirect to /admin/login
  → session valid → allow through

/admin/login
  → Supabase magic link OR GitHub OAuth
  → callback: /api/auth/callback
    → exchange code for session
    → redirect to /admin
```

---

## Multi-tenant routing (Phase 2 — not v1)

```
middleware.ts intercepts:
  hostname = req.headers.get('host')
  if hostname === 'kura.dev' → serve marketing site
  if hostname ends with '.kura.dev' → extract username, set header X-Portfolio-User
  if custom domain → lookup in portfolios table, set X-Portfolio-User
  public portfolio layout reads X-Portfolio-User to scope all DB queries
```

---

## Environment Variables

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=        # server-only, never exposed to client

# App
NEXT_PUBLIC_APP_URL=              # https://yourdomain.com
ADMIN_EMAIL=                      # locks /admin to this email in v1

# Phase 2
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
```
