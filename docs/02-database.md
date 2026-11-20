# Kura — Database Schema & RLS Policies

> Supabase Postgres. Row Level Security enforced on every table.
> Multi-tenant from day 1 — even in Phase 1 (single user), the schema
> is shaped so Phase 4 (SaaS) requires zero breaking changes.

---

## 0. Design Principles

1. **Every row is owned.** Every content table has a `user_id` column
   referencing `auth.users`. RLS uses this to scope all queries.

2. **`portfolios` is the tenant root.** All content tables reference
   `portfolio_id`, not `user_id` directly. One user can have one portfolio
   in v1. In v2, the door is open for multiple portfolios per user.

3. **JSONB for flexible content.** Section content varies by type.
   Rather than one table per section type, `sections.content` is JSONB
   validated at the application layer (Zod). Schema evolution = no migrations.

4. **Soft deletes on critical tables.** Projects and sections use
   `archived_at` instead of hard DELETE. Data is recoverable.

5. **Timestamps on everything.** `created_at` and `updated_at` on every table.
   `updated_at` is auto-updated via trigger.

---

## 1. Schema Overview

```
auth.users (Supabase managed)
    │
    ▼
users ──────────────────────────────┐
    │                               │
    ▼                               │
portfolios                          │
    │                               │
    ├──► sections                   │
    ├──► projects                   │
    ├──► experience                 │
    ├──► design_tokens              │
    └──► media                      │
                                    │
plans ◄─────────────────────────────┘
subscriptions
```

---

## 2. Full Schema (SQL)

### Trigger: auto-update `updated_at`

```sql
-- Run once. Applied to every table that needs it.
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

### Table: `users`

Extends Supabase `auth.users`. Created automatically on first sign-in
via a Supabase Auth trigger.

```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email         TEXT NOT NULL UNIQUE,
  username      TEXT UNIQUE,                    -- used for subdomain in v2
  full_name     TEXT,
  plan_id       TEXT NOT NULL DEFAULT 'free'    -- 'free' | 'pro'
                REFERENCES plans(id),
  stripe_customer_id  TEXT UNIQUE,              -- populated on first Stripe checkout
  onboarded_at  TIMESTAMPTZ,                    -- NULL = not yet onboarded
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

**Notes:**
- `username` is validated at app layer: lowercase, alphanumeric + hyphens,
  3–30 chars, no reserved words (admin, api, www, kura, etc.)
- `plan_id` is a string FK to `plans` table, not an enum, so new plans
  can be added without schema changes.

---

### Table: `plans`

```sql
CREATE TABLE plans (
  id                    TEXT PRIMARY KEY,        -- 'free' | 'pro'
  name                  TEXT NOT NULL,
  max_projects          INT NOT NULL DEFAULT 3,  -- -1 = unlimited
  max_sections          INT NOT NULL DEFAULT 6,  -- -1 = unlimited
  custom_domain         BOOLEAN NOT NULL DEFAULT FALSE,
  analytics             BOOLEAN NOT NULL DEFAULT FALSE,
  stripe_price_id       TEXT,                    -- NULL for free plan
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Seed data
INSERT INTO plans (id, name, max_projects, max_sections, custom_domain, analytics, stripe_price_id)
VALUES
  ('free', 'Free',  3,  6,  FALSE, FALSE, NULL),
  ('pro',  'Pro',   -1, -1, TRUE,  TRUE,  'price_XXXXXXXXXXXXXXXX');
```

---

### Table: `portfolios`

The tenant root. One per user in v1, potentially many in v2.

