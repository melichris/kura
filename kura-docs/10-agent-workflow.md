# Kura — AI Agent Workflow & Senior Dev Proposal

> This document defines exactly how Kura is built using AI agents.
> Every session, every agent, every ticket follows this workflow.
> Read this before opening Claude Code. Read this before writing a ticket.
> This is the operating contract between you (Meli) and every AI agent.

---

## 0. The Core Problem This Document Solves

AI agents have one critical failure mode:

> **They start coding before they understand the system.**

They hallucinate file paths, invent their own patterns, ignore conventions,
and produce code that works in isolation but breaks the architecture.

The solution is not better prompting on the fly.
The solution is a documentation-first system where the agent has no room
to guess — because everything is already specified.

That is what the Kura docs folder is. Every agent reads it before building anything.

---

## 1. Tooling Stack

| Role | Tool | Why |
|---|---|---|
| Primary build agent | Claude Code | Reads docs, works in repo, runs commands, edits files. Best for sustained multi-file work |
| Ticket writing + architecture review | Claude.ai (this Project) | Senior dev brain — tickets, reviews, decisions, context |
| Version control | GitHub | PR-based flow, branch protection on main |
| Project tracking | GitHub Issues | Free, lives next to the code, no extra tool needed in v1 |
| Environment secrets | Vercel dashboard + `.env.local` | Never committed to repo |
| DB migrations | Supabase CLI | `supabase db push` — versioned, reviewable, reversible on dev |

> When collaborators join (Phase 4+), migrate tracking to Linear.
> Until then, GitHub Issues is sufficient.

---

## 2. Role Separation

Two tools, two clearly separated roles. Never mix them.

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│   CLAUDE.AI (THIS PROJECT)      │    │   CLAUDE CODE (IN THE REPO)     │
│   Senior Dev Brain              │    │   Hands in the Codebase         │
│                                 │    │                                 │
│ · Write tickets                 │    │ · Read docs + ticket            │
│ · Define acceptance criteria    │    │ · WHY → WHAT → HOW → BUILD      │
│ · Architecture decisions        │    │ · Create/edit files             │
│ · Review PR diffs               │    │ · Run commands (tsc, lint)      │
│ · Catch edge cases              │    │ · Open PR                       │
│ · Update build log              │    │ · Never scope-creep             │
│ · Phase gate decisions          │    │ · Never guess architecture      │
└─────────────────────────────────┘    └─────────────────────────────────┘
         ▲                                          │
         │         You (Meli) bridges both          │
         └──────────────────────────────────────────┘
```

**You are the bridge.** You bring the ticket from here into Claude Code.
You bring the PR diff back here for review. You merge. You sign off.

---

## 3. The Exact Session Workflow

Every working session follows this shape, in this order. No shortcuts.

```
STEP 1 — Read the build log (2 min)
  Open 07-build-log.md
  Know: current phase, last ticket completed, next ticket, any blockers
  If blockers exist → resolve them here before touching Claude Code

STEP 2 — Write the ticket here (10–15 min)
  Say: "Let's work on KURA-XXX"
  I write the full ticket using the template from 09-project-management.md
  We review scope together — nothing ambiguous before it goes to the agent
  Ticket is confirmed → move to Step 3

STEP 3 — Prime Claude Code (5 min)
  Open Claude Code in the kura repo root
  Paste the Agent Context Prompt (Section 6 of this document)
  Paste the ticket
  Wait for the agent to confirm it has read the relevant docs
  Do not proceed if the agent skips the doc-reading step

STEP 4 — Agent builds (variable — 20 min to 2h)
  Agent follows: WHY → WHAT → HOW → BUILD → VERIFY
  You observe — do not interrupt mid-flow unless:
    · Agent is going out of scope
    · Agent is modifying files outside the ticket
    · Agent asks a question it shouldn't need to ask (means a doc is missing)
  If the agent asks what it should already know from the docs → stop,
  answer the question here, then update the relevant doc before continuing

STEP 5 — Review here (10–15 min)
  Come back to this chat
  Paste the PR diff, or describe what was built
  I review as a senior dev:
    · Does it match the acceptance criteria?
    · Does it follow the architecture from 01-architecture.md?
    · Are edge cases handled?
    · Any TypeScript `any` types or console.logs?
    · Are design tokens used correctly (no hardcoded values)?
  I give you a clear verdict: APPROVE / REQUEST CHANGES + what to fix

