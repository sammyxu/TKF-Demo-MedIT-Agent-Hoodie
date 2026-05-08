# Agent Plan — Parallel POC Build (HTML → Next.js)

Goal: ship the [poc.md](../poc.md) POC fast — convert [../html/dashboard.html](../html/dashboard.html) and [../html/submit.html](../html/submit.html) into a runnable Next.js app backed by SQLite via server actions. Single hardcoded student. Full CRUD on `submission`. **No SSO, no admin, no rewards, no API routes.**

## How many agents

**Four.** The work has one short setup bottleneck and three independent ports that all sit on top of it. Splitting any further would create artificial coordination cost; collapsing further (the previous 3-agent plan bundled scaffold + data contract + shell + styles into one agent) made Phase 0 the critical path for everything else.

| Phase 0 (solo, short) | Phase 1 (3-way parallel) |
|---|---|
| **A** — scaffold + data contract | **B** — shell + styles (`layout.tsx`, `globals.css`) |
|                                  | **C** — `/dashboard` (read + delete + edit-link) |
|                                  | **D** — `/submit` (create + edit form) |

```
Phase 0 (solo)         Phase 1 (parallel)
┌────────────┐         ┌─────────┐
│  Agent A   │────┬───▶│ Agent B │  shell + globals.css (byte-for-byte port)
│ scaffold   │    │
│ + contract │    ├───▶│ Agent C │  /dashboard (queries.list + DeleteButton)
└────────────┘    │
                  └───▶│ Agent D │  /submit (create + edit)
```

Why not 3 (collapse B into A): porting the navbar/sidebar/styles byte-for-byte is the largest mechanical job in the POC. Putting it on the critical path delays C and D for no reason — they can write their page JSX against class names from the mockup the moment the scaffold exists.

Why not 5+ (split the dashboard page from its delete island, etc.): the islands are 20-line client components that ship with the page that consumes them. Cutting them out creates merge-order pain without saving time.

## Sequencing

- **Phase 0** — Agent A alone, until scaffold + data contract are in place. One short pass.
- **Phase 1** — B, C, D in parallel against A's contract (`db.ts`, `points.ts`, `actions.ts`, plus the still-empty `layout.tsx`/`globals.css` that B is filling). C and D can write their page bodies before B finishes the shell; the visual smoke pass happens after all three are done.
- **Integration** — single smoke pass against the [poc.md](../poc.md) verification checklist.

## Shared contract (owned by Agent A)

| Artefact | File | Consumers |
|---|---|---|
| Project scaffold (`create-next-app`) | `poc/` subdirectory | B, C, D |
| DB singleton, schema, types, queries | `poc/src/lib/db.ts` | C (reads), D (reads via `getSubmission`), A in actions |
| Points helper (`balance`, `ANNUAL_CAP`, `POINTS_PER_HOUR`) | `poc/src/lib/points.ts` | C |
| Server actions (`createSubmission`, `updateSubmission`, `deleteSubmission`, `getSubmission`) | `poc/src/app/actions.ts` | C (delete), D (create/update/get) |
| Root redirect → `/dashboard` | `poc/src/app/page.tsx` | — |

Once these land, B/C/D can develop their files without touching each other's or A's contract files.

---

## Agent A — Scaffold & Data Contract

**Mission:** unblock B, C, D. Stand up the Next.js project and the data layer + server actions. **Do not write the shell, the styles, or any page bodies.**

**Owns**
- `poc/package.json`, Next.js + Tailwind + ESLint config (from `create-next-app`)
- `poc/data/` directory (the `.db` file is created at first import)
- `poc/src/lib/db.ts` — raw `better-sqlite3` singleton, schema bootstrap, `Submission` type, `STUDENT_ID = 1`, `queries` prepared statements (per [poc.md §DB singleton](../poc.md))
- `poc/src/lib/points.ts` — `ANNUAL_CAP = 25`, `POINTS_PER_HOUR = 1`, `balance(rows)` (per [poc.md §Points helper](../poc.md))
- `poc/src/app/actions.ts` — `'use server'` with `createSubmission` / `updateSubmission` / `deleteSubmission` / `getSubmission` (per [poc.md §Server actions](../poc.md))
- `poc/src/app/page.tsx` — server component that `redirect('/dashboard')`
- **Stub files only** for B's territory: create empty `poc/src/app/layout.tsx` (minimal `<html><body>{children}</body></html>` so Next.js boots) and empty `poc/src/app/globals.css`. B replaces both.

**Done when**
- `npm run dev` from `poc/` boots without errors. `/dashboard` and `/submit` 404 (their files don't exist yet — that's C and D).
- The DB file is created at `poc/data/poc.db` on first import.
- B, C, D can `import { queries, STUDENT_ID, type Submission } from '@/lib/db'`, `import { balance } from '@/lib/points'`, and `import { createSubmission, updateSubmission, deleteSubmission, getSubmission } from '@/app/actions'` with no missing references.

**Out of scope:** shell, styles, page bodies, any business rule beyond the four-line `validate()` in `actions.ts`, auth, admin, rewards.

---

## Agent B — Shell & Styles

**Mission:** port the navbar + sidebar + global styles from the mockups into the Next.js shell, byte-for-byte.

**Owns**
- `poc/src/app/layout.tsx` — navbar + sidebar shell from the mockups, byte-for-byte. Strip only the **Sign out** button, the **Rewards** sidebar/tabbar entry, and the avatar's hardcoded initials (leave the avatar circle as static decoration). Convert sidebar/navbar `<a>` to `next/link`. Active-item highlight via `usePathname()` (small client component). Load **Barlow** via `next/font/google` at the same weights the mockup uses.
- `poc/src/app/globals.css` — concatenated `<style>` blocks from [../html/dashboard.html](../html/dashboard.html) and [../html/submit.html](../html/submit.html), exact duplicates removed by hand. **No refactor, no rename, no Tailwind substitution.**
- `poc/src/components/Nav.tsx` (or similar small client component) for the active-link highlight, if needed.

