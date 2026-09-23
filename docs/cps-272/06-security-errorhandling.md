# CPS-272 — Security, Error Handling & Audit Design

Companion to `01 §6.3/§11`. All controls here become acceptance criteria in `05` (esp. tickets D-06/D-07). Nothing in this file requires new crypto primitives — it composes existing `OTPUtils` (HMAC-SHA256 + constant-time compare), `secret.key`, Keycloak, and TLS.

---

## 1. Threat model (public self-service surface)

| # | Threat | Controls |
|---|---|---|
| T-01 | Account enumeration (register/resolve/login/OTP) | Uniform responses + timing normalisation + masked destinations + slug-hash audit (§4) |
| T-02 | OTP brute force / replay | 6-char entropy + short TTL + `max_attempts` + single-use `is_expended` + resend invalidation + per-pending/IP buckets (§3) |
| T-03 | OTP interception (email/SMS) | Short TTL, no OTP in logs/URLs, TLS-only, optional Phase-2 device binding |
| T-04 | Credential stuffing / password spray | Rate limits, CAPTCHA escalation, Keycloak lockout preserved (`blocked` + `max_login_attempts`), breach-list check |
| T-05 | Bot mass-registration | CAPTCHA on `initiate` (+ escalation), device/IP signals, identifier cool-down after reject |
| T-06 | Tenant spoofing / cross-tenant creation | Server re-resolves slug→tenant→realm on every call; client `tenant_id` never trusted |
| T-07 | UI bypass of KYC gating | Triple enforcement: portal filter + `/apps`/`access-rights` omission + token claim deny (`kyc_verified`) |
| T-08 | Reviewer PII/biometric over-collection | Match-results-only default; raw evidence gated, watermarked, audited; `KYC_REGISTRATION_REVIEW` least-privilege |
| T-09 | Orphaned Keycloak users on partial failure | Create order + compensating disable + reconciliation job; no Keycloak write before OTP success |
| T-10 | Notification injection / phishing | Templates versioned, no executable links bypassing OTP, per-tenant sender IDs, no sensitive details in redo notices |

Out of scope for Phase 1 (reserve hooks): WebAuthn/passkeys, SIM-swap signals, device attestation, magic links.

---

## 2. Credential & secret handling

1. **Passwords (new accounts): memory-hard hash + per-password salt** (Argon2id preferred, bcrypt acceptable) — decision required (`01 §12 Q1`). Rationale: existing `EncryptionUtils` (reversible encryption for legacy face-login decrypt-then-grant) must **not** be extended; reversible storage of self-service passwords would violate HLR §21 and create a decryption-oracle target.
2. **OTP at rest:** `HMAC-SHA256(otp_code, secret.key)` via existing `OTPUtils.hashOtp`; compare with `MessageDigest.isEqual` (already implemented). Raw codes exist only in transit (TLS) and in the sender payload.
3. **`secret.key`:** existing file-based key (`init.secret-key-path`) retained; rotation procedure must re-hash nothing (HMAC verification is point-in-time — old OTPs simply expire; document rotation window).
4. **Keycloak `client_secret`:** server-side only (CPS-164). Public endpoints never accept or return it. Tenant `client_id` (public) is the only credential-adjacent value exposed via `resolve`.
5. **Envelope encryption** for `pending_registrations.identifier_enc` (KMS/data-key per environment); decrypt only inside `initiate` (send) and `verify` (create) handlers.
6. **Tokens:** access/refresh handling unchanged; OIDC `svi_session` cookie `HttpOnly+Secure+SameSite`; new `kyc_*` claims are informational (enforcement also server-side, never claim-only).

---

## 3. Rate limiting & CAPTCHA design

New `PublicRateLimitFilter` (before any business logic on `/public/**`):

| Bucket | Key | Suggested default (tenant-overridable) |
|---|---|---|
| `resolve` | IP | 60/min/IP |
| `initiate` | identifier_hash + IP | 5/hr/identifier, 20/hr/IP |
| `verify` | pending_id + IP | 10/pending lifetime, 30/min/IP |
| `resend` | pending_id | cooldown 60s, max 5 lifetime |
| `bootstrap/submit` (KYC self) | user + IP | 20/hr/user (submit 10/day/user) |
| login (existing, tighten) | username + IP | keep `max_login_attempts` + add IP bucket |

- Exceeding returns `429 {code: RATE_LIMITED, retry_after_sec}` (no state leakage).
- CAPTCHA: required on `initiate`; re-challenge on 3rd `verify` failure and on any `429`. Verify server-side (score ≥ 0.5 for v3 or Turnstile success); store `captcha_score` on pending row for fraud analytics.
- Device fingerprint (`X-Device-FP`: canvas-agnostic hash of UA+screen+tz+storage) and `X-Forwarded-For`/IP are **signals**, never sole identifiers.

---

## 4. Anti-enumeration rules (normative)

