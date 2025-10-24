# FinDogAI PRD Technical & Product Review

## Critical Blocking Issues

### 1. **Voice Pipeline Performance Claims Are Unrealistic**

**Issue:** NFR1 claims "<1.5s post-wake-word to first STT token" on mid-range Android devices, but this ignores fundamental constraints:

- **Network latency:** Google Cloud STT requires 200-500ms just for network round-trip to europe-west1
- **Audio buffering:** Mobile devices need 500-1000ms minimum audio buffer for quality STT
- **Processing overhead:** JSON parsing, intent recognition, UI updates add 200-300ms
- **Real-world conditions:** Car cabin noise, engine vibration, poor 4G signal significantly degrade performance

**Reality Check:** Achievable target is 3-5 seconds, not 1.5 seconds.

### 2. **Offline Voice Pipeline Is Technically Impossible As Specified**

**Issue:** FR13 + FR16-17 claim "all voice flows function without network connectivity" but the architecture depends on cloud STT/LLM/TTS services.

**Contradictions:**
- STT requires Google Cloud Speech-to-Text (cloud service)
- LLM intent recognition uses OpenAI GPT-4 (cloud service)  
- TTS uses Google Cloud Text-to-Speech (cloud service)
- "Mock mode" for development ≠ functional offline voice recognition

**Missing:** On-device STT/LLM models that can actually parse Czech craftsman terminology.

### 3. **Czech Language Accuracy Claims Lack Foundation**

**Issue:** NFR6 claims ">90% intent classification accuracy" and goals mention ">90% Czech voice accuracy" without considering:

- **Domain-specific terminology:** Craftsman jargon, tool names, location names in Czech
- **Noisy environments:** Construction sites, vehicle cabins, outdoor work
- **Accent variations:** Regional Czech dialects across Central Europe
- **Code-switching:** Czech-English mixing common in technical contexts

**Missing:** Realistic accuracy targets based on domain-specific testing.

## Technical Feasibility Issues

### 4. **Firestore Counter-Based Sequential IDs Will Fail Under Load**

**Issue:** FR18 specifies `FieldValue.increment()` for sequential numbering, but this creates race conditions:

```typescript
// This will fail with concurrent users
await doc.update({ jobNumber: FieldValue.increment(1) });
```

**Problems:**
- Multiple users creating jobs simultaneously = duplicate numbers
- Offline queue replay = number gaps and conflicts
- No atomic counter guarantees in Firestore

**Solution Needed:** Distributed ID generation strategy or accept non-sequential IDs.

### 5. **Multi-Tenant Security Rules Are Incomplete**

**Issue:** FR10 mentions "tenantId-scoped isolation" but the data model shows `/users/{tenantId}/` where `tenantId = user.uid`.

**Security Gaps:**
- Team members need access to same tenant data, but each has different `user.uid`
- Custom claims for `tenant_id` mentioned in Story 1.3 but not in security model
- No mechanism for team member invitation/authorization
- Audit logs accessible by all team members (privacy violation)

### 6. **Offline Sync Conflict Resolution Missing**

**Issue:** FR13 promises "automatic background sync" but provides no conflict resolution strategy.

**Scenarios Not Addressed:**
- User A and B both edit Job #123 offline, then reconnect
- Deleted job on device A, updated on device B
- Sequential ID conflicts from offline operations
- Partial sync failures (some documents succeed, others fail)

## Product Logic & Workflow Problems

### 7. **Voice Flows Don't Match Real Craftsman Workflows**

**Issue:** The two MVP voice flows (Set Active Job + Start Journey) miss critical daily operations:

**Missing Voice Flows:**
- "Add material cost" (most frequent operation)
- "Record work hours" (daily requirement)
- "End journey" (complete the travel loop)
- "Quick expense" (coffee, tolls, parking)

**Current Flows Are Secondary:**
- Setting active job happens once per day
- Journey tracking is subset of transport costs

### 8. **Hands-Free Operation Claims Are Overstated**

**Issue:** FR17 requires "confirmation loop using TTS" but this breaks hands-free operation:

**Real-World Problems:**
- User driving, hears TTS confirmation, needs to respond "Accept" or "Retry"
- Voice response while driving = safety hazard
- Gloved hands can't operate "Accept/Retry/Cancel buttons"
- Engine noise interferes with TTS playback

**Missing:** True hands-free confirmation (e.g., "Say 'yes' to confirm").

