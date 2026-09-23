# CPS-272 — TDD Main: Self-Service User Registration with KYC Integration

**Status:** Draft for Lead/TL review · **Type:** Technical Design (no code in this ticket) · **Sources:** High-Level Requirements (FR-01…FR-33); workspace analysis of `svi-authentication-springboot-kyc`, `generic-kyc-owa`, `kyc-api`, `svi-authenticationportal-react-kyc`.

---

## 1. Objective & Scope

### 1.1 Objective
Enable **unassisted self-service registration** for whitelisted tenants: any person can create their own RBAC account with a verified email/mobile (Step 1: *"do you control this contact?"*), then complete identity verification through the **same KYC onboarding module** used by frontliners (Step 2: *"are you the legitimate person?"*), with KYC status enforced as an **authorization attribute** alongside roles/permissions.

### 1.2 In scope
- Public self-registration architecture + flow (tenant-scoped portal).
- Tenant identification + Keycloak realm integration.
- OTP verification + account creation (email + mobile).
- Integration with existing auth-service/Cassandra/RBAC (no RBAC redesign).
- Integration with the current KYC back office (`svi-kyc-api-springboot-kyc`: person registry, MegaMatcher ABIS face verify/identify/enroll/adjudicate, PhilSys eVerify QR passthrough, HFiles evidence refs).
- KYC status + account lifecycle + immutable history + attempt versioning.
- Required API/interface + database changes (design-level schemas/contracts).
- Security, error handling, auditability, configurability.
- Ticket breakdown (see `05-user-stories-tickets.md`).

### 1.3 Out of scope
- Redesign of OCR/liveness/biometric/PhilSys algorithms (reuse as-is).
- BPO work-item engine redesign (lifecycle history is the system-of-record; BPO applicability noted in §10).
- Native mobile SDKs (responsive web only in this phase; bridge hooks reserved).
- Migration of legacy `svi-authentication-java-kyc` callers (flagged as dependency).

### 1.4 FR traceability
| Requirement group | FRs | Design section |
|---|---|---|
| Account registration | FR-01…FR-06 | §4, §5, `03 §2`, `04 §2–3` |
| Auth + KYC decision | FR-07…FR-12 | §6, `02` seq diagrams |
| KYC module | FR-13…FR-21 | §7, `03 §5` |
| Exception + review | FR-22…FR-33 | §8, `03 §5`, `04 §4` |
| NFR security/audit/config | HLR §21 | `06-security-errorhandling.md` |

---

## 2. Current State & Gap Analysis (evidence-based)

### 2.1 auth-service (`svi-authentication-springboot-kyc`)
- Stack: Spring Boot 4.0.5, Java 25, Cassandra (`auth_system` keyspace), Keycloak (one realm per `tenants.realm_id`), context path `/spring/auth-services`.
- `POST /user/register` (`UserController:60`) is gated by `@Permissions + @Authenticated + @Authorized` — **admin/frontliner only**. No public registration path exists.
- Auth: `POST /token` (password grant via Keycloak), `POST /login/face` (password-decrypt + `KYCAPIUtil.verifyFaceBiometrics`), `GET /token?sub&session_state` (refresh), `POST /logout`, OIDC provider (`/oidc/authorize|token|jwks`, `AuthCode` table, `svi_session` cookie).
- RBAC: `users → user_roles / user_groups → group_roles → role_permissions → applications/permissions`, plus group-scoped roles (`scoped_roles*`). Enforcement in `AppIDFilter → PermissionsFilter` (server-side traversal, no ALLOW FILTERING). `RoleService.getApps` / `UserService.getAccessRights` already skip `requiresKYC && !verified` — **the gating hook exists, but `verified` is derived ad hoc from `KYCAPIUtil.getProfileStatus` per request**.
- Tenant: `tenants{tenant_id, realm_id, client_id, client_secret(server-only), is_active, is_deleted}`; secret injection is server-side (CPS-164). `TenantSettingsProvider` serves `KYC_VERIFICATION_ENABLED, SESSION_TIMEOUT…` from `tenant_settings`.
- OTP: `totp{tenant_id,user_id,otp_code(hashed HMAC-SHA256), attempts, channel, expires_at, is_expended}` + `OTPUtils` + `EmailSenderUtils`; used **only for forgot-password** (`generateOTP → verifyOTP → PASSWORD_RESET session → forgotPassword`). SMS send is stubbed/logged. `GenerateOTPRequestDTO` resolves by `username+tenantId` (good anti-takeover pattern to reuse).
- User entity: `users{username,password(encrypted),email,mobile,applicant_id,linked_person_id,is_active,is_deleted,is_blocked,…}` — **no `kyc_status` column, no lifecycle history table, no pending-registration concept**.
- Audit: dual layer — `AuditLoggerFilter` (request log) + `audit_trail{tenant_id,audit_id,event,old_data,new_data JSON,ip}` via `AuditTrailUtils` (40+ `AuditEvents`).

