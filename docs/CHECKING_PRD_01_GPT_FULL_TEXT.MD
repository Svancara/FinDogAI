
I will open and read docs/prd.md to analyze its contents for logical errors, incompatibilities, broken flows, data model issues, missing information, ambiguities, technical feasibility, and security/privacy concerns.


I’ll load the full PRD in three chunks to review it comprehensively and efficiently.



## PRD Review Findings (docs/prd.md)

Below are issues organized by severity. For each, I include: section/area, description, impact, and suggested fix.

### Critical

1) Firestore Security Rules vs UI masking requirements
- Where: Story 4.6 (Costs Summary), Story 5.4 (Privilege Enforcement in UI), Story 5.5 (Security Rules)
- Issue: The PRD expects restricted users to read costs/advances with amounts masked (“Amount: ***”), while reading other fields. Firestore Security Rules cannot selectively redact fields: document reads are all-or-nothing.
- Impact: Either restricted users will still see amounts (if reads are allowed), or the UI will break (if reads are denied) because no partial-field redaction is possible at the server level.
- Suggestion:
  - Choose one pattern and update the PRD:
    - Split data: separate “public” vs “financial” docs/collections; allow restricted users to read only the public docs.
    - Materialized sanitized views: Cloud Functions write sanitized summaries that restricted users can read.
    - Deny reads for restricted users: UI shows only non-sensitive, derived lists without document reads.

2) Data model conflict: embed vs reference for costs/events
- Where: “CRITICAL ARCHITECTURAL PRINCIPLE” (Epic 2, line ~618) vs Firestore Data Model (lines ~291–299)
- Issue: The PRD states costs/events will never reference resources and must store full copies. But the data model shows costs with a resourceId field.
- Impact: Confusion in implementation and data integrity, risk of mixed approaches, and potential referential issues on delete.
- Suggestion:
  - Make embedded resource copies canonical; remove resourceId from costs/events or mark it explicitly as optional, non-authoritative metadata.
  - Define the embedded resource schema and versioning policy (e.g., store name, number, rate fields; keep a resourceSnapshotVersion).

3) GDPR/Privacy posture for external providers (STT/LLM/TTS)
- Where: Voice Providers (lines ~351–356), NFR9 (EU region), FR23 (export), Story 3.7 (error logs)
- Issue: Firebase is EU-resident, but STT/LLM/TTS providers (OpenAI/Google) may process data outside the EU. No explicit policy on data residency, retention, DPA status, or disallowing model training. Story 3.7 logs “transcription” and “intent” into audit logs, potentially storing PII for 1 year.
- Impact: GDPR compliance risk; potential cross-border transfers without clarity; storage of voice content in audit logs increases exposure.
- Suggestion:
  - Specify EU-region endpoints or EU-only processing where possible; sign DPAs; disable provider retention/training.
  - Do not log raw transcriptions or intents in audit logs; log only minimal error metadata (codes, categories, timestamps). If needed, store transient logs with short TTL and exclude from export.

### Major

4) Person Profile stored under tenant path
- Where: Data Model personProfile (lines ~313–315), Story 2.6
- Issue: personProfile is tenant-scoped, but it contains user PII and global preferences (language, AI toggle). Multi-tenant users will have duplicated or inconsistent personal data across tenants.
- Impact: Confusing UX for users who belong to multiple tenants; privacy duplication; increased risk of stale data.
- Suggestion:
  - Move canonical user profile to /users/{uid}/profile; mirror only tenant-specific overrides under tenants if required (e.g., tenant locale overrides).
  - Clarify which preferences are global vs tenant-specific and how conflicts are resolved.

5) Data export delivered via email
- Where: FR23
- Issue: Export is delivered via email, potentially as an attachment or direct link.
- Impact: Email attachments with full-tenant JSON present security/privacy risk; email is not a secure transport layer and can be forwarded or stored indefinitely.
- Suggestion:
  - Generate a short-lived signed URL (Storage download token) and require authenticated download; optionally protect behind app auth; avoid emailing full data directly.

6) Units mismatch in examples/UX
- Where: Story 4.2 (odometer “km” fixed string), several TTS/UI examples
- Issue: The PRD shows fixed “km” in readbacks/UI strings, but businessProfile can be miles.
- Impact: User confusion and incorrect confirmations in imperial-unit contexts.
- Suggestion:
  - Parameterize units everywhere (UI and TTS); add formatting rules for distances and currency across locales.

7) Ambiguous voice disambiguation behavior
- Where: Story 3.4/3.6 (“if multiple jobs match, return first active job”)
- Issue: Returning the “first match” for ambiguous titles can cause wrong-job selection without user awareness.
- Impact: Misfiled costs/journeys; trust erosion.
- Suggestion:
  - Require explicit disambiguation if >1 match: TTS lists top 2–3 candidates, or ask for job number. Keep “first match” only for exact numeric IDs.

