# Kura — Design Token Engine

> The design token engine is Kura's core differentiator.
> A developer uploads one JSON file. The entire portfolio re-themes instantly.
> No code changes. No redeployment. Live preview before activation.

---

## 0. How It Works — End to End

```
Developer uploads token.json
        │
        ▼
POST /api/tokens
        │
        ├── validator.ts   → Zod parse → field-level errors if invalid
        │
        ├── parser.ts      → normalize to internal TokenMap
        │
        ├── generator.ts   → produce :root { --color-bg: #0b0e12; ... } string
        │
        ├── Supabase Storage → store raw JSON in tokens/ bucket
        │
        └── INSERT design_tokens (raw_json, generated_css, is_active=false)
                │
                ▼
        Admin preview → inject generated_css into iframe :root
                │
                ▼
        Developer clicks "Activate"
                │
                ▼
        PATCH /api/tokens/:id/activate
                │
                ├── UPDATE design_tokens SET is_active = false (all others)
                └── UPDATE design_tokens SET is_active = true (this one)
                        │
                        ▼
                Public site picks up new CSS on next request
```

---

## 1. Token File Format

Kura uses a **simplified W3C Design Token format**.
The full W3C spec is complex — Kura's format is a strict subset
that covers everything a portfolio needs.

### Format rules
- Must be valid JSON
- Top-level keys are **token groups**: `colors`, `typography`, `spacing`,
  `gradients`, `borders`, `shadows`
- Each token has a `value` field (required) and an optional `description`
- Values are plain CSS values — no references to other tokens in v1

### Full example: `kura-dark-lime.token.json`

```json
{
  "$name": "Dark Lime Theme",
  "$description": "Kura default — dark background with lime accent",
  "$version": "1.0.0",

  "colors": {
    "bg": {
      "value": "#0b0e12",
      "description": "Primary dark background"
    },
    "bg-green": {
      "value": "#0d1a13",
      "description": "Dark green background — How I Work section"
    },
    "bg-cream": {
      "value": "#f0ece3",
      "description": "Cream background — About section"
    },
    "bg-lime": {
      "value": "#c8f135",
      "description": "Lime background — Contact section"
    },
    "text-primary": {
      "value": "#ffffff"
    },
    "text-muted": {
      "value": "#888888"
    },
    "text-dark": {
      "value": "#1a1a1a",
      "description": "Text on light backgrounds"
    },
    "accent": {
      "value": "#c8f135",
      "description": "Lime green — dots, active states, links"
    },
    "card-bg": {
      "value": "#161b1f"
    },
    "card-border": {
      "value": "#2a2f35"
    },
    "navbar-bg": {
      "value": "#0b0e12"
    },
    "navbar-border": {
      "value": "#1e2328"
    }
  },

  "gradients": {
    "card-green": {
      "value": "linear-gradient(135deg, #1a3a2a, #0d2018)",
      "description": "DeduSoft card"
    },
    "card-orange": {
      "value": "linear-gradient(135deg, #8b4a1a, #5a2e0a)",
      "description": "MobileNews card"
    },
    "card-blue": {
      "value": "linear-gradient(135deg, #1a2a3a, #0d1a28)",
      "description": "SMS Platform card"
    },
    "card-purple": {
      "value": "linear-gradient(135deg, #2a1a3a, #1a0d28)",
      "description": "NEXAH Pay card"
    },
    "card-teal": {
      "value": "linear-gradient(135deg, #1a2a2a, #0d1a1a)",
      "description": "Liongate card"
    },
    "card-gold": {
      "value": "linear-gradient(135deg, #3a2a0a, #28180a)",
      "description": "Paligo card"
    }
  },

  "typography": {
    "font-heading": {
      "value": "'Syne', sans-serif"
    },
    "font-body": {
      "value": "'Inter', sans-serif"
    },
    "font-size-hero": {
      "value": "clamp(3rem, 8vw, 7rem)",
      "description": "Hero headline"
    },
    "font-size-section-heading": {
      "value": "clamp(2rem, 4vw, 3.5rem)",
      "description": "Per-section large headings"
    },
    "font-size-contact-heading": {
      "value": "clamp(2.5rem, 6vw, 6rem)",
      "description": "Contact section giant heading"
    },
    "font-size-body": {
      "value": "0.95rem"
    },
    "font-size-label": {
      "value": "0.75rem"
    },
    "font-size-card-title": {
      "value": "1.5rem"
    },
    "font-weight-heading": {
      "value": "800"
    },
    "font-weight-body": {
      "value": "400"
    },
    "letter-spacing-label": {
      "value": "0.15em"
    }
  },

  "spacing": {
    "section-padding-y": {
      "value": "120px",
      "description": "Vertical padding on all sections"
    },
    "section-padding-x": {
      "value": "40px",
      "description": "Horizontal padding — collapses on mobile"
    },
    "content-max-width": {
      "value": "1200px"
    },
    "card-padding": {
      "value": "24px"
    },
    "card-radius": {
      "value": "16px"
    },
    "navbar-height": {
      "value": "64px"
    },
    "avatar-radius": {
      "value": "24px"
    }
  },

  "borders": {
    "card-border-width": {
      "value": "1px"
    },
    "card-border-style": {
      "value": "solid"
    },
    "navbar-border-width": {
      "value": "1px"
    },
    "button-border-radius": {
      "value": "9999px",
      "description": "Pill shape"
    },
    "tag-border-color": {
      "value": "#3a3a3a"
    }
  },

  "shadows": {
    "card-hover": {
      "value": "0 8px 32px rgba(0, 0, 0, 0.3)"
    }
  }
}
```

