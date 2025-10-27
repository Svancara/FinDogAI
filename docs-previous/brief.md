# Project Brief: FinDogAI

## Executive Summary

FinDogAI is a voice-controlled mobile AI assistant designed to help small entrepreneurs and craftsmen record job costs and advances in real-time with minimal manual data entry. The application addresses the critical pain point of end-of-day cost recall by enabling hands-free voice capture while driving or working on-site, built with an offline-first architecture using Firebase/Firestore for seamless multi-device synchronization. The MVP focuses on two core voice flows—setting an active job and recording journey costs with odometer readings—to prove the voice-first, offline-capable value proposition for solo craftsmen in the European market.

## Problem Statement

### Current State and Pain Points

Small entrepreneurs and craftsmen face a persistent, costly problem: **remembering and accurately recording all costs, purchases, mileage, and work hours at the end of each workday**. This manual recall process creates several cascading issues:

- **Revenue leakage:** Forgotten expenses and unbilled work hours directly reduce profitability (estimated 5-15% of billable costs go unrecorded)
- **Cognitive burden:** After a full day of physical work, mental energy for detailed data entry is minimal
- **Data entry friction:** Mobile keyboard typing while tired leads to errors and incomplete records
- **Delayed documentation:** End-of-day entry means losing contextual details (exact odometer readings, material quantities, store names)
- **Safety concerns:** Attempting to type notes while driving between job sites is dangerous

### Impact Quantification

For a solo craftsman handling 10-15 jobs per month with average job value of €2,000-€5,000:
- Unrecorded costs: €200-€500 per month in lost reimbursements
- Time waste: 30-60 minutes daily on manual data entry (10-20 hours/month)
- Error correction: Additional time reconciling receipts with vague memory

### Why Existing Solutions Fall Short

Current approaches fail to meet the unique needs of mobile craftsmen:

- **Generic expense trackers** (Expensify, Mint) require manual typing and don't support job-specific cost allocation or multi-device offline sync
- **Voice assistants** (Siri, Google Assistant) lack domain-specific intent recognition and don't integrate with job management workflows
- **Job management software** (Jobber, Housecall Pro) focus on scheduling/invoicing but lack real-time voice capture and robust offline operation
- **Note-taking apps** with voice recording create transcripts but don't extract structured data (amounts, odometer readings, job assignments)

### Urgency and Importance

The post-pandemic shift toward solo entrepreneurship and gig economy work has increased the population of craftsmen operating without administrative support. Additionally:
- Rising fuel and material costs make accurate expense tracking financially critical
- Tax authorities increasingly require detailed documentation
- Competition forces thin margins where every unbilled cost matters
- Safety regulations and liability concerns around distracted driving are tightening

## Proposed Solution

### Core Concept and Approach

FinDogAI combines **voice-first interaction**, **offline-first architecture**, and **AI-powered natural language understanding** to create a hands-free cost tracking assistant optimized for mobile craftsmen. The solution enables users to speak naturally while driving ("I'm going to Brno to handle Smith's order, odometer is 12345") and receive intelligent confirmation before data is persisted.

**Key Technical Approach:**
- **On-device keyword spotting (KWS)** for optional wake-word activation ("Hey FinDog") to enable truly hands-free operation
- **Streaming Speech-to-Text (STT)** with multilingual support (Czech, English priority)
- **Intent recognition and slot filling** using LLM-based NLU to parse user utterances into structured job events
- **Confirmation loop with Text-to-Speech (TTS)** to ensure accuracy before persisting data
- **Firestore offline persistence** with automatic background sync when connectivity returns
- **Active job context** that auto-assigns subsequent costs to the currently selected job

### Key Differentiators

1. **True hands-free operation:** Optional wake-word detection eliminates need to unlock phone or press buttons while driving
2. **Offline-first by design:** Full functionality without network connectivity, unlike cloud-dependent competitors
3. **Domain-optimized NLU:** Purpose-built intent recognition for craftsman-specific entities (jobs, vehicles, materials, odometer readings)
4. **Conversational confirmation:** TTS readback provides safety and accuracy verification through audio-only interface
5. **Multi-tenant from day one:** Built for potential SaaS model with team expansion capability

### Why This Solution Will Succeed

**Market timing:** Solo craftsmen are increasingly tech-comfortable (smartphone penetration >90% in target demographics) but underserved by existing vertical SaaS.

