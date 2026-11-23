# KURA-001: Scaffold Next.js monorepo + tooling

**Phase:** 0 — Foundation
**Epic:** Foundation
**Estimate:** 2h 30m (2h base + 30 min buffer)
**Status:** Queued

---

### Summary

Scaffold the Kura Next.js monorepo with the exact folder structure from
`01-architecture.md`, configured with TypeScript strict, Tailwind CSS v4,
ESLint, Supabase client setup, and the CI workflow file.

---

### User Story

As a developer working on Kura,
I want a fully configured repo scaffold,
so that every subsequent ticket builds on a consistent, working foundation.

---

### Context / Background

This is the first ticket. Nothing exists in the repo yet except the docs folder.
The scaffold must match `01-architecture.md` exactly — no improvisation.
The CI workflow must exist after this ticket so Stage 2 branch protection
can be enabled immediately after merge.

---

### Detailed Description

Create the following from scratch:

1. Initialize Next.js 15 app with App Router and TypeScript strict
2. Install and configure Tailwind CSS v4
3. Install and configure ESLint (Next.js ruleset)
4. Create the full folder structure from `01-architecture.md`
5. Set up Supabase client files (`lib/supabase/client.ts`, `server.ts`, `middleware.ts`)
6. Create `.env.example` with all variables from `08-deployment.md`
7. Create `next.config.ts` with standalone output and security headers
8. Create `.github/workflows/ci.yml` with typecheck, lint, build jobs
9. Create root `middleware.ts` (auth pass-through stub — expanded in KURA-016)
10. Add the `kura-docs/` folder to the repo

**Out of scope:**

- No UI components
- No DB migrations
- No actual Supabase project connection (env vars are stubs)
- No pages or layouts

---

### Acceptance Criteria

- [ ] `npm run dev` starts without errors
- [ ] `npx tsc --noEmit` passes with zero errors
- [ ] `npm run lint` passes with zero errors
- [ ] `npm run build` completes successfully
- [ ] Folder structure matches `01-architecture.md` exactly
- [ ] `lib/supabase/client.ts`, `server.ts`, `middleware.ts` exist and are typed
- [ ] `.env.example` contains all required variables from `08-deployment.md`
- [ ] `next.config.ts` has standalone output + security headers
- [ ] `.github/workflows/ci.yml` exists with 3 jobs: typecheck, lint, build
- [ ] `kura-docs/` folder committed to repo root with all 11 docs
- [ ] No `any` TypeScript types
- [ ] No hardcoded values

---

### Technical Notes

**Exact folder structure to create:**

```
kura/
├── app/
│   ├── (portfolio)/
│   │   └── .gitkeep
│   ├── admin/
│   │   └── .gitkeep
│   └── api/
│       └── health/
│           └── route.ts
├── components/
│   ├── portfolio/
│   │   └── .gitkeep
│   ├── admin/
│   │   └── .gitkeep
│   └── ui/
│       └── .gitkeep
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── middleware.ts
│   └── types/
│       ├── portfolio.ts
│       ├── tokens.ts
│       ├── sections.ts
│       └── user.ts
├── supabase/
│   └── migrations/
│       └── .gitkeep
├── public/
├── kura-docs/
├── middleware.ts
├── next.config.ts
├── tailwind.config.ts
├── .env.example
└── .github/
    └── workflows/
        └── ci.yml
```

**Key implementation details:**

`lib/supabase/client.ts`:

```typescript
import { createBrowserClient } from "@supabase/ssr";
export const createClient = () =>
  createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
```

`lib/supabase/server.ts`:

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
export const createClient = async () => {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { getAll: () => cookieStore.getAll() } },
  );
};
```

`app/api/health/route.ts`:

```typescript
export function GET() {
  return Response.json({ status: "ok", ts: Date.now() });
}
```

`middleware.ts` (root — stub):

```typescript
import { type NextRequest, NextResponse } from "next/server";
export async function middleware(_req: NextRequest) {
  return NextResponse.next();
}
export const config = {
  matcher: ["/admin/:path*"],
};
```

`lib/types/` — all 4 files start as empty exports:

```typescript
export {};
```

**Packages to install:**

```
@supabase/ssr          ← use this, NOT @supabase/auth-helpers-nextjs (deprecated)
@supabase/supabase-js
```

**Tailwind:** Use v4 config syntax — not v3. They differ significantly.

---

### Test Plan

| Scenario     | Steps                | Expected result                      |
| ------------ | -------------------- | ------------------------------------ |
| Dev server   | `npm run dev`        | Starts on localhost:3000, no errors  |
| Health check | `GET /api/health`    | `{ "status": "ok", "ts": 000 }`      |
| TypeScript   | `npx tsc --noEmit`   | Zero errors                          |
| Lint         | `npm run lint`       | Zero errors                          |
| Build        | `npm run build`      | Completes, standalone output present |
| CI           | Push branch, open PR | All 3 GitHub Actions jobs pass       |

---

### Dependencies

- Blocked by: Nothing
- Blocks: KURA-002, KURA-003, all Phase 1 tickets
- Related to: `kura-docs/01-architecture.md`, `kura-docs/08-deployment.md`

---

### Risks & Open Points

- Tailwind v4 config syntax differs from v3 — agent must not use v3 syntax
- Use `@supabase/ssr` package only — `@supabase/auth-helpers-nextjs` is deprecated

---

### Definition of Done

- [ ] All acceptance criteria checked
- [ ] `npm run dev`, `tsc`, `lint`, `build` all pass locally
- [ ] PR opened against `develop` using standard PR template from `09-project-management.md`
- [ ] CI passes on the PR
- [ ] After merge: Stage 2 branch protection enabled on `main` and `develop`
- [ ] `07-build-log.md` updated

---

### Commit Message

```
chore(scaffold): initialize Next.js 15 monorepo with TypeScript, Tailwind, Supabase clients, and CI

Refs: #kura-001
```

### PR Title

```
chore(scaffold): initialize monorepo — KURA-001
```
