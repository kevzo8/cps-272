# High-Level Requirements: Self-Service Registration and KYC Onboarding for RBAC (For Review)

> **Provenance:** transcribed from the approved/for-review requirements text supplied with CPS-272.
> This file is the citation target for every `HLR §n` and `FR-nn` reference across `docs/cps-272/`.
> The three architecture images from the original (`image-20260814-082757/082917/083023.png`) are not
> embedded here; their captions are preserved below. The companion BPO-applicability note referenced
> in §3 is a separate document and is not included.

## 1. Objective

The solution will provide a two-step self-service registration process:

- **Account Registration** – Any user can create an RBAC account using a verified email address or mobile number as the username.
- **KYC Onboarding** – After login, users who need access to KYC-protected applications/features must complete identity verification.

The design separates account ownership verification from identity verification:

- Step 1: "Do you control this email/mobile number?"
- Step 2: "Are you the legitimate person associated with this account?"

This allows basic account creation to remain simple while ensuring that applications requiring verified identity only become available after successful KYC.

## 2. High-Level Architecture

- `image-20260814-082757.png` — Users who aren't yet verified enter the onboarding module, which captures data and runs it through PhilSys (if the submitted ID is a PNID) before a decision is made.
- `image-20260814-082917.png` — Exceptions land in a review queue, where an authorized reviewer chooses one of three outcomes — and a redo loops the user back into onboarding.
- `image-20260814-083023.png` — (architecture diagram).

## 3. User Account Lifecycle

The account should have explicit lifecycle states to distinguish account registration from KYC.

| Status | Description |
|---|---|
| ACTIVE_KYC_NOT_VERIFIED | Contact information verified and account created, but KYC is incomplete |
| KYC_IN_PROGRESS | User has started KYC onboarding |
| KYC_REVIEW | KYC requires manual review |
| REDO_REQUIRED | Reviewer requires the user to redo the KYC onboarding |
| KYC_VERIFIED | User successfully completed KYC |
| KYC_REJECTED | KYC could not be completed/approved |
| SUSPENDED | Account temporarily blocked by an authorized administrator |
| DEACTIVATED | Account is no longer usable |

Importantly, KYC status should be independent of account activation status. A user can therefore have
Active Account + KYC Not Verified, and can log in, but cannot access applications/features that require KYC.

### Lifecycle History and Audit Trail

Each user account lifecycle transition and significant lifecycle activity shall be recorded in the database
as a new immutable record. The system shall not update or overwrite the previous lifecycle status record.

Example journey: ACTIVE_KYC_NOT_VERIFIED → KYC_IN_PROGRESS → KYC_REVIEW → REDO_REQUIRED →
KYC_IN_PROGRESS → KYC_VERIFIED.

Example lifecycle history:

| Sequence | Status / Activity | Date/Time | Source/Actor |
|---|---|---|---|
| 1 | ACCOUNT_CREATED | Aug 14, 2026 10:00 | System |
| 2 | ACTIVE_KYC_NOT_VERIFIED | Aug 14, 2026 10:00 | System |
| 3 | KYC_STARTED | Aug 14, 2026 10:15 | User |
| 4 | KYC_SUBMITTED | Aug 14, 2026 10:25 | User |
| 5 | KYC_REVIEW | Aug 14, 2026 10:26 | System |
| 6 | REDO_REQUIRED | Aug 15, 2026 09:30 | Reviewer |
| 7 | KYC_STARTED | Aug 16, 2026 14:10 | User |
| 8 | KYC_SUBMITTED | Aug 16, 2026 14:25 | User |
| 9 | KYC_VERIFIED | Aug 16, 2026 14:26 | System |

### Lifecycle History Requirements

- Each lifecycle transition shall be inserted as a new database record.
- Previous lifecycle records shall not be updated or deleted as part of normal processing.
- Each record should contain, at minimum: User Account ID; Lifecycle Status or Activity; Timestamp;
  Source/Actor; Related KYC Attempt ID, where applicable; Reason or remarks, where applicable;
  Reference to the related transaction/case, where applicable.
- The history shall support both status transitions and significant activities that do not necessarily change the account's current status.
- The system shall be able to determine the user's current state from the latest applicable lifecycle record.
- The lifecycle history shall support audit, troubleshooting, compliance review, and investigation of the user's registration and KYC journey.
- KYC onboarding attempts should have their own identifiers so that lifecycle events can be associated with the specific onboarding attempt that generated them.
- Lifecycle history records should be treated as immutable audit records. Any correction or subsequent action should result in another record rather than modifying the historical record.

