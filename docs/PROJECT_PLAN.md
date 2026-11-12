# Family Safety — Project Plan

**Status:** Planning baseline (not an implementation specification for the unavailable 42-category list)  
**Repository baseline:** `vina192326/FAMILY-SAFETY` was inspected on 2026-11-12. GitHub reports an empty repository (`size: 0`) with no source tree or documentation to assess. The prompt mentions 42 existing feature categories but does not include their names or descriptions. Consequently, this plan covers the requirements actually supplied; mapping, deduplicating, and prioritizing the 42 categories is a Phase 0 deliverable once that list is provided.

## 1. Product outcome and principles

Build two native Android applications—**Family Safety Parent** and **Family Safety Kids**—backed by a secure service. A family can explicitly link and unlink devices, set age-appropriate safety rules, and use clearly disclosed location, screen-time, alerts, and assistance features. A child-facing app must make active sharing visible and provide understandable controls and explanations. This is a family-safety product, not covert surveillance or a means to bypass Android controls.

Product principles:

- **Consent and visibility:** clear pairing approval, visible sharing/assistance state, purpose-limited collection, access history, and easy unlinking/deletion.
- **Platform honesty:** no promises of silent screen capture, arbitrary remote control, guaranteed background location, or bypassing OEM battery restrictions. Implement only documented Android APIs and permitted distribution paths.
- **Safety by default:** collect the minimum data, keep sensitive data for the shortest useful period, encrypt it, and restrict parent access to linked family members.
- **Modular growth:** feature modules depend on stable domain contracts, not on each other's UI or database implementation. Features can be enabled by capability and policy rather than rebuilding the whole system.
- **Deliver working slices:** every phase has testable acceptance criteria and an explicit implemented/partial/unavailable status. No UI control is presented as functional before its end-to-end behavior exists.
- **Affordable, not “free forever”:** prefer open standards and self-hostable components; show operational costs and quotas before enabling metered services.

## 2. Scope and requirement discovery

### Supplied core scope

Two Android apps; secure backend; multiple parents and child devices; durable authorized pairing; family dashboard; optional location sharing; device and screen-time management; transparent remote assistance; emergency alerts; AI assistance; data export/share/delete; Hindi and English; light/dark themes; offline behavior; extensible APIs; automated tests; technical documentation; and no mandatory subscription.

### Phase 0 discovery (required before feature implementation)

1. Obtain the actual 42-category list, including expected behavior and any existing designs/code. Tag each item as core, optional, deferred, restricted by Android, or unsupported.
2. Define target ages, guardian/child roles, supported countries, data residency expectations, retention periods, and emergency-response disclaimers.
3. Decide minimum Android SDK / target SDK based on current Play policy and device coverage; explicitly test Motorola and Xiaomi models and Android versions.
4. Agree on service availability, recovery and deletion objectives, expected families/devices, and initial monthly hosting budget.
5. Produce data-flow diagrams, threat model, permission inventory, and privacy notices before collecting sensitive data.

**Open dependency:** Do not claim that these are the “42 existing features” until the source list is supplied and reconciled.

## 3. Proposed architecture

### Android clients

- Kotlin, Jetpack Compose, Material 3, AndroidX, coroutines/Flow, Navigation Compose, and a modular Gradle build (`core`, `feature-*`, `parentApp`, `kidsApp`). Share models and domain logic only where doing so does not couple the two apps' permissions or user experiences.
- MVVM / unidirectional state flow with repositories and use cases; Hilt or another single, documented DI approach; Room for local structured/offline data; DataStore for non-sensitive preferences; WorkManager for deferrable sync. Use Android Keystore-backed encryption for local secrets.
- Separate app IDs, signing, release tracks, permission sets, and role-specific navigation. Keep location, usage access, notifications, and screen-capture permissions feature-gated and requested in context, with a reason and a graceful off state.
- Use platform APIs such as `UsageStatsManager` only after user-granted special access; location via Android location APIs with runtime permission and foreground-service rules; `MediaProjection` only for user-started, system-consented screen sharing. Verify current Android/Play policies at implementation time.

### Backend and API

