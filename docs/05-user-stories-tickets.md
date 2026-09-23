# CPS-272 — Features, User Stories & Jira-Ready Tickets

## How to use this file in Jira

**Epic:** CPS-272 Self-Service Registration with KYC (this TDD ticket). Structure below: 4 features with 21 stories and tasks.

**Field mapping (copy-paste per ticket):**

| This file | Jira field | Notes |
|---|---|---|
| Title | Summary | Keep the ID prefix (`A-01 …`) so IDs survive import |
| Type | Issue Type | `Story`, or `Task` where marked (ops/docs, no end-user value) |
| Priority | Priority | Scale below; project uses Blocker/Critical/Major/Minor/Trivial |
| Story Points | Story Points | Fibonacci; 8-pointers must be broken into subtasks at sprint planning |
| Component(s) | Component/s | Create if missing: `auth-service`, `kyc-api`, `auth-portal`, `owa`, `reviewer-ui`, `platform` |
| Labels | Labels | Always `cps-272` plus the feature label (`self-registration`, `kyc-decision`, `kyc-onboarding`, `review-ops`) |
| Epic | Epic Link | CPS-272 |
| Dependencies | Linked Issues (using the `is blocked by` link type) | Ticket IDs below; `01 DEP-xx` = external dependency in `01 §11`; `01 §12 Qn` = open question needing a decision first |
| Use cases | Linked Issues (using the `relates to` link type, pointing at docs) | UC-xx scenarios in `08-use-cases.md` that this ticket serves |

**Priority scale (maps to the Jira defaults Highest through Lowest):**

| Priority | Meaning | Examples here |
|---|---|---|
| Blocker | Pilot cannot launch without it; nothing else unblocks it | (none — pilot ships incrementally behind flags) |
| Critical | Core happy path or fraud/security boundary; blocks other stories | A-01, A-02, A-03, B-01, C-01, C-02a/b/c, D-02, D-02b, D-06 |
| Major | Important for completeness, operability, or UX; shippable one sprint later | A-04, B-02, C-03, D-01, D-03, D-04, D-05, D-07, D-08 |
| Minor | Admin/support hardening; no user impact if deferred | B-03 |
| Trivial | Cosmetic/polish | (none) |

**Story-point scale (Fibonacci, relative effort including tests and review):** 1 = trivial (under half a day) · 2–3 = small (a few days) ·
5 = medium (most of a sprint slot with review) · 8 = large (must split into subtasks, still one sprint) ·
13 = too large (never used here — the old oversized C-02 was split into C-02a/b/c for exactly this reason).

**Personas & roles (who is who):**