8) Audit logs retention vs right-to-erasure policy not fully reconciled
- Where: FR8 (1-year TTL), NFR12 (cascade delete includes audit logs)
- Issue: The PRD states both a 1-year TTL and cascade deletion for GDPR. Legal bases/retention exceptions aren’t discussed.
- Impact: Potential legal conflict or accidental removal of data needed for compliance/audits—or conversely, retention beyond user expectations.
- Suggestion:
  - Clarify legal basis for retention, whether erasure requests override TTL, and when exceptions apply. Document in Compliance Requirements.

9) Streaming vs non-streaming STT discrepancy
- Where: Background (line ~22 references streaming STT), Story 3.3 (non-streaming for MVP)
- Issue: Background suggests streaming STT; MVP story mandates non-streaming.
- Impact: Expectation misalignment with stakeholders; performance assumptions (latency) may differ.
- Suggestion:
  - Clarify explicitly: MVP uses non-streaming STT; streaming is a Phase 2 performance optimization. Update Goals/Background to avoid implying streaming in MVP.

10) Journey/Transport cost fallback without vehicle rate
- Where: Story 4.7 (End Journey fallback: amount null if missing rate)
- Issue: Accepting a transport cost with amount: null can break financial summaries without clear handling.
- Impact: Incomplete financial data; confusing totals.
- Suggestion:
  - Define UI “needs pricing” queue; block PDF/export until resolved; or prevent cost creation without rate and guide the user to set rate first.

### Minor

11) “Cascade delete” wording
- Where: NFR12
- Issue: The term “cascade delete functionality for user data deletion” could be read as fully automated deletion. Some data may require exceptions (legal obligations), and invitations/memberships may be logically separate.
- Impact: Ambiguity in expected deletion scope and timing.
- Suggestion:
  - Spell out objects included/excluded; define SLA; link to right-to-erasure policy.

12) JWT example roles not used elsewhere
- Where: User-to-Tenant Mapping (lines ~339–346)
- Issue: JWT example shows roles array but the PRD relies on membership documents for privileges.
- Impact: Minor confusion.
- Suggestion:
  - Remove roles from the example or note they are illustrative only; privileges come from membership docs.

13) KWS licensing/footprint
- Where: Voice Providers (Porcupine/Vosk)
- Issue: No mention of licensing, wake-word model training, or privacy posture for on-device KWS.
- Impact: Legal/packaging gotchas later.
- Suggestion:
  - Add a licensing note and verify on-device-only processing; document wake-word customization approach.

14) iOS caveat for native builds
- Where: Epic 1.6 (builds), Platforms section
- Issue: iOS builds require macOS/Xcode; constraints not stated.
- Impact: Surprises in CI or dev onboarding.
- Suggestion:
  - Note macOS requirement for iOS builds and planned build pipeline approach.

15) Accessibility “no haptic feedback required”
- Where: UX paradigms
- Issue: Blanket statement could reduce accessibility for some users; haptics can aid eyes-free interaction.
- Impact: Small UX miss.
- Suggestion:
  - Consider optional haptic feedback with a toggle.

16) Numeric formatting and localization
- Where: Multiple readback examples (thousands separators, decimal commas vs dots)
- Issue: Examples mix separators; no formatting rules defined for cs-CZ vs en-US.
- Impact: Minor confusion.
- Suggestion:
  - Define locale-aware number/currency formatting in NFRs.

17) Invite code UX
- Where: Stories 1.7–1.8
- Issue: “Code not stored, only hash” is good, but no rate limits for redemption flow mentioned (only creation).
- Impact: Brute force risk (low, but present).
- Suggestion:
  - Rate-limit redemption attempts and consider App Check or device fingerprinting.

18) Export scope vs privileges
- Where: FR23
- Issue: Export includes “audit logs” and all financials regardless of who triggers export?
- Impact: Potential privilege bypass if not restricted.
- Suggestion:
  - Specify export requires owner; include server-side check.

## Summary of Overall Quality and Completeness

- Strong, detailed PRD with clear goals, epics, and acceptance criteria.
- Key contradictions to resolve:
  - Field-level access masking vs Firestore Rules capability.
  - Costs/events schema: embed vs reference.
  - Privacy/GDPR specifics for STT/LLM/TTS, including audit logging of transcriptions.
  - Tenant-scoped personProfile design for multi-tenant users.
- After addressing the above, the PRD is implementable for the MVP timeline.

I created a summary document at the repository root:
- CHECKING_PRD_01_GPT.MD (overall assessment, top risks, and recommended next actions)

If you want, I can update the PRD sections with tracked changes (suggested edits) to resolve the critical items.
