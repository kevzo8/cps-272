# CPS-272 — Self-Service User Registration with KYC Integration
## Technical Design Document (TDD) Index

**Ticket:** CPS-272 — Technical Design: Self-Service User Registration with KYC Integration
**Workspace analysed:** `svi-authentication-springboot-kyc` (auth-service SB), `generic-kyc-owa` (onboarding web app, Angular 16), `svi-kyc-api-springboot-kyc` (current KYC back office; legacy Jersey `kyc-api` retired), `svi-authenticationportal-react-kyc` (auth portal, React 19), `svi-authentication-java-kyc` (legacy)
**Design scope:** system architecture only — no implementation in this ticket.

### How to read this folder

**Start here:** `TDD-CPS-272-Self-Service-Registration-KYC.md` — the canonical Technical Design Document,
structured per the team TDD template (Executive Summary → Appendix). The files below are its annexes.

**Presenting to devs?** Use `../cps-272-presentation/index.html` — a single-file guided deck (high-level design,
diagrams, use cases, API, data, glossary, and all 21 tickets as collapsible items). See its `README.md`.

| File | Contents | Audience |
|------|----------|----------|
| `TDD-CPS-272-Self-Service-Registration-KYC.md` | **Canonical TDD (template-compliant):** summary, context, goals, solution, architecture + flows, front-end, back-end + decision tables, API, data model, rollout, observability, security/testing, decisions, appendix | Lead/TL, reviewers, all implementers |
| `HLR-Self-Service-Registration-KYC.md` | **Requirements source:** transcribed High-Level Requirements incl. FR-01…FR-33 — citation target for every `HLR §n` reference | All readers (read first alongside the TDD) |
| `01-TDD-main.md` | **Design rationale companion:** objectives, scope, FR traceability, current-state gap analysis with file-level evidence, target architecture, lifecycle + BPO note, dependencies/assumptions/risks, open questions | Lead/TL, reviewers, all implementers |
| `02-architecture-diagrams.md` | As-Is vs To-Be visual comparison + diff table, C4 context/container, self-registration sequence, login+KYC decision, KYC onboarding + PhilSys + biometrics, review/redo, tenant resolution — all in Mermaid | Architects, frontend + backend devs |
| `03-api-design.md` | New + changed REST interfaces for auth-service and the KYC back office, header/tenant contract, error model, OIDC claim changes | Backend devs, QA |
| `04-data-model.md` | Cassandra (`auth_system` + KYC `customer_kyc`) schema deltas, immutable lifecycle history, attempt versioning, indexes, retention | Backend devs, DBA |
| `05-user-stories-tickets.md` | Jira-ready epic/feature/story breakdown with priority, story points, components, requirements, acceptance criteria, FR mapping, sprint buckets | PM, TL, QA |
| `06-security-errorhandling.md` | Threat model, OTP/password/CAPTCHA/rate-limit design, anti-enumeration, PII/biometric handling, error catalogue, audit requirements | Security reviewer, backend + QA |
| `07-glossary.md` | Definitions: slug, realm, tenant, pending registration, OTP, attempt, review case, KYC status, lifecycle history, applicant IDs, callback, BPO | All readers, new joiners |
| `08-use-cases.md` | End-to-end scenarios UC-01…UC-10 (actors, flows, alternates) with ticket and FR coverage matrix | All readers, QA (UAT scripts) |
| `09-rollout-plan.md` | Phased rollout, open-question decision log with defaults, workstream order, pre-launch checklist, pilot gates, sign-offs | TL, PM, Release Manager |

### One-page summary

Today registration is **assisted-only**:

- `POST /user/register` requires `@Authorized + @Authenticated + @Permissions` — i.e. an already-logged-in admin/frontliner creates the user. There is **no public self-registration endpoint**.
- `generic-kyc-owa` boots with Keycloak `login-required` against a **single hard-coded realm** (`keycloakConfiguration.json`), has **no tenant / X-App-ID / X-Tenant-ID concept**, and is redeployed per tenant. Anonymous users cannot reach it.
- `svi-authenticationportal-react-kyc` has **no `/register` route** — only `/login/*` + `/main/*`. Tenant is resolved post-hoc via `POST /tenant {username}`.
- OTP (`totp` table + `GenerateOTPRequestDTO/VerifyOTPRequestDTO`) exists **only for forgot-password** (`PASSWORD_RESET` session), SMS is stubbed, and `users` has only `is_active/is_deleted/is_blocked` flags — **no KYC lifecycle state, no lifecycle history, no attempt versioning**.
- KYC back office (`svi-kyc-api-springboot-kyc`) has face verify/identify/enroll, PhilSys QR passthrough, and a person registry — but **no review-case resource, no approve/redo/reject, no attempt tracking, and no document-inspection/OCR provider**.

Target (this design):

```text
Verified Contact → Active Account (KYC_NOT_VERIFIED) → Login → KYC decision
  → Self-service OWA (same module, public mode) → Automated verification
  → Exception → non-real-time Review Queue → Approve / Redo / Reject
  → Redo = notify → login → same module → new attempt (old attempts immutable)
  → KYC_VERIFIED → Roles/Permissions + KYC claim → App access
```

Key architectural choices (details in `01-TDD-main.md`):

1. **New public Self-Service surface**, tenant-scoped (`/s/{tenantSlug}/register`), reusing auth-portal (Step 1) + OWA (Step 2) codebases — not a fourth app.
2. **Tenant resolution service** decoupled from username: slug → `tenant_id + realm_id + client_id(public only)` via new throttled public endpoint; secret never leaves server (preserves CPS-164 fix).
3. **Pending-registration + registration-OTP pattern** modelled on existing `totp` (HMAC-SHA256, single-use, TTL) — account row + Keycloak user created **only after OTP verify**.
4. **Immutable `user_lifecycle_history`** table as system-of-record; `users` keeps a materialised `kyc_status` for fast gating; KYC attempts versioned (`attempt_no`, `SUPERSEDED`, never overwritten).
5. **KYC status as authorization attribute/claim**, not as roles — enforced in `RoleService.getApps`, `UserService.getAccessRights`, `PermissionsFilter`, and OIDC `id_token` (`kyc_verified` claim). Prevents UI-bypass.
6. **Review queue is a thin orchestration over the existing applicant store** — new `review_case` resource + `approve/redo/reject` transitions that write both KYC attempt status and auth lifecycle history.

### Definition-of-Done traceability

| CPS-272 DoD | Covered in |
|-------------|------------|
| Design completed + reviewed by Lead/TL | This folder; open questions in `01 §12` |
| Addresses High-Level Requirements FR-01…FR-33 | `01 §2` traceability + `05` per-story FR mapping |
| KYC Back Office integration clearly defined | `01 §7`, `03 §5`, `04 §4` |
| Implementation tickets identified/created/linked | `05` — copy-paste Jira titles/descriptions/AC |
| Dependencies + risks documented | `01 §11–12` |

> All diagrams are Mermaid so they render in GitLab/GitHub/VS Code without image attachments.
