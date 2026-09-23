# CPS-272 — Use Cases

What the system does, told through its users. Each use case names its actors (personas in `05`),
preconditions, numbered main flow, alternate flows, and postconditions, then maps to tickets and FRs.
Diagrams for these flows live in `02-architecture-diagrams.md`.

**The four pieces, in plain words:** the **self-service portal** (part of the auth portal) is where a person
creates an account and logs in; the **auth service** owns accounts, logins, permissions, and the audit trail,
backed by one Keycloak login realm per tenant; the **onboarding web app (OWA)** is the identity-verification
module (photograph an ID, take a selfie, fill in details); the **KYC back office** checks the evidence
(document inspector, national-ID check, face matching) and hosts the human review queue. Self-service reuses
the same OWA and back office the frontliners use today — only the way users reach them changes.

---

## UC-01 — Self-registration happy path
- **Actors:** Registrant (primary); auth service, Keycloak, email/SMS sender (system).
- **Preconditions:** Tenant enabled for self-registration with a slug (readable link key); Registrant has an email address or mobile number and no account yet.
- **Trigger:** Registrant opens `https://portal.example.com/self-service/{slug}/register` (for example from an SMS or poster).
- **Main flow:**
  1. Portal resolves the slug to the tenant and shows that tenant's branded form with its password rules.
  2. Registrant picks email or mobile, enters it with a password, accepts the Terms and Privacy Notice, and passes the anti-bot check.
  3. System stores a temporary pending record (not an account) and sends a one-time passcode to the contact.
  4. Registrant enters the code on the verify page.
  5. System creates the Keycloak login and the account (default role, KYC status NOT_STARTED) and writes the first lifecycle history entries.
  6. Done page points the Registrant to login.
- **Alternates:** 3a. Identifier already registered — identical reply, no code sent, nothing revealed. 4a. Wrong code — retry with attempts left counter (see UC-02 for expiry/resend).
- **Postconditions:** One active unverified account; Registrant can log in; audit trail complete.
- **Tickets:** A-01, A-02, A-03, A-04, D-05 (code message), D-06 (protections), D-08 (flags) · **FR:** FR-01…FR-06.

## UC-02 — OTP resend and expiry recovery
- **Actors:** Registrant.
- **Preconditions:** Pending record from UC-01 step 3 exists.
- **Trigger:** Code never arrived, expired (5–10 minutes), or was mistyped too many times.
- **Main flow:**
  1. Registrant taps resend (after the 60-second cooldown, max 5 resends); the old code dies and a new one arrives.
  2. Registrant enters the fresh code and continues UC-01 at step 4.
- **Alternates:** 1a. Pending record expired (30 minutes) — Registrant restarts registration; nothing to clean up (self-deleted). 1b. Resend limit hit — too-many-requests reply with wait time.
- **Postconditions:** Same as UC-01, or a clean restart with no residue.
- **Tickets:** A-02, A-04, D-06 · **FR:** FR-03, FR-06.

## UC-03 — Login with KYC routing
- **Actors:** Registered User.
- **Preconditions:** Account from UC-01 exists.
- **Trigger:** User logs in with contact plus password.
- **Main flow:**
  1. System checks credentials against the tenant's Keycloak realm.
  2. System reads the stored KYC status and returns it with the next action.
  3. Verified users see all their applications; unverified users see only non-gated ones plus a Start/Continue verification button; under-review users see a waiting screen; rejected, suspended, or deactivated users see the reason and support path.
- **Alternates:** 2a. Verification back office unreachable — login still works; gated apps deny, the rest allow.
- **Postconditions:** Correct landing per status; gated APIs deny unverified direct calls (claim-enforced).
- **Tickets:** B-01, B-02 · **FR:** FR-07…FR-10.

## UC-04 — Self KYC onboarding to approval
- **Actors:** Registered User (unverified); OWA; KYC back office (document inspector, national-ID check, face matcher).
- **Preconditions:** Logged in; status NOT_STARTED, IN_PROGRESS, or REDO_REQUIRED.
- **Trigger:** User taps Start/Continue verification.
- **Main flow:**
  1. OWA opens in self mode for the tenant and resumes or starts verification attempt number N.
  2. User picks ID type and photographs front (and back where needed); inspector checks tampering, machine-readable zone, and expiry; user confirms the read data.
  3. Philippine National ID only: system runs the PhilSys national-ID check and files the result as evidence.
  4. User completes liveness and selfie; system checks liveness, portrait similarity, and searches all enrolled faces for duplicates.
  5. User fills the tenant-configured extra form and submits; decision rules approve when every check passes.
  6. Back office notifies auth service; user becomes KYC_VERIFIED with full app access.
- **Alternates:** 4a/5a. Any check inconclusive or a duplicate hit — attempt goes UNDER_REVIEW, user sees "under review" (see UC-05…UC-08).
- **Postconditions:** Verified status, or a review case with full evidence (all attempts preserved).
- **Tickets:** C-01, C-02a, C-02b, C-02c, C-03 · **FR:** FR-11…FR-21, FR-30, FR-31.

## UC-05 — Redo journey to approval
- **Actors:** Reviewer; Registered User.
- **Preconditions:** A review case the reviewer cannot approve as-is (blurry ID, poor selfie, inconclusive liveness, conflicting data).
- **Trigger:** Reviewer chooses Request Redo with a reason code and plain-language instructions.
- **Main flow:**
  1. System marks the attempt redo-requested, sets the user REDO_REQUIRED, and notifies them (email/SMS) without technical details.
  2. User logs in, sees Continue verification, and re-enters the same module.
  3. A brand-new attempt number starts; user retakes ID/selfie or picks another supported ID and resubmits.
  4. New evidence is processed fresh; prior attempts stay on file.
