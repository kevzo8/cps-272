# CPS-272 — Glossary

Short definitions for reviewers and implementers. Normative details live in the annex referenced per term.

## Tenancy & identity

| Term | Meaning | Example / where used |
|---|---|---|
| **Slug** | Short, URL-safe, human-readable nickname for a tenant, used in self-service links (`/self-service/:slug/...`). Lowercase letters, numbers, hyphens (`[a-z0-9-]{3,64}`); unique and effectively immutable. | `navotas-demo` in `/self-service/navotas-demo/register`. Stored in `tenants.slug` + `tenants_by_slug` (`04 §2.1`) |
| **Tenant** | One customer organisation in the RBAC system. Owns users, roles, apps, settings. | `tenants{tenant_id, realm_id, ...}` (`04 §2.1`) |
| **Realm (Keycloak realm)** | The Keycloak identity domain backing one tenant. 1 tenant = 1 realm. Authentication (password grant) happens against the tenant's realm. | `tenants.realm_id`, e.g. `NAVOTAS_DEMO` (`01 §5`) |
| **Client ID** | Public identifier of the portal app within a realm. Safe to expose in the browser. | Sent as `X-Client-ID` (`03 §1`) |
| **Client secret** | Confidential credential for server-to-server calls (Keycloak, KYC-API). Never sent to or accepted from the browser. | Injected server-side only (CPS-164, `01 §5.3`) |
| **App ID** | Identifier of the application the user is accessing (portal, OWA). Used by `AppIDFilter` to check the user is associated with that app. | `X-App-ID` header (`03 §1`) |

## Registration (Step 1)

| Term | Meaning | Example / where used |
|---|---|---|
| **Pending registration** | Pre-account record created at `initiate` time. Holds the encrypted identifier + hashed password until OTP is verified. Self-deletes via TTL if abandoned. Never a `users` row. | `pending_registrations` table, 30-min TTL (`04 §2.3`) |
| **OTP (one-time passcode)** | Short single-use code sent to the user's email/mobile proving they control that contact. Hashed (HMAC-SHA256) at rest, short expiry, attempt-capped. | `registration_otps` table (`04 §2.4`) |
| **Contact verification** | Step 1 question: "Do you control this email/mobile number?" Proves ownership, says nothing about real-world identity. | HLR separation principle (`01 §3`) |
| **Uniform response** | API reply shaped identically whether an identifier/tenant exists or not, so callers can't probe for real accounts. Includes timing normalisation. | `202 + pending_id` either way (`06 §4`) |
| **T&C / privacy version** | Version stamp of the Terms and Privacy Notice the user accepted (e.g. `2026-08-01`), stored on the account for compliance. | `users.tnc_version` (`04 §2.2`) |

## KYC & review (Step 2)

| Term | Meaning | Example / where used |
|---|---|---|
| **Identity verification (KYC)** | Step 2 question: "Are you the legitimate person behind this account?" ID + liveness + biometrics + records checks. | `01 §7` |
| **Applicant** | The KYC-side record for a person being verified (may predate or outlive any single attempt). | `applicant{applicant_id, tenant_id, ...}` (`04 §4.1`) |
| **Onboarding attempt** | One self-contained KYC submission (captures, OCR, verification results). Numbered per applicant; old attempts are `SUPERSEDED`, never overwritten. Latest `APPROVED` = current record. | `kyc_attempt`, `attempt_no` (`01 §7.3`, `04 §4.2`) |
| **1:N matching** | Biometric search of one face/fingerprint against many enrolled identities, used to detect the same person holding another account. A hit always opens a review case — never auto-merges or auto-rejects. | MegaMatcher `/biometric` (`01 §7.2`) |
| **Adjudication** | Back-office-only verdict on a 1:N hit: `SAME_PERSON` (duplicate → reject/fraud path), `DIFFERENT_PERSON` (hit cleared, automated path resumes), or `INCONCLUSIVE` (→ redo). Hits stay PENDING until verdict; users only ever see "under review". | `01 §8.2`, `03 §5.3`, `05 D-02b` |
| **PhilSys eVerify (PNID)** | Government verification invoked when the submitted ID is a Philippine National ID. Result stored as an event, never overwriting user data. | `/psa/query/qr` (`01 §7.2`) |
| **OCR provenance** | Keeping three separate values: data extracted by OCR vs confirmed by the user vs obtained from external verification — for auditability. | `ocr_provenance{ocr, user_confirmed, externally_verified}` (`04 §4.2`) |
| **Review case** | Work item for a KYC submission needing human review. States `PENDING → IN_REVIEW → DECIDED`; outcomes exactly approve / request-redo / reject. | `review_case` table (`04 §4.3`) |
| **Redo (REDO_REQUIRED)** | Reviewer decision asking the user to re-verify (unclear ID, poor selfie, possible duplicate…). User is notified in plain language and retries in the same module as a new attempt. | `01 §8.3`, `05 D-04` |
| **KYC status** | The user's current identity-verification state: `NOT_STARTED / IN_PROGRESS / KYC_REVIEW / REDO_REQUIRED / KYC_VERIFIED / KYC_REJECTED`. Stored as a materialised cache on `users`, truth in history. | `users.kyc_status` (`04 §2.2`) |

