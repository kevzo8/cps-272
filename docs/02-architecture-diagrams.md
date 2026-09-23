# CPS-272 — Architecture & Diagrams (Mermaid)

All diagrams are Mermaid — they render in GitLab/GitHub/VS Code. Source of truth for flows in `01-TDD-main.md`.

---

## As-Is vs To-Be — visual comparison (read first)

Same visual grammar in both charts: rounded boxes are entry points, diamonds are decisions,
loop-backs show rework. The table below maps each stage.

**AS-IS — assisted registration (today):**

```mermaid
flowchart TB
    subgraph ASIS[AS-IS — assisted only]
        direction TB
        A1([Person needs account]) --> A2[Frontliner account<br/>provisioned out-of-band<br/>by an admin]
        A2 --> A3[Frontliner logs into OWA<br/>Keycloak login-required<br/>one baked-in realm per build]
        A3 --> A4[Frontliner creates the user<br/>POST /user/register<br/>admin auth required]
        A4 --> A5[Frontliner drives OWA<br/>on a shared device<br/>ID plus selfie plus form]
        A5 --> A6[KYC-API verifies<br/>DOT plus PhilSys plus 1:N match]
        A6 --> A7{Exception<br/>or duplicate?}
        A7 -->|No| A8[User logs into portal<br/>username-first tenant pick<br/>live KYC lookup per request]
        A7 -->|Yes| A9[Ad-hoc fix:<br/>manual status patch<br/>or delete plus re-enroll]
        A9 --> A5
    end
```

**PROPOSED — self-service (this design):**

```mermaid
flowchart TB
    subgraph TOBE[PROPOSED — self-service]
        direction TB
        B1([Person needs account]) --> B2[Opens tenant link<br/>/self-service/slug/register<br/>no account needed]
        B2 --> B3[Proves contact control<br/>OTP before any<br/>account exists]
        B3 --> B4[Account auto-created<br/>Keycloak plus Cassandra<br/>plus history rows]
        B4 --> B5[Logs in, routed<br/>by kyc_status<br/>to OWA self mode]
        B5 --> B6[Self-drives the same<br/>OWA pipeline<br/>attempt is versioned]
        B6 --> B7{Exception<br/>or duplicate?}
        B7 -->|No| B8[KYC_VERIFIED<br/>full app access<br/>claim-enforced]
        B7 -->|Yes| B9[Review case:<br/>approve or redo or reject<br/>non-real-time]
        B9 -->|Redo| B10[Notified, logs in<br/>same module<br/>new attempt]
        B10 --> B6
        B9 -->|Approve| B8
    end
```

**Entry-point contrast (how tenant is known):**

```mermaid
flowchart LR
    subgraph ASIS2[AS-IS entry]
        direction LR
        X1[OWA build per tenant<br/>realm baked into config] --> X2[Frontliner login<br/>required before<br/>anything loads]
    end
```

```mermaid
flowchart LR
    subgraph TOBE2[PROPOSED entry]
        direction LR
        Y1[Any device opens<br/>/self-service/slug/*] --> Y2[Public resolve<br/>slug to tenant plus realm<br/>throttled] --> Y3[Self register<br/>then user login<br/>then OWA self mode]
    end
```

### Stage-by-stage diff

| Stage | As-Is | Proposed | What changes (ticket) |
|---|---|---|---|
| Tenant identification | Baked-in realm per OWA build; portal resolves tenant from username (needs account) | Slug-first public resolve, no account needed | A-01 |
| Account creation | Admin/frontliner calls authed `POST /user/register`; bulk import | Pending record plus OTP, account auto-created only after verify | A-02, A-03, A-04 |
| Contact proof | None for new users (trusts frontliner) | OTP single-use + TTL before creation | A-02, A-03 |
| OWA access | Frontliner Keycloak login, shared device | End-user Bearer, own device, runtime tenant | C-01 |
| Applicant binding | None (counsellor-attributed) | `applicant.self_user_id` bound, idempotent resume | C-01, C-03 |
| Verification pipeline | DOT + PhilSys + 1:N, assisted capture | Identical pipeline, self capture | C-02 (reuse) |
| Exception handling | Ad-hoc status patch or delete + re-enroll | `review_case` + back-office adjudication (PENDING on hits) + approve/redo/reject, attempts immutable | D-01, D-02, D-02b, D-03 |
| Redo | Manual rework by staff | Notify user, same module, new attempt | D-04 |
| KYC gating | Portal hides tiles; live KYC lookup per request | Attribute + `kyc_verified` claim, materialised status, triple enforcement | B-01, B-02 |
| Audit trail | Request logs + `audit_trail` only | Plus append-only `user_lifecycle_history` with attempt linkage | D-07 |

---

## 1. C4 Context — Self-service in the existing ecosystem