The lifecycle history is intended to serve as the system of record for the user's account journey, while the
existing BPO service shall continue to be used for activities that require queue management and assignment
to authorized workers. The applicability of BPO for self-service registration lifecycle management is discussed
separately in "BPO Applicability for Self-Service Registration Lifecycle Management". The detailed design should
ensure that lifecycle history can support the expected volume of potentially millions of self-service
registrations without requiring each registration activity to become a BPO work item.

## 4. Step 1 – Self-Service Account Registration

### 4.1 Registration Options

The user should be able to register using either email address or mobile number. The selected email/mobile
number becomes the user's login identifier/username.

Basic registration information: Email OR mobile number; Password; Confirm password; Acceptance of Terms and
Conditions / Privacy Notice; CAPTCHA or equivalent anti-bot mechanism, where appropriate.

### 4.2 Contact Verification Before Account Creation

The system should not create the final active user account immediately. Instead: user enters email/mobile and
password; system validates the format; system checks whether the identifier is already registered; system
generates a one-time verification code/link; verification code/link is sent to the email/mobile; user enters
the OTP or follows the verification link; system validates the verification. Only after successful verification
is the user account created/activated.

Security requirements: OTP must have a short expiration period. OTP must be single-use. Limit OTP resend
attempts. Limit failed verification attempts. Apply rate limiting by identifier, device/IP and other appropriate
signals. Do not reveal whether a specific email/mobile number belongs to an existing account. Password must meet
the organization's password policy and be securely hashed. Verification events must be audited.

## 5. Registration UI Flow

Screen 1 – Create Account: channel selector (Email / Mobile Number); Email/Mobile Number field; Password;
Confirm Password; Terms and Conditions checkbox; Privacy Notice acknowledgement; [Create Account].

Screen 2 – Verify Contact Information: "A verification code has been sent to:" masked destination
(`xxxx@example.com` or `09******1234`); six-box Verification Code entry; [Verify]; "Didn't receive the code?
[Resend Code]". Successful verification: "Account successfully created." The user can then proceed to login.

## 6. Step 2 – Login and KYC Access Decision

After registration, the user logs in using email + password, or mobile number + password. The authentication
service validates the credentials. After successful authentication, the system retrieves the user's KYC status.

Decision: Login Successful → Check KYC Status → KYC VERIFIED leads to normal access to authorized applications;
NOT VERIFIED leads to start/continue KYC onboarding. If the user's status is REDO_REQUIRED, the system shall
notify the user that additional verification is required and provide access to the KYC onboarding module to
start a new onboarding attempt.

## 7. Application-Level KYC Enforcement

KYC should not only be checked by the portal UI. The authorization layer should also expose KYC status as an
authorization attribute/claim, where appropriate. Example user: Account Status ACTIVE, KYC Status VERIFIED,
plus Roles and Permissions. An application requiring KYC can enforce: Account ACTIVE AND Required
Role/Permission AND KYC VERIFIED. This prevents a user from bypassing the UI and directly invoking an
application/API that requires KYC.

## 8. KYC Onboarding Module

When a logged-in user does not have verified KYC status, the system should redirect or launch the KYC
Onboarding Module, supporting: ID type selection; ID image capture/upload; OCR; ID data extraction and
validation; liveness detection; selfie capture; biometric 1:N matching; PhilSys verification when the ID is
PNID; additional personal information; KYC decision; exception handling; manual review where required.

## 9. KYC UI Flow

- Screen 1 – KYC Introduction: "Verify Your Identity… You will need: a valid government-issued ID, your face/selfie, additional personal information." [Start Verification].
- Screen 2 – ID Selection: ID Type dropdown (e.g. Philippine National ID / PNID); [Continue]. The selected ID type determines the applicable verification process.
- Screen 3 – ID Capture: front of ID camera preview + [Capture]; back of ID where applicable.
- Screen 4 – OCR Processing: extracted information shown for review (Full Name, Date of Birth, masked ID Number) with [Confirm] / [Retake ID]. The system should retain the distinction between data extracted from OCR, data supplied by the user, and data obtained from external verification (important for auditability).

## 10. PNID / PhilSys Verification

If the selected ID is PNID, the system should invoke the PhilSys verification service: PNID Selected →
Capture PNID → OCR / Extract PNID Information → PhilSys eVerify → Compare Verification Result → Successful
continues KYC, Failed goes to Exception Handling. The result should be stored as a verification event, rather
than simply replacing the user's submitted information.

## 11. Liveness and Selfie Capture