### 9. **Active Job Context Model Is Flawed**

**Issue:** FR1 assumes craftsmen work on one job at a time, but reality is more complex:

**Real Scenarios:**
- Pick up materials for Job A, drive to Job B, work on Job C
- Emergency call interrupts planned job
- Multiple small jobs in one day
- Shared resources (vehicle) across multiple jobs

**Missing:** Multi-job context switching and cost allocation rules.

## Implementation Gaps

### 10. **Error Handling Scenarios Are Completely Missing**

**Critical Gaps:**
- Voice recognition fails (background noise, accent, technical terms)
- Network timeout during voice confirmation
- Firestore write fails due to security rules
- Device storage full (offline queue)
- Battery dies during voice operation
- User speaks in wrong language

### 11. **PDF Generation Requirements Are Underspecified**

**Issue:** FR14 mentions "basic PDF reports" but lacks essential details:

**Missing Specifications:**
- PDF template/layout requirements
- Multi-language support (Czech/English)
- Currency formatting (CZK vs EUR)
- VAT calculation display
- Digital signature requirements (EU invoicing)
- File size limits for email attachment

### 12. **Team Member Privilege System Is Incomplete**

**Issue:** FR11 mentions "privilege toggles" but the model is too simplistic:

**Missing Privilege Types:**
- View vs. edit vs. delete permissions
- Job-specific permissions (can only see assigned jobs)
- Time-based permissions (temporary access)
- Approval workflows (manager must approve large expenses)

## Business Model Inconsistencies

### 13. **MVP Scope vs. 6-Month Timeline Mismatch**

**Issue:** 6 epics with 30+ user stories is not achievable in 6 months for 1-2 developers:

**Epic Complexity Analysis:**
- Epic 1: 2-3 weeks (infrastructure)
- Epic 2: 4-5 weeks (data model + CRUD)
- Epic 3: 6-8 weeks (voice pipeline - most complex)
- Epic 4: 4-5 weeks (cost tracking)
- Epic 5: 3-4 weeks (compliance)
- Epic 6: 2-3 weeks (polish)

**Total:** 21-28 weeks = 5-7 months (optimistic, no testing/debugging time)

### 14. **Revenue Leakage Reduction Claims Lack Validation**

**Issue:** Goals claim to "eliminate 5-15% revenue leakage" but the solution may not address root causes:

**Actual Leakage Sources:**
- Forgetting to bill for materials (not just recording costs)
- Underestimating time (not just tracking hours)
- Not charging for travel time/mileage
- Informal "small favors" that become unpaid work

**Missing:** Integration with invoicing/billing systems.

### 15. **70%+ Weekly Retention Target Is Unrealistic**

**Issue:** Consumer app retention benchmarks don't apply to B2B productivity tools:

**Reality Check:**
- Craftsmen use tools sporadically (not daily like social apps)
- Learning curve for voice commands reduces initial retention
- Offline-first means less engagement tracking
- Small user base makes percentage metrics volatile

## Priority Fixes Before Architecture Phase

### **Immediate Blockers (Must Fix):**

1. **Redefine offline voice capabilities** - Specify which voice features work offline vs. require connectivity
2. **Realistic performance targets** - Update latency requirements based on network/processing constraints
3. **Complete security model** - Define team member authentication and data access patterns
4. **Conflict resolution strategy** - Specify how offline sync conflicts are handled

### **Critical Clarifications Needed:**

1. **Voice flow error handling** - Define fallback behaviors for recognition failures
2. **Multi-job workflow** - Clarify how craftsmen switch between jobs during the day
3. **True hands-free confirmation** - Replace visual confirmation with voice-only flows
4. **Sequential ID generation** - Choose distributed ID strategy that works offline

### **Missing Essential Requirements:**

1. **Data export/backup** - GDPR compliance requires data portability
2. **Invoice integration** - Cost tracking without billing integration has limited value
3. **Expense categorization** - Tax deduction categories for business expenses
4. **Multi-currency support** - Cross-border work common in Central Europe

### **Scope Reduction Recommendations:**

**Phase 1 (3 months):** Core data model + manual entry + basic voice (online only)
**Phase 2 (6 months):** Offline sync + advanced voice flows
**Phase 3 (9 months):** Multi-language + team features + reporting

The current PRD attempts to solve too many problems simultaneously. Focus on proving voice-assisted cost entry works reliably before adding offline complexity and multi-tenant features.
