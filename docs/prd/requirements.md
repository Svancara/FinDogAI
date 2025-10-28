# Requirements
### Compliance Requirements (GDPR)

1. **Data export/backup** - GDPR compliance requires data portability


### Functional Requirements

**FR1:** The system shall provide a "Set Active Job" voice flow that accepts natural language input including numeric job IDs (e.g., "Set active job to 123" or "Set active job to Smith, Brno"), confirms via TTS, and persists the active job context to auto-assign subsequent costs by default. Subsequent commands apply to the active job unless an inline job override is specified (e.g., "… for job 123"), in which case the operation targets that job without changing the active job.

**FR2:** The system shall provide a "Start Journey" voice flow that accepts natural language input with destination, vehicle, and odometer reading (e.g., "I'm going to Brno in Transporter, odometer 12345"), confirms via TTS, and creates a journey event under the active job.

**FR3:** The system shall provide manual entry screens for creating, editing, and deleting costs across five categories: Transport, Material, Labor, Machine, and Other, with support for referencing items by sequential ID in voice commands (e.g., "Update material 45"). Manual forms default the targeted job to the current Active Job but allow changing the job before save.

**FR4:** The system shall provide CRUD operations for Jobs with fields: auto-assigned jobNumber (server-allocated sequence), title, description, status (active/completed/archived), currency (ISO 4217), VAT rate, and budget.

**FR5:** The system shall provide a Business Profile setup for one-time configuration of default currency, VAT rate, and distance unit (km/miles).

**FR6:** The system shall provide Resources Management for Team Members (minimum one: the user), Vehicles, and Machines, each with properties (name, hourly rates for labor/machines, per-distance rates for vehicles).

**FR7:** The system shall maintain basic audit metadata on all database entities: createdAt timestamp, createdBy compound identity object {uid, memberNumber, displayName}, updatedAt timestamp, and updatedBy compound identity object {uid, memberNumber, displayName}. This provides rich audit context within the tenant scope without requiring additional lookups.

**FR8:** The system shall implement comprehensive audit logging via Cloud Functions triggers (onCreate/onUpdate/onDelete) that capture full operation history to a tenant-scoped `audit_logs` subcollection under `/tenants/{tenantId}`, including: operation type, timestamp, author (compound identity object from createdBy/updatedBy), old values (for UPDATE), and complete object snapshots (for DELETE). Audit logs have owner-only UI access (representative and team member have no access), are excluded from exports, and auto-expire after 1 year for cost optimization.

**FR9:** The system shall track Advances as a per-job subcollection with auto-assigned ordinalNumber (server-allocated per-job sequence), displaying sum of advances vs sum of costs.

**FR10:** The system shall implement Firebase Authentication (email/password minimum) with Firestore Security Rules enforcing membership-based multi-tenant isolation under `/tenants/{tenantId}/**`. Access is allowed only for authenticated users that have a membership document at `/tenants/{tenantId}/members/{request.auth.uid}`; documents MUST include `tenantId` that matches the path; per-operation access is enforced via `membership.role`.

**FR11:** The system shall use role-based access control on membership documents (`/tenants/{tenantId}/members/{uid}`) with fields: `role ∈ {owner, representative, teamMember}` and `status ∈ {active, disabled}`. Role semantics:
- Owner: full access to all functions and data, including audit logs and exports.
- Owner's representative: Jobs are read-only; cannot change Business Profile; cannot change privileges; cannot view audit_logs; cannot export any data; all other functions and data are fully available (including costs and resources).
- Team member: Jobs are read-only (only active jobs visible; only `jobNumber` and `title` shown); cannot change Business Profile, privileges, or resources; cannot export any data; no access to the advances collection; all other functions and data are available (including adding/editing costs).

**FR12:** The system shall require team member authentication and identification, storing the compound identity object {uid, memberNumber, displayName} with all operations for audit trail purposes. This enables both machine-readable (uid) and human-readable (memberNumber, displayName) audit attribution.

**FR13:** The system shall enable Firestore offline persistence, ensuring all manual flows function without network connectivity, with automatic background sync and visible sync status indicators. Production voice flows require network connectivity; when offline, voice interactions are disabled and the app shall work fully via manual flows; no audio is recorded or queued.

**FR14 (Phase 2):** The system shall generate basic PDF reports for jobs showing: job title, cost breakdown by category, sum of advances, net balance, with email option via device's default mail client. This functionality is out of scope for MVP and will be delivered in Phase 2.

**FR15:** The system shall support Czech and English UI localization and voice recognition, with user-selectable language preference (Czech default).

**FR16:** The system shall use on-device Keyword Spotting (KWS) for optional wake-word activation to enable hands-free voice flow initiation.

**FR17:** The system shall implement a confirmation loop using TTS to read back parsed voice input before persisting data, allowing user correction via voice ("yes"/"no") or touch (Accept/Retry/Cancel buttons). Voice confirmation ("Say 'yes' to confirm") is the primary hands-free path; touch buttons provide fallback for noisy environments or when touch interaction is preferred. TTS may use platform-native offline voices when available; otherwise voice confirmation requires connectivity.

