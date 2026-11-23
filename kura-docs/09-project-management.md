# Kura — Project Management & Engineering Workflow

> This document is the operating manual for building Kura.
> Every AI agent, every session, every contributor reads this first.

---

## 0. Golden Rules

1. **Documentation before code.** No feature gets built without a spec doc and a ticket.
2. **No phase 2 work while phase 1 gates are open.** Discipline over speed.
3. **Every decision gets a rationale.** Not just what — why.
4. **Minimal diffs over full rewrites.** Change only what needs changing.
5. **Every session ends with the build log updated.** No exceptions.

---

## 1. Project Identity

| Field        | Value                                       |
| ------------ | ------------------------------------------- |
| Product name | Kura                                        |
| Repo name    | `kura`                                      |
| Owner        | Meli Christian                              |
| Contact      | melichris.work@gmail.com                    |
| GitHub       | github.com/melichris                        |
| Stack        | Next.js 15 · TypeScript · Supabase · Vercel |
| Docs folder  | `kura-docs/` (this folder)                  |

---

## 2. Documentation Index

Every document below must exist and be up to date before the corresponding
phase begins. An AI agent must read all relevant docs before starting any task.

| File                       | Status    | Purpose                                  |
| -------------------------- | --------- | ---------------------------------------- |
| `00-overview.md`           | ✅ Done   | Vision, phases, decisions                |
| `01-architecture.md`       | ✅ Done   | Stack, folder structure, data flows      |
| `02-database.md`           | ⏳ Next   | Full schema + RLS policies               |
| `03-design-tokens.md`      | ⏳ Queued | Token format + engine spec               |
| `04-admin-ux.md`           | ⏳ Queued | Every admin screen, field, state         |
| `05-sections.md`           | ⏳ Queued | Section types + content schema           |
| `06-saas-plan.md`          | ⏳ Queued | Multi-tenant, billing, plans, limits     |
| `07-build-log.md`          | ⏳ Queued | Ticket-by-ticket progress log            |
| `08-deployment.md`         | ⏳ Queued | Vercel setup → VPS migration             |
| `09-project-management.md` | ✅ Done   | This file — workflow, conventions, gates |

---

## 3. Build Phases

Each phase has a hard Definition of Done (DoD). No next phase starts
until every DoD checkbox is checked, deployed, and smoke-tested.

### Phase 0 — Foundation

**Goal:** Everything needed to start coding exists in writing.

DoD:

- [ ] All docs 00–09 written and reviewed
- [ ] GitHub repo created with correct folder structure
- [ ] Supabase project created (dev + prod)
- [ ] Vercel project linked to repo
- [ ] `.env.example` committed
- [ ] `develop` and `main` branches created
- [ ] Initial DB migration file written

### Phase 1 — Public Portfolio

**Goal:** Meli's portfolio is live, DB-driven, fully responsive.

DoD:

- [ ] All 6 sections render from DB data
- [ ] Design tokens inject correctly from active token record
- [ ] Responsive on mobile (<768px)
- [ ] Deployed to Vercel on `main`
- [ ] Lighthouse score: Performance ≥ 90, Accessibility ≥ 95
- [ ] No console errors in production build

### Phase 2 — Admin CMS

**Goal:** Every piece of portfolio content is editable via `/admin`.

DoD:

- [ ] Auth gate working (magic link + GitHub OAuth)
- [ ] Profile editor: photo, bio, links
- [ ] Project manager: add, edit, reorder, archive
- [ ] Section manager: toggle visibility, reorder, edit content
- [ ] Experience + education manager
- [ ] CV upload with version history
- [ ] All changes reflect on public site within 60 seconds

### Phase 3 — Design Token Engine

**Goal:** Upload a JSON token file, preview it live, activate it.

DoD:

- [ ] Token upload validates against Zod schema, returns field-level errors
- [ ] generator.ts produces valid `:root {}` CSS from any valid token file
- [ ] Live preview renders in admin iframe before activation
- [ ] Activating a token updates the public site on next request
- [ ] Previous tokens stored and restorable
- [ ] Example token file documented in `03-design-tokens.md`

### Phase 4 — SaaS Layer

**Goal:** Other developers can create their own Kura portfolio.

DoD:

- [ ] User registration + onboarding flow
- [ ] Subdomain routing working (`username.kura.dev`)
- [ ] RLS enforced — users can only access their own data
- [ ] Free plan limits enforced (3 projects, no custom domain)
- [ ] Stripe integration: checkout, webhooks, plan upgrades
- [ ] Pro plan unlocks: unlimited projects + custom domain