**Focused MVP:** By narrowing MVP scope to two voice flows (set active job + journey tracking), we can achieve production-quality voice UX without the complexity of comprehensive feature coverage.

**Technology maturity:** Recent advances in on-device voice processing, affordable cloud STT/TTS APIs, and Firestore's offline capabilities make this technically feasible with modest resources.

**Clear pain point:** Unlike many "nice-to-have" productivity tools, this directly addresses revenue leakage—a painful, measurable problem with obvious ROI.

### High-Level Product Vision

Users open the app in the morning, say "Set active job to Smith, Brno," and throughout the day simply speak their costs as they occur. The app handles transcription, intent parsing, confirmation, and persistence—even when driving through areas without cell coverage. At day's end, the user has a complete, accurate cost record ready for export to PDF or email to the client.

## Target Users

### Primary User Segment: Solo Mobile Craftsmen

**Demographic Profile:**
- Age: 30-55 years old
- Occupation: Independent tradespeople (electricians, plumbers, carpenters, cleaners, landscapers, renovation contractors)
- Geography: Czech Republic, Slovakia, Austria, Germany (Central European focus)
- Tech comfort: Smartphone users, comfortable with apps like WhatsApp and Google Maps
- Business size: Solo operators or very small teams (1-3 people)

**Current Behaviors and Workflows:**
- Manage 5-20 active jobs simultaneously with varying stages of completion
- Drive 50-200 km daily between job sites and supplier visits
- Purchase materials from multiple retailers (sometimes 3-5 stops per day)
- Work 8-12 hour days with limited breaks for administrative tasks
- Currently use: paper notes, receipts in glove box, mental recall, basic spreadsheets

**Specific Needs and Pain Points:**
- **Safety:** Cannot safely type while driving but need to capture trip details immediately
- **Accuracy:** Odometer readings, exact material costs, and timestamps must be precise for billing and tax purposes
- **Simplicity:** No time or patience for complex UI navigation or data entry forms
- **Reliability:** Need offline capability because job sites and rural routes often lack connectivity
- **Job context:** All costs must be correctly attributed to specific jobs for proper client billing

**Goals They're Trying to Achieve:**
- Increase billable revenue by capturing 100% of reimbursable costs
- Reduce administrative overhead (save 30+ minutes per day)
- Improve safety by eliminating distracted driving from note-taking
- Provide professional, detailed cost reports to clients
- Simplify tax documentation and bookkeeping

### Secondary User Segment: Small Trade Teams (Post-MVP)

**Profile:**
- Small businesses with 2-5 team members
- Crew lead assigns jobs and tracks team labor across multiple simultaneous job sites
- Need multi-user coordination and consolidated reporting

**Note:** This segment is explicitly out-of-scope for MVP but informs architecture decisions (multi-tenant design, team member resources in data model).

## Goals & Success Metrics

### Business Objectives

- **MVP Launch:** Achieve production-ready MVP within 6 months with core voice flows operational
- **Early Adoption:** Acquire 50 active users (dogfooding + beta) within first 3 months post-launch
- **Validation Metrics:** Achieve 70%+ weekly retention and 80%+ voice flow success rate among beta users
- **Revenue Model Validation:** Convert 20% of trial users to paid subscription ($10-15/month) by month 6
- **Geographic Expansion:** Validate Czech language voice accuracy >90%, establish English as second language

### User Success Metrics

- **Cost Capture Rate:** Users record 90%+ of actual job costs (measured via user surveys and dogfooding)
- **Time Savings:** Reduce daily administrative time by 50% (from 30-60 min to <15 min)
- **Voice Flow Completion:** 80%+ success rate for "Start Journey" and "Set Active Job" flows (technical telemetry)
- **Offline Reliability:** 95%+ of offline-created records successfully sync within 10 seconds of reconnection
- **User Satisfaction:** Net Promoter Score (NPS) >40 among active users

### Key Performance Indicators (KPIs)

