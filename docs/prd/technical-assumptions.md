# Technical Assumptions

### Repository Structure: Monorepo

**Decision:** Single monorepo containing mobile app (Angular/Ionic/Capacitor), Cloud Functions, shared TypeScript types, and e2e tests.

**Rationale:**
- Simplifies cross-cutting changes (data model updates affect app + functions + types)
- Manageable complexity for MVP team size (1-2 developers)
- Easier dependency management and versioning
- Tools like Nx or Turborepo support monorepo workflows efficiently

**Structure:**
```
/packages
  /mobile-app       (Angular + Ionic + Capacitor)
  /functions        (Cloud Functions - Node.js)
  /shared-types     (TypeScript interfaces shared across packages)
  /e2e-tests        (End-to-end Playwright tests)
```

### Service Architecture

**High-Level Architecture:** Hybrid Client-Heavy with Serverless Backend

**Components:**
1. **Client-Side (Angular/Ionic App):**
   - Primary business logic lives in client
   - Firestore direct access via SDK (offline-first)
   - Voice pipeline orchestration (STT → LLM → TTS)
   - Local state management (NgRx or Signals)

2. **Serverless Backend (Cloud Functions):**
   - **onCreate/onUpdate/onDelete triggers:** Audit logging to `/tenants/{tenantId}/audit_logs` subcollection
   - **HTTPS callable:** PDF generation (pdfmake library) — Phase 2
   - **HTTPS callable:** allocateSequence (transactional) — allocates next number for tenant-level sequences (jobNumber, vehicleNumber, machineNumber, teamMemberNumber) and per-job sequence (ordinalNumber)
   - **onCreate assignment triggers:** when an entity is created without its sequence field (offline create), assign the next number atomically and write it back to the document
   - **Scheduled function:** Audit log TTL cleanup (runs daily, deletes entries >1 year old)
   - **Security Rules enforcement:** Firestore Security Rules handle authorization (multi-tenant isolation)

**Rationale:**
- Firestore offline capabilities eliminate need for custom sync backend
- Client-heavy approach minimizes Cloud Functions costs (pay per invocation)
- Security Rules provide declarative authorization without custom backend logic
- Audit logging via triggers is transparent to client (no code changes required for compliance)

**Why NOT Microservices:** Overkill for MVP scale; added operational complexity (service discovery, inter-service communication) not justified for expected load (50-100 users).

### Testing Requirements

**MVP Testing Strategy:**

1. **Unit Tests (Required):**
   - Services, utilities, state management logic
   - Target: 70%+ code coverage for business logic
   - Tools: Jest, Jasmine (Angular default)

2. **Integration Tests (Required for Voice Pipeline):**
   - Voice flow end-to-end (mock STT/LLM/TTS responses)
   - Firestore offline sync scenarios (airplane mode simulation)
   - Target: Core flows (Set Active Job, Start Journey, End Journey, Add Material Cost, Record Work Hours, Quick Expense) covered

3. **E2E Tests (Selective):**
   - Critical user journeys: Onboarding → Create Job → Voice Command → PDF Export (Phase 2)
   - Tools: **Playwright** (web and mobile), **MCP Browser tools** for debugging and interactive testing
   - **NOT Cypress** (explicitly excluded)
   - Target: 5-7 happy path scenarios

4. **Manual Testing (Essential):**
   - Voice recognition accuracy in real environments (car cabin, outdoor noise)
   - Touch interface usability testing (touch target sizes, button discoverability)
   - Multi-device sync validation (phone + tablet)

**NOT in MVP:**
- Performance/load testing (not needed for <100 users)
- Accessibility automated testing (WCAG compliance via manual audit)
- Security penetration testing (post-beta, pre-launch milestone)

### Additional Technical Assumptions and Requests