Recommended initial shape: a **modular monolith** with stateless HTTPS API, PostgreSQL, a background job queue, and push notification adapters. This is cheaper and easier to operate than starting with microservices; extract services only when scale or isolation justifies it.

- Versioned REST/JSON API described with OpenAPI; short-lived access tokens and rotating refresh credentials; MFA/passkeys as an incremental account-security improvement; rate limits and abuse detection.
- PostgreSQL as the source of truth for accounts, family membership, device links, permissions/consent records, policy versions, alerts, and audit events. Enforce tenant/family boundaries in application authorization and database constraints; consider PostgreSQL row-level security as defense in depth.
- Store high-volume location observations separately from core relational records with a strict retention policy; begin with partitioned PostgreSQL tables and indexes. Add a time-series extension or separate store only after measured need. Never retain a precise location history by default without a clear, configurable purpose and retention window.
- Use object storage only for explicit exports or user-provided attachments, with short-lived signed URLs, encryption, quotas, and lifecycle deletion. Avoid storing screen recordings by default.
- Push (e.g. Firebase Cloud Messaging) is a best-effort wake/alert channel, not a guaranteed real-time transport. Include in-app sync and visible “last updated” status.
- AI is an optional adapter behind a provider interface. Start with deterministic, local or server-side rule-based assistance; do not send children’s location, messages, or identifiers to a model by default. Require explicit, informed opt-in before any external model processing and disclose provider, data handling, and cost.

### Security and privacy controls

Threat-model account takeover, malicious/overreaching guardians, lost devices, token theft, cross-family access, insecure exports, replayed pairing, backend compromise, and abusive automation. Use TLS, encryption at rest, least privilege, secret management, key rotation, audit trails for sensitive access, dependency scanning, secure backups, and tested restore/deletion procedures. Do not promise end-to-end encryption until key recovery, multi-parent access, export, and revocation semantics are designed and reviewed.

Pair devices with an expiring single-use invitation/QR or code, authenticated parent approval, child-device confirmation, server-side authorization, and revocation. A “permanent” link means persistent until revoked—not immutable credentials or permission to operate invisibly. Re-authenticate for sensitive actions and notify relevant device users of new links, changed rules, exports, and assistance sessions.

### Offline and synchronization

Room-backed read access and queued low-risk settings changes where safe; explicit conflict/version handling; idempotency keys for retried writes; server timestamps; clear stale/offline indicators. Never queue a remote action that could unexpectedly execute later (such as starting screen share or changing a safety-critical setting) without fresh confirmation.

## 4. Data model starting point

Core relational entities: `accounts`, `families`, `family_memberships` (role/status), `devices`, `device_links` (state, scopes, created/revoked timestamps), `consents` (purpose, policy version, actor, time), `safety_policies` (versioned), `location_samples` (minimal fields, retention/partition key), `screen_time_summaries`, `alerts`, `assistance_sessions`, `audit_events`, and `export_jobs`.

Use UUID identifiers, UTC timestamps, explicit ownership/family foreign keys, uniqueness constraints, soft deletion only where needed for revocation/audit, and documented retention/purge jobs. Keep authentication secrets out of application tables where possible. Define a deletion workflow covering primary records, derived data, queues, object storage, backups (with documented backup expiry), and third-party processors. Keep only the minimal non-identifying audit record required for security or legal obligations, with a documented basis and retention.

## 5. At least 20 additional feature opportunities

Estimates are rough planning ranges, not quotes. Android compatibility is conditional on API availability, user grants, OEM behavior, and current Google Play policy. Backend cost is incremental monthly USD at a small pilot (roughly hundreds of families, low traffic), excluding engineering, tax, and support. Costs can be $0 on a local/self-hosted pilot, but production always has hosting, backup, monitoring, and labor costs; free tiers and prices change. “Low” is roughly 1–3 engineer-weeks; “Medium” 3–8; “High” 2+ engineer-months or substantial policy/security work.

