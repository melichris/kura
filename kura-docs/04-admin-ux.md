# Kura — Admin Panel UX Specification

> Every screen in /admin is documented here.
> This is the build spec for the admin CMS.
> AI agents building admin screens must read this file in full before starting.

---

## 0. Admin Shell

**Route:** `/admin`
**Auth:** Supabase Auth session required. No session → redirect to `/admin/login`.

### Layout

```
┌─────────────────────────────────────────────────────────┐
│  NAVBAR (fixed, 64px tall)                              │
│  [Kura logo]  [Portfolio preview link]  [Sign out]      │
├──────────────┬──────────────────────────────────────────┤
│              │                                          │
│  SIDEBAR     │  MAIN CONTENT AREA                       │
│  (240px)     │  (flex-1, scrollable)                    │
│              │                                          │
│  Dashboard   │                                          │
│  Profile     │                                          │
│  Projects    │                                          │
│  Sections    │                                          │
│  Experience  │                                          │
│  CV          │                                          │
│  Tokens      │                                          │
│  Settings    │                                          │
│              │                                          │
└──────────────┴──────────────────────────────────────────┘
```

### Sidebar nav items

| Label | Route | Icon |
|---|---|---|
| Dashboard | `/admin` | LayoutDashboard |
| Profile | `/admin/profile` | User |
| Projects | `/admin/projects` | Briefcase |
| Sections | `/admin/sections` | Layers |
| Experience | `/admin/experience` | Clock |
| CV | `/admin/cv` | FileText |
| Design Tokens | `/admin/tokens` | Palette |
| Settings | `/admin/settings` | Settings |

### Admin nav bar

- Left: Kura logo (small, dark variant)
- Center: breadcrumb — "Admin / {current page name}"
- Right:
  - "View portfolio →" link (opens public site in new tab)
  - Avatar + "Sign out" dropdown

### Admin color palette (separate from portfolio tokens)

Admin UI uses its own fixed palette — never inherits the user's design tokens.

```css
--admin-bg: #0f1117;
--admin-surface: #1a1f27;
--admin-border: #2a2f3a;
--admin-text: #e8eaf0;
--admin-muted: #6b7280;
--admin-accent: #c8f135;
--admin-danger: #ef4444;
--admin-success: #22c55e;
```

---

## 1. Screen: Dashboard `/admin`

**Purpose:** Quick overview of portfolio state + recent activity.

### Layout

```
┌─────────────────────────────────────────┐
│  Good morning, Meli.                    │
│  Your portfolio is [Published / Draft]  │
├────────────┬────────────┬───────────────┤
│  Projects  │  Sections  │  Active Token │
│  6 total   │  6 visible │  Dark Lime    │
│  3 featured│  1 hidden  │  Activated    │
│            │            │  2h ago       │
├────────────┴────────────┴───────────────┤
│  QUICK ACTIONS                          │
│  [+ Add project]  [Edit profile]        │
│  [Upload token]   [View portfolio]      │
├─────────────────────────────────────────┤
│  RECENT ACTIVITY (last 5 changes)       │
│  · Project "DeduSoft" updated — 2h ago  │
│  · Token "Dark Lime" activated — 2h ago │
│  · Section "Hero" edited — 1d ago       │
└─────────────────────────────────────────┘
```

### Data needed

- `portfolios` row (is_published, active_token_id)
- COUNT projects WHERE archived_at IS NULL
- COUNT projects WHERE is_featured = true
- COUNT sections WHERE is_visible = true / false
- active design_tokens row (name, created_at)

### States

- **Published:** green dot + "Live" badge next to portfolio name
- **Draft:** orange dot + "Draft" badge
- Toggle publish/unpublish button directly on dashboard

---

## 2. Screen: Profile `/admin/profile`

**Purpose:** Edit all portfolio-level identity fields.

### Sections on this page

#### 2a. Identity

| Field | Type | Validation | Maps to |
|---|---|---|---|
| Full name | text input | required, max 80 | `portfolios.display_name` |
| Tagline | text input | max 100 | `portfolios.tagline` |
| Location | text input | max 80 | `portfolios.location` |
| Short bio | textarea | max 240 chars, counter shown | `portfolios.bio_short` |
| Long bio | textarea | max 600 chars, counter shown | `portfolios.bio_long` |
| Public email | email input | valid email format | `portfolios.email` |

#### 2b. Links

| Field | Type | Validation | Maps to |
|---|---|---|---|
| GitHub URL | url input | valid URL, optional | `portfolios.github_url` |
| LinkedIn URL | url input | valid URL, optional | `portfolios.linkedin_url` |
| Personal website | url input | valid URL, optional | `portfolios.website_url` |

