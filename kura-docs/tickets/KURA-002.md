# KURA-002: Supabase project setup + initial migrations

**Phase:** 0 — Foundation
**Epic:** Foundation
**Estimate:** 2h 30m (2h base + 30 min buffer)
**Status:** Queued

---

### Summary

Create and configure the Supabase dev project, write all database migration
files in the correct order, and generate TypeScript types from the schema.
No data is seeded in this ticket — that is KURA-012.

---

### User Story

As a developer working on Kura,
I want a fully migrated Supabase database with RLS enforced on every table,
so that all subsequent tickets can read and write data against a real schema.

---

### Context / Background

The full schema is defined in `kura-docs/02-database.md`. That document
is the single source of truth — the agent must read it in full before
writing a single migration file. Do not invent column names, types, or
policies that are not in that document.

Two Supabase projects exist:

- `kura-dev` — used for development and PR previews (this ticket targets this one)
- `kura-prod` — production, never touched until Phase 1 gate is passed

---

### Detailed Description

1. Install Supabase CLI and link to the `kura-dev` project
2. Write migration files in `supabase/migrations/` in the exact order below
3. Run `supabase db push` against `kura-dev` to apply migrations
4. Verify all tables, indexes, and RLS policies exist in the Supabase dashboard
5. Generate TypeScript types and write to `lib/types/database.types.ts`
6. Update type stub files in `lib/types/` to import from `database.types.ts`

**Out of scope:**

- No seed data (KURA-012)
- No Storage bucket creation (KURA-002 is DB only)
- No auth provider configuration (KURA-016)
- Do not touch `kura-prod` under any circumstances

---

### Migration Files

Create these files in `supabase/migrations/` in this exact order.
Each file must be prefixed with a timestamp: `YYYYMMDDHHmmss_name.sql`
Use a fixed base timestamp of `20261120000001` incrementing by 1 per file.

| File                               | Contents                                                            |
| ---------------------------------- | ------------------------------------------------------------------- |
| `20261120000001_plans.sql`         | `plans` table + seed rows (free, pro)                               |
| `20261120000002_users.sql`         | `users` table + `update_updated_at` trigger function + auth trigger |
| `20261120000003_portfolios.sql`    | `portfolios` table + trigger + indexes                              |
| `20261120000004_sections.sql`      | `sections` table + trigger + indexes                                |
| `20261120000005_projects.sql`      | `projects` table + trigger + indexes                                |
| `20261120000006_experience.sql`    | `experience` table + trigger + indexes                              |
| `20261120000007_design_tokens.sql` | `design_tokens` table + partial unique index                        |
| `20261120000008_portfolios_fk.sql` | `ALTER TABLE portfolios ADD CONSTRAINT fk_active_token`             |
| `20261120000009_media.sql`         | `media` table + indexes                                             |
| `20261120000010_rls.sql`           | ALL RLS policies for every table                                    |

**All SQL for each file is in `kura-docs/02-database.md` — copy exactly,
do not rewrite or simplify.**

---

### Acceptance Criteria

- [ ] All 10 migration files exist in `supabase/migrations/`
- [ ] `supabase db push` runs with zero errors against `kura-dev`
- [ ] All tables visible in Supabase dashboard: `users`, `plans`, `portfolios`, `sections`, `projects`, `experience`, `design_tokens`, `media`
- [ ] RLS is enabled on every table (visible in dashboard → Authentication → Policies)
- [ ] `plans` table has 2 rows: `free` and `pro`
- [ ] `update_updated_at` trigger function exists and is attached to all relevant tables
- [ ] Auth trigger `on_auth_user_created` exists on `auth.users`
- [ ] `lib/types/database.types.ts` generated and committed
- [ ] No `any` types in generated or hand-written type files
- [ ] `npx tsc --noEmit` passes after type generation

---

### Technical Notes

**Supabase CLI commands:**