STEP 6 — Fix and merge (variable)
  If changes requested → back to Claude Code with the review notes
  If approved → merge PR into develop
  develop → main only on phase gate

STEP 7 — Update build log (5 min)
  Come back here
  I update 07-build-log.md:
    · Ticket marked complete
    · Session entry appended
    · Next ticket confirmed
  Session done.
```

---

## 4. The Agent Context Prompt

**Copy this exactly into every new Claude Code session.**
Do not paraphrase. Do not shorten. The agent needs every word.

```
You are building Kura — a design-token-driven portfolio CMS and SaaS platform.

BEFORE YOU DO ANYTHING ELSE, read these files in this exact order:
1. kura-docs/00-overview.md        — product vision, phases, decisions
2. kura-docs/01-architecture.md    — stack, folder structure, data flows
3. kura-docs/09-project-management.md — workflow, conventions, definition of done
4. kura-docs/07-build-log.md       — current state, last completed ticket, next up
5. kura-docs/[RELEVANT DOC]        — the doc specific to today's ticket

After reading, confirm:
- What phase we are in
- What the last completed ticket was
- What today's ticket is
- Which files will be created or modified

Then read the ticket below and follow the analysis protocol:
WHY → state why this ticket exists and what it unblocks
WHAT → list every file that will be created or modified
HOW → full step-by-step implementation plan BEFORE writing any code
BUILD → implement following the patterns in the docs exactly
VERIFY → check every acceptance criterion explicitly
LOG → note what was done (I will update the build log)

HARD RULES — never violate these:
- No hardcoded colors, fonts, or sizes. All values via CSS custom properties.
- No `any` TypeScript types.
- No `console.log` in production code.
- No data fetching inside components. Data comes from props only.
- No files modified outside the ticket scope.
- No new dependencies without explicit approval.
- All DB access via lib/supabase/server.ts (server) or lib/supabase/client.ts (client).
- All API input validated with Zod before any DB call.
- Minimal diffs — only change what the ticket requires.
- If you are unsure about an architectural decision, stop and ask.
  Do not guess and do not proceed.

Stack: Next.js 15 App Router · TypeScript strict · Tailwind CSS v4 ·
Supabase (Auth + Postgres + Storage) · Zod · Vercel

---