| Persona | Who they are | Person or thing? | Stories |
|---|---|---|---|
| Registrant | The individual onboarding themselves — no account yet (Step 1) through just-created account. *The person to be onboarded.* | Person (end user) | A-01…A-04 |
| Registered User | A registrant after account creation, logged in. Carries a KYC status (unverified, then verified). | Person (end user) | B-01, B-02, C-01, C-02x, D-04, D-05 |
| Tenant | The customer organisation (local government unit, agency, or enterprise) — owns the login realm, configuration, and users. Never itself a login identity. | Organisation | Context for all |
| Administrator | Platform or RBAC admin: provisions staff accounts, configures tenants, suspends accounts, reads lifecycle history. Admin accounts are provisioned, never self-registered. | Person (staff) | B-03 |
| Frontliner | Assisted-channel staff who operate the onboarding app on a shared device for walk-in registrants (today's flow; still supported). | Person (staff) | As-is actor; C-01 regression |
| Reviewer | Back-office worker who decides review cases (approve, redo, or reject). Holds the reviewer permission. | Person (staff) | D-01, D-02, D-03 |
| Adjudicator | Back-office biometric specialist who decides identity-match hits. Holds the adjudication permission. | Person (staff) | D-02b |
| Support Analyst | Investigates user journeys through lifecycle and audit exports. | Person (staff) | D-07 |
| Auditor | Compliance reviewer who needs immutable, traceable records. | Person (staff/external) | C-03 |
| Security Engineer | Owns abuse-hardening of the public surface. | Person (staff) | D-06 |
| Release Manager | Owns pilot enablement, feature flags, dashboards, and rollback. | Person (staff) | D-08 |

**Story format:** every story bolds its three parts — **who** (persona), then **action** (capability), then **result** (benefit):
*As a **Registrant**, I want to **open a tenant link** so that **I land in the right organisation**.*

**Terms:** technical terms are defined in plain words where they first appear in each ticket, with the full
definitions in `07-glossary.md`. Key shorthands: HLR = the approved High-Level Requirements source document
(`HLR §n` cites its sections); FR = one numbered Functional Requirement from the HLR (FR-01…FR-33);
NFR = quality attributes such as security and auditability (HLR §21).

**Agile notes:** every story below is a vertical, independently demoable slice with testable acceptance criteria
(checkboxes = your UAT script). Predecessors are listed under Dependencies — the build order at the bottom is already
topologically sorted. Sizing assumes one backend or frontend dev plus reviewer; cross-cutting stories (C-01, D-04) need
both disciplines and should be swarmed or split into backend/frontend subtasks at planning.

---

## Feature A — Public Self-Registration (Step 1) · label `self-registration`

### A-01 — Tenant Resolution and Portal Bootstrap [Backend and Frontend]
- **Title:** `A-01 Self-Service Tenant Resolution by Slug and Portal Bootstrap`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** auth-service, auth-portal
- **User story:** As a **Registrant**, I want to **open a tenant-specific registration link** so that **I register under the correct organisation and login realm without needing an existing account**.
- **Requirements:**
  1. New endpoint `POST /public/tenants/resolve` that takes a slug (the tenant's short readable key in the link, for example `navotas-demo`) and returns the tenant ID, login realm, display name, password rules, and supported channels (email or SMS). It must be rate-limited, and unknown or disabled slugs must get an identical generic reply so nobody can probe which tenants exist.
  2. New endpoint `GET /public/config/{slug}` returning cached portal setup for that tenant (allowed ID types, required form fields, terms versions, anti-bot key).
  3. New `tenants.slug` column with a `tenants_by_slug` lookup table, plus per-tenant settings rows seeded for pilot tenants.
  4. Portal shell under `/self-service/:slug/*` that renders a generic "registration not available" page for disabled or unknown slugs.
- **Acceptance criteria:**
  - [ ] Valid slug returns tenant, realm, and policy in under 300 ms (95th percentile) with a `RESOLVE_TENANT` audit entry.
  - [ ] Unknown versus disabled slugs are indistinguishable in response shape and timing (within 10 percent).
  - [ ] No client secret or internal IDs leak in responses.
  - [ ] Burst traffic trips the rate limit to a `429 Too Many Requests` reply with a retry delay.
- **Use cases:** UC-01 · **FR:** FR-01, FR-02 · **Dependencies:** PM input (pilot slug list); blocks A-02, A-03, A-04.

### A-02 — Registration Initiation and OTP Dispatch [Backend]
- **Title:** `A-02 Pending Registration With OTP Initiation and Resend`
- **Type:** Story · **Priority:** Critical · **Story Points:** 8 · **Component(s):** auth-service
- **User story:** As a **Registrant**, I want **a one-time passcode (OTP) sent to my email address or mobile number** so that **I can prove I control it before any account exists**.
- **Requirements:**
  1. New endpoint `POST /public/registration/initiate` that validates the identifier, password, terms acceptance, and anti-bot token, silently tolerates already-registered identifiers (same reply either way), then writes a pending registration (a temporary pre-account record with a 30-minute self-delete timer — not a user account) and a hashed OTP record, and sends the code by email or SMS.
  2. New endpoint `POST /public/registration/resend` with a 60-second cooldown and a maximum of 5 resends; each resend invalidates the previous code.
  3. New endpoint `GET /public/registration/status` for safe polling (time left, resend availability — no personal data).
  4. Every reply identical whether the identifier is new or taken (no account-existence oracle); all steps audited.
- **Acceptance criteria:**
  - [ ] No user record and no identity-system (Keycloak) entry is created at this stage (verified by database and login-system query).
  - [ ] The OTP is stored one-way hashed, expires per tenant setting (5 to 10 minutes), and each code works only once.
  - [ ] Resend past cooldown or limit returns `429` without revealing any state.
  - [ ] Email and SMS paths covered by contract tests (SMS through a mock provider adapter).
- **Use cases:** UC-01, UC-02 · **FR:** FR-01, FR-02, FR-03, FR-06 · **Dependencies:** blocked by A-01; external `01 DEP-02` (SMS vendor), `01 DEP-03` (email throughput).

### A-03 — OTP Verification and Account Creation [Backend]
- **Title:** `A-03 OTP Verification With Keycloak and Cassandra Account Creation`
- **Type:** Story · **Priority:** Critical · **Story Points:** 8 · **Component(s):** auth-service
- **User story:** As a **Registrant who has proven control of my contact** (the email address or mobile number I registered with, confirmed by entering the OTP), I want **my account created automatically** so that **nobody can squat my contact and I can log in immediately**.
- **Requirements:**
  1. New endpoint `POST /public/registration/verify` that checks the code with a timing-safe comparison, counts attempts, and handles expired or exhausted codes with clear terminal replies.
  2. On success only, in this order: create the user in Keycloak (the identity system that checks passwords at login), then the user record plus username lookup in Cassandra, assign the tenant's default role, and append `ACCOUNT_CREATED` and `ACTIVE_KYC_NOT_VERIFIED` entries to the lifecycle history (the append-only audit log of the account's journey). Audit the registration.
  3. If creation fails halfway, the half-created login is disabled and a cleanup reconciliation job heals leftovers — no orphaned enabled logins.
  4. Passwords stored with a memory-hard hash (decision `01 §12 Q1: Argon2id versus bcrypt`); never reversibly encrypted.
- **Acceptance criteria:**
  - [ ] Entering the same code twice fails the second time (single use).
  - [ ] Exhausted or expired flows return terminal codes; abandoned pending records self-delete on expiry.
  - [ ] The Keycloak identity links to the Cassandra user and login works immediately.
  - [ ] Injected partial failure leaves no enabled half-created login (cleanup job heals within the agreed time).
- **Use cases:** UC-01 · **FR:** FR-03, FR-04, FR-05, FR-06 · **Dependencies:** blocked by A-02; external `01 DEP-01` (Keycloak Admin), decision `01 §12 Q1` (hashing).

### A-04 — Registration Pages: Create, Verify, and Done [Frontend]
- **Title:** `A-04 Self-Service Register, Verify, and Done Pages`
- **Type:** Story · **Priority:** Major · **Story Points:** 5 · **Component(s):** auth-portal
- **User story:** As a **Registrant**, I want **clear create-account and verify-code screens matching the approved mockups** so that **I can self-register without help**.
- **Requirements:**
  1. Three pages: `/register` (email-or-mobile selector, identifier, password plus confirmation, terms and privacy checkboxes with version stamps, anti-bot widget), `/verify` (code boxes, resend timer, masked destination such as `j…@…` or `09******1234`), `/done` (account-created confirmation with login button).
  2. Client-side validation mirrors the server; destinations always masked (partially hidden).
  3. Errors mapped to the `03/06` catalogue wording (nothing that reveals whether an account exists).
- **Acceptance criteria:**
  - [ ] Matches the HLR §5 layouts on desktop and mobile widths.
  - [ ] Full flow (resolve, initiate, verify, done, login) passes end to end on staging.
  - [ ] Accessibility: labels, focus order, one-time-code autofill (`autocomplete=one-time-code`).
- **Use cases:** UC-01, UC-02 · **FR:** FR-01, FR-02 · **Dependencies:** blocked by A-01 (contracts); attach HLR §5 mockups in Jira.

---

## Feature B — Login, KYC Access Decision, and RBAC Attribute · label `kyc-decision`

### B-01 — Login Enrichment and KYC Decision [Backend]
- **Title:** `B-01 Login Response With KYC Status and OIDC Claims`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** auth-service
- **User story:** As a **Registered User**, I want to **be routed to identity verification only when needed** so that **features that do not need verification work for me immediately**.
- **Requirements:**
  1. Login (`POST /token`, request unchanged) additionally returns the user's KYC status, current verification-attempt number, and next action — start, continue (with onboarding link), wait (under review), or blocked (with reason).
  2. The OIDC identity token (the signed token apps use to know who logged in) gains machine-readable KYC claims (`kyc_verified` true/false and `kyc_status`) so downstream applications can enforce access without trusting the portal UI.
  3. App listing (`GET /apps`) and access-rights (`GET /user/access-rights`) label each entry with whether verification is required, and the permission filter recognises the new self-onboarding permission.
- **Acceptance criteria:**
  - [ ] A login with status `REDO_REQUIRED` returns a continue action plus onboarding link; `KYC_REVIEW` returns wait with no relaunch.
  - [ ] A direct API call to a verification-gated function with `kyc_verified=false` is denied even when bypassing the UI (negative test).
  - [ ] Non-gated applications keep working when the KYC back office is down (deny only where verification is required).
- **Use cases:** UC-03 · **FR:** FR-07, FR-08, FR-09, FR-10 · **Dependencies:** blocked by A-03 (lifecycle tables and stored status).

### B-02 — KYC-Gated App UX and Status Page [Frontend]
- **Title:** `B-02 KYC-Aware App List and Status Page`
- **Type:** Story · **Priority:** Major · **Story Points:** 3 · **Component(s):** auth-portal
- **User story:** As a **Registered User**, I want to **see which applications need verification and what my current verification state is** so that **I know what to do next**.
- **Requirements:** Application tiles filtered and badged by verification requirement; a status page covering start, continue, under-review, redo, rejected, and blocked states; redo button deep-links into onboarding with the attempt hint.
- **Acceptance criteria:**
  - [ ] Verified users see the full list; others see the filtered list plus a call to action; review and reject states render correctly.
- **Use cases:** UC-03, UC-05 · **FR:** FR-09, FR-10, FR-11, FR-12 · **Dependencies:** blocked by B-01.

### B-03 — Suspend, Unsuspend, and Lifecycle History Lookup [Backend]
- **Title:** `B-03 Admin Suspend and Lifecycle History Lookup`
- **Type:** Story · **Priority:** Minor · **Story Points:** 3 · **Component(s):** auth-service
- **User story:** As an **Administrator**, I want to **temporarily suspend or reactivate accounts and read their lifecycle history** so that **I can handle abuse and support cases**.
- **Requirements:** Admin endpoints to suspend and unsuspend a user (distinct from automatic lockout and from deletion), plus a paged read-only endpoint for that user's lifecycle history.
- **Acceptance criteria:**
  - [ ] A suspended user's login is denied with a reason; the history shows who acted and when.
- **Use cases:** UC-09 · **FR:** FR-06 (audit), HLR §3 · **Dependencies:** blocked by A-03 (history table).

---

## Feature C — Self-Service KYC Onboarding (Step 2, Same Module) · label `kyc-onboarding`

### C-01 — OWA Self-Mode Bootstrap [Frontend and Backend]
- **Title:** `C-01 OWA Self Mode With Applicant Self-Bootstrap`
- **Type:** Story · **Priority:** Critical · **Story Points:** 8 · **Component(s):** owa, kyc-api
- **User story:** As an **unverified Registered User**, I want to **start identity verification by myself in the same onboarding module the frontliners use** so that **I do not need in-person assistance**.
- **Requirements:**
  1. A public onboarding route `/self/:slug` in the onboarding web app (OWA) that resolves the tenant at load time, accepts the end user's own login token (not a frontliner's), behind a `self` launch-mode flag.
  2. A `POST /kyc/self/bootstrap` endpoint that creates the caller's verification applicant record (or resumes the open one — exactly one open verification attempt per applicant): session binding via new `applicant.self_user_id`, durable identity binding via existing `PUT /person/{person_id}/linked_user_ids` (check first with `HEAD .../linked_user`).
  3. Assisted (frontliner-driven) mode byte-for-byte untouched, proven by the existing regression suite staying green.
