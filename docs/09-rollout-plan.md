# CPS-272 — Rollout Plan: Phases, Decisions, Checklists

How this actually ships. Phases map to the sprint buckets in `05-user-stories-tickets.md`;
open questions map to `01 §12`; risks map to `01 §11` and TDD §14.

## 1. Phase plan

| Phase | Goal | Tickets | Exit criteria (gate) |
|---|---|---|---|
| **0 — Decide + spike** (sprint 1, parallel track) | Answer blocking questions; prove risky integrations | Decisions Q1–Q3; spikes: Keycloak Admin throughput, email burst test | Decisions recorded; vendors shortlisted; spikes demoed |
| **1 — Foundations** (sprints 1–3) | Public registration + login work end to end | A-01…A-04, B-01, B-02, D-06 alongside, D-08 flags | Registration→login E2E on staging; security review of public trio passed; no PII in logs verified |
| **2 — Verification pipeline** (sprints 4–6) | Self onboarding approves clean cases | C-01, C-03, C-02a/b/c, B-03 | Approve + review-branch E2E; assisted regression green; PhilSys sandbox certified |
| **3 — Human review ops** (sprints 6–8) | Reviewers clear every exception path | D-01, D-02, D-02b, D-04, D-03, D-05, D-07 | Redo E2E; adjudication E2E with engine call verified; reviewers trained; SLA dashboard live |
| **4 — Pilot + expansion** | One tenant live, then one at a time | D-08 | §5 targets met; go/no-go per tenant |

## 2. Open questions decision log

"Proposed default" lets work start; owners confirm or override by the decide-by date.

| ID | Question (`01 §12`) | Proposed default (ship unblocked) | Owner | Blocks | Decide by |
|---|---|---|---|---|---|
| Q1 | Password hashing + legacy migration | Argon2id for new accounts (bcrypt acceptable); legacy passwords migrate on next successful login | Security | A-03 | Phase 0 |
| Q2 | SMS vendor, sender IDs, template ownership | Contract a vendor with alphanumeric sender IDs; if late, ship email-only pilot behind the channel flag | Product | A-02, D-05 | Phase 0 |
| Q3 | Slug minting and rename policy | Team-minted lowercase slugs, immutable; rename = new slug + redirect row | RBAC admin | A-01 | Phase 0 |
| Q4 | Retry limits before forced review | 3 attempts, then forced review case | Product | C-02c | Phase 2 |
| Q5 | Raw evidence retention vs metadata | Metadata + hashes retained indefinitely; raw blobs under legal hold (purge default to confirm with Legal) | Legal / Product | D-07 purge jobs (non-MVP) | Phase 3 |
| Q6 | OWA build strategy | Single runtime-config build, tenant from URL slug | Frontend leads | C-01 | Phase 2 |
| Q7 | BPO mirror depth on day one | Mirror review-case events only, never per-registration activity | Product / BPO | D-01 (non-blocking) | Phase 3 |
| Q8 | PhilSys error-code taxonomy | Every mismatch routes to a review case; no automatic decisions on external data | KYC owner | C-02b | Phase 2 |

## 3. How we implement (workstream order)

1. **Contracts first.** No story starts without its `03` API section and `04` schema reviewed (Definition of Ready in `05`).
2. **Backend builds behind flags; frontend builds against contract mocks in parallel.** Integration happens per feature, not in a big bang.
3. **Security alongside, not after.** Rate limits, CAPTCHA, and uniform replies land in the same stories as the endpoints they protect (D-06 with A-02/A-03).
4. **Data migration in four moves:** additive DDL → batched backfill of `users.kyc_status` (verify null-count zero) → seed slugs, settings, default roles → dual-read period with fallback-rate alert → flag on per tenant.
5. **Environments graduate:** dev (mock SMS/PhilSys) → staging (sandbox vendors, volume seed) → prod pilot.
6. **Rehearse failure:** rollback drill, callback-backlog drain, review-SLA breach drill — all before pilot.

## 4. Pre-launch checklist

| Area | Item | Owner | Status |
|---|---|---|---|
| Secrets | Key rotation procedure documented; Keycloak service accounts per realm; SMS keys in vault, never in bundles | Platform / Security | ☐ |
| Identity | Keycloak Admin rate limits confirmed; orphan reconciliation job scheduled | Platform | ☐ |
| Vendors | SMS contract + sender IDs live; email burst test passed | Product | ☐ |
| Edge | WAF rules for `/public/*`; CORS allowlist per slug; TLS verified | Platform | ☐ |
| Data | Backfill null-count zero; pilot slug + settings + default role seeded; reviewer/adjudicator accounts provisioned | Backend / RBAC admin | ☐ |
| People | Reviewers trained on queue + adjudication; support macros and runbooks published; pilot users informed | Ops / Product | ☐ |
| Observability | Dashboards (OTP delivery, callback lag, queue depth/SLA, abuse blocks) + paging alerts wired | Backend | ☐ |
| Rollback | Flag-off drill passed and recorded | Release Manager | ☐ |

## 5. Pilot entry, exit, expansion, rollback

- **Entry:** all phase exits green + checklist above complete.
- **Exit targets (30 days, tune per tenant):** registration completion rate at or above tenant baseline; OTP delivery p95 under 30 s; status-callback lag p95 under 60 s; review SLA met on 95%+ of cases; zero enumeration incidents; assisted channel volume unaffected.
- **Expansion:** one tenant at a time, each clearing the same entry bar.
- **Rollback trigger:** any SLO breach unrecoverable within 24 hours → tenant flag off (no database rollback needed).

## 6. Top risks (condensed)

| Risk | Mitigation | Owner |
|---|---|---|
| Account enumeration via public endpoints | Uniform replies + timing + CAPTCHA + rate limits (§`06 §3–4`, D-06) | Security |
| OTP interception / SIM-swap | Short TTL, single-use, no OTP in logs; device binding reserved Phase 2 | Security |
| Duplicate-identity fraud | 1:N hits always become adjudicated cases; fraud tables retained | Product / Security |
| Keycloak/Cassandra partial creates | Ordered create + compensating disable + reconciliation job | Backend |
| KYC back-office outage coupling | Materialised status; fail-closed only where verification is required | Backend |
| Review queue overload | Priority + SLA + assignment + escalation + redo analytics | Ops |

## 7. Sign-offs

| Gate | Approver |
|---|---|
| Design approved | Lead Developer / TL |
| Public-surface security review | Security Team |
| Pilot go | Product + Lead |
| Per-tenant expansion | Tenant sponsor |