### Phase 5 — VPS Migration

**Goal:** Kura runs self-hosted with full infra control.

DoD:

- [ ] Docker Compose file for full stack
- [ ] Coolify or nginx config documented
- [ ] Zero-downtime deploy process documented
- [ ] Custom domain support working via nginx proxy
- [ ] Monitoring + logging in place

---

## 4. Work Breakdown Structure

```
Epic
 └── Feature
      └── Ticket (KURA-XXX)
           └── PR (feature/KURA-XXX-description)
```

### Ticket ID format

```
KURA-001, KURA-002, KURA-003 ...
```

Sequential, never reused. Tracked in `07-build-log.md`.

### Ticket file convention

Every ticket is a dedicated markdown file saved at:

```
kura-docs/tickets/KURA-XXX.md
```

Never write a ticket only in chat. The file is the source of truth.
The agent reads the file directly from the repo.

### kura-docs folder structure

```
kura-docs/                        ← system docs — always at root
├── 00-overview.md
├── 01-architecture.md
├── 02-database.md
├── 03-design-tokens.md
├── 04-admin-ux.md
├── 05-sections.md
├── 06-saas-plan.md
├── 07-build-log.md
├── 08-deployment.md
├── 09-project-management.md
├── 10-agent-workflow.md
└── tickets/                      ← one file per ticket
    ├── KURA-001.md
    ├── KURA-002.md
    └── ...
```

System docs = architecture, schema, specs, workflow. Never deleted, only updated.
Tickets = one per feature, written before the agent builds it.

### Ticket template

Every ticket is written in this format before work begins:

```markdown
### KURA-XXX: [Short title]

**Phase:** [0–5]
**Epic:** [e.g. Admin CMS]

---

#### Summary

One sentence describing what this ticket delivers.

---

#### User Story

As a [role],
I want [capability],
so that [outcome].

---

#### Context / Background

Why does this ticket exist? What problem does it solve?
What does an AI agent need to know about the codebase before starting?

---

#### Detailed Description

Exact scope of this ticket.

**Out of scope:**

- List anything that could be confused as in-scope but isn't.

---

#### Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

---

#### Technical Notes

- File paths affected
- Patterns to follow
- Edge cases to handle
- Gotchas specific to this codebase

---

#### Test Plan

| Scenario   | Steps | Expected result |
| ---------- | ----- | --------------- |
| Happy path | ...   | ...             |
| Edge case  | ...   | ...             |
| Error case | ...   | ...             |

---

#### Dependencies

- Blocked by: KURA-XXX
- Blocks: KURA-XXX
- Related to: KURA-XXX

---

#### Risks & Open Points

- Risk or question that needs resolution

---

#### Definition of Done

- [ ] Code written and working
- [ ] Edge cases handled
- [ ] No TypeScript errors
- [ ] No console errors
- [ ] PR opened against `develop`
- [ ] Build log updated

---

#### Estimate

- Base: X hours
- Buffer: +30 min
- **Total: X hours 30 min**

#### Commit message

`feat(scope): short description`

#### PR title

`feat(scope): short description — KURA-XXX`
```

---

## 5. Ticket Analysis Protocol (for AI agents)

When an AI agent receives a ticket, it must follow this exact sequence:

```
1. READ   → all docs listed in Documentation Index relevant to this ticket
2. WHY    → state why this ticket exists and what it unblocks
3. WHAT   → list every file that will be created or modified
4. HOW    → step-by-step implementation plan before writing any code
5. BUILD  → write the code following the patterns in the docs
6. VERIFY → check every acceptance criterion explicitly
7. LOG    → note completion in 07-build-log.md
```

**Never skip step 1.** An agent that starts coding without reading the docs
will produce code that conflicts with the architecture.

---

## 6. Branching Strategy

```
main          → production (Vercel auto-deploy on push)
develop       → integration (all features merge here first)
feature/KURA-XXX-short-description
fix/KURA-XXX-short-description
chore/KURA-XXX-short-description
docs/KURA-XXX-short-description
```

### Rules

- PRs always go `feature/* → develop`
- `develop → main` only on phase gate (all DoD met)
- Branch names are lowercase, hyphen-separated
- Never commit directly to `main` or `develop`
- Delete feature branches after merge

---

## 7. Commit Convention

```
feat(scope): short description        # new feature
fix(scope): short description         # bug fix
chore(scope): short description       # config, deps, tooling, scripts
docs(scope): short description        # documentation only
refactor(scope): short description    # no behaviour change
test(scope): short description        # tests only
style(scope): short description       # formatting, no logic change
```

### Scope examples