#### 2c. Company

| Field | Type | Validation | Maps to |
|---|---|---|---|
| Current employer name | text input | max 80, optional | `portfolios.company_name` |
| Employer URL | url input | optional | `portfolios.company_url` |
| Own company name | text input | max 80, optional | `portfolios.own_company_name` |
| Own company URL | url input | optional | `portfolios.own_company_url` |

#### 2d. Profile photo

- Upload area: drag-and-drop or click to browse
- Accept: `.jpg`, `.jpeg`, `.png`, `.webp`
- Max size: 2MB
- Preview shown after upload (circular crop preview)
- On upload: POST to `/api/upload` → Supabase Storage `avatars/` bucket
- On success: update `portfolios.avatar_url`
- Existing photo shown with "Remove" option

### Behaviour

- Each section has its own "Save" button (not one global save)
- Save → PATCH `/api/portfolio` with changed fields only
- Success: inline "Saved ✓" confirmation (not a toast — stays visible 3s)
- Error: inline field-level error messages

---

## 3. Screen: Projects `/admin/projects`

**Purpose:** Manage all portfolio projects — add, edit, reorder, archive.

### 3a. Project list view

```
┌─────────────────────────────────────────────────┐
│  Projects                        [+ Add project] │
│  6 active · 0 archived                           │
├────┬──────────────────┬────────────┬─────────────┤
│ ⠿  │ Title            │ Featured   │ Actions     │
├────┼──────────────────┼────────────┼─────────────┤
│ ⠿  │ DS  DeduSoft     │ ● Featured │ Edit Archive│
│ ⠿  │ MN  MobileNews   │ ● Featured │ Edit Archive│
│ ⠿  │ SMS SMS Platform │ ● Featured │ Edit Archive│
│ ⠿  │ PAY NEXAH Pay    │ ● Featured │ Edit Archive│
│ ⠿  │ LG  Liongate     │ ● Featured │ Edit Archive│
│ ⠿  │ PL  Paligo       │ ● Featured │ Edit Archive│
└────┴──────────────────┴────────────┴─────────────┘
│  [Show archived (0)]                             │
└──────────────────────────────────────────────────┘
```

- `⠿` = drag handle for reordering (updates `order_index` on drop)
- Featured toggle: clicking the dot toggles `is_featured`
- Plan limit warning shown if at free tier max: 
  "You're using 3/3 projects on the Free plan. Upgrade to add more."

### 3b. Project form (add / edit)

Accessed via "Edit" or "+ Add project" → opens as a full page or slide-over panel.

| Field | Type | Validation | Maps to |
|---|---|---|---|
| Abbreviation | text input | 2–3 chars, uppercase | `projects.abbreviation` |
| Company | text input | required, max 80 | `projects.company` |
| Category | text input | required, max 80 | `projects.category` |
| Project title | text input | required, max 120 | `projects.title` |
| Your role | text input | required, max 120 | `projects.role` |
| Description | textarea | required, max 400 | `projects.description` |
| Tags | tag input | 1–5 tags, max 30 chars each | `projects.tags` |
| Card gradient | gradient picker | select from 6 options | `projects.card_gradient` |
| Links | repeatable link rows | label + url pairs, max 3 | `projects.links` |
| Featured | toggle | boolean | `projects.is_featured` |

**Gradient picker UI:** 6 small gradient swatches (green, orange, blue, purple, teal, gold).
Clicking one selects it. The card preview on the right updates live.

**Card preview:** Right side of the form shows a live preview of the project card
as it will appear on the portfolio. Updates on every field change.

**Links field:** Each link row has two inputs — Label and URL — plus a remove button.
"+ Add link" button adds a new row. Max 3 links.

### Save behaviour

- POST `/api/projects` (new) or PATCH `/api/projects/:id` (edit)
- Validation errors shown inline under each field
- Success: redirect to project list + "Project saved" toast

### Archive

- Clicking "Archive" → confirmation modal:
  "Archive this project? It will be removed from your portfolio but not deleted."
- On confirm: PATCH `/api/projects/:id` with `{ archived_at: new Date() }`

---

## 4. Screen: Sections `/admin/sections`

**Purpose:** Control which sections appear on the portfolio, their order,
and their content.

### 4a. Section list

