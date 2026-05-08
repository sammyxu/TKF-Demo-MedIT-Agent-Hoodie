# Implementation Guide — Next.js + SQLite

## Stack
| Concern | Choice |
|---|---|
| Framework | Next.js 14 (App Router, TypeScript) |
| Database | SQLite via `better-sqlite3` (single `.db` file, synchronous) |
| Auth | NextAuth.js v5 (SSO: Microsoft Entra / Google / SAML) |
| Styling | Tailwind CSS (mobile-first) |
| Validation | Zod (shared between API + forms) |
| Testing | Vitest + React Testing Library + Playwright |

> Note: `code.md` mentions Drizzle, but the `html-to-nextjs` skill mandates **raw `better-sqlite3` with no ORM**. When running the skill, the no-ORM constraint wins (per repo CLAUDE.md).

## Project layout
```
tfom-ambassador/
├── app/
│   ├── (auth)/login/             # SSO login
│   ├── (student)/                # Protected student routes
│   │   ├── dashboard/  submit/  rewards/
│   ├── (admin)/                  # Protected admin routes
│   │   ├── dashboard/  submissions/  catalog/
│   └── api/                      # Route handlers
│       ├── auth/[...nextauth]/   # NextAuth
│       ├── submissions/  points/  catalog/  redeem/
├── lib/
│   ├── db/         # connection singleton + schema
│   ├── auth.ts     # NextAuth config
│   ├── points.ts   # point engine
│   ├── validations.ts  # Zod schemas
│   └── middleware.ts   # RBAC helpers
├── components/  ui/  forms/  tables/  layout/
├── middleware.ts   # route protection
└── data/tfom.db    # SQLite (gitignored)
```

## Core entities
`Student`, `Admin`, `Activity`, `Submission`, `PointLedger`, `Reward`, `Redemption`.

## API surface
| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/submissions` | Create submission |
| PATCH | `/api/submissions/:id` | Approve/reject (admin) |
| GET | `/api/points` | Student balance + remaining allowance |
| GET | `/api/catalog` | List rewards with stock |
| POST | `/api/redeem` | Redeem points for reward |

## Point engine (FR-03) rules
- Convert hours → points using confirmed rate (OQ-01)
- Enforce **25-pt annual cap** server-side before approval finalizes
- Write to `PointLedger` atomically on approval
- Reject (or warn) if approval would breach the cap

## RBAC
Middleware-level route protection: `(student)` and `(admin)` route groups gated by NextAuth session role claim.

## Testing strategy
- Vitest: unit tests on point engine edge cases (exact cap, over-cap, zero hours, fractional hours)
- RTL: component-level tests for forms and tables
- Playwright: E2E flows (Student: login → submit → track → redeem; Admin: login → review → approve)

## CI/CD
Automated build + deploy to staging on merge; production promotion is manual.
