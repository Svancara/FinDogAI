# FinDogAI Product Requirements Document (PRD)

## Goals and Background Context

### Goals

- Enable solo mobile craftsmen to capture 100% of job costs in real-time through hands-free voice interaction
- Reduce daily administrative overhead from 30-60 minutes to under 15 minutes
- Provide offline-first cost tracking that works reliably without network connectivity
- Eliminate revenue leakage from forgotten expenses (currently 5-15% of billable costs)
- Deliver production-ready MVP with core voice flows (Set Active Job + Start Journey) within 6 months
- Achieve 70%+ weekly retention and 80%+ voice flow success rate among beta users
- Validate Czech language voice accuracy >90% with English as second language support

### Background Context

Small entrepreneurs and craftsmen in Central Europe face a persistent profitability problem: accurately remembering and recording all costs, purchases, mileage, and work hours at day's end. This manual recall process creates revenue leakage (estimated 5-15% of billable costs go unrecorded), cognitive burden after long workdays, and safety concerns from attempting to type notes while driving. Current solutions—generic expense trackers, voice assistants, and job management software—fail to address the unique combination of needs: hands-free voice capture, offline-first reliability, domain-specific intent recognition for craftsman workflows, and job-specific cost allocation.

FinDogAI combines voice-first interaction, offline-first Firebase/Firestore architecture, and AI-powered natural language understanding to create a hands-free assistant optimized for mobile craftsmen. The MVP focuses on two critical voice flows (Set Active Job + Start Journey with Odometer) to prove the value proposition. The solution leverages on-device keyword spotting for true hands-free operation, streaming speech-to-text with multilingual support, LLM-based intent recognition, and conversational TTS confirmation—all functioning fully offline with automatic sync when connectivity returns.

### Change Log

| Date | Version | Description | Author |
|------|---------|-------------|--------|
| 2025-10-23 | v1.0 | Initial PRD creation from approved Project Brief | John (PM) |
| 2025-10-23 | v1.1 | Incorporated remarks: audit logging, team member privileges, sequential ID voice references | John (PM) |

## Requirements

### Functional Requirements

**FR1:** The system shall provide a "Set Active Job" voice flow that accepts natural language input including numeric job IDs (e.g., "Set active job to 123" or "Set active job to Smith, Brno"), confirms via TTS, and persists the active job context to auto-assign subsequent costs.

**FR2:** The system shall provide a "Start Journey" voice flow that accepts natural language input with destination, vehicle, and odometer reading (e.g., "I'm going to Brno in Transporter, odometer 12345"), confirms via TTS, and creates a journey event under the active job.

**FR3:** The system shall provide manual entry screens for creating, editing, and deleting costs across five categories: Transport, Material, Labor, Machine, and Other, with support for referencing items by sequential ID in voice commands (e.g., "Update material 45").

**FR4:** The system shall provide CRUD operations for Jobs with fields: auto-assigned jobNumber (Firestore counter), title, description, status (active/completed/archived), currency (ISO 4217), VAT rate, and budget.

**FR5:** The system shall provide a Business Profile setup for one-time configuration of default currency, VAT rate, and distance unit (km/miles).

**FR6:** The system shall provide Resources Management for Team Members (minimum one: the user), Vehicles, and Machines, each with properties (name, hourly rates for labor/machines, per-distance rates for vehicles).

**FR7:** The system shall maintain basic audit metadata on all database entities: createdAt timestamp, createdBy user ID, updatedAt timestamp, and updatedBy user ID.

**FR8:** The system shall implement comprehensive audit logging via Cloud Functions triggers (onCreate/onUpdate/onDelete) that capture full operation history to a separate audit_logs collection, including: operation type, timestamp, author (user ID), old values (for UPDATE), and complete object snapshots (for DELETE). Audit logs are backend-only (no user-facing UI) and auto-expire after 1 year for cost optimization.

**FR9:** The system shall track Advances as a per-job subcollection with auto-assigned ordinalNumber (Firestore counter), displaying sum of advances vs sum of costs.

**FR10:** The system shall implement Firebase Authentication (email/password minimum) with Firestore Security Rules enforcing tenantId-scoped read/write isolation under `/users/{tenantId}/` paths.

**FR11:** The system shall support basic team member privilege system where the business owner (tenant creator) can assign individual privilege toggles to team members: privilege to add cost events (transport, material, labor, machine, other) and privilege to view job financial state (budget, actual costs, profit). No role templates for MVP.

**FR12:** The system shall require team member authentication and identification, storing the team member ID with all operations for audit trail purposes.

**FR13:** The system shall enable Firestore offline persistence, ensuring all voice and manual flows function without network connectivity, with automatic background sync and visible sync status indicators.

**FR14:** The system shall generate basic PDF reports for jobs showing: job title, cost breakdown by category, sum of advances, net balance, with email option via device's default mail client.

**FR15:** The system shall support Czech and English UI localization and voice recognition, with user-selectable language preference (Czech default).

**FR16:** The system shall use on-device Keyword Spotting (KWS) for optional wake-word activation to enable hands-free voice flow initiation.

**FR17:** The system shall implement a confirmation loop using TTS to read back parsed voice input before persisting data, allowing user correction.

**FR18:** The system shall use FieldValue.increment() for sequential auto-numbering (jobNumber, teamMemberNumber, vehicleNumber, machineNumber per tenant; ordinalNumber for costs/advances/events per job) to enable voice-friendly numeric references.

### Non-Functional Requirements

**NFR1:** Voice pipeline latency shall achieve post-wake-word to first STT token in <1.5 seconds on mid-range Android devices.

**NFR2:** Round-trip voice confirmation (user speaks → TTS responds) shall complete in <3 seconds on typical 4G network conditions.

**NFR3:** Offline write operations shall provide instant local persistence with <100ms UI feedback.

**NFR4:** Firestore sync on reconnection shall commit queued writes within 10 seconds (median).

**NFR5:** Speech-to-Text accuracy shall maintain <5% error rate for short utterances in low-noise conditions.

**NFR6:** Intent classification accuracy shall exceed 90% for core flows (StartJourney, SetActiveJob, AddMaterial).

**NFR7:** The system shall support iOS 15+ (Safari, native WebView) and Android 10+ (Chrome, native WebView).

**NFR8:** The system shall support optional desktop browser access (Chrome, Edge, Safari) for admin/reporting tasks.

**NFR9:** Firebase services shall target Europe (Belgium) region for GDPR/DSGVO compliance with EU data residency requirements.

**NFR10:** The system shall stay within Firebase free tier where feasible; voice API costs (STT/TTS/LLM) estimated at $0.10-0.30 per user per day during active usage.

**NFR11:** Audit log storage shall be optimized to minimize Firestore costs while meeting compliance requirements; estimated 2-3x increase in write operations due to Cloud Functions triggers. TTL (Time-To-Live) policy of 1 year enforced via scheduled Cloud Function for automatic log cleanup.

**NFR12:** The system shall implement cascade delete functionality for user data deletion to support GDPR data portability and right-to-erasure, including audit log entries.

**NFR13:** The system shall use configurable provider endpoints for STT, LLM, and TTS to enable cost optimization and vendor flexibility.

**NFR14:** Cloud Functions audit triggers shall execute asynchronously without blocking client operations, with <500ms execution time for standard CRUD operations.

## User Interface Design Goals

### Overall UX Vision

FinDogAI prioritizes a **voice-first, eyes-free interaction model** optimized for mobile craftsmen operating vehicles and working on-site with gloves. The UI serves as a visual confirmation layer and fallback for manual operations, not the primary interaction paradigm. Design emphasizes **large touch targets, minimal text entry, clear visual feedback for offline/sync status, and audio-centric workflows**. The application should feel like a conversational assistant that happens to have a visual interface, rather than a traditional form-heavy app with voice bolted on.

### Key Interaction Paradigms

1. **Voice-First with Visual Confirmation:** Primary flow is speak → see confirmation → approve/correct. Voice commands trigger immediate visual feedback showing parsed entities (job ID, amounts, odometer readings) with large Accept/Retry buttons.

2. **Active Job Context Banner:** Persistent visual indicator at top of every screen showing currently active job (number + title) with quick-tap to change. Banner is always visible (non-dismissible) to reinforce the mental model that all actions apply to the active job unless specified otherwise.

3. **Sequential ID Shortcuts:** All lists (jobs, costs, resources) display prominent numeric IDs as primary visual identifiers, supporting voice commands like "Set active job to 123" or "Delete cost 45".

4. **Offline-First Status Visibility:** Persistent connectivity indicator with sync queue count. Offline mode is presented as normal operation with standard UI appearance (no special color theme)—no blocking warnings, just informative badges.

5. **Glove-Friendly Ergonomics:** Minimum 48dp touch targets, high contrast colors, avoid swipe gestures that require precision. Favor large buttons and voice over complex navigation hierarchies. No haptic feedback required.

### Core Screens and Views

1. **Voice Command Hub (Home Screen):** Central microphone button (wake-word optional), active job display, recent activity feed, quick stats (costs today, sync status). Primary entry point for voice interactions.