```sql
CREATE TABLE portfolios (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  slug              TEXT NOT NULL UNIQUE,         -- URL slug: melichris, john-doe
  display_name      TEXT,                         -- overrides user full_name on site
  tagline           TEXT,                         -- "Lead Software Engineer"
  location          TEXT,                         -- "Douala, Cameroon"
  bio_short         TEXT,                         -- hero subtext (~200 chars)
  bio_long          TEXT,                         -- about section (~500 chars)
  email             TEXT,                         -- public contact email
  github_url        TEXT,
  linkedin_url      TEXT,
  website_url       TEXT,
  company_name      TEXT,                         -- current employer
  company_url       TEXT,
  own_company_name  TEXT,                         -- founder's company name
  own_company_url   TEXT,
  avatar_url        TEXT,                         -- Supabase Storage URL
  cv_url            TEXT,                         -- latest CV Storage URL
  active_token_id   UUID,                         -- FK added after design_tokens created
  is_published      BOOLEAN NOT NULL DEFAULT FALSE,
  custom_domain     TEXT UNIQUE,                  -- e.g. melichris.dev (Phase 4)
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER portfolios_updated_at
  BEFORE UPDATE ON portfolios
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE INDEX idx_portfolios_user_id ON portfolios(user_id);
CREATE INDEX idx_portfolios_slug ON portfolios(slug);
```

---

### Table: `sections`

One row per section on the portfolio. Content shape varies by type —
stored as JSONB, validated at app layer.

```sql
CREATE TABLE sections (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id  UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,  -- for RLS
  type          TEXT NOT NULL,          -- see Section Types below
  label         TEXT,                   -- custom label override (e.g. "02 / MY WORK")
  order_index   SMALLINT NOT NULL DEFAULT 0,
  is_visible    BOOLEAN NOT NULL DEFAULT TRUE,
  content       JSONB NOT NULL DEFAULT '{}',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER sections_updated_at
  BEFORE UPDATE ON sections
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE INDEX idx_sections_portfolio_id ON sections(portfolio_id);
CREATE INDEX idx_sections_order ON sections(portfolio_id, order_index);
```

**Section types** (enforced by Zod at app layer, not DB constraint):
```
hero | about | work | how_i_work | experience | contact | custom
```

**Example `content` JSONB per type** — documented fully in `05-sections.md`.

---

### Table: `projects`

```sql
CREATE TABLE projects (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id    UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  abbreviation    TEXT NOT NULL,          -- "DS", "LG", "SMS" (2–3 chars)
  company         TEXT NOT NULL,          -- "DEDUSOFT", "LIONGATE"
  category        TEXT NOT NULL,          -- "LEADERSHIP", "PRODUCTS", "WEBSITES"
  title           TEXT NOT NULL,
  role            TEXT NOT NULL,
  description     TEXT NOT NULL,
  tags            TEXT[] NOT NULL DEFAULT '{}',   -- ["Next.js", "Quality", "Web"]
  card_gradient   TEXT NOT NULL DEFAULT 'green',  -- maps to predefined gradient key
  links           JSONB NOT NULL DEFAULT '[]',
  -- [{"label": "Read case study", "url": "#"}, {"label": "Visit company", "url": "#"}]
  is_featured     BOOLEAN NOT NULL DEFAULT TRUE,   -- shows in selected work grid
  order_index     SMALLINT NOT NULL DEFAULT 0,
  archived_at     TIMESTAMPTZ,                     -- soft delete
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER projects_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE INDEX idx_projects_portfolio_id ON projects(portfolio_id);
CREATE INDEX idx_projects_featured ON projects(portfolio_id, is_featured)
  WHERE archived_at IS NULL;
```

**`card_gradient` keys** (maps to CSS gradient in design tokens):
```
green | orange | blue | purple | teal | gold
```

---

### Table: `experience`

Work history and education in one table, differentiated by `entry_type`.

```sql
CREATE TABLE experience (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id  UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  entry_type    TEXT NOT NULL CHECK (entry_type IN ('work', 'education')),
  title         TEXT NOT NULL,           -- "Lead Software Engineer" / "Master of Technology"
  organization  TEXT NOT NULL,           -- "LIONGATE SARL" / "IUG"
  location      TEXT,
  year_start    TEXT NOT NULL,           -- "2026" (TEXT to allow "2022" etc.)
  year_end      TEXT,                    -- NULL = present
  is_current    BOOLEAN NOT NULL DEFAULT FALSE,
  description   TEXT,
  order_index   SMALLINT NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER experience_updated_at
  BEFORE UPDATE ON experience
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE INDEX idx_experience_portfolio_id ON experience(portfolio_id);
CREATE INDEX idx_experience_type ON experience(portfolio_id, entry_type);
```