- **Alternates:** 4a. New attempt also fails — another review case per business rules (no silent loops: retry caps force review).
- **Postconditions:** Approval (→ UC-04 step 6) or a new case; complete attempt chain for audit.
- **Tickets:** D-02, D-04, B-02 (call to action), C-02c (submit), D-05 (message) · **FR:** FR-12, FR-26…FR-29.

## UC-06 — Duplicate hit adjudicated: same person, rejected
- **Actors:** Adjudicator (primary); Reviewer; Registered User.
- **Preconditions:** Face search matched an existing enrolled identity (UC-04 step 4); case PENDING adjudication.
- **Trigger:** Adjudicator opens the candidate comparison (probe photo versus matched candidates, side by side).
- **Main flow:**
  1. Adjudicator compares the images and data and records SAME_PERSON with confidence and notes.
  2. Case converts to duplicate handling; Reviewer rejects (optionally escalates to fraud per tenant policy).
  3. User is notified of rejection with reason and support path; contact enters cool-down against re-registration.
- **Alternates:** 1a. Evidence unclear — INCONCLUSIVE verdict routes to UC-05 with fresh-capture instructions.
- **Postconditions:** No second account created; fraud trail preserved; user told nothing about biometrics.
- **Tickets:** D-02b, D-02, C-02b · **FR:** FR-22, FR-23, FR-25, FR-26, FR-32.

## UC-07 — Duplicate hit adjudicated: different person, approved
- **Actors:** Adjudicator; Registered User.
- **Preconditions:** Same as UC-06.
- **Trigger:** Adjudicator opens the comparison.
- **Main flow:**
  1. Adjudicator records DIFFERENT_PERSON — lookalike, not a duplicate.
  2. The hit clears; the attempt resumes the automated decision path and approves if all else passes.
  3. User becomes verified without ever knowing a match was examined.
- **Postconditions:** Legitimate user unblocked; verdict and evidence retained for audit.
- **Tickets:** D-02b, C-02b, B-01 · **FR:** FR-22, FR-25, FR-26.

## UC-08 — Reviewer triage and decision
- **Actors:** Reviewer.
- **Preconditions:** One or more PENDING cases (any issue type).
- **Trigger:** Reviewer opens the queue at shift start.
- **Main flow:**
  1. Queue lists cases by priority and deadline; Reviewer claims one.
  2. Case detail shows account reference, submitted ID, OCR values with provenance, national-ID result, liveness, match result (scores only), extra info, audit trail, prior attempts, and the potential match side by side where permitted.
  3. Reviewer approves, requests redo (→ UC-05), or rejects with reason codes; each writes audit, notifies the user, and updates auth status.
- **Alternates:** 2a. Duplicate-type case — approve is server-blocked until adjudication (→ UC-06/UC-07).
- **Postconditions:** Case DECIDED; user notified; auth lifecycle mirrors the outcome.
- **Tickets:** D-01, D-02, D-03 · **FR:** FR-22…FR-26, FR-32, FR-33.

## UC-09 — Admin suspension and investigation
- **Actors:** Administrator; Support Analyst.
- **Preconditions:** Report of abuse or a user support ticket.
- **Trigger:** Admin opens the user record.
- **Main flow:**
  1. Support Analyst reads the immutable lifecycle history (every registration, login, attempt, decision, notification) and exports the journey.
  2. Administrator suspends (or reinstates) the account with a reason; login immediately honors it.
- **Postconditions:** Account contained or restored; every action itself in the history.
- **Tickets:** B-03, D-07 · **FR:** FR-06, FR-32.

## UC-10 — Assisted registration still works (regression)
- **Actors:** Frontliner; walk-in Registrant.
- **Preconditions:** Frontliner holds a provisioned staff account; tenant OWA build unchanged.
- **Trigger:** Walk-in citizen without a capable device asks for help.
- **Main flow:** Frontliner logs into OWA as today, creates the user through the admin endpoint, and operates capture on the shared device; verification, review, and redo behave exactly as before.
- **Postconditions:** Assisted and self-service users end in identical data shapes and statuses.
- **Tickets:** C-01 (regression proof), D-08 (pilot parity check) · **FR:** all (no regression).

---

## Coverage matrix (use case → tickets → FRs)

| Use case | Tickets | FRs |
|---|---|---|
| UC-01 Happy-path registration | A-01, A-02, A-03, A-04, D-05, D-06, D-08 | FR-01…FR-06 |
| UC-02 OTP recovery | A-02, A-04, D-06 | FR-03, FR-06 |
| UC-03 Login routing | B-01, B-02 | FR-07…FR-10 |
| UC-04 Self onboarding approval | C-01, C-02a/b/c, C-03 | FR-11…FR-21, FR-30, FR-31 |
| UC-05 Redo journey | D-02, D-04, B-02, C-02c, D-05 | FR-12, FR-26…FR-29 |
| UC-06 Same-person adjudication | D-02b, D-02, C-02b | FR-22, FR-23, FR-25, FR-26, FR-32 |
| UC-07 Cleared-hit approval | D-02b, C-02b, B-01 | FR-22, FR-25, FR-26 |
| UC-08 Reviewer triage | D-01, D-02, D-03 | FR-22…FR-26, FR-32, FR-33 |
| UC-09 Suspension/investigation | B-03, D-07 | FR-06, FR-32 |
| UC-10 Assisted regression | C-01, D-08 | All (parity) |