1. `resolve`: unknown slug ≡ disabled slug (same JSON shape, `enabled:false`, `code: REGISTRATION_NOT_AVAILABLE`, normalised delay e.g. 150–250 ms jitter).
2. `initiate`: new identifier ≡ taken identifier (same `202 + pending_id`-shaped response; taken path creates a **decoy pending** that sends no code but behaves identically through verify-fail — or returns real pending with generic outcome; pick one per security review, document choice).
3. `verify`: wrong-code vs expired vs exhausted are distinguishable **only** to the holder of `pending_id` (un-guessable UUID) — safe. Never reveal whether identifier exists.
4. `forgotPasswordEmailHint`-style masking reused for OTP destinations (`j•••@g•••.com`, `09******1234`).
5. Login keeps existing `INVALID_USER_OR_PASS` generic (no change).

---

## 5. Error catalogue (public + reviewer + callbacks)

Shape: `{status: "failed", code, message (user-safe, localisable by key), retry_after_sec?, attempts_remaining?}`. `message` never contains internals; `X-Request-ID` correlates to server logs.

| HTTP | Code | When | Notes |
|---|---|---|---|
| 200 | `REGISTRATION_NOT_AVAILABLE` | slug unknown/disabled | Generic, timing-normalised |
| 202 | — | initiate/resend accepted | Uniform |
| 201 | — | verify → created | No tokens |
| 400 | `INVALID_IDENTIFIER / WEAK_PASSWORD / INVALID_CHANNEL / CAPTCHA_FAILED / TERMS_NOT_ACCEPTED` | validation | Field-level, safe |
| 400 | `INVALID_OTP` | wrong code, attempts left | Include `attempts_remaining` |
| 403 | `KYC_REQUIRED` | direct call to KYC-gated API without verification | Reviewer/user-safe |
| 403 | `REVIEWER_ONLY` | missing `KYC_REGISTRATION_REVIEW` | |
| 404 | `PENDING_NOT_FOUND` | bad `pending_id` | Same as expired to holder? Keep distinct (UUID unguessable) |
| 409 | `ATTEMPT_ALREADY_OPEN` | concurrent bootstrap | Return existing attempt |
| 410 | `OTP_EXPIRED / PENDING_EXPIRED` | TTL passed | Offer resend/restart |
| 423 | `ACCOUNT_SUSPENDED / ACCOUNT_DEACTIVATED / KYC_REJECTED` | terminal states | With support path, no internals |
| 429 | `RATE_LIMITED / RESEND_COOLDOWN / OTP_ATTEMPTS_EXCEEDED / RESEND_LIMIT_EXCEEDED` | throttles | `retry_after_sec` |
| 503 | `OTP_DELIVERY_FAILED / IDP_UNAVAILABLE / KYC_UNAVAILABLE` | downstream outage | Safe retry; OTP not consumed on delivery failure |

KYC callback failures: auth-service returns `2xx` only after durable lifecycle append; else `5xx` for retry with same `idempotency_key`.

---

## 6. PII & biometric minimisation

- **Log allowlist:** tenant/user/case IDs, reason **codes**, message-ids, scores-outcomes (pass/fail), latencies. **Never:** OTP, password, raw identifier, ID images, selfies, biometric templates, PhilSys payload fields.
- **Reviewer evidence:** default bundle excludes raw templates and full ID images beyond need-to-know crops; full blobs require explicit "reveal" action (audited, watermarked, time-boxed). Biometric probe-vs-candidate comparison is further restricted to `KYC_BIOMETRIC_ADJUDICATION` holders via the adjudication endpoint (same gating + audit).
- **Provenance retention:** `ocr_extracted` vs `user_confirmed` vs `externally_verified` kept separately (HLR §9) so corrections never destroy evidence.
- **Retention:** raw blobs per legal schedule (decision `01 §12 Q5`); attempt metadata + hashes retained for audit; purge jobs are explicit tickets (not silent TTL except `pending_*`).

---

## 7. Audit completeness (maps to `05 D-07`)

Every row carries `tenant_id, actor(source/user/reviewer/system), ip, request_id, timestamp`. Required events (all via existing `audit_trail` + `user_lifecycle_history` where user-scoped):

`RESOLVE_TENANT, REGISTRATION_INITIATED, OTP_SENT_REGISTRATION, OTP_VERIFIED_REGISTRATION, OTP_FAILED, REGISTER_USER_SELF, LOGIN (+method), KYC_STARTED, KYC_SUBMITTED, OCR_RESULT, EXTERNAL_VERIFY (PhilSys), LIVENESS_RESULT, BIOMETRIC_MATCH, KYC_DECISION, REVIEW_CASE_CREATED, ADJUDICATION_VERDICT, REVIEWER_DECISION, KYC_REDO_NOTIFIED, USER_NOTIFIED (approve/reject), SUSPEND/REACTIVATE/DEACTIVATE, CALLBACK_APPLIED`.

Support query: `GET /admin/users/{id}/lifecycle` must reproduce the HLR §3 nine-row golden history in E2E.

---

## 8. Operational security

- Public endpoints behind WAF + bot defence; TLS 1.2+ only; HSTS; strict CORS allowlist per slug (no `*` with credentials).
- Session/token handling unchanged; add alerting on: OTP verify-fail spikes, resolve-fail spikes, callback lag > SLO, review SLA breach, Keycloak create-fail rate.
- Secrets (`secret.key`, Keycloak service accounts, SMS keys) in vault/mounted files only — never in docs, logs, or frontend bundles (repo guardrail already states this).