### 2.2 Onboarding web app (`generic-kyc-owa`, Angular 16)
- Routes driven by `pageRoutingConfig.json`: `landing → getStarted(consent) → idSelect → id capture → selfie → (biometrics) → onlineAppForm → review → submit → lastPage(QR/email)`.
- Backend calls (legacy context paths, to be re-pointed for self mode): `POST /kyc/submit/hfiles` (FormData with `ID_FRONT_IMG/VID, SELFIE_IMG/VID…`), face calls, `POST /get-qrcode|send-email|send-sms`, `GET /getcounselordetails|options`, PhilSys `/psa/query/qr` for PNID. Document inspection + OCR provider is TBD (no Innovatrics/DOT in the current stack — see Q9).
- **Gaps for self-service:** (a) bootstrap requires Keycloak `login-required` on a **single baked-in realm** (`keycloakConfiguration.json`); (b) **no tenant headers/params**; multi-tenancy = rebuild per tenant; (c) **no applicant bootstrap API** — frontliner just drives the same device; (d) contact-info page is local-only; `ng-otp-input` is installed but unused.

### 2.3 KYC back office (`svi-kyc-api-springboot-kyc`, Spring Boot 4 / Java 25)
- Current service (replaces the legacy Jersey `kyc-api`; no MariaDB, Innovatrics/DOT, or OCR). Context path `/spring/gen-kyc-api`. Verified on `origin/development` (post-merge HEAD): the checked-out skeleton is stale — implement against the HEAD, re-verify before build.
- Stores (Cassandra `customer_kyc` + `svi_person_db`, HFiles evidence refs, Solr 7.5 person-search core): `applicant{tenant_id, applicant_id, biographics, application_status, current_workflow_status, kyc_verification_result}`, `facedb_result{tenant_id, subject_id, applicant_id, encounter_id, duplicate_id, hit_score}`, `person{tenant_id, person_id, linked_user_id MAP, …}` + `person_by_contact` / `person_by_identity` indexes, `svi_personid_synonym` (person↔biometric bridge), `audit_trail`.
- Real endpoints: `POST /customers/verify/face` (1:1), `POST /customers/identify/face` (1:N or by person_id), `POST /customers/enroll/face` (`SUCCESS`/`DUPLICATE_FOUND`/`ADJUDICATION_WAITING`), `PATCH /customers/update/face`, `PATCH /customers/adjudication` (per-hit `UNIQUE|DUPLICATE`), `GET /customers/biometrics/face`, `POST /customers/save/transaction`, `GET/POST /person` (CRUD + Solr search), `GET /person/{id}[/transactions]`, `HEAD /person/{id}[/linked_user]`, `PUT/DELETE /person/{id}/linked_user_ids[/{user}]` (tenant→user binding), `POST /person/{id}/link-face-biometric`, `GET|POST /person/{id}/docs[/{txn}]` (validated evidence upload), `POST /psa/query/qr` (PhilSys eVerify passthrough).
- **Gaps:** no first-class `review_case` resource; **no `approve/redo/reject` transitions**; no attempt counter (resubmits risk overwriting); no status callback to auth-service (KYC only pulls tenant/secrets/settings from auth-service today); no `self_user_id` binding (but `person.linked_user_id` map + link/unlink endpoints exist — see §7.1); reviewer sees raw evidence without a scoped evidence-view contract.

### 2.4 Auth portal (`svi-authenticationportal-react-kyc`, React 19)
- Routes: `/login/*` (username → tenant-password → password / face / device, forgot-password-email→otp→reset), `/main/*` behind `MainSessionGate` (OIDC `code + PKCE + svi_session` cookie). Headers: `X-Client-ID (localStorage), X-App-ID (env), X-Tenant-ID, X-Realm-ID, Authorization: Bearer (memory-only)`.
- **No `/register` route**; `userEndpoints.register` templates exist in `AuthWebserviceConfig.json` but have no callers. Tenant is picked **after** username (`POST /tenant {username}`), which does not work for a brand-new user with no account yet.