---

### Table: `design_tokens`

Stores uploaded token files and their generated CSS.

```sql
CREATE TABLE design_tokens (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id    UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,                -- human label: "Dark Lime Theme"
  raw_json        JSONB NOT NULL,               -- parsed token file content
  generated_css   TEXT NOT NULL,               -- :root { --color-bg: #0b0e12; ... }
  storage_path    TEXT NOT NULL,               -- Supabase Storage path
  is_active       BOOLEAN NOT NULL DEFAULT FALSE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
  -- no updated_at: tokens are immutable once created; activate or create new
);

-- Only one active token per portfolio — enforced by partial unique index
CREATE UNIQUE INDEX idx_one_active_token
  ON design_tokens(portfolio_id)
  WHERE is_active = TRUE;

CREATE INDEX idx_tokens_portfolio_id ON design_tokens(portfolio_id);
```

**After creating this table, add the FK to portfolios:**
```sql
ALTER TABLE portfolios
  ADD CONSTRAINT fk_active_token
  FOREIGN KEY (active_token_id) REFERENCES design_tokens(id)
  ON DELETE SET NULL;
```

---

### Table: `media`

All uploaded files: avatars, CV versions, section assets.

```sql
CREATE TABLE media (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  portfolio_id  UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  media_type    TEXT NOT NULL CHECK (media_type IN ('avatar', 'cv', 'asset')),
  file_name     TEXT NOT NULL,
  storage_path  TEXT NOT NULL,              -- Supabase Storage path
  public_url    TEXT NOT NULL,              -- resolved public URL
  size_bytes    INT,
  mime_type     TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,  -- for versioned files (CV)
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_media_portfolio_id ON media(portfolio_id);
CREATE INDEX idx_media_type ON media(portfolio_id, media_type);
```

---

### Table: `subscriptions` (Phase 4)

```sql
-- Created in Phase 4 migration, not Phase 0.
-- Documented here for schema planning completeness.

CREATE TABLE subscriptions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  stripe_subscription_id TEXT NOT NULL UNIQUE,
  stripe_price_id       TEXT NOT NULL,
  plan_id               TEXT NOT NULL REFERENCES plans(id),
  status                TEXT NOT NULL,   -- 'active' | 'canceled' | 'past_due'
  current_period_start  TIMESTAMPTZ NOT NULL,
  current_period_end    TIMESTAMPTZ NOT NULL,
  cancel_at_period_end  BOOLEAN NOT NULL DEFAULT FALSE,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Row Level Security (RLS)

RLS is enabled on every table. The pattern is consistent:
- SELECT: user can read their own rows
- INSERT: user can insert rows where user_id = auth.uid()
- UPDATE: user can update their own rows
- DELETE: user can delete their own rows (or soft-delete at app layer)

Public portfolio data (sections, projects, experience) also needs
an anonymous SELECT policy for the public site.

```sql
-- ================================================================
-- USERS
-- ================================================================
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own row"
  ON users FOR SELECT
  USING (id = auth.uid());

CREATE POLICY "Users can update own row"
  ON users FOR UPDATE
  USING (id = auth.uid());


-- ================================================================
-- PORTFOLIOS
-- ================================================================
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;

-- Public: anyone can read published portfolios (for public site)
CREATE POLICY "Public can read published portfolios"
  ON portfolios FOR SELECT
  USING (is_published = TRUE);

-- Owner: full access to own portfolio
CREATE POLICY "Owner can read own portfolio"
  ON portfolios FOR SELECT
  USING (user_id = auth.uid());