| Additional feature | Description and benefit | Technical requirements | Android compatibility / limitations | Backend needs | Difficulty | Possible incremental operating cost |
|---|---|---|---|---|---|---|
| 1. Consent and permission center | Per-feature explanation, state, revocation, and last access; improves trust and reduces surprise. | Permission inventory, consent versioning, settings UI, audit events. | Android permission controls remain OS-owned; some special access requires Settings screens. | Store consent and audit state; no extra service. | Medium | $0–$20/month storage/compute at pilot. |
| 2. Pairing safety and recovery | Expiring QR/code, device confirmation, recovery flow, and link revocation; prevents accidental or stale links. | Cryptographically random one-time tokens, expiry, replay protection, recovery UX. | All supported Android versions; QR scanning may use camera permission or manual code. | Pairing endpoints, rate limits, audit. | Medium | $0–$20/month. |
| 3. Trusted contacts and check-in | Child can send a scheduled “I’m okay” check-in or alert selected trusted adults. | Contact verification, notification UX, rate limits. | Notifications are user-controllable and not guaranteed. | Contact/membership ACLs, notification dispatch. | Medium | $0–$25/month plus possible SMS charges (avoid SMS by default). |
| 4. Safe-arrival geofences | Notify on arrival/departure from family-defined places, with configurable boundaries. | Geofencing/location permission, battery-aware sampling, deduplication. | Background location restrictions and OEM battery management can delay events; avoid promising exactness. | Place metadata, event processing, retention controls. | High | $0–$50/month at pilot; mapping/geocoding may be metered. |
| 5. Location freshness and sharing schedule | Choose when location is shared and show accuracy, age, and pause state. | Location foreground service only where justified, scheduling, accuracy display. | Background access requires appropriate permission and policy justification; user can revoke. | Store minimal samples and schedule state. | Medium | $0–$50/month; map tiles/provider usage may cost extra. |
| 6. Battery and connectivity health | Surface last sync, battery level (where available), and connection quality so families distinguish offline from unsafe. | Android battery/network APIs, sync telemetry. | Values are approximate; background reporting may be delayed. | Small device-status records and push. | Low | $0–$15/month. |
| 7. App update and security hygiene reminders | Show app version, update availability, and security guidance. | Package/version metadata and release feed. | No silent app updates; Play or user controls installation. | Optional signed release metadata. | Low | $0–$10/month. |
| 8. Family routines and schedules | Shared routines (school, bedtime, check-in) and reminders, avoiding intrusive monitoring. | Time-zone-aware rules, local notifications, rule editor. | Notifications can be disabled or delayed. | Sync routine definitions, minimal event status. | Medium | $0–$20/month. |
| 9. Screen-time coaching insights | Explain trends and encourage agreed breaks instead of punitive scoring. | Usage access, on-device aggregation, accessible charts. | Usage access is user-granted special access; counts vary by Android/OEM. | Optional summaries only; retention limits. | Medium | $0–$20/month. |
| 10. Focus / bedtime modes | Family-agreed reminders and schedules, with child-visible rationale and override/emergency path. | Scheduling, notification/channel design, optional supported OS controls. | Do not claim the app can force-stop arbitrary apps; stronger controls require supported/managed-device APIs and policy review. | Versioned policy and event logs. | High | $0–$25/month. |
| 11. Emergency escalation ladder | Alert family members in order and track acknowledgement; clarify that this is not emergency dispatch. | Alert state machine, acknowledgement, deduplication. | Push/Internet is not guaranteed; provide clear fallback instructions. | Queue, push, audit, retry and expiry policy. | Medium | $0–$30/month; SMS/voice escalation may be metered and opt-in. |
| 12. Check-in timer and missed-check-in workflow | User-initiated timer with escalating reminders if not acknowledged. | Local timer, server sync, safe cancellation UX. | Background work is best effort; never infer an emergency solely from a missed check-in. | Scheduled jobs and notifications. | Medium | $0–$25/month. |
| 13. Scam and suspicious-link education | User-submitted link check plus age-appropriate safety tips; avoid covert message scanning. | Safe URL parsing, vetted reputation provider or local education content. | Do not inspect private messages or clipboard in the background. | Optional reputation API; data minimization. | Medium | $0–$100+/month depending on provider/volume; begin with no-cost educational content. |
| 14. Family digital-safety learning | Short Hindi/English lessons, conversation prompts, and quizzes. | Localized content, accessible UI, offline cache. | Broad compatibility; no sensitive permission needed. | CMS or versioned static content. | Low | $0–$20/month. |
| 15. Privacy-preserving AI safety guide | Explain settings and provide general safety advice without receiving location or message content. | Provider abstraction, safety filters, opt-in, data minimization, evaluation. | Network/model service needed for cloud AI; disclose data flow and age restrictions. | AI gateway, quotas, redaction, audit and spend cap. | High | $0–$200+/month depending on model and usage; provider spend can rise quickly. |
| 16. Data access/export and deletion center | Download a readable archive, inspect retention, request deletion, and track completion. | Export job, secure archive, re-authentication, deletion state machine. | Local download/share must use Android scoped storage and user-selected destination. | Background jobs, expiring object storage, purge verification. | High | $0–$50/month at pilot, plus storage/egress. |
| 17. Account/session security dashboard | Review active sessions, revoke lost devices, view security events, enable stronger authentication. | Secure token rotation, session inventory, re-authentication. | Passkey/biometric APIs depend on OS and credential provider; biometrics protect local unlock, not server identity alone. | Session and audit records; email delivery optional. | Medium | $0–$30/month; email provider may have paid tiers. |
| 18. Parent role and approval controls | Multiple guardians with scoped roles and approval for high-impact changes. | Role-based authorization, dual-control policy, conflict handling. | Works across app versions if API capabilities are versioned. | Membership roles, authorization tests, audit. | High | $0–$25/month. |
| 19. Accessibility and low-bandwidth mode | TalkBack-friendly workflows, scalable text, reduced media, and resilient sync for more families. | Compose semantics, localization QA, cache and payload optimization. | Device accessibility services remain user-controlled. | Smaller API payloads; optional content CDN. | Medium | $0–$20/month. |
| 20. Device migration and replacement | Transfer a child profile to a replacement phone with clear old-device revocation. | Secure re-pairing, data migration, session invalidation. | Android backup/device migration behavior varies; app should not rely on unrestricted device identifiers. | Pairing, revocation, migration audit. | Medium | $0–$20/month. |
| 21. Family data retention controls | Configure short location retention and optional category-level data deletion. | Retention UI, policy engine, scheduled purge, metrics. | On-device and OS-managed copies have separate controls. | Partition-aware purge and backup expiry documentation. | High | $0–$30/month; may lower storage costs. |
| 22. Transparent remote-assistance session | Child starts or approves a session; persistent visible indicator, time limit, stop control, and no hidden capture. | `MediaProjection`, foreground service, WebRTC or equivalent, explicit session lifecycle. | Android requires system-mediated capture consent; no silent start or arbitrary remote input. OS/API/Play restrictions apply and must be reviewed. | Short-lived signaling, TURN relay when needed, no recording by default. | High | $0–$100+/month; bandwidth/TURN can dominate and must be opt-in/costed. |
| 23. Safety plan and trusted-place notes | Family-shared instructions for meeting points, contacts, and travel plans. | Encrypted local cache, access control, expiry. | Broad compatibility; avoid exposing information on lock screen by default. | Small encrypted records, audit. | Low | $0–$15/month. |
| 24. Product status and capability catalog | Show which features are working on this device, unavailable, or require permission/API level. | Capability checks, feature flags, app/backend version negotiation. | Capability depends on OS, manufacturer, and permission state. | Versioned feature catalog/configuration. | Medium | $0–$15/month. |

