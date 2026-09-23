# CPS-272 — Data Model Design

Cassandra (`auth_system`) is authoritative for accounts + lifecycle; KYC-API stores (MariaDB applicant domain + Cassandra evidence index + GFS blobs + Solr) remain authoritative for verification evidence. **No cross-service direct DB access** — sync only via `POST /internal/kyc-status-callback` (`03 §5.4`).

Conventions: CQL-style definitions, `uuid`/`text`/`timestamp`/`boolean`/`int`/`map<text,text>`. TTLs in seconds. Partitions designed for millions of self-registrations: point lookups by `(tenant_id, user_id)` / `(tenant_id, username)`, never full scans, no `ALLOW FILTERING`.

---

## 1. Change summary

| # | Change | Store | Reason (FR/HLR) |
|---|--------|-------|-----------------|
| 1 | `tenants.slug` unique + `tenant_settings` self-service rows | auth Cassandra | Slug-first resolution (§5 in `01`); per-tenant opt-in |
| 2 | `users.kyc_status`, `users.kyc_attempt_no`, `users.kyc_updated_at`, `users.registration_channel`, `users.tnc_version`, `users.suspended` (+ reason) | auth Cassandra | Materialised gating state; HLR §3/§18 |
| 3 | NEW `pending_registrations` (TTL) | auth Cassandra | Pre-account state; FR-03/FR-04 |
| 4 | NEW `registration_otps` (hash, single-use, TTL) | auth Cassandra | Generalised OTP; HLR §4.2 |
| 5 | NEW `user_lifecycle_history` (append-only) | auth Cassandra | Immutable audit trail; HLR §3 |
| 6 | NEW `kyc_attempts` (auth mirror) | auth Cassandra | Attempt↔lifecycle join without KYC lookup |
| 7 | `audit_trail` event additions | auth Cassandra | New `AuditEvents` codes |
| 8 | `applicant.self_user_id`, `applicant.source_channel`, `review_case` + `kyc_attempt` tables | KYC-API (MariaDB + Cassandra index) | Self binding, review state machine, versioning |
| 9 | `users_by_username` write path for self-created users | auth Cassandra | Existing table, new writer (public flow) |

---

## 2. Auth-service (Cassandra `auth_system`)

### 2.1 `tenants` — add slug
```cql
ALTER TABLE tenants ADD slug text;
-- uniqueness enforced by app-level lookup table (Cassandra has no unique constraint):
CREATE TABLE IF NOT EXISTS tenants_by_slug (
  slug text PRIMARY KEY,
  tenant_id uuid
);
```
- Slug `[a-z0-9-]{3,64}`, immutable after first enable (rename = new slug + redirect row, old slug tombstoned after grace).
- Bootstrapped for whitelisted tenants only; others have no row → `REGISTRATION_NOT_AVAILABLE`.

### 2.2 `users` — add materialised KYC + registration columns
```cql
ALTER TABLE users ADD kyc_status text;            -- NOT_STARTED|IN_PROGRESS|KYC_REVIEW|REDO_REQUIRED|KYC_VERIFIED|KYC_REJECTED
ALTER TABLE users ADD kyc_attempt_no int;
ALTER TABLE users ADD kyc_updated_at timestamp;
ALTER TABLE users ADD registration_channel text;  -- email|sms
ALTER TABLE users ADD tnc_version text;
ALTER TABLE users ADD privacy_version text;
ALTER TABLE users ADD suspended boolean;          -- admin SUSPENDED, distinct from blocked(lockout)
ALTER TABLE users ADD suspend_reason text;
```
- Defaults for existing rows: `kyc_status='NOT_STARTED'` (backfill job, batched, throttled), `suspended=false`.
- `kyc_status` is **cache only**; history in §2.5 is truth. Writes to `kyc_status` occur only inside the lifecycle-append transaction handler.
- Account-state derivation for clients: `DEACTIVATED=is_deleted · SUSPENDED=suspended · LOCKED=blocked · ACTIVE otherwise`; KYC composite `ACTIVE_KYC_NOT_VERIFIED = ACTIVE ∧ kyc!=VERIFIED` kept for HLR wording compat.