- **Acceptance criteria:**
  - [ ] Opening `/self` without any frontliner login succeeds with the user's own login token.
  - [ ] Reopening resumes the open attempt (same attempt number); no duplicate applicant records.
  - [ ] The assisted end-to-end flow still passes unchanged.
- **Use cases:** UC-04, UC-10 · **FR:** FR-11, FR-12, FR-13 · **Dependencies:** blocked by A-01 (slug resolution), B-01 (auth model); relates to assisted regression suite.

### C-02a — Self Capture, Document Inspection, and OCR Provenance [Backend and Integration]
- **Title:** `C-02a Self ID Capture With Document Inspection and OCR Provenance`
- **Type:** Story · **Priority:** Critical · **Story Points:** 8 · **Component(s):** kyc-api, owa
- **User story:** As a **Registered User**, I want to **photograph my ID by myself with the same fraud checks as assisted onboarding** so that **my document is verified remotely**.
- **Requirements:**
  1. OWA captures ID front and back (portrait crop included); document inspection — tamper, machine-readable-zone, and expiry checks — via the provider selected in `01 §12 Q9` (no Innovatrics/DOT in the current stack).
  2. Keep three-way provenance for extracted data: the value read by OCR (text recognition, same TBD provider), the value confirmed or corrected by the user, and the value obtained from external verification — stored separately in `kyc_attempt.ocr_provenance`, never overwriting each other.
  3. Evidence files go to HFiles through the existing validated upload (`POST /person/{person_id}/docs/{transaction_id}`, extended for attempt linkage if needed); the database keeps only file references and hashes; no raw personal data in logs.