```bash
# Install CLI (if not already installed)
npm install -g supabase

# Login
supabase login

# Link to kura-dev project (get project ref from Supabase dashboard)
supabase link --project-ref <kura-dev-project-ref>

# Apply migrations
supabase db push

# Generate TypeScript types
supabase gen types typescript --linked > lib/types/database.types.ts
```

**Type stub updates** — after generating `database.types.ts`,
update `lib/types/portfolio.ts`, `tokens.ts`, `sections.ts`, `user.ts`
to re-export relevant types from `database.types.ts`:

```typescript
// lib/types/user.ts
import type { Database } from "./database.types";
export type User = Database["public"]["Tables"]["users"]["Row"];
export type Portfolio = Database["public"]["Tables"]["portfolios"]["Row"];

// lib/types/sections.ts
import type { Database } from "./database.types";
export type Section = Database["public"]["Tables"]["sections"]["Row"];
export type SectionInsert = Database["public"]["Tables"]["sections"]["Insert"];
export type SectionUpdate = Database["public"]["Tables"]["sections"]["Update"];

// lib/types/portfolio.ts
import type { Database } from "./database.types";
export type Project = Database["public"]["Tables"]["projects"]["Row"];
export type Experience = Database["public"]["Tables"]["experience"]["Row"];
export type Media = Database["public"]["Tables"]["media"]["Row"];

// lib/types/tokens.ts
import type { Database } from "./database.types";
export type DesignToken = Database["public"]["Tables"]["design_tokens"]["Row"];
export type DesignTokenInsert =
  Database["public"]["Tables"]["design_tokens"]["Insert"];
```

**`.env.local` required before running CLI:**

```env
NEXT_PUBLIC_SUPABASE_URL=https://<kura-dev-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<kura-dev-anon-key>
SUPABASE_SERVICE_ROLE_KEY=<kura-dev-service-role-key>
```

These are never committed. `.env.local` is already in `.gitignore`.

**Important:** The `fk_active_token` constraint in `20261120000008_portfolios_fk.sql`
must come AFTER `20261120000007_design_tokens.sql` — the FK references
`design_tokens.id` which must exist first.

---

### Test Plan

| Scenario         | Steps                                 | Expected Result                 |
| ---------------- | ------------------------------------- | ------------------------------- |
| Migrations apply | `supabase db push`                    | Zero errors, all tables created |
| Tables exist     | Supabase dashboard → Table Editor     | 8 tables visible                |
| RLS active       | Dashboard → Authentication → Policies | Policies on every table         |
| Plans seeded     | Query `SELECT * FROM plans`           | 2 rows: free, pro               |
| Auth trigger     | Dashboard → Database → Functions      | `handle_new_user` exists        |
| Types generated  | `lib/types/database.types.ts` exists  | File present, no TS errors      |
| TypeScript       | `npx tsc --noEmit`                    | Zero errors                     |
| CI               | Push branch, open PR to `develop`     | All 3 checks pass               |

---

### Dependencies

- Blocked by: KURA-001 ✅
- Blocks: KURA-003, KURA-004, all data-dependent tickets
- Related to: `kura-docs/02-database.md` (source of truth for all SQL)

---

### Risks & Open Points

- Migration order is critical — the FK in file 8 depends on file 7
- Do not use `supabase db reset` on a linked project — it wipes all data
- Generated `database.types.ts` may be large — commit as-is, do not edit manually
- If a migration fails midway, fix the SQL and re-run `supabase db push` — it only applies unapplied migrations

---

### Definition of Done

- [ ] All acceptance criteria checked
- [ ] `supabase db push` passed with zero errors
- [ ] `npx tsc --noEmit` passes
- [ ] `npm run lint` passes
- [ ] PR opened against `develop` using PR template
- [ ] CI passes on the PR
- [ ] `kura-docs/07-build-log.md` updated

---

### Commit Message

```
chore(db): add Supabase migrations and generate TypeScript types

Refs: #kura-002
```

### PR Title

```
chore(db): Supabase migrations + TypeScript types — KURA-002
```
