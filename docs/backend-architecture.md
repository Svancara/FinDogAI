# Backend Architecture Document: FinDogAI MVP

## Document Information

**Version:** 1.0
**Date:** 2025-10-28
**Author:** System Architect
**Status:** Draft
**Related Documents:**
- PRD: `docs/prd/index.md`
- UI Architecture: `docs/ui-architecture.md`
- Architect Handoff: `docs/architect-handoff.md`

---

## 1. Introduction

### 1.1 Purpose

This document provides comprehensive architectural specifications for the FinDogAI backend system, a serverless Firebase-based infrastructure supporting a voice-first mobile application for craftsmen to capture job costs hands-free.

### 1.2 Scope

The backend architecture covers:
- Firebase services configuration and integration
- Firestore data model and multi-tenant isolation
- Cloud Functions architecture and implementation
- Security rules and authentication patterns
- API integration and connectivity
- Performance optimization strategies
- Deployment and operational procedures

### 1.3 Audience

- Backend developers implementing Cloud Functions
- Frontend developers integrating Firebase SDK
- DevOps engineers managing deployment pipelines
- Security auditors reviewing GDPR compliance
- Product managers understanding technical constraints

### 1.4 Architectural Principles

1. **Offline-First**: Firestore offline persistence ensures app functionality without network
2. **Client-Heavy**: Business logic resides primarily in the mobile app
3. **Serverless**: Cloud Functions handle only server-side operations (sequential IDs, audit logging, PDF generation)
4. **Multi-Tenant Isolation**: Security Rules enforce tenant-level data segregation
5. **GDPR Compliance**: All data stored in EU region (europe-west1) with export/deletion support
6. **Cost Optimization**: Minimize Cloud Functions invocations; leverage free tier where possible
7. **Graceful Degradation**: Voice features degrade to manual entry when offline

---

## 2. High Level Architecture

### 2.1 Firebase Services Overview

FinDogAI leverages four core Firebase services:

#### Firebase Authentication
- **Purpose**: User identity management
- **Region**: Global (identity tokens)
- **Features Used**:
  - Email/password authentication (MVP)
  - Custom claims for tenant ID injection
  - OAuth providers (Phase 2: Google, Apple)
- **Multi-Tenancy**: Custom claim `tenant_id` added after user-to-tenant mapping resolution

#### Cloud Firestore
- **Purpose**: Primary database for tenant data
- **Region**: europe-west1 (Belgium) for GDPR compliance
- **Features Used**:
  - Offline persistence with automatic sync
  - Real-time listeners for UI updates
  - Subcollections for hierarchical data (jobs > costs, jobs > advances)
  - Composite indexes for queries
  - Security Rules for authorization
- **Data Model**: Tenant-scoped collections under `/tenants/{tenantId}/`

#### Cloud Functions (2nd Gen)
- **Purpose**: Server-side business logic
- **Region**: europe-west1 (co-located with Firestore)
- **Runtime**: Node.js 20
- **Functions**:
  - **onCreate/onUpdate/onDelete triggers**: Audit logging
  - **onCreate triggers**: Sequential number assignment for offline-created entities
  - **HTTPS callable**: `allocateSequence` (transactional ID allocation)
  - **HTTPS callable**: `processVoiceCommand` (LLM integration for voice NLU) - Phase 2
  - **HTTPS callable**: `generatePDF` (PDF report generation) - Phase 2
  - **HTTPS callable**: `exportData` (GDPR data export)
  - **Scheduled function**: `cleanupAuditLogs` (TTL enforcement, daily)

#### Cloud Storage for Firebase
- **Purpose**: File storage for exports and attachments
- **Region**: europe-west1
- **Features Used**:
  - Signed URLs for secure download
  - Lifecycle policies for automatic cleanup (exports expire after 7 days)
  - Security Rules for access control

### 2.2 System Architecture Diagram Description

```
┌─────────────────────────────────────────────────────────────────┐
│                        Mobile App (Angular/Ionic)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Voice Module │  │ State Mgmt   │  │ Offline Queue│          │
│  │ (STT→LLM→TTS)│  │ (NgRx/Signals)│  │ (IndexedDB)  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│         └──────────────────┼──────────────────┘                  │
│                            │                                      │
└────────────────────────────┼──────────────────────────────────────┘
                             │ Firebase SDK
                             │ (AngularFire)
                             │
┌────────────────────────────┼──────────────────────────────────────┐
│                  Firebase Services (europe-west1)                 │
│                            │                                      │
│  ┌─────────────────────────▼──────────────────────────┐          │
│  │              Cloud Firestore                       │          │
│  │  ┌──────────────────────────────────────────┐      │          │
│  │  │ /tenants/{tenantId}/                     │      │          │
│  │  │   ├─ members/                            │      │          │
│  │  │   ├─ jobs/{jobId}/                       │      │          │
│  │  │   │   ├─ costs/                          │      │          │
│  │  │   │   ├─ advances/                       │      │          │
│  │  │   │   └─ events/                         │      │          │
│  │  │   ├─ vehicles/                           │      │          │
│  │  │   ├─ machines/                           │      │          │
│  │  │   ├─ teamMembers/                        │      │          │
│  │  │   ├─ audit_logs/                         │      │          │
│  │  │   └─ businessProfile (doc)               │      │          │
│  │  │ /user_tenants/{uid}/memberships/         │      │          │
│  │  │ /sequences/{tenantId}/counters/          │      │          │
│  │  └──────────────────────────────────────────┘      │          │
│  │                      ▲                              │          │
│  └──────────────────────┼──────────────────────────────┘          │
│                         │                                         │
│  ┌──────────────────────▼──────────────────────────┐             │
│  │       Cloud Functions (Node.js 20)              │             │
│  │  ┌────────────────────────────────────────┐     │             │
│  │  │ Firestore Triggers:                    │     │             │
│  │  │  - auditLogger (onCreate/Update/Delete)│     │             │
│  │  │  - assignSequence (onCreate)           │     │             │
│  │  ├────────────────────────────────────────┤     │             │
│  │  │ HTTPS Callable:                        │     │             │
│  │  │  - allocateSequence                    │     │             │
│  │  │  - exportData                          │     │             │
│  │  │  - generatePDF (Phase 2)               │     │             │
│  │  ├────────────────────────────────────────┤     │             │
│  │  │ Scheduled:                             │     │             │
│  │  │  - cleanupAuditLogs (daily at 2 AM)    │     │             │
│  │  └────────────────────────────────────────┘     │             │
│  └──────────────────────┬───────────────────────────┘             │
│                         │                                         │
│  ┌──────────────────────▼──────────────────────────┐             │
│  │         Cloud Storage                           │             │
│  │  - Data exports (7-day TTL)                     │             │
│  │  - PDF reports (Phase 2)                        │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
│  ┌────────────────────────────────────────────────┐              │
│  │         Firebase Authentication                │              │
│  │  - Email/password (MVP)                        │              │
│  │  - Custom claims: {tenant_id}                  │              │
│  └────────────────────────────────────────────────┘              │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────────────┐
│                     External Services                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐     │
│  │ Google Cloud   │  │ OpenAI API     │  │ Picovoice      │     │
│  │ STT/TTS        │  │ GPT-4 (LLM)    │  │ (KWS - local)  │     │
│  │ (cs-CZ voices) │  │ Whisper (STT)  │  │                │     │
│  └────────────────┘  └────────────────┘  └────────────────┘     │
└───────────────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow Between Components

#### 2.3.1 User Authentication Flow

```
1. User enters email/password → Firebase Auth
2. Auth returns UID → App queries /user_tenants/{uid}/memberships
3. If no memberships exist:
   a. Cloud Function creates new tenant at /tenants/{newTenantId}
   b. Creates membership at /tenants/{newTenantId}/members/{uid}
   c. Writes mapping at /user_tenants/{uid}/memberships/{newTenantId}
   d. Sets custom claim: tenant_id = newTenantId
   e. Forces token refresh
4. If memberships exist:
   a. Loads active tenantId from custom claim
   b. Verifies membership at /tenants/{tenantId}/members/{uid}
   c. Loads user role and permissions
5. App initializes with tenant context
```

#### 2.3.2 Voice Command Flow (Online)

```
1. User taps microphone → App records audio
2. Audio sent to Google Cloud STT → Transcription returned
3. Transcription sent to OpenAI GPT-4 → Structured intent returned
4. App queries Firestore for referenced entities (job, vehicle, etc.)
5. Google Cloud TTS generates confirmation audio
6. User confirms → App writes to Firestore
7. Firestore triggers auditLogger Cloud Function
8. Audit log written to /tenants/{tenantId}/audit_logs
```

#### 2.3.3 Offline Data Sync Flow

```
1. User creates cost while offline → Written to IndexedDB
2. Firestore SDK queues write with pending status
3. On reconnection → Firestore syncs queued writes
4. Server validates write via Security Rules
5. If validation passes:
   a. onCreate trigger fires → assignSequence allocates ordinalNumber
   b. auditLogger trigger fires → Creates audit log
   c. Success synced to client
6. If validation fails:
   a. Client receives error
   b. App shows sync error in UI with retry/discard options
```

#### 2.3.4 Sequential Number Allocation

**Online Path:**
```
1. User creates job → App calls allocateSequence({type: 'jobNumber', tenantId})
2. Cloud Function runs transaction:
   a. Reads /sequences/{tenantId}/counters/jobNumber
   b. Increments counter atomically
   c. Returns new jobNumber
3. App writes job with allocated jobNumber
```

**Offline Path:**
```
1. User creates job offline → App writes without jobNumber (null)
2. Document queued in Firestore offline cache
3. On sync → Document written to server
4. onCreate trigger (assignSequence) fires:
   a. Detects missing jobNumber
   b. Allocates next number transactionally
   c. Updates document with jobNumber