```mermaid
flowchart LR
    U[Public User<br/>unassisted device] -->|HTTPS| PORTAL[Self-Service Portal<br/>auth-portal extended<br/>/self-service/:slug/register]
    U -->|HTTPS| OWA[Onboarding Web App<br/>generic-kyc-owa self mode<br/>/self/:slug]
    PORTAL -->|public APIs<br/>resolve/initiate/verify| AUTH[auth-service SB<br/>RBAC + OTP + lifecycle<br/>Cassandra + Keycloak]
    OWA -->|bootstrap/submit/evidence| KYCAPI[KYC Back Office API<br/>kyc-api<br/>Applicant + DOT + PhilSys + MegaMatcher]
    OWA -->|status + review events| AUTH
    KYCAPI -->|review cases| REVIEW[Reviewer UI<br/>KYC_REGISTRATION_REVIEW]
    REVIEW -->|approve / redo / reject| KYCAPI
    KYCAPI -->|status callback| AUTH
    AUTH -->|notify| MAIL[Email sender]
    AUTH -->|notify| SMS[SMS provider<br/>new adapter]
    AUTH -->|create/disable user| KC[Keycloak<br/>1 realm per tenant]
    PORTAL -->|login + OIDC| AUTH
    OWA -->|launch token| AUTH
    AUTH -->|audit| AUDIT[(audit_trail)]
```

## 2. Container view — auth-service internals (new in bold)

```mermaid
flowchart TB
    subgraph PORTALS[Portals]
        P1[Register pages<br/>verify/password/done]
        P2[Login + OIDC callback]
        P3[OWA self pages<br/>ID/selfie/forms]
    end
    subgraph AUTH[auth-service SB]
        PUB[**PublicSelfRegistrationController<br/>/public/tenants/resolve<br/>/public/registration/*<br/>/public/config/:slug**]
        PRIV[Existing: /token /login/face<br/>/user/* /admin/* /oidc/*]
        SVC[**SelfRegistrationService<br/>PendingService + OTPVerify**]
        LSVC[**LifecycleService<br/>append-only history**]
        KSVC[Existing: TenantService<br/>AuthenticationService<br/>UserService + KeycloakUtils<br/>OTPUtils + EmailSenderUtils]
        FILT[Filters: RateLimit public<br/>+ existing Authorized/<br/>Authenticated/AppID/Permissions]
    end
    subgraph DATA[Data - Cassandra auth_system]
        T[(tenants<br/>+ slug)]
        PR[(pending_registrations<br/>+ registration_otps)]
        LH[(user_lifecycle_history<br/>+ kyc_attempts)]
        US[(users<br/>+ kyc_status)]
    end
    P1 --> PUB
    P2 --> PRIV
    P3 --> PRIV
    PUB --> FILT --> SVC --> KSVC
    SVC --> LSVC
    KSVC --> DATA
    LSVC --> DATA
```

> New tables: `pending_registrations`, `registration_otps`, `user_lifecycle_history`, `kyc_attempts` (+ `users.kyc_status` etc.). Full DDL in `04-data-model.md`.

## 3. End-to-end user journey (two steps)

```mermaid
stateDiagram-v2
    [*] --> Register: open tenant registration link
    Register --> OTPVerify: submit identifier, password, captcha
    OTPVerify --> AccountCreated: OTP ok, create login and account
    OTPVerify --> Register: OTP fail, expired, or resend
    AccountCreated --> Login: proceed to login
    Login --> KYCDecision: login returns KYC status
    KYCDecision --> Apps: KYC_VERIFIED
    KYCDecision --> Onboarding: NOT_STARTED, IN_PROGRESS, REDO_REQUIRED
    KYCDecision --> Waiting: KYC_REVIEW
    KYCDecision --> Blocked: REJECTED, SUSPENDED, DEACTIVATED
    Onboarding --> AutoApproved: all checks pass
    Onboarding --> InReview: exception or duplicate
    AutoApproved --> Apps
    InReview --> Reviewer: review case
    Reviewer --> Apps: approve
    Reviewer --> Onboarding: redo as new attempt
    Reviewer --> Blocked: reject
```

## 4. Sequence — Step 1 self-registration (OTP before account)

