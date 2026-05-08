# POC — Dashboard + Submit (CRUD only)

A minimal proof-of-concept that converts [html/dashboard.html](html/dashboard.html) and [html/submit.html](html/submit.html) into a working Next.js app backed by SQLite. **No SSO, no login, no admin, no rewards, no API routes.** A single hardcoded student. Goal: prove the data path end-to-end (create → list → edit → delete) using server components + server actions only.

## Scope

**In:**
- `/dashboard` — reads submissions, shows points balance + 25-pt cap progress, recent submissions table/cards.
- `/submit` — creates a submission, redirects to `/dashboard?submitted=1`.
- Inline **Edit** + **Delete** controls on each dashboard row (so all four CRUD operations are exercised).
- One entity: `submission`.

**Out (defer to full build):**
- SSO / NextAuth / login routes
- **Any HTTP API layer** — no `app/api/*` route handlers. CRUD goes through server actions invoked directly from components.
- Admin approval queue (submissions are auto-approved on create — see [Simplifications](#simplifications))
- Rewards catalog + redemption
- Multi-student / academic-year boundaries (OQ-04)
- Deduplication policy (OQ-03) — keep the form's client-side hint but do not enforce server-side
- File uploads, email, audit log

## Stack

Per [docs/code.md](docs/code.md) and the `html-to-nextjs` skill — **no ORM**, **no API routes**.

| Layer | Choice |
|---|---|
| Framework | Next.js 15 (App Router) + TypeScript |
| Styling | Inline `<style>` blocks from the mockups, copied **verbatim** into `app/globals.css` (see [Styles](#styles)) |
| DB | SQLite via `better-sqlite3` (single `data/poc.db` file, synchronous) |
| Mutations | Server Actions (`'use server'`) — called directly from forms / client islands |
| Reads | Server components query the DB directly via the `queries` helper |
| State | None — `revalidatePath('/dashboard')` after each mutation |

Scaffold into a subdirectory so the repo root stays clean:

```bash
npx create-next-app@latest poc --typescript --tailwind --eslint --app --src-dir --no-turbopack
cd poc
npm i better-sqlite3
npm i -D @types/better-sqlite3
mkdir -p data
```

## Simplifications for POC

These are deliberate shortcuts — flag them in the eventual real build.

1. **Hardcoded student.** A constant `STUDENT_ID = 1`, `STUDENT_NAME = "Jane Doe"`. No `users` table needed; the `submissions` table just has a nullable `student_id` for forward compatibility but every row is `1`.
2. **Auto-approve.** Since there's no admin to approve, the create action writes `status = 'approved'` and `points_awarded = hours_worked` (1 pt/hr — placeholder rate, OQ-01 in PRD). The "pending" badge stays in the UI as a possible future state but the POC never produces it.
3. **Annual cap not enforced.** Display only. Real build must enforce server-side per [CLAUDE.md](CLAUDE.md) hard constraints.
4. **No auth.** Server actions trust the hardcoded `STUDENT_ID`. Acceptable for POC; do **not** deploy.
5. **Strip the navbar's "Sign out" button** and the avatar's hardcoded initials — leave the wordmark and the avatar as static decoration.

## Data model

One table. Drop the `data/poc.db` file to reset.

```sql
CREATE TABLE IF NOT EXISTS submissions (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  student_id      INTEGER NOT NULL DEFAULT 1,
  activity_name   TEXT    NOT NULL,
  activity_date   TEXT    NOT NULL,                 -- ISO YYYY-MM-DD
  hours_worked    REAL    NOT NULL,
  description     TEXT,
  status          TEXT    NOT NULL DEFAULT 'approved'
                          CHECK (status IN ('pending','approved','rejected')),
  points_awarded  REAL    NOT NULL DEFAULT 0,
  created_at      TEXT    NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_submissions_student_date
  ON submissions(student_id, activity_date DESC);
```

## Project layout

No `api/` directory — all writes go through `actions.ts`.

```
poc/
├── data/poc.db
├── src/
│   ├── app/
│   │   ├── globals.css                  # mockup styles, verbatim
│   │   ├── layout.tsx                   # navbar + sidebar shell (shared)
│   │   ├── page.tsx                     # → redirect to /dashboard
│   │   ├── actions.ts                   # 'use server' — createSubmission / updateSubmission / deleteSubmission
│   │   ├── dashboard/page.tsx           # server component, reads DB
│   │   └── submit/page.tsx              # client form, calls server actions
│   └── lib/
│       ├── db.ts                        # singleton + typed helpers
│       └── points.ts                    # balance/remaining calc
└── package.json
```

## DB singleton — `src/lib/db.ts`

Singleton pattern survives Next.js dev hot-reload. Schema runs on first import.

```ts
import Database from 'better-sqlite3';
import path from 'node:path';

const globalForDb = globalThis as unknown as { db?: Database.Database };

export const db = globalForDb.db ?? new Database(path.join(process.cwd(), 'data', 'poc.db'));
db.pragma('journal_mode = WAL');
db.pragma('foreign_keys = ON');

db.exec(`
  CREATE TABLE IF NOT EXISTS submissions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    student_id INTEGER NOT NULL DEFAULT 1,
    activity_name TEXT NOT NULL,
    activity_date TEXT NOT NULL,
    hours_worked REAL NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'approved'
      CHECK (status IN ('pending','approved','rejected')),
    points_awarded REAL NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
  );
  CREATE INDEX IF NOT EXISTS idx_submissions_student_date
    ON submissions(student_id, activity_date DESC);
`);

if (process.env.NODE_ENV !== 'production') globalForDb.db = db;

export type Submission = {
  id: number;
  student_id: number;
  activity_name: string;
  activity_date: string;
  hours_worked: number;
  description: string | null;
  status: 'pending' | 'approved' | 'rejected';
  points_awarded: number;
  created_at: string;
};

export const STUDENT_ID = 1;

export const queries = {
  list: db.prepare<[number], Submission>(
    `SELECT * FROM submissions WHERE student_id = ? ORDER BY activity_date DESC, id DESC`
  ),
  get: db.prepare<[number, number], Submission>(
    `SELECT * FROM submissions WHERE id = ? AND student_id = ?`
  ),
  insert: db.prepare(
    `INSERT INTO submissions (student_id, activity_name, activity_date, hours_worked, description, status, points_awarded)
     VALUES (?, ?, ?, ?, ?, 'approved', ?)`
  ),
  update: db.prepare(
    `UPDATE submissions SET activity_name = ?, activity_date = ?, hours_worked = ?, description = ?, points_awarded = ?
     WHERE id = ? AND student_id = ?`
  ),
  remove: db.prepare(`DELETE FROM submissions WHERE id = ? AND student_id = ?`),
};
```

## Points helper — `src/lib/points.ts`

```ts
import { Submission } from './db';

export const ANNUAL_CAP = 25;
export const POINTS_PER_HOUR = 1; // POC placeholder — real value gated by OQ-01

export function balance(rows: Submission[]) {
  const earned = rows
    .filter(r => r.status === 'approved')
    .reduce((sum, r) => sum + r.points_awarded, 0);
  return {
    earned,
    cap: ANNUAL_CAP,
    remaining: Math.max(0, ANNUAL_CAP - earned),
    pct: Math.min(100, (earned / ANNUAL_CAP) * 100),
  };
}
```

## Server actions — `src/app/actions.ts`

The full CRUD surface. Components import these directly; no `fetch`, no JSON, no route handlers.

```ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { queries, STUDENT_ID } from '@/lib/db';
import { POINTS_PER_HOUR } from '@/lib/points';

type SubmissionInput = {
  activity_name: string;
  activity_date: string;
  hours_worked: number;
  description?: string | null;
};

function validate(input: SubmissionInput) {
  if (!input.activity_name || input.activity_name.trim().length < 2) throw new Error('Invalid name');
  if (!input.activity_date) throw new Error('Invalid date');
  const hours = Number(input.hours_worked);
  if (!hours || hours < 0.5 || hours > 24) throw new Error('Invalid hours');
  return hours;
}

export async function createSubmission(input: SubmissionInput) {
  const hours = validate(input);
  const points = hours * POINTS_PER_HOUR;
  queries.insert.run(
    STUDENT_ID,
    input.activity_name.trim(),
    input.activity_date,
    hours,
    input.description?.trim() || null,
    points,
  );
  revalidatePath('/dashboard');
  redirect('/dashboard?submitted=1');
}

export async function updateSubmission(id: number, input: SubmissionInput) {
  const hours = validate(input);
  const points = hours * POINTS_PER_HOUR;
  const result = queries.update.run(
    input.activity_name.trim(),
    input.activity_date,
    hours,
    input.description?.trim() || null,
    points,
    id,
    STUDENT_ID,
  );
  if (!result.changes) throw new Error('Not found');
  revalidatePath('/dashboard');
  redirect('/dashboard?submitted=1');
}

export async function deleteSubmission(id: number) {
  queries.remove.run(id, STUDENT_ID);
  revalidatePath('/dashboard');
}

export async function getSubmission(id: number) {
  return queries.get.get(id, STUDENT_ID) ?? null;
}
```

## Pages

### `src/app/dashboard/page.tsx` (server component)

- Calls `queries.list.all(STUDENT_ID)` directly — same process, no network hop.
- Computes `balance(rows)` and renders the points widget.
- Maps rows to the existing table/`mobile-list` markup from [html/dashboard.html:580-684](html/dashboard.html#L580-L684) **without altering classes, structure, or inline styling tokens**.
- Reads `searchParams.submitted` to render the success banner.
- **Delete** is a tiny client island that calls `deleteSubmission(id)` (the server action) and then `router.refresh()` — no fetch.
- **Edit** = a `next/link` to `/submit?id=<id>`; the submit page becomes a dual-purpose create/edit form.

### `src/app/submit/page.tsx` (client component)

- Port the form from [html/submit.html:538-661](html/submit.html#L538-L661) into JSX. **Keep every class name, attribute, and inline style from the mockup** — only the `<form>` action wiring changes.
- On mount, if `?id=<n>` is present, call `getSubmission(Number(id))` (server action) and prefill.
- On submit: call `createSubmission(input)` or `updateSubmission(id, input)` directly. Both server actions perform the redirect — the client just `await`s.
- Keep the existing client validation (min 2 chars, date ≤ today, hours 0.5–24, char counter). Drop the `await new Promise(r => setTimeout(r, 1400))` mock delay.
- Drop the duplicate-detection demo block (script line ~819) — out of scope.

### `src/app/layout.tsx`

- Wrap children in the navbar + sidebar from the mockups, **byte-for-byte**.
- Replace `<a href="/dashboard">` / `<a href="/submit">` with `next/link` (href values unchanged).
- Mark the active item via `usePathname()` (small client component for the nav).
- Strip only the **Sign out** button and the **Rewards** sidebar/tabbar entry — leave everything else (wordmark, avatar, spacing, colours) untouched.

## Styles

**Maintain the HTML mockups' styling exactly.** This is the [CLAUDE.md](CLAUDE.md) "Preserve mockup styles verbatim" hard constraint and is non-negotiable for the POC.

Concrete rules:

1. Copy the entire `<style>...</style>` blocks from [html/dashboard.html](html/dashboard.html) and [html/submit.html](html/submit.html) into `src/app/globals.css` **verbatim**. Both files share most tokens; concatenate them and remove only exact duplicates by hand. Do not "clean up", refactor into Tailwind utilities, rename CSS variables, or substitute hex values.
2. Preserve every `class`, `id`, and inline `style` attribute on ported markup. JSX rename only: `class` → `className`, `for` → `htmlFor`. Nothing else.
3. Load the **Barlow** font in `layout.tsx` via `next/font/google` instead of the `<link>` tag, but keep the same weights the mockup uses.
4. Do not add Tailwind classes to ported elements. The Tailwind setup from `create-next-app` exists for any **new** components only (e.g., the Delete client island), and even those should reuse the mockup's CSS variables (`--uoft-blue`, etc.) instead of inventing colours.
5. SVGs, icons, and decorative elements (caret, gradients) copy across unchanged.

For anything the mockups don't cover — empty states, edit-mode affordances on the dashboard row, error toasts — invoke the **`uoft-web-styling`** skill (bundled at [docs/.claude/skills/uoft-web-styling.skill](docs/.claude/skills/uoft-web-styling.skill)). The skill encodes the U of T palette, typography, button styles, and caret design element so additions stay on-brand and consistent with the existing mockup CSS.

## Verification checklist

Run `npm run dev` and walk through:

- [ ] `/` redirects to `/dashboard`; the page loads with an empty table + "0" balance.
- [ ] Visual diff against [html/dashboard.html](html/dashboard.html) and [html/submit.html](html/submit.html) — colours, spacing, typography, sidebar, navbar, cards, and form fields render **identically** to the static mockup.
- [ ] Click **Submit Activity** → fill form → submit → redirected to `/dashboard?submitted=1` with success banner; new row appears at top.
- [ ] Balance number, remaining, and progress bar update without a hard reload (server component re-renders on navigation).
- [ ] Click **Edit** on a row → form prefills → change hours → save → row updates, balance recalculates.
- [ ] Click **Delete** on a row → row disappears, balance recalculates.
- [ ] Rotate to mobile width: sidebar hides, bottom tabbar shows, mobile card list replaces the table — same breakpoints as the mockup.
- [ ] No requests to `/api/*` in the Network tab — only RSC payloads and server-action POSTs to the page URL.

## Known dev pitfalls

### "All black, no content" on `/dashboard` or `/submit` (Turbopack stale cache)

**Symptom.** You open `http://localhost:3000/dashboard` (or `/submit`) and the entire page is solid black with nothing visible. Looks like the app is broken.

**What's actually happening.** Both pages are returning `404`, and Next.js's built-in not-found page contains:

```css
@media (prefers-color-scheme: dark) { body { background: #000; color: #fff; } }
```

On a dark-mode system the 404 renders as black-on-black — the "404 / This page could not be found." text is technically there but visually invisible. Network tab will show `404` for both routes.

**Root cause.** Turbopack's `.next/` cache gets out of sync — typically after a `next build` ran while a `next dev` server was already running, or when multiple lockfiles confuse Turbopack about the workspace root (Agent A flagged the `/Users/tampak2/package-lock.json` vs `poc/package-lock.json` ambiguity). The dev server then claims the App Router routes don't exist even though the files are on disk.

**Fix.**

```bash
# stop the dev server, then:
rm -rf .next && npm run dev
```

Or use the script we ship:

```bash
npm run dev:clean
```

**Prevention** (already applied in this POC):

1. `next.config.ts` pins `turbopack.root` to the `poc/` directory so the multi-lockfile warning goes away and Turbopack always picks the right root.
2. `src/app/not-found.tsx` overrides Next's default 404 with an on-brand page that uses our explicit colour tokens (no `prefers-color-scheme` flip). If a route ever 404s again, it will be obviously a 404 — not a black void.
3. `package.json` exposes `npm run dev:clean` for the rare case the cache still gets wedged.

If you see the symptom anyway: hard-refresh the browser (`Cmd+Shift+R`) to bust the cached 404 HTML, then `npm run dev:clean`.

## Known POC gaps (read before extending)

- Annual cap is **displayed but not enforced**. Real build: reject `createSubmission` if `balance.earned + new_points > 25`.
- Auto-approval bypasses the entire admin workflow (FR-04, FR-05). Real build: status defaults to `'pending'`, admin route flips it.
- No CSRF protection beyond Next.js's built-in server-action origin check, no rate limiting, no auth. Do not deploy.
- Points-per-hour is hardcoded at 1.0 — real value is gated by **OQ-01** in the [PRD](docs/prd.md).
- Academic-year filtering is missing — `balance()` sums all rows ever. Gated by **OQ-04**.
- No HTTP API surface exists. If/when external clients (mobile, integrations) are needed, add `app/api/*` route handlers that wrap the same `queries` helpers — server actions stay for the web UI.
