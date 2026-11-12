# FAMILY SAFETY PARENT — Owner, Public Access and WhatsApp Support Specification

**Status: Planned; not implemented.** The repository currently has planning documentation only—no Android application, backend, database, or tests. This document turns the requested owner/public access behavior into an implementation-ready security specification. It must not be represented in the app as a working system until the backend and clients are implemented and the acceptance tests below pass.

## 1. Product and role model

The product name for this application is **FAMILY SAFETY PARENT**. It has one email/password entry flow followed by one common Login Code screen. The server—not the client—determines role and authorized account after validating the submitted code. The app must not expose separate Owner/Public login choices or reveal Owner identity to unauthenticated/public users.

Roles are server-side authorization attributes:

- **OWNER:** may use owner administration endpoints and the Owner Dashboard, subject to step-up authentication for sensitive actions.
- **PUBLIC_PARENT:** may use only the signed-in account's own parent features and explicitly authorized family/device resources. This role has no administrative API access.

Do not treat a numeric code as a substitute for account authorization. Every code is bound to an account and role in the database; a valid code alone must not grant access to another account. Every request is authorized server-side against the authenticated session, role, account, and resource ownership. Hiding a screen or button is not a security control.

## 2. Two-step sign-in behavior

1. The client submits email and password over HTTPS to the authentication service.
2. The server validates the account credentials and account state. The client then displays the single, common Login Code input; it does not ask the user to select a role.
3. The client submits the code to a short-lived, purpose-limited second-step challenge associated with that successful first step. Do not put codes in URLs, analytics, crash reports, or ordinary logs.
4. The backend verifies the code's salted password hash, active state, expiry/usage policy, account binding, and role. It atomically consumes or updates applicable use counters and records a security event.
5. The backend returns a narrowly scoped session/token containing the server-assigned account and role (or an opaque session whose role is resolved server-side). The app routes to the Owner or Public Parent dashboard based on this authenticated response.
6. Invalid, expired, suspended, revoked, exhausted, or unbound codes produce a generic “Invalid Login Code” response. Do not disclose which condition failed. Keep the second-step challenge short-lived and invalidate it after success, timeout, or repeated failures.

Use a common client screen and response contract for both roles. Do not cache a privileged role as trusted local state. On app resume and API calls, the server remains the source of truth; handle revoked sessions by returning to sign-in.

### Credential and code safety

- Never put active Owner Codes, login codes, passwords, signing secrets, or the support phone number in Android source, APK assets, repository history, test fixtures intended for production, telemetry, or public documentation.
- Store passwords with a modern adaptive password hash (e.g. Argon2id or a suitably configured platform-supported alternative); store login codes as per-code salted, slow password hashes with unique salts. Use a server-side pepper only if it can be safely held in a secret manager and rotated.
- The supplied initial Owner Codes and WhatsApp destination are bootstrap inputs, not source constants. During secure deployment, an operator uses a one-time protected provisioning procedure to create/verify the authorized Owner account, hash the two supplied codes, bind both records to that account and the OWNER role, and store the support destination in protected backend configuration. Do not echo secrets in command output or logs. Disable the bootstrap path after provisioning; do not silently create an Owner on first public app launch.
- The initial numeric codes have limited entropy and must not be treated as strong MFA or as a long-term shared master secret. Pair them with the authenticated account, strict online throttling, anomaly alerts and preferably a phishing-resistant factor (passkey/security key or TOTP) for Owner access and step-up actions. Consider requiring immediate replacement at first secure Owner use. Never display a code again after its one-time creation view; provide replace/revoke instead of plaintext recovery.
- Enforce rate limits by account, challenge, network/device risk signals, and global abuse controls; use increasing delays and temporary lockouts without creating an easy denial-of-service vector against Owner accounts. Alert and audit repeated failures. Do not rely on IP-only throttling.

## 3. Owner code management

Only an authenticated Owner can access the Owner Code Management API and UI. Creating, replacing, suspending, activating, deleting/revoking, changing code policy, and changing Owner access require recent re-authentication plus the configured additional verification factor. Revoke relevant sessions and notify the bound account when Owner access changes.

Owner can see a list of code records and metadata (masked identifier/fingerprint, label, bound account, role, status, created/updated/last-used timestamps, expiry, usage policy, and audit history). Because only a hash is retained, Owner cannot view existing plaintext codes. Generate new high-entropy codes using a cryptographically secure random generator; show the code only once immediately after creation and require secure handling. Replacement creates a new credential and revokes the old one; do not edit the existing plaintext in place.

Prevent removal or suspension of the last usable Owner access unless a verified recovery route exists and has been tested. The check and state change must be transactional to avoid concurrent requests disabling all Owner access. Recovery must be controlled, auditable, rate-limited, and independent of public-user code issuance. Emergency recovery should require an out-of-band operator process, not a hidden default credential.

## 4. Public Code Generator and account management