2. **Jobs List:** Scrollable job cards showing `[jobNumber] Title - Status` with financial summary (costs vs budget). Sorted by status first, then by jobNumber ascending. Tap to view details, long-press for quick "Set Active" action.

3. **Job Detail View:** Tabbed interface: Costs (breakdown by category with sequential IDs), Advances (ordinalNumber + amount), Events timeline, PDF export action.

4. **Cost Entry Form (Manual Fallback):** Category-specific forms (Transport/Material/Labor/Machine/Other) with large number pads, vehicle/machine picker, voice dictation for descriptions. Pre-filled with active job context.

5. **Resources Management:** Three tabs (Team Members, Vehicles, Machines). List view with sequential IDs and key properties (rates). Owner can toggle individual privileges on Team Member detail screen.

6. **Settings/Business Profile:** One-time setup wizard style for currency, VAT, distance unit, language preference, voice provider configuration (for developers).

7. **Voice Confirmation Modal:** Full-screen overlay during voice interaction showing real-time transcription, parsed entities in structured format, and Accept/Retry/Cancel actions. TTS readback plays while visual display updates—user waits for audio confirmation before proceeding (non-dismissible during TTS playback).

### Accessibility: WCAG AA

- Target WCAG 2.1 Level AA compliance for MVP
- High contrast color ratios (4.5:1 minimum for text)
- Screen reader support for visual feedback during voice flows (announce "Job 123 set as active")
- TTS readback serves as primary accessibility feature for vision-impaired users
- Large text mode support (iOS/Android system settings)
- Voice commands provide alternative to all touch interactions for core workflows

### Branding

**Visual Identity:** Clean, utilitarian design avoiding "tech startup" aesthetics. Think rugged toolbox rather than sleek dashboard.

**Color Palette (Functional High-Contrast):**

| Purpose | Color | Hex Code | Usage |
|---------|-------|----------|--------|
| Primary Action | Safety Orange | `#FF6B35` | Main CTA buttons, microphone button, active states |
| Success/Sync | Forest Green | `#2D6A4F` | Sync success indicators, confirmation checkmarks |
| Warning/Attention | Amber | `#F4A261` | Validation warnings, pending sync queue badge |
| Error/Destructive | Signal Red | `#D62828` | Delete actions, error messages, failed operations |
| Background | Off-White | `#F8F9FA` | Main screen background (reduces eye strain vs pure white) |
| Surface/Cards | White | `#FFFFFF` | Cards, modals, input fields |
| Text Primary | Charcoal | `#212529` | Body text, headings (AAA contrast on off-white) |
| Text Secondary | Slate Gray | `#6C757D` | Secondary info, metadata, timestamps |
| Border/Divider | Light Gray | `#DEE2E6` | Borders, dividers, disabled states |
| Active Job Banner | Deep Blue | `#1D3557` | Background for persistent active job banner (high contrast with white text) |

**Typography:**
- Font Family: Roboto (Android), SF Pro (iOS), fallback to system sans-serif for web
- Body Text: 16px minimum (1rem)
- Headings: Bold weight, 20px-28px depending on hierarchy
- Job Numbers/Sequential IDs: Monospace variant (Roboto Mono) at 18px for scannability

**Iconography:** Material Design Icons (solid fill), minimum 24x24dp, not line art—better visibility in bright outdoor light.

**Tone:** Confident, direct, unpretentious. Messages like "Job 23 set. Ready to record costs." not "Great! You've successfully configured your active job context!"

### Target Device and Platforms: Web Responsive (Mobile-First)

- **Primary:** Mobile phones (iOS 15+, Android 10+) in portrait orientation—5.5" to 6.7" screens
- **Secondary:** Tablets (iPad, Android tablets) for office/desk use cases (viewing reports, bulk data entry)
- **Tertiary:** Desktop browsers (Chrome/Edge/Safari) for admin tasks, but not optimized—mobile-first design scales up
- **Native Builds:** Capacitor builds for iOS and Android with platform-specific permissions (microphone, filesystem) and optional OS integrations (Siri Shortcuts future consideration)
- **Network Conditions:** Optimized for 4G with offline-first, gracefully handles 3G degradation (slower voice API responses but fully functional)

## Technical Assumptions

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
   - **onCreate/onUpdate/onDelete triggers:** Audit logging to `audit_logs` collection
   - **HTTPS callable:** PDF generation (pdfmake library)
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
   - Target: Core flows (Set Active Job, Start Journey) fully covered

3. **E2E Tests (Selective):**
   - Critical user journeys: Onboarding → Create Job → Voice Command → PDF Export
   - Tools: **Playwright** (web and mobile), **MCP Browser tools** for debugging and interactive testing
   - **NOT Cypress** (explicitly excluded)
   - Target: 5-7 happy path scenarios

4. **Manual Testing (Essential):**
   - Voice recognition accuracy in real environments (car cabin, outdoor noise)
   - Glove usability testing (touch target sizes, button discoverability)
   - Multi-device sync validation (phone + tablet)

**NOT in MVP:**
- Performance/load testing (not needed for <100 users)
- Accessibility automated testing (WCAG compliance via manual audit)
- Security penetration testing (post-beta, pre-launch milestone)

### Additional Technical Assumptions and Requests

**Firestore Data Model:**
```
/users/{tenantId}/
  /jobs/{jobId}
    - jobNumber (counter), title, status, currency, vatRate, budget
    - createdAt, createdBy, updatedAt, updatedBy (audit metadata)
    /costs/{costId}
      - ordinalNumber (counter), category, amount, description, resourceId
      - createdAt, createdBy, updatedAt, updatedBy
    /advances/{advanceId}
      - ordinalNumber (counter), amount, date, note
      - createdAt, createdBy, updatedAt, updatedBy
    /events/{eventId}
      - ordinalNumber (counter), type, timestamp, data (journey details, etc.)
      - createdAt, createdBy, updatedAt, updatedBy
  /vehicles/{vehicleId}
    - vehicleNumber (counter), name, distanceUnit (from business profile) ratePer distance unit
    - createdAt, createdBy, updatedAt, updatedBy
  /machines/{machineId}
    - machineNumber (counter), name, hourlyRate
    - createdAt, createdBy, updatedAt, updatedBy
  /teamMembers/{teamMemberId}
    - teamMemberNumber (counter), name, hourlyRate, privileges {canAddCosts, canViewFinancials}
    - authUserId (Firebase Auth UID for login mapping)
    - createdAt, createdBy, updatedAt, updatedBy
  /businessProfile (document)
    - currency, vatRate, distanceUnit
    - createdAt, updatedAt
  /personProfile (document)
    - displayName, email, language, preferredVoiceProvider
    - createdAt, updatedAt

/audit_logs/{logId}
  - operation (CREATE|UPDATE|DELETE), collection, documentId, tenantId
  - timestamp, authorId (team member ID)
  - before (old data for UPDATE), after (new data for CREATE/UPDATE)
  - ttl (expires 1 year from timestamp)
```

**Voice Pipeline Providers (Configurable via Environment Variables):**
- **STT:** Google Cloud Speech-to-Text (primary for Czech), OpenAI Whisper (fallback/testing)
- **LLM/NLU:** OpenAI GPT-4-turbo (primary), Groq Llama3 (cost-efficient alternative), Ollama (local dev/testing)
- **TTS:** Google Cloud Text-to-Speech (Czech voices: cs-CZ-Wavenet-A), platform-native fallback (Web Speech API)
- **KWS:** Porcupine (Picovoice) or Vosk for on-device wake-word, configurable wake phrase (default: "Hey FinDog")

**Security & Compliance:**
- **Firebase Region:** europe-west1 (Belgium) for Firestore, Storage, Functions—GDPR/DSGVO compliance
- **Authentication:** Firebase Auth email/password for MVP; add OAuth (Google) post-MVP
- **Data Isolation:** Firestore Security Rules enforce `request.auth.token.tenant_id == tenantId` for all reads/writes
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
- **Native Builds:** Capacitor CLI for iOS/Android builds (local dev), CI/CD via GitHub Actions (future)
- **Functions:** Firebase CLI (`firebase deploy --only functions`)
- **Database:** Firestore Security Rules deployed via Firebase CLI

## Epic List

### **Epic 1: Foundation & Authentication Infrastructure**
**Goal:** Establish project setup, Firebase infrastructure, multi-tenant authentication, and basic data model with a deployable "hello world" that proves the tech stack works end-to-end.

### **Epic 2: Core Data Management & Business Setup**
**Goal:** Implement Jobs, Resources (Team Members, Vehicles, Machines), and Business/Person Profiles with full CRUD operations, enabling users to set up their business context before voice flows.

### **Epic 3: Voice Pipeline & Active Job Context**
**Goal:** Build the voice infrastructure (STT → LLM → TTS pipeline) and implement the "Set Active Job" voice flow with confirmation loop, establishing the foundational voice-first interaction pattern.

### **Epic 4: Journey Tracking & Cost Management**
**Goal:** Implement "Start Journey" voice flow and manual cost entry screens for all five categories (Transport, Material, Labor, Machine, Other), delivering complete cost capture functionality.