### 2.3 NEW `pending_registrations` (pre-account, self-cleaning)
```cql
CREATE TABLE IF NOT EXISTS pending_registrations (
  pending_id uuid PRIMARY KEY,
  tenant_id uuid,
  channel text,                 -- email|sms
  identifier_hash text,         -- HMAC-SHA256(identifier normalised) for rate-limit joins; raw identifier encrypted
  identifier_enc text,          -- envelope-encrypted identifier (decrypt only at verify/send time)
  password_hash text,           -- memory-hard hash (Argon2id/bcrypt — decision in 06 §2), never reversible
  tnc_version text,
  privacy_version text,
  captcha_score double,
  device_fp text,
  ip_address text,
  resend_count int,
  verify_attempts int,
  status text,                  -- PENDING|VERIFIED|EXPIRED|EXHAUSTED (terminal rows linger to TTL for forensics)
  expires_at timestamp,
  created_at timestamp
) WITH default_time_to_live = 1800;  -- 30 min; per-tenant override
```
- Secondary index needs: lookup **only by `pending_id`** (partition key) — no enumeration endpoint. Identifier-hash index is **not** created (prevents oracle scans); duplicate-identifier throttling uses in-memory/Redis counters + `registration_otps` counts, not table scans.
- Raw identifier + password never in logs; `identifier_enc` uses envelope encryption (KMS/data-key per env).

### 2.4 NEW `registration_otps` (single-use codes)
```cql
CREATE TABLE IF NOT EXISTS registration_otps (
  tenant_id uuid,
  pending_id uuid,
  otp_id uuid,
  otp_hash text,                -- HMAC-SHA256(otp_code, secret.key) — same OTPUtils pattern as totp
  channel text,
  attempts int,
  max_attempts int,             -- tenant config, default 5
  resend_of uuid,               -- prior otp_id this replaces (chain for forensics)
  is_expended boolean,
  expires_at timestamp,
  created_at timestamp,
  PRIMARY KEY ((tenant_id, pending_id), created_at, otp_id)
) WITH CLUSTERING ORDER BY (created_at DESC)
  AND default_time_to_live = 3600;
```
- Why not reuse `totp`: `totp` PK is `(tenant_id, user_id, otp_code)` — no `user_id` exists pre-account. Options: (a) this new table (recommended, zero risk to forgot-password), or (b) generalise `totp` with `purpose + pending_id` —留 decision to DBA; both share `OTPUtils` hashing/constant-time compare.
- Verify rule: only the **latest unexpired unexpended** row is valid; older rows auto-`is_expended` on resend.

### 2.5 NEW `user_lifecycle_history` (immutable system-of-record)
```cql
CREATE TABLE IF NOT EXISTS user_lifecycle_history (
  tenant_id uuid,
  user_id uuid,
  seq timeuuid,                 -- clustering: total order per user
  status_or_activity text,      -- ACCOUNT_CREATED|ACTIVE_KYC_NOT_VERIFIED|OTP_SENT|OTP_VERIFIED|KYC_STARTED|KYC_SUBMITTED|KYC_REVIEW|REDO_REQUIRED|KYC_VERIFIED|KYC_REJECTED|SUSPENDED|REACTIVATED|DEACTIVATED|LOGIN|… (closed enum in code)
  occurred_at timestamp,
  actor_type text,              -- system|user|reviewer|admin
  actor_id text,                -- user_id / reviewer id / 'system'
  kyc_attempt_id text,          -- applicant_id#attempt_no where applicable
  review_case_id text,
  reason_code text,
  remarks text,
  ip_address text,
  PRIMARY KEY ((tenant_id, user_id), seq)
) WITH CLUSTERING ORDER BY (seq ASC);
```
- **INSERT-only** in app code (no UPDATE/DELETE paths; revoke via new row). Current state = `SELECT … LIMIT 1` with `CLUSTERING ORDER DESC` (or keep ASC + read last page — pick one and stick to it).
- Minimum columns per HLR §3: user id, status/activity, timestamp, actor, attempt id, reason, case ref — all present.
- Volume: partition per user keeps rows narrow; millions of users = millions of partitions (healthy). Paged reads for support UI (`GET /admin/users/{id}/lifecycle`). Retention: history rows are **never TTL'd** by default; PII inside `remarks` is minimised (codes, not free text); separate retention job handles legal-hold vs purge per policy (decision in `01 §12`).