- **Acceptance criteria:**
  - [ ] A tampered or expired sample ID is flagged exactly as in assisted mode (parity test against the selected provider).
  - [ ] User corrections never overwrite OCR-extracted values (proven by querying all three).
  - [ ] Uploaded evidence is retrievable per attempt via the docs endpoints.
- **Use cases:** UC-04 · **FR:** FR-13, FR-14, FR-19 (partial) · **Dependencies:** blocked by C-01, C-03; decision `01 §12 Q9` (inspection/OCR provider).

### C-02b — PhilSys, Liveness, and Biometric Matching [Backend and Integration]
- **Title:** `C-02b PhilSys Verification With Liveness and Biometric Matching`
- **Type:** Story · **Priority:** Critical · **Story Points:** 8 · **Component(s):** kyc-api, owa
- **User story:** As a **Registered User**, I want **my face and national ID checked against official sources** so that **impersonation is caught**.
- **Requirements:**
  1. PhilSys eVerify QR passthrough (`POST /psa/query/qr`, invoked only when the submitted ID is a Philippine National ID); its result is stored as a verification event and never overwrites user data. Other ID types skip this step cleanly.
  2. Selfie capture with quality checks; the liveness session id is forwarded to the eVerify gateway (no standalone liveness engine in the back office).
  3. One-to-many (1:N) biometric search via `POST /customers/identify/face` — comparing the face against enrolled faces to catch a second account by the same person. Any hit sends the attempt to human review with a pending adjudication; nothing is ever auto-merged or auto-rejected.