```
┌──────────────────────────────────────────────────┐
│  Sections                                        │
│  Drag to reorder. Toggle to show/hide.           │
├────┬──────────────────┬──────────┬───────────────┤
│ ⠿  │ Section          │ Visible  │ Actions       │
├────┼──────────────────┼──────────┼───────────────┤
│ ⠿  │ 🏠 Hero          │ ● ON     │ Edit          │
│ ⠿  │ 👤 About         │ ● ON     │ Edit          │
│ ⠿  │ 💼 Selected Work │ ● ON     │ Edit          │
│ ⠿  │ ⚙  How I Work   │ ● ON     │ Edit          │
│ ⠿  │ 📋 Experience    │ ● ON     │ Edit          │
│ ⠿  │ 📬 Contact       │ ● ON     │ Edit          │
└────┴──────────────────┴──────────┴───────────────┘
│  [+ Add custom section]                          │
└──────────────────────────────────────────────────┘
```

- Drag handle reorders: saves new `order_index` values on drop
- Visible toggle: PATCH `/api/sections/:id` → `{ is_visible: !current }`
- "Edit" opens the section content editor for that section type

### 4b. Section content editors

Each section type has a tailored form. Documented per type:

#### Hero section editor

| Field | Type | Maps to `content` key |
|---|---|---|
| Label text | text input | `label` |
| Headline line 1 | text input | `headline_1` |
| Headline line 2 (italic) | text input | `headline_2_italic` |
| Headline line 3 | text input | `headline_3` |
| Subtext | textarea | `subtext` |
| Primary button label | text input | `cta_primary_label` |
| Primary button URL | text input | `cta_primary_url` |
| Secondary button label | text input | `cta_secondary_label` |
| Secondary button URL | text input | `cta_secondary_url` |
| Bottom tags | repeatable text (3 items) | `bottom_tags` |
| Photo overlay text | text input | `photo_overlay` |

#### About section editor

| Field | Type | Maps to `content` key |
|---|---|---|
| Section label | text input | `section_label` |
| Headline | text input | `headline` |
| Headline italic subtitle | text input | `headline_italic` |
| Body paragraph 1 | textarea | `body_1` |
| Body paragraph 2 | textarea | `body_2` |
| GitHub link label | text input | `github_link_label` |
| GitHub link URL | url input | `github_link_url` |

> Note: body_2 is auto-populated from `portfolios.company_name` and
> `portfolios.own_company_name` if left blank. Can be overridden here.

#### How I Work section editor

4 skill columns, each with:

| Field | Type | Maps to `content` key |
|---|---|---|
| Column 1 heading | text input | `col_1_heading` |
| Column 1 skills | textarea (comma-separated) | `col_1_skills` |
| Column 2 heading | text input | `col_2_heading` |
| Column 2 skills | textarea | `col_2_skills` |
| Column 3 heading | text input | `col_3_heading` |
| Column 3 skills | textarea | `col_3_skills` |
| Column 4 heading | text input | `col_4_heading` |
| Column 4 skills | textarea | `col_4_skills` |
| Body text (right side) | textarea | `body` |

#### Contact section editor

| Field | Type | Maps to `content` key |
|---|---|---|
| Section label | text input | `section_label` |
| Headline line 1 | text input | `headline_1` |
| Headline line 2 (italic) | text input | `headline_2_italic` |
| Body text | textarea | `body` |
| Display email | email input | `email` |
| Extra links | repeatable (label + url) max 3 | `extra_links` |

#### Custom section editor

For sections of type `custom`:

| Field | Type |
|---|---|
| Section label | text input |
| Headline | text input |
| Body (rich text) | markdown textarea with preview |
| Layout | select: "text-only", "two-column", "centered" |

### Save behaviour