CREATE POLICY "Owner can insert portfolio"
  ON portfolios FOR INSERT
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "Owner can update own portfolio"
  ON portfolios FOR UPDATE
  USING (user_id = auth.uid());

CREATE POLICY "Owner can delete own portfolio"
  ON portfolios FOR DELETE
  USING (user_id = auth.uid());


-- ================================================================
-- SECTIONS
-- ================================================================
ALTER TABLE sections ENABLE ROW LEVEL SECURITY;

-- Public: read visible sections of published portfolios
CREATE POLICY "Public can read visible sections"
  ON sections FOR SELECT
  USING (
    is_visible = TRUE
    AND EXISTS (
      SELECT 1 FROM portfolios p
      WHERE p.id = sections.portfolio_id
      AND p.is_published = TRUE
    )
  );

CREATE POLICY "Owner can read own sections"
  ON sections FOR SELECT
  USING (user_id = auth.uid());

CREATE POLICY "Owner can insert sections"
  ON sections FOR INSERT
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "Owner can update own sections"
  ON sections FOR UPDATE
  USING (user_id = auth.uid());

CREATE POLICY "Owner can delete own sections"
  ON sections FOR DELETE
  USING (user_id = auth.uid());


-- ================================================================
-- PROJECTS
-- ================================================================
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public can read active featured projects"
  ON projects FOR SELECT
  USING (
    archived_at IS NULL
    AND EXISTS (
      SELECT 1 FROM portfolios p
      WHERE p.id = projects.portfolio_id
      AND p.is_published = TRUE
    )
  );

CREATE POLICY "Owner can read own projects"
  ON projects FOR SELECT
  USING (user_id = auth.uid());

CREATE POLICY "Owner can insert projects"
  ON projects FOR INSERT
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "Owner can update own projects"
  ON projects FOR UPDATE
  USING (user_id = auth.uid());

CREATE POLICY "Owner can delete own projects"
  ON projects FOR DELETE
  USING (user_id = auth.uid());


-- ================================================================
-- EXPERIENCE
-- ================================================================
ALTER TABLE experience ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public can read experience"
  ON experience FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM portfolios p
      WHERE p.id = experience.portfolio_id
      AND p.is_published = TRUE
    )
  );

CREATE POLICY "Owner can manage own experience"
  ON experience FOR ALL
  USING (user_id = auth.uid());


-- ================================================================
-- DESIGN TOKENS
-- ================================================================
ALTER TABLE design_tokens ENABLE ROW LEVEL SECURITY;

-- Public needs to read the active token to render the public site
CREATE POLICY "Public can read active token"
  ON design_tokens FOR SELECT
  USING (
    is_active = TRUE
    AND EXISTS (
      SELECT 1 FROM portfolios p
      WHERE p.id = design_tokens.portfolio_id
      AND p.is_published = TRUE
    )
  );

CREATE POLICY "Owner can manage own tokens"
  ON design_tokens FOR ALL
  USING (user_id = auth.uid());


-- ================================================================
-- MEDIA
-- ================================================================
ALTER TABLE media ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public can read active media"
  ON media FOR SELECT
  USING (
    is_active = TRUE
    AND EXISTS (
      SELECT 1 FROM portfolios p
      WHERE p.id = media.portfolio_id
      AND p.is_published = TRUE
    )
  );

CREATE POLICY "Owner can manage own media"
  ON media FOR ALL
  USING (user_id = auth.uid());