```
feat(admin): add project form validation
fix(tokens): handle empty CSS variable in generator
chore(deps): upgrade supabase-js to 2.x
docs(schema): add RLS policy comments
refactor(sections): extract content renderer to separate component
```

### Rules

- Subject line: max 72 characters, imperative mood, no period
- Never use `update`, `change`, `misc`, or `wip` as the description
- Reference ticket in body when relevant: `Closes KURA-XXX`

---

## 8. PR Template

Every PR follows this structure:

```markdown
## Summary

What does this PR do in 1–2 sentences?

## Related Ticket

Linked to KURA-XXX

## Type of Change

- [ ] `feat(scope): short summary`

## Changes Made

- File or component: what changed and why

## Acceptance Criteria

- [ ] Criterion from ticket
- [ ] Criterion from ticket

## Screenshots / Visual Evidence

(attach if UI changed)

## Test Plan

| Scenario | Steps | Expected result | Pass/Fail |
| -------- | ----- | --------------- | --------- |

## Risks & Additional Notes

Any deployment notes, follow-up tickets, or known limitations.
```

---

## 9. Ticket Queue (Phase 0 → Phase 1)

Ordered. Do not reorder without updating this table.

| ID       | Title                                                 | Phase | Status    |
| -------- | ----------------------------------------------------- | ----- | --------- |
| KURA-001 | Scaffold Next.js monorepo + tooling                   | 0     | ⏳ Queued |
| KURA-002 | Supabase project setup + initial migration            | 0     | ⏳ Queued |
| KURA-003 | Vercel project + env vars + CI                        | 0     | ⏳ Queued |
| KURA-004 | Public portfolio layout + design tokens CSS injection | 1     | ⏳ Queued |
| KURA-005 | HeroSection component (DB-driven)                     | 1     | ⏳ Queued |
| KURA-006 | AboutSection component                                | 1     | ⏳ Queued |
| KURA-007 | WorkSection + ProjectCard components                  | 1     | ⏳ Queued |
| KURA-008 | ExperienceSection component                           | 1     | ⏳ Queued |
| KURA-009 | ContactSection + FooterSection                        | 1     | ⏳ Queued |
| KURA-010 | Seed DB with Meli's portfolio data                    | 1     | ⏳ Queued |
| KURA-011 | Responsive pass + Lighthouse audit                    | 1     | ⏳ Queued |

> Phase 2+ tickets are defined after Phase 1 gate is passed.

---

## 10. Session Protocol

Every working session (human or AI agent) must:

**At session start:**

1. Read `07-build-log.md` — know the current state
2. Read the docs relevant to today's ticket
3. Confirm the ticket is unblocked before starting

**During session:**

- Follow the ticket analysis protocol (section 5)
- Raise blockers immediately, do not work around them silently
- Prefer minimal diffs — do not refactor outside the ticket scope

**At session end:**

1. Update `07-build-log.md` with what was completed
2. Note any open decisions or blockers discovered
3. Confirm next ticket in queue

---

## 11. AI Agent Context Prompt

When starting a new AI agent session on Kura, always open with:

```
You are working on Kura — a design-token-driven portfolio CMS and SaaS platform.

Before doing anything:
1. Read 00-overview.md — product vision and decisions
2. Read 01-architecture.md — stack, folder structure, data flows
3. Read 09-project-management.md — this workflow document
4. Read 07-build-log.md — current state and last completed ticket
5. Read the doc relevant to today's ticket (e.g. 02-database.md for schema work)

Then read the ticket. Follow the ticket analysis protocol:
WHY → WHAT → HOW → BUILD → VERIFY → LOG

Stack: Next.js 15 App Router · TypeScript strict · Tailwind CSS v4 ·
Supabase (Auth + Postgres + Storage) · Zod · Vercel

Conventions:
- All components in components/ — portfolio/ for public, admin/ for CMS, ui/ for shared
- All DB access via lib/supabase/server.ts on server, lib/supabase/client.ts on client
- All API validation via Zod before any DB call
- Minimal diffs — only change what the ticket requires
- No console.log in production code
- TypeScript strict — no `any`
```

---

## 12. Definition of a Shippable Ticket

A ticket is not done until ALL of the following are true:

- [ ] All acceptance criteria in the ticket are met
- [ ] TypeScript compiles with zero errors (`tsc --noEmit`)
- [ ] No ESLint errors
- [ ] No `console.log` statements left in
- [ ] No `any` types introduced
- [ ] The feature works in both development and production build
- [ ] PR is opened with the standard template
- [ ] `07-build-log.md` is updated
