# Kura — Sections Specification

> This document defines every section type Kura supports.
> For each type: the content JSONB schema, the Zod validator,
> the default seed values (Meli's portfolio), and the rendering spec.
>
> AI agents building section components must read this file
> alongside 03-design-tokens.md before writing any component code.

---

## 0. How Sections Work

Each row in the `sections` table has:
- `type` — identifies which component renders it
- `content` — JSONB object whose shape depends on `type`
- `order_index` — position on the page (ascending)
- `is_visible` — whether it renders on the public site

The public portfolio `page.tsx` does this:

```typescript
const sections = await getSections(portfolioId)
// sections is ordered by order_index, filtered by is_visible = true

return sections.map((section) => {
  switch (section.type) {
    case 'hero':        return <HeroSection key={section.id} content={section.content} portfolio={portfolio} />
    case 'about':       return <AboutSection key={section.id} content={section.content} portfolio={portfolio} />
    case 'work':        return <WorkSection key={section.id} content={section.content} projects={projects} />
    case 'how_i_work':  return <HowIWorkSection key={section.id} content={section.content} />
    case 'experience':  return <ExperienceSection key={section.id} content={section.content} experience={experience} />
    case 'contact':     return <ContactSection key={section.id} content={section.content} portfolio={portfolio} />
    case 'custom':      return <CustomSection key={section.id} content={section.content} />
    default:            return null
  }
})
```

**Key rule:** Components never fetch their own data.
All data is fetched once in `page.tsx` (server component) and passed as props.

---

## 1. Section Type: `hero`

### Content schema

```typescript
// lib/types/sections.ts

export const HeroContentSchema = z.object({
  label:               z.string().max(120).default('LEAD SOFTWARE ENGINEER, DOUALA, CAMEROON'),
  headline_1:          z.string().max(60).default('Building software'),
  headline_2_italic:   z.string().max(60).default('that moves'),
  headline_2_suffix:   z.string().max(60).default('things'),
  headline_3:          z.string().max(60).default('forward.'),
  subtext:             z.string().max(300).default(''),
  cta_primary_label:   z.string().max(40).default('Explore my work'),
  cta_primary_url:     z.string().default('#work'),
  cta_secondary_label: z.string().max(40).default('Get in touch'),
  cta_secondary_url:   z.string().default('#contact'),
  photo_overlay:       z.string().max(20).default('MC / 001'),
  bottom_tags:         z.array(z.string().max(40)).length(3).default([
    'ENGINEER FOR SCALE',
    'DESIGN FOR PEOPLE',
    'SHIP WITH CARE',
  ]),
})

export type HeroContent = z.infer<typeof HeroContentSchema>
```

### Seed data (Meli's portfolio)

```json
{
  "label": "LEAD SOFTWARE ENGINEER, DOUALA, CAMEROON",
  "headline_1": "Building software",
  "headline_2_italic": "that moves",
  "headline_2_suffix": "things",
  "headline_3": "forward.",
  "subtext": "I'm Meli Christian. I build web products and backend systems, lead engineering teams, and help ship reliable software.",
  "cta_primary_label": "Explore my work",
  "cta_primary_url": "#work",
  "cta_secondary_label": "Get in touch",
  "cta_secondary_url": "#contact",
  "photo_overlay": "MC / 001",
  "bottom_tags": [
    "ENGINEER FOR SCALE",
    "DESIGN FOR PEOPLE",
    "SHIP WITH CARE"
  ]
}
```

### Rendering spec

```
BACKGROUND: var(--color-bg)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)
PADDING-TOP: calc(var(--spacing-section-padding-y) + var(--spacing-navbar-height))
LAYOUT: two columns — left 55% / right 45%
MIN-HEIGHT: 100vh

LEFT COLUMN:
  · label row: lime dot (8px circle, --color-accent) + content.label text
      font: var(--font-body), 0.75rem uppercase, letter-spacing 0.15em
      color: var(--color-accent)
  · headline block (margin-top: 32px):
      Line 1: content.headline_1
        font: var(--font-heading), var(--font-size-hero), weight 800
        color: var(--color-text-primary)
      Line 2: content.headline_2_italic + " " + content.headline_2_suffix
        italic part: font-style italic, color: var(--color-accent)
        suffix part: color: var(--color-text-primary)
      Line 3: content.headline_3
        color: var(--color-text-primary)
        last char (period): color: var(--color-accent)
  · subtext (margin-top: 24px):
      font: var(--font-body), 1.1rem, color: var(--color-text-muted)
      max-width: 480px
  · button row (margin-top: 40px, gap: 16px):
      Primary: background var(--color-accent), color #000, font-weight 700
        border-radius: var(--border-button-border-radius)
        padding: 14px 28px
        hover: background darken 10%
      Secondary: background transparent, color var(--color-text-primary)
        border: 1.5px solid var(--color-text-primary)
        border-radius: var(--border-button-border-radius)
        padding: 14px 28px
        hover: background rgba(255,255,255,0.08)

RIGHT COLUMN:
  · photo container:
      width: 100%, max-width: 420px
      height: ~420px
      border-radius: var(--spacing-avatar-radius)
      background: #1a1f27 (placeholder bg — not a token, it's the placeholder state)
      overflow: hidden
      position: relative
      If portfolio.avatar_url: render <img> filling the container, object-fit cover
      If no avatar: render initials placeholder
        background: #1a1f27, centered initials (e.g. "MC")
        font: var(--font-heading), 4rem, color: var(--color-text-muted)
        border: 2px dashed var(--color-card-border)
  · overlay text (bottom-right of photo, absolute):
      content.photo_overlay
      font: var(--font-body), 0.75rem, color: var(--color-text-muted)
      padding: 8px 12px
  · below-photo row (margin-top: 16px, justify-content: space-between):
      left: "THINK, BUILD, IMPROVE" — 0.7rem uppercase muted
      right: "EST. DOUALA, CM" — 0.7rem uppercase muted

BOTTOM ROW (margin-top: 80px, border-top: 1px solid var(--color-card-border)):
  padding-top: 32px
  display: flex, justify-content: space-between
  content.bottom_tags rendered as 3 muted uppercase small labels
  font: var(--font-body), 0.75rem, letter-spacing 0.15em
  color: var(--color-text-muted)

MOBILE (<768px):
  Stack to single column (right column below left)
  Photo max-height: 300px
  headline font-size: clamp(2.5rem, 10vw, 4rem)
  bottom tags: flex-direction column, gap 8px
```

---

## 2. Section Type: `about`

### Content schema

```typescript
export const AboutContentSchema = z.object({
  section_label:      z.string().max(60).default('01 / A LITTLE ABOUT ME'),
  headline:           z.string().max(60).default('More than code.'),
  headline_italic:    z.string().max(60).default("A builder's mindset."),
  body_1:             z.string().max(400).default(''),
  body_2:             z.string().max(400).default(''),
  github_link_label:  z.string().max(40).default('Explore my GitHub'),
  github_link_url:    z.string().default(''),
})

export type AboutContent = z.infer<typeof AboutContentSchema>
```

### Seed data

```json
{
  "section_label": "01 / A LITTLE ABOUT ME",
  "headline": "More than code.",
  "headline_italic": "A builder's mindset.",
  "body_1": "I work from requirements through design, development, testing, and deployment. I enjoy solving complex technical problems and working with teams to build products people can rely on.",
  "body_2": "I lead software engineering at LIONGATE SARL. I'm also the founder, CEO, and technical lead at DeduSoft, where I work on products including custom digital solutions for organizations.",
  "github_link_label": "Explore my GitHub",
  "github_link_url": "https://github.com/melichris"
}
```

### Rendering spec

```
BACKGROUND: var(--color-bg-cream)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)
LAYOUT: two columns — left 40% / right 55%, gap 80px

TOP: section_label
  font: var(--font-body), var(--font-size-label), uppercase
  letter-spacing: var(--letter-spacing-label)
  color: var(--color-text-muted) (dark muted — on cream bg use #666)
  margin-bottom: 48px

LEFT COLUMN:
  · headline:
      font: var(--font-heading), clamp(2.5rem, 4vw, 3.5rem), weight 800
      color: var(--color-text-dark)
  · headline_italic (below, no margin):
      font: var(--font-heading), clamp(1.8rem, 3vw, 2.5rem), italic, weight 700
      color: #666 (muted dark — not a token, stays fixed on cream bg)

RIGHT COLUMN:
  · body_1:
      font: var(--font-body), 1.2rem, line-height 1.8
      color: var(--color-text-dark)
      margin-bottom: 24px
  · body_2:
      font: var(--font-body), 0.95rem, line-height 1.7
      color: #555 (secondary dark muted)
      margin-bottom: 32px
  · github link:
      text-decoration: underline
      text-underline-offset: 4px
      color: var(--color-text-dark)
      font-weight: 600
      hover: color: var(--color-accent) — but since bg is cream, use a dark hover (#000)

MOBILE: stack to single column, left above right
```

---

## 3. Section Type: `work`

### Content schema

```typescript
export const WorkContentSchema = z.object({
  section_label:  z.string().max(60).default('02 / SELECTED WORK'),
  headline_1:     z.string().max(80).default('Products, platforms,'),
  headline_2:     z.string().max(80).default('and real-world impact.'),
  body:           z.string().max(300).default(
    'A selection of work across entrepreneurship, engineering leadership, fintech, telecommunications, commerce, and education. Some client work remains private.'
  ),
  show_archive:   z.boolean().default(true),
})

export type WorkContent = z.infer<typeof WorkContentSchema>
```

> Projects data comes from the `projects` table, not from `content`.
> The section content only controls the section header copy.

### Rendering spec

```
BACKGROUND: var(--color-bg)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)

SECTION HEADER (two columns — left 50% / right 45%):
  LEFT:
    · section_label — same label style as all sections
    · headline_1 + headline_2:
        font: var(--font-heading), clamp(2rem, 3.5vw, 3rem), weight 800
        color: var(--color-text-primary)
        headline_2 ends with a lime dot:
          append <span style="color: var(--color-accent)">.</span>
          and remove the period from the string
  RIGHT:
    · body text:
        font: var(--font-body), 1rem, line-height 1.75
        color: var(--color-text-muted)
        margin-top: 48px (aligns roughly with headline bottom)

PROJECT GRID (margin-top: 64px):
  display: grid
  grid-template-columns: repeat(2, 1fr)
  gap: 24px
  Render <ProjectCard> for each project WHERE is_featured = true AND archived_at IS NULL
  Ordered by order_index ASC

PROJECT CARD spec (see Section 3a below)

ARCHIVE BOX (margin-top: 48px, only if show_archive = true):
  See Section 3b below

MOBILE: grid becomes single column
```

### 3a. ProjectCard rendering spec

```
CARD CONTAINER:
  background: var(--color-card-bg)
  border: 1px solid var(--color-card-border)
  border-radius: var(--spacing-card-radius)
  overflow: hidden
  transition: border-color 0.2s, transform 0.2s
  hover:
    border-color: var(--color-accent)
    transform: translateY(-4px)
    box-shadow: var(--shadow-card-hover)

CARD BANNER (top ~160px):
  background: var(--gradient-card-{project.card_gradient})
  position: relative
  display: flex, align-items: center, justify-content: center

  · Abbreviation text (project.abbreviation):
      font: var(--font-heading), 5rem, weight 800
      color: rgba(255,255,255,0.9)
      letter-spacing: -0.02em
      z-index: 2

  · Decorative shapes (pure CSS, z-index 1):
      Two rounded rectangles (outline only, rgba(255,255,255,0.08))
      One: width 120px, height 80px, border-radius 12px
        position: absolute, top: 20px, right: 20px, rotate: 15deg
      Two: width 80px, height 56px, border-radius 8px
        position: absolute, bottom: 16px, left: 24px, rotate: -10deg
      border: 1.5px solid rgba(255,255,255,0.12)

CARD BODY (padding: var(--spacing-card-padding)):
  · tag line: "{project.company} · {project.category}"
      font: var(--font-body), 0.7rem, uppercase
      letter-spacing: 0.12em
      color: var(--color-accent)
      margin-bottom: 10px

  · title (project.title):
      font: var(--font-heading), var(--font-size-card-title), weight 700
      color: var(--color-text-primary)
      margin-bottom: 6px

  · role (project.role):
      font: var(--font-body), 0.9rem, weight 600
      color: var(--color-text-muted)
      margin-bottom: 12px

  · description (project.description):
      font: var(--font-body), 0.9rem, line-height 1.65
      color: var(--color-text-muted)
      margin-bottom: 16px

  · tags row (gap: 8px, flex-wrap: wrap):
      Each tag: project.tags array
      Style:
        border: 1px solid var(--border-tag-border-color)
        border-radius: var(--border-button-border-radius)
        padding: 4px 12px
        font: var(--font-body), 0.75rem
        color: var(--color-text-muted)
        background: transparent
      margin-bottom: 20px

  · links row (margin-top: auto, display: flex, justify-content: space-between):
      LEFT side — project.links (first 2):
        display: flex, gap: 16px
        Each link:
          color: var(--color-accent)
          font: var(--font-body), 0.85rem, font-weight 500
          text-decoration: none
          hover: text-decoration underline
      RIGHT side:
        "Public reference" text
        font: var(--font-body), 0.75rem
        color: var(--color-text-muted)
```

### 3b. Archive box rendering spec

```
CONTAINER:
  background: #0f1419 (slightly lighter than card bg — not a token)
  border: 1px solid var(--color-card-border)
  border-radius: var(--spacing-card-radius)
  padding: 32px 40px
  display: flex
  align-items: center
  justify-content: space-between

LEFT side:
  · "There's more behind the work."
      font: var(--font-heading), 1.3rem, weight 700
      color: var(--color-text-primary)
  · "Explore the wider project archive, including internal initiatives and client collaborations."
      font: var(--font-body), 0.9rem
      color: var(--color-text-muted)
      margin-top: 6px

RIGHT side:
  · Button: "View all projects" (collapsed) / "Hide full archive" (expanded)
      background: var(--color-accent)
      color: #000
      border-radius: var(--border-button-border-radius)
      padding: 12px 24px
      font-weight: 700, font-size: 0.9rem
      onClick: toggles PROJECT DIRECTORY section below

PROJECT DIRECTORY (collapsible, animated max-height):
  Renders below the archive box when expanded.
  See Section 3c below.
```

### 3c. Project directory rendering spec

```
CONTAINER:
  margin-top: 32px
  border: 1px solid var(--color-card-border)
  border-radius: var(--spacing-card-radius)
  overflow: hidden
  animation: slideDown 0.3s ease

DIRECTORY HEADER:
  padding: 32px 40px 24px
  · "PROJECT DIRECTORY" — section label style, lime
  · "All work & contributions" — font-heading, 2rem, white
  · row (space-between):
      left: "Browse by company, or search by project name, discipline, or technology."
            font-body, 0.9rem, muted
      right: "{total} entries" — 0.85rem, muted

FILTER + SEARCH ROW (padding: 0 40px 24px, gap: 12px):
  LEFT: filter pills (gap 8px)
    "All" | "DeduSoft" | "Liongate" | "NEXAH" | "The Net"
    Active pill: background var(--color-accent), color #000
    Inactive: border 1px solid var(--color-card-border), color muted
    onClick: filters the list below

  RIGHT: search input
    background: var(--color-card-bg)
    border: 1px solid var(--color-card-border)
    border-radius: 8px
    padding: 8px 16px
    color: var(--color-text-primary)
    placeholder: "Search projects, skills..."
    width: 240px
    onInput: filters the list in real time (client-side, no API call)

DIRECTORY LIST:
  Each row (padding: 24px 40px):
    display: grid
    grid-template-columns: 32px 1fr auto
    gap: 16px
    border-bottom: 1px solid var(--color-card-border)
    last row: no border-bottom

    LEFT: row number (01, 02 ...) — 0.85rem, muted, font-mono
    CENTER:
      · tag: "COMPANY · TYPE" — 0.7rem uppercase lime, margin-bottom 4px
      · title — 1.1rem white bold, margin-bottom 4px
      · description — 0.85rem muted
      · role — 0.8rem muted, margin-top 2px
    RIGHT (align-center, gap 12px):
      · "Case study" — lime text link, 0.85rem
      · "Quick details" — pill button, border var(--color-card-border),
          color white, padding 6px 16px, border-radius full, 0.8rem

MOBILE: search input becomes full width, filter pills wrap
```

---

## 4. Section Type: `how_i_work`

### Content schema

```typescript
export const HowIWorkContentSchema = z.object({
  section_label: z.string().max(60).default('03 / HOW I WORK'),
  headline:      z.string().max(60).default('Across the stack.'),
  headline_italic: z.string().max(60).default('Beyond the code.'),
  body:          z.string().max(300).default(
    'My strongest work happens where product thinking, quality engineering, and reliable delivery meet.'
  ),
  columns: z.array(z.object({
    heading: z.string().max(60),
    skills:  z.string().max(300),   // comma-separated string
  })).length(4).default([
    { heading: 'Product & frontend',   skills: 'React, TypeScript, Next.js, UI architecture, accessibility' },
    { heading: 'Backend & systems',    skills: 'PHP, Node.js, Python, REST APIs, SQL, microservices' },
    { heading: 'Cloud & delivery',     skills: 'Docker, CI/CD, Linux, infrastructure, deployment workflows' },
    { heading: 'Leadership & quality', skills: 'Solution design, code reviews, mentoring, Agile delivery' },
  ]),
})

export type HowIWorkContent = z.infer<typeof HowIWorkContentSchema>
```

### Rendering spec

```
BACKGROUND: var(--color-bg-green)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)

SECTION HEADER (two columns — left 45% / right 50%):
  LEFT:
    · section_label — label style
    · headline:
        font: var(--font-heading), clamp(2rem, 3.5vw, 3rem), weight 800
        color: var(--color-text-primary)
    · headline_italic:
        font: var(--font-heading), clamp(1.5rem, 2.5vw, 2rem), italic
        color: var(--color-text-muted)
        margin-top: 4px
  RIGHT:
    · body:
        font: var(--font-body), 1rem, line-height 1.75
        color: var(--color-text-muted)
        margin-top: 48px

DIVIDER (margin: 48px 0):
  border-top: 1px solid rgba(255,255,255,0.1)

SKILLS GRID:
  display: grid
  grid-template-columns: repeat(4, 1fr)
  Each column separated by: border-right: 1px solid rgba(255,255,255,0.08)
  Last column: no border-right
  padding: 0 32px for inner columns, 0 32px 0 0 for first, 0 0 0 32px for last

  Each column:
    · number label: "0{n} /" — 0.75rem, muted, uppercase, margin-bottom 16px
    · heading: 1.3rem, white, font-weight 600, margin-bottom 12px
    · skills: 0.85rem, muted, line-height 1.75
        rendered as plain text (comma-separated as-is)

MOBILE:
  Header: single column
  Skills grid: 2 columns (2×2), vertical dividers removed, horizontal divider between rows
```

---

## 5. Section Type: `experience`

### Content schema

```typescript
export const ExperienceContentSchema = z.object({
  section_label:  z.string().max(60).default('04 / EXPERIENCE'),
  headline_1:     z.string().max(60).default('The journey'),
  headline_2:     z.string().max(60).default('so far.'),
  body:           z.string().max(300).default(
    'From mentoring and hands-on development to architecture and engineering leadership.'
  ),
  education_label: z.string().max(60).default('CONTINUING TO LEARN'),
})

export type ExperienceContent = z.infer<typeof ExperienceContentSchema>
```

> Work and education entries come from the `experience` table, not from `content`.
> `content` controls only the section copy.

### Rendering spec

```
BACKGROUND: var(--color-bg)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)

SECTION HEADER (two columns — left 50% / right 45%):
  LEFT:
    · section_label — label style
    · headline_1:
        font: var(--font-heading), clamp(2.5rem, 4vw, 3.5rem), weight 800
        color: var(--color-text-primary)
    · headline_2:
        same font, same size
        last char (period): color var(--color-accent)
        (strip period from string, append <span> with lime period)
  RIGHT:
    · body — 1rem, muted, line-height 1.75, margin-top 48px

DIVIDER (margin: 48px 0):
  border-top: 1px solid var(--color-card-border)

WORK EXPERIENCE ROWS:
  Each row: display grid, grid-template-columns: 20% 35% 40%, gap 32px
  padding: 28px 0
  border-bottom: 1px solid var(--color-card-border)
  hover: background rgba(255,255,255,0.02), transition 0.2s

  LEFT (20%):
    · year range: "{year_start} to {year_end ?? 'present'}"
        font: var(--font-body), 0.9rem
        color: var(--color-accent)
        font-weight: 500

  CENTER (35%):
    · role title:
        font: var(--font-body), 1.2rem, weight 600
        color: var(--color-text-primary)
    · organization:
        font: var(--font-body), 0.75rem, uppercase
        letter-spacing: 0.1em
        color: var(--color-text-muted)
        margin-top: 4px

  RIGHT (40%):
    · description:
        font: var(--font-body), 0.9rem, line-height 1.7
        color: var(--color-text-muted)

EDUCATION BLOCK (margin-top: 64px):
  Two columns — left 20% / right 75%

  LEFT:
    · education_label — 0.75rem, uppercase, lime, letter-spacing 0.15em

  RIGHT:
    Each education entry:
      padding: 20px 0
      border-bottom: 1px solid var(--color-card-border)
      display: flex, justify-content: space-between, align-items: baseline

      LEFT side of row:
        · title — 1rem, white, font-weight 500
        · organization — 0.8rem, muted, margin-top 2px

      RIGHT side of row:
        · year_end ?? 'In progress (since {year_start})' — 0.85rem, muted

MOBILE:
  Work rows: stack to single column (year on top, then role, then description)
  Education: single column, year on separate line below org
```

---

## 6. Section Type: `contact`

### Content schema

```typescript
export const ContactContentSchema = z.object({
  section_label:    z.string().max(60).default('05 / LET\'S CONNECT'),
  headline_1:       z.string().max(60).default('Have something'),
  headline_2_italic: z.string().max(60).default('worth building?'),
  body:             z.string().max(300).default(
    "Whether you're exploring a product, looking for engineering leadership, or want to collaborate on something meaningful, let's talk."
  ),
  email_label:      z.string().max(80).default(''),  // falls back to portfolio.email
  extra_links: z.array(z.object({
    label: z.string().max(40),
    url:   z.string(),
  })).max(3).default([
    { label: 'GitHub',   url: '' },
    { label: 'LinkedIn', url: '' },
    { label: 'DeduSoft', url: '' },
  ]),
})

export type ContactContent = z.infer<typeof ContactContentSchema>
```

### Rendering spec

```
BACKGROUND: var(--color-bg-lime)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)

· section_label:
    font: var(--font-body), 0.75rem, uppercase, letter-spacing 0.15em
    color: rgba(26,26,26,0.5) — dark muted on lime bg
    margin-bottom: 32px

· headline_1:
    font: var(--font-heading), var(--font-size-contact-heading), weight 800
    color: var(--color-text-dark)
    line-height: 1

· headline_2_italic:
    font: var(--font-heading), var(--font-size-contact-heading), weight 800
    font-style: italic
    color: var(--color-text-dark)
    margin-bottom: 32px

· body:
    font: var(--font-body), 1.1rem, line-height 1.75
    color: rgba(26,26,26,0.7)
    max-width: 560px
    margin-bottom: 40px

· email link:
    display: block
    font: var(--font-heading), 1.8rem, weight 700
    color: var(--color-text-dark)
    text-decoration: underline
    text-underline-offset: 6px
    margin-bottom: 32px
    href: mailto:{portfolio.email}
    hover: opacity 0.75

    Text shown: content.email_label if set, else portfolio.email

· extra links row (gap: 32px):
    Each link:
      font: var(--font-body), 1rem, weight 500
      color: var(--color-text-dark)
      text-decoration: underline
      text-underline-offset: 4px
      hover: opacity 0.65

MOBILE: headline font-size clamp(2.5rem, 10vw, 4rem)
```

---

## 7. Section Type: `custom`

### Content schema

```typescript
export const CustomContentSchema = z.object({
  section_label: z.string().max(60).default(''),
  headline:      z.string().max(120).default(''),
  body:          z.string().max(2000).default(''),  // supports markdown
  layout:        z.enum(['text-only', 'two-column', 'centered']).default('text-only'),
})

export type CustomContent = z.infer<typeof CustomContentSchema>
```

### Rendering spec

```
BACKGROUND: var(--color-bg) (default — custom sections inherit dark bg)
PADDING: var(--spacing-section-padding-y) var(--spacing-section-padding-x)

layout = 'text-only':
  Max-width 720px, centered
  section_label → headline → body (markdown rendered)

layout = 'two-column':
  Left 45%: section_label + headline
  Right 50%: body (markdown rendered)

layout = 'centered':
  text-align center, max-width 800px, centered
  section_label → headline → body

Body markdown rendering:
  h2 → font-heading, 1.5rem, white
  p  → font-body, 1rem, muted, line-height 1.75
  a  → color accent, underline
  ul/ol → muted, 0.95rem, line-height 1.8
  code → monospace, bg #1a1f27, padding 2px 6px, border-radius 4px
```

---

## 8. Combined Zod Schema (for API validation)

```typescript
// lib/types/sections.ts — master union

export const SectionContentSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('hero'),        content: HeroContentSchema }),
  z.object({ type: z.literal('about'),       content: AboutContentSchema }),
  z.object({ type: z.literal('work'),        content: WorkContentSchema }),
  z.object({ type: z.literal('how_i_work'),  content: HowIWorkContentSchema }),
  z.object({ type: z.literal('experience'),  content: ExperienceContentSchema }),
  z.object({ type: z.literal('contact'),     content: ContactContentSchema }),
  z.object({ type: z.literal('custom'),      content: CustomContentSchema }),
])
```

---

## 9. Default Section Seed (Meli's portfolio — `0011_seed_meli.sql`)

```sql
-- Assumes portfolio_id and user_id are known from portfolio seed

INSERT INTO sections (portfolio_id, user_id, type, order_index, is_visible, content)
VALUES
  ('{portfolio_id}', '{user_id}', 'hero',       0, true, '{hero content json}'),
  ('{portfolio_id}', '{user_id}', 'about',      1, true, '{about content json}'),
  ('{portfolio_id}', '{user_id}', 'work',       2, true, '{work content json}'),
  ('{portfolio_id}', '{user_id}', 'how_i_work', 3, true, '{how_i_work content json}'),
  ('{portfolio_id}', '{user_id}', 'experience', 4, true, '{experience content json}'),
  ('{portfolio_id}', '{user_id}', 'contact',    5, true, '{contact content json}');
```

---

## 10. Section Component Checklist

Before marking a section component as complete, verify:

- [ ] Uses only CSS custom properties from design tokens — no hardcoded colors/fonts
- [ ] All text content comes from `content` prop or `portfolio` prop — no hardcoded strings
- [ ] Renders correctly when optional fields are empty (fallback to defaults from schema)
- [ ] Responsive: tested at 375px, 768px, 1200px, 1440px
- [ ] No data fetching inside the component — data comes from props only
- [ ] Hover/transition states match the rendering spec
- [ ] Fade-in on scroll: `opacity 0 → 1`, `translateY 20px → 0`, `0.6s ease`
      implemented via IntersectionObserver on the section's root element