**Firestore Data Model:**
```
/tenants/{tenantId}/
  members/{uid}
    - role (owner|representative|teamMember)
    - status (active|disabled), lastSeenAt
  invites/{inviteId}
    - codeHash, expiresAt, createdBy, presetRole, consumedAt (optional), email (optional)
  jobs/{jobId}
    - tenantId, jobNumber (sequence), title, status, currency, vatRate, budget
    - createdAt, createdBy, updatedAt, updatedBy (audit metadata)
    costs/{costId}
      - tenantId, ordinalNumber (sequence), category, amount, description
      - resource (object copy at creation; non-authoritative): transport embeds vehicleNumber, name, distanceUnit, ratePerDistanceUnit; labor embeds teamMemberNumber, name, hourlyRate; machine embeds machineNumber, name, hourlyRate
      - createdAt, createdBy, updatedAt, updatedBy
    advances/{advanceId}
      - tenantId, ordinalNumber (sequence), amount, date, note
      - createdAt, createdBy, updatedAt, updatedBy
    events/{eventId}
      - tenantId, ordinalNumber (sequence), type, timestamp, data (journey details, etc.)
      - createdAt, createdBy, updatedAt, updatedBy
  jobs_public/{jobId}
    - jobNumber, title, status (sanitized read model for teamMember role)
  vehicles/{vehicleId}
    - tenantId, vehicleNumber (sequence), name, distanceUnit (from business profile), ratePerDistanceUnit
    - createdAt, createdBy, updatedAt, updatedBy
  machines/{machineId}
    - tenantId, machineNumber (sequence), name, hourlyRate
    - createdAt, createdBy, updatedAt, updatedBy
  teamMembers/{teamMemberId}
    - tenantId, teamMemberNumber (sequence), name, hourlyRate
    - authUserId (Firebase Auth UID for login mapping; owner's team member #1 maps to owner)
    - createdAt, createdBy, updatedAt, updatedBy
  businessProfile (document)
    - tenantId, currency, vatRate, distanceUnit
    - createdAt, updatedAt
  personProfile (document)
    - tenantId, displayName, email, language, preferredVoiceProvider, aiSupportEnabled (boolean)
    - createdAt, updatedAt
  audit_logs/{logId}
    - operation (CREATE|UPDATE|DELETE), collection, documentId, tenantId
    - timestamp, authorId (team member ID)
    - before (old data for UPDATE), after (new data for CREATE/UPDATE)
    - ttl (expires 1 year from timestamp)

/user_tenants/{uid}/memberships/{tenantId}
  - tenantId, role ("owner"|"representative"|"teamMember") snapshot (optional)
  - createdAt (derived), updatedAt

```

#### User-to-Tenant Mapping (user_tenants)

- Purpose: Persist a mapping from Firebase Auth UID to a tenantId to enable strict multi-tenant isolation while using Firebase Authentication.
- Collection name: user_tenants (standardized).
- Document ID: {uid}
- Fields: tenantId (UUID), createdAt (serverTimestamp)
- Sign-in flow:
  1) Extract `uid` from Firebase ID token
  2) Resolve mapping at `user_tenants/{uid}` (or `/user_tenants/{uid}/memberships/{tenantId}` index)
  3) If absent, create a new tenant and store the mapping atomically
  4) Set custom claim `tenant_id = tenantId` and force-refresh the ID token
- JWT example (roles illustrative only; authorization is enforced via membership documents):
```json
{
  "user_id": "firebase-uid",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "roles": ["user"]
}
```
- Security: Firestore Rules require membership at `/tenants/{tenantId}/members/{uid}`; the mapping collection (`user_tenants`) is read-only to the client and maintained by Cloud Functions.
- MVP policy: Default is one user → one tenant; invitation flows allow additional memberships. The active tenant is selected via the `tenant_id` custom claim (see Story 1.3b).


**Voice Pipeline Providers (Configurable via Environment Variables):**
- **STT:** Google Cloud Speech-to-Text (primary for Czech), OpenAI Whisper (fallback/testing)
- **LLM/NLU:** OpenAI GPT-4-turbo (primary), Groq Llama3 (cost-efficient alternative), Ollama (local dev/testing)
- **TTS:** Google Cloud Text-to-Speech (Czech voices: cs-CZ-Wavenet-A), platform-native fallback (Web Speech API)
- **KWS:** Porcupine (Picovoice) or Vosk for on-device wake-word, configurable wake phrase (default: "Hey FinDog"). Verify on-device-only processing and licensing compliance; no audio leaves the device for wake-word detection.

**Security & Compliance:**
- **Firebase Region:** europe-west1 (Belgium) for Firestore, Storage, Functions—GDPR/DSGVO compliance
- **Authentication:** Firebase Auth email/password for MVP; add OAuth (Google) post-MVP
- **Data Isolation:** Firestore Security Rules enforce membership-based authorization: access to `/tenants/{tenantId}/**` requires membership at `/tenants/{tenantId}/members/{request.auth.uid}`; roles gate reads/writes
- **GDPR Right-to-Erasure:** Cloud Function implements cascade delete of tenant data including audit logs

**Dependency Management:**
- Node.js: v20 LTS (Cloud Functions, build tooling)
- Angular: v20+ (latest stable)
- Ionic: v8+ (latest stable with Capacitor support)
- Firebase SDK: v11+ (@angular/fire)

**Developer Experience:**
- **Local Development:** Firebase Emulators (Auth, Firestore, Functions, Storage) for offline dev
- **Voice API Mocks:** Configurable to use mock responses (skip API costs during feature dev)
- **Environment Config:** `.env` files for API keys, provider endpoints (never commit to git)

**Deployment Pipeline:**
- **Hosting:** Firebase Hosting for PWA
- **Native Builds:** Capacitor CLI for iOS/Android builds (local dev), CI/CD via GitHub Actions (future). Note: iOS builds require macOS/Xcode or a macOS CI runner.
- **Functions:** Firebase CLI (`firebase deploy --only functions`)
- **Database:** Firestore Security Rules deployed via Firebase CLI