- **Voice Recognition Accuracy (STT):** <5% error rate for short utterances in low-noise conditions, measured via transcription comparison
- **Intent Classification Accuracy (NLU):** >90% correct intent detection for core flows (StartJourney, SetActiveJob, AddMaterial)
- **Voice Flow Time-to-Completion:** <30 seconds median time from wake-word to confirmed data persistence
- **Offline Sync Success Rate:** >98% of queued writes successfully commit when online (measured via client telemetry)
- **Active Job Context Usage:** 70%+ of costs/events leverage active job auto-assignment (indicates users adopt the workflow)
- **Weekly Active Users (WAU):** Track engagement; target 3+ sessions per week for active craftsmen
- **Export Usage:** 60%+ of completed jobs exported to PDF within 7 days of completion (indicates real billing usage)

## MVP Scope

### Core Features (Must Have)

- **Voice Flow: Set Active Job**
  User says "Set active job to [job name/description]"; system confirms and sets application state so all subsequent costs auto-assign to that job. Works fully offline. _Rationale: This is the workflow multiplier—once set, all other voice commands become simpler._

- **Voice Flow: Start Journey with Odometer**
  User says "I'm going to [destination] in [vehicle], odometer is [number]"; system parses intent, asks clarifying questions if needed, confirms via TTS, and creates journey event under active job. Works fully offline. _Rationale: Transportation costs are the most frequent mobile data entry pain point._

- **Manual Entry Screens for Costs**
  UI forms to create/edit/delete costs (Transport, Material, Labor, Machine, Other) with job assignment, amounts, descriptions, timestamps. _Rationale: Voice is primary UX but manual editing is necessary for corrections and cases where voice is inappropriate._

- **Jobs Management (CRUD)**
  Create, view, edit, archive jobs with fields: jobNumber (auto-assigned via Firestore counter), title, description, status (active/completed/archived), currency, vatRate, budget. Includes subcollections for costs, advances, events. _Rationale: Jobs are the organizing principle for all cost data._

- **Business Profile Defaults**
  One-time setup: currency (ISO 4217), VAT rate, distance unit (km/miles). These become defaults for new jobs but can be overridden per-job. _Rationale: Reduces repetitive data entry for consistent business parameters._

- **Resources Management: Team Members (Self), Vehicles, Machines**
  Users define resources with properties (name, hourly rate for labor/machines, km/mile rate for vehicles). Minimum one team member (the user themselves) is mandatory. _Rationale: Enables cost calculation (e.g., journey km × vehicle rate) and supports intent recognition ("Transporter" → specific vehicle)._

- **Advances Tracking**
  Per-job subcollection of client payments with auto-assigned ordinalNumber using Firestore counter. UI displays sum of advances vs sum of costs. _Rationale: Craftsmen often receive partial payments and need to track net amounts owed._

- **Authentication & Multi-tenant Isolation**
  Firebase Auth (email/password minimum); Firestore Security Rules enforce tenantId-scoped reads/writes; all data stored under `/users/{tenantId}/` paths. _Rationale: Hard requirement for data security and future SaaS model._

- **Offline-first with Firestore Persistence**
  Enable Firestore offline persistence; all MVP flows (voice and manual) fully operational without network; automatic background sync with visible sync status indicators. _Rationale: Core differentiator and critical for reliability in field conditions._

- **Basic PDF Export**
  Generate simple PDF report for a job showing: job title, cost breakdown by category, sum of advances, net balance. Email option via user's default mail client. _Rationale: Minimum viable billing artifact for clients; avoids building complex invoicing._

- **Czech and English Language Support**
  UI localization and voice recognition for both languages; user selects preferred language in profile. Czech is default. _Rationale: Target market is Central Europe with Czech priority; English enables broader testing and future expansion._

### Out of Scope for MVP

- Invoicing and payment processing (Stripe integration, invoice templates, payment tracking)
- Team collaboration features (role-based permissions, shared job assignments, audit trails)
- Advanced analytics and dashboards (trend reports, profitability analysis, forecasting)
- Receipt scanning with OCR (smart expense categorization from photos)
- GPS-based automatic distance tracking (using device location instead of manual odometer)
- Custom wake-word configuration UI (MVP uses fixed wake phrase or push-to-talk only)
- Subscription management and payment infrastructure (Stripe billing)
- Multi-language support beyond Czech and English
- Machine labor tracking (start/stop machine work flows)—demoted to post-MVP
- Advanced security features (2FA, biometric auth, SSO)
- Export formats beyond basic PDF (Excel, QuickBooks integration, API access)
- Sharded/distributed Firestore counters (not needed for expected traffic)

### MVP Success Criteria