Prepare for Face Verification → Liveness Check → Selfie Captured → Biometric 1:N Matching. The system should
determine whether the submitted face is from a live person, whether the biometric matches an existing
person/account within the authorized biometric population, and whether the match generates a potential
duplicate/conflict. Because the biometric process is 1:N, the system should specifically account for the
possibility that the same person may already have another account.

## 12. Additional Information

After biometric and ID processing, the system can request additional information required by the client's KYC
policy (address, contact information, other demographics, required declarations, client-specific information).
The module should make the required fields configurable rather than hard-coding them into the RBAC system.

## 13. Automated KYC Decision

All required checks passing (ID valid AND OCR successful AND PhilSys verified if PNID AND liveness passed AND
biometric match / no duplicate conflict AND required information complete) yields KYC VERIFIED: "Identity
verification successful. You can now access applications and features that require KYC verification." If the
result requires non-real-time review, the system shall create a review case and set the KYC status to KYC_REVIEW.

## 14. KYC Exception and Duplicate Conflict Management

The KYC review process shall be non-real-time. Submissions that cannot be automatically approved shall be placed
in a KYC Review Queue for review by an authorized person, with one of three decisions:

### 14.1 Approve

If the submitted information and verification results are acceptable, the reviewer can approve. Result: user's
KYC status is updated to KYC_VERIFIED.

### 14.2 Request KYC Redo

If additional verification is required (unclear or invalid ID capture, inconclusive OCR, poor-quality selfie,
liveness issue, potential duplicate biometric match, conflicting identity information, need for another
supported ID, other client-defined exceptions), the reviewer can request a redo with reason and instructions.
The system shall then: update KYC status to REDO_REQUIRED; notify the user (email or mobile); allow login and
continuation of KYC; redirect to the same KYC Onboarding Module; allow retaking ID and selfie, another
supported ID, and/or updated information; create a new onboarding attempt and re-run verification. The user does
not need technical details; the notification simply instructs completion of identity verification again.

### 14.3 Reject

If the registration cannot proceed, the reviewer may reject it. A rejection reason shall be recorded, and the
user shall be notified accordingly.

## 15. KYC Onboarding Attempt Versioning

Each KYC submission shall be treated as a separate onboarding attempt. A new attempt shall not overwrite or
delete the data from previous attempts (e.g. Attempt #1 with review decision REDO, then Attempt #2 with new
captures resulting in APPROVED). The previous attempt shall remain available for audit and traceability
purposes, subject to the applicable data retention policy. The latest approved attempt shall become the user's
current KYC record.

Duplicate biometric handling: a potential duplicate detected through 1:N matching creates a review case rather
than automatically rejecting or merging the account. The reviewer may request a redo; the new attempt undergoes
verification again; if it passes, the user may be approved; if another conflict is detected, a new review case
may be created per business rules. The system shall not automatically delete, overwrite, or merge previous
onboarding data. All previous attempts, verification results, reviewer decisions, and timestamps shall be
retained as part of the audit trail.

## 16. Reviewer UI

Review queue showing Case, User, Issue, Priority, Status (e.g. KYC-001 / New User / Potential Duplicate / High /
Pending; KYC-002 / New User / ID-Name Conflict / Medium / Pending). Access restricted by a dedicated
administrative permission, e.g. KYC_REGISTRATION_REVIEW.

### 16.1 Review Case Details

Visible: account identifier; registration date/time; KYC status; submitted ID information; OCR results; PhilSys
verification result where applicable; liveness result; biometric matching result; additional information; audit
history; previous KYC attempts where applicable. Potential existing match: existing account reference; existing
KYC status; matching confidence/result; relevant identity attributes; existing biometric enrollment reference.
Sensitive biometric data should be appropriately protected; reviewers should generally see match results and
controlled evidence rather than unrestricted raw biometric templates.

## 17. User Notification and Re-Onboarding Flow

Reviewer → Request KYC Redo → System updates KYC Status to REDO_REQUIRED → Send Notification (Email and
Mobile/SMS if applicable) → User Logs In → System Detects REDO_REQUIRED → "Continue KYC Verification" → Same
KYC Onboarding Module → New KYC Attempt → Automated KYC Processing. The notification should provide a general
instruction and should not expose sensitive details such as biometric match information.

## 18. KYC Status and Attempt Status

Overall KYC status and per-attempt status are maintained separately.

Overall KYC status: NOT_STARTED (not started) · IN_PROGRESS (currently completing) · KYC_REVIEW (needs
non-real-time review) · REDO_REQUIRED (reviewer requires redo) · KYC_VERIFIED (completed) · KYC_REJECTED
(rejected).

