---
name: html-to-nextjs
description: Convert one or more static HTML files into a fully functional Next.js 15 (App Router) + TypeScript application backed by SQLite via better-sqlite3, with complete CRUD (Create, Read, Update, Delete) for every data entity present in the HTML. Use this skill whenever the user asks to convert, port, scaffold, wire up, or "turn into a real app" any HTML mockup — particularly for the TFOM Student Ambassador System dashboards and forms. Trigger on requests like "convert this HTML to Next.js", "scaffold the app from the mockup", "build the backend for these pages", or "wire up CRUD for the dashboard". This skill covers entity extraction, project scaffolding, the SQLite singleton pattern with typed query helpers, App Router API routes, React component conversion (preserving original styles verbatim), optional seeding, and verification. Enforces no ORM — raw better-sqlite3 only — and never redesigns the UI.
---

# HTML → Next.js + SQLite CRUD Skill

## Purpose
Convert one or more static HTML files into a fully functional Next.js 15 (App Router) application backed by a SQLite database, implementing complete CRUD (Create, Read, Update, Delete) for every data entity present in the HTML.

---

## Invocation
The user will point you at one or more `.html` files (paths or content). Execute the process below in order. Do not skip steps.

---

## Process

### Step 1 — Analyse the HTML

Read every target HTML file and extract:

- **Entities**: each distinct type of data (a table of "submissions", a list of "rewards", etc.). Each becomes a DB table.
- **Fields**: for each entity, list field name, inferred SQLite type (`TEXT`, `INTEGER`, `REAL`, `NUMERIC`), and constraints (NOT NULL, UNIQUE, DEFAULT, CHECK).
- **Relationships**: foreign keys implied by the UI (e.g. activity belongs_to student → `student_id INTEGER REFERENCES students(id)`).
- **Existing styles**: all `<style>` blocks and class names — copy them exactly; do not redesign.
- **Page states**: every state shown in the HTML (default, loading, empty, error, confirmation, cap-reached, etc.) must be implemented.

Present the entity model to the user as a markdown table. Wait for confirmation before writing any code.

---

### Step 2 — Scaffold the Project

```bash
npx create-next-app@latest <app-name> \
  --typescript --tailwind --eslint \
  --app --src-dir --no-import-alias
cd <app-name>
npm install better-sqlite3
npm install --save-dev @types/better-sqlite3
```

Create `data/` at the project root and add it to `.gitignore` — the SQLite file lives there:

```
# .gitignore additions
data/*.db
```

---

### Step 3 — Database Layer

Create `src/lib/db.ts` using the singleton pattern from `references/sqlite-patterns.md`:

1. Module-level singleton with `global.__db` for dev hot-reload safety.
2. `pragma journal_mode = WAL` and `pragma foreign_keys = ON` always set.
3. One `initDb()` function that creates all tables with `CREATE TABLE IF NOT EXISTS` and returns the `Database` instance.
4. Typed query helpers (one set per entity): `getAll`, `getById`, `create`, `update`, `remove`.
5. Export a single `TypedDb` type that bundles the `Database` instance — import only this in routes.

Use raw `better-sqlite3` queries. No ORM, no query builder, no Prisma.

---

### Step 4 — API Routes

For **each entity** create two route files following the templates in `references/api-patterns.md`:

| File | Methods |
|---|---|
| `src/app/api/<entity>/route.ts` | `GET` (list) · `POST` (create) |
| `src/app/api/<entity>/[id]/route.ts` | `GET` (one) · `PUT` (full update) · `PATCH` (partial update) · `DELETE` |

Rules for every route:
- Call `initDb()` at the top of each handler — never import a pre-connected db instance across modules.
- Validate required fields; return `400` with `{ error: "..." }` on bad input.
- Return `{ data: ... }` on success with correct status: `200` (GET/PUT/PATCH), `201` (POST), `204` (DELETE, no body).
- Return `404` when an id is not found.
- Wrap in try/catch; return `500` on unexpected errors.

---

### Step 5 — React Components

Convert each HTML section into a component in `src/components/`:

- Copy all `<style>` blocks verbatim into `src/app/globals.css` (append, don't overwrite Tailwind directives).
- Add `"use client"` to any component that owns form state, event handlers, or calls `fetch`.
- Use the `useCrud` hook pattern from `references/api-patterns.md` to wire list/create/update/delete to the API.
- Implement every page state identified in Step 1.
- Form submissions must show an inline loading indicator and clear the form on success.

Page entry points go in `src/app/` following App Router file conventions (`page.tsx`, `layout.tsx`).

---

### Step 6 — Seed Data (optional)

If the HTML contains example rows (tables with data, populated lists), create `src/lib/seed.ts` that inserts them with `INSERT OR IGNORE` so re-running is safe. Call it from a `npm run seed` script in `package.json`.

---

### Step 7 — Verify

```bash
npm run dev
```

Open the app in the browser. Exercise every CRUD operation for every entity. Confirm all page states render correctly. Fix any TypeScript errors (`npm run build` must pass with zero errors).

---

## Hard Constraints

- **No ORM.** Raw `better-sqlite3` only.
- **Preserve original styles.** Never redesign the UI — convert structure and wire up data.
- **TypeScript strict mode.** No `any`. All query helpers return typed interfaces.
- **Keep route files thin.** Business logic and query helpers live in `src/lib/db.ts`, not in routes.
- **No environment variables for the DB path.** Use `path.join(process.cwd(), 'data', 'app.db')`.