```

---

## 4. Supabase Auth Trigger

Auto-create a `users` row when a new Supabase Auth user signs up:

```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO users (id, email, full_name)
  VALUES (
    NEW.id,
    NEW.email,
    NEW.raw_user_meta_data->>'full_name'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

---

## 5. Supabase Storage Buckets

```sql
-- Run in Supabase dashboard or via management API

-- avatars: profile photos
INSERT INTO storage.buckets (id, name, public)
VALUES ('avatars', 'avatars', TRUE);

-- tokens: uploaded design token JSON files
INSERT INTO storage.buckets (id, name, public)
VALUES ('tokens', 'tokens', FALSE);   -- private; served via signed URL

-- cvs: uploaded CV PDF files
INSERT INTO storage.buckets (id, name, public)
VALUES ('cvs', 'cvs', FALSE);         -- private; owner generates download link

-- assets: section images, project screenshots
INSERT INTO storage.buckets (id, name, public)
VALUES ('assets', 'assets', TRUE);
```

**Storage RLS policies (key ones):**
```sql
-- avatars: anyone can read, only owner can upload
CREATE POLICY "Public can read avatars"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'avatars');

CREATE POLICY "Owner can upload avatar"
  ON storage.objects FOR INSERT
  WITH CHECK (
    bucket_id = 'avatars'
    AND auth.uid()::text = (storage.foldername(name))[1]
  );

-- tokens: owner only
CREATE POLICY "Owner can manage tokens"
  ON storage.objects FOR ALL
  USING (
    bucket_id = 'tokens'
    AND auth.uid()::text = (storage.foldername(name))[1]
  );
```

---

## 6. Migration File Plan

```
supabase/migrations/
├── 0001_plans.sql              -- plans table + seed data
├── 0002_users.sql              -- users table + auth trigger
├── 0003_portfolios.sql         -- portfolios table
├── 0004_sections.sql           -- sections table
├── 0005_projects.sql           -- projects table
├── 0006_experience.sql         -- experience table
├── 0007_design_tokens.sql      -- design_tokens table + portfolios FK
├── 0008_media.sql              -- media table
├── 0009_rls.sql                -- all RLS policies
├── 0010_storage.sql            -- storage buckets + policies
└── 0011_seed_meli.sql          -- Phase 0 seed: Meli's portfolio data
```

Each migration file is idempotent where possible (`CREATE IF NOT EXISTS`,
`ON CONFLICT DO NOTHING` on seeds). Never edit a committed migration —
always add a new one.

---

## 7. Plan Enforcement (App Layer)

Plan limits are **not** enforced by DB constraints — they are enforced
in API route handlers before INSERT operations.

```typescript
// lib/plans/enforce.ts (pseudocode pattern)

async function checkProjectLimit(portfolioId: string, userId: string) {
  const user = await getUser(userId)
  const plan = await getPlan(user.plan_id)

  if (plan.max_projects === -1) return // unlimited

  const count = await countProjects(portfolioId)
  if (count >= plan.max_projects) {
    throw new PlanLimitError(
      `Free plan allows ${plan.max_projects} projects. Upgrade to Pro for unlimited.`
    )
  }
}
```

This pattern is used in:
- `POST /api/projects` → check `max_projects`
- `POST /api/sections` → check `max_sections`
- `PATCH /api/portfolios/:id` (custom domain) → check `custom_domain`

---

## 8. Key Query Patterns

### Fetch full public portfolio (server component)

```typescript
// Fetches everything needed to render the public site in one pass

const { data: portfolio } = await supabase
  .from('portfolios')
  .select(`
    *,
    sections(*),
    projects(*),
    experience(*),
    design_tokens!active_token_id(generated_css)
  `)
  .eq('slug', slug)
  .eq('is_published', true)
  .single()
```

### Fetch active design token CSS only (layout.tsx)

```typescript
const { data } = await supabase
  .from('portfolios')
  .select('design_tokens!active_token_id(generated_css)')
  .eq('slug', slug)
  .single()

const css = data?.design_tokens?.generated_css ?? defaultTokenCSS
```

### Reorder sections (admin drag-and-drop)

```typescript
// PATCH /api/sections/reorder
// body: { items: [{ id: uuid, order_index: number }] }

const updates = items.map(({ id, order_index }) =>
  supabase
    .from('sections')
    .update({ order_index })
    .eq('id', id)
    .eq('user_id', userId)
)

await Promise.all(updates)
```
