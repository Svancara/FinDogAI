# PRD Review Summary — FinDogAI (CHECKING_PRD_01_GPT)

Date: 2025-10-26
Reviewer: Augment Agent (GPT‑5)
Scope: Review of docs/prd.md for logical consistency, feasibility, completeness, security/privacy, and data model alignment.

## Overall Assessment
Status: Amber (good structure and coverage, but a few critical issues to resolve before implementation)

The PRD is comprehensive, well-structured, and actionable: goals are concrete, acceptance criteria are detailed, and epics/stories form a coherent execution plan. The offline-first stance, voice-first UX, security rules intent, and testing strategy are strong. However, there are several critical items that must be addressed prior to development:

- Firestore Security Rules cannot provide field-level redaction as described (amounts hidden but other fields readable). This requires a data model and access control redesign.
- Data model inconsistency between “embed full resource copies” vs “resourceId” references in costs/events.
- GDPR/Privacy gaps for external providers (STT/LLM/TTS): region, data residency, DPA, retention, and logging of transcriptions need explicit policies.
- Multi-tenant handling of personProfile (stored under tenant) will duplicate user PII and creates confusing behavior for users in multiple tenants.

## Strengths
- Clear MVP scope with phased deferrals; measurable KPIs/NFRs (latency, WER, F1).
- Detailed acceptance criteria for voice flows and offline behavior with good error handling.
- Sensible monorepo and serverless architecture for MVP scale.
- Testing scope across unit/integration/E2E and emulator coverage.

## Key Gaps/Risks (Top 5)
1) Field-level access control mismatch (Security Rules vs UI expectations).
2) Costs/Events data model: embed vs reference contradiction.
3) GDPR/data residency and logging policy for voice providers (OpenAI/Google, etc.).
4) Person Profile under tenant (multi-tenant duplication, privacy confusion).
5) Data export delivery via email (security: large sensitive JSON as email attachment/link) needs a secure distribution approach.

## Readiness
- Implementable after addressing the above gaps.
- Recommended to resolve design decisions (access control + data model + privacy policy) before sprinting on Epics 2, 4, and 5.

## Severity Breakdown (count)
- Critical: 3–4
- Major: 6–8
- Minor: 6–10

## Recommended Next Actions (Pre‑Implementation)
1) Access control redesign: choose one of these patterns and update PRD:
   - Separate “public” vs “financial” documents/collections; or
   - Cloud Function–materialized sanitized views; or
   - Fully deny read of cost docs to restricted users (UI shows only derived non-sensitive lists).
2) Costs/Events schema: remove resourceId or clearly mark it as optional metadata only; define the canonical embedded resource shape and versioning.
3) Privacy/Data protection addendum:
   - Pick EU endpoints/regions where possible; document data flow diagrams; include DPAs; disable provider data retention/training; redact PII; don’t log raw audio/transcriptions in audit logs.
4) Person Profile: move to /users/{uid}/profile (or mirror minimal tenant-scoped preferences only). Clarify cross-tenant behavior.
5) Secure data export: deliver via authenticated downloadable link with short-lived signed URL; avoid email attachments of full exports.
6) Unit handling: ensure distance/odometer labels match businessProfile units in all UI/TTS.

## Conclusion
The PRD is close to implementation-ready. Resolve the critical access-control and data model contradictions and formalize privacy posture for external providers; after that, the plan is strong and executable for a 6‑month MVP.