5. Client receives updated document with jobNumber
```

### 2.4 Multi-Tenant Architecture

#### 2.4.1 Isolation Strategy

**Tenant-Scoped Collections:**
All business data stored under `/tenants/{tenantId}/` path prefix:
- `/tenants/{tenantId}/members/` - User memberships and roles
- `/tenants/{tenantId}/jobs/` - Job records
- `/tenants/{tenantId}/vehicles/` - Resource definitions
- `/tenants/{tenantId}/audit_logs/` - Compliance logs

**Cross-Tenant Index:**
- `/user_tenants/{uid}/memberships/{tenantId}` - User-to-tenant mapping

**Sequence Management:**
- `/sequences/{tenantId}/counters/{counterType}` - Tenant-scoped counters

#### 2.4.2 Authorization Model

**Security Rules Enforcement:**
```javascript
// All tenant data requires membership verification
match /tenants/{tenantId}/{document=**} {
  allow read, write: if request.auth != null
    && exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid))
    && get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.status == 'active';
}
```

**Role-Based Access Control (RBAC):**
- **Owner**: Full access to all tenant data including audit logs and exports
- **Representative**: Cannot modify business profile, privileges, or access audit logs
- **Team Member**: Read-only access to active jobs (jobs_public collection); can add costs

#### 2.4.3 Custom Claims

Firebase ID token includes:
```json
{
  "user_id": "firebase-uid-12345",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "email_verified": true
}
```

Note: Role authorization is NOT in custom claims. It's enforced by Security Rules checking `/tenants/{tenantId}/members/{uid}` document.

---

## 3. Detailed Design

### 3.1 Firestore Data Model

#### 3.1.1 Collections Structure

**Tenant-Scoped Collections:**

```
/tenants/{tenantId}/
  ├─ members/{uid}                    - User roles and permissions
  ├─ invites/{inviteId}               - Pending team invitations
  ├─ jobs/{jobId}/                    - Job records
  │   ├─ costs/{costId}               - Job costs (subcollection)
  │   ├─ advances/{advanceId}         - Customer advances (subcollection)
  │   └─ events/{eventId}             - Journey events (subcollection)
  ├─ jobs_public/{jobId}              - Sanitized job data for team members
  ├─ vehicles/{vehicleId}             - Vehicle resources
  ├─ machines/{machineId}             - Machine/equipment resources
  ├─ teamMembers/{teamMemberId}       - Team member resources
  ├─ audit_logs/{logId}               - Compliance audit trail
  ├─ businessProfile (document)       - Tenant settings (currency, VAT, units)
  └─ personProfile (document)         - User preferences (language, voice settings)
```

**Global Collections:**

```
/user_tenants/{uid}/
  └─ memberships/{tenantId}           - User-to-tenant mapping index

/sequences/{tenantId}/
  └─ counters/{counterType}           - Sequential number generators
      ├─ jobNumber                    - Tenant-level job counter
      ├─ vehicleNumber                - Tenant-level vehicle counter
      ├─ machineNumber                - Tenant-level machine counter
      ├─ teamMemberNumber             - Tenant-level team member counter
      └─ job_{jobId}_ordinalNumber    - Per-job cost/advance/event counter
```

#### 3.1.2 Collection Access Patterns

| Collection | Read Pattern | Write Pattern | Offline Support |
|------------|--------------|---------------|-----------------|
| tenants/{tid}/members | Startup, auth check | Admin only | No (requires online) |
| tenants/{tid}/jobs | List view, filters | CRUD by owner/rep | Yes (full offline) |
| tenants/{tid}/jobs/{jid}/costs | Job detail view | High frequency | Yes (full offline) |
| tenants/{tid}/vehicles | Resource selector | Low frequency | Yes (cached) |
| tenants/{tid}/audit_logs | Admin export only | Auto (triggers) | No (server-side only) |
| user_tenants/{uid} | Auth flow only | Cloud Function | No (custom claims) |
| sequences/{tid} | Never (client) | Cloud Function only | No (server transaction) |

### 3.2 Cloud Functions Architecture

#### 3.2.1 Function Categories

**Audit Logging Triggers (High Frequency)**

```typescript
// packages/functions/src/triggers/auditLogger.ts

export const auditJobCreate = onDocumentCreated(
  'tenants/{tenantId}/jobs/{jobId}',
  async (event) => {
    const snapshot = event.data;
    const tenantId = event.params.tenantId;
    const jobId = event.params.jobId;
    const data = snapshot?.data();

    if (!data) return;

    await admin.firestore()
      .collection(`tenants/${tenantId}/audit_logs`)
      .add({
        operation: 'CREATE',
        collection: 'jobs',
        documentId: jobId,
        tenantId,
        timestamp: admin.firestore.FieldValue.serverTimestamp(),
        authorId: data.createdBy,
        after: data,
        ttl: admin.firestore.Timestamp.fromDate(
          new Date(Date.now() + 365 * 24 * 60 * 60 * 1000) // 1 year
        )
      });
  }
);

// Similar functions for:
// - auditJobUpdate
// - auditJobDelete
// - auditCostCreate/Update/Delete
// - auditVehicleCreate/Update/Delete
// - auditMachineCreate/Update/Delete
// - auditTeamMemberCreate/Update/Delete
// - auditAdvanceCreate/Update/Delete
```

**Performance Target:** <500ms execution time (NFR14)

**Sequential Number Assignment Triggers (Offline Sync)**

```typescript
// packages/functions/src/triggers/assignSequence.ts

export const assignJobNumber = onDocumentCreated(
  'tenants/{tenantId}/jobs/{jobId}',
  async (event) => {
    const snapshot = event.data;
    const tenantId = event.params.tenantId;
    const jobId = event.params.jobId;
    const data = snapshot?.data();

    // Only assign if jobNumber is missing (offline-created document)
    if (!data || data.jobNumber !== null) return;

    const db = admin.firestore();

    await db.runTransaction(async (transaction) => {
      const counterRef = db.doc(`sequences/${tenantId}/counters/jobNumber`);
      const counterDoc = await transaction.get(counterRef);

      let nextNumber = 1;
      if (counterDoc.exists) {
        nextNumber = (counterDoc.data()?.current || 0) + 1;
      }

      // Update counter
      transaction.set(counterRef, { current: nextNumber }, { merge: true });

      // Update job with assigned number
      transaction.update(db.doc(`tenants/${tenantId}/jobs/${jobId}`), {
        jobNumber: nextNumber,
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });
    });
  }
);

// Similar functions for:
// - assignVehicleNumber
// - assignMachineNumber
// - assignTeamMemberNumber
// - assignOrdinalNumber (for costs/advances/events within a job)
```

**HTTPS Callable Functions**

```typescript
// packages/functions/src/callable/allocateSequence.ts

export const allocateSequence = onCall<AllocateSequenceRequest>(
  async (request) => {
    const { tenantId, type, jobId } = request.data;
    const uid = request.auth?.uid;

    if (!uid || !tenantId) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    // Verify membership
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberDoc.exists || memberDoc.data()?.status !== 'active') {
      throw new HttpsError('permission-denied', 'Not an active member');
    }

    const db = admin.firestore();
    let nextNumber: number;

    await db.runTransaction(async (transaction) => {
      const counterKey = jobId ? `job_${jobId}_ordinalNumber` : type;
      const counterRef = db.doc(`sequences/${tenantId}/counters/${counterKey}`);
      const counterDoc = await transaction.get(counterRef);

      nextNumber = (counterDoc.data()?.current || 0) + 1;

      transaction.set(counterRef, { current: nextNumber }, { merge: true });
    });

    return { sequenceNumber: nextNumber! };
  }
);

interface AllocateSequenceRequest {
  tenantId: string;
  type: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber';
  jobId?: string; // Required for ordinalNumber allocation
}
```

```typescript
// packages/functions/src/callable/exportData.ts

export const exportData = onCall<ExportDataRequest>(
  async (request) => {
    const { tenantId } = request.data;
    const uid = request.auth?.uid;

    if (!uid || !tenantId) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    // Verify owner role
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
      throw new HttpsError('permission-denied', 'Owner role required');
    }

    // Collect all tenant data (excluding audit_logs)
    const exportData = {
      exportedAt: new Date().toISOString(),
      tenantId,
      businessProfile: await exportCollection(`tenants/${tenantId}/businessProfile`),
      jobs: await exportCollection(`tenants/${tenantId}/jobs`),
      vehicles: await exportCollection(`tenants/${tenantId}/vehicles`),
      machines: await exportCollection(`tenants/${tenantId}/machines`),
      teamMembers: await exportCollection(`tenants/${tenantId}/teamMembers`),
      // Subcollections for each job
      costs: await exportJobSubcollections(tenantId, 'costs'),
      advances: await exportJobSubcollections(tenantId, 'advances'),
      events: await exportJobSubcollections(tenantId, 'events')
    };

    // Write to Cloud Storage with 7-day TTL
    const bucket = admin.storage().bucket();
    const fileName = `exports/${tenantId}/${Date.now()}_data_export.json`;
    const file = bucket.file(fileName);

    await file.save(JSON.stringify(exportData, null, 2), {
      contentType: 'application/json',
      metadata: {
        customTime: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString() // 7 days
      }
    });

    // Generate signed URL (valid for 1 hour)
    const [url] = await file.getSignedUrl({
      action: 'read',
      expires: Date.now() + 60 * 60 * 1000 // 1 hour
    });

    return { downloadUrl: url, expiresAt: new Date(Date.now() + 60 * 60 * 1000).toISOString() };
  }
);

interface ExportDataRequest {
  tenantId: string;
}
```

**Scheduled Functions**

```typescript
// packages/functions/src/scheduled/cleanupAuditLogs.ts

export const cleanupAuditLogs = onSchedule(
  'every day 02:00',
  async (event) => {
    const db = admin.firestore();
    const cutoffDate = admin.firestore.Timestamp.fromDate(
      new Date(Date.now() - 365 * 24 * 60 * 60 * 1000) // 1 year ago
    );

    // Query audit logs older than 1 year
    const tenantsSnapshot = await db.collection('tenants').listDocuments();

    for (const tenantRef of tenantsSnapshot) {
      const expiredLogs = await db
        .collection(`${tenantRef.path}/audit_logs`)
        .where('timestamp', '<', cutoffDate)
        .limit(500) // Process in batches
        .get();

      if (expiredLogs.empty) continue;

      const batch = db.batch();
      expiredLogs.docs.forEach(doc => batch.delete(doc.ref));
      await batch.commit();

      console.log(`Deleted ${expiredLogs.size} expired audit logs from ${tenantRef.id}`);
    }
  }
);
```

#### 3.2.2 Function Deployment Configuration

```json
// packages/functions/package.json (engines and dependencies)
{
  "engines": {
    "node": "20"
  },
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0"
  }
}
```

```javascript
// firebase.json (functions configuration)
{
  "functions": [
    {
      "source": "packages/functions",
      "codebase": "default",
      "runtime": "nodejs20",
      "region": "europe-west1",
      "concurrency": 80,
      "memory": "256MB",
      "timeout": "60s"
    }
  ]
}
```

### 3.3 Security Rules

#### 3.3.1 Firestore Security Rules Structure

```javascript
// firestore.rules

rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function getTenantId() {
      return request.auth.token.tenant_id;
    }

    function isMember(tenantId) {
      return isAuthenticated()
        && exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));
    }

    function isActiveMember(tenantId) {
      return isMember(tenantId)
        && get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.status == 'active';
    }

    function getMemberRole(tenantId) {
      return get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role;
    }

    function isOwner(tenantId) {
      return isActiveMember(tenantId) && getMemberRole(tenantId) == 'owner';
    }

    function isOwnerOrRepresentative(tenantId) {
      let role = getMemberRole(tenantId);
      return isActiveMember(tenantId) && (role == 'owner' || role == 'representative');
    }

    function documentHasTenantId(tenantId) {
      return request.resource.data.tenantId == tenantId;
    }

    function auditMetadataValid() {
      return request.resource.data.createdAt == request.time
        && request.resource.data.createdBy == request.auth.uid
        && request.resource.data.updatedAt == request.time
        && request.resource.data.updatedBy == request.auth.uid;
    }

    function updateMetadataValid() {
      return request.resource.data.updatedAt == request.time
        && request.resource.data.updatedBy == request.auth.uid
        && request.resource.data.createdAt == resource.data.createdAt
        && request.resource.data.createdBy == resource.data.createdBy;
    }

    // User-tenant mappings (read-only for clients)
    match /user_tenants/{uid}/memberships/{tenantId} {
      allow read: if isAuthenticated() && request.auth.uid == uid;
      allow write: if false; // Only Cloud Functions can write
    }

    // Sequence counters (server-only)
    match /sequences/{tenantId}/counters/{counterType} {
      allow read, write: if false; // Only Cloud Functions can access
    }

    // Tenant data
    match /tenants/{tenantId} {

      // Members collection
      match /members/{uid} {
        allow read: if isActiveMember(tenantId);
        allow create: if false; // Only via invite consumption (Cloud Function)
        allow update: if isOwner(tenantId); // Owner can change roles/status
        allow delete: if isOwner(tenantId) && uid != request.auth.uid; // Can't delete self
      }

      // Invites collection
      match /invites/{inviteId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid();
        allow update: if false; // Immutable after creation
        allow delete: if isOwner(tenantId);
      }

      // Business profile (single document)
      match /businessProfile {
        allow read: if isActiveMember(tenantId);
        allow write: if isOwner(tenantId) && documentHasTenantId(tenantId);
      }

      // Person profile (single document)
      match /personProfile {
        allow read, write: if isActiveMember(tenantId);
      }

      // Jobs collection
      match /jobs/{jobId} {
        allow read: if isOwnerOrRepresentative(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid();
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid();
        allow delete: if false; // Use soft delete (status: archived)

        // Costs subcollection
        match /costs/{costId} {
          allow read: if isActiveMember(tenantId); // Team members can read
          allow create: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid();
          allow update: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid();
          allow delete: if isOwnerOrRepresentative(tenantId);
        }

        // Advances subcollection
        match /advances/{advanceId} {
          allow read: if isOwnerOrRepresentative(tenantId);
          allow create: if isOwnerOrRepresentative(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid();
          allow update: if isOwnerOrRepresentative(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid();
          allow delete: if isOwnerOrRepresentative(tenantId);
        }

        // Events subcollection
        match /events/{eventId} {
          allow read: if isActiveMember(tenantId);
          allow create: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid();
          allow update: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid();
          allow delete: if isOwnerOrRepresentative(tenantId);
        }
      }

      // Jobs public (sanitized read-only view for team members)
      match /jobs_public/{jobId} {
        allow read: if isActiveMember(tenantId);
        allow write: if false; // Maintained by Cloud Function triggers
      }

      // Vehicles collection
      match /vehicles/{vehicleId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid();
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid();
        allow delete: if isOwnerOrRepresentative(tenantId);
      }

      // Machines collection
      match /machines/{machineId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid();
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid();
        allow delete: if isOwnerOrRepresentative(tenantId);
      }

      // Team members collection
      match /teamMembers/{teamMemberId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid();
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid();
        allow delete: if isOwnerOrRepresentative(tenantId);
      }

      // Audit logs (owner-only read, Cloud Functions write)
      match /audit_logs/{logId} {
        allow read: if isOwner(tenantId);
        allow write: if false; // Only Cloud Functions can write
      }
    }
  }
}
```

#### 3.3.2 Storage Security Rules

```javascript
// storage.rules

rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    function isAuthenticated() {
      return request.auth != null;
    }

    function isTenantOwner(tenantId) {
      return firestore.exists(/databases/(default)/documents/tenants/$(tenantId)/members/$(request.auth.uid))
        && firestore.get(/databases/(default)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role == 'owner';
    }

    // Data exports (owner-only)
    match /exports/{tenantId}/{fileName} {
      allow read: if isAuthenticated() && isTenantOwner(tenantId);
      allow write: if false; // Only Cloud Functions can write
    }

    // PDF reports (Phase 2)
    match /reports/{tenantId}/{fileName} {
      allow read: if isAuthenticated() && isTenantOwner(tenantId);
      allow write: if false; // Only Cloud Functions can write
    }
  }
}
```

---

## 4. Data Model

### 4.1 Complete Schema Definitions

#### 4.1.1 Tenant Collections

**Collection: `/tenants/{tenantId}/members/{uid}`**

```typescript
interface Member {
  tenantId: string;           // UUID, matches path parameter
  role: 'owner' | 'representative' | 'teamMember';
  status: 'active' | 'disabled';
  lastSeenAt: Timestamp | null;
  createdAt: Timestamp;       // serverTimestamp
  updatedAt: Timestamp;       // serverTimestamp
}
```

**Indexes:**
- Single field: `status` (ASC)
- Composite: `status` (ASC) + `role` (ASC)

---

**Collection: `/tenants/{tenantId}/invites/{inviteId}`**

```typescript
interface Invite {
  tenantId: string;           // UUID
  codeHash: string;           // SHA-256 hash of 6-digit code
  expiresAt: Timestamp;       // 7 days from creation
  createdBy: string;          // UID of inviter
  presetRole: 'representative' | 'teamMember';
  consumedAt?: Timestamp;     // Set when invite is accepted
  email?: string;             // Optional: pre-assign to specific email
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Indexes:**
- Single field: `expiresAt` (ASC)
- Single field: `consumedAt` (ASC)
- Composite: `codeHash` (ASC) + `consumedAt` (ASC)

---

**Collection: `/tenants/{tenantId}/jobs/{jobId}`**

```typescript
interface Job {
  tenantId: string;           // UUID
  jobNumber: number | null;   // Sequential, nullable for offline creation
  title: string;              // e.g., "Smith, Brno - Kitchen Renovation"
  description?: string;
  status: 'active' | 'completed' | 'archived';
  currency: string;           // ISO 4217 (e.g., "CZK", "EUR")
  vatRate: number;            // Percentage (e.g., 21 for 21%)
  budget?: number;            // Optional budget amount
  createdAt: Timestamp;
  createdBy: string;          // UID
  updatedAt: Timestamp;
  updatedBy: string;          // UID
}
```

**Indexes:**
- Single field: `jobNumber` (ASC) - must be unique per tenant
- Single field: `status` (ASC)
- Composite: `status` (ASC) + `updatedAt` (DESC)
- Composite: `createdBy` (ASC) + `createdAt` (DESC)

---

**Subcollection: `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}`**

```typescript
type CostCategory = 'transport' | 'material' | 'labor' | 'machine' | 'other';

interface CostBase {
  tenantId: string;
  ordinalNumber: number | null; // Sequential per job
  category: CostCategory;
  amount: number;               // Numeric amount
  description: string;
  date: Timestamp;              // When cost occurred
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}

interface TransportCost extends CostBase {
  category: 'transport';
  resource: {
    vehicleNumber: number;
    name: string;               // Snapshot at creation
    distanceUnit: 'km' | 'miles';
    ratePerDistanceUnit: number;
  };
  distance: number;
  startOdometer?: number;
  endOdometer?: number;
  destination?: string;
}

interface MaterialCost extends CostBase {
  category: 'material';
  resource?: {
    supplierName?: string;
    materialType?: string;
  };
  quantity?: number;
  unitPrice?: number;
}

interface LaborCost extends CostBase {
  category: 'labor';
  resource: {
    teamMemberNumber: number;
    name: string;
    hourlyRate: number;
  };
  hours: number;
}

interface MachineCost extends CostBase {
  category: 'machine';
  resource: {
    machineNumber: number;
    name: string;
    hourlyRate: number;
  };
  hours: number;
}

interface OtherCost extends CostBase {
  category: 'other';
  resource?: Record<string, any>; // Flexible schema
}

type Cost = TransportCost | MaterialCost | LaborCost | MachineCost | OtherCost;
```

**Indexes:**
- Single field: `ordinalNumber` (ASC)
- Single field: `category` (ASC)
- Composite: `category` (ASC) + `date` (DESC)
- Composite: `date` (DESC) + `category` (ASC)

---

**Subcollection: `/tenants/{tenantId}/jobs/{jobId}/advances/{advanceId}`**

```typescript
interface Advance {
  tenantId: string;
  ordinalNumber: number | null; // Sequential per job
  amount: number;
  date: Timestamp;
  note?: string;
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}
```

**Indexes:**
- Single field: `ordinalNumber` (ASC)
- Single field: `date` (DESC)

---

**Subcollection: `/tenants/{tenantId}/jobs/{jobId}/events/{eventId}`**

```typescript
type EventType = 'journey_start' | 'journey_end' | 'manual_entry';

interface EventBase {
  tenantId: string;
  ordinalNumber: number | null;
  type: EventType;
  timestamp: Timestamp;
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}

interface JourneyStartEvent extends EventBase {
  type: 'journey_start';
  data: {
    vehicleNumber: number;
    vehicleName: string;
    destination: string;
    startOdometer: number;
  };
}

interface JourneyEndEvent extends EventBase {
  type: 'journey_end';
  data: {
    vehicleNumber: number;
    vehicleName: string;
    endOdometer: number;
    distance: number;
    autoCreatedCostId?: string; // Reference to auto-created transport cost
  };
}

type Event = JourneyStartEvent | JourneyEndEvent | EventBase;
```

**Indexes:**
- Single field: `type` (ASC)
- Composite: `type` (ASC) + `timestamp` (DESC)

---

**Collection: `/tenants/{tenantId}/jobs_public/{jobId}`**

```typescript
interface JobPublic {
  jobNumber: number;
  title: string;
  status: 'active' | 'completed' | 'archived';
}
```

Purpose: Sanitized read-only view for team members (FR11).

---

**Collection: `/tenants/{tenantId}/vehicles/{vehicleId}`**

```typescript
interface Vehicle {
  tenantId: string;
  vehicleNumber: number | null; // Sequential
  name: string;                  // e.g., "Transporter VW"
  distanceUnit: 'km' | 'miles';  // Copied from business profile at creation
  ratePerDistanceUnit: number;   // e.g., 8.50 CZK/km
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}
```

**Indexes:**
- Single field: `vehicleNumber` (ASC) - unique per tenant
- Single field: `name` (ASC)

---

**Collection: `/tenants/{tenantId}/machines/{machineId}`**

```typescript
interface Machine {
  tenantId: string;
  machineNumber: number | null;
  name: string;                 // e.g., "Concrete Mixer"
  hourlyRate: number;
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}
```

**Indexes:**
- Single field: `machineNumber` (ASC) - unique per tenant

---

**Collection: `/tenants/{tenantId}/teamMembers/{teamMemberId}`**

```typescript
interface TeamMember {
  tenantId: string;
  teamMemberNumber: number | null;
  name: string;
  hourlyRate: number;
  authUserId: string | null;    // Firebase Auth UID for login mapping
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}
```

**Indexes:**
- Single field: `teamMemberNumber` (ASC) - unique per tenant
- Single field: `authUserId` (ASC) - for auth lookup

---

**Document: `/tenants/{tenantId}/businessProfile`**

```typescript
interface BusinessProfile {
  tenantId: string;
  currency: string;             // ISO 4217
  vatRate: number;              // Default VAT percentage
  distanceUnit: 'km' | 'miles';
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

---

**Document: `/tenants/{tenantId}/personProfile`**

```typescript
interface PersonProfile {
  tenantId: string;
  displayName: string;
  email: string;
  language: 'cs' | 'en';
  preferredVoiceProvider?: 'google' | 'openai' | 'platform-native';
  aiSupportEnabled: boolean;    // Global toggle for voice features
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

---

**Collection: `/tenants/{tenantId}/audit_logs/{logId}`**

```typescript
type AuditOperation = 'CREATE' | 'UPDATE' | 'DELETE';

interface AuditLog {
  operation: AuditOperation;
  collection: string;           // e.g., "jobs", "costs", "vehicles"
  documentId: string;
  tenantId: string;
  timestamp: Timestamp;
  authorId: string;             // Team member ID (not UID)
  before?: any;                 // Old data for UPDATE operations
  after?: any;                  // New data for CREATE/UPDATE operations
  ttl: Timestamp;               // Auto-delete after this date (1 year)
}
```

**Indexes:**
- Single field: `ttl` (ASC) - for cleanup queries
- Composite: `timestamp` (DESC) + `collection` (ASC)
- Composite: `authorId` (ASC) + `timestamp` (DESC)

---

#### 4.1.2 Global Collections

**Collection: `/user_tenants/{uid}/memberships/{tenantId}`**

```typescript
interface UserTenantMapping {
  tenantId: string;
  role?: 'owner' | 'representative' | 'teamMember'; // Optional snapshot
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

Purpose: Enable user-to-tenant resolution during auth flow.

---

**Collection: `/sequences/{tenantId}/counters/{counterType}`**

```typescript
interface SequenceCounter {
  current: number;  // Last allocated number
}
```

Counter types:
- `jobNumber`
- `vehicleNumber`
- `machineNumber`
- `teamMemberNumber`
- `job_{jobId}_ordinalNumber` (per-job costs/advances/events)

**Access:** Cloud Functions only (Security Rules deny client access).

---

### 4.2 Entity Relationships

```
Tenant (1) ──< Members (N)
           ──< Invites (N)
           ──< Jobs (N)
           ──< Vehicles (N)
           ──< Machines (N)
           ──< TeamMembers (N)
           ──< AuditLogs (N)
           ──  BusinessProfile (1)
           ──  PersonProfile (1)

Job (1) ──< Costs (N)
        ──< Advances (N)
        ──< Events (N)

Cost (N) ──> Vehicle (0..1)  [snapshot copy]
         ──> Machine (0..1)  [snapshot copy]
         ──> TeamMember (0..1) [snapshot copy]

Event (N) ──> Vehicle (0..1)  [snapshot copy]

User (1) ──< UserTenantMappings (N)
```

**Key Design Decisions:**

1. **Subcollections for Job-Related Data**: Costs, advances, and events are subcollections under `/tenants/{tenantId}/jobs/{jobId}/` to optimize offline queries (fetch job + all costs in single query).

2. **Resource Snapshots**: Costs and events embed resource data (vehicle name, hourly rate) at creation time. This prevents historical data corruption if resource definitions change later.

3. **Soft Deletes**: Jobs use `status: archived` instead of document deletion to preserve audit trail and cost references.

4. **Nullable Sequential Numbers**: Documents created offline have `null` sequential numbers until server-side assignment completes.

---

### 4.3 Required Indexes

#### 4.3.1 Composite Indexes (firestore.indexes.json)

```json
{
  "indexes": [
    {
      "collectionGroup": "jobs",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "updatedAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "costs",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "category", "order": "ASCENDING" },
        { "fieldPath": "date", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "audit_logs",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "timestamp", "order": "DESCENDING" },
        { "fieldPath": "collection", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "events",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "type", "order": "ASCENDING" },
        { "fieldPath": "timestamp", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

#### 4.3.2 Index Deployment

```bash
# Deploy indexes
firebase deploy --only firestore:indexes

# Indexes build asynchronously; monitor progress:
# Firebase Console > Firestore > Indexes tab
```

---

## 5. Interface and Connectivity

### 5.1 Firebase SDK Integration

#### 5.1.1 AngularFire Configuration

```typescript
// packages/mobile-app/src/app/app.config.ts

import { provideFirebaseApp, initializeApp } from '@angular/fire/app';
import { provideAuth, getAuth } from '@angular/fire/auth';
import { provideFirestore, getFirestore, enableIndexedDbPersistence } from '@angular/fire/firestore';
import { provideFunctions, getFunctions } from '@angular/fire/functions';
import { provideStorage, getStorage } from '@angular/fire/storage';

export const appConfig: ApplicationConfig = {
  providers: [
    provideFirebaseApp(() => initializeApp(environment.firebase)),
    provideAuth(() => getAuth()),
    provideFirestore(() => {
      const firestore = getFirestore();
      enableIndexedDbPersistence(firestore).catch((err) => {
        if (err.code === 'failed-precondition') {
          console.warn('Offline persistence failed: multiple tabs open');
        } else if (err.code === 'unimplemented') {
          console.warn('Offline persistence not supported by browser');
        }
      });
      return firestore;
    }),
    provideFunctions(() => getFunctions(undefined, 'europe-west1')),
    provideStorage(() => getStorage()),
    // ... other providers
  ]
};
```

#### 5.1.2 Environment Configuration

```typescript
// packages/mobile-app/src/environments/environment.ts

export const environment = {
  production: false,
  firebase: {
    apiKey: 'AIza...',
    authDomain: 'findogai-dev.firebaseapp.com',
    projectId: 'findogai-dev',
    storageBucket: 'findogai-dev.appspot.com',
    messagingSenderId: '123456789',
    appId: '1:123456789:web:abcdef'
  },
  voiceProviders: {
    stt: 'mock', // 'google-cloud' | 'openai-whisper' | 'mock'
    llm: 'mock', // 'openai-gpt4' | 'groq-llama3' | 'ollama' | 'mock'
    tts: 'platform-native' // 'google-cloud' | 'platform-native' | 'mock'
  },
  apiKeys: {
    googleCloud: '',
    openai: '',
    groq: ''
  }
};
```

```typescript
// packages/mobile-app/src/environments/environment.prod.ts

export const environment = {
  production: true,
  firebase: {
    apiKey: 'AIza...',
    authDomain: 'findogai.firebaseapp.com',
    projectId: 'findogai-prod',
    storageBucket: 'findogai-prod.appspot.com',
    messagingSenderId: '987654321',
    appId: '1:987654321:web:fedcba'
  },
  voiceProviders: {
    stt: 'google-cloud',
    llm: 'openai-gpt4',
    tts: 'google-cloud'
  },
  apiKeys: {
    googleCloud: process.env['GOOGLE_CLOUD_API_KEY'],
    openai: process.env['OPENAI_API_KEY'],
    groq: ''
  }
};
```

### 5.2 Cloud Function Triggers

#### 5.2.1 Firestore Triggers

**Trigger Types:**
- `onDocumentCreated`: Fires when new document is written
- `onDocumentUpdated`: Fires when document is modified
- `onDocumentDeleted`: Fires when document is removed
- `onDocumentWritten`: Fires on create, update, or delete

**Example: Audit Logging on Job Creation**

```typescript
import { onDocumentCreated } from 'firebase-functions/v2/firestore';

export const onJobCreated = onDocumentCreated(
  {
    document: 'tenants/{tenantId}/jobs/{jobId}',
    region: 'europe-west1'
  },
  async (event) => {
    const snapshot = event.data;
    const { tenantId, jobId } = event.params;

    // Write audit log
    await event.firestore
      .collection(`tenants/${tenantId}/audit_logs`)
      .add({
        operation: 'CREATE',
        collection: 'jobs',
        documentId: jobId,
        tenantId,
        timestamp: event.time,
        authorId: snapshot?.data()?.createdBy,
        after: snapshot?.data(),
        ttl: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000)
      });
  }
);
```

#### 5.2.2 HTTPS Callable Functions

**Client-Side Invocation:**

```typescript
// packages/mobile-app/src/app/services/sequence.service.ts

import { Injectable, inject } from '@angular/core';
import { Functions, httpsCallable } from '@angular/fire/functions';

@Injectable({ providedIn: 'root' })
export class SequenceService {
  private functions = inject(Functions);

  async allocateJobNumber(tenantId: string): Promise<number> {
    const callable = httpsCallable<AllocateSequenceRequest, AllocateSequenceResponse>(
      this.functions,
      'allocateSequence'
    );

    const result = await callable({ tenantId, type: 'jobNumber' });
    return result.data.sequenceNumber;
  }

  async allocateOrdinalNumber(tenantId: string, jobId: string): Promise<number> {
    const callable = httpsCallable<AllocateSequenceRequest, AllocateSequenceResponse>(
      this.functions,
      'allocateSequence'
    );

    const result = await callable({ tenantId, type: 'ordinalNumber', jobId });
    return result.data.sequenceNumber;
  }
}

interface AllocateSequenceRequest {
  tenantId: string;
  type: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber' | 'ordinalNumber';
  jobId?: string;
}

interface AllocateSequenceResponse {
  sequenceNumber: number;
}
```

### 5.3 Real-Time Listeners

#### 5.3.1 Collection Queries with Real-Time Updates

```typescript
// packages/mobile-app/src/app/services/job.service.ts

import { Injectable, inject } from '@angular/core';
import { Firestore, collection, query, where, orderBy, collectionData } from '@angular/fire/firestore';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class JobService {
  private firestore = inject(Firestore);

  getActiveJobs(tenantId: string): Observable<Job[]> {
    const jobsRef = collection(this.firestore, `tenants/${tenantId}/jobs`);
    const activeQuery = query(
      jobsRef,
      where('status', '==', 'active'),
      orderBy('updatedAt', 'desc')
    );

    return collectionData(activeQuery, { idField: 'id' }) as Observable<Job[]>;
  }

  getJobCosts(tenantId: string, jobId: string): Observable<Cost[]> {
    const costsRef = collection(this.firestore, `tenants/${tenantId}/jobs/${jobId}/costs`);
    const costsQuery = query(costsRef, orderBy('date', 'desc'));

    return collectionData(costsQuery, { idField: 'id' }) as Observable<Cost[]>;
  }
}
```

#### 5.3.2 Document Snapshots

```typescript
// Real-time sync for business profile
import { doc, docData } from '@angular/fire/firestore';

getBusinessProfile(tenantId: string): Observable<BusinessProfile> {
  const profileRef = doc(this.firestore, `tenants/${tenantId}/businessProfile`);
  return docData(profileRef) as Observable<BusinessProfile>;
}
```

### 5.4 Offline Persistence Configuration

#### 5.4.1 Persistence Settings

```typescript
// Enable offline persistence (IndexedDB)
import { enableIndexedDbPersistence } from '@angular/fire/firestore';

const firestore = getFirestore();
enableIndexedDbPersistence(firestore, {
  synchronizeTabs: true // Sync across browser tabs
}).catch((err) => {
  if (err.code === 'failed-precondition') {
    console.warn('Multiple tabs open; persistence enabled in first tab only');
  } else if (err.code === 'unimplemented') {
    console.warn('Browser does not support offline persistence');
  }
});
```

#### 5.4.2 Offline Behavior

**Write Operations:**
- Writes succeed immediately against local cache
- Queued automatically for server sync
- `hasPendingWrites` metadata tracks sync status

**Read Operations:**
- Reads always succeed from cache if data exists
- `fromCache` metadata indicates source

**Example: Pending Writes Detection**

```typescript
import { doc, onSnapshot } from '@angular/fire/firestore';

const jobRef = doc(firestore, `tenants/${tenantId}/jobs/${jobId}`);

onSnapshot(jobRef, (snapshot) => {
  const data = snapshot.data();
  const isPending = snapshot.metadata.hasPendingWrites;
  const isFromCache = snapshot.metadata.fromCache;

  console.log('Job data:', data);
  console.log('Pending sync:', isPending);
  console.log('Offline mode:', isFromCache);
});
```

---

## 6. Technology Stack

### 6.1 Firebase Services

| Service | Version | Purpose | Region |
|---------|---------|---------|--------|
| Firebase Authentication | v11+ | User identity management | Global |
| Cloud Firestore | v11+ | NoSQL database with offline sync | europe-west1 |
| Cloud Functions (2nd Gen) | v5+ | Serverless backend logic | europe-west1 |
| Cloud Storage for Firebase | v11+ | File storage (exports, PDFs) | europe-west1 |
| Firebase Hosting | v13+ | PWA hosting | Global CDN |

### 6.2 Cloud Functions Runtime

**Node.js Version:** 20 LTS (Long-Term Support)

**Key Dependencies:**

```json
{
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0",
    "pdfmake": "^0.2.10" // Phase 2: PDF generation
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.3.0",
    "eslint": "^8.56.0",
    "jest": "^29.7.0"
  }
}
```

### 6.3 TypeScript for Type Safety

**Shared Types Package:** `/packages/shared-types`

```typescript
// packages/shared-types/src/models/job.ts

export interface Job {
  tenantId: string;
  jobNumber: number | null;
  title: string;
  description?: string;
  status: 'active' | 'completed' | 'archived';
  currency: string;
  vatRate: number;
  budget?: number;
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}

// Imported by both mobile app and Cloud Functions
```

**Benefits:**
- Type safety across monorepo packages
- Single source of truth for data models
- Compile-time validation of API contracts
- Autocomplete in IDEs

### 6.4 Supporting Libraries

#### 6.4.1 Voice API SDKs

```json
{
  "dependencies": {
    "@google-cloud/speech": "^6.5.0",
    "@google-cloud/text-to-speech": "^5.2.0",
    "openai": "^4.28.0",
    "@picovoice/porcupine-web": "^3.0.0"
  }
}
```

#### 6.4.2 Utility Libraries

```json
{
  "dependencies": {
    "uuid": "^9.0.1",           // UUID generation
    "date-fns": "^3.3.1",       // Date manipulation
    "zod": "^3.22.4"            // Runtime validation
  }
}
```

---

## 7. Development and Deployment

### 7.1 Environment Configuration

#### 7.1.1 Firebase Projects

| Environment | Project ID | Purpose |
|-------------|------------|---------|
| Development | findogai-dev | Local development with emulators |
| Staging | findogai-staging | Pre-production testing |
| Production | findogai-prod | Live user data |

#### 7.1.2 Environment Variables

**Local Development (.env.local - NOT committed to git):**

```bash
# Firebase configuration (from Firebase Console)
FIREBASE_API_KEY=AIza...
FIREBASE_AUTH_DOMAIN=findogai-dev.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-dev

# Voice API keys
GOOGLE_CLOUD_API_KEY=AIza...
OPENAI_API_KEY=sk-proj-...
GROQ_API_KEY=gsk_...

# Provider selection (for development)
STT_PROVIDER=mock
LLM_PROVIDER=mock
TTS_PROVIDER=platform-native
```

**Production (.env.production - values stored in CI/CD secrets):**

```bash
FIREBASE_PROJECT_ID=findogai-prod
GOOGLE_CLOUD_API_KEY=${GOOGLE_CLOUD_API_KEY}
OPENAI_API_KEY=${OPENAI_API_KEY}
STT_PROVIDER=google-cloud
LLM_PROVIDER=openai-gpt4
TTS_PROVIDER=google-cloud
```

### 7.2 CI/CD Pipeline

#### 7.2.1 GitHub Actions Workflow

**File:** `.github/workflows/deploy.yml`

```yaml
name: Deploy to Firebase

on:
  push:
    branches:
      - main        # Production deployment
      - staging     # Staging deployment

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:ci

      - name: Run Security Rules tests
        run: npm run test:rules

  deploy-functions:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Firebase CLI
        run: npm install -g firebase-tools

      - name: Deploy Cloud Functions
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
          GOOGLE_CLOUD_API_KEY: ${{ secrets.GOOGLE_CLOUD_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          firebase use ${{ github.ref == 'refs/heads/main' && 'findogai-prod' || 'findogai-staging' }}
          firebase deploy --only functions

      - name: Deploy Firestore Rules and Indexes
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: firebase deploy --only firestore:rules,firestore:indexes

  deploy-hosting:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Build mobile app (PWA)
        run: |
          cd packages/mobile-app
          npm run build:prod

      - name: Deploy to Firebase Hosting
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: |
          firebase use ${{ github.ref == 'refs/heads/main' && 'findogai-prod' || 'findogai-staging' }}
          firebase deploy --only hosting
```

### 7.3 Firebase Deployment Process

#### 7.3.1 Manual Deployment (Development)

```bash
# Select Firebase project
firebase use findogai-dev

# Deploy all services
firebase deploy

# Deploy specific services
firebase deploy --only functions
firebase deploy --only firestore:rules
firebase deploy --only firestore:indexes
firebase deploy --only hosting
firebase deploy --only storage
```

#### 7.3.2 Functions Deployment Best Practices

```bash
# Deploy specific function (for faster iteration)
firebase deploy --only functions:allocateSequence

# Deploy with environment config
firebase functions:config:set \
  voice.stt_provider=google-cloud \
  voice.llm_provider=openai-gpt4 \
  voice.tts_provider=google-cloud

# View current config
firebase functions:config:get
```

### 7.4 Monitoring and Logging

#### 7.4.1 Cloud Functions Logs

```bash
# View real-time logs
firebase functions:log

# Filter by function name
firebase functions:log --only allocateSequence

# View logs in Cloud Console
# https://console.cloud.google.com/logs/query?project=findogai-prod
```

#### 7.4.2 Firestore Monitoring

**Firebase Console:**
- **Usage Tab:** Read/write counts, storage size
- **Indexes Tab:** Index build status
- **Rules Playground:** Test Security Rules

**Cloud Logging Queries:**

```sql
-- Slow queries (>1s execution time)
resource.type="cloud_firestore_database"
protoPayload.methodName="google.firestore.v1.Firestore.RunQuery"
protoPayload.latency > "1s"

-- Security Rules denials
resource.type="cloud_firestore_database"
protoPayload.status.code=7
```

#### 7.4.3 Performance Monitoring

```typescript
// packages/mobile-app/src/app/app.config.ts

import { providePerformance, getPerformance } from '@angular/fire/performance';

export const appConfig: ApplicationConfig = {
  providers: [
    providePerformance(() => getPerformance()),
    // Tracks page load, network requests, custom traces
  ]
};
```

**Custom Traces:**

```typescript
import { trace } from '@angular/fire/performance';

const t = await trace(performance, 'voice_command_round_trip');
t.start();

// ... voice pipeline execution ...

t.stop();
```

---

## 8. Security Considerations

### 8.1 Authentication Flow

#### 8.1.1 Email/Password Authentication (MVP)

**Registration Flow:**

```typescript
// packages/mobile-app/src/app/services/auth.service.ts

import { Auth, createUserWithEmailAndPassword, signInWithEmailAndPassword } from '@angular/fire/auth';

async register(email: string, password: string): Promise<void> {
  const credential = await createUserWithEmailAndPassword(this.auth, email, password);
  const uid = credential.user.uid;

  // Check if user already has tenant mapping
  const mappingRef = doc(this.firestore, `user_tenants/${uid}/memberships`);
  const mappingSnap = await getDoc(mappingRef);

  if (!mappingSnap.exists()) {
    // New user → Create tenant via Cloud Function
    const createTenant = httpsCallable(this.functions, 'createTenantForUser');
    await createTenant({ uid, email });

    // Force token refresh to get custom claim
    await credential.user.getIdToken(true);
  }
}
```

**Sign-In Flow:**

```typescript
async signIn(email: string, password: string): Promise<void> {
  const credential = await signInWithEmailAndPassword(this.auth, email, password);
  const token = await credential.user.getIdTokenResult();

  // Extract tenantId from custom claim
  const tenantId = token.claims['tenant_id'];

  if (!tenantId) {
    throw new Error('User not associated with any tenant');
  }

  // Verify membership status
  const memberRef = doc(this.firestore, `tenants/${tenantId}/members/${credential.user.uid}`);
  const memberSnap = await getDoc(memberRef);

  if (!memberSnap.exists() || memberSnap.data().status !== 'active') {
    throw new Error('Account is disabled');
  }

  // Store tenant context in app state
  this.store.dispatch(setActiveTenant({ tenantId }));
}
```

#### 8.1.2 OAuth Providers (Phase 2)

Planned providers:
- Google OAuth (`GoogleAuthProvider`)
- Apple Sign-In (`OAuthProvider`)

### 8.2 Authorization Patterns

#### 8.2.1 Role-Based Access Control (RBAC)

**Security Rules Helpers:**

```javascript
function getMemberRole(tenantId) {
  return get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role;
}

function isOwner(tenantId) {
  return getMemberRole(tenantId) == 'owner';
}

function canReadAuditLogs(tenantId) {
  return isOwner(tenantId); // Only owners
}

function canExportData(tenantId) {
  return isOwner(tenantId); // Only owners
}

function canManageTeam(tenantId) {
  let role = getMemberRole(tenantId);
  return role == 'owner' || role == 'representative';
}

function canWriteCosts(tenantId) {
  return isActiveMember(tenantId); // All active members
}
```

#### 8.2.2 Custom Claims for Tenant Context

**Cloud Function: Set Custom Claim**

```typescript
import { auth } from 'firebase-admin';

export async function setTenantClaim(uid: string, tenantId: string): Promise<void> {
  await auth().setCustomUserClaims(uid, { tenant_id: tenantId });
}
```

**Client-Side Claim Verification:**

```typescript
const token = await user.getIdTokenResult();
const tenantId = token.claims['tenant_id'];

if (!tenantId) {
  console.error('Missing tenant_id custom claim');
  // Trigger re-authentication or tenant setup
}
```

### 8.3 Data Encryption

#### 8.3.1 Encryption at Rest

**Firestore:** All data encrypted at rest using AES-256 (Google-managed keys).

**Cloud Storage:** All files encrypted at rest using AES-256.

**No additional action required:** Encryption is automatic.

#### 8.3.2 Encryption in Transit

**TLS 1.2+:** All Firebase SDK connections use HTTPS/TLS.

**Certificate Pinning (Optional - Phase 2):** Capacitor HTTP plugin supports certificate pinning for mobile apps.

### 8.4 API Security

#### 8.4.1 Cloud Functions Authentication

```typescript
export const allocateSequence = onCall<AllocateSequenceRequest>(
  { enforceAppCheck: true }, // Require App Check token (Phase 2)
  async (request) => {
    // Verify user is authenticated
    if (!request.auth) {
      throw new HttpsError('unauthenticated', 'User must be signed in');
    }

    // Verify tenant membership
    const { tenantId } = request.data;
    const uid = request.auth.uid;

    const memberSnap = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberSnap.exists || memberSnap.data()?.status !== 'active') {
      throw new HttpsError('permission-denied', 'Not an active member');
    }

    // Proceed with operation
  }
);
```

#### 8.4.2 Rate Limiting (Phase 2)

**Firebase App Check:**
- Protects against abuse from non-mobile clients
- Verifies requests originate from legitimate app instances
- Integrates with reCAPTCHA Enterprise (web) and DeviceCheck/Play Integrity (mobile)

**Cloud Functions Quotas:**
- Default: 1,000 invocations/100 seconds per function
- Configurable via `concurrency` and `maxInstances` settings

---

## 9. Performance Considerations

### 9.1 Firestore Query Optimization

#### 9.1.1 Efficient Query Patterns

**Use Composite Indexes:**

```typescript
// Efficient: Uses composite index (status + updatedAt)
const activeJobsQuery = query(
  collection(firestore, `tenants/${tenantId}/jobs`),
  where('status', '==', 'active'),
  orderBy('updatedAt', 'desc'),
  limit(20)
);

// Inefficient: Requires full collection scan without index
const badQuery = query(
  collection(firestore, `tenants/${tenantId}/jobs`),
  where('title', '>=', 'A'),
  orderBy('title'),
  orderBy('updatedAt', 'desc') // Requires composite index
);
```

**Pagination with `startAfter`:**

```typescript
let lastVisible: DocumentSnapshot;

// First page
const firstPage = await getDocs(
  query(collection(firestore, `tenants/${tenantId}/jobs`), limit(20))
);
lastVisible = firstPage.docs[firstPage.docs.length - 1];

// Next page
const nextPage = await getDocs(
  query(
    collection(firestore, `tenants/${tenantId}/jobs`),
    startAfter(lastVisible),
    limit(20)
  )
);
```

#### 9.1.2 Subcollection vs Root Collection Trade-offs

**Subcollection Design (Chosen):**
```
/tenants/{tid}/jobs/{jid}/costs/{cid}
```

**Advantages:**
- Efficient job + costs queries (single listener)
- Offline performance: All job data cached together
- Clear hierarchical relationship

**Disadvantages:**
- Cannot query across all costs globally
- Requires `collectionGroup` queries for cross-job aggregation

**Root Collection Alternative (Not Chosen):**
```
/tenants/{tid}/costs/{cid}
  - jobId: string
```

**Advantages:**
- Easy cross-job queries
- Simpler data model

**Disadvantages:**
- Requires separate queries for job + costs
- Larger cache footprint for offline

**Decision Rationale:** MVP prioritizes job-centric workflows where users view one job at a time. Subcollections optimize for this pattern.

### 9.2 Cloud Function Cold Starts

#### 9.2.1 Cold Start Mitigation

**Concurrency Configuration:**

```typescript
import { onCall } from 'firebase-functions/v2/https';

export const allocateSequence = onCall({
  region: 'europe-west1',
  concurrency: 80,        // Process up to 80 requests per instance
  minInstances: 0,        // Scale to zero when idle (cost optimization)
  maxInstances: 10,       // Cap to prevent runaway costs
  memory: '256MB',        // Minimal memory for lightweight functions
  timeoutSeconds: 60
}, async (request) => {
  // Function logic
});
```

**Warm-Up Strategy (Optional):**

```typescript
// Scheduled function to keep instances warm during peak hours
export const keepWarm = onSchedule(
  'every 5 minutes 08:00-18:00',
  async () => {
    // No-op function; just keeps instances alive
    console.log('Keep-warm ping');
  }
);
```

**Performance Target:** <200ms cold start for lightweight functions (256MB memory).

#### 9.2.2 Optimize Function Size

**Tree-Shaking and Bundling:**

```json
// package.json
{
  "scripts": {
    "build": "tsc && esbuild src/index.ts --bundle --platform=node --outfile=lib/index.js"
  }
}
```

**Benefits:**
- Reduces deployment size (faster cold starts)
- Eliminates unused dependencies

### 9.3 Caching Strategies

#### 9.3.1 Firestore Client-Side Cache

**Offline Persistence:**
- IndexedDB cache stores up to ~40MB on web
- Capacitor apps: Storage limited by device

**Cache Management:**

```typescript
import { enablePersistentCacheIndexAutoCreation } from '@angular/fire/firestore';

// Enable automatic index creation for cached queries
enablePersistentCacheIndexAutoCreation(firestore);
```

**Cache Invalidation:**

```typescript
import { clearIndexedDbPersistence } from '@angular/fire/firestore';

// Clear cache (e.g., on sign-out)
await clearIndexedDbPersistence(firestore);
```

#### 9.3.2 Application-Level Caching

**NgRx Entity Adapter (Normalized State):**

```typescript
import { EntityState, EntityAdapter, createEntityAdapter } from '@ngrx/entity';

export interface JobsState extends EntityState<Job> {
  selectedJobId: string | null;
}

export const jobsAdapter: EntityAdapter<Job> = createEntityAdapter<Job>({
  selectId: (job) => job.id,
  sortComparer: (a, b) => b.updatedAt.toMillis() - a.updatedAt.toMillis()
});

// Benefits:
// - O(1) lookups by ID
// - Automatic de-duplication
// - Optimized updates
```

**Resource Caching (Vehicles, Machines):**

```typescript
// Cache resources for 1 hour (low churn rate)
@Injectable()
export class ResourceCacheService {
  private cache = new Map<string, { data: any; timestamp: number }>();
  private TTL = 60 * 60 * 1000; // 1 hour

  get(key: string): any | null {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (Date.now() - entry.timestamp > this.TTL) {
      this.cache.delete(key);
      return null;
    }

    return entry.data;
  }

  set(key: string, data: any): void {
    this.cache.set(key, { data, timestamp: Date.now() });
  }
}
```

### 9.4 Cost Optimization

#### 9.4.1 Firestore Cost Breakdown

| Operation | Cost (Free Tier) | Cost (Paid) |
|-----------|------------------|-------------|
| Document reads | 50,000/day | $0.06 / 100,000 |
| Document writes | 20,000/day | $0.18 / 100,000 |
| Document deletes | 20,000/day | $0.02 / 100,000 |
| Storage | 1 GB | $0.18 / GB/month |

**Optimization Strategies:**

1. **Use Offline Persistence:** Reduces redundant reads
2. **Limit Query Results:** Always use `limit()` for list views
3. **Batch Writes:** Use `writeBatch()` for multiple operations
4. **TTL for Audit Logs:** Automatic cleanup reduces storage costs
5. **Avoid Real-Time Listeners for Static Data:** Use one-time `getDocs()` for resources

#### 9.4.2 Cloud Functions Cost Breakdown

| Metric | Free Tier | Cost (Paid) |
|--------|-----------|-------------|
| Invocations | 2M/month | $0.40 / 1M |
| Compute (GB-sec) | 400,000/month | $0.0000025 / GB-sec |
| Networking (egress) | 5 GB/month | $0.12 / GB |

**Optimization Strategies:**

1. **Minimize Trigger Functions:** Audit logging adds 3x write cost (acceptable for compliance)
2. **Use `onCall` for Client-Driven Operations:** Avoids unnecessary background processing
3. **Optimize Memory Allocation:** 256MB sufficient for most functions
4. **Set Aggressive Timeouts:** Prevents long-running functions (60s max)

#### 9.4.3 Voice API Cost Estimates

**Google Cloud Speech-to-Text:**
- Standard model: $0.006 / 15 seconds
- Enhanced model: $0.009 / 15 seconds
- **Estimate:** 20 voice commands/day × $0.006 = $0.12/user/day

**OpenAI GPT-4:**
- Input: $0.03 / 1K tokens
- Output: $0.06 / 1K tokens
- **Estimate:** 20 intents × 200 tokens avg = $0.024/user/day

**Google Cloud Text-to-Speech:**
- Standard voices: $4.00 / 1M characters
- WaveNet voices: $16.00 / 1M characters
- **Estimate:** 20 confirmations × 100 chars = $0.032/user/day

**Total Voice Cost:** ~$0.18/user/day (active usage)

**Mitigation:**
- Fallback to platform-native TTS (free)
- Groq/Llama3 for cost-efficient LLM ($0.001/1M tokens)
- User setting to disable voice features

---

## 10. Testing Strategy

### 10.1 Unit Testing for Cloud Functions

#### 10.1.1 Jest Configuration

```json
// packages/functions/jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70
    }
  }
};
```

#### 10.1.2 Example: Sequence Allocation Tests

```typescript
// packages/functions/src/__tests__/allocateSequence.test.ts

import { allocateSequence } from '../callable/allocateSequence';
import * as admin from 'firebase-admin';

// Mock Firebase Admin
jest.mock('firebase-admin', () => ({
  firestore: jest.fn(() => ({
    doc: jest.fn(),
    runTransaction: jest.fn()
  }))
}));

describe('allocateSequence', () => {
  let mockTransaction: any;
  let mockCounterRef: any;

  beforeEach(() => {
    mockTransaction = {
      get: jest.fn(),
      set: jest.fn()
    };

    mockCounterRef = {
      path: 'sequences/tenant123/counters/jobNumber'
    };

    (admin.firestore as any).mockReturnValue({
      doc: jest.fn(() => mockCounterRef),
      runTransaction: jest.fn((callback) => callback(mockTransaction))
    });
  });

  it('should allocate first jobNumber when counter does not exist', async () => {
    mockTransaction.get.mockResolvedValue({ exists: false });

    const result = await allocateSequence({
      data: { tenantId: 'tenant123', type: 'jobNumber' },
      auth: { uid: 'user123' }
    } as any);

    expect(result.sequenceNumber).toBe(1);
    expect(mockTransaction.set).toHaveBeenCalledWith(
      mockCounterRef,
      { current: 1 },
      { merge: true }
    );
  });

  it('should increment existing counter', async () => {
    mockTransaction.get.mockResolvedValue({
      exists: true,
      data: () => ({ current: 42 })
    });

    const result = await allocateSequence({
      data: { tenantId: 'tenant123', type: 'jobNumber' },
      auth: { uid: 'user123' }
    } as any);

    expect(result.sequenceNumber).toBe(43);
    expect(mockTransaction.set).toHaveBeenCalledWith(
      mockCounterRef,
      { current: 43 },
      { merge: true }
    );
  });

  it('should reject unauthenticated requests', async () => {
    await expect(
      allocateSequence({
        data: { tenantId: 'tenant123', type: 'jobNumber' },
        auth: null
      } as any)
    ).rejects.toThrow('User must be authenticated');
  });
});
```

### 10.2 Integration Testing with Firebase Emulator

#### 10.2.1 Emulator Configuration

```json
// firebase.json
{
  "emulators": {
    "auth": { "port": 9099 },
    "firestore": { "port": 8080 },
    "functions": { "port": 5001 },
    "storage": { "port": 9199 },
    "ui": { "enabled": true, "port": 4000 }
  }
}
```

#### 10.2.2 Example: End-to-End Sequence Allocation Test

```typescript
// packages/functions/src/__tests__/integration/sequenceAllocation.test.ts

import { initializeApp } from 'firebase-admin/app';
import { getFirestore } from 'firebase-admin/firestore';

describe('Sequence Allocation Integration', () => {
  let firestore: FirebaseFirestore.Firestore;

  beforeAll(() => {
    process.env.FIRESTORE_EMULATOR_HOST = 'localhost:8080';
    initializeApp({ projectId: 'demo-test' });
    firestore = getFirestore();
  });

  beforeEach(async () => {
    // Clear Firestore emulator data
    await fetch('http://localhost:8080/emulator/v1/projects/demo-test/databases/(default)/documents', {
      method: 'DELETE'
    });
  });

  it('should allocate sequential job numbers', async () => {
    const tenantId = 'tenant123';

    // Simulate 3 job creations
    const job1Ref = await firestore.collection(`tenants/${tenantId}/jobs`).add({
      tenantId,
      jobNumber: null,
      title: 'Job 1',
      status: 'active'
    });

    const job2Ref = await firestore.collection(`tenants/${tenantId}/jobs`).add({
      tenantId,
      jobNumber: null,
      title: 'Job 2',
      status: 'active'
    });

    const job3Ref = await firestore.collection(`tenants/${tenantId}/jobs`).add({
      tenantId,
      jobNumber: null,
      title: 'Job 3',
      status: 'active'
    });

    // Trigger assignSequence Cloud Function (would fire automatically in real environment)
    // For testing, call directly or wait for emulator trigger

    // Wait for functions to process
    await new Promise(resolve => setTimeout(resolve, 2000));

    // Verify sequential numbers assigned
    const job1 = await job1Ref.get();
    const job2 = await job2Ref.get();
    const job3 = await job3Ref.get();

    expect(job1.data()?.jobNumber).toBe(1);
    expect(job2.data()?.jobNumber).toBe(2);
    expect(job3.data()?.jobNumber).toBe(3);
  });
});
```

### 10.3 Security Rules Testing

#### 10.3.1 Rules Test Configuration

```typescript
// packages/functions/src/__tests__/rules/firestoreRules.test.ts

import { initializeTestEnvironment, assertSucceeds, assertFails } from '@firebase/rules-unit-testing';
import { readFileSync } from 'fs';

describe('Firestore Security Rules', () => {
  let testEnv: any;

  beforeAll(async () => {
    testEnv = await initializeTestEnvironment({
      projectId: 'demo-test',
      firestore: {
        rules: readFileSync('firestore.rules', 'utf8'),
        host: 'localhost',
        port: 8080
      }
    });
  });

  afterAll(async () => {
    await testEnv.cleanup();
  });

  beforeEach(async () => {
    await testEnv.clearFirestore();
  });

  describe('Jobs Collection', () => {
    it('should allow owner to read jobs', async () => {
      const tenantId = 'tenant123';
      const ownerId = 'user123';

      // Setup: Create membership
      await testEnv.withSecurityRulesDisabled(async (context: any) => {
        await context.firestore().doc(`tenants/${tenantId}/members/${ownerId}`).set({
          tenantId,
          role: 'owner',
          status: 'active'
        });

        await context.firestore().doc(`tenants/${tenantId}/jobs/job1`).set({
          tenantId,
          jobNumber: 1,
          title: 'Test Job',
          status: 'active'
        });
      });

      // Test: Owner can read
      const ownerContext = testEnv.authenticatedContext(ownerId, { tenant_id: tenantId });
      const jobRef = ownerContext.firestore().doc(`tenants/${tenantId}/jobs/job1`);

      await assertSucceeds(jobRef.get());
    });

    it('should deny team member from reading full job details', async () => {
      const tenantId = 'tenant123';
      const teamMemberId = 'user456';

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context: any) => {
        await context.firestore().doc(`tenants/${tenantId}/members/${teamMemberId}`).set({
          tenantId,
          role: 'teamMember',
          status: 'active'
        });

        await context.firestore().doc(`tenants/${tenantId}/jobs/job1`).set({
          tenantId,
          jobNumber: 1,
          title: 'Test Job',
          status: 'active',
          budget: 50000 // Sensitive data
        });
      });

      // Test: Team member cannot read full job
      const memberContext = testEnv.authenticatedContext(teamMemberId, { tenant_id: tenantId });
      const jobRef = memberContext.firestore().doc(`tenants/${tenantId}/jobs/job1`);

      await assertFails(jobRef.get());
    });

    it('should allow team member to read jobs_public', async () => {
      const tenantId = 'tenant123';
      const teamMemberId = 'user456';

      await testEnv.withSecurityRulesDisabled(async (context: any) => {
        await context.firestore().doc(`tenants/${tenantId}/members/${teamMemberId}`).set({
          tenantId,
          role: 'teamMember',
          status: 'active'
        });

        await context.firestore().doc(`tenants/${tenantId}/jobs_public/job1`).set({
          jobNumber: 1,
          title: 'Test Job',
          status: 'active'
        });
      });

      const memberContext = testEnv.authenticatedContext(teamMemberId, { tenant_id: tenantId });
      const jobPublicRef = memberContext.firestore().doc(`tenants/${tenantId}/jobs_public/job1`);

      await assertSucceeds(jobPublicRef.get());
    });
  });

  describe('Audit Logs', () => {
    it('should allow owner to read audit logs', async () => {
      const tenantId = 'tenant123';
      const ownerId = 'user123';

      await testEnv.withSecurityRulesDisabled(async (context: any) => {
        await context.firestore().doc(`tenants/${tenantId}/members/${ownerId}`).set({
          role: 'owner',
          status: 'active'
        });

        await context.firestore().doc(`tenants/${tenantId}/audit_logs/log1`).set({
          operation: 'CREATE',
          collection: 'jobs',
          timestamp: new Date()
        });
      });

      const ownerContext = testEnv.authenticatedContext(ownerId);
      const logRef = ownerContext.firestore().doc(`tenants/${tenantId}/audit_logs/log1`);

      await assertSucceeds(logRef.get());
    });

    it('should deny representative from reading audit logs', async () => {
      const tenantId = 'tenant123';
      const repId = 'user456';

      await testEnv.withSecurityRulesDisabled(async (context: any) => {
        await context.firestore().doc(`tenants/${tenantId}/members/${repId}`).set({
          role: 'representative',
          status: 'active'
        });

        await context.firestore().doc(`tenants/${tenantId}/audit_logs/log1`).set({
          operation: 'CREATE',
          collection: 'jobs'
        });
      });

      const repContext = testEnv.authenticatedContext(repId);
      const logRef = repContext.firestore().doc(`tenants/${tenantId}/audit_logs/log1`);

      await assertFails(logRef.get());
    });
  });
});
```

#### 10.3.2 Running Rules Tests

```bash
# Run Security Rules tests
npm run test:rules

# With emulator (must be running)
firebase emulators:exec --only firestore "npm run test:rules"
```

### 10.4 End-to-End Testing

#### 10.4.1 Playwright Configuration

```typescript
// packages/e2e-tests/playwright.config.ts

import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30000,
  use: {
    baseURL: 'http://localhost:4200', // Angular dev server
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { browserName: 'chromium' }
    },
    {
      name: 'mobile-android',
      use: {
        browserName: 'chromium',
        ...devices['Pixel 5']
      }
    }
  ],
  webServer: {
    command: 'npm run start',
    port: 4200,
    reuseExistingServer: true
  }
});
```

#### 10.4.2 Example: Voice Flow E2E Test (with Mocks)

```typescript
// packages/e2e-tests/tests/voiceFlow.spec.ts

import { test, expect } from '@playwright/test';

test.describe('Voice Command Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Setup: Sign in with test user
    await page.goto('/login');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await page.waitForURL('/dashboard');
  });

  test('should set active job via voice command', async ({ page }) => {
    // Enable mock voice providers
    await page.evaluate(() => {
      localStorage.setItem('VOICE_MOCK_MODE', 'true');
    });

    // Navigate to Voice Command Hub
    await page.click('a[href="/voice"]');

    // Verify microphone button is visible
    const micButton = page.locator('button[aria-label="Microphone"]');
    await expect(micButton).toBeVisible();

    // Simulate voice input (mock mode)
    await micButton.click();

    // Wait for confirmation modal
    const modal = page.locator('app-voice-confirmation-modal');
    await expect(modal).toBeVisible();

    // Verify transcription displayed
    await expect(modal.locator('text=Heard: Set active job to 123')).toBeVisible();

    // Verify parsed result
    await expect(modal.locator('text=Job: 123 - Smith, Brno')).toBeVisible();

    // Accept confirmation
    await modal.locator('button:has-text("Accept")').click();

    // Verify active job banner updated
    const banner = page.locator('app-active-job-banner');
    await expect(banner).toContainText('[123] Smith, Brno');

    // Verify success message
    await expect(page.locator('text=Job 123 set as active')).toBeVisible();
  });

  test('should handle voice recognition error gracefully', async ({ page }) => {
    // Setup: Force STT error in mock mode
    await page.evaluate(() => {
      localStorage.setItem('VOICE_MOCK_ERROR', 'stt_timeout');
    });

    await page.click('a[href="/voice"]');
    await page.click('button[aria-label="Microphone"]');

    // Verify error message
    await expect(page.locator('text=Network timeout. Check connection and retry.')).toBeVisible();

    // Verify Retry and Manual Entry buttons available
    await expect(page.locator('button:has-text("Retry")')).toBeVisible();
    await expect(page.locator('button:has-text("Manual Entry")')).toBeVisible();
  });
});
```

---

## 11. Migration and Operations

### 11.1 Backup Strategies

#### 11.1.1 Automated Firestore Backups

**Cloud Scheduler + Cloud Functions:**

```typescript
// packages/functions/src/scheduled/backupFirestore.ts

import { onSchedule } from 'firebase-functions/v2/scheduler';
import { getFirestore } from 'firebase-admin/firestore';

export const dailyBackup = onSchedule(
  'every day 03:00',
  async (event) => {
    const projectId = process.env.GCLOUD_PROJECT;
    const databaseName = `projects/${projectId}/databases/(default)`;

    const client = new FirestoreAdminClient();
    const bucket = `gs://${projectId}-backups`;

    const [response] = await client.exportDocuments({
      name: databaseName,
      outputUriPrefix: `${bucket}/${new Date().toISOString()}`,
      collectionIds: [] // Empty = export all collections
    });

    console.log(`Backup operation: ${response.name}`);
  }
);
```

**Cloud Storage Lifecycle Policy:**

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "Delete" },
        "condition": { "age": 30 }
      }
    ]
  }
}
```

Retention: 30 days of daily backups.

#### 11.1.2 Manual Export

```bash
# Export all Firestore data
gcloud firestore export gs://findogai-prod-backups/manual-export-$(date +%Y%m%d)

# Export specific collections
gcloud firestore export gs://findogai-prod-backups/tenants-export \
  --collection-ids=tenants
```

### 11.2 Disaster Recovery

#### 11.2.1 Recovery Point Objective (RPO)

**Target:** 24 hours

**Strategy:** Daily automated backups at 3 AM UTC.

#### 11.2.2 Recovery Time Objective (RTO)

**Target:** 4 hours

**Procedure:**

1. **Identify Incident:** Alert from monitoring or user report
2. **Assess Scope:** Determine which data is affected
3. **Restore from Backup:**
   ```bash
   # Import from backup
   gcloud firestore import gs://findogai-prod-backups/2025-10-27T03:00:00Z
   ```
4. **Verify Data Integrity:** Run smoke tests
5. **Communicate Status:** Update users via status page

#### 11.2.3 Disaster Recovery Testing

**Quarterly DR Drill:**
1. Restore backup to staging environment
2. Verify all collections and documents intact
3. Test application functionality against restored data
4. Document any gaps or failures
5. Update recovery procedures

### 11.3 Database Migrations

#### 11.3.1 Schema Changes

**Strategy: Additive Migrations Only (No Breaking Changes)**

**Example: Adding New Field to Jobs**

```typescript
// Migration script: packages/functions/src/migrations/addBudgetField.ts

import { getFirestore } from 'firebase-admin/firestore';

export async function addBudgetField() {
  const firestore = getFirestore();
  const tenantsSnapshot = await firestore.collection('tenants').listDocuments();

  for (const tenantRef of tenantsSnapshot) {
    const jobsSnapshot = await firestore.collection(`${tenantRef.path}/jobs`).get();

    const batch = firestore.batch();
    let count = 0;

    for (const jobDoc of jobsSnapshot.docs) {
      if (!jobDoc.data().hasOwnProperty('budget')) {
        batch.update(jobDoc.ref, { budget: null });
        count++;

        // Firestore batch limit: 500 operations
        if (count % 500 === 0) {
          await batch.commit();
        }
      }
    }

    if (count % 500 !== 0) {
      await batch.commit();
    }

    console.log(`Added budget field to ${count} jobs in tenant ${tenantRef.id}`);
  }
}
```

**Execution:**

```bash
# Run migration via Cloud Functions shell
firebase functions:shell
> addBudgetField().then(() => console.log('Done'))
```

#### 11.3.2 Data Transformations

**Example: Normalize Currency Codes**

```typescript
// Migration: Convert legacy currency values to ISO 4217
export async function normalizeCurrencyCodes() {
  const firestore = getFirestore();

  const currencyMap: Record<string, string> = {
    'Kč': 'CZK',
    'CZK': 'CZK',
    '€': 'EUR',
    'EUR': 'EUR'
  };

  const tenantsSnapshot = await firestore.collection('tenants').listDocuments();

  for (const tenantRef of tenantsSnapshot) {
    const profileDoc = await firestore.doc(`${tenantRef.path}/businessProfile`).get();

    if (profileDoc.exists) {
      const oldCurrency = profileDoc.data()?.currency;
      const newCurrency = currencyMap[oldCurrency];

      if (newCurrency && newCurrency !== oldCurrency) {
        await profileDoc.ref.update({ currency: newCurrency });
        console.log(`Updated currency from ${oldCurrency} to ${newCurrency}`);
      }
    }
  }
}
```

### 11.4 Operational Runbooks

#### 11.4.1 Runbook: High Firestore Read/Write Costs

**Symptoms:**
- Firebase billing alert triggered
- Unexpected spike in Firestore operations

**Diagnosis:**
1. Check Firebase Console > Usage tab for operation breakdown
2. Query Cloud Logging for high-frequency queries:
   ```sql
   resource.type="cloud_firestore_database"
   protoPayload.methodName="google.firestore.v1.Firestore.RunQuery"
   ```
3. Identify top queries by `protoPayload.request.structuredQuery`

**Resolution:**
1. Optimize queries (add `limit()`, use indexes)
2. Reduce real-time listener frequency (use polling for non-critical data)
3. Enable offline persistence if not already enabled
4. Add caching layer for frequently accessed resources

**Prevention:**
- Set up billing alerts at 50%, 75%, 90% of budget
- Monitor query performance with Firebase Performance Monitoring

---

#### 11.4.2 Runbook: Security Rules Blocking Legitimate Requests

**Symptoms:**
- Users report "Permission denied" errors
- Cloud Logging shows `protoPayload.status.code=7` (PERMISSION_DENIED)

**Diagnosis:**
1. Check Cloud Logging for denied requests:
   ```sql
   resource.type="cloud_firestore_database"
   protoPayload.status.code=7
   ```
2. Note `protoPayload.request.path` and `protoPayload.authenticationInfo.principalEmail`
3. Test Security Rules in Firebase Console > Firestore > Rules Playground

**Resolution:**
1. Verify user has active membership at `/tenants/{tenantId}/members/{uid}`
2. Check if `tenantId` custom claim is set:
   ```typescript
   const token = await admin.auth().getUser(uid).then(u => u.customClaims);
   console.log('Custom claims:', token);
   ```
3. If missing, trigger claim refresh:
   ```typescript
   await setTenantClaim(uid, tenantId);
   ```
4. If rules are incorrect, deploy hotfix:
   ```bash
   firebase deploy --only firestore:rules
   ```

**Prevention:**
- Add integration tests for Security Rules changes
- Test rules in staging before production deployment

---

#### 11.4.3 Runbook: Cloud Function Timeout

**Symptoms:**
- User reports operation "stuck" or never completes
- Cloud Logging shows function timeout errors

**Diagnosis:**
1. Check Cloud Functions logs:
   ```bash
   firebase functions:log --only allocateSequence
   ```
2. Look for `Function execution took X ms, finished with status: timeout`
3. Identify slow operations (Firestore queries, external API calls)

**Resolution:**
1. **Short-term:** Increase timeout in function config:
   ```typescript
   export const allocateSequence = onCall({
     timeoutSeconds: 120 // Increase from default 60s
   }, async (request) => { ... });
   ```
2. **Long-term:** Optimize function logic:
   - Use transactions for atomic operations
   - Avoid sequential API calls (parallelize with `Promise.all`)
   - Cache frequently accessed data

**Prevention:**
- Set performance budgets (e.g., <5s for callable functions)
- Monitor function duration with Cloud Monitoring

---

#### 11.4.4 Runbook: Audit Log Storage Exceeding Budget

**Symptoms:**
- Firestore storage costs increasing unexpectedly
- Audit logs collection growing faster than expected

**Diagnosis:**
1. Check storage size in Firebase Console > Firestore > Usage
2. Query largest collections:
   ```bash
   gcloud firestore indexes composite describe \
     --index=projects/findogai-prod/databases/(default)/collectionGroups/audit_logs/indexes/...
   ```
3. Verify TTL cleanup function is running:
   ```bash
   firebase functions:log --only cleanupAuditLogs
   ```

**Resolution:**
1. **Immediate:** Manually delete old logs:
   ```typescript
   const cutoff = new Date(Date.now() - 365 * 24 * 60 * 60 * 1000);
   const oldLogs = await firestore
     .collectionGroup('audit_logs')
     .where('timestamp', '<', cutoff)
     .limit(10000)
     .get();

   const batch = firestore.batch();
   oldLogs.docs.forEach(doc => batch.delete(doc.ref));
   await batch.commit();
   ```

2. **Long-term:** Verify scheduled cleanup is working:
   - Check Cloud Scheduler job status
   - Increase cleanup batch size if needed

**Prevention:**
- Set up alerts for storage thresholds
- Monitor audit log growth trends monthly

---

## Conclusion

This backend architecture document provides comprehensive specifications for the FinDogAI MVP backend infrastructure. The Firebase-based serverless architecture prioritizes:

1. **Offline-First Functionality:** Firestore offline persistence ensures core features work without network connectivity
2. **Multi-Tenant Isolation:** Security Rules enforce strict tenant-level data segregation
3. **Scalability:** Serverless Cloud Functions scale automatically with demand
4. **GDPR Compliance:** EU region deployment, audit logging, and data export capabilities
5. **Cost Optimization:** Minimal Cloud Functions usage; leverage free tiers where possible
6. **Developer Experience:** Monorepo structure, TypeScript type safety, Firebase Emulator Suite

### Next Steps

1. **Review and Approval:** Stakeholder review of architecture decisions
2. **Infrastructure Setup:** Provision Firebase projects (dev, staging, prod)
3. **Implementation Kickoff:** Begin with Epic 1 (Foundation & Authentication)
4. **Monitoring Setup:** Configure alerts for performance and cost thresholds
5. **Documentation Maintenance:** Keep architecture docs synchronized with implementation

### Appendices

**A. Glossary**
- **STT:** Speech-to-Text
- **TTS:** Text-to-Speech
- **LLM:** Large Language Model
- **KWS:** Keyword Spotting
- **RBAC:** Role-Based Access Control
- **TTL:** Time-To-Live
- **WER:** Word Error Rate

**B. References**
- Firebase Documentation: https://firebase.google.com/docs
- Firestore Security Rules: https://firebase.google.com/docs/firestore/security/get-started
- Cloud Functions (2nd Gen): https://firebase.google.com/docs/functions/2nd-gen
- GDPR Compliance Guide: https://firebase.google.com/support/privacy

**C. Change Log**

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2025-10-28 | 1.0 | Initial comprehensive backend architecture document | System Architect |

---

**Document End**