**The MVP is successful if:**

1. **Voice Flow Validation:** 10 dogfooding users can complete both core voice flows (Set Active Job + Start Journey with Odometer) end-to-end on mobile devices with >80% success rate across 100+ attempts.

2. **Offline Reliability:** In airplane mode testing, users can execute both voice flows; upon reconnection, data appears on a second logged-in device within 10 seconds without user intervention, in 95%+ of test cases.

3. **Real-world Adoption Signal:** 5 users outside the development team use the app for 14+ consecutive days, recording 20+ cost entries each, and report in exit interviews that it saved them time and captured costs they would have forgotten.

4. **Technical Foundations:** STT/NLU pipeline demonstrates <5% error rate for core intents in Czech language with low-background-noise conditions; Firebase multi-tenant security rules pass penetration testing with no cross-tenant data leakage.

5. **Export Completeness:** Users successfully export at least 3 completed jobs to PDF with accurate cost summaries and report PDFs are "good enough" to send to clients.

## Post-MVP Vision

### Phase 2 Features

- **Receipt Scanning with OCR:** Capture photos of receipts; AI extracts line items, amounts, merchant names, and suggests assignment to active job or material cost entries.
- **GPS-based Distance Tracking:** Optional automatic journey recording using device GPS; calculates distance and prompts user to confirm trip details when arriving at destination.
- **Machine Labor Tracking:** Voice flows for "Start excavator" / "Stop excavator" with automatic cost calculation based on machine hourly rates.
- **Team Collaboration (Basic):** Support 2-5 team members per tenant; crew lead can assign jobs and view team labor costs; basic role permissions (admin vs team member).
- **Enhanced Export Options:** Excel spreadsheets, CSV for import into accounting software, customizable PDF templates with branding.
- **Subscription Tiers:** Implement Stripe integration with Trial (30 days, 5 jobs max), Basic ($10/month, unlimited jobs, 1 user), Premium ($25/month, team features, advanced reporting).

### Long-term Vision (12-24 months)

**FinDogAI evolves from a personal cost-tracking assistant into a comprehensive job management platform for small trade businesses:**

- **Full Invoicing & Payments:** Generate professional invoices directly from job costs; integrate with Stripe or local payment processors for client billing and online payment acceptance.
- **Advanced Analytics:** Profitability dashboards, cost trends over time, job-type analysis, budget vs actual reporting, predictive insights for bidding.
- **Client Portal:** Lightweight client-facing interface where customers can view job progress, approve costs in real-time, and receive automated status updates.
- **Expanded Integrations:** QuickBooks, Xero, DATEV accounting exports; calendar sync for job scheduling; CRM-lite for managing client contacts and job history.
- **Multi-language Expansion:** German, Polish, Slovak, Hungarian language support for broader Central/Eastern European market.

### Expansion Opportunities

- **Vertical Specialization:** Tailored versions for specific trades (electrician-specific inspection checklists, plumber warranty tracking, landscaper seasonal planning).
- **Geographic Markets:** After Central Europe validation, expand to similar markets (UK, Scandinavia, North America) with localized voice models and currency support.
- **Platform Extensions:** Capacitor native builds for iOS/Android with deeper OS integration (Siri Shortcuts, Android Auto, wearable support for Apple Watch/Wear OS).
- **B2B SaaS Model:** Franchise or contracting companies license FinDogAI for their distributed field workforce (e.g., 50-500 technician deployments).

## Technical Considerations

### Platform Requirements

- **Target Platforms:** Mobile-first (iOS and Android) via Capacitor hybrid build; Progressive Web App (PWA) as fallback with degraded voice capabilities
- **Browser/OS Support:**
  - iOS 15+ (Safari, native WebView)
  - Android 10+ (Chrome, native WebView)
  - Desktop browsers (Chrome, Edge, Safari) for admin/reporting tasks (optional secondary use case)
- **Performance Requirements:**
  - Post-wake-word to first STT token: <1.5 seconds on mid-range Android device
  - Round-trip voice confirmation (user speaks → TTS responds): <3 seconds on typical 4G network
  - Offline write operations: instant local persistence with <100ms UI feedback
  - Sync on reconnect: queued writes commit to Firestore within 10 seconds (median)

### Technology Preferences