---

## 2. CSS Custom Property Mapping

The generator maps each token to a CSS custom property using this convention:

```
colors.bg              →  --color-bg
colors.text-primary    →  --color-text-primary
colors.card-bg         →  --color-card-bg
gradients.card-green   →  --gradient-card-green
typography.font-heading →  --font-heading
typography.font-size-hero → --font-size-hero
spacing.section-padding-y → --spacing-section-padding-y
borders.card-radius    →  --border-card-radius
shadows.card-hover     →  --shadow-card-hover
```

**Naming convention:** `--{group-singular}-{token-key}`

Groups map to prefixes:
```
colors     → --color-
gradients  → --gradient-
typography → --font-   (font-* keys) or --font-size-* / --font-weight-*
spacing    → --spacing-
borders    → --border-
shadows    → --shadow-
```

### Generated `:root {}` output

```css
:root {
  /* Colors */
  --color-bg: #0b0e12;
  --color-bg-green: #0d1a13;
  --color-bg-cream: #f0ece3;
  --color-bg-lime: #c8f135;
  --color-text-primary: #ffffff;
  --color-text-muted: #888888;
  --color-text-dark: #1a1a1a;
  --color-accent: #c8f135;
  --color-card-bg: #161b1f;
  --color-card-border: #2a2f35;
  --color-navbar-bg: #0b0e12;
  --color-navbar-border: #1e2328;

  /* Gradients */
  --gradient-card-green: linear-gradient(135deg, #1a3a2a, #0d2018);
  --gradient-card-orange: linear-gradient(135deg, #8b4a1a, #5a2e0a);
  --gradient-card-blue: linear-gradient(135deg, #1a2a3a, #0d1a28);
  --gradient-card-purple: linear-gradient(135deg, #2a1a3a, #1a0d28);
  --gradient-card-teal: linear-gradient(135deg, #1a2a2a, #0d1a1a);
  --gradient-card-gold: linear-gradient(135deg, #3a2a0a, #28180a);

  /* Typography */
  --font-heading: 'Syne', sans-serif;
  --font-body: 'Inter', sans-serif;
  --font-size-hero: clamp(3rem, 8vw, 7rem);
  --font-size-section-heading: clamp(2rem, 4vw, 3.5rem);
  --font-size-contact-heading: clamp(2.5rem, 6vw, 6rem);
  --font-size-body: 0.95rem;
  --font-size-label: 0.75rem;
  --font-size-card-title: 1.5rem;
  --font-weight-heading: 800;
  --font-weight-body: 400;
  --letter-spacing-label: 0.15em;

  /* Spacing */
  --spacing-section-padding-y: 120px;
  --spacing-section-padding-x: 40px;
  --spacing-content-max-width: 1200px;
  --spacing-card-padding: 24px;
  --spacing-card-radius: 16px;
  --spacing-navbar-height: 64px;
  --spacing-avatar-radius: 24px;

  /* Borders */
  --border-card-border-width: 1px;
  --border-card-border-style: solid;
  --border-navbar-border-width: 1px;
  --border-button-border-radius: 9999px;
  --border-tag-border-color: #3a3a3a;

  /* Shadows */
  --shadow-card-hover: 0 8px 32px rgba(0, 0, 0, 0.3);
}
```

---

## 3. How Components Consume Tokens

Every public portfolio component uses CSS custom properties exclusively.
Never hardcode a color, font, or size in a component — always reference a token.