Only Owner can access the Public Code Generator and Public User Management sections. Support single and bulk generation with per-batch limits and clear one-time display of generated plaintext. Public codes are bound to an intended account and PUBLIC_PARENT role; they do not themselves create broad administrative privileges. Activation should require the account binding and whatever user confirmation the approved onboarding flow specifies.

Owner can inspect generated-code metadata, linked account, activation/last-use state, expiry, usage count/limit, and audit history. Owner can replace, activate, suspend, deactivate, or revoke codes. “Edit” means changing metadata/policy or issuing a replacement, not retrieving or mutating an already-issued secret. Define what deleting a code means (revoke credential and retain minimal audit evidence); do not delete records in a way that erases required security audit history.

Public Parent accounts must not be able to issue any Owner or Public codes, manage other accounts, access owner settings, or call administrative APIs. All list/detail endpoints must scope results server-side to the authorized Owner or the public account's own resources. Bulk endpoints need maximum batch sizes and per-owner quotas.

## 5. Owner Dashboard and Public Parent Dashboard

Owner Dashboard sections requested:

A. Owner Code Management  
B. Public Code Generator  
C. Public User Management  
D. WhatsApp Support Settings  
E. Owner Security Settings  
F. Login History  
G. Administrative Activity Logs  
H. App Configuration

Every section is unavailable to PUBLIC_PARENT accounts at both routing and API layers. Administrative actions and data views are recorded in an append-only or tamper-evident audit stream with actor, action, target, time, result, and safe metadata—never plaintext credentials or sensitive WhatsApp message content.

Public Parent users see only their own parent dashboard and authorized family features, plus the optional WhatsApp Support button when support is enabled. No owner name/identity, code-management entry point, administrative settings, phone number text, or administrative response data is shown.

## 6. WhatsApp Support

The support destination is server-managed configuration, seeded with the supplied initial number during secure backend provisioning. Do not copy the destination into source code or commit it to repository documentation. Store it encrypted at rest (or in a protected configuration service); restrict reads and writes to the support-link service and Owner configuration API. Validate normalized international digits and reject malformed values before saving. Only an authenticated Owner can change the number or toggle support ON/OFF; require recent re-authentication, audit the change, and avoid returning the number to public APIs.

When enabled, the Public Parent UI renders one button labeled **WhatsApp Support**—not the number. Tapping it requests a short-lived/current support link from the backend, then opens WhatsApp through a supported Android intent or HTTPS `wa.me` link. The optional prefilled message must be generic, contain no family/device/location/account data, and be user-editable before sending. If WhatsApp is unavailable, show a clear fallback state without exposing the support number in app UI.

Owner Settings has a **Test WhatsApp Support** button that uses the same server-configured destination and opens a test chat/message. The test must not send a message automatically. Updating the destination changes subsequently generated links; no app update is needed.

**Privacy limitation:** the phone number is not displayed as text in the Public UI, but WhatsApp's click-to-chat URL/intent must identify a recipient. The destination may therefore be visible to the device user, Android, WhatsApp, and relevant link-handling infrastructure when the button is tapped. Do not claim that it is cryptographically hidden from the user or WhatsApp. Do not embed it in analytics or logs.

## 7. Backend authorization and suggested API boundaries

Use authenticated server-side sessions and explicit authorization checks on every route. Suggested versioned routes (names illustrative; define in OpenAPI):

- `POST /v1/auth/email` — first-step authentication; returns a short-lived challenge, not a privileged session.
- `POST /v1/auth/code` — validates challenge/code and issues the role-bound session.
- `POST /v1/auth/step-up` — recent password plus strong second factor for sensitive actions.
- `POST/GET/PATCH/DELETE /v1/owner/codes` — Owner-only code lifecycle; creation response reveals plaintext once only.
- `POST/GET/PATCH/DELETE /v1/owner/public-codes` — Owner-only public-code generation and lifecycle.
- `GET /v1/owner/users` and scoped account management routes — Owner-only; never allow Public Parent access.
- `GET/PATCH /v1/owner/settings/whatsapp` — Owner-only setting API; GET should not return the raw number unless needed in the Owner editor and explicitly authorized.
- `POST /v1/support/whatsapp-link` — authenticated PUBLIC_PARENT access to a minimal link response only while support is enabled; never returns administrative configuration beyond the link action.
- `GET /v1/owner/login-history`, `/v1/owner/audit-events`, `/v1/owner/config` — Owner-only, paginated and audited.
- Public parent routes must always derive account/family scope from the verified session, never trust an account ID supplied by the client as sufficient authorization.

Use CSRF protection if cookie-based sessions are used, secure token rotation and revocation, strict CORS, request validation, content security controls where applicable, TLS, secret management, and API versioning. Avoid long-lived bearer tokens in local storage; use platform-protected secure storage and short lifetimes. Keep error responses generic enough not to disclose code existence or account enumeration.