### **Epic 5: Audit Logging & Team Privileges**
**Goal:** Implement Cloud Functions audit triggers, basic team member privilege system, and compliance features (GDPR data deletion, audit log TTL cleanup).

### **Epic 6: Reporting, Export & MVP Polish**
**Goal:** Add PDF generation, job financial summaries, offline sync status visibility, and final UX polish to deliver production-ready MVP.

## Epic 1: Foundation & Authentication Infrastructure

**Expanded Goal:** Establish the technical foundation for FinDogAI by setting up the Angular/Ionic/Capacitor monorepo, configuring Firebase services (Auth, Firestore, Functions), implementing multi-tenant authentication with Security Rules, deploying a basic "health check" screen, and proving the complete CI/CD pipeline works. This epic delivers a deployable application that validates the tech stack end-to-end.

### Story 1.1: Project Initialization & Monorepo Setup

**As a** developer,
**I want** a properly configured monorepo with Angular/Ionic/Capacitor structure,
**so that** I can develop the mobile app, Cloud Functions, and shared types in a unified workspace.

**Acceptance Criteria:**

1. Monorepo created using Nx or Turborepo with workspace configuration
2. `/packages/mobile-app` initialized with Angular 20+ and Ionic 8+
3. `/packages/functions` initialized with Cloud Functions (Node.js v20)
4. `/packages/shared-types` created with TypeScript interfaces
5. `/packages/e2e-tests` initialized with Playwright
6. Root-level `package.json` with workspace scripts (`npm run dev`, `npm run build`, `npm test`)
7. `.gitignore` configured (node_modules, .env, build artifacts)
8. `README.md` with setup instructions and architecture overview
9. All packages build successfully without errors
10. Dev server runs locally on `http://localhost:4200`

### Story 1.2: Firebase Project Configuration

**As a** developer,
**I want** Firebase project configured with europe-west1 region and necessary services enabled,
**so that** the app can authenticate users and persist data with GDPR compliance.

**Acceptance Criteria:**

1. Firebase project created via Firebase Console (project ID: `findogai-mvp` or similar)
2. Firestore database initialized in `europe-west1` (Belgium) region
3. Firebase Authentication enabled with Email/Password provider
4. Firebase Hosting enabled for PWA deployment
5. Cloud Functions configured for `europe-west1` deployment
6. Firebase Storage enabled with `europe-west1` location
7. Firebase CLI installed and authenticated locally (`firebase login`)
8. `.firebaserc` file created with project alias
9. `firebase.json` configured for Hosting, Functions, Firestore rules
10. Firebase Emulators installed (Auth, Firestore, Functions, Storage)
11. Emulator configuration in `firebase.json` with ports defined
12. `npm run emulators` starts all emulators successfully

### Story 1.3: Multi-Tenant Authentication & User Registration

**As a** new user,
**I want** to register with email/password and have a tenant automatically created,
**so that** my data is isolated from other users in the system.

**Acceptance Criteria:**

1. Registration screen with fields: email, password, displayName
2. Form validation: email format, password strength (min 8 chars), required fields
3. On successful registration, Firebase Auth user created
4. Custom claim `tenant_id` set to user's UID (user is their own tenant)
5. Firestore document `/users/{tenantId}/personProfile` created with displayName, email, language (default: cs), createdAt
6. Firestore document `/users/{tenantId}/businessProfile` created with currency (CZK), vatRate (21%), distanceUnit (km), createdAt
7. Firestore document `/users/{tenantId}/teamMembers/{tenantMemberId}` auto-created for registering user with teamMemberNumber: 1, authUserId: user.uid, privileges: {canAddCosts: true, canViewFinancials: true}
8. Success message displayed: "Welcome, [displayName]! Your account is ready."
9. User automatically logged in and redirected to home screen
10. Registration errors handled gracefully (email already exists, weak password, network failure)

### Story 1.4: User Login & Session Management

**As a** registered user,
**I want** to log in with my email/password,
**so that** I can access my data across devices.

**Acceptance Criteria:**

1. Login screen with fields: email, password
2. "Remember me" checkbox (persists session locally)
3. On successful login, Firebase Auth session established
4. Custom claim `tenant_id` retrieved and stored in app state
5. User redirected to home screen after login
6. Login errors handled: "Invalid email/password", "User not found", "Too many attempts"
7. "Forgot password" link triggers Firebase password reset email
8. Session persists across app restarts (if "Remember me" checked)
9. Logout button clears session and returns to login screen
10. Logged-in users cannot access registration screen (redirect to home)

### Story 1.5: Firestore Security Rules for Multi-Tenant Isolation

**As a** system architect,
**I want** Firestore Security Rules enforcing tenant-scoped data access,
**so that** users cannot read or write other tenants' data.

**Acceptance Criteria:**

1. Security Rules file `firestore.rules` created
2. Rule: Users can only access `/users/{tenantId}/**` where `tenantId == request.auth.token.tenant_id`
3. Rule: Unauthenticated users have no read/write access (except public collections if needed in future)
4. Rule: Audit logs (`/audit_logs/{logId}`) are write-only from Cloud Functions (client cannot write)
5. Rules deployed to Firebase via `firebase deploy --only firestore:rules`
6. Manual testing: User A cannot read User B's jobs (returns permission denied)
7. Manual testing: Unauthenticated request to Firestore returns permission denied
8. Security Rules validated via Firebase Emulator Suite (unit tests for rules)
9. Rules include comments explaining tenant isolation logic
10. Rules version controlled in git repository

### Story 1.6: Basic Health Check Screen & CI/CD Validation

**As a** developer,
**I want** a deployable "health check" screen that validates Firebase connectivity,
**so that** I can confirm the deployment pipeline works end-to-end.

**Acceptance Criteria:**

1. Home screen displays after login: "FinDogAI Health Check"
2. Screen shows: Firebase Auth status (✓ Authenticated as [email])
3. Screen shows: Firestore connectivity status (✓ Connected to [region])
4. Screen shows: Current tenant ID (✓ Tenant: [tenantId])
5. Screen shows: App version and build timestamp
6. Health check writes a test document to `/users/{tenantId}/healthCheck` and reads it back
7. If write/read succeeds, display "✓ Firestore Read/Write OK"
8. If offline, display "⚠ Offline Mode - Sync pending"
9. PWA deployed to Firebase Hosting via `firebase deploy --only hosting`
10. Deployed app accessible at `https://findogai-mvp.web.app` (or custom domain)
11. Health check screen functional in deployed PWA
12. Capacitor Android build generates APK successfully (`npx cap build android`)

## Epic 2: Core Data Management & Business Setup

**Expanded Goal:** Implement the foundational data entities (Jobs, Vehicles, Machines, Team Members) with full CRUD operations, sequential ID generation via Firestore counters, and enable users to configure Business and Person Profiles. This epic establishes the business context that voice flows will reference, delivering a functional job management system before voice features are added.

**CRITICAL ARCHITECTURAL PRINCIPLE:** Costs and Events collections will NEVER use references to resources (Vehicles, Machines, Team Members). Instead, they will store complete copies of resource data at the time of creation. This enables resource deletion without referential integrity checking and provides audit trail of historical resource states.

### Story 2.1: Jobs CRUD with Sequential Job Numbers

**As a** craftsman,
**I want** to create, view, edit, and archive jobs with auto-assigned job numbers,
**so that** I can organize my work and reference jobs by number in voice commands.

**Acceptance Criteria:**

1. Jobs List screen displays all jobs sorted by status (active, completed, archived), then by jobNumber ascending
2. Each job card shows: `[jobNumber] Title - Status` with financial summary (total costs, budget, remaining)
3. "Create Job" button navigates to job creation form
4. Job creation form fields: title (required), description, budget (optional), currency (default from businessProfile), vatRate (default from businessProfile)
5. On save, `jobNumber` auto-assigned via FieldValue.increment() on `/users/{tenantId}/counters/jobCounter`
6. New job saved to `/users/{tenantId}/jobs/{jobId}` with: jobNumber, title, description, status: "active", currency, vatRate, budget, createdAt, createdBy (current team member ID)
7. Job detail screen displays all job fields with Edit and Archive buttons
8. Edit job updates: title, description, budget, currency, vatRate, updatedAt, updatedBy
9. Archive job changes status to "archived" (soft delete, not physically removed)
10. Empty state message: "No jobs yet. Create your first job to get started."
11. Offline mode: Job CRUD works without network, syncs when online

### Story 2.2: Vehicles Management with Rate Configuration

**As a** craftsman,
**I want** to manage my vehicles with per-distance rates,
**so that** journey costs can be calculated automatically based on odometer readings.

**Acceptance Criteria:**

