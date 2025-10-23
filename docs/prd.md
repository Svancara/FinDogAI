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
    - vehicleNumber (counter), name, ratePerKm, ratePerMile
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