## 8. Suggested database records

- `accounts`: normalized unique email, password hash, state, created/updated timestamps.
- `account_roles` or role field with constrained server-side values and an explicit account relationship.
- `login_codes`: code ID, account ID, role, salted hash, status, label, expiry, usage limit/count, created/updated/last-used/revoked timestamps; never plaintext.
- `auth_challenges`: account/challenge binding, purpose, expiry, attempt count, consumed timestamp, risk metadata minimized.
- `sessions`: account ID, server-resolved role/authorization version, issued/expiry/revocation metadata, refresh-token hash.
- `whatsapp_support_config`: encrypted destination, enabled flag, generic prefill template/version, editor account, updated timestamp.
- `admin_audit_events`: actor, action, target, result, timestamp, request correlation ID, redacted metadata; append-only access controls.
- `login_events`: success/failure category, account/challenge reference where safe, timestamp, risk signals subject to retention limits; never log raw codes/passwords.

Apply unique/foreign-key constraints and transactional code-use counters, restrict database access by service role, encrypt backups, and define retention/purge schedules for authentication events and audit metadata.

## 9. Acceptance tests and release gates

### Authentication and roles

- Valid email/password reaches the same common Login Code screen for both roles; no role choice or public Owner identity is shown.
- Valid active Owner-bound code issues Owner authorization and routes to Owner Dashboard; valid Public-bound code issues only Public Parent authorization.
- Invalid, expired, revoked, suspended, wrong-account, exhausted, and replayed codes fail generically; challenge reuse fails.
- Repeated attempts trigger configured throttling and security events without leaking whether an account/code exists.
- Client role tampering, altered request bodies, direct deep links, and calls to every Owner endpoint as PUBLIC_PARENT are denied server-side (403/404 according to the documented disclosure policy).
- A revoked code/session loses access promptly; tests cover token refresh and concurrent revocation.

### Owner code and recovery controls

- Public Parent cannot list, generate, edit, activate, suspend, revoke, or inspect any code or Owner setting, even with handcrafted HTTP requests.
- Owner code mutations fail without recent re-authentication and additional verification.
- Plaintext appears only once at generation, never in database, logs, analytics, crash reports, exports, or subsequent list responses.
- Last-active-Owner protection is race-safe; it cannot be bypassed by parallel requests. Recovery is verified and tested before the final active route can be disabled.
- Bulk generation enforces limits, account binding, expiry/usage rules, and audit logging.

### WhatsApp

- Disabled support yields no public button/link; enabled support renders only the button label in the Public UI.
- Public APIs and UI do not expose the support number as a displayed field; button tap opens the correctly configured WhatsApp recipient, and an unavailable-app fallback works.
- Owner can update/disable/enable the destination only after step-up; successful updates affect future links and are audited. Test button opens but does not send.
- Prefilled content contains no personal/family/device data; verify logs and analytics contain neither raw destination nor message content.
- Document the intentional recipient visibility to WhatsApp and the user after tapping.

### Delivery status

Current status for all features in this document is **Unavailable / Planned**. Mark a feature **Partially implemented** only when its remaining gaps are stated in the app/docs; mark **Implemented** only after backend and Android end-to-end behavior, automated negative authorization tests, security review, and release documentation pass.

## 10. Implementation sequence

1. Confirm the authorized Owner account, recovery process, target distribution path, and whether Owner access requires a passkey/TOTP in addition to the requested email/password and code.
2. Implement backend identity, challenge lifecycle, secure code storage, role enforcement, audit, rate limits, and automated authorization tests before building owner UI.
3. Provision the initial Owner account and two Owner-bound codes through a protected one-time operator process; provision WhatsApp configuration the same way. Do not commit supplied secret values.
4. Implement the shared email/code sign-in flow and role-based navigation in the Parent Android app.
5. Build Owner Code Management and Public Code Generator against real backend endpoints; verify one-time secret display and step-up requirements.
6. Implement Public Parent account scoping and verify no administrative API/data exposure.
7. Implement protected WhatsApp settings, public button/link, Owner test action, and privacy disclosure.
8. Run the acceptance suite, security review, and real-device tests; update the project completion checklist before calling any capability available.

## 11. Decisions still required

- Define account creation/verification and how a Public Parent account becomes bound to a generated code; a login code must not be an invitation to impersonate an arbitrary email account.
- Specify whether public codes are one-time activation codes or reusable login codes. Prefer one-time activation followed by normal secure account authentication; reusable shared codes are weaker and harder to revoke safely.
- Confirm recovery contacts/procedures and the additional Owner factor. The supplied short numeric codes alone are not suitable as a durable high-assurance Owner factor.
- Set code expiry, usage limits, session duration, lockout/recovery policy, audit retention, and maximum bulk-generation size.
- Decide whether WhatsApp support is available to every signed-in Public Parent or only eligible accounts, and approve the generic prefilled message.