**Reads from A:** nothing — pure UI shell. May read the contract types if it wants to render anything dynamic in the navbar (none required by spec).

**Spec**
- Both `<style>` blocks copy across **verbatim**. Same CSS variables (`--uoft-blue`, etc.), same hex values, same selectors. JSX rename only: `class` → `className`, `for` → `htmlFor`.
- Sidebar entries link to `/dashboard` and `/submit` only. Mark active via `usePathname()`.
- Mobile bottom tabbar mirrors the sidebar entries (per the mockup's existing breakpoints).
- Do not add Tailwind classes to the ported markup. Tailwind exists for net-new components only; even those should reuse the mockup's CSS variables.

**Done when** opening either `/dashboard` or `/submit` (once C and D land them) renders inside the correct shell at desktop *and* mobile breakpoints, indistinguishable from the static mockups around the page-body region.

---

## Agent C — `/dashboard` page + Delete island

**Mission:** port the dashboard mockup into a server component, wire reads to `queries.list`, render the points widget from `balance()`, and add the **Delete** client island + **Edit** link.

**Owns**
- `poc/src/app/dashboard/page.tsx` — server component
- `poc/src/components/DeleteSubmissionButton.tsx` — tiny client island that calls `deleteSubmission(id)` then `router.refresh()`
- (optional) `poc/src/components/SubmittedBanner.tsx` — reads `searchParams.submitted`

**Reads from A:** `queries.list`, `STUDENT_ID`, `Submission` type, `balance`, `deleteSubmission` server action.

**Spec**
- Call `queries.list.all(STUDENT_ID)` directly (no fetch, no API).
- Compute `balance(rows)` and bind to the points widget (number, remaining, progress bar).
- Map rows to the existing table + `mobile-list` markup from [../html/dashboard.html:580-684](../html/dashboard.html#L580-L684) **without altering classes, structure, or inline styling tokens**. JSX rename only: `class` → `className`, `for` → `htmlFor`.
- **Edit** = `next/link` to `/submit?id=<id>` on each row.
- **Delete** = render `<DeleteSubmissionButton id={row.id} />` on each row.
- Render the success banner when `searchParams.submitted === '1'`.
- Empty state: if `rows.length === 0`, render an on-brand empty state (use the `uoft-web-styling` skill — bundled at [.claude/skills/uoft-web-styling.skill](.claude/skills/uoft-web-styling.skill)).

**Done when** balance, table, mobile-list, edit-link, and delete all work end-to-end; the page visually matches [../html/dashboard.html](../html/dashboard.html) at desktop and mobile breakpoints (once B's shell is in).

---

## Agent D — `/submit` page (create + edit form)

**Mission:** port the submit mockup into a client component that handles both create (no `?id`) and edit (with `?id=<n>`).

**Owns**
- `poc/src/app/submit/page.tsx` — client component (`'use client'`), dual-purpose create/edit form

**Reads from A:** `createSubmission`, `updateSubmission`, `getSubmission` server actions.

**Spec**
- Port the form markup from [../html/submit.html:538-661](../html/submit.html#L538-L661) into JSX. **Keep every class name, attribute, and inline style.** JSX rename only: `class` → `className`, `for` → `htmlFor`.
- On mount, if `?id=<n>` is present, call `getSubmission(Number(id))` and prefill all fields. Switch the page heading and submit-button label to indicate edit mode.
- On submit:
  - create mode → `await createSubmission(input)` (the action redirects)
  - edit mode → `await updateSubmission(id, input)` (the action redirects)
- Keep the existing client validation: name min 2 chars, date ≤ today, hours 0.5–24, char counter on description.
- Drop the mocked `await new Promise(r => setTimeout(r, 1400))` delay.
- Drop the duplicate-detection demo block (script line ~819) — out of scope.

**Done when** create writes a row + redirects to `/dashboard?submitted=1`; navigating from `/dashboard` Edit prefills the form; saving updates the row and redirects; client validation matches the mockup.

---

## Coordination

- **One agent per file.** Ownership tables above are exhaustive. If you think you need to touch another agent's file, stop and surface it.
- **Communicate via the contract, not the implementation.** B, C, D import from `@/lib/db`, `@/lib/points`, `@/app/actions`. They never read each other's source.
- **B vs C/D ordering.** C and D can write their JSX before B finishes the shell — class names live in the mockup, not in B's output. Final visual verification waits on B.
- **No API routes.** All writes are server actions. If you find yourself reaching for `app/api/*`, you are off-spec — see [poc.md §Stack](../poc.md).
- **No ORM.** Raw `better-sqlite3` only ([../CLAUDE.md](../CLAUDE.md) hard constraint, reinforced by the `html-to-nextjs` skill).
- **Preserve mockup styles verbatim.** Non-negotiable ([../CLAUDE.md](../CLAUDE.md), [poc.md §Styles](../poc.md)). For anything the mockups don't cover (empty state, edit affordance, error toast), invoke the `uoft-web-styling` skill.
- **Single integration pass at the end.** Walk the [poc.md §Verification checklist](../poc.md) once B, C, D all report done.

## What is explicitly deferred (do not build)

Per [poc.md §Scope](../poc.md): SSO / NextAuth, admin approval queue, rewards catalog + redemption, multi-student handling, academic-year boundaries, server-side annual-cap enforcement, deduplication, file uploads, email, audit log, and **any HTTP API layer**.