## Data & authorization

| Term | Meaning | Example / where used |
|---|---|---|
| **Lifecycle history** | Append-only per-user log of every account/KYC transition and significant activity. Insert-only; current state = latest row. The system-of-record for the user's journey. | `user_lifecycle_history` (`04 §2.5`) |
| **Materialised status** | Cached copy of the latest lifecycle state on the `users` row for fast login/gating reads. Updated only by the lifecycle-append handler. | `users.kyc_status` (`04 §3`) |
| **KYC as attribute** | Design rule: verification state travels with the user as data (and token claim), not as roles like `KYC_VERIFIED_CUSTOMER` — avoiding role explosion. | `User{roles[], kyc_status}` + `kyc_verified` claim (`01 §9`) |
| **`applicant_id` vs `linked_person_id`** | `applicant_id` = KYC applicant record from onboarding; `linked_person_id` = established person identity linked after verification. Both carried on `users`. | `User` entity (`04 §1`) |
| **Status callback** | Idempotent service-to-service call by which KYC-API notifies auth-service of authoritative KYC transitions (with `idempotency_key`; stale events never regress state). | `POST /internal/kyc-status-callback` (`03 §5.4`) |
| **BPO** | Business-process outsourcing / work-item engine for queueing tasks to human workers. Used for reviewer assignment, not per-registration tracking (volume: millions). | `01 §10.3` |

## People & roles (who is who — see personas table in `05`)