### 2.5 Consolidated gap list
| # | Gap | Impact |
|---|-----|--------|
| G-01 | No public registration endpoint; existing one requires auth | Blocks FR-01 |
| G-02 | No tenant identification before account exists | Blocks tenant-scoped self-service |
| G-03 | OTP scoped to existing users + forgot-password only | Must generalise for pre-account verification |
| G-04 | No `kyc_status` / lifecycle history / attempt versioning in auth DB | Blocks HLR §3, §15, §18 |
| G-05 | OWA single-realm, no tenant context, login-required | Blocks unassisted access |
| G-06 | No review-case state machine in KYC back office | Blocks FR-22…FR-26 |
| G-07 | KYC gating is per-request live lookup, no cached claim | Latency + outage coupling; blocks §7 enforcement |
| G-08 | SMS OTP + CAPTCHA + rate-limit incomplete | Blocks NFR security |
| G-09 | Keycloak user lifecycle (create/disable/delete) not specified for self-service abuse cases | Orphaned IdP accounts risk |
| G-10 | Auth-service KYC client still targets legacy paths; document inspection + OCR provider undecided (no DOT/Innovatrics in current stack) | Self pipeline needs re-pointing (C-02c) + provider decision (Q9) |

---

## 3. Design Principles (from HLR §22, made testable)

1. **Three capabilities, three owners:** Account Management (auth-service owns) → Identity/KYC (KYC-API + OWA own) → Authorization (auth-service RBAC owns). Cross-capability communication only via versioned APIs + events, never direct DB access.
2. **KYC is an attribute, not a role.** No `KYC_VERIFIED_*` role explosion. Enforcement = `account ACTIVE ∧ permission ∧ kyc_verified` (`01 §6.3`).
3. **Contact verification ≠ identity verification.** OTP proves control; KYC proves personhood. Never conflate their timestamps/audit events.
4. **Deny by default, fail closed.** Any KYC lookup failure on a KYC-gated app denies access (existing `getApps` behaviour preserved and extended to tokens).
5. **Immutable history, materialised current state.** `user_lifecycle_history` is append-only; `users.kyc_status` is a cache of the latest record for O(1) gating.
6. **Same KYC module for assisted + self-service.** One OWA codebase, two launch modes (`mode=assisted|self`) with identical verification pipeline; differences isolated to bootstrap/auth/attribution.
7. **Tenant isolation is server-authoritative.** Client-supplied `tenant_id` is always re-resolved server-side from slug/realm/session; Keycloak `client_secret` never reaches the browser.

---

## 4. Target Architecture Overview

### 4.1 System context
```text
[Public User Device] ──HTTPS──▶ [Self-Service Portal (auth-portal extended)]
        │                                │  Step 1: tenant resolve, CAPTCHA, OTP, create account
        │                                ▼
        │                        [auth-service SB (new public APIs)]
        │                          ├─ Tenant resolution (public, throttled)
        │                          ├─ Pending registration + OTP (public, throttled)
        │                          ├─ Keycloak Admin (create user) + Cassandra users
        │                          ├─ Login/token + OIDC (existing, enriched with kyc claim)
        │                          └─ Lifecycle history writer
        │                                │  Step 2: launch OWA self mode
        ▼                                ▼
[Self-Service OWA (same Angular app, public route)] ─▶ [KYC back office (Spring Boot)]
        │  ID capture / doc inspect (TBD) / PhilSys QR / MegaMatcher 1:N via /customers/*
        └─▶ [Review Queue (KYC BO UI + new review-case API)] ─▶ notify (email/SMS)
```

Full C4 + sequences: see `02-architecture-diagrams.md`.