- **Acceptance criteria:**
  - [ ] The national-ID path invokes PhilSys and stores the event; other ID types skip cleanly.
  - [ ] A duplicate biometric always yields `UNDER_REVIEW` plus a review case; the user sees only a generic "under review" message.
- **Use cases:** UC-04, UC-06, UC-07 · **FR:** FR-15, FR-16, FR-17, FR-18 · **Dependencies:** blocked by C-01, C-03.

### C-02c — Additional Info, Attempt Submit, and Status Callback [Backend and Integration]
- **Title:** `C-02c Additional Info Form With Attempt Submit and Status Callback`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** kyc-api
- **User story:** As a **Registered User**, I want to **submit my verification and receive a decision** so that **I finish onboarding without waiting on staff**.
- **Requirements:**
  1. A tenant-configurable extra-information form (fields come from configuration, never hard-coded per tenant).
  2. A submit endpoint for the attempt that runs the decision rules: either approved, or a review case for a human.
  3. A status callback from the KYC back office to the auth service carrying an idempotency key (a unique key per event so retried deliveries are applied exactly once) with a guard so stale events never move the status backwards.
  4. Re-point the auth-service KYC client (`KYCAPIUtil` + `kyc-api-config`) from legacy paths to the Spring Boot paths (`/spring/gen-kyc-api/...`); keep the pull-based tenant/secret/settings lookups untouched.
- **Acceptance criteria:**
  - [ ] An all-checks-pass submission marks the user verified in auth plus lifecycle history within the agreed time; retried callbacks are safe.
  - [ ] An exception submission creates a review case and marks the user under review; redelivered callbacks never regress the status.
  - [ ] No auth-service call path still targets a legacy KYC endpoint (verified by config + contract test).
- **Use cases:** UC-04, UC-05 · **FR:** FR-19, FR-20, FR-21 · **Dependencies:** blocked by C-02a, C-02b.

### C-03 — Attempt Versioning and Applicant Binding [Backend]
- **Title:** `C-03 KYC Attempt Versioning With Auth-Side Mirror`
- **Type:** Story · **Priority:** Major · **Story Points:** 5 · **Component(s):** kyc-api, auth-service
- **User story:** As an **Auditor**, I want **every onboarding submission preserved as its own numbered attempt** so that **the full history is traceable**.
- **Requirements:** An attempt table on the KYC side plus a lightweight mirror on the auth side; each new attempt marks older ones SUPERSEDED (kept for audit, no longer current — data never overwritten or deleted); the latest approved attempt is the user's current verified record.
- **Acceptance criteria:**
  - [ ] Two sequential attempts retain both evidence sets; the current-record query returns the latest approved one.
- **Use cases:** UC-04, UC-05 · **FR:** FR-21, FR-30, FR-31 (HLR §15) · **Dependencies:** blocked by C-01.

---

## Feature D — Review Queue, Adjudication, Redo, Notifications, and Hardening · label `review-ops`

### D-01 — Review-Case API and Queue [Backend]
- **Title:** `D-01 Review Case Queue With Scoped Evidence and SLA`
- **Type:** Story · **Priority:** Major · **Story Points:** 8 · **Component(s):** kyc-api
- **User story:** As a **Reviewer**, I want **a queue of exception cases with right-sized evidence** so that **I can triage without seeing raw biometric data**.
- **Requirements:** A paged review-queue endpoint with filters (tenant, status, issue type, priority); a case-detail endpoint returning a scoped evidence bundle (match results and confidence scores, never raw biometric templates); automatic priority and deadline (SLA) per issue type; case assignment to reviewers.
- **Acceptance criteria:**
  - [ ] Queue pagination and filters work at volume seed; the evidence response contains no biometric templates by default.
- **Use cases:** UC-08 · **FR:** FR-22, FR-23, FR-24, FR-25, FR-33 · **Dependencies:** blocked by C-03.

