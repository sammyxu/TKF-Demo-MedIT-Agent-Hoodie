# TFOM Student Ambassador System

A planning + design workspace for the **TFOM Student Ambassador System** — a web app that replaces the manual Google Forms / Sheets workflow used by the University of Toronto **Temerty Faculty of Medicine** student volunteer program.

Students log activities, earn points (capped at 25/year), admins approve, and students redeem points for rewards.

> **No application code lives in this repo.** It contains specifications, static HTML mockups, and Claude Code skills that drive the eventual implementation. The Next.js app gets *built from* these inputs.

---

## What's in here

| Path | Purpose |
|---|---|
| [`docs/`](docs/) | Authoritative specs — PRD, plan, implementation guide, UI guide, POC + agent plan |
| [`html/`](html/) | Static HTML mockups — the **visual source of truth** for styling |
| [`.claude/skills/`](.claude/skills/) | Claude Code skills — `html-to-nextjs`, `uoft-web-styling` |
| [`prompt.txt`](prompt.txt) | Two ready-to-use build prompts (POC ~10–20 min, full ~60 min) |
| [`CLAUDE.md`](CLAUDE.md) | Repo-level guidance for Claude Code sessions |
| [`videos/`](videos/) | Final product demo recording |

---

## Documents

Read in order — see [`docs/index.md`](docs/index.md) for the full table of contents.

| Doc | Covers |
|---|---|
| [`docs/prd.md`](docs/prd.md) | Six functional requirements (FR-01..FR-06), business rules (25-pt cap, reward catalog, dedup), seven open questions |
| [`docs/plan.md`](docs/plan.md) | 4-week delivery plan, milestones, risks, definition of done |
| [`docs/code.md`](docs/code.md) | Implementation guide — Next.js + SQLite stack, layout, point engine, RBAC |
| [`docs/ui.md`](docs/ui.md) | U of T brand system — palette, typography, components, accessibility |
| [`docs/poc.md`](docs/poc.md) | POC scope — `/dashboard` + `/submit` CRUD on `submission`, server actions only |
| [`docs/agent.md`](docs/agent.md) | 4-agent parallel build plan (Phase 0 scaffold + contract → Phase 1 shell / dashboard / submit) |

### Document precedence

When sources disagree, follow this order:

1. **PRD** — functional + non-functional requirements, business logic
2. **POC spec** — for any work scoped to the POC, overrides `code.md`
3. **`code.md`** — full-build stack and structural decisions
4. **`ui.md`** + `html/` mockups — visual + styling
5. **Development Plan** — sequencing only, not requirements

Two specific overrides:

- `code.md` mentions Drizzle in places — **the `html-to-nextjs` skill's no-ORM rule wins.** Use raw `better-sqlite3`.
- `code.md` says "Next.js 14" — use whatever `create-next-app@latest` resolves to (currently Next 16 + React 19 + Tailwind v4).

---

## Hard constraints

- **25-point annual cap** per student per academic year — server-enforced in the full build (POC displays only)
- **SSO required** in the full build (provider gated by OQ-02; POC has no auth)
- **Mobile-first responsive** — sidebar collapses, table → card list, bottom tab bar
- **Preserve mockup styles verbatim** — copy `<style>` blocks from `html/` into `globals.css`; `class` → `className`, `for` → `htmlFor`, no Tailwind substitution on ported markup
- **No ORM** — raw `better-sqlite3` with typed query helpers in `src/lib/db.ts`
- **No `app/api/*` in the POC** — all writes go through server actions

---

## How to build the app

The repo ships **two prompts** in [`prompt.txt`](prompt.txt). Open Claude Code in this directory and paste one.

### POC build (~10–20 min)

`/dashboard` + `/submit`, single hardcoded student, no auth, full CRUD on `submission`.

```text
As a senior developer, and based on poc.md and the native claude code SKILLS
included in this project, build this into a NextJS app (proof-of-concept)
for the dashboard and student entry page. Use sub-agents (agent.md).
```

Drives the 4-agent parallel pattern from [`docs/agent.md`](docs/agent.md):

```
Phase 0 (solo)         Phase 1 (parallel)
┌────────────┐         ┌─────────┐
│  Agent A   │────┬───▶│ Agent B │  shell + globals.css
│ scaffold   │    │
│ + contract │    ├───▶│ Agent C │  /dashboard
└────────────┘    │
                  └───▶│ Agent D │  /submit
```

### Full build (~60 min)

All six FRs — SSO, admin queue, rewards catalog, point ledger, RBAC. Prompt variant included in [`prompt.txt`](prompt.txt).

---

## Skills

Read these before scaffolding — they're authoritative.

- **[`html-to-nextjs`](.claude/skills/html-to-nextjs/SKILL.md)** — converts static HTML into a Next.js 15 + TypeScript app backed by SQLite (no ORM, raw `better-sqlite3`); enforces verbatim style preservation.
- **[`uoft-web-styling`](.claude/skills/uoft-web-styling/SKILL.md)** — encodes the U of T brand: palette, typography, buttons, cards, motion, the caret element, accessibility checklist.

---

## After scaffolding

Once a session scaffolds the app (typically into `poc/`), the standard scripts apply:

```bash
cd poc
npm run dev          # start dev server
npm run dev:clean    # rm -rf .next && npm run dev (Turbopack stale-cache fix)
npm run build        # type-check + production build
npm run lint
```

If `/dashboard` or `/submit` renders all-black on a dark-mode system, that's the documented Turbopack-stale-cache 404 — see [`docs/poc.md`](docs/poc.md#known-dev-pitfalls).

---

## Status

Planning workspace, not a deployable app. The repo regenerates `poc/` from scratch each build session — trust [`docs/poc.md`](docs/poc.md) as the spec to rebuild from rather than reconstructing prior output.