```mermaid
sequenceDiagram
    autonumber
    actor U as User device
    participant P as Self-Service Portal
    participant A as auth-service (public APIs)
    participant C as CAPTCHA provider
    participant K as Keycloak (tenant realm)
    participant D as Cassandra
    U->>P: Open /self-service/:slug/register
    P->>A: POST /public/tenants/resolve {slug}
    A->>D: lookup tenants by slug
    A-->>P: {tenant_id, realm_id, enabled, policy, captcha key}
    U->>P: Fill identifier + password + T&C + captcha
    P->>A: POST /public/registration/initiate {slug, channel, identifier, pw hash? No—plaintext over TLS, captcha}
    A->>C: verify captcha token
    A->>D: check users_by_username (existence, but do NOT reveal)
    A->>D: INSERT pending_registrations (TTL) + registration_otps (hashed, expiry)
    A->>U: send OTP via email/SMS (message-id only in logs)
    A-->>P: {pending_id, expires_in, resend_cooldown} (uniform)
    U->>P: Enter OTP
    P->>A: POST /public/registration/verify {pending_id, otp}
    A->>D: load otp row, constant-time compare, attempts++
    alt OTP valid
        A->>K: Admin create user {username, email, attrs}
        A->>D: INSERT users + users_by_username + default role + lifecycle [ACCOUNT_CREATED, ACTIVE_KYC_NOT_VERIFIED]
        A->>D: mark otp expended, delete pending
        A-->>P: {account_created}
    else OTP invalid/expired/exhausted
        A->>D: update attempts / expire pending
        A-->>P: generic error (no existence oracle) + retry/resend hints
    end
```

Key points: no `users` or Keycloak row before OTP success; uniform responses; every step audited; rate limits by identifier + IP (see `06`).

## 5. Sequence — Tenant-aware login + KYC decision

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant P as Portal
    participant A as auth-service
    participant K as Keycloak
    participant D as Cassandra
    U->>P: Login {identifier, password} on /self-service/:slug/login
    P->>A: POST /public/tenants/resolve {slug} (cached)
    P->>A: POST /token {username, password, realm_id}
    A->>D: tenants by realm_id → active check
    A->>D: users + users_by_username → active/blocked/deleted checks
    A->>K: password grant (tenant realm)
    A->>D: read users.kyc_status (+ attempt_no)
    A->>D: reset login attempts, save session, audit LOGIN
    A-->>P: {access_token, session_state, customer_id, kyc_status, kyc_action{action, onboarding_url}}
    alt KYC_VERIFIED
        P->>A: GET /apps + /user/access-rights (full list)
    else NOT_VERIFIED / IN_PROGRESS / REDO_REQUIRED
        P->>A: GET /apps (KYC-gated omitted) + show Start/Continue KYC
    else KYC_REVIEW
        P-->>U: Under-review screen (no relaunch)
    else REJECTED / SUSPENDED
        P-->>U: Blocked screen + reason + support
    end
```

OIDC apps receive `kyc_verified` + `kyc_status` claims in `id_token` (see `03 §3.3`) so direct API calls cannot bypass the portal.

## 6. Sequence — Step 2 KYC onboarding (self mode, PhilSys + 1:N)

```mermaid
sequenceDiagram
    autonumber
    actor U as Logged-in user
    participant O as OWA (self mode)
    participant A as auth-service
    participant K as KYC-API
    participant DOT as DOT Innovatrics
    participant PS as PhilSys eVerify
    participant MM as MegaMatcher ABIS
    U->>O: Open onboarding_url (Bearer + slug)
    O->>A: GET /user/access-rights (prove KYC_SELF_ONBOARD)
    O->>K: POST /kyc/self/bootstrap {slug} (X-Tenant-ID) → {applicant_id, attempt_no} (resume IN_PROGRESS)
    O->>U: ID select → capture front/back
    O->>DOT: POST /dot/upload + /customers/inspect-id
    DOT-->>O: tamper/MRZ/expiry + portrait crop
    O->>U: OCR confirm (keep ocr vs user vs verified provenance)
    alt ID is PNID
        O->>PS: POST /psa/query/qr {pnid, face}
        PS-->>O: match/mismatch event (stored, not overwriting)
    end
    O->>DOT: POST /customers/inspect-selfie (quality + liveness + doc-portrait similarity)
    O->>MM: POST /biometric + /verify/face (1:N identify)
    alt 1:N hit (similar face or info)
        MM-->>K: {duplicate_id, hit_score} → PENDING adjudication (never auto-merge)
        K->>K: attempt UNDER_REVIEW, user sees generic under-review only
    end
    O->>U: Additional-info form (tenant-config fields)
    O->>K: POST /kyc/self/attempts/{n}/submit
    K->>K: rules engine: ID ok ∧ OCR ok ∧ PhilSys ok (if PNID) ∧ liveness ∧ 1:N ok ∧ info complete?
    alt all pass
        K->>A: callback KYC_VERIFIED {applicant_id, attempt_no}
        A->>A: lifecycle append KYC_VERIFIED, users.kyc_status=VERIFIED
    else needs human
        K->>K: create review_case {issue, priority} + attempt UNDER_REVIEW
        K->>A: callback KYC_REVIEW
        A->>A: lifecycle append KYC_REVIEW
    end