## 6. Phased delivery plan and exit gates

Phases are sequenced by dependency, not calendar promise. Each phase ends with code review, automated tests, a short release note, and an updated feature-status matrix.

### Phase 0 — Requirements and architecture
**Deliver:** reconcile the 42 categories when supplied; user roles and abuse cases; Android/device support matrix; permission and policy review; data-flow diagrams; threat model; API outline; architecture decision records; cost model.  
**Exit:** owners approve scope, sensitive data purposes/retention, Android distribution path, and explicit unavailable/unsupported behaviors.

### Phase 1 — Repository foundation and engineering quality
**Deliver:** Kotlin multi-module skeleton for two apps and shared modules; CI build/lint/unit tests; dependency and secret scanning; conventions; localization/theme/accessibility foundations; documentation and completion checklist.  
**Exit:** clean reproducible build, test pipeline, no placeholder controls represented as complete.

### Phase 2 — Secure backend and data model
**Deliver:** modular monolith, PostgreSQL migrations, OpenAPI, authentication/session lifecycle, family and membership authorization, audit, rate limiting, backup/restore and data deletion design.  
**Exit:** authorization and tenant-isolation tests; security review; migration/restore tested; privacy/retention model documented.

### Phase 3 — Parent app foundations
**Deliver:** sign-in, family creation, dashboard shell, device status, Hindi/English, light/dark, accessible states, offline/read-only indicators.  
**Exit:** functional end-to-end account and family workflows; no sensitive data shown without authorization.