### D-02 — Approve, Redo, and Reject Decisions With Callback [Backend]
- **Title:** `D-02 Reviewer Approve, Redo, and Reject Decisions`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** kyc-api
- **User story:** As a **Reviewer**, I want **exactly three outcomes — approve, ask for redo, or reject** so that **every user gets a clear next step**.
- **Requirements:** Three transition endpoints sharing a tenant-configured list of reason codes; redo carries plain-language user instructions (never technical internals); every decision writes KYC-side audit, notifies the user, and emits the auth-service status callback.
- **Acceptance criteria:**
  - [ ] Each decision moves the attempt, the applicant record, and the auth lifecycle together (eventually consistent, idempotent).
  - [ ] Redo notifications contain no biometric or conflict internals.
- **Use cases:** UC-05, UC-08 · **FR:** FR-26, FR-27, FR-32 · **Dependencies:** blocked by D-01.

### D-02b — Back-Office Biometric Adjudication [Backend]
- **Title:** `D-02b Biometric Hit Adjudication by the Back Office`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** kyc-api
- **User story:** As a **back-office Adjudicator**, I want **identity-match hits to wait in pending state for my verdict** so that **no duplicate is auto-cleared and no user learns they were flagged as a possible duplicate**.
- **Requirements:** A candidate-comparison endpoint (probe photo versus matched candidates, side by side) restricted to adjudication-permission holders, with gated access, watermarking, and audit; a verdict endpoint accepting same person, different person, or inconclusive, routed as follows — different person clears the hit and resumes automation, same person converts to reject-or-fraud handling with re-registration cool-down (the same contact is blocked from registering again for a configured number of days), inconclusive asks the user for a fresh capture. The verdict handler must also drive the existing `PATCH /customers/adjudication` (`request_id` + per-hit `hit_subject_id → UNIQUE|DUPLICATE`: different person sends `UNIQUE`, same person sends `DUPLICATE`) so the ABIS leaves adjudication-waiting state — the new case API adds the workflow, permission, routing, and audit the engine call lacks. Server rule: a duplicate case cannot be approved without a prior different-person verdict. All user-facing copy stays a generic "under review".
- **Acceptance criteria:**
  - [ ] A hit attempt stays under review with no user-visible duplicate hint until a verdict exists.
  - [ ] Different person resumes the automated path; same person routes to reject or fraud handling plus cool-down; inconclusive routes to redo with fresh capture.
  - [ ] Approving an unadjudicated duplicate case is rejected by the server; every verdict is audited with the actor.
  - [ ] The engine-level adjudication call is issued per verdict (verified in engine logs); a case cannot close while the engine still waits.
- **Use cases:** UC-06, UC-07 · **FR:** FR-22, FR-23, FR-25, FR-26, FR-32 · **Dependencies:** blocked by D-01.

### D-03 — Reviewer UI [Frontend]
- **Title:** `D-03 Review Queue and Case Detail Screens`
- **Type:** Story · **Priority:** Major · **Story Points:** 5 · **Component(s):** reviewer-ui
- **User story:** As a **Reviewer**, I want **queue and case-detail screens matching the approved layout** so that **I can decide cases efficiently**.
- **Requirements:** Queue table (case, user reference, issue, priority, status, deadline); case detail (account, submitted ID, OCR with provenance, national-ID check result, liveness, biometric result, extra info, audit trail, prior attempts, potential match); reviewer areas gated by the reviewer permission and the adjudication view by the adjudication permission. Host: new review routes in `svi-sso-admin-panel` (preferred — it already hosts users/roles/tenants pages) or the KYC back-office admin UI; confirm at sprint planning.
- **Acceptance criteria:**
  - [ ] Unauthorized users get `403 Forbidden`; all decisions are audited with the actor.
- **Use cases:** UC-08 · **FR:** FR-24, FR-25, FR-33 · **Dependencies:** blocked by D-01, D-02, D-02b.

### D-04 — Re-Onboarding (Redo) Flow [Frontend and Backend]
- **Title:** `D-04 Redo Notification With Login and New Attempt`
- **Type:** Story · **Priority:** Major · **Story Points:** 5 · **Component(s):** auth-portal, owa, kyc-api
- **User story:** As a **Registered User asked to redo verification**, I want to **log in and retry in the same module, retaking my ID and selfie or choosing another supported ID** so that **I can complete verification**.
- **Requirements:** Redo notification triggers (email, SMS where configured) with plain-language instructions; login returns a continue action deep-linking into onboarding; the module opens a brand-new attempt number and re-runs verification; prior attempts untouched.
- **Acceptance criteria:**
  - [ ] End to end: reviewer asks redo, user is notified, logs in, retries, is approved, and gains application access.
- **Use cases:** UC-05 · **FR:** FR-12, FR-27, FR-28, FR-29 · **Dependencies:** blocked by D-02 (redo decision), B-02 (status page call to action).