- PATCH `/api/sections/:id` with `{ content: { ...updatedFields } }`
- Merge strategy: app layer merges new content into existing JSONB
  (don't overwrite unedited keys)
- Success: "Section saved ✓" inline confirmation

---

## 5. Screen: Experience `/admin/experience`

**Purpose:** Manage work history and education entries.

### Layout

Two tabs: **Work** | **Education**

Each tab shows a reorderable list of entries.

### Entry card (work)

```
┌──────────────────────────────────────────────┐
│ ⠿  2026 to present                           │
│    Lead Software Engineer                    │
│    LIONGATE SARL                             │
│    Leading architecture, engineering...      │
│                              [Edit] [Delete] │
└──────────────────────────────────────────────┘
```

### Entry form (work)

| Field | Type | Validation | Maps to |
|---|---|---|---|
| Job title | text input | required, max 120 | `experience.title` |
| Company | text input | required, max 120 | `experience.organization` |
| Location | text input | optional, max 80 | `experience.location` |
| Start year | text input | required, 4 digits | `experience.year_start` |
| End year | text input | optional, 4 digits | `experience.year_end` |
| Currently here | toggle | boolean | `experience.is_current` |
| Description | textarea | max 300 | `experience.description` |

> When "Currently here" is ON, end year field is hidden and
> `year_end` is set to NULL.

### Entry form (education)

| Field | Type | Validation | Maps to |
|---|---|---|---|
| Degree / qualification | text input | required, max 120 | `experience.title` |
| Institution | text input | required, max 120 | `experience.organization` |
| Start year | text input | required | `experience.year_start` |
| End year | text input | optional | `experience.year_end` |
| In progress | toggle | boolean | `experience.is_current` |
| Notes | textarea | optional, max 200 | `experience.description` |

### Save behaviour

- POST `/api/experience` (new) or PATCH `/api/experience/:id` (edit)
- DELETE `/api/experience/:id` with confirmation modal

---

## 6. Screen: CV `/admin/cv`

**Purpose:** Upload and manage CV PDF versions.

### Layout

```
┌──────────────────────────────────────────┐
│  CV                                      │
│  Upload your CV as a PDF.                │
│  The latest active version will appear   │
│  as a download link on your portfolio.   │
├──────────────────────────────────────────┤
│  ACTIVE VERSION                          │
│  📄 meli-christian-cv-2026.pdf           │
│  Uploaded 3 days ago · 142KB             │
│  [Download] [Deactivate]                 │
├──────────────────────────────────────────┤
│  UPLOAD NEW VERSION                      │
│  ┌──────────────────────────────┐        │
│  │  Drag PDF here or click      │        │
│  │  Max 5MB · PDF only          │        │
│  └──────────────────────────────┘        │
├──────────────────────────────────────────┤
│  VERSION HISTORY (last 5)                │
│  📄 meli-cv-2025.pdf — Nov 2025 [Restore]│
│  📄 meli-cv-2024.pdf — Jan 2025 [Restore]│
└──────────────────────────────────────────┘
```

### Upload flow

1. User selects PDF → client validates: type = `application/pdf`, size ≤ 5MB
2. POST `/api/cv` with FormData
3. Server: upload to `cvs/` bucket → INSERT into `media` table
4. Mark previous CV `is_active = false`, new one `is_active = true`
5. Update `portfolios.cv_url` to new public URL
6. Show new CV as active version

### Restore

Clicking "Restore" on a version history item:
- Sets that `media` row `is_active = true`
- Sets all others `is_active = false`
- Updates `portfolios.cv_url`

---

## 7. Screen: Design Tokens `/admin/tokens`

**Purpose:** Upload, preview, and activate design token themes.

### Layout

```
┌──────────────────────────────────────────────────┐
│  Design Tokens                                   │
│  Upload a token JSON file to re-theme your       │
│  portfolio instantly.                            │
│  [Download starter token file]                   │
├──────────────────────────────────────────────────┤
│  ACTIVE THEME                                    │
│  🎨 Dark Lime Theme                              │
│  Activated 2 hours ago                           │
│  [View token file] [Replace theme]               │
├──────────────────────────────────────────────────┤
│  UPLOAD NEW THEME                                │
│  ┌──────────────────────────────┐                │
│  │  Drag .json here or click    │                │
│  │  Max 100KB · JSON only       │                │
│  └──────────────────────────────┘                │
│  Token name: [________________]                  │
│  [Upload & Preview]                              │
├──────────────────────────────────────────────────┤
│  LIVE PREVIEW                   [hidden by default]
│  ┌──────────────────────────────┐                │
│  │  iframe: public portfolio    │                │
│  │  with preview CSS injected   │                │
│  └──────────────────────────────┘                │
│  [✓ Activate this theme] [✕ Discard]            │
├──────────────────────────────────────────────────┤
│  THEME HISTORY                                   │
│  🎨 Minimal Light — Nov 2025  [Preview] [Restore]│
│  🎨 Dark Blue — Oct 2025      [Preview] [Restore]│
└──────────────────────────────────────────────────┘
```

### Upload & preview flow

1. User drops `.json` file → client reads + parses JSON
   - Invalid JSON → immediate client-side error, no upload
2. User sets token name (auto-populated from `$name` field if present)
3. Click "Upload & Preview" → POST `/api/tokens`
   - 400 response → show field-level errors under the upload area
   - 201 response → receive `{ id, preview_css }`
4. Preview panel reveals: iframe loads public portfolio URL
5. postMessage to iframe: `{ type: 'KURA_PREVIEW_CSS', css: preview_css }`
   - Public portfolio layout listens for this message in development/preview mode
   - Injects CSS into a `<style id="kura-preview">` tag
6. User reviews live preview
7. Click "Activate" → PATCH `/api/tokens/:id/activate`
   - Toast: "Theme activated. Your portfolio is now live with the new theme."
8. Click "Discard" → DELETE `/api/tokens/:id` (removes the unactivated record)

### Validation error display

If the uploaded JSON fails Zod validation, errors appear inline:

```
✕ Upload failed — invalid token file

  colors.bg.value → Token value cannot be empty
  $version        → Version must be semver: 1.0.0
```

---

## 8. Screen: Settings `/admin/settings`

**Purpose:** Domain, slug, plan, danger zone.

### 8a. Portfolio settings

| Field | Type | Notes |
|---|---|---|
| Portfolio slug | text input | URL slug, validated format, unique check on save |
| Custom domain | text input | Pro plan only — greyed out with upgrade CTA on free |
| Published | toggle | Toggles `portfolios.is_published` |

### 8b. Plan & billing (Phase 4)

Shows current plan, usage, upgrade/downgrade options.

```
Free Plan
─────────────────────────────
Projects:  3 / 3  ████████░░
Sections:  6 / 6  ████████░░
Custom domain: ✗

[Upgrade to Pro — $X/month]
```

### 8c. Account

| Action | Behaviour |
|---|---|
| Change email | Supabase Auth email change flow |
| Change password | Supabase Auth password reset |
| Sign out | Clear session, redirect to `/admin/login` |

### 8d. Danger zone

```
┌──────────────────────────────────────────────┐
│  ⚠ Danger Zone                               │
│                                              │
│  Delete portfolio                            │
│  Permanently deletes all your content.       │
│  This cannot be undone.                      │
│  [Delete my portfolio]                       │
│                                              │
│  Delete account                              │
│  Removes your account and all data.          │
│  [Delete my account]                         │
└──────────────────────────────────────────────┘
```

Both actions require typing "DELETE" to confirm.

---

## 9. Admin Login `/admin/login`

```
┌──────────────────────────────┐
│         [Kura logo]          │
│                              │
│    Sign in to your admin     │
│                              │
│  [Continue with GitHub]      │
│                              │
│       ── or ──               │
│                              │
│  Email: [__________________] │
│  [Send magic link]           │
│                              │
│  You'll receive a sign-in    │
│  link in your inbox.         │
└──────────────────────────────┘
```

- GitHub OAuth → Supabase Auth → callback `/api/auth/callback` → `/admin`
- Magic link → Supabase Auth → email sent confirmation screen → user clicks link → `/admin`
- If user has no portfolio yet → redirect to `/admin/onboarding`

---

## 10. Onboarding Flow `/admin/onboarding`

Runs once after first sign-in. 3 steps.

```
Step 1: Who are you?
  → Full name, tagline, location, email
  → Creates portfolios row + sets slug from name

Step 2: What do you do?
  → Short bio (hero subtext)
  → Long bio (about section)
  → GitHub URL, LinkedIn URL

Step 3: You're set.
  → "Your portfolio is ready. It's currently in draft mode."
  → [Go to dashboard] [View portfolio preview]
  → Seeds default sections from template
```

---

## 11. Global Admin UI Patterns

### Confirmation modals

Used for: archive project, delete entry, delete portfolio, delete account.

```
┌──────────────────────────────┐
│  Archive "DeduSoft"?         │
│                              │
│  This project will be hidden │
│  from your portfolio.        │
│                              │
│  [Cancel]  [Archive]         │
└──────────────────────────────┘
```

### Toast notifications

Position: bottom-right. Auto-dismiss after 4 seconds.

- Success: dark surface + green left border + ✓ icon
- Error: dark surface + red left border + ✕ icon
- Info: dark surface + lime left border + ℹ icon

### Form states

| State | Behaviour |
|---|---|
| Idle | Normal field appearance |
| Dirty | Save button becomes active |
| Saving | Button shows spinner, fields disabled |
| Saved | Inline "Saved ✓" for 3s, button returns to idle |
| Error | Field border turns red, error message below field |

### Empty states

Every list (projects, experience, tokens) shows an empty state when empty:

```
┌──────────────────────────────┐
│                              │
│    No projects yet.          │
│    Add your first project    │
│    to get started.           │
│                              │
│    [+ Add project]           │
│                              │
└──────────────────────────────┘
```

### Plan limit banners

Shown at the top of a page when user is at plan limit:

```
⚠ You've reached the Free plan limit for projects (3/3).
  Upgrade to Pro for unlimited projects. [Upgrade →]
```
