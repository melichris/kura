# Kura — Product Overview

> Design-token-driven portfolio CMS and SaaS platform for developers.

---

## Vision

Kura is a portfolio platform built in two phases:

**Phase 1 — Personal CMS**
Meli Christian's own portfolio site, fully content-managed through a private
admin panel. Every section, project, skill, and style choice lives in the
database. No code changes needed to update the site.

**Phase 2 — SaaS Platform**
Other developers get their own Kura portfolio. Each user gets a subdomain
(`username.kura.dev`) or a custom domain. The same admin panel, the same
design token engine, scoped per user via Supabase Row Level Security.

---

## Core Differentiator — The Design Token Engine

A developer uploads a JSON file following a simplified W3C Design Token format.
Kura parses, validates, and stores it. The system generates a `:root {}` CSS
block server-side and injects it into the public site via the Next.js layout.

The result: any section, color, font, spacing — fully swappable without
touching code. Live preview in the admin before publishing.

---

## Architecture Decisions

| Concern             | Decision                                              | Rationale                                               |
|---------------------|-------------------------------------------------------|---------------------------------------------------------|
| Framework           | Next.js 15 (App Router)                               | SSR + SSG, API routes, Vercel-native                    |
| Monorepo structure  | Single Next.js repo — /app + /admin + /api            | Simpler for v1; no Turborepo overhead yet               |
| Auth                | Supabase Auth                                         | Free, Postgres-backed, works with Vercel, RLS-ready     |
| Database            | Supabase Postgres                                     | RLS perfect for multi-tenant from day 1                 |
| Storage             | Supabase Storage                                      | Images, design token files, CV uploads in one place     |
| Styling             | Tailwind CSS + CSS custom properties (design tokens)  | Utility-first + runtime theming                         |
| Deployment v1       | Vercel                                                | Zero-config Next.js, fast iteration                     |
| Deployment final    | VPS (Docker Compose / Coolify)                        | Cost control at scale, full infra ownership             |
| Monetization        | Free tier + paid plans via Stripe                     | Free: 3 projects, no custom domain / Pro: unlimited     |

---

## Product Phases

### Phase 1 — Meli's Portfolio (v1)

Deliverables:
- Public portfolio at `/` — SSR, fully DB-driven
- Admin panel at `/admin` — private, auth-gated
- Design token upload → live re-theming
- Section manager: add, reorder, toggle visibility
- Project manager: add, edit, archive, reorder
- CV upload with version history
- Profile editor: photo, bio, links, social
- Experience and education manager
- Responsive public site matching the agreed design

### Phase 2 — SaaS (v2)

Deliverables:
- Multi-tenant routing via subdomain middleware
- User registration + onboarding flow
- Plan enforcement (free/pro limits in middleware + API)
- Stripe billing integration
- Template gallery (Meli's design = Template 001)
- Custom domain support (CNAME + Vercel domain API or VPS nginx)
- Basic analytics per portfolio (views, section clicks)

---

## Naming

| Item         | Value                  |
|--------------|------------------------|
| Product name | **Kura**               |
| Repo name    | `kura`                 |
| v1 URL       | `melichris.dev` (TBD)  |
| SaaS URL     | `kura.dev` (TBD)       |

---

## Documentation Index

| File                  | Contents                                      |
|-----------------------|-----------------------------------------------|
| `00-overview.md`      | This file — vision, decisions, phases         |
| `01-architecture.md`  | Stack, folder structure, data flow            |
| `02-database.md`      | Full schema + RLS policies                    |
| `03-design-tokens.md` | Token engine spec + JSON format               |
| `04-admin-ux.md`      | Admin panel UX spec, flows, wireframes        |
| `05-sections.md`      | Section types + content schema (jsonb)        |
| `06-saas-plan.md`     | Multi-tenant, billing, plans, limits          |
| `07-build-log.md`     | Ticket-by-ticket progress log                 |
| `08-deployment.md`    | Vercel setup → VPS migration plan             |