- **Frontend:** Angular 20+ with Ionic Framework for mobile UI components; Capacitor for native builds
- **Backend:** Firebase ecosystem (Auth, Firestore, Cloud Functions, Storage)
  - **Rationale:** Firestore's offline-first architecture and automatic sync are purpose-built for this use case; eliminates need for custom backend sync logic
- **Database:** Cloud Firestore with offline persistence enabled; Security Rules for multi-tenant isolation
  - **Data Model:** `/users/{tenantId}/` root with subcollections for Jobs, Vehicles, TeamMembers, Machines
  - **Counters:** FieldValue.increment() for sequential jobNumber/teamMemberNumber/vehicleNumber/machineNumber (per tenant) and ordinalNumber (per job for costs/advances/events)
- **Hosting/Infrastructure:**
  - Firebase Hosting for PWA delivery
  - Cloud Functions (Node.js) for server-side operations (PDF generation, counter management, validation)
  - Firebase Storage for images (receipts, job photos) with tenant-prefixed paths

### Architecture Considerations

- **Repository Structure:** Monorepo with separate packages for mobile app (Angular/Ionic), cloud functions, shared types, and e2e tests
  - **Rationale:** Simplifies cross-cutting changes and shared logic; manageable for MVP team size

- **Service Architecture:** Hybrid model—thin serverless (Cloud Functions) for specific operations (PDF generation, Firestore trigger validations), primarily client-side logic leveraging Firestore offline capabilities
  - **Rationale:** Minimizes backend complexity and cost for MVP; Firestore Security Rules handle most authorization logic

- **Voice Pipeline Integration:**
  - Optional on-device KWS (Porcupine, Vosk, or platform-native) for wake-word detection
  - STT provider: Google Cloud Speech-to-Text (primary for Czech), OpenAI Whisper (fallback/comparison testing)
  - NLU/Intent: LLM-based (OpenAI GPT-4, Groq Llama, OpenRouter, or Ollama) with structured output prompting for entity extraction
  - TTS provider: Google Cloud Text-to-Speech (Czech voices), platform-native TTS as fallback
  - **Configurable providers:** Developer can swap STT/LLM/TTS endpoints via environment config for cost optimization

- **Integration Requirements:**
  - Firebase SDK (Auth, Firestore, Storage, Functions)
  - @angular/fire (official AngularFire library)
  - Capacitor plugins: @capacitor/network (connectivity status), @capacitor/filesystem (local audio storage for voice memos)
  - Voice SDKs: Google Cloud client libraries (STT/TTS), HTTP clients for LLM APIs
  - PDF generation: Cloud Functions using pdfmake or similar library

- **Security/Compliance:**
  - **Multi-tenant isolation:** Firestore Security Rules enforce `request.auth.token.tenant_id == tenantId` for all reads/writes under `/users/{tenantId}/`
  - **Storage isolation:** Images/receipts stored under `/tenants/{tenantId}/media/` with signed URL access only
  - **Privacy:** No raw audio stored by default (transient STT requests only); optional diagnostics mode with explicit opt-in
  - **GDPR/DSGVO readiness:** User data deletion via Cloud Function (cascade delete all tenant docs); export functionality for data portability
  - **Data residency:** Use Firebase Europe (Belgium) region for Firestore and Storage to comply with EU data protection requirements

## Constraints & Assumptions

### Constraints

- **Budget:** Bootstrap/self-funded MVP; minimize recurring costs by staying within Firebase free tier where feasible; voice API costs (STT/TTS/LLM) estimated at $0.10-0.30 per user per day during active usage
- **Timeline:** 6-month target for production-ready MVP with core voice flows; assumes part-time development effort (evenings/weekends)
- **Resources:** Solo developer or 2-person team (developer + PM/tester); no dedicated UX designer (leverage Ionic defaults); no dedicated DevOps (Firebase handles infrastructure)
- **Technical:**
  - Must use Firebase/Firestore (architectural foundation already chosen)
  - Voice recognition limited by quality of available STT models for Czech language
  - On-device KWS feasibility depends on Capacitor plugin availability and device capabilities (may require custom native modules)
  - Hybrid app performance subject to WebView limitations (not pure native)

### Key Assumptions