1. Resources Management screen has "Vehicles" tab
2. Vehicles list displays: `[vehicleNumber] Name - Rate: X CZK/km` (or /mi based on businessProfile distanceUnit)
3. "Add Vehicle" button navigates to vehicle creation form
4. Vehicle form reads distanceUnit from businessProfile and displays single rate field labeled "Rate per [Km/Mile]" accordingly
5. Vehicle form fields: name (required, e.g., "Transporter", "Škoda Octavia"), rate (required, labeled based on distanceUnit)
6. On save, `vehicleNumber` auto-assigned via FieldValue.increment() on `/users/{tenantId}/counters/vehicleCounter`
7. Vehicle saved to `/users/{tenantId}/vehicles/{vehicleId}` with: vehicleNumber, name, distanceUnit (copied from businessProfile), ratePerDistanceUnit, createdAt, createdBy
8. Edit vehicle updates: name, rate, updatedAt, updatedBy (distanceUnit remains immutable from creation time)
9. Delete vehicle removes from Firestore immediately without reference checking (costs/events contain full vehicle copies)
10. Empty state: "No vehicles yet. Add your first vehicle to track journey costs."
11. Vehicle deletion shows confirmation: "Delete [vehicleNumber] Name? This cannot be undone."

### Story 2.3: Machines Management with Hourly Rates

**As a** craftsman,
**I want** to manage my machines/equipment with hourly rates,
**so that** machine labor costs can be tracked per job.

**Acceptance Criteria:**

1. Resources Management screen has "Machines" tab
2. Machines list displays: `[machineNumber] Name - Rate: X CZK/hour`
3. "Add Machine" button navigates to machine creation form
4. Machine form fields: name (required, e.g., "Excavator", "Drill"), hourlyRate (required)
5. On save, `machineNumber` auto-assigned via FieldValue.increment() on `/users/{tenantId}/counters/machineCounter`
6. Machine saved to `/users/{tenantId}/machines/{machineId}` with: machineNumber, name, hourlyRate, createdAt, createdBy
7. Edit machine updates: name, hourlyRate, updatedAt, updatedBy
8. Delete machine removes from Firestore immediately without reference checking (costs/events contain full machine copies)
9. Empty state: "No machines yet. Add equipment to track machine labor costs."
10. Machine deletion shows confirmation: "Delete [machineNumber] Name? This cannot be undone."

### Story 2.4: Team Members Management with Privilege Toggles

**As a** business owner,
**I want** to manage team members and assign individual privileges,
**so that** I can control who can add costs and view financial details.

**Acceptance Criteria:**