### 2.6 NEW `kyc_attempts` (auth-side mirror for joins)
```cql
CREATE TABLE IF NOT EXISTS kyc_attempts (
  tenant_id uuid,
  user_id uuid,
  attempt_no int,
  applicant_id text,
  attempt_status text,          -- IN_PROGRESS|SUBMITTED|UNDER_REVIEW|REDO_REQUESTED|APPROVED|REJECTED|SUPERSEDED
  review_case_id text,
  created_at timestamp,
  updated_at timestamp,
  PRIMARY KEY ((tenant_id, user_id), attempt_no)
) WITH CLUSTERING ORDER BY (attempt_no DESC);
```
- Written only from the KYC callback handler; lets lifecycle reads show attempt context without calling KYC-API.

### 2.7 `audit_trail` — new events (existing table, new `AuditEvents` codes)
`RESOLVE_TENANT, REGISTRATION_INITIATED, OTP_SENT_REGISTRATION, OTP_VERIFIED_REGISTRATION, OTP_FAILED, REGISTER_USER_SELF, KYC_STATUS_CALLBACK, KYC_REDO_NOTIFIED, SUSPEND_USER, REACTIVATE_USER, SELF_BOOTSTRAP`. `old_data/new_data` carry **codes + hashes, never OTP/password/raw biometrics**.

### 2.8 `tenant_settings` — new rows (existing table, no DDL)
```
SELF_REGISTRATION_ENABLED=true|false
SELF_REGISTRATION_SLUG=<slug>            (if slug kept in settings instead of tenants.slug)
SELF_REGISTRATION_APP_ID=<uuid>
SELF_REGISTRATION_DEFAULT_ROLE=customer|applicant
OTP_EXPIRY_SEC=600 · OTP_MAX_ATTEMPTS=5 · OTP_RESEND_COOLDOWN_SEC=60 · OTP_MAX_RESENDS=5
REGISTRATION_PENDING_TTL_SEC=1800
PASSWORD_MIN_LENGTH=12 · PASSWORD_POLICY=…
KYC_REQUIRED_APPS=<csv app_ids> (informational; enforcement stays in applications.requires_kyc)
NOTIFY_CHANNELS=email,sms · TEMPLATE_VERSION=…
KYC_MAX_ATTEMPTS_BEFORE_REVIEW=3 · IDENTIFIER_COOLDOWN_AFTER_REJECT_DAYS=30
```

---

## 3. Alternatives considered

| Decision | Recommended | Alternative | Why recommended wins |
|---|---|---|---|
| OTP store | New `registration_otps` | Add `purpose` to `totp` | Zero regression risk to forgot-password hot path; cleaner TTL/partition |
| History ordering | `seq timeuuid` clustering | Counter column | Counters are slow + non-idempotent under retry; timeuuid is idempotent + ordered |
| `kyc_status` on `users` | Materialised cache column | Always join history | Gating is per-login/per-apps-call; history scan per request is too hot |
| Slug store | `tenants.slug` + `tenants_by_slug` | Settings row only | Real column enables future DB-level constraints + admin UX; settings fallback kept |

---

## 4. KYC-API stores

### 4.1 `applicant` — add self binding (MariaDB + Cassandra mirror)
```sql
ALTER TABLE applicant
  ADD COLUMN self_user_id VARCHAR(64) NULL,      -- auth users.user_id (uuid as text)
  ADD COLUMN source_channel VARCHAR(16) NOT NULL DEFAULT 'assisted',  -- assisted|self
  ADD COLUMN current_attempt_no INT NOT NULL DEFAULT 1,
  ADD INDEX idx_applicant_self (self_user_id),
  ADD INDEX idx_applicant_tenant_status (tenant_id, application_status);
```