### 4.2 Components (new vs reused vs changed)
| Component | Owner repo | Disposition |
|---|---|---|
| Self-Service Registration UI (`/self-service/:slug/register|verify|password|done`) | `svi-authenticationportal-react-kyc` (extend) | **NEW pages**, reuse `api.ts`, tenant store, modals |
| Public auth-service APIs: `POST /public/tenants/resolve`, `/public/registration/*`, `GET /public/config/{slug}` | `svi-authentication-springboot-kyc` | **NEW controller** (`PublicSelfRegistrationController`), reuse filters pattern (new public rate-limit filter), `TenantService`, `OTPUtils`, `KeycloakUtils`, `AuditTrailUtils` |
| `pending_registrations`, `registration_otps` (or generalised `totp`), `user_lifecycle_history`, `kyc_attempts` (+ `users.kyc_status` etc.) | auth Cassandra | **NEW tables / columns** (`04-data-model.md`) |
| OWA self mode (`/self/:slug`, `mode=self`, applicant self-bootstrap) | `generic-kyc-owa` | **CHANGED**: new public route + `SelfBootstrapService`, reuse capture/biometric/submit pipeline; re-point KYC calls from legacy paths to Spring Boot `/customers/*`, `/psa/query/qr` |
| Review-case API + attempt API (`/review-cases`, `/kyc/attempts`, approve/redo/reject) | `svi-kyc-api-springboot-kyc` | **NEW resources** over existing `applicant`/`facedb_result` stores; adjudication extends existing `PATCH /customers/adjudication` |
| Reviewer UI (queue + case detail + evidence viewer) | KYC BO frontend (existing admin, scope in `05`) | **CHANGED/NEW screens** gated by new `KYC_REGISTRATION_REVIEW` permission |
| Login/KYC decision + `kyc_verified` claim in access + ID tokens | auth-service | **CHANGED**: enrich `POST /token`, `GET /apps`, `GET /user/access-rights`, `POST /oidc/token` |
| Notifications (OTP, redo, approve, reject) | auth-service via existing `EmailSenderUtils` + new SMS provider adapter | **CHANGED**: templates + provider interface |

### 4.3 Deployment / tenancy
- No new deployable service in Phase 1. Auth-service and KYC-API are extended; portals are extended.
- Tenant onboarding for self-service is **opt-in per tenant**: `tenant_settings{self_registration_enabled, self_registration_app_id, self_registration_default_role, otp_* , kyc_required_apps…}` + a `tenant_slug` (see §5). Tenants not whitelisted resolve to "not available" without revealing tenant existence details (anti-enumeration, `06`).
- OWA remains a per-tenant static build **or** moves to a single build with runtime `?slug=` resolution — decision deferred to OWA ticket (both supported by API design; recommended: runtime config to stop rebuild-per-tenant).

---

## 5. Tenant Identification & Keycloak Realm Integration

### 5.1 Why slug-first (not username-first)
The existing portal resolves tenant via `POST /tenant {username}` — impossible for a user with no account. Self-service therefore resolves tenant **from the entry URL**, before any username exists:

```text
https://portal.example.com/s/{tenantSlug}/register
https://onboarding.example.com/self/{tenantSlug}
```

### 5.2 Tenant resolution contract
- New **public, aggressively rate-limited** endpoint: `POST /public/tenants/resolve {slug} → {tenant_id, realm_id, display_name, logo_url, self_registration_enabled, supported_channels, password_policy_summary, captcha_site_key}`.
- Server rules: slug is normalised (lowercase, `[a-z0-9-]{3,64}`); unknown/disabled slug returns **generic** `REGISTRATION_NOT_AVAILABLE` (same shape + timing as success-minus-fields) to avoid tenant enumeration; every call audited (`RESOLVE_TENANT` with slug hash, IP, outcome).
- `tenant_slug` is a new unique column on `tenants` (or `tenant_settings` row `SELF_REGISTRATION_SLUG` if DBA prefers no column change — recommended: real column with unique index; see `04 §2`).
- Downstream calls carry `tenant_id` **but the server re-resolves** `slug → tenant → realm` on every public call and rejects mismatches. Client values are hints, never authority.