```css
/* ✅ Correct */
.section {
  background: var(--color-bg);
  font-family: var(--font-body);
  padding: var(--spacing-section-padding-y) var(--spacing-section-padding-x);
}

.project-card {
  background: var(--color-card-bg);
  border: var(--border-card-border-width) var(--border-card-border-style) var(--color-card-border);
  border-radius: var(--spacing-card-radius);
}

.project-card:hover {
  border-color: var(--color-accent);
  box-shadow: var(--shadow-card-hover);
}

/* ❌ Never do this */
.section {
  background: #0b0e12;
  font-family: 'Inter', sans-serif;
}
```

---

## 4. Validator Schema (Zod)

```typescript
// lib/tokens/validator.ts

import { z } from 'zod'

const TokenValue = z.object({
  value: z.string().min(1, 'Token value cannot be empty'),
  description: z.string().optional(),
})

const ColorGroup = z.record(z.string(), TokenValue)
const GradientGroup = z.record(z.string(), TokenValue)
const TypographyGroup = z.record(z.string(), TokenValue)
const SpacingGroup = z.record(z.string(), TokenValue)
const BorderGroup = z.record(z.string(), TokenValue)
const ShadowGroup = z.record(z.string(), TokenValue)

export const TokenFileSchema = z.object({
  $name: z.string().min(1).max(80),
  $description: z.string().max(300).optional(),
  $version: z.string().regex(/^\d+\.\d+\.\d+$/, 'Version must be semver: 1.0.0').optional(),

  colors: ColorGroup.optional(),
  gradients: GradientGroup.optional(),
  typography: TypographyGroup.optional(),
  spacing: SpacingGroup.optional(),
  borders: BorderGroup.optional(),
  shadows: ShadowGroup.optional(),
}).refine(
  (data) => {
    // At least one token group must be present
    const groups = ['colors', 'gradients', 'typography', 'spacing', 'borders', 'shadows']
    return groups.some((g) => data[g as keyof typeof data] !== undefined)
  },
  { message: 'Token file must contain at least one token group' }
)

export type TokenFile = z.infer<typeof TokenFileSchema>
```

**Validation error response shape:**

```json
{
  "error": "Invalid token file",
  "issues": [
    {
      "path": ["colors", "bg", "value"],
      "message": "Token value cannot be empty"
    },
    {
      "path": ["$version"],
      "message": "Version must be semver: 1.0.0"
    }
  ]
}
```

---

## 5. Parser

```typescript
// lib/tokens/parser.ts

import type { TokenFile } from './validator'

export type TokenMap = Map<string, string>
// Key: CSS property name (e.g. "--color-bg")
// Value: CSS value (e.g. "#0b0e12")

const GROUP_PREFIX: Record<string, string> = {
  colors: '--color',
  gradients: '--gradient',
  typography: '--font',
  spacing: '--spacing',
  borders: '--border',
  shadows: '--shadow',
}

export function parseTokenFile(file: TokenFile): TokenMap {
  const map: TokenMap = new Map()

  for (const [group, prefix] of Object.entries(GROUP_PREFIX)) {
    const tokens = file[group as keyof TokenFile]
    if (!tokens || typeof tokens !== 'object') continue

    for (const [key, token] of Object.entries(tokens)) {
      if (typeof token === 'object' && 'value' in token) {
        const cssVar = `${prefix}-${key}`
        map.set(cssVar, token.value)
      }
    }
  }

  return map
}
```

---

## 6. Generator

```typescript
// lib/tokens/generator.ts

import type { TokenMap } from './parser'

const DEFAULT_CSS = `
:root {
  --color-bg: #0b0e12;
  --color-accent: #c8f135;
  --color-text-primary: #ffffff;
  --color-text-muted: #888888;
  --font-heading: 'Syne', sans-serif;
  --font-body: 'Inter', sans-serif;
}
`.trim()

export function generateRootCSS(tokenMap: TokenMap): string {
  if (tokenMap.size === 0) return DEFAULT_CSS

  const declarations = Array.from(tokenMap.entries())
    .map(([prop, value]) => `  ${prop}: ${value};`)
    .join('\n')

  return `:root {\n${declarations}\n}`
}

export { DEFAULT_CSS }
```

---

## 7. API Route: `POST /api/tokens`