### 4.2 NEW `kyc_attempt` (MariaDB authoritative; Cassandra index for reads)
```sql
CREATE TABLE kyc_attempt (
  applicant_id VARCHAR(64) NOT NULL,
  attempt_no INT NOT NULL,
  tenant_id VARCHAR(64) NOT NULL,
  status VARCHAR(24) NOT NULL,       -- IN_PROGRESS|SUBMITTED|UNDER_REVIEW|REDO_REQUESTED|APPROVED|REJECTED|SUPERSEDED
  id_type VARCHAR(32), id_no_hash VARCHAR(128),
  ocr_provenance JSON,               -- {ocr:{…}, user_confirmed:{…}, externally_verified:{…}}
  dot_session_id VARCHAR(128), philsys_txn_id VARCHAR(128),
  liveness_result VARCHAR(32), biometric_encounter_id VARCHAR(128), biometric_hit JSON,
  evidence_gfs_refs JSON,            -- HFiles/GFS pointers, never raw blobs in DB
  submitted_at DATETIME, decided_at DATETIME, decided_by VARCHAR(64),
  PRIMARY KEY (applicant_id, attempt_no)
);
```

### 4.3 NEW `review_case` (MariaDB)
```sql
CREATE TABLE review_case (
  case_id VARCHAR(64) PRIMARY KEY,
  tenant_id VARCHAR(64) NOT NULL,
  applicant_id VARCHAR(64) NOT NULL,
  attempt_no INT NOT NULL,
  issue_type VARCHAR(32) NOT NULL,   -- UNCLEAR_ID|OCR_CONFLICT|PHILSYS_MISMATCH|LIVENESS_FAIL|DUP_BIOMETRIC|CONFLICT_ATTRS|OTHER
  priority VARCHAR(16) NOT NULL DEFAULT 'MEDIUM',  -- HIGH|MEDIUM|LOW (rules-derived)
  status VARCHAR(16) NOT NULL DEFAULT 'PENDING',   -- PENDING|IN_REVIEW|DECIDED
  assignee VARCHAR(64) NULL,
  sla_due DATETIME NOT NULL,
  decision VARCHAR(16) NULL,         -- APPROVED|REDO|REJECTED
  adjudication_verdict VARCHAR(16) NULL,  -- SAME_PERSON|DIFFERENT_PERSON|INCONCLUSIVE (DUP_BIOMETRIC only, back-office)
  adjudicated_by VARCHAR(64) NULL,
  adjudicated_at DATETIME NULL,
  reason_code VARCHAR(32) NULL, remarks TEXT NULL,
  created_at DATETIME, updated_at DATETIME,
  INDEX idx_case_queue (tenant_id, status, priority, sla_due)
);
```

### 4.4 Evidence & retention
Raw ID/selfie/video stay in GFS/HFiles (existing); DB holds **references + hashes + results**. Reviewer evidence API redacts raw biometric templates by default. Retention policy (raw vs metadata) is a legal decision (`01 §12`) — schema supports per-attempt purge (delete blobs, keep `kyc_attempt` metadata + hashes).

---

## 5. Migration & backfill plan (design)

1. Additive-only DDL first (new tables/columns nullable) — zero downtime, rollback = ignore new paths.
2. Backfill `users.kyc_status='NOT_STARTED'` in batches (token-range scans, throttled); verify with `SELECT COUNT(*) WHERE kyc_status=null` → 0.
3. Seed `tenants_by_slug` + `tenant_settings` rows for pilot tenants only.
4. Seed `permissions` rows (`KYC_SELF_ONBOARD`, `KYC_REGISTRATION_REVIEW`) + default-role grants.
5. Dual-read period for gating: prefer materialised status, fall back to live KYC lookup on miss; alert on fallback rate.
6. Cutover: enable `SELF_REGISTRATION_ENABLED` per tenant; monitor callback lag (`kyc_updated_at` vs `event_at`).

## 6. Index & query discipline

- Auth reads are always `(tenant_id, user_id)` or `(tenant_id, username)` or `(slug)` — no cross-partition scans in request paths.
- Review queue reads use `(tenant_id, status, priority, sla_due)` index with pagination (`page_size ≤ 50`).
- No `ALLOW FILTERING` anywhere (matches existing repo discipline). Support/admin exports go through bounded batch jobs, not request paths.