### 5.3 Keycloak realm integration
- 1 tenant = 1 Keycloak realm (`tenants.realm_id`), unchanged.
- **User creation (post-OTP only):** auth-service calls Keycloak Admin API `POST /admin/realms/{realm}/users` (existing `KeycloakUtils.createUser` pattern from `InitialSetupUtils`) with `username=email|mobile, email, attributes{tenant_id, kyc_status}`, `enabled=true`, `emailVerified` set from our OTP (not Keycloak's own email flow), credentials = Argon2/bcrypt-hashed password per policy (see `06`; existing reversible `EncryptionUtils` must not be extended to new accounts).
- **Keycloak user ID → Cassandra `users.identity_provider_id`** (existing column), `users_by_username{tenant_id, username → user_id}` row created atomically with `users` row.
- **Abuse handling:** failed-OTP-exhausted or CAPTCHA-failed pending registrations never touch Keycloak (no orphan). Admin deactivate/delete propagates to Keycloak (`disable` first, `delete` only via explicit admin action with audit) — prevents orphaned IdP logins.
- **Login unchanged mechanically** (`POST /token` password grant against the tenant's realm), but response + tokens are enriched with KYC state (see §6).

---

## 6. Step 1 — Self-Service Account Registration (detailed)

### 6.1 Registration options & input
- Identifier: **email OR mobile**, becomes `users.username`. UI radio selector (per HLR §4.1).
- Fields: identifier, password, confirm password, T&C + privacy checkboxes (versioned: store `tnc_version`, `privacy_version` accepted), CAPTCHA token (reCAPTCHA v3 / Turnstile — tenant-configurable).
- Server validation: format (`@Email` / `^\+?[0-9]{7,15}$` — reuse `RegisterUserRequestDTO` validators), password policy (length/complexity/breach list — tenant config), T&C versions current, CAPTCHA score ≥ threshold.

### 6.2 Contact verification BEFORE account creation (HLR §4.2)
```text
1. POST /public/registration/initiate {slug, channel, identifier, password, tnc…, captcha}
   → validate → anti-enumeration checks → create pending_registrations row (TTL e.g. 30 min)
   → create registration_otps row (HMAC-SHA256 hash, 6-char, expiry e.g. 5–10 min tenant-config)
   → send OTP (email via EmailSenderUtils / SMS via new provider adapter; log only message-id)
   → return generic {pending_id, expires_in, resend_cooldown} — NEVER reveal existence
2. POST /public/registration/verify {pending_id, otp_code}
   → constant-time compare → success: Keycloak create + Cassandra users + users_by_username
     + lifecycle history [ACCOUNT_CREATED, ACTIVE_KYC_NOT_VERIFIED]
     + default role assignment (tenant_settings.self_registration_default_role)
     + audit REGISTER_USER(+OTP_VERIFIED) → return {account_created} (no tokens yet)
   → failure: attempts++ → lockout/expiry handling per §6.4
3. User proceeds to normal login (Step 2, §6.4 in HLR numbering / §6 here).
```

- **Pending rows, not user rows:** no `users` / `users_by_username` / Keycloak row exists until OTP success (FR-04). `pending_registrations` is TTL'd (Cassandra default TTL) so abandoned flows self-clean.
- Reuse of `totp` vs new table: recommended **new `registration_otps` table** Enforcing `purpose=REGISTRATION_VERIFY` (keeps forgot-password `totp` semantics untouched; both share `OTPUtils` hashing). Alternative (generalise `totp` with `purpose` column) documented in `04 §3` if DBA prefers fewer tables.
- **Resend:** `POST /public/registration/resend {pending_id}` — cooldown (e.g. 60s), max resends (e.g. 5), each resend invalidates prior code (`is_expended=true`) and issues a new hashed code.

### 6.3 Security requirements (summary; full in `06`)
OTP short expiry + single-use; resend/attempt caps; rate limits by identifier + IP/device; **uniform responses** (no account-existence oracle); passwords hashed with memory-hard function + per-password salt; all verification events audited; CAPTCHA + device/IP signals; email/SMS templates contain no links that bypass OTP (magic-link out of scope Phase 1).

### 6.4 Login & KYC access decision (HLR §6)
- Login request/response unchanged in shape; **semantics enriched**:
  - After Keycloak auth + active/blocked/deleted checks (existing), server reads materialised `users.kyc_status` (+ `users.kyc_attempt_no` for deep-linking) and returns `kyc_status` in `POST /token` response body.
  - Access token (Keycloak) is untouched; **SVI OIDC `id_token` gains `kyc_verified: boolean` + `kyc_status: string`** (see `03 §6`) so downstream apps can enforce without extra lookups.
  - `GET /apps` and `GET /user/access-rights` keep their existing `requiresKYC && !verified → skip` logic, but source of truth becomes the materialised status (with async refresh hook when KYC-API callback arrives), eliminating per-request fan-out latency and outage coupling (stale-cache policy in `03 §6`).
- Decision matrix (UI + API identical):
  - `KYC_VERIFIED` → normal app list.
  - `ACTIVE_KYC_NOT_VERIFIED | KYC_IN_PROGRESS | REDO_REQUIRED` → app list **minus** KYC-gated apps + `kyc_action: {action: START|CONTINUE, onboarding_url}`.
  - `KYC_REVIEW` → read-only status + "under review" (no relaunch).
  - `KYC_REJECTED | SUSPENDED | DEACTIVATED` → blocked with reason code + support path.

---

## 7. Step 2 — KYC Onboarding Module (reuse, self mode)

### 7.1 Same module, two launch modes
| Concern | Assisted (`mode=assisted`, today) | Self (`mode=self`, new) |
|---|---|---|
| Bootstrap auth | Frontliner Keycloak login (`login-required`) | **End-user Bearer** (just-registered account) + `slug`; OWA calls `POST /kyc/self/bootstrap` to create-or-resume applicant bound to `self user_id` |
| Tenant context | Baked-in build config | Runtime `slug → tenant_id` (+ `X-Tenant-ID` header on every KYC call) |
| Applicant binding | None (counsellor-attributed) | Two-level binding: `applicant.self_user_id` (= auth `users.user_id`) for the onboarding session + durable `person.linked_user_id{tenant → user}` map via `PUT /person/{id}/linked_user_ids`; duplicate bootstrap returns existing `IN_PROGRESS` attempt |
| ID/selfie/biometric pipeline | As today (OWA capture; checks via current back-office paths) | **Same stages, Spring Boot paths**: ID select → capture → document inspection (provider TBD, Q9) → OCR confirm (provider TBD) → PhilSys QR (`/psa/query/qr`, PNID) → selfie + liveness session → MegaMatcher 1:N (`/customers/identify/face`, hits via enroll/identify adjudication flow) → additional info → submit |
| Attribution | `AGENT_NAME/AGENT_ID` | `source_channel=self`, `device_fp`, IP; no agent fields |

### 7.2 Verification pipeline (Spring Boot paths)
ID capture (OWA) → document inspection (provider TBD, Q9: tamper/expiry/portrait checks) → OCR extract via TBD provider (retain **three-way provenance**: `ocr_extracted` vs `user_confirmed` vs `externally_verified`) → if PNID → PhilSys eVerify QR passthrough (`POST /psa/query/qr`, result stored as verification **event**, never overwriting user data) → selfie capture + liveness session id forwarded to eVerify → MegaMatcher 1:N (`POST /customers/identify/face`; enroll path returns `SUCCESS`/`DUPLICATE_FOUND`/`ADJUDICATION_WAITING` with `encounter_id`/`duplicate_id`/`hit_score`) → configurable additional-info form → rules-engine decision → `APPROVED → KYC_VERIFIED` | `NEEDS_REVIEW → KYC_REVIEW + review_case` | `auto-reject only on hard-fail policy` (configurable; default is review, never silent auto-reject on biometrics alone).

### 7.3 Attempt versioning (HLR §15)
Every submit = new `kyc_attempts{applicant_id, attempt_no, …status…}` row + immutable evidence references (HFiles refs, PhilSys txn IDs, MegaMatcher `encounter_id`). Prior attempts transition to `SUPERSEDED` (status change only, data untouched). Latest `APPROVED` attempt = current KYC record. Auth lifecycle history links each transition to `kyc_attempt_id`.

---

## 8. Exception, Duplicate & Reviewer Flow (HLR §14, §16–17)

### 8.1 Non-real-time review (preferred model)
Exceptions (unclear ID, OCR conflict, PhilSys mismatch, liveness fail, **potential duplicate 1:N hit**, conflicting attributes) → `review_case{case_id, tenant_id, applicant_id, attempt_no, issue_type, priority, status(PENDING|IN_REVIEW|DECIDED), sla_due}` + notify reviewer pool. **No auto-merge/delete** on duplicate hits — always a case.

### 8.2 Biometric adjudication (back-office only)
A 1:N hit — similar face/fingerprint above threshold, or closely matching identity info across accounts (the OWA `ADJUDICATION_WAITING` outcome) — puts the attempt in `UNDER_REVIEW` and the case in `PENDING` **adjudication**. The user is told only "under review" and stays in `KYC_REVIEW` until a verdict exists; they are never told a duplicate was suspected. Adjudication is performed exclusively by back-office adjudicators holding a dedicated `KYC_BIOMETRIC_ADJUDICATION` permission — general reviewers see match-result-only evidence and cannot clear a biometric hit, and adjudication is not exposed to self-service users, frontliners, or any public API. The adjudicator compares probe vs candidate(s) side-by-side (gated, watermarked, audited evidence view; candidate images via existing `GET /customers/biometrics/face`, hit enrichment as in `AbisAdjudicationEnricher`) and records one verdict:
- **DIFFERENT_PERSON** → hit cleared; the attempt resumes the automated decision path (approves if everything else passes, otherwise normal review).
- **SAME_PERSON** → duplicate confirmed; the case converts to duplicate handling (reviewer rejects, optionally fraud-escalates per tenant policy; identifier/person cool-down ensures retries re-hit rather than slip through).
- **INCONCLUSIVE** → request redo with fresh capture (non-technical user instructions).

Server rule: a `DUP_BIOMETRIC` case cannot be **approved** without a prior `DIFFERENT_PERSON` verdict; the verdict handler must also drive the existing `PATCH /customers/adjudication` (`request_id` + per-hit `UNIQUE` for different-person, `DUPLICATE` for same-person) so the ABIS leaves adjudication-waiting state. Redo/reject remain available. Adjudication carries the shortest SLA (default HIGH/4h) since it blocks onboarding.

### 8.3 Reviewer decisions (exactly three)
1. **Approve** → attempt `APPROVED`, applicant `VERIFIED`, auth lifecycle `KYC_VERIFIED`, notify user, close case.
2. **Request redo** → attempt `REDO_REQUESTED`, applicant stays `REDO_REQUIRED`, auth lifecycle `REDO_REQUIRED` with `{reason_code, user_instructions (non-technical), sla}`, notify via configured channel (email and/or SMS — **never include biometric/conflict details**), user re-enters same OWA self module → new attempt → reprocess. Prior data retained.
3. **Reject** → attempt `REJECTED`, applicant `REJECTED`, auth lifecycle `KYC_REJECTED` with `reason_code`, notify, close case. Re-registration policy is tenant-config (default: identifier blocked for N days to prevent retry loops).

### 8.4 Reviewer UI (HLR §16)
Queue (`case, user(ref, not raw PII beyond need-to-know), issue, priority, status, SLA`) + case detail (account identifier, registration time, KYC status, submitted ID, OCR with provenance, PhilSys result, liveness, biometric **match result/confidence only** — no raw templates, additional info, audit, prior attempts, potential-match side-by-side with controlled fields). Gated by new permission `KYC_REGISTRATION_REVIEW` (auth-service `permissions` row + `PermissionsFilter` mapping; reviewer accounts are provisioned, never self-registered).

---

## 9. RBAC Integration (HLR §19)

- KYC stays an **attribute**: `User{account_status, kyc_status, roles[], permissions[]}`. Apps declare `requires_kyc` (existing `applications.requires_kyc` + `permissions.requires_kyc` columns — reused, no new role naming).
- Enforcement points (all three, so UI bypass is impossible):
  1. Portal UI filters app tiles (UX).
  2. `GET /apps` / `GET /user/access-rights` omit KYC-gated entries when `kyc_status != VERIFIED`.
  3. `PermissionsFilter` + OIDC `kyc_verified` claim deny direct API calls lacking verification.
- No migration of existing roles; default self-registered role (e.g. `customer`/`applicant` per tenant) grants only non-KYC permissions + `KYC_SELF_ONBOARD` (new permission allowing own-applicant bootstrap/submit, nothing else).

---

## 10. Lifecycle, History & BPO Note (HLR §3, §18)

### 10.1 Dual-status model
- **Account status** (existing flags, surfaced as enum for clients): `ACTIVE | SUSPENDED | DEACTIVATED` (derived from `is_active/is_deleted/is_blocked`; `SUSPENDED` = admin-blocked, distinct from auto `blocked` lockout — see `04 §2`).
- **KYC status** (new `users.kyc_status`): `NOT_STARTED | IN_PROGRESS | KYC_REVIEW | REDO_REQUIRED | KYC_VERIFIED | KYC_REJECTED`. Initial value `NOT_STARTED` (surfaced as `ACTIVE_KYC_NOT_VERIFIED` composite in legacy contexts for HLR compatibility).
- Overall `users` row never stores history — only current snapshot + `kyc_attempt_no`, `kyc_updated_at`.

### 10.2 Immutable history
`user_lifecycle_history{tenant_id, user_id, seq, status_or_activity, timestamp, actor(source/user/reviewer/system), kyc_attempt_id?, reason?, case_id?}` — **INSERT-only** (no UPDATE/DELETE in normal processing; corrections are new rows). Supports both transitions and non-transition activities (`ACCOUNT_CREATED, OTP_SENT, OTP_VERIFIED, LOGIN, KYC_STARTED, KYC_SUBMITTED, KYC_REVIEW, REDO_REQUIRED, KYC_VERIFIED, KYC_REJECTED, SUSPENDED…`). Current state = latest row. Designed for millions of rows (partition by `(tenant_id, user_id)`, time-ordered clustering, TTL policy for raw PII per retention schedule — aggregates retained).

### 10.3 BPO applicability
Lifecycle history is the **system-of-record** for the self-service journey and does **not** create a BPO work item per registration (volume: millions). BPO is used only for (a) review cases requiring assignment/SLA among authorised workers, and (b) existing assisted flows. Review-case events are mirrored into lifecycle history (`case_id` link) so either system can reconstruct the journey.

---

## 11. Dependencies, Assumptions, Risks

### Dependencies
| ID | Dependency | Owner | Note |
|----|------------|-------|------|
| DEP-01 | Keycloak Admin API availability + service account per realm | Platform | User create/disable; rate limits to confirm |
| DEP-02 | SMS provider (OTP + notifications) | Vendor | Current SMS is stubbed — contract + SLA needed |
| DEP-03 | Email sender throughput (OTP burst) | Platform | Existing `EmailSenderUtils` load-test |
| DEP-04 | KYC back-office review-case + attempt endpoints (new) + ABIS adjudicate (exists: `PATCH /customers/adjudication`) | KYC BO team | Contract in `03 §5` |
| DEP-05 | OWA self-mode + portal register pages | Frontend teams | `05` tickets |
| DEP-06 | `KYC_REGISTRATION_REVIEW` permission + reviewer provisioning | RBAC admin | |
| DEP-07 | Legacy `svi-authentication-java-kyc` parity check | Backend | Do not assume endpoint migrated until callers verified (per README guardrail) |

### Assumptions
A-01: 1 tenant = 1 Keycloak realm (unchanged). A-02: Email + SMS channels both required; tenants may disable one channel. A-03: PhilSys/MegaMatcher contracts unchanged (merging-branch paths); document-inspection/OCR provider TBD (Q9). A-04: Cassandra TTL + counters available as today. A-05: OIDC `svi_session` cookie flow unchanged; only claims enriched. A-06: English + local-language notification templates are client-supplied. A-07: implement against `origin/development` HEAD of `svi-kyc-api-springboot-kyc` (local skeleton checkout is stale); re-verify endpoint/entity details at build kickoff.

### Risks & mitigations
| Risk | Likelihood/Impact | Mitigation |
|------|-------------------|------------|
| Account enumeration via register/OTP/resolve | H/H | Uniform responses + timing normalisation + CAPTCHA + rate limits + slug-hash audit (`06`) |
| OTP interception / SIM-swap | M/H | Short TTL, single-use, attempt caps, no OTP in logs, optional device binding Phase 2 |
| Duplicate-identity fraud (same person, N accounts) | M/H | 1:N matching → review case, never auto-merge; redo supported; fraud tables retained |
| Keycloak/Cassandra partial failure (orphan users) | M/M | Create order: pending → Keycloak → Cassandra with compensating disable + reconciliation job |
| KYC-API outage blocks login | M/M | Materialised `kyc_status` + stale-cache policy; KYC-gated deny, non-gated allow |
| Review queue overload / SLA breach | M/M | Priority + SLA + pool assignment; auto-escalation; redo-reason analytics |
| PII/biometric over-exposure to reviewers | M/H | Evidence-viewer with match-results-only default; raw images gated + watermarked + audited |
| Reversible password storage extended to new accounts | M/H | **Do not extend `EncryptionUtils`**; new accounts use memory-hard hash (decision required, `06 §2`) |
| Scope creep into biometric algorithm tuning | M/M | Explicitly out of scope; pipeline reused as black box |

---

## 12. Open Questions (for Lead/TL review)
1. Password hashing: confirm Argon2id vs bcrypt and migration path for existing `EncryptionUtils`-encrypted passwords.
2. SMS vendor + sender ID + per-tenant template ownership.
3. `tenant_slug` allocation process (who mints, vanity vs UUID-fallback, rename policy).
4. KYC retry limits: max attempts before forced review/reject (tenant-config default?).
5. Data retention: raw ID/selfie retention period vs attempt metadata; legal hold interaction.
6. OWA build strategy: single runtime-config build vs continued per-tenant builds.
7. BPO mirror depth: which review events must also become BPO work items on day one?
8. PhilSys failure modes: which eVerify error codes auto-redo vs review?
9. Document inspection + OCR provider: no Innovatrics/DOT in the current stack — select the ID authenticity/OCR provider (or reuse OWA-side capability) and define its contract for C-02a before self capture ships.

## 13. Deliverable Map (DoD)
- Architecture/sequence/flows → `02-architecture-diagrams.md`.
- API/database changes → `03-api-design.md`, `04-data-model.md`.
- Dependencies/assumptions/risks → §11–12 above.
- Implementation tickets → `05-user-stories-tickets.md` (titles + stories + requirements + AC, FR-mapped).