| Term | Meaning | Example / where used |
|---|---|---|
| **Registrant** | The individual onboarding themselves; no account yet through just-created account. *The person to be onboarded.* | `05` A-01…A-04 |
| **Registered User** | A registrant after account creation, logged in; carries `kyc_status`. | `05` B/C/D stories |
| **Tenant** | The customer organisation (LGU, agency, enterprise) — owns realm, config, users. An organisation, never a login identity. | `tenants` table |
| **Administrator** | RBAC/platform admin: provisions staff, configures tenants, suspends accounts. Never self-registered. | `05` B-03 |
| **Frontliner** | Assisted-channel staff driving OWA on a shared device for walk-ins (today's flow). | As-is flow, `05` C-01 regression |
| **Reviewer** | Back-office worker deciding cases (approve/redo/reject). Holds `KYC_REGISTRATION_REVIEW`. | `05` D-01…D-03 |
| **Adjudicator** | Back-office biometric specialist deciding 1:N hits. Holds `KYC_BIOMETRIC_ADJUDICATION`. | `05` D-02b |
| **Support Analyst / Auditor** | Investigates journeys via exports; needs immutable traceable records for compliance. | `05` C-03, D-07 |
| **Security Engineer / Release Manager** | Owns public-surface hardening; owns pilot flags and rollback. | `05` D-06, D-08 |

## Process & requirements (how to read the tickets in `05`)

| Term | Meaning | Example / where used |
|---|---|---|
| **HLR** | High-Level Requirements — the source requirements file this design implements (`HLR-Self-Service-Registration-KYC.md`: "High-Level Requirements: Self-Service Registration and KYC Onboarding for RBAC"). `HLR §n` cites its sections (e.g. HLR §5 = registration mockups; HLR §20 = the FR-01…FR-33 list). | `05` FR column, TDD §3 |
| **FR** | Functional Requirement — one numbered requirement from the HLR (FR-01…FR-33, e.g. FR-03 = verify contact ownership before creating the account). Every story lists the FRs it implements so coverage is auditable. | `05` per-ticket FR mapping |
| **NFR** | Non-Functional Requirement — quality attributes from HLR §21 (security, rate limiting, auditability, configurability) rather than user-visible behaviour. | `05` D-06/D-07, `06` |
| **DoD** | Definition of Done — the CPS-272 exit criteria (design reviewed, HLR addressed, KYC-BO integration defined, tickets created/linked, risks documented). | `00` traceability table |
| **AC** | Acceptance Criteria — the checkbox list on each ticket; all boxes must pass for the ticket to close. Doubles as the UAT script. | `05` per-ticket checkboxes |
| **UC** | Use Case — one end-to-end scenario with actors, main and alternate flows (UC-01…UC-10). | `08-use-cases.md` |

## Platform & quality terms

| Term | Meaning | Example / where used |
|---|---|---|
| **RBAC** | Role-based access control — users get permissions through roles (directly or via groups), never hard-coded per user. | Auth-service model, `01 §9` |
| **Keycloak / IdP** | The identity provider: the system that stores login credentials and checks passwords. One Keycloak realm serves one tenant. | Login, `05` A-03 |
| **OIDC (token/claim)** | OpenID Connect: the standard by which apps learn who logged in. The identity token carries claims (small facts such as `kyc_verified`). | `05` B-01, `03 §3.3` |
| **CAPTCHA** | "Completely Automated Public Turing test to tell Computers and Humans Apart" — anti-bot check (image puzzle, risk score, or equivalent) required before sending verification codes. | `05` A-02, `06 §3` |
| **TTL** | Time to live — automatic self-delete timer on temporary data (pending records, codes). | `04 §2.3–§2.4` |
| **SLA** | Service-level agreement — here, the deadline by which a review or adjudication case must be decided (for example 4 hours for duplicate hits). | `05` D-01, DT-03 |
| **PII** | Personally identifiable information (names, ID numbers, faces). Minimised in logs; masked in UI; raw forms gated. | `06 §6` |
| **E2E / UAT** | End-to-end test (full journey across systems); user acceptance testing (customer sign-off run, here the checkbox criteria and `08` scenarios). | `05`, `08` |
| **Cool-down** | Configured period during which a rejected or duplicate-linked contact cannot register again (prevents retry loops). | `05` D-02b, `01 §8.2` |
| **PNID** | Philippine National ID number — the ID type that triggers the PhilSys national check. | `05` C-02b |
| **OWA** | Onboarding Web App — the Angular identity-verification module (ID capture, selfie, forms). Runs assisted (frontliner-driven) and self-service modes. | `01 §7`, `05` C-01 |
| **OCR** | Optical Character Recognition — reading text (name, birthdate, ID number) out of ID photos. | `05` C-02a |
| **WAF** | Web Application Firewall — edge filter blocking malicious traffic before it reaches the public APIs. | `09` pre-launch |
| **CORS** | Cross-Origin Resource Sharing — browser rules deciding which web origins may call the APIs; allowlisted per tenant slug. | `06 §8` |
| **HMAC-SHA256** | Hash-based Message Authentication Code with SHA-256 — the one-way hashing used for stored OTP codes, checked with constant-time comparison. | `05` A-02 |
| **JWT** | JSON Web Token — signed token carrying login and session claims (including the new KYC claims). | `05` B-01 |
| **UUID** | Universally Unique Identifier — random IDs used for tenants, users, sessions, and idempotency keys. | `04` |
| **ABIS** | Automated Biometric Identification System — the MegaMatcher engine behind one-to-many face/fingerprint search and adjudication. | `05` C-02b, D-02b |
| **GFS / HFiles** | The file/object storage holding raw evidence blobs (ID photos, selfies, video); databases keep only references and hashes. | `04 §4.4` |
| **PKCE** | Proof Key for Code Exchange — the challenge/verifier mechanism securing the OIDC login code flow. | Portal login |
| **TLS** | Transport Layer Security — encryption in transit (HTTPS). | `06` |
| **CDN** | Content Delivery Network — hosts the Mermaid and markdown libraries the deck loads on first view. | Presentation README |
| **TDD** | Technical Design Document — this design (`TDD-CPS-272-Self-Service-Registration-KYC.md`, team template). | Index |
| **CPS** | Jira project prefix for this workstream's tickets (CPS-272 epic, A/B/C/D stories). | `05` |
| **LGU** | Local Government Unit — the typical tenant organisation (also: agencies, enterprises). | `05` personas |
| **WAR** | Web Application Archive — the Java deployable built for the portal and onboarding backends. | Build docs |