KYC attempt status: IN_PROGRESS (current attempt) · SUBMITTED (submitted for processing) · UNDER_REVIEW (being
reviewed) · REDO_REQUESTED (reviewer requested a new attempt) · APPROVED (successful KYC) · REJECTED (rejected) ·
SUPERSEDED (replaced by a newer attempt).

## 19. Recommended RBAC Integration

KYC should become an attribute used together with roles and permissions, rather than being treated as a role
itself. Example: User has Roles (Customer, Applicant), Permissions (VIEW_APPLICATION, SUBMIT_TRANSACTION), and
KYC Status (NOT_VERIFIED / VERIFIED). An application requiring KYC defines Permission + KYC Verified rather
than creating roles such as KYC_VERIFIED_CUSTOMER / KYC_VERIFIED_APPLICANT / KYC_VERIFIED_ADMIN. This prevents
role proliferation.

## 20. High-Level Functional Requirements

Account Registration: FR-01 allow users to initiate self-service registration. FR-02 support email or mobile
number as the username. FR-03 verify ownership of the supplied email/mobile before creating the account. FR-04
prevent activation of an account without successful verification. FR-05 securely store user credentials. FR-06
maintain registration and verification audit logs.

Authentication and KYC: FR-07 authenticate registered users using their verified username and password. FR-08
retrieve the user's KYC status after successful authentication. FR-09 restrict KYC-protected
applications/features to users with the required KYC status. FR-10 allow non-KYC-verified users to access
functions that do not require KYC. FR-11 allow users to initiate or continue KYC onboarding. FR-12 allow users
with REDO_REQUIRED status to restart KYC onboarding using the same KYC module.

KYC: FR-13 support ID capture. FR-14 support OCR. FR-15 support liveness detection. FR-16 support selfie
capture. FR-17 perform biometric 1:N matching where required. FR-18 invoke PhilSys verification for supported
PNID workflows. FR-19 capture additional KYC information. FR-20 generate a KYC decision based on configurable
verification rules. FR-21 create a separate KYC onboarding attempt for each new onboarding submission.

Exception and Review Management: FR-22 identify configurable KYC exceptions and conflicts. FR-23 create review
cases for exceptions requiring non-real-time human review. FR-24 provide an authorized reviewer queue. FR-25
reviewer can review the evidence associated with an attempt. FR-26 reviewer can approve, request KYC redo, or
reject an onboarding attempt. FR-27 notify the user when a KYC redo is required. FR-28 user can redo KYC
onboarding using the same KYC module. FR-29 user can retake the ID and selfie or provide another supported ID
during a redo. FR-30 retain previous KYC onboarding attempts for audit and traceability. FR-31 do not
automatically delete, overwrite, or merge previous KYC onboarding data when a new attempt is created. FR-32 all
reviewer decisions and manual actions shall be auditable. FR-33 prevent unauthorized users from accessing KYC
review functions.

## 21. Non-Functional Requirements

Security: passwords securely hashed; OTPs expiring and single-use; rate-limited registration, OTP, and login
endpoints; KYC and biometric data encrypted in transit and at rest; biometric templates/references not exposed
unnecessarily; administrative KYC actions appropriately authorized; session management per organizational
standards; protection against account enumeration during registration and recovery.

Auditability — the system should record: registration attempts; contact verification; login events; KYC
initiation; KYC onboarding attempt number/version; ID capture; OCR result; external verification result;
liveness result; biometric match result; KYC decision; exception creation; reviewer decision; KYC redo request;
user notification; manual override/rejection; account status changes.

Configurability — preferably configurable: supported ID types; OTP expiry and retry limits; KYC verification
rules; KYC-required applications/features; biometric matching thresholds; conflict types; reviewer permissions;
review SLA; KYC retry limits; required additional information; KYC redo reasons and user instructions;
notification channels and templates.

## 22. Recommended Overall Design Principle

Treat this as three related but distinct capabilities: Account Management ("Create and authenticate an
account."), Identity/KYC Management ("Establish and verify the real-world identity behind the account."), and
RBAC Authorization ("Determine what the verified user is allowed to access."). Overall: Verified Contact →
Active Account → KYC Onboarding → Automated Verification → Exception Review if Needed → KYC Verified →
Roles/Permissions → Application Access. For exceptions: Exception Detected → Non-Real-Time Review → Approve /
Request KYC Redo / Reject. For redo: Notify User → User Logs In → Same KYC Module → New KYC Attempt →
Reprocess. Previous attempts remain preserved for audit and traceability, while the latest successful attempt
becomes the user's current verified KYC record.