```

## 7. Sequence — Review (approve / redo / reject) + re-onboarding

```mermaid
sequenceDiagram
    autonumber
    participant R as Reviewer (KYC_REGISTRATION_REVIEW)
    participant J as Adjudicator (BACK-OFFICE ONLY)
    participant Q as Reviewer UI
    participant K as KYC-API
    participant A as auth-service
    participant U as User
    R->>Q: Open queue GET /review-cases?tenant&status=PENDING
    Q->>K: GET /review-cases/{id} (scoped evidence: match results, no raw templates)
    opt Biometric hit - back-office adjudication first
        J->>K: POST /review-cases/{id}/adjudicate {verdict}
        K->>K: DIFFERENT_PERSON clears hit and resumes path, else duplicate or redo handling
    end
    alt Approve
        R->>K: POST /review-cases/{id}/approve {notes}
        K->>A: callback KYC_VERIFIED
        A->>A: lifecycle KYC_VERIFIED + notify (email/SMS, templates)
    else Redo
        R->>K: POST /review-cases/{id}/request-redo {reason_code, instructions}
        K->>K: attempt REDO_REQUESTED, applicant REDO_REQUIRED
        K->>A: callback REDO_REQUIRED {reason, instructions}
        A->>A: lifecycle REDO_REQUIRED + notify (non-technical wording)
        U->>U: Login → Continue KYC → same OWA → NEW attempt_no → reprocess (§6)
    else Reject
        R->>K: POST /review-cases/{id}/reject {reason_code}
        K->>A: callback KYC_REJECTED
        A->>A: lifecycle KYC_REJECTED + notify + identifier cool-down
    end
```

## 8. State machines

### 8.1 KYC status (auth materialised view; HLR §18 overall status)

```mermaid
stateDiagram-v2
    [*] --> NOT_STARTED: account created
    NOT_STARTED --> IN_PROGRESS: bootstrap or start
    IN_PROGRESS --> IN_PROGRESS: resubmit within attempt
    IN_PROGRESS --> KYC_REVIEW: submit, needs review
    IN_PROGRESS --> KYC_VERIFIED: submit, auto-approve
    KYC_REVIEW --> KYC_VERIFIED: reviewer approve
    KYC_REVIEW --> REDO_REQUIRED: reviewer redo
    KYC_REVIEW --> KYC_REJECTED: reviewer reject
    REDO_REQUIRED --> IN_PROGRESS: user starts new attempt
    KYC_VERIFIED --> [*]
    KYC_REJECTED --> [*]
```

### 8.2 Attempt status (per onboarding submission; HLR §18)

```mermaid
stateDiagram-v2
    [*] --> IN_PROGRESS: bootstrap creates attempt N
    IN_PROGRESS --> SUBMITTED: user submits
    SUBMITTED --> UNDER_REVIEW: rules, review case
    SUBMITTED --> APPROVED: rules, auto-approve
    UNDER_REVIEW --> APPROVED: reviewer approve
    UNDER_REVIEW --> REDO_REQUESTED: reviewer redo
    UNDER_REVIEW --> REJECTED: reviewer reject
    REDO_REQUESTED --> SUPERSEDED: next attempt created
    APPROVED --> SUPERSEDED: newer APPROVED replaces current
    REJECTED --> SUPERSEDED: newer attempt created
    SUPERSEDED --> [*]
    APPROVED --> [*]
    REJECTED --> [*]
```
\* Superseded `APPROVED` rows are retained for audit; "current" = latest `APPROVED`.

### 8.3 Review-case status

```mermaid
stateDiagram-v2
    [*] --> PENDING: case created
    PENDING --> IN_REVIEW: reviewer opens/claims
    IN_REVIEW --> DECIDED: approve, redo, or reject
    DECIDED --> [*]
    note right of IN_REVIEW
        DUP_BIOMETRIC cases need a
        back-office adjudication verdict
        before approve is allowed
    end note
```

## 9. Deployment / network sketch (logical)

```mermaid
flowchart TB
    subgraph CLIENT[Public internet]
        B[Browser / mobile web<br/>portal + OWA]
    end
    subgraph DMZ[DMZ / WAF + rate limit]
        WAF[WAF + bot defence<br/>CAPTCHA verify + throttling]
    end
    subgraph APP[App tier]
        PORTAL[auth-portal static + OWA static]
        AUTH[auth-service SB ×N]
        KYC[KYC-API ×N]
    end
    subgraph DATA[Data tier]
        CAS[(Cassandra<br/>auth_system + customer_kyc)]
        MAR[(MariaDB<br/>applicant domain)]
        GFS[(GFS/HFiles<br/>evidence blobs)]
        SOLR[(Solr<br/>search index)]
    end
    subgraph IDP[Identity]
        KC[Keycloak<br/>per-realm]
    end
    B --> WAF --> PORTAL
    PORTAL --> AUTH
    PORTAL --> KYC
    AUTH --> CAS
    AUTH --> KC
    KYC --> MAR
    KYC --> CAS
    KYC --> GFS
    KYC --> SOLR
```
