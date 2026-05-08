# Development Plan — 4-Week PoC Delivery

## Team
3 developers (Backend, Frontend, Fullstack) + 1 QA + 1 Business Analyst.

## Guiding principles
- **Resolve all 7 open questions in Week 1** before feature work begins
- Backend-first on the point engine and approval flow (critical path)
- Vertical slices, not horizontal layers
- QA runs in parallel from Week 1
- Scope frozen after Week 1

## Timeline at a glance
| Week | Focus | Key deliverable |
|---|---|---|
| 1 | Kickoff · Discovery · Foundations | OQs resolved · auth working · CI/CD live · API contract finalized |
| 2 | Core feature dev (parallel BE/FE) | FR-01 to FR-04 on staging |
| 3 | Feature completion · integration · QA execution | FR-05 + full integration · QA defect report |
| 4 | Stabilization · UAT · launch | Production deploy · stakeholder sign-off |

## Critical-path milestones
- **Day 1:** All 7 open questions answered (BA)
- **End of Week 1:** SSO + RBAC working, CI/CD live (Dev 1, Dev 3)
- **End of Week 2:** FR-01–FR-04 on staging
- **End of Week 3:** All 6 FRs integrated, QA execution complete
- **Day 25:** UAT sign-off
- **Day 28:** Production launch

## Top risks
| Risk | Mitigation |
|---|---|
| OQs unresolved by Day 2 | BA escalates to leadership; team works on unblocked items |
| SSO integration delays | Username/password fallback for PoC |
| Scope creep after Week 1 | Hard scope freeze |
| Point calc logic bugs | Unit-test every edge case before frontend integration |
| UAT reveals UX gaps | Informal stakeholder preview in Week 3 |

## Definition of done
**Feature:** merged via PR · unit tests pass · on staging · QA signed off · no Critical/High defects.
**PoC:** all 6 FRs delivered · all 5 constraints satisfied · all 5 success criteria met · UAT sign-off obtained.

## Key assumptions
- OQ-01 and OQ-02 closed by Day 1
- Standard 5-day weeks, no planned absences
- Stakeholders available for UAT in Week 4
- No data migration from Google Sheets needed for PoC
