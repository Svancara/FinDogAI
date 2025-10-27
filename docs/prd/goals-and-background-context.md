# Goals and Background Context

### Goals

- Enable solo mobile craftsmen to capture 100% of job costs in real-time through hands-free voice interaction
- Reduce daily administrative overhead from 30-60 minutes to under 15 minutes
- Provide offline-first cost tracking that works reliably without network connectivity
- Reduce revenue leakage from forgotten expenses by ≥5% in pilot (baseline 5–15%), measured via capture completeness and near-real-time entry metrics; full elimination requires invoicing/billing integration (Phase 2)
- Deliver production-ready MVP with core voice flows (Set Active Job, Start Journey) plus high-frequency capture commands (End Journey, Add Material Cost, Record Work Hours, Quick Expense) within 6 months
- MVP scope for 6 months (1–2 devs): Epics 1–4 + essential slices of Epics 5 and 6 only (5.1 audit triggers, 5.5 Security Rules; 6.2 offline sync status, 6.3 conflict resolution, 6.6 final QA). Defer advanced privileges, OAuth, PDF generation (Issue #11), and deep UI polish to Phase 2.

- Achieve 80%+ voice flow success rate; pilot engagement target: ≥50% 30-day active rate among onboarded users (B2B-appropriate), measured on a rolling 30-day window
- Validation of leakage reduction (MVP): Track proxy KPIs (capture completeness: % of costs captured same day; journey→transport cost pairing rate; % of days with zero end-of-day backlog). Compare pre/pilot baseline vs 4-week pilot. Invoicing/billing impact deferred to Phase 2.
- Establish baseline Czech voice accuracy using domain test sets: STT WER ≤15% (median) and ≤25% (P95) in controlled conditions; intent F1 ≥0.85 for core flows; English supported as secondary

### Background Context

Small entrepreneurs and craftsmen in Central Europe face a persistent profitability problem: accurately remembering and recording all costs, purchases, mileage, and work hours at day's end. This manual recall process creates revenue leakage (estimated 5-15% of billable costs go unrecorded), cognitive burden after long workdays, and safety concerns from attempting to type notes while driving. Current solutions—generic expense trackers, voice assistants, and job management software—fail to address the unique combination of needs: hands-free voice capture, offline-first reliability, domain-specific intent recognition for craftsman workflows, and job-specific cost allocation.

FinDogAI combines voice-first interaction, offline-first Firebase/Firestore architecture, and AI-powered natural language understanding to create a hands-free assistant optimized for mobile craftsmen. The MVP focuses on foundational voice flows (Set Active Job + Start Journey with Odometer) and includes high-frequency capture commands (End Journey, Add Material Cost, Record Work Hours, Quick Expense) to match daily workflows. The solution leverages on-device keyword spotting for true hands-free operation, non-streaming (short-utterance) speech-to-text in MVP (streaming planned for Phase 2) with multilingual support, LLM-based intent recognition, and conversational TTS confirmation—production voice flows require network STT/LLM/TTS; offline development uses mock providers; when offline in the field, voice interactions are disabled and the app operates without AI support via manual flows; on-device KWS and platform-native TTS may operate offline for availability notifications; automatic sync when connectivity returns.

### Change Log

| Date | Version | Description | Author |
|------|---------|-------------|--------|
| 2025-10-23 | v1.0 | Initial PRD creation from approved Project Brief | John (PM) |
| 2025-10-23 | v1.1 | Incorporated remarks: audit logging, team member privileges, sequential ID voice references | John (PM) |
