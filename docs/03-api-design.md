# CPS-272 — API / Interface Design

Design-level contracts (field names follow the repo's `SNAKE_CASE` Jackson strategy). No implementation in this ticket. Error catalogue + auth details in `06-security-errorhandling.md`.

Conventions: base `/spring/auth-services` (auth-service SB) and KYC-API base as today. `X-Request-ID` echoed on every response. Times are ISO-8601 UTC. All public endpoints return **uniform shapes** for success-vs-privacy-sensitive failures.

---

## 1. Header & tenant contract (all services)

| Header | Required | Notes |
|---|---|---|
| `X-App-ID` | Portal→auth on authenticated calls (existing `AppIDFilter`) | Self-registration default app comes from `tenant_settings.self_registration_app_id`; client sends it, server re-validates association on authed calls only |
| `X-Client-ID` | Authenticated calls (existing `AuthenticatedFilter`) | Public calls do **not** send secret; server injects `client_secret` for Keycloak/KYC calls (CPS-164 preserved) |
| `X-Tenant-ID` | Authenticated calls | Public calls send `slug`; server resolves + ignores mismatched `tenant_id` hints |
| `X-Realm-ID` | Login (`POST /token` uses `realm_id` in body — unchanged) | Public login resolves realm from slug first, then sends existing body shape |
| `Authorization: Bearer` | Authenticated calls (memory-only in portal, `svi_session` cookie for OIDC) | OWA self mode uses end-user Bearer (not frontliner) |
| `X-Request-ID`, `X-Device-FP`, `X-Forwarded-For` (infra) | Recommended on public calls | Rate-limit + audit signals |

---

## 2. auth-service — NEW public controller `PublicSelfRegistrationController`

> New file (design): `controllers/PublicSelfRegistrationController.java` + `services/SelfRegistrationService.java`. Public endpoints use a **new `PublicRateLimitFilter`** (IP + identifier buckets) instead of `@Authorized/@Authenticated/@Permissions`. `@EnabledCORS` with strict allowlist per slug. `@AuditLogger` on all.

### 2.1 `POST /public/tenants/resolve` — tenant discovery (public, throttled)
Resolves entry slug before any account exists. Replaces username-first `POST /tenant` for new users.

Request:
```json
{ "slug": "navotas-demo", "app_id": "9def2757-…" }
```
Success (enabled tenant) `200`:
```json
{
  "status": "success",
  "data": {
    "tenant_id": "uuid", "realm_id": "NAVOTAS_DEMO",
    "display_name": "Navotas Demo", "logo_url": "https://…/logo.png",
    "self_registration_enabled": true,
    "supported_channels": ["email", "sms"],
    "password_policy": {"min_length": 12, "require_upper": true, "require_digit": true, "require_symbol": true},
    "captcha_site_key": "…", "tnc_version": "2026-08-01", "privacy_version": "2026-08-01",
    "onboarding_url_template": "https://onboarding…/self/{slug}"
  }
}
```
Disabled/unknown slug → `200` with `self_registration_enabled: false` + generic `code: REGISTRATION_NOT_AVAILABLE` (same timing; see `06 §4`). Never 404-distinguishable.

### 2.2 `GET /public/config/{slug}` — portal bootstrap (public, cached)
Static-ish per-tenant UI config (allowed ID types, required additional-info fields, notification channels, review SLA text). Backed by `tenant_settings` + KYC `ui-config` passthrough. `Cache-Control: public, max-age=300`.

### 2.3 `POST /public/registration/initiate` (public, strict rate limit)
```json
{
  "slug": "navotas-demo",
  "channel": "email|sms",
  "identifier": "user@example.com | +639…",
  "password": "…plaintext over TLS…",
  "tnc_accepted": true, "tnc_version": "2026-08-01",
  "privacy_accepted": true, "privacy_version": "2026-08-01",
  "captcha_token": "…"
}
```
Server: validate → CAPTCHA verify → normalise identifier → existence check (silent) → create `pending_registrations` (TTL 30 min) + `registration_otps` (hashed, 5–10 min) → dispatch OTP → audit `REGISTRATION_INITIATED`.

Response (uniform whether identifier is new or taken) `202`:
```json
{ "status": "success", "data": { "pending_id": "uuid", "channel": "email", "masked_to": "u•••@e•••.com", "expires_in_sec": 600, "resend_cooldown_sec": 60 } }
```

### 2.4 `POST /public/registration/verify` (public, strict rate limit)
```json
{ "pending_id": "uuid", "otp_code": "A1B2C3" }
```
Success `201` (account created; **no tokens** — user must log in):
```json
{
  "status": "success",
  "data": { "account_created": true, "tenant_id": "uuid", "username": "user@example.com", "kyc_status": "NOT_STARTED", "next": "/self-service/navotas-demo/login" }
}
```
Failure: `400 INVALID_OTP` (attempts left), `410 OTP_EXPIRED`, `429 OTP_ATTEMPTS_EXCEEDED` / `RESEND_LIMIT` — all with `attempts_remaining` where safe; existence never revealed.

Side effects on success: Keycloak create → `users` + `users_by_username` + default role (`user_roles`) → lifecycle `ACCOUNT_CREATED, ACTIVE_KYC_NOT_VERIFIED` → audit `REGISTER_USER`. On total failure: compensating Keycloak disable if Cassandra write failed (reconciliation job covers leftovers).

### 2.5 `POST /public/registration/resend` + `GET /public/registration/status`
- `resend {pending_id}` → invalidates prior code, issues new, enforces cooldown + max resends → `202` uniform.
- `status?pending_id=` → `{exists, expires_in_sec, attempts_remaining, resend_available_in_sec}` — no PII.

### 2.6 Throttling & CAPTCHA summary
Per-IP + per-identifier token buckets (e.g. initiate 5/hr/identifier, 20/hr/IP; verify 10/p pending; resend cooldown 60s/max 5). CAPTCHA required on `initiate` (score ≥ 0.5 v3 or Turnstile pass) and on 3rd verify failure. Details in `06 §3`.

---

## 3. auth-service — CHANGED existing endpoints

### 3.1 `POST /token` (login) — enrich, don't reshape
Request unchanged (`username, password, realm_id, device_id?`). Response **adds** (backwards-compatible):
```json
{
  "status": "success",
  "access_token": "…", "session_state": "…", "customer_id": "…",
  "kyc_status": "NOT_STARTED|IN_PROGRESS|KYC_REVIEW|REDO_REQUIRED|KYC_VERIFIED|KYC_REJECTED",
  "kyc_attempt_no": 2,
  "kyc_action": { "action": "START|CONTINUE|NONE|BLOCKED", "onboarding_url": "https://…/self/{slug}?attempt=3", "reason_code": "REDO_UNCLEAR_ID" }
}
```
Source: materialised `users.kyc_status` (no live KYC fan-out on the hot path; async refresh via §5 callbacks).

### 3.2 `GET /apps`, `GET /user/access-rights` — attribute enforcement
Logic unchanged (`requiresKYC && kyc!=VERIFIED → omit`), source changes to materialised status + `Cache-Control: private`. Response adds per-app `kyc_required: true|false` so UI can explain, not just hide. `PermissionsFilter` mapping gains `KYC_SELF_ONBOARD` (own-applicant only) and `KYC_REGISTRATION_REVIEW` (reviewers only).

### 3.3 `POST /oidc/token` (`id_token`) — new claims
```json
{ "…existing…": "…", "kyc_verified": true, "kyc_status": "KYC_VERIFIED", "tenant_id": "uuid", "assurance": "otp+kyc" }
```
Downstream KYC-gated apps deny when `kyc_verified != true` even with valid signature. Key rotation/TTL unchanged.

### 3.4 Admin additions (existing `AdminController` style)
- `POST /admin/user/{id}/suspend|unsuspend` (distinct from lockout `blocked` and soft-delete).
- `GET /admin/users/{id}/lifecycle` (paged history for support/audit).
- `PUT /admin/tenants/{id}/self-registration` (enable/disable + slug + defaults + channel toggles).
- Permission rows: `KYC_SELF_ONBOARD`, `KYC_REGISTRATION_REVIEW` (+ `permissions.requires_kyc` reuse).

---

## 4. Portal (React) — NEW routes (no new deployable)

| Route | Purpose | Calls |
|---|---|---|
| `/self-service/:slug/register` | Step 1 form (channel radio, identifier, password, T&C, CAPTCHA) | `resolve` → `initiate` |
| `/self-service/:slug/verify?pending=` | OTP entry, resend, expiry handling | `verify`, `resend`, `status` |
| `/self-service/:slug/done` | Account-created → login CTA | — |
| `/self-service/:slug/login` | Existing password/face/device pages tenant-pre-resolved (skip username→tenant step) | `POST /token` (existing) |
| `/self-service/:slug/kyc-status` | Status polling: review-wait, redo-CTA, reject explanation | `GET /user/access-rights`, KYC `GET /status` |

Reuse `api.ts`, `auth-fetch.ts` headers, modal/error patterns. Unknown/disabled slug renders generic "registration not available" (no tenant oracle).

---

## 5. KYC back office (`svi-kyc-api-springboot-kyc`) — NEW + CHANGED resources

> New first-class resources over the existing Cassandra stores (`customer_kyc.applicant`, `facedb_result`, `person*`). Base path `/spring/gen-kyc-api`. `tenant_id` from `X-Tenant-ID` header (fallback JWT `tenant_id` claim — existing `JWTUtils.getTenantIdFromHeader` pattern); existing auth chain (`Authorized → Authenticated → AppID → Permissions → AuditLogger`) applies to new endpoints. Reuse as-is: `POST /customers/verify/face` (1:1), `POST /customers/identify/face` (1:N or by person_id), `POST /customers/enroll/face` (`SUCCESS`/`DUPLICATE_FOUND`/`ADJUDICATION_WAITING`), `PATCH /customers/update/face`, `GET /customers/biometrics/face` (evidence image), `POST /customers/save/transaction` (person registry), `GET /person`, `POST /psa/query/qr` (PhilSys eVerify passthrough).

### 5.1 `POST /kyc/self/bootstrap` (authenticated end-user Bearer)
Creates-or-resumes the caller's applicant + open attempt. Binds `applicant.self_user_id = auth users.user_id` (new column, `04 §4`) for the session, and writes the durable identity link via existing `PUT /person/{person_id}/linked_user_ids {user_id}` (tenant→user map; check with `HEAD .../linked_user`).
```json
// request
{ "slug": "…", "consent_version": "2026-08-01" }
// response 200
{ "applicant_id": "…", "attempt_no": 3, "attempt_status": "IN_PROGRESS", "resume_from": "SELFIE" }
```
Duplicate bootstrap with open attempt returns the same attempt (idempotent). One open attempt per applicant enforced.

### 5.2 `POST /kyc/self/attempts/{n}/submit` (authenticated)
Runs the pipeline (document inspection via TBD provider, PhilSys QR, MegaMatcher face via `/customers/identify|enroll/face`, rules) then transitions attempt + applicant and **pushes a status callback** to auth-service (§5.4). Response: `{applicant_id, attempt_no, decision: APPROVED|UNDER_REVIEW, review_case_id?}`.

### 5.3 Review cases (reviewer Bearer + `KYC_REGISTRATION_REVIEW`)
- `GET /review-cases?tenant_id=&status=PENDING&issue=&priority=&page=` → paged queue (minimal PII).
- `GET /review-cases/{case_id}` → scoped evidence bundle: account ref, registration time, KYC status, ID + OCR with **provenance** (`ocr/user/verified`), PhilSys event, liveness result, biometric **match result/confidence only**, additional info, prior attempts, potential-match controlled fields. Raw templates/images gated + watermarked + audited.
- `POST /review-cases/{id}/approve {notes}` → attempt `APPROVED`, applicant `VERIFIED`.
- `POST /review-cases/{id}/request-redo {reason_code, instructions}` → attempt `REDO_REQUESTED`, applicant `REDO_REQUIRED`. `reason_code` from tenant-config codelist (`UNCLEAR_ID, OCR_INCONCLUSIVE, POOR_SELFIE, LIVENESS_ISSUE, DUP_BIOMETRIC, CONFLICT_ATTRS, OTHER`); `instructions` are user-facing, non-technical.
- `POST /review-cases/{id}/reject {reason_code, notes}` → attempt `REJECTED`, applicant `REJECTED` + identifier cool-down.
- `GET /review-cases/{id}/adjudication` (adjudicator Bearer + `KYC_BIOMETRIC_ADJUDICATION`, back-office only) → candidate list for a `DUP_BIOMETRIC` case: probe vs candidate refs, hit scores, controlled identity fields; side-by-side images via existing `GET /customers/biometrics/face`, gated + watermarked + audited.
- `POST /review-cases/{id}/adjudicate {verdict: SAME_PERSON|DIFFERENT_PERSON|INCONCLUSIVE, confidence, notes}` → records verdict + audit: `DIFFERENT_PERSON` clears the hit (attempt resumes automated path); `SAME_PERSON` converts to duplicate handling (→ reject/fraud path + cool-down); `INCONCLUSIVE` → redo with fresh capture. The handler must also drive the existing `PATCH /customers/adjudication` (`request_id` + per-hit `hit_subject_id → UNIQUE|DUPLICATE`): `DIFFERENT_PERSON` sends `UNIQUE`, `SAME_PERSON` sends `DUPLICATE`. Server rule: a `DUP_BIOMETRIC` case cannot transition to approve without a prior `DIFFERENT_PERSON` verdict.
- All four **emit the auth callback** (§5.4) and write KYC-side audit.

### 5.4 `POST /internal/kyc-status-callback` (auth-service, mTLS/service token)
KYC back office → auth-service authoritative transition (never trust client-reported status). No callback infrastructure exists today (the back office only pulls tenant/secrets/settings from auth-service), so sender retries, receiver idempotency, and service credentials are all new in this ticket:
```json
{
  "tenant_id": "uuid", "applicant_id": "…", "self_user_id": "uuid",
  "attempt_no": 3, "kyc_status": "KYC_VERIFIED|KYC_REVIEW|REDO_REQUIRED|KYC_REJECTED|IN_PROGRESS",
  "review_case_id": "…?", "reason_code": "…?", "event_at": "2026-…Z", "idempotency_key": "…"
}
```
Auth-service validates service credential + `tenant/applicant/user` binding, appends lifecycle history, updates materialised `users.kyc_status`, notifies user. Retried with idempotency key; stale events (older `attempt_no`) are recorded but do not regress current state.

### 5.5 NEW: applicant/attempt status reads
The Spring Boot service has no applicant-status endpoint today (no `GET /status` equivalent). Add `GET /kyc/self/status?applicant_id=` returning `{applicant_id, kyc_status, attempt_no, attempt_status, review_case_id, decided_at, reason_code}` sourced from the `review_case`/`kyc_attempt` tables (§5.3, `04 §4`) — additive, owned by the same tickets as the case API.

---

## 6. Caching, staleness & failure semantics

- Login/`/apps`/`access-rights` read materialised `users.kyc_status` (p95 < existing per-request KYC fan-out). Callback applies within seconds; `GET /self-service/:slug/kyc-status` may force-refresh (rate-limited) for users staring at "under review".
- If KYC-API is down: self-bootstrap/submit/review return `503 KYC_UNAVAILABLE` with `retry_after`; login still succeeds; KYC-gated apps deny (fail closed), non-gated allow.
- If Keycloak is down during `verify`: pending stays open, OTP not consumed, `503 IDP_UNAVAILABLE` with safe retry; reconciliation job heals partial creates.

## 7. Versioning & compatibility

- All new endpoints are additive; no existing request/response field removed. `POST /user/register` (admin) unchanged — self-service uses the new public trio only.
- `permissions.ssoFilePath/authFilePath` JSON gains two codes; unknown-code behaviour stays deny.
- OIDC discovery unchanged; only `id_token` claims added (verifiers ignoring unknown claims unaffected).

## 8. Conformance to the current codebase (verified)

Checked against `src/main/java/com/svi/authentication/controllers` + `utils/ResponseUtils.java`:

| Convention today | This design | Verdict |
|---|---|---|
| Base path `/spring/auth-services` (`application.properties`) + resource prefixes (`/admin`, `/user`, `/tenant`, `/groups`, `/config`, `/oidc/*`; auth verbs at root: `/token`, `/login/face`, `/logout`, `/generate-otp`, `/verify-otp`, `login/forgot-password*`) | New endpoints live under the same base; resource nouns for new areas (`/kyc/self/*`, `/review-cases/*` in KYC-API) | Conform |
| Kebab-case multi-word paths (`/login/face`, `/forgot-password`, `/access-rights`, `/required-actions`, `/client-id`) | All new paths kebab-case (`/public/tenants/resolve`, `/public/registration/initiate`, `/review-cases/{id}/request-redo`) | Conform |
| Snake-case path params (`{user_id}`, `{role_id}`, `{group_id}`) | New params snake-case (`{slug}` is single-word; `{case_id}`, `{pending_id}` in query/body) | Conform |
| Envelope `{status: "success"\|"failed", message?, ...}` (`ResponseUtils.create`) | Error shape keeps `status` + `message` keys and adds `code` + safe extras (`retry_after_sec`, `attempts_remaining`) — additive, same serializer | Conform |
| Public endpoints are root paths with **no** auth annotations (public by absence: `/token`, `/generate-otp`, …) | New public endpoints grouped under an explicit `/public/` namespace with a dedicated `PublicRateLimitFilter` instead of relying on annotation absence | **Intentional deviation** (see DEC-11): explicitness lets WAF/filters treat the whole subtree as untrusted, and prevents a future `@Authorized` copy-paste from silently locking (or a missing annotation from silently opening) a public route |
| `POST /user/register` requires `@Permissions + @Authenticated + @Authorized` | Untouched — self-service uses only the new public trio, never this endpoint | Conform |