### Phase 4 — Kids app foundations
**Deliver:** child-facing onboarding, plain-language privacy/permission explanations, visible linked-family status, help and unlink/report pathway, Hindi/English and accessibility.  
**Exit:** child can understand active sharing, see status, and complete supported revocation/help steps; test with non-developer users.

### Phase 5 — Authorized device linking
**Deliver:** expiring invitation, parent/child confirmation, multi-parent/multi-device membership, device naming, re-authentication, revoke/unlink, audit notifications.  
**Exit:** replay, expired-token, account recovery, cross-family and revoked-device tests pass.

### Phase 6 — Location and device status
**Deliver:** opt-in location sharing, freshness/accuracy and pause controls, battery/network status, offline-safe sync, bounded retention. Add geofences only after permission and battery review.  
**Exit:** permission denied/revoked, background restriction, stale data, and OEM battery cases are clearly represented; location deletion is verified.

### Phase 7 — Screen-time and family rules
**Deliver:** usage summaries after explicit special access, agreed schedules/reminders, transparent limitations, coaching-oriented reports.  
**Exit:** no claim of arbitrary app blocking unless a separately approved and policy-compliant device-management design supports it; test with access revoked and on target OEMs.

### Phase 8 — Remote assistance
**Deliver:** user-initiated/approved screen-sharing proof of concept via supported APIs, persistent visible state, explicit stop/timeout, no recording by default, cost/bandwidth control.  
**Exit:** consent and OS capture dialogs are mandatory; session cannot be hidden or silently restarted; privacy and Play policy review completed. Remote input/control remains unavailable unless independently verified as permitted and supported.

### Phase 9 — Alerts, reports, export and deletion
**Deliver:** emergency-style family alerts with non-emergency disclaimer, acknowledgement, export job, secure share/download, deletion workflow and retention reporting.  
**Exit:** delivery limitations are visible; signed links expire; deletion covers primary data and documents backup expiry; export authorization tests pass.

### Phase 10 — AI and automation
**Deliver:** start with deterministic safety guidance and optional, isolated provider integration; user-facing disclosure, consent, redaction, safety evaluation, quotas and spend caps.  
**Exit:** no child data is sent to external AI by default; prompt/data handling reviewed; test harmful/incorrect advice and gracefully disable unavailable providers.

### Phase 11 — Security, privacy and real-device validation
**Deliver:** threat-model review, dependency and API security tests, permission abuse tests, privacy review, crash/performance checks, Motorola and Xiaomi test matrix across supported Android versions, backup/restore and deletion drills.  
**Exit:** critical/high findings resolved or explicitly accepted with mitigation; OEM limitations documented; no hidden collection or unsupported permission path.

### Phase 12 — APKs and release readiness
**Deliver:** signed debug/internal and release-candidate APKs (and AAB if using Play), reproducible build instructions, signing-key custody plan, release notes, privacy disclosures, support/rollback plan.  
**Exit:** install/upgrade/uninstall tests, Play policy check if applicable, crash-free pilot criteria, signing and secrets not committed. APK generation is a build deliverable only after source, CI, and signing are established.

### Phase 13 — Operate and expand
**Deliver:** service SLOs, monitoring without sensitive payload logging, incident response, cost dashboards/alerts, migration policy, feature flags, support process, quarterly privacy/permission review.  
**Exit:** operational ownership, backup recovery, deletion SLAs, and feature completion statuses are maintained.