### D-05 — Notifications (OTP and Lifecycle) [Backend]
- **Title:** `D-05 Notification Templates With Delivery Audit`
- **Type:** Story · **Priority:** Major · **Story Points:** 5 · **Component(s):** auth-service, platform
- **User story:** As a **Registered User**, I want **timely verification and decision notifications** so that **I always know what to do next**.
- **Requirements:** A versioned template set (OTP code, review received, redo requested, approved, rejected); a swappable SMS provider adapter replacing the current stub; delivery receipts (message IDs) audited; per-tenant sender names and branding.
- **Acceptance criteria:**
  - [ ] All five notification types render with correct tenant branding; delivery failures retry and raise alerts.
- **Use cases:** UC-01, UC-05 · **FR:** FR-27 (HLR §17) · **Dependencies:** blocked by A-02 (OTP wording), D-02 (decision wording); external `01 DEP-02`.

### D-06 — Abuse Hardening: Rate Limits and CAPTCHA [Backend]
- **Title:** `D-06 Public Surface Hardening Against Abuse`
- **Type:** Story · **Priority:** Critical · **Story Points:** 5 · **Component(s):** auth-service
- **User story:** As a **Security Engineer**, I want **public endpoints protected by rate limits, anti-bot checks, and anti-probing replies** so that **codes and logins cannot be brute-forced and accounts cannot be enumerated**.
- **Requirements:** Token-bucket limits per identifier, IP address, and device; anti-bot (CAPTCHA) check on registration start with escalation after failures; identical replies and normalised timing for sensitive outcomes; every blocked attempt audited.
- **Acceptance criteria:**
  - [ ] Simulated burst and credential-stuffing attacks are blocked; legitimate flows unaffected; responses and timing reveal nothing.
- **Use cases:** UC-01, UC-02 · **FR:** NFR security (HLR §21) · **Dependencies:** blocked by A-02, A-03 (endpoints to protect); implements `06 §3–§4`.

### D-07 — Audit and Reporting [Backend]
- **Title:** `D-07 Audit Completeness With Support Exports`
- **Type:** Story · **Priority:** Major · **Story Points:** 3 · **Component(s):** auth-service, kyc-api
- **User story:** As a **Support Analyst**, I want **the full registration-to-verification journey queryable** so that **I can investigate any case**.
- **Requirements:** Every event in the HLR audit list emitted (registration, code sent and verified, login, attempt started and submitted, ID capture, OCR result, external checks, liveness, biometric match, decision, exception, adjudication verdict, reviewer decision, notifications, status changes); paged exports for support and compliance.
- **Acceptance criteria:**
  - [ ] The golden-path run reproduces the HLR §3 example history (nine ordered rows) with attempt linkage.
- **Use cases:** UC-09 · **FR:** FR-06, FR-32 (HLR §21 auditability) · **Dependencies:** final sweep after all features; implements `06 §7`.

### D-08 — Pilot Rollout and Runbooks [Operations and Docs]
- **Title:** `D-08 Pilot Tenant Enablement With Rollback Plan`
- **Type:** Task · **Priority:** Major · **Story Points:** 2 · **Component(s):** platform
- **User story:** As a **Release Manager**, I want **per-tenant feature flags, health dashboards, and a rehearsed rollback** so that **the pilot launches safely**.
- **Requirements:** Per-tenant enablement flags; dual-read monitoring during migration; alerts for status-callback lag and review/adjudication deadline breaches; a rollback procedure that only flips flags (no database rollback needed).
- **Acceptance criteria:**
  - [ ] Pilot tenant onboarded; rollback drill passes; code-delivery, callback-lag, and review-deadline dashboards live.
- **Use cases:** UC-01, UC-10 · **FR:** — (rollout) · **Dependencies:** blocked by A-01 (flags); runs alongside pilot.

---

## Pre-creation verification — proven not yet built

Checked 2026-09-14 against the code in `svi-authentication-springboot-kyc` (auth service),
`generic-kyc-owa` (onboarding app), `kyc-api` (KYC back office), and
`svi-authenticationportal-react-kyc` (portal). "Nearest existing thing" shows what was inspected
so a reviewer can re-verify in seconds. This proves absence **in code** — the PM must still search
Jira for duplicate summaries before creating (checklist below).