**FR18:** The system shall assign sequential, voice-friendly numbers via Cloud Functions using transactional allocation (not client-side counters): jobNumber, teamMemberNumber, vehicleNumber, machineNumber per tenant; ordinalNumber per job (costs/advances/events). Online: allocate via HTTPS callable; Offline: assigned on sync by onCreate triggers. Gaps acceptable; duplicates prohibited.

**FR19:** The system shall implement an offline sync conflict resolution policy. All mutable documents include `createdAt`, `createdBy` (compound identity object), `updatedAt` (serverTimestamp), and `updatedBy` (compound identity object). Conflicts are resolved via last-write-wins (LWW) at the document level; sequential numbers are allocated server-side to prevent ID conflicts; jobs use soft delete (`status: archived`) instead of destructive delete. Updates to deleted documents and other sync errors are surfaced in a "Sync Issues" UI with options to Discard, Retry, or Recreate as new. Audit logs provide traceability for manual review and recovery.

**FR20:** The system shall support true hands-free voice confirmation via wake-word ("Hey FinDog") followed by "yes" or "no" responses to accept or retry voice commands. This enables safe operation while driving or when hands are occupied. Touch-based Accept/Retry/Cancel buttons remain available as fallback for noisy environments or when hands are free.

**FR21:** The system shall gracefully handle voice recognition errors and edge cases: background noise interference, unrecognized accents/technical terms, network timeouts during STT/LLM/TTS, Firestore write failures, device storage full (offline queue), battery-critical warnings during voice operations, and wrong-language detection. Each error scenario shall provide clear user feedback and recovery options (Retry, Switch to Manual Entry, Cancel).

**FR22:** The system shall support multi-job context and inline job targeting. Any voice or manual command may specify an inline job override (e.g., "for job 123") to apply that single operation to the specified job without changing the Active Job. If no override is specified, the Active Job is used by default. If there is no Active Job and none is specified inline, the user is prompted to specify a job or set an Active Job. The Active Job changes only via the "Set Active Job" flow.

**FR23:** The system shall provide GDPR-compliant data export functionality, allowing owners to download all their tenant data (jobs, costs, advances, team members, vehicles, machines) in JSON format via a Cloud Function that generates a short-lived signed URL for authenticated download (no direct email attachments). Audit logs are excluded from export. Server-side privilege check: owner required.



### Non-Functional Requirements

**NFR1:** Voice pipeline latency (post-wake-word to first STT token) shall achieve ≤3.0 seconds (median) and ≤5.0 seconds (P95) on mid-range Android devices under typical 4G conditions.

**NFR2:** Round-trip voice confirmation (user speaks → TTS responds) shall complete in ≤8.0 seconds (median) and ≤12.0 seconds (P95) on typical 4G network conditions.

**NFR3:** Offline write operations shall provide instant local persistence with <100ms UI feedback.

**NFR4:** Firestore sync on reconnection shall commit queued writes within 10 seconds (median).

**NFR5:** Speech-to-Text quality (cs-CZ) shall achieve Word Error Rate ≤15% (median) and ≤25% (P95) on a 100+ phrase domain test set in controlled conditions; track numeric terms (digits, amounts) with ≥90% exact-match accuracy in controlled conditions.

**NFR6:** Intent recognition (core flows) shall achieve F1 ≥0.85 on a curated domain test set; numeric entity extraction (jobNumber, odometer) exact-match accuracy ≥90% in controlled conditions; report metrics separately for moderate noise.

**NFR7:** The system shall support iOS 15+ (Safari, native WebView) and Android 10+ (Chrome, native WebView).

**NFR8:** The system shall support optional desktop browser access (Chrome, Edge, Safari) for admin/reporting tasks.

**NFR9:** Firebase services shall target Europe (Belgium) region for GDPR/DSGVO compliance with EU data residency requirements.

**NFR10:** The system shall stay within Firebase free tier where feasible; voice API costs (STT/TTS/LLM) estimated at $0.10-0.30 per user per day during active usage.

**NFR11:** Audit log storage shall be optimized to minimize Firestore costs while meeting compliance requirements; estimated 2-3x increase in write operations due to Cloud Functions triggers. TTL (Time-To-Live) policy of 1 year enforced via scheduled Cloud Function for automatic log cleanup.

**NFR12:** The system shall implement cascade delete functionality for user data deletion to support GDPR data portability and right-to-erasure, including audit log entries, except where legal obligations require retention; retention/exceptions are documented in the Compliance policy.

**NFR13:** The system shall use configurable provider endpoints for STT, LLM, and TTS to enable cost optimization and vendor flexibility.

**NFR14:** Cloud Functions audit triggers shall execute asynchronously without blocking client operations, with <500ms execution time for standard CRUD operations.


**NFR15:** All numeric, currency, and distance values are formatted per user locale (cs-CZ vs en-US), including thousands separators, decimal marks, currency symbols, and unit labels.