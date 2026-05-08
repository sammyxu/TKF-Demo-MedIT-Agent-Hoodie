# PRD — TFOM Student Ambassador System PoC

## What it is
A web-based platform replacing the manual Google Forms + Sheets workflow for student volunteer activity submissions, point tracking, and reward redemption at U of T's Temerty Faculty of Medicine.

## Why it exists
The current process is manual, error-prone, opaque to students, and non-scalable. The PoC must deliver automation, transparency, and reliability for both students and admins.

## User roles
- **Student** — submit activities, track points, redeem rewards
- **Admin** — review/approve submissions, manage catalog

## Functional requirements (FR-01 … FR-06)
| ID | Requirement |
|---|---|
| FR-01 | Activity submission with validation |
| FR-02 | Admin approval workflow |
| FR-03 | Automatic point calculation (hours → points) |
| FR-04 | Real-time point tracking for students |
| FR-05 | Reward catalog + redemption with stock check |
| FR-06 | SSO authentication + role-based access control |

## Non-functional requirements
- SSO required (NFR-01)
- Mobile-friendly responsive UI (NFR-02)
- Scalable architecture (NFR-03)
- Maintainable by a small team (NFR-04)

## Hard business rules
- **25-point annual cap** per student per academic year (server-enforced)
- Hours → points conversion (rate TBD — see OQ-01)
- Deduplication required before approval enters the queue
- Reward catalog: Pen (2), Umbrella (10), Water Bottle (20), Hoodie (25)

## Open questions blocking development
| # | Question | Blocks |
|---|---|---|
| OQ-01 | Hours → points conversion rate | FR-03 |
| OQ-02 | SSO provider | FR-06 |
| OQ-03 | What counts as a duplicate | Dedup logic |
| OQ-04 | Academic year boundaries | Cap enforcement |
| OQ-05 | Reward stock manual or auto-decrement | FR-05 |
| OQ-06 | Admin approval SLA | UX/notifications |
| OQ-07 | Single vs. multi-tier admin roles | RBAC design |

## Success criteria (SC-01 … SC-05)
Reduce manual work · Prevent errors/duplicates · Student transparency · Scalable & maintainable · Realistic to implement.

## Out of scope for PoC
Wireframes, full API docs, notifications, gamification, reporting, bulk admin tools, CSV export.