```typescript
// app/api/tokens/route.ts

import { NextRequest, NextResponse } from 'next/server'
import { createServerClient } from '@/lib/supabase/server'
import { TokenFileSchema } from '@/lib/tokens/validator'
import { parseTokenFile } from '@/lib/tokens/parser'
import { generateRootCSS } from '@/lib/tokens/generator'

export async function POST(req: NextRequest) {
  const supabase = createServerClient()

  // 1. Auth check
  const { data: { user }, error: authError } = await supabase.auth.getUser()
  if (authError || !user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // 2. Parse request body
  const body = await req.json()

  // 3. Validate
  const parsed = TokenFileSchema.safeParse(body.tokenFile)
  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Invalid token file', issues: parsed.error.issues },
      { status: 400 }
    )
  }

  const tokenFile = parsed.data

  // 4. Parse to TokenMap → generate CSS
  const tokenMap = parseTokenFile(tokenFile)
  const generatedCSS = generateRootCSS(tokenMap)

  // 5. Get user's portfolio
  const { data: portfolio } = await supabase
    .from('portfolios')
    .select('id')
    .eq('user_id', user.id)
    .single()

  if (!portfolio) {
    return NextResponse.json({ error: 'Portfolio not found' }, { status: 404 })
  }

  // 6. Upload raw JSON to Supabase Storage
  const fileName = `${user.id}/${Date.now()}-${tokenFile.$name.replace(/\s+/g, '-')}.json`
  await supabase.storage
    .from('tokens')
    .upload(fileName, JSON.stringify(tokenFile), { contentType: 'application/json' })

  // 7. Insert record (not active yet)
  const { data: token, error: insertError } = await supabase
    .from('design_tokens')
    .insert({
      portfolio_id: portfolio.id,
      user_id: user.id,
      name: tokenFile.$name,
      raw_json: tokenFile,
      generated_css: generatedCSS,
      storage_path: fileName,
      is_active: false,
    })
    .select()
    .single()

  if (insertError) {
    return NextResponse.json({ error: 'Failed to save token' }, { status: 500 })
  }

  // 8. Return token id + preview CSS
  return NextResponse.json({
    id: token.id,
    name: token.name,
    preview_css: generatedCSS,
  }, { status: 201 })
}
```

---

## 8. API Route: `PATCH /api/tokens/[id]/activate`

```typescript
// app/api/tokens/[id]/activate/route.ts

export async function PATCH(
  _req: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const { data: portfolio } = await supabase
    .from('portfolios')
    .select('id')
    .eq('user_id', user.id)
    .single()

  if (!portfolio) return NextResponse.json({ error: 'Not found' }, { status: 404 })

  // Deactivate all tokens for this portfolio
  await supabase
    .from('design_tokens')
    .update({ is_active: false })
    .eq('portfolio_id', portfolio.id)

  // Activate the requested token
  const { error } = await supabase
    .from('design_tokens')
    .update({ is_active: true })
    .eq('id', params.id)
    .eq('portfolio_id', portfolio.id) // RLS + extra safety check

  if (error) return NextResponse.json({ error: 'Activation failed' }, { status: 500 })

  // Update portfolio.active_token_id for fast JOIN in public layout
  await supabase
    .from('portfolios')
    .update({ active_token_id: params.id })
    .eq('id', portfolio.id)

  return NextResponse.json({ success: true })
}
```

---

## 9. Admin UI: Token Upload Flow

The `TokenUploader` admin component does the following:

```
1. File picker → accept .json only, max 100KB
2. Client-side: read file, JSON.parse()
   → show parse error immediately if invalid JSON
3. POST /api/tokens with { tokenFile: parsedJSON }
   → on 400: show field-level validation errors inline
   → on 201: receive { id, preview_css }
4. Inject preview_css into admin iframe via postMessage
   → iframe renders public portfolio with new theme
   → user sees live preview without activating
5. User reviews preview
   → "Activate" → PATCH /api/tokens/:id/activate
   → "Discard" → DELETE /api/tokens/:id (optional)
6. On activation: toast "Theme activated" + public site updates
```

---

## 10. Required Token Keys (Minimum Valid File)

A token file is valid as long as it passes Zod schema validation and
contains at least one group. However, missing tokens fall back to
CSS custom property defaults defined in the global stylesheet.

**Recommended minimum for a complete re-theme:**

```json
{
  "$name": "My Theme",
  "colors": {
    "bg": { "value": "..." },
    "accent": { "value": "..." },
    "text-primary": { "value": "..." },
    "text-muted": { "value": "..." },
    "text-dark": { "value": "..." }
  },
  "typography": {
    "font-heading": { "value": "..." },
    "font-body": { "value": "..." }
  }
}
```

With just these 7 tokens, the entire site re-themes coherently.
Every other token falls back to its default value.

---

## 11. File Delivery for Developers

In the admin panel, under Settings or the Tokens page, provide:

- A downloadable `kura-dark-lime.token.json` starter file
- A link to this documentation page
- An inline JSON schema viewer showing all supported token keys
- Validation feedback with the exact path to any invalid field

The goal: a developer with their own design system can map their
tokens to Kura's format in under 30 minutes.