[PASTE TICKET HERE]
```

---

## 5. What You (Meli) Personally Own

These decisions and actions cannot be delegated to any AI agent.
You are accountable for them.

| Action | Why you own it |
|---|---|
| Reviewing and approving every PR before merge | You are accountable for what ships to production |
| Reviewing DB migration files before `supabase db push` | Migrations on prod are irreversible |
| Setting and rotating all environment variables | Security boundary — no agent sees production secrets |
| Activating design tokens on the live portfolio | Your aesthetic, your brand, your call |
| Signing off on each phase gate (DoD checklist) | You decide when a phase is complete |
| Deciding when to scope-change a ticket | Only you can expand or contract scope |
| Approving any new npm dependency | Every dependency is a maintenance and security decision |

---

## 6. What the Agent Owns

Everything not in the list above.

| Action | Agent handles it |
|---|---|
| Writing all production code | ✅ |
| Creating and editing files | ✅ |
| Running TypeScript, ESLint, build checks | ✅ |
| Opening PRs with the standard template | ✅ |
| Updating documentation when code changes it | ✅ |
| Writing tests | ✅ |
| Following the section rendering spec exactly | ✅ |

---

## 7. Risk Register & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Agent ignores docs and invents patterns | High without the prompt | High | Always start with the Agent Context Prompt. If agent skips doc-reading, restart the session. |
| Agent expands ticket scope | Medium | Medium | Tickets are scoped small deliberately. If agent modifies files outside the ticket, stop and roll back those changes. |
| DB migration errors in production | Low with discipline | High | Always test migrations on `kura-dev` Supabase project. Never push directly to `kura-prod`. |
| Design tokens breaking the layout | Low | Medium | Token preview iframe (KURA-030) catches this before activation. |
| Phase creep — Phase 2 work during Phase 1 | Medium | Medium | Build log gates this. If a ticket isn't in the current phase queue, it doesn't get built. |
| Lost context between sessions | Low | High | Build log + this Project memory = zero context loss between sessions. |
| Agent introduces `any` types or hardcoded values | Medium | Medium | CI typecheck catches `any`. PR review here catches hardcoded values. |
| Dependency added without approval | Low | Medium | Hard rule in Agent Context Prompt. CI build diff shows `package.json` changes. |

---

## 8. If the Agent Gets Stuck or Confused

Signs the agent is confused:
- Asks a question that is answered in the docs
- Proposes a file structure different from `01-architecture.md`
- Starts writing code before stating WHY/WHAT/HOW
- References a library not in the stack
- Creates files in wrong directories

What to do:
1. Stop the agent immediately
2. Come back here
3. Paste the agent's question or proposed approach
4. I will either:
   - Point to the doc that answers it (agent context gap)
   - Update a doc that was missing the answer (doc gap)
   - Make an architectural decision and document it (new decision)
5. Resume the session with the updated context

**Never let the agent proceed on a guess.**
Every guess compounds into technical debt.

---

## 9. GitHub Issues Setup

Create these labels in the GitHub repo before KURA-001:

| Label | Color | Usage |
|---|---|---|
| `phase:0` | #6b7280 | Foundation tickets |
| `phase:1` | #3b82f6 | Public portfolio tickets |
| `phase:2` | #8b5cf6 | Admin CMS tickets |
| `phase:3` | #f59e0b | Design token engine tickets |
| `phase:4` | #ec4899 | SaaS layer tickets |
| `type:feat` | #22c55e | New feature |
| `type:fix` | #ef4444 | Bug fix |
| `type:chore` | #6b7280 | Config, deps, tooling |
| `type:docs` | #0ea5e9 | Documentation only |
| `status:queued` | #d1d5db | Not started |
| `status:in-progress` | #fbbf24 | Currently being built |
| `status:review` | #a78bfa | PR open, awaiting review |
| `status:done` | #22c55e | Merged and complete |
| `blocked` | #ef4444 | Cannot proceed — dependency missing |

Each GitHub Issue maps 1:1 to a Kura ticket (KURA-001 = Issue #1).
Issue title format: `KURA-XXX: [Short title]`

---

## 10. Branch Protection Rules

Set these in GitHub → Settings → Branches before writing a single line of code.

### `main` branch

```
✅ Require a pull request before merging
✅ Require approvals: 1 (you review your own PRs from the agent)
✅ Require status checks to pass:
     · typecheck
     · lint
     · build
✅ Require branches to be up to date before merging
✅ Do not allow bypassing the above settings
❌ Allow force pushes — OFF
❌ Allow deletions — OFF
```

### `develop` branch

```
✅ Require status checks to pass:
     · typecheck
     · lint
✅ Allow direct push for chore/docs commits
❌ Allow force pushes — OFF
```

---

## 11. Realistic Timeline

Working at 3–4 sessions per week, each session 2–3 hours:

| Phase | Tickets | Estimated duration |
|---|---|---|
| Phase 0 — Foundation | 3 | 1 week |
| Phase 1 — Public portfolio | 11 | 2–3 weeks |
| Phase 2 — Admin CMS | 12 | 3–4 weeks |
| Phase 3 — Design token engine | 7 | 1–2 weeks |
| **Total to working product** | **33** | **~8–10 weeks** |

Phase 4 (SaaS) is a separate product sprint.
Scope and ticket it after Phase 3 ships and is stable.

---

## 12. The Single Most Important Rule

> **Never let an agent write code for a ticket that does not have a written spec.**

Every time you skip the ticket and say "just build this", you get code
that works today and blocks you in three weeks.

The documentation we wrote is what makes the agent reliable.
The workflow above is what keeps the documentation current.
Use both. Every session. No exceptions.

---

## 13. Quick Reference Card

Print this or keep it visible during every session.

```
SESSION START
─────────────
1. Read 07-build-log.md
2. Write ticket here (Claude.ai)
3. Paste Agent Context Prompt into Claude Code
4. Paste ticket
5. Confirm agent read the docs before it codes

DURING BUILD
─────────────
· Watch for scope creep
· If agent guesses → stop → answer here → update doc → resume

SESSION END
─────────────
1. Review PR diff here
2. Approve or request changes
3. Merge to develop
4. Update build log
5. Confirm next ticket

PHASE GATE
─────────────
· All DoD checkboxes checked?
· Deployed and smoke tested?
· Lighthouse audit passed? (Phase 1)
· Only then: develop → main
```