- **Target users own smartphones:** 95%+ of solo craftsmen in target market have Android or iOS devices
- **Acceptable voice accuracy:** Czech STT accuracy of 85-90% is "good enough" for MVP when combined with confirmation loops
- **Offline duration:** Typical offline periods are 15 minutes to 2 hours (not days); users reconnect at least once per work day for sync
- **Firebase scalability:** Firestore can handle expected MVP load (50-100 users × 20-50 writes/day) without sharding counters
- **Willingness to pay:** Solo craftsmen will pay $10-15/month for a tool that saves 30+ minutes/day and captures unbilled costs
- **Voice UX adoption:** Users are comfortable speaking to their phone (WhatsApp voice messages are widely used in target demographic)
- **Internet connectivity:** Users have mobile data plans (4G minimum) for online portions of workflow; offline is exception-handling, not primary mode
- **Language coverage:** Czech + English covers 80%+ of addressable Central European market for MVP validation

## Risks & Open Questions

### Key Risks

- **Voice Recognition Accuracy (High Impact, Medium Likelihood):** Czech language STT models may have 10-15% error rates for domain-specific terms (material names, street names, client surnames), leading to user frustration. _Mitigation: Implement robust confirmation loops with TTS readback; allow quick voice or manual corrections; iterate on custom vocabulary/pronunciation dictionaries._

- **Offline Sync Conflicts (Medium Impact, Low Likelihood):** If user edits same cost entry on two devices while offline, Firestore's last-write-wins may cause data loss. _Mitigation: Use FieldValue.increment() for counters (atomic); design UI to minimize concurrent editing scenarios; implement optimistic UI with conflict detection and manual resolution prompts._

- **On-device KWS Reliability (Medium Impact, Medium Likelihood):** Wake-word detection may have high false positive/negative rates, battery drain concerns, or platform limitations (iOS background restrictions). _Mitigation: Make wake-word optional (default to push-to-talk); extensive device testing; power management tuning; clear user education on battery impact._

- **User Adoption Friction (High Impact, Medium Likelihood):** Initial setup (create vehicles, set rates, explain active job concept) may be too complex for non-technical users. _Mitigation: Guided onboarding wizard with sensible defaults; optional sample job pre-populated with demo data; short video tutorials; in-app contextual help._

- **Cost of Voice APIs (Medium Impact, Medium Likelihood):** STT/TTS/LLM usage may exceed budget if users generate 50+ voice interactions per day. _Mitigation: Implement client-side request batching where possible; use cheaper STT tiers for short utterances; provide fallback to platform-native (free) voice APIs; monitor per-user API costs and implement usage caps for free tier._

- **Market Validation (High Impact, High Likelihood):** Assumed pain point may not be strong enough to drive adoption; users may prefer existing manual workflows. _Mitigation: Extensive dogfooding and early beta testing (10+ real craftsmen); iterate based on feedback; pivot scope if voice UX doesn't resonate but manual tracking with offline sync is valuable._

### Open Questions

- **What is the optimal wake-word phrase?** ("Hey FinDog" vs "Hey Assistant" vs no wake-word, push-to-talk only?) → Requires user testing for memorability and low false-positive rates
- **How do we handle noisy environments?** (Construction sites, vehicle cabins with engine noise) → Test various STT noise suppression settings; may need external microphone or Bluetooth headset recommendation
- **Should we support multiple active jobs simultaneously?** (User working on two job sites in one day) → Investigate whether "switch active job" voice command is sufficient or if we need job context stacking
- **What level of offline intelligence is feasible?** (Can we run small LLM on-device for offline intent recognition, or fall back to keyword grammar only?) → Depends on model size, device capabilities, and acceptable latency
- **How do we handle currency conversion for cross-border work?** (Czech craftsman doing job in Austria with EUR costs) → Out of MVP scope but may need earlier than Phase 2 if common
- **What is the optimal confirmation UX?** (Always confirm via TTS, or allow "auto-confirm" mode for trusted flows after training period?) → A/B test in beta
- **Should PDF export happen on client or server?** (Client-side JS library vs Cloud Function with headless browser) → Trade-off between offline capability and rendering quality/consistency

### Areas Needing Further Research