## 7. Testing and release strategy

- **Unit:** domain rules, policy evaluation, schedule/time-zone edge cases, retention, authorization helpers.
- **Android:** Compose UI tests, repository/sync tests, permission-granted/denied/revoked paths, Room migration tests, accessibility checks, localization, dark/light themes.
- **Backend:** API contract tests, database migration tests, authorization matrix, family isolation, rate limits, idempotency, deletion and export tests.
- **Integration:** pairing, push as best-effort, offline/reconnect, stale data, account/session revocation, export expiry.
- **Security/privacy:** dependency and secret scans, static analysis, abuse-case review, log/analytics redaction, backup access, permission scope and consent audits. Arrange an independent security assessment before broad launch.
- **Devices:** emulator coverage plus physical Motorola and Xiaomi devices; record model, Android version, permission state, battery-optimization setting, behavior, and known limitations.
- **Release gates:** no critical/high security defect, no false feature-complete state, tested rollback/restore, documented cost and privacy notices, and product approval for each sensitive capability.

## 8. Cost-aware implementation options

Start with a small single-region deployment of an open-source backend and managed PostgreSQL only if its operational simplicity is worth the bill. A self-hostable stack (for example, Kotlin/Ktor or a well-maintained JVM framework, PostgreSQL, and a small job worker) avoids early vendor lock-in but still requires patching, backups, monitoring, and on-call responsibility. Alternatives such as a hosted backend can accelerate a pilot but require careful review of free-tier limits, data residency, vendor lock-in, and paid usage.

Cost controls: no continuous high-frequency GPS by default; on-device aggregation for screen-time; bounded retention and partition purges; compact sync payloads; push instead of paid SMS; no default media recording; quotas on exports and AI; per-family rate limits; budget alerts; load-test before scaling; and cost estimates shown before opt-in to metered maps, TURN, email/SMS, storage egress, or AI. Revisit provider prices and terms at procurement time—free tiers are not a product guarantee.

## 9. Feature completion checklist (baseline)

Use these labels in the issue tracker, release notes, and in-app capability view:

- **Implemented:** complete end-to-end behavior, required tests, docs, privacy disclosure, and device validation are present.
- **Partially implemented:** specify exact supported paths and known gaps; disable or label unsupported actions.
- **Unavailable:** not built, permission/API/policy unavailable, or intentionally excluded; never display as working.

| Capability | Initial status | Completion evidence required |
|---|---|---|
| Parent Android application | Unavailable (empty repository) | Release build and end-to-end tests |
| Kids Android application | Unavailable (empty repository) | Release build and child-facing privacy tests |
| Secure backend / authentication | Unavailable | API, threat model, auth and isolation tests |
| Authorized multi-device pairing | Unavailable | Expiry, approval, revocation and replay tests |
| Location sharing | Unavailable | Permission, freshness, retention, deletion and OEM tests |
| Device and screen-time monitoring | Unavailable | Permission transparency and documented API limitations |
| Remote assistance | Unavailable | Explicit OS consent, visible session, stop/timeout tests |
| Alerts / offline sync | Unavailable | Best-effort semantics and offline/retry tests |
| AI assistance | Unavailable | Opt-in, data-flow disclosure, safety and cost controls |
| Export / sharing / deletion | Unavailable | Auth, expiring download, purge and backup-expiry tests |
| Hindi/English, themes, accessibility | Unavailable | Localization and UI/accessibility test coverage |
| Documentation / automated tests | Unavailable | CI and maintained developer/user docs |

## 10. Immediate next actions

1. Treat this document as the planning baseline and create project issues from the phase exit gates.
2. Provide the 42 feature-category names/descriptions so they can be mapped to this plan without inventing requirements.
3. Confirm intended app distribution (Play Store, private distribution, or both), target age groups, initial supported Android versions/devices, expected pilot scale, and monthly budget ceiling.
4. Review the Phase 0 privacy, permission, and abuse-case decisions before enabling location, usage access, or screen sharing.
5. Begin with a tested foundation and one complete, low-risk vertical slice—account, family membership, and transparent device pairing—rather than a mock dashboard.