| Ticket | Would already exist if… | Evidence it does not (nearest existing thing) |
|---|---|---|
| A-01 | resolve/config endpoints, slug storage, portal shell | No `slug` anywhere in auth-service src; no `/public/` routes in controllers; no `/self-service/` or `/register` route in portal |
| A-02 | initiate/resend/status, pending and OTP tables | No `pending_reg` or `registration_otp` in auth-service; existing `totp` table is forgot-password scoped (`AuthenticationServiceImpl.java:649`, `PASSWORD_RESET` session) |
| A-03 | verify endpoint, post-OTP creation, reconciliation | No verify-for-registration path; Keycloak user creation exists only in one-off bootstrap (`InitialSetupUtils`), never behind OTP |
| A-04 | register/verify/done pages | No `/register` route in portal (only unused endpoint templates in JSON config plus an unused CSS class) |
| B-01 | KYC in login response and identity token | Login returns `customer_id` only (`AuthenticationServiceImpl.java:177,348`); identity-token claims are `iss, sub, aud, iat, exp, nonce, email, name, preferred_username` only (`OidcIdTokenService.java:53-70`) |
| B-02 | verification status page | No status page in portal; app list has no verification awareness |
| B-03 | suspend endpoints, lifecycle read API | No `suspend` anywhere in auth-service src; no history table |
| C-01 | self route, applicant self-bootstrap | No `/self` route in OWA routing (every "self" hit is "selfie"); no bootstrap, `self_user_id`, or person-link wiring in the back office |
| C-02a | self capture pipeline + TBD inspection provider | No self-service capture orchestration; no document-inspection/OCR provider in the back office (Q9) |
| C-02b | PhilSys QR, liveness passthrough, and 1:N wiring for self flow | `POST /psa/query/qr`, `/customers/identify|enroll/face` exist and are reused; no self-flow wiring or hit-to-case path |
| C-02c | attempt submit, status callback, client re-point | No `callback` push from KYC back office to auth service (`KYCAPIUtil` only pulls); auth-service client still targets legacy paths; no self submit endpoint |
| C-03 | attempt tables | No `kyc_attempt` in any analysed repo |
| D-01 | review-case API | Only a generic status-patch endpoint plus fraud-list reads; no case resource |
| D-02 | approve/redo/reject transitions | No transition endpoints; no reason-code list |
| D-02b | case-level verdict API | Only the engine-level `PATCH /customers/adjudication` exists (per-hit `UNIQUE|DUPLICATE`, no case, no permission, no routing); the ticket wraps it and must call it |
| D-03 | reviewer screens | No dedicated reviewer frontend in the analysed sibling repos (host surface still to confirm) |
| D-04 | redo notify plus deep link plus new attempt | None of the three exist |
| D-05 | SMS adapter, five templates | SMS channel accepted but stubbed (`AuthenticationServiceImpl.java:730` logs "not yet fully implemented"); only the forgot-password email template exists |
| D-06 | rate-limit filter, server-side anti-bot | None in auth-service; portal has zero anti-bot; OWA has a widget but disabled (`environment.ts:118 `enableRecaptcha: false`) on assisted-only forms |
| D-07 | new audit event codes | `AuditEvents` covers existing flows only; none of the new codes exist |
| D-08 | self-registration flags and settings | No `SELF_REG` or `self-registration` anywhere in auth-service src |

## Ticket creation checklist (for PM)

- [ ] Search Jira first: query summaries and descriptions for `self-service`, `self-registration`, `CPS-272`, and each ticket ID (`A-01`…`D-08`) to rule out duplicates already filed by another author. Code-level absence (table above) does not prove Jira-level absence.
- [ ] Create Epic link: each ticket's Epic Link = CPS-272. Total scope ≈ 114 points across 21 tickets.
- [ ] Set Components/Labels per ticket; attach `01`–`04` links to every backend story; attach HLR §5 mockups to A-04 and HLR §16 table to D-03.
- [ ] Build order (topologically sorted — respect `is blocked by`): A-01, A-02, A-03, A-04, B-01, C-01, C-03, C-02a, C-02b, C-02c, D-01, D-02, D-02b, D-04, D-03, B-02 and B-03, then D-05, D-06 and D-07, then D-08.
- [ ] Indicative sprint buckets (adjust to velocity; split 8-pointers into subtasks at planning):
  - Sprint 1: A-01 and D-08 (flags), plus decisions `01 §12 Q1 to Q3`.
  - Sprint 2: A-02, A-03, D-06 (harden as you build).
  - Sprint 3: A-04, B-01, B-02.
  - Sprint 4: C-01, C-03, B-03.
  - Sprint 5: C-02a, C-02b.
  - Sprint 6: C-02c, D-01.
  - Sprint 7: D-02, D-02b, D-04.
  - Sprint 8: D-03, D-05, D-07, and pilot (D-08).
- [ ] Phase plan and decision owners: `09-rollout-plan.md`; confirm Q1–Q3 before sprint 1.
- [ ] Definition of Ready per story: contract section in `03` and the schema section in `04` reviewed; open questions in `01 §12` resolved or explicitly deferred.