- **Czech Language STT Benchmarking:** Comparative testing of Google Cloud Speech-to-Text vs Azure vs OpenAI Whisper for Czech recognition accuracy with domain-specific vocabulary
- **Competitor Feature Analysis:** Deep dive into Jobber, Housecall Pro, Fergus, ServiceTitan to identify differentiation gaps and potential partnership opportunities
- **User Workflow Shadowing:** Spend 2-3 days with actual craftsmen to observe real cost-recording behaviors, identify edge cases, and validate assumed pain points
- **Firebase Cost Modeling:** Detailed cost projection based on realistic usage patterns (reads, writes, storage, functions invocations, bandwidth) to validate free tier feasibility
- **Accessibility Compliance:** Understand WCAG AA requirements for voice-controlled UX, screen reader compatibility, and large-button driving mode design
- **Internationalization Strategy:** Research locale-specific requirements for number formatting, date/time, currency symbols, address formats across target markets
- **Legal/Tax Documentation Requirements:** Understand what cost documentation is legally required for tax filings in Czech Republic, Slovakia, Austria, Germany (may inform required fields and export formats)

## Appendices

### A. Research Summary

**Market Research Conducted:**
- Reviewed 15+ expense tracking apps, voice assistant capabilities, and trade-specific SaaS platforms
- Analyzed App Store reviews for Jobber, Housecall Pro, Expensify (1,200+ reviews sampled)
- Identified consistent complaints: offline unreliability, complex UI, lack of voice input, poor mobile optimization

**Competitive Landscape:**
- No direct competitor offers combination of voice-first + offline-first + trade-specific NLU
- Closest: QuickBooks Self-Employed (expense tracking but no voice, poor offline), Veryfi (receipt OCR but no job context)
- Opportunity: Blue ocean in Central European mobile craftsman segment

**Technical Feasibility Studies:**
- Prototyped Firestore offline sync with 100-write queue → validated <10s sync time on reconnect
- Tested Google Cloud Speech-to-Text with Czech audio samples → 87% accuracy baseline (improvable with custom vocabulary)
- Evaluated Porcupine wake-word detection on Android → 2-3% false positive rate, <1% battery drain per hour

### B. Stakeholder Input

- **Early User Interviews (3 craftsmen):** Confirmed odometer tracking is highest-friction pain point; expressed skepticism about voice accuracy but excited to try; emphasized need for simple, large-button UI for work-glove use
- **Technical Advisor Feedback:** Recommended Firestore over custom PostgreSQL backend for MVP speed; cautioned about KWS battery concerns; suggested progressive enhancement (start with push-to-talk, add wake-word later)

### C. References

- Initial Project Description: `docs/Initial_project_description.md`
- Firebase Offline Persistence: https://firebase.google.com/docs/firestore/manage-data/enable-offline
- Autoincrement Counter Strategy: `docs/autoincrement_counter_in_firestore.md`
- External Voice Research: `docs/ExternalDocs/` (STT, TTS, wake-word detection findings)
- Angular + Ionic Documentation: https://ionicframework.com/docs/angular/overview
- Capacitor Plugins: https://capacitorjs.com/docs/plugins

## Next Steps

### Immediate Actions

1. **Finalize Project Brief** – Review this document with stakeholders; incorporate feedback; mark as v1.0 approved
2. **Set up Development Environment** – Initialize Angular 20 + Ionic project; configure Firebase project (Europe region); create monorepo structure
3. **Prototype Voice Pipeline** – Build isolated proof-of-concept for STT → LLM intent parsing → TTS confirmation loop with Czech test cases
4. **Design Core Data Model** – Implement Firestore schema (`/users/{tenantId}/Jobs`, `Vehicles`, etc.) with Security Rules; test offline persistence
5. **Create Wireframes** – Sketch 5-7 key screens (Main Dashboard, Jobs List, Voice Input, Cost Entry, Settings) for developer reference
6. **Begin PM Handoff** – Schedule PRD kickoff session with PM agent to translate this brief into detailed requirements and epic/story breakdown

### PM Handoff

This Project Brief provides the full context for **FinDogAI**. Please start in 'PRD Generation Mode', review the brief thoroughly to work with the user to create the PRD section by section as the template indicates, asking for any necessary clarification or suggesting improvements.

**Key areas for PRD to expand:**
- Functional requirements list (FR1, FR2...) derived from core features
- Non-functional requirements (NFR1, NFR2...) extracted from constraints and technical considerations
- Detailed epic breakdown with user stories and acceptance criteria
- UI/UX design goals for voice-first interaction patterns
- Architecture handoff prompt for technical design document