1. Resources Management screen has "Team Members" tab
2. Team members list displays: `[teamMemberNumber] Name - Rate: X CZK/hour` with privilege badges (💰 Can Add Costs, 👁️ Can View Financials)
3. Registering user auto-created as team member #1 with full privileges (from Story 1.3) - this is the business owner
4. "Add Team Member" button navigates to team member form
5. Form fields: name (required), hourlyRate (required), email (optional for future auth), privileges: canAddCosts (toggle), canViewFinancials (toggle)
6. On save, `teamMemberNumber` auto-assigned via FieldValue.increment() on `/users/{tenantId}/counters/teamMemberCounter`
7. Team member saved to `/users/{tenantId}/teamMembers/{teamMemberId}` with: teamMemberNumber, name, hourlyRate, privileges, authUserId (null for MVP), createdAt, createdBy
8. Edit team member updates: name, hourlyRate, privileges, updatedAt, updatedBy (teamMemberNumber #1 cannot modify privileges - toggles disabled)
9. Delete team member removes from Firestore immediately without reference checking (costs/events contain full team member copies)
10. **Owner protection:** Team member #1 cannot be deleted - delete button hidden/disabled with tooltip: "Business owner (Member #1) cannot be deleted"
11. Non-owner team member deletion shows confirmation: "Delete [teamMemberNumber] Name? This cannot be undone."
12. Empty state: "No team members yet" should never occur (owner is always present as #1)

### Story 2.5: Business Profile Configuration

**As a** business owner,
**I want** to configure business-wide defaults (currency, VAT rate, distance unit),
**so that** new jobs inherit these settings and I don't repeat data entry.

**Acceptance Criteria:**

1. Settings screen has "Business Profile" section
2. Fields displayed: Currency (dropdown: CZK, EUR, USD with ISO 4217 codes), VAT Rate (percentage, default 21%), Distance Unit (radio: Kilometers / Miles)
3. On first access, fields pre-filled from `/users/{tenantId}/businessProfile` (created in Story 1.3)
4. Save button updates businessProfile document with: currency, vatRate, distanceUnit, updatedAt
5. Success message: "Business profile updated"
6. New jobs created after update use new defaults
7. Existing jobs retain their original settings (not retroactively changed)
8. Validation: VAT rate must be 0-100%, currency must be valid ISO code
9. Offline mode: Changes saved locally, sync when online
10. Settings persist across app restarts

### Story 2.6: Person Profile & Language Selection

**As a** user,
**I want** to configure my personal preferences (display name, language),
**so that** the app UI and voice interactions use my preferred language.

**Acceptance Criteria:**

1. Settings screen has "Person Profile" section
2. Fields displayed: Display Name (text), Email (read-only, from Firebase Auth), Language (dropdown: Czech / English)
3. On first access, fields pre-filled from `/users/{tenantId}/personProfile` (created in Story 1.3)
4. Language change immediately saves to Firestore (`personProfile.language`) before UI update to ensure persistence
5. After save completes, language change immediately updates current screen (Person Profile) UI text (Czech ↔ English using Angular i18n)
6. Language preference applied to all subsequent screens and app restarts
7. Voice provider configuration field: preferredVoiceProvider (dropdown: Google Cloud / OpenAI Whisper / Platform Native) for developers
8. Language preference passed to STT/TTS providers (cs-CZ or en-US locale)
9. Save button (for displayName/voiceProvider) updates personProfile document with: displayName, preferredVoiceProvider, updatedAt
10. Success message: "Profile updated" (in newly selected language)
11. Offline mode: Changes saved locally, sync when online
12. Logout button present at bottom of settings screen

## Epic 3: Voice Pipeline & Active Job Context

**Expanded Goal:** Build the complete voice infrastructure (STT → LLM NLU → TTS confirmation pipeline) with configurable providers, implement offline voice mock mode, and deliver the "Set Active Job" voice flow with full confirmation loop. This epic establishes the foundational voice-first interaction pattern that all subsequent voice features will follow, proving the core technical value proposition of hands-free operation.

### Story 3.1: Voice Service Architecture & Provider Configuration

**As a** developer,
**I want** a configurable voice service layer that supports multiple STT/LLM/TTS providers,
**so that** the app can switch providers for cost optimization and testing without code changes.

**Acceptance Criteria:**

1. Voice service module created in `/packages/mobile-app/src/app/services/voice/`
2. Environment configuration file supports provider endpoints: `STT_PROVIDER` (google-cloud | openai-whisper | mock), `LLM_PROVIDER` (openai-gpt4 | groq-llama3 | ollama | mock), `TTS_PROVIDER` (google-cloud | platform-native | mock)
3. Voice service implements three interfaces: `ISpeechToText`, `IIntentRecognition`, `ITextToSpeech`
4. Each interface has multiple implementations (e.g., GoogleCloudSTT, OpenAIWhisperSTT, MockSTT)
5. Provider factory pattern selects implementation based on environment config
6. Mock providers return hardcoded responses for offline development (no API calls)
7. API credentials loaded from `.env` file (GOOGLE_CLOUD_API_KEY, OPENAI_API_KEY, etc.)
8. Voice service handles errors gracefully: network failures, API quota exceeded, invalid credentials
9. Voice service logs provider selection and latency metrics for debugging
10. Unit tests validate each provider implementation independently
11. Integration test validates provider switching via config change (no code recompile needed)
12. README documents environment variable configuration for all providers

### Story 3.2: Microphone Access & Audio Recording

**As a** user,
**I want** to trigger voice recording via button press (push-to-talk),
**so that** I can capture voice commands hands-free while driving or working.

**Acceptance Criteria:**

1. Voice Command Hub (home screen) displays large circular microphone button (Safety Orange #FF6B35)
2. Button shows microphone icon (Material Design solid fill) with label "Tap to Speak"
3. Tap button requests microphone permission (iOS/Android via Capacitor)
4. If permission denied, show error: "Microphone access required. Enable in Settings."
5. Once permission granted, tap starts audio recording (visual feedback: button pulses, label changes to "Listening...")
6. Audio captured via Web Audio API (browser) or Capacitor Audio plugin (native)
7. Recording stops automatically after 10 seconds of silence OR user taps button again
8. Manual stop: Tap button again, label changes to "Processing...", recording ends
9. Recorded audio converted to format required by STT provider (WAV, FLAC, or OGG)
10. While recording, display real-time audio level indicator (waveform or volume bars)
11. Offline mode: Recording works without network, audio buffered for STT when online
12. Error handling: "Recording failed. Check microphone connection."

### Story 3.3: Speech-to-Text Integration

**As a** developer,
**I want** recorded audio transcribed to text via STT provider,
**so that** voice commands can be parsed into structured intents.

**Acceptance Criteria:**

1. Voice service sends recorded audio to configured STT provider (Google Cloud Speech-to-Text primary)
2. STT request includes language parameter (cs-CZ or en-US based on personProfile.language)
3. STT request configured for short utterances (single phrase mode, not streaming for MVP)
4. Response returns transcribed text: e.g., "Set active job to 123"
5. Transcription displayed in real-time on Voice Confirmation Modal as "Heard: [transcription]"
6. If STT fails (network error, API error), display: "Could not transcribe audio. Try again."
7. Retry button allows user to re-record without dismissing modal
8. STT latency tracked: Target <1.5 seconds from recording end to transcription display
9. Mock mode returns predefined transcription instantly (e.g., "set active job to one two three")
10. Czech language transcription tested with sample phrases: "Nastav aktivní práci na 123"
11. Empty transcription (silence detected) shows: "No speech detected. Please try again."
12. Transcription text passed to Intent Recognition service (Story 3.4)

### Story 3.4: Intent Recognition & Slot Filling with LLM

**As a** developer,
**I want** transcribed text parsed into structured intents and entities via LLM,
**so that** voice commands can be executed as app actions.

**Acceptance Criteria:**

1. Intent Recognition service sends transcription to configured LLM provider with structured prompt
2. Prompt template: "Extract intent and entities from: '{transcription}'. Intents: SetActiveJob. Return JSON: {intent, entities}."
3. For "Set active job to 123" (or Czech equivalent), LLM returns: `{intent: "SetActiveJob", entities: {jobIdentifier: "123"}}`
4. jobIdentifier can be: numeric jobNumber ("123"), partial job title ("Smith"), or full title ("Smith, Brno")
5. If jobIdentifier is numeric, query Firestore for job with matching jobNumber
6. If jobIdentifier is text, query Firestore for job with title containing text (case-insensitive, partial match)
7. If multiple jobs match, return first active job (sorted by jobNumber)
8. If no job matches, return error: "Job not found. Please say job number or title."
9. Intent recognition latency target: <1 second for LLM response
10. Mock mode returns hardcoded intent: `{intent: "SetActiveJob", entities: {jobIdentifier: "123"}}`
11. Unrecognized intent returns: "I didn't understand. Try: 'Set active job to [number or name]'"
12. Parsed intent and matched job data passed to Confirmation Loop (Story 3.5)

### Story 3.5: Text-to-Speech Confirmation Loop

**As a** user,
**I want** to hear the app read back my parsed voice command via TTS,
**so that** I can confirm accuracy before data is saved.

**Acceptance Criteria:**

1. Voice Confirmation Modal displays parsed intent in structured format: "✓ Job: **123 - Smith, Brno**"
2. TTS service generates audio from confirmation text: "Setting active job to one two three, Smith, Brno"
3. TTS uses configured provider (Google Cloud Text-to-Speech primary) with language from personProfile
4. Czech TTS voice: cs-CZ-Wavenet-A (female) or cs-CZ-Wavenet-B (male) - user preference in future
5. TTS audio plays automatically when modal displays parsed result
6. While TTS plays, Accept/Retry/Cancel buttons are disabled (visual state: grayed out)
7. After TTS completes (audio ends), buttons become enabled
8. User cannot dismiss modal during TTS playback (prevents accidental interruption)
9. Accept button: Executes action (sets active job), dismisses modal, shows success message
10. Retry button: Clears modal, returns to microphone button ready state (user can re-record)
11. Cancel button: Dismisses modal, no action taken
12. TTS latency target: <2 seconds from intent parse to audio playback start
13. Mock mode plays platform-native TTS (Web Speech API) or skips audio playback
14. Offline mode: TTS queued, plays when connectivity returns (or uses platform-native TTS)

### Story 3.6: Set Active Job Voice Flow (End-to-End)

**As a** craftsman,
**I want** to say "Set active job to [number or name]" and have the app confirm and set the active job,
**so that** subsequent costs are auto-assigned without manual selection.

**Acceptance Criteria:**

1. User taps microphone button, says: "Set active job to 123" (or Czech: "Nastav aktivní práci na 123")
2. STT transcribes → LLM parses → Job #123 queried from Firestore → TTS confirms
3. Voice Confirmation Modal displays: "✓ Job: **123 - Smith, Brno**" with Accept/Retry/Cancel buttons
4. TTS plays: "Setting active job to one two three, Smith, Brno"
5. User taps Accept → Active job state saved to app state (NgRx/Signals) and localStorage for persistence
6. Active Job Banner (persistent, top of screen) updates to show: "[123] Smith, Brno" with Deep Blue background
7. Success message (toast): "Job 123 set as active" (in user's language)
8. Modal dismisses, user returns to Voice Command Hub
9. Active job persists across app restarts (loaded from localStorage on app init)
10. If no active job set, banner shows: "No active job. Tap to select." (tapping opens jobs list)
11. Tapping active job banner opens quick "Change Active Job" modal with jobs list (tap to select new job)
12. Voice flow works fully offline: transcription/intent/TTS use mock mode, active job saved locally
13. End-to-end flow (tap mic → accept) completes in <10 seconds on 4G with real providers
14. Voice flow tested with: numeric job IDs ("123"), partial titles ("Smith"), full titles ("Smith, Brno"), Czech phrases
15. Error handling: "Job 999 not found. Say a valid job number or name."

---

### **Rationale for Epic 3 Stories:**

**Story Sequencing:**
- 3.1: Infrastructure first (provider architecture must exist before implementing features)
- 3.2: Microphone access (can't transcribe without audio input)
- 3.3: STT (converts audio to text)
- 3.4: LLM NLU (converts text to structured intent)
- 3.5: TTS (confirmation loop pattern)
- 3.6: End-to-end integration (validates entire pipeline with Set Active Job)

**Vertical Slice Validation:**
- After 3.2: Audio recording works (can capture voice)
- After 3.3: Transcription works (can see text from voice)
- After 3.4: Intent parsing works (can extract job references)
- After 3.5: Confirmation loop pattern established (reusable for all voice flows)
- After 3.6: First complete voice flow delivers user value (hands-free job switching)

**Mock Mode Importance:**
- Enables offline development (no API costs during feature work)
- Faster iteration (no network latency)
- Playwright tests can validate voice UX without real API calls

**Why Set Active Job First (not Start Journey)?**
- Simpler: No odometer parsing, no vehicle lookup, fewer entities
- Establishes pattern: All voice flows follow same STT→LLM→TTS→Confirm structure
- Foundation for Epic 4: Start Journey requires active job context to work

---

## ⚠️ MANDATORY ELICITATION CHECKPOINT (Epic 3)

**Select 1-9 or just type your question/feedback:**

**1. Proceed to next epic** (Epic 4: Journey Tracking & Cost Management)

**2. Expand or Contract for Audience** – Add voice pipeline diagrams, simplify for PMs

**3. Explain Reasoning (CoT Step-by-Step)** – Why this voice architecture

**4. Critique and Refine** – Challenge story boundaries, missing acceptance criteria

**5. Analyze Logical Flow and Dependencies** – Validate voice pipeline sequence

**6. Identify Potential Risks and Unforeseen Issues** – STT accuracy, LLM costs, latency

**7. Challenge from Critical Perspective** – Is LLM overkill for intent recognition? YAGNI check

**8. Agile Team Perspective Shift** – Mobile dev, voice UX, QA perspectives

**9. Stakeholder Round Table** – Craftsman user, developer, PM viewpoints

---

Select 1-9 or just type your question/feedback:

## Epic 4: Journey Tracking & Cost Management

**Expanded Goal:** Implement the "Start Journey" voice flow with odometer reading parsing and vehicle lookup, create manual cost entry screens for all five cost categories (Transport, Material, Labor, Machine, Other), and enable advances tracking. This epic completes the core cost capture functionality, delivering the full MVP value proposition of hands-free expense recording with manual fallback options.

### Story 4.1: Start Journey Intent Recognition & Entity Parsing

**As a** developer,
**I want** LLM to parse "Start Journey" voice commands extracting destination, vehicle, and odometer reading,
**so that** journey costs can be automatically created under the active job.

**Acceptance Criteria:**

1. Intent Recognition prompt updated to include StartJourney intent alongside SetActiveJob
2. Prompt template: "Extract intent and entities from: '{transcription}'. Intents: SetActiveJob, StartJourney. Return JSON: {intent, entities}."
3. For "I'm going to Brno in Transporter, odometer 12345" (or Czech equivalent), LLM returns: `{intent: "StartJourney", entities: {destination: "Brno", vehicleIdentifier: "Transporter", odometerReading: 12345}}`
4. vehicleIdentifier can be: numeric vehicleNumber ("1"), partial vehicle name ("Trans"), or full name ("Transporter")
5. odometerReading extracted as integer (handles various formats: "12345", "twelve thousand three hundred forty-five", "1 2 3 4 5")
6. If vehicleIdentifier is numeric, query Firestore for vehicle with matching vehicleNumber
7. If vehicleIdentifier is text, query Firestore for vehicle with name containing text (case-insensitive)
8. If multiple vehicles match, return first vehicle (sorted by vehicleNumber)
9. If no vehicle matches, return error: "Vehicle not found. Please say vehicle number or name."
10. If no active job set, return error: "No active job. Please set an active job first."
11. Mock mode returns hardcoded intent: `{intent: "StartJourney", entities: {destination: "Brno", vehicleIdentifier: "1", odometerReading: 12345}}`
12. Parsed intent with matched vehicle data passed to Confirmation Loop

### Story 4.2: Start Journey Voice Flow with Journey Event Creation

**As a** craftsman,
**I want** to say "I'm going to [destination] in [vehicle], odometer [reading]" and have a journey event created,
**so that** I can track transportation costs without manual entry while driving.

**Acceptance Criteria:**

1. User taps microphone button, says: "I'm going to Brno in Transporter, odometer 12345" (or Czech: "Jedu do Brna v Transporteru, kilometr 12345")
2. STT transcribes → LLM parses → Vehicle queried → TTS confirms
3. Voice Confirmation Modal displays: "✓ Journey to **Brno** | Vehicle: **[1] Transporter** | Odometer: **12,345 km**"
4. TTS plays: "Starting journey to Brno in Transporter, odometer one two three four five"
5. User taps Accept → Journey event created in `/users/{tenantId}/jobs/{activeJobId}/events/{eventId}`
6. Event document: ordinalNumber (counter), type: "journey_start", timestamp (now), data: {destination, vehicle: {full vehicle object copy}, odometerStart: 12345, odometerEnd: null, calculatedDistance: null, calculatedCost: null}, createdAt, createdBy (current team member ID)
7. Success message (toast): "Journey to Brno started" (in user's language)
8. Journey event displayed in Job Detail → Events timeline: "[ordinalNumber] Journey to Brno - Transporter - 12,345 km"
9. Voice flow works fully offline: event saved locally, syncs when online
10. End-to-end flow completes in <10 seconds on 4G
11. Error handling: "No vehicles found. Add a vehicle in settings first."
12. If odometer reading missing from transcription, prompt: "Please say odometer reading."

### Story 4.3: Manual Cost Entry - Transport Category

**As a** craftsman,
**I want** to manually enter transportation costs with vehicle and distance details,
**so that** I can record journey costs when voice is unavailable or make corrections.

**Acceptance Criteria:**

1. Job Detail screen has "Add Cost" button → opens category selector (Transport, Material, Labor, Machine, Other)
2. Select "Transport" → opens Transport Cost entry form (job context is implicit - current job from Job Detail screen)
3. Form fields: Vehicle (dropdown showing `[vehicleNumber] Name`), **Input Mode toggle (Distance / Manual Amount)**, Description (optional text), Date/Time (default: now)
4. **Distance Mode:** Distance in km/miles (number with unit label from businessProfile), Amount auto-calculated (distance × vehicle.ratePerDistanceUnit) displayed read-only
5. **Manual Amount Mode:** Amount (number with currency), Distance field hidden, Vehicle still required
6. On save, cost created in `/users/{tenantId}/jobs/{jobId}/costs/{costId}` where jobId is the current job context
7. Cost document: ordinalNumber (counter), category: "transport", amount, vehicle: {full vehicle object copy with vehicleNumber, name, distanceUnit, ratePerDistanceUnit}, distance (null if manual amount mode), description, timestamp, createdAt, createdBy
8. Cost displayed in Job Detail → Costs tab: "[ordinalNumber] Transport - [vehicleNumber] Vehicle Name - X km - Y CZK" (or "- Y CZK" if distance is null)
9. Edit cost: Tap cost item → opens form pre-filled with original mode (distance/manual), allows updates to distance/amount/description/vehicle, job context remains unchanged
10. Delete cost: Swipe left or long-press → confirmation: "Delete cost [ordinalNumber]?" → removes from Firestore
11. Job financial summary updates: Total costs recalculated, budget remaining updated
12. Offline mode: Cost CRUD works without network, syncs when online
13. Validation: If Distance Mode, distance must be positive number and vehicle must be selected; if Manual Amount Mode, amount must be positive number and vehicle must be selected
14. **Use case clarification:** Distance Mode allows user to enter "I drove 50 km today" (without odometer readings), system calculates cost from vehicle rate

### Story 4.4: Manual Cost Entry - Material, Labor, Machine, Other Categories

**As a** craftsman,
**I want** to manually enter costs for materials, labor, machines, and miscellaneous expenses,
**so that** I can track all job-related costs comprehensively.

**Acceptance Criteria:**

1. "Add Cost" category selector includes: Transport (Story 4.3), **Material**, **Labor**, **Machine**, **Other** (job context is implicit - current job from Job Detail screen)
2. **Material form:** **Input Mode toggle (Quantity + Unit Price / Total Amount Only)**, Description (required), Supplier (optional), Date/Time
3. **Material Quantity Mode:** Quantity (number of units), Unit Price (price per unit), Amount auto-calculated (quantity × unitPrice) displayed read-only
4. **Material Total Amount Mode:** Amount (number with currency), Quantity and Unit Price fields hidden
5. Material cost document: ordinalNumber, category: "material", amount, quantity (null if total amount mode), unitPrice (null if total amount mode), description, supplier, timestamp, createdAt, createdBy
6. **Labor form:** Team Member (dropdown `[teamMemberNumber] Name`), **Input Mode toggle (Hours / Manual Amount)**, Description (optional), Date/Time
7. **Labor Hours Mode:** Hours (number), Amount auto-calculated (hours × teamMember.hourlyRate) displayed read-only
8. **Labor Manual Amount Mode:** Amount (number with currency), Hours field hidden, Team Member still required
9. Labor cost document: ordinalNumber, category: "labor", amount, teamMember: {full team member object copy}, hours (null if manual amount mode), description, timestamp, createdAt, createdBy
10. **Machine form:** Machine (dropdown `[machineNumber] Name`), **Input Mode toggle (Hours / Manual Amount)**, Description (optional), Date/Time
11. **Machine Hours Mode:** Hours (number), Amount auto-calculated (hours × machine.hourlyRate) displayed read-only
12. **Machine Manual Amount Mode:** Amount (number with currency), Hours field hidden, Machine still required
13. Machine cost document: ordinalNumber, category: "machine", amount, machine: {full machine object copy}, hours (null if manual amount mode), description, timestamp, createdAt, createdBy
14. **Other form:** Amount (number with currency), Description (required), Date/Time
15. Other cost document: ordinalNumber, category: "other", amount, description, timestamp, createdAt, createdBy
16. All costs saved to `/users/{tenantId}/jobs/{jobId}/costs/{costId}` where jobId is the current job context
17. All categories support Edit (update amount/description/resource, mode preserved from creation, job context remains unchanged) and Delete (with confirmation)
18. Job Detail → Costs tab displays costs grouped by category with sequential ordinalNumbers and totals per category
19. Cost display shows calculated details when available: "Transport - 50 km - 250 CZK" vs "Transport - 250 CZK" (manual), "Material - 10 units × 5 CZK - 50 CZK" vs "Material - 50 CZK" (manual)
20. Privilege enforcement: If current team member has `canAddCosts: false`, "Add Cost" button hidden/disabled

### Story 4.5: Advances Tracking

**As a** craftsman,
**I want** to record client advance payments per job,
**so that** I can track net amounts owed after subtracting advances from total costs.

**Acceptance Criteria:**

1. Job Detail screen has "Advances" tab (alongside Costs, Events)
2. Advances tab displays list: "[ordinalNumber] Amount - Date - Note" with total sum at bottom
3. "Add Advance" button opens advance entry form (job context is implicit - current job from Job Detail screen)
4. Form fields: Amount (required, number with currency from job), Date (default: today), Note (optional, e.g., "Initial deposit", "Payment #2")
5. On save, advance created in `/users/{tenantId}/jobs/{jobId}/advances/{advanceId}` where jobId is the current job context
6. Advance document: ordinalNumber (counter), amount, date, note, createdAt, createdBy
7. Edit advance: Tap item → opens form pre-filled, allows updates to amount/date/note, updatedAt, updatedBy
8. Delete advance: Swipe left → confirmation: "Delete advance [ordinalNumber]?" → removes from Firestore
9. Job Detail header displays financial summary: "Costs: X CZK | Advances: Y CZK | **Net Due: (X - Y) CZK**" (green if positive, red if negative)
10. Net Due calculation updates in real-time as costs/advances added/removed
11. Privilege enforcement: Advances visible only if `canViewFinancials: true`, otherwise tab hidden
12. Offline mode: Advance CRUD works without network, syncs when online

### Story 4.6: Job Costs Summary & Category Breakdown

**As a** craftsman,
**I want** to view job costs broken down by category with totals,
**so that** I can understand where money is being spent per job.

**Acceptance Criteria:**

1. Job Detail → Costs tab displays costs grouped by category: Transport, Material, Labor, Machine, Other
2. Each category section shows: **Category Total: X CZK** with expandable list of individual costs
3. Costs list format: "[ordinalNumber] Description - Amount - Date" with resource info (vehicle/machine/team member if applicable)
4. Tap cost item → opens edit form (Story 4.3/4.4)
5. Top of Costs tab shows **Overall Total: X CZK** (sum of all categories)
6. Budget indicator: If job has budget defined, show progress bar: "X CZK of Y CZK budget (Z% used)" with color coding (green <75%, amber 75-90%, red >90%)
7. Empty state per category: "No transport costs yet. Add or record via voice."
8. Costs sortable by: Date (newest first - default), Amount (highest first), Category
9. Search/filter: Text input filters costs by description/resource name
10. Privilege enforcement: If `canViewFinancials: false`, costs list shows descriptions/dates but amounts are hidden: "Amount: ***"
11. Export to CSV button (future enhancement - show disabled with tooltip: "Coming soon")
12. Offline mode: Costs display from local cache, sync status indicator if pending writes

---

## Epic 5: Audit Logging & Team Privileges

**Expanded Goal:** Implement Cloud Functions audit triggers (onCreate/onUpdate/onDelete) for all collections, scheduled TTL cleanup function, and enforce team member privilege checks in UI and Security Rules. This epic delivers compliance features and multi-user access control, preparing the MVP for team use and regulatory requirements.

### Story 5.1: Cloud Functions Audit Triggers for Jobs & Costs

**As a** system architect,
**I want** Firestore triggers to automatically log all CRUD operations on jobs and costs,
**so that** we have a complete audit trail for compliance and debugging.

**Acceptance Criteria:**

1. Cloud Function `onJobCreate` triggers on `/users/{tenantId}/jobs/{jobId}` onCreate
2. Function writes to `/audit_logs/{logId}` with: operation: "CREATE", collection: "jobs", documentId: jobId, tenantId, timestamp (server time), authorId (from job.createdBy), after: {full job document}, ttl: (timestamp + 1 year)
3. Cloud Function `onJobUpdate` triggers on `/users/{tenantId}/jobs/{jobId}` onUpdate
4. Function writes audit log with: operation: "UPDATE", before: {old job document}, after: {new job document}
5. Cloud Function `onJobDelete` triggers on `/users/{tenantId}/jobs/{jobId}` onDelete
6. Function writes audit log with: operation: "DELETE", before: {deleted job document}, after: null
7. Same pattern for Cost triggers: `onCostCreate`, `onCostUpdate`, `onCostDelete` on `/users/{tenantId}/jobs/{jobId}/costs/{costId}`
8. Audit log documents immutable (Security Rules: write-only from Cloud Functions, no client write access)
9. Triggers execute asynchronously (<500ms execution time, non-blocking to client operations)
10. Local emulator testing: Triggers fire when creating/updating/deleting jobs/costs via app
11. Error handling: If audit write fails, log error but don't block original operation (audit is supplementary)
12. Deployed to Firebase: `firebase deploy --only functions`

### Story 5.2: Cloud Functions Audit Triggers for Resources & Advances

**As a** system architect,
**I want** audit triggers for vehicles, machines, team members, and advances,
**so that** all resource changes are logged for historical tracking.

**Acceptance Criteria:**

1. Cloud Functions created for Vehicles: `onVehicleCreate`, `onVehicleUpdate`, `onVehicleDelete` on `/users/{tenantId}/vehicles/{vehicleId}`
2. Cloud Functions created for Machines: `onMachineCreate`, `onMachineUpdate`, `onMachineDelete` on `/users/{tenantId}/machines/{machineId}`
3. Cloud Functions created for TeamMembers: `onTeamMemberCreate`, `onTeamMemberUpdate`, `onTeamMemberDelete` on `/users/{tenantId}/teamMembers/{teamMemberId}`
4. Cloud Functions created for Advances: `onAdvanceCreate`, `onAdvanceUpdate`, `onAdvanceDelete` on `/users/{tenantId}/jobs/{jobId}/advances/{advanceId}`
5. All triggers follow same pattern as Story 5.1: operation, collection, documentId, tenantId, timestamp, authorId, before/after, ttl
6. Subcollections (costs, advances, events) include parent context in audit log: jobId field for traceability
7. Triggers deployed and tested in emulator: Create/update/delete vehicle → audit log entry created
8. Production deployment: All triggers live and logging operations
9. Monitoring: Cloud Functions console shows successful executions, minimal errors
10. Cost optimization: Triggers use minimal compute (no external API calls, just Firestore writes)

### Story 5.3: Scheduled Audit Log TTL Cleanup Function

**As a** system architect,
**I want** a daily scheduled function to delete audit logs older than 1 year,
**so that** storage costs are controlled while meeting compliance retention requirements.

**Acceptance Criteria:**

1. Cloud Function `scheduledAuditLogCleanup` configured with Cloud Scheduler: runs daily at 2:00 AM UTC
2. Function queries `/audit_logs` where `ttl < now()` (finds expired logs)
3. Function deletes expired logs in batches of 500 (Firestore batch write limit)
4. If >500 expired logs exist, function continues in batches until all deleted
5. Function logs summary: "Deleted X audit logs older than 1 year"
6. Function execution time: <30 seconds for typical load (<10,000 expired logs)
7. Error handling: If batch delete fails, retry with exponential backoff (3 retries max)
8. Scheduled function testable via manual trigger: `firebase functions:shell` or HTTP callable wrapper
9. Production deployment: Function scheduled and running automatically
10. Monitoring: Check Cloud Functions logs weekly to confirm cleanup executes successfully

### Story 5.4: Privilege Enforcement in UI (Cost Entry & Financial Visibility)

**As a** business owner,
**I want** UI to hide/disable features based on team member privileges,
**so that** restricted users cannot access unauthorized functionality.

**Acceptance Criteria:**

1. On app init, load current user's team member record from `/users/{tenantId}/teamMembers` where `authUserId == currentUser.uid`
2. Store privileges (`canAddCosts`, `canViewFinancials`) in app state (NgRx/Signals) for reactive UI updates
3. **canAddCosts = false:** Hide/disable "Add Cost" button, "Add Advance" button, voice microphone button (costs-related commands)
4. **canAddCosts = false:** Cost/Advance edit/delete buttons hidden in lists
5. **canViewFinancials = false:** Job financial summary shows "Budget: ***", "Costs: ***", "Net Due: ***" (amounts hidden)
6. **canViewFinancials = false:** Costs tab displays cost descriptions/dates but amounts replaced with "***"
7. **canViewFinancials = false:** Advances tab entirely hidden (not just amounts)
8. **canViewFinancials = false:** Job budget field hidden in job creation/edit forms
9. Privilege tooltips: When feature disabled, tooltip explains: "Your account doesn't have permission to add costs. Contact business owner."
10. Privilege checks reactive: If business owner updates team member privileges, affected user's UI updates on next app data refresh (within 1 minute or on manual refresh)
11. Team member #1 (owner) always has full privileges regardless of database values
12. If no team member record found for current user, default to no privileges (safety fallback)

### Story 5.5: Firestore Security Rules for Privilege Enforcement

**As a** system architect,
**I want** Security Rules to enforce privilege checks on server-side,
**so that** users cannot bypass UI restrictions via direct API calls.

**Acceptance Criteria:**

1. Security Rules updated to load team member document for request.auth user
2. Rule helper function: `function getTeamMember() { return get(/databases/$(database)/documents/users/$(request.auth.token.tenant_id)/teamMembers/$(request.auth.uid)).data }`
3. **Write access to costs/advances:** `allow create, update, delete: if getTeamMember().privileges.canAddCosts == true`
4. **Read access to financial fields:** Costs/advances readable only if `getTeamMember().privileges.canViewFinancials == true` OR operations are non-financial (descriptions/dates)
5. **Team member #1 bypass:** `allow read, write: if getTeamMember().teamMemberNumber == 1` (owner always has access)
6. **Job budget field protection:** Job writes that change `budget` field require `canViewFinancials == true`
7. Security Rules tested via emulator with multiple test users (owner, restricted user, no-privilege user)
8. Test: User with `canAddCosts: false` attempting to create cost → permission denied
9. Test: User with `canViewFinancials: false` attempting to read advance → permission denied (or sanitized response)
10. Rules deployed: `firebase deploy --only firestore:rules`
11. Production validation: Create test user with restricted privileges, verify API calls respect rules
12. Rules commented explaining privilege logic for future maintainers

---

## Epic 6: Reporting, Export & MVP Polish

**Expanded Goal:** Implement PDF report generation for jobs via Cloud Function, add offline sync status visibility with queue indicators, enhance UI polish (loading states, error messages, empty states), and deliver final QA pass for production readiness. This epic transforms the functional MVP into a polished, production-ready application.

### Story 6.1: PDF Report Generation Cloud Function

**As a** craftsman,
**I want** to export job reports as PDF with costs breakdown and financial summary,
**so that** I can send professional documentation to clients.

**Acceptance Criteria:**

1. Job Detail screen has "Export PDF" button (icon: download/document)
2. Button tap calls Cloud Function (HTTPS callable): `generateJobPDF({jobId})`
3. Function queries job, costs (all categories), advances from Firestore
4. Function uses pdfmake library to generate PDF with:
   - Header: Job title, jobNumber, status, date range (createdAt to now)
   - Costs section: Table with columns: Category, Description, Amount, Date (grouped by category with subtotals)
   - Total Costs: Sum of all costs
   - Advances section: Table with columns: Advance #, Amount, Date, Note
   - Total Advances: Sum of all advances
   - **Net Due: (Total Costs - Total Advances)** highlighted in bold
   - Footer: "Generated by FinDogAI on [date]" with branding
5. Function returns PDF as base64-encoded string or Firebase Storage URL
6. App displays PDF preview (in-app viewer or browser tab)
7. App provides "Share" button → opens native share sheet (email, messaging apps, save to files)
8. Email option pre-fills subject: "Job Report - [jobNumber] [title]" with PDF attached
9. Function execution time: <5 seconds for typical job (<100 costs)
10. Error handling: If function fails, show: "PDF generation failed. Try again later."
11. Offline mode: "Export PDF" button disabled with tooltip: "Requires internet connection"
12. PDF respects privileges: If `canViewFinancials: false`, amounts in PDF are hidden/sanitized (unlikely scenario for PDF export, but safeguarded)

### Story 6.2: Offline Sync Status Visibility

**As a** user,
**I want** clear visual indicators of offline mode and pending sync operations,
**so that** I know when my data is syncing to the cloud.

**Acceptance Criteria:**

1. Top status bar (or within Active Job Banner) displays connectivity indicator: "🟢 Online" or "🟠 Offline"
2. Offline mode shows additional info: "Offline - X items pending sync" (X = queued Firestore writes)
3. When app reconnects, indicator briefly shows: "🔄 Syncing..." with spinner
4. After successful sync: "✓ Synced" message displays for 3 seconds, then returns to "🟢 Online"
5. If sync fails (e.g., Security Rules rejection, network timeout), show: "⚠️ Sync issues - [X] items pending" with "Retry" button
6. Tapping "Retry" button manually triggers Firestore sync attempt
7. Sync queue detailed view (optional expandable): Lists pending operations: "Job #123 - Update pending", "Cost #45 - Create pending"
8. Firestore offline persistence enabled in app init (already required from Epic 1, validating here)
9. Sync status persists across app restarts: Pending writes resume when app reopens
10. Testing: Enable airplane mode, create job/cost, re-enable network → verify sync completes and status updates
11. Sync status uses minimal battery: Checks connectivity passively (network status events, not polling)
12. Empty state: If online with no pending writes, just show "🟢 Online" (no clutter)

### Story 6.3: UI Polish - Loading States & Skeleton Screens

**As a** user,
**I want** smooth loading experiences with skeleton screens instead of spinners,
**so that** the app feels fast and responsive.

**Acceptance Criteria:**

1. Jobs List screen displays skeleton screens (gray pulsing placeholders) while loading from Firestore
2. Skeleton matches job card layout: Rectangle for jobNumber/title, smaller rectangles for status/financial summary
3. Job Detail screen shows skeleton for costs/advances/events tabs while loading
4. Voice Confirmation Modal shows loading spinner during STT/LLM/TTS processing with label: "Transcribing...", "Understanding...", "Confirming..."
5. Form submissions (create/update job, cost, resource) show button loading state: spinner inside button, button disabled, label: "Saving..."
6. Successful operations show brief success feedback (checkmark icon + toast message: "Job created") before dismissing
7. Error states use toast notifications (red background, error icon, clear message) that auto-dismiss after 5 seconds with manual dismiss option (X button)
8. Empty states (no jobs, no costs, no resources) use friendly illustrations + action-oriented text: "No jobs yet. Create your first job!" with large CTA button
9. Image loading (future: if receipts/photos added) uses blur-up technique (tiny thumbnail → full image)
10. All loading states tested: Slow 3G simulation (Chrome DevTools) to verify skeleton screens appear and transitions are smooth
11. No jarring layout shifts (Content Layout Shift - CLS score <0.1 in Lighthouse)
12. Loading states respect accessibility: Screen readers announce "Loading jobs" / "Jobs loaded"

### Story 6.4: Error Handling & User Feedback Polish

**As a** user,
**I want** clear, actionable error messages when things go wrong,
**so that** I know what happened and how to fix it.

**Acceptance Criteria:**

1. Network errors: "Network error. Check your connection and try again."
2. Firestore permission denied errors: "Access denied. Your account may not have permission for this action."
3. Voice recognition errors: "Could not understand. Please try again." (with Retry button)
4. API quota exceeded (STT/LLM/TTS): "Service temporarily unavailable. Try again in a few minutes."
5. Validation errors inline on forms (red text below field): "Job title is required", "Budget must be a positive number"
6. Delete confirmations use clear language: "Delete job #123 'Smith, Brno'? This cannot be undone." with "Cancel" (default focus) and "Delete" (red, requires tap)
7. Unsaved changes warning: If user navigates away from form with edits, show: "Discard unsaved changes?" with Cancel/Discard options
8. Session expiration: If Firebase Auth token expires, show: "Session expired. Please log in again." and redirect to login
9. No generic errors: Replace "Error: undefined" with meaningful messages
10. Error logging: All errors logged to console (dev) or analytics service (prod) for debugging, including stack traces
11. Retry mechanisms: Network-related errors include "Retry" button that re-attempts operation
12. User-tested: Run app with intentional errors (delete non-existent job, exceed API quota via mocking) to validate messages are clear

### Story 6.5: Final QA Pass & Production Readiness Checklist

**As a** PM,
**I want** a comprehensive QA validation of all MVP features,
**so that** we can confidently deploy to production.

**Acceptance Criteria:**

1. **Functional Testing:** All user stories from Epics 1-6 manually tested and passing
2. **Voice Flows:** Both voice flows (Set Active Job, Start Journey) tested with 20+ variations (different phrasings, Czech + English, numeric/text identifiers) - >90% success rate
3. **Offline Testing:** Airplane mode scenarios validated: Create job/cost offline → sync on reconnection → verify data on second device
4. **Multi-Device Sync:** Same tenant logged in on phone + tablet → changes on one device appear on other within 10 seconds
5. **Privilege Enforcement:** Test user with restricted privileges cannot access financial data or add costs (UI + API)
6. **Security Rules:** Attempt cross-tenant data access (User A tries to read User B's jobs) → permission denied
7. **Performance:** Lighthouse audit scores: Performance >80, Accessibility >90, Best Practices >90, SEO >80
8. **PWA Requirements:** App installable, works offline, has app manifest and service worker (Firebase Hosting handles this)
9. **Capacitor Builds:** Android APK builds successfully, installs on test device, all features functional (microphone, offline, etc.)
10. **Error Handling:** No unhandled exceptions in console during 30-minute usage session
11. **Data Integrity:** Sequential IDs (jobNumber, vehicleNumber, etc.) increment correctly without gaps or duplicates across 100+ creates
12. **Audit Logging:** Verify audit logs created for all CRUD operations, TTL cleanup function executes successfully
13. **Browser Compatibility:** Tested on Chrome, Edge, Safari (iOS + desktop) - all features work
14. **Load Testing (Basic):** 10 concurrent users creating jobs/costs → no errors, acceptable latency (<2s for writes)
15. **Production Deployment:** Deploy to Firebase Hosting + Functions, accessible at production URL, all env variables configured
16. **Post-Deployment Smoke Test:** Register new user → create job → add cost via voice → export PDF → all steps succeed
17. **Rollback Plan:** Document rollback procedure if critical bugs found post-launch

---

### **Rationale for Epics 4-6:**

**Epic 4 (Journey Tracking & Cost Management):**
- Completes the "cost capture" value proposition (voice + manual for all categories)
- Start Journey is the most valuable voice flow (addresses driving safety + odometer tracking pain point)
- Manual cost entry provides fallback and correction mechanism (critical for MVP trust)
- Advances tracking rounds out job financials (costs - advances = net due)

**Epic 5 (Audit Logging & Compliance):**
- Backend-heavy, can be developed in parallel with Epic 4 UX work
- Audit triggers are "set and forget" (minimal ongoing maintenance)
- Privilege enforcement prepares MVP for team use (Phase 2 readiness)
- Compliance features (audit logs, GDPR-ready) are table stakes for EU market

**Epic 6 (Polish & Production Readiness):**
- PDF export is minimum viable billing artifact (replaces complex invoicing for MVP)
- Offline sync visibility addresses user anxiety ("Is my data saved?")
- UI polish transforms functional MVP into production-quality app
- QA checklist ensures confident launch (no critical bugs slip through)

**Sequential Dependencies:**
- Epic 4 requires Epic 3 (voice pipeline) and Epic 2 (resources)
- Epic 5 can start after Epic 2 (data model exists) - parallel to Epic 4
- Epic 6 requires Epics 1-5 complete (polishes the complete feature set)

---

**Epics 4, 5, and 6 complete!** The PRD now contains the full MVP story breakdown with 30+ user stories spanning foundation → data → voice → costs → compliance → polish.

**Next steps:**
1. Execute PM Checklist
2. Generate handoff prompts for UX Expert and Architect

Ready to proceed? Type **1** to continue with PM Checklist execution.