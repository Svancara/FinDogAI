# Architecture Design Request: FinDogAI MVP

## Context
You are receiving a comprehensive PRD for FinDogAI, a voice-first mobile application for craftsmen to capture job costs hands-free. The PRD has been validated and is ready for architectural design.

## PRD Location
The complete PRD is sharded at: `docs/prd/`
- Start with: `docs/prd/index.md` for navigation
- Requirements: `docs/prd/requirements.md` (23 FRs, 15 NFRs)
- Technical decisions: `docs/prd/technical-assumptions.md`
- Epics 1-6: Individual epic files with detailed stories

## Key Technical Decisions Already Made

### Confirmed Architecture:
- **Monorepo structure** (Angular/Ionic/Capacitor + Cloud Functions + shared types)
- **Firebase Backend** (europe-west1 for GDPR)
- **Offline-first** with Firestore sync
- **Client-heavy architecture** (business logic in app, minimal backend)
- **Multi-tenant isolation** via Security Rules
- **Voice pipeline**: STT → LLM → TTS with configurable providers

### Technology Stack:
- Frontend: Angular 20+, Ionic 8+, Capacitor
- Backend: Firebase (Auth, Firestore, Functions, Storage)
- Voice: Google Cloud STT/TTS, OpenAI GPT-4 for NLU
- Testing: Jest, Playwright (NOT Cypress)
- Node.js v20 LTS

## Areas Requiring Deep Architecture Analysis

### 1. Voice Pipeline Optimization
- Review `docs/prd/epic-3-voice-pipeline-active-job-context.md`
- Design for <8s round-trip latency with fallback providers
- Battery optimization for continuous KWS
- Mock provider architecture for development

### 2. Offline Sync & Conflict Resolution
- Review FR19 in `docs/prd/requirements.md`
- Design LWW conflict resolution at document level
- Sequential ID allocation (online + offline paths)
- Sync queue visibility and error recovery

### 3. Multi-tenant Data Model
- Review Firestore structure in `docs/prd/technical-assumptions.md`
- Optimize for offline performance with subcollections
- Design audit log triggers without blocking operations
- Plan for cascade deletes (GDPR)

### 4. Security Architecture
- Membership-based isolation patterns
- Role enforcement (owner/representative/teamMember)
- Audit logging compliance (1-year TTL)
- Data export for GDPR

### 5. Performance Requirements
- Voice latency: ≤3s median STT, ≤8s round-trip
- Offline writes: <100ms feedback
- Sync on reconnect: <10s for queued writes
- Czech STT accuracy: WER ≤15% median

## Critical Technical Risks to Address

1. **Czech Language STT Accuracy**
   - Mitigation strategies if WER targets not met
   - Fallback to platform-native STT
   - Custom vocabulary/models needed?

2. **Voice API Costs**
   - $0.10-0.30/user/day estimate
   - Cost optimization strategies
   - Provider switching architecture

3. **iOS Build Dependencies**
   - macOS/Xcode requirements
   - CI/CD pipeline design
   - TestFlight distribution strategy

4. **Offline Queue Size**
   - Device storage constraints
   - Queue management strategies
   - Partial sync failure handling

## Deliverables Expected

1. **System Architecture Document**
   - High-level component diagram
   - Voice pipeline sequence diagram
   - Data flow diagrams
   - Deployment architecture

2. **Technical Design Decisions**
   - Detailed Firestore data model
   - Security Rules implementation plan
   - Cloud Functions architecture
   - State management approach (NgRx vs Signals)

3. **Implementation Roadmap**
   - Technical dependencies between epics
   - Risk mitigation timeline
   - Performance testing strategy
   - Monitoring & observability plan

4. **Development Environment Setup**
   - Local development with Firebase Emulators
   - Voice provider mocking
   - Multi-device testing approach
   - CI/CD pipeline design

## Questions to Answer

1. How to handle voice commands when multiple jobs have similar names?
2. Should we implement request queuing for voice API calls?
3. What's the strategy for progressive enhancement (basic → voice features)?
4. How to handle regulatory changes in data retention requirements?
5. Should audit logs be partitioned for better query performance?

## Success Criteria for Architecture

- Supports 100+ concurrent users without degradation
- Voice features degrade gracefully when offline
- Sync conflicts resolve predictably without data loss
- Meets all NFR performance targets
- Scalable to 10,000+ users with minimal changes
- Development velocity: 1-2 devs can deliver in 6 months

## Next Steps

1. Review the complete PRD at `docs/prd/`
2. Focus on Epic 1 (Foundation) and Epic 3 (Voice Pipeline) first
3. Identify any missing technical requirements
4. Propose architecture with rationale for key decisions
5. Flag any PRD elements that need clarification

## Additional Context

- PRD completeness: 92% (validated by PM checklist)
- MVP scope: Epics 1-4 + essential parts of 5-6
- Phase 2: PDF generation, OAuth, advanced permissions
- Timeline: 6 months with 1-2 developers
- Primary market: Czech Republic craftsmen

## PRD Validation Summary

### Strengths
- Exceptionally comprehensive with 23 detailed functional requirements
- Outstanding epic/story structure with clear dependencies
- Excellent voice UX design with dual confirmation modes
- Thorough offline handling throughout
- Smart MVP scoping with clear Phase 1 vs Phase 2 separation

### Minor Gaps (Non-blocking)
- Stakeholder alignment documentation not included
- Revenue leakage validation methodology needs specifics
- Czech STT accuracy risk mitigation strategies
- User research findings not summarized
- Competitive analysis mentioned but not detailed

### Review Status
- **Overall Completeness:** 92%
- **MVP Scope:** Just Right for 6-month timeline
- **Architecture Readiness:** READY - Can proceed immediately
- **Technical Decisions:** Clear with good rationales
- **Risk Identification:** Comprehensive with metrics

## Contact

For PRD clarifications or updates, consult:
- PRD Version: v1.1 (2025-10-23)
- Location: `docs/prd/` (sharded structure)
- Original: `docs/prd.md` (monolithic backup)