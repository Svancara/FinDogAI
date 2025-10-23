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

