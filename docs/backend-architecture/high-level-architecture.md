[Back to Index](./index.md)

# High Level Architecture

## 2.1 Firebase Services Overview

FinDogAI leverages four core Firebase services:

### Firebase Authentication
- **Purpose**: User identity management
- **Region**: Global (identity tokens)
- **Features Used**:
  - Email/password authentication (MVP)
  - Custom claims for tenant ID injection
  - OAuth providers (Phase 2: Google, Apple)
- **Multi-Tenancy**: Custom claim `tenant_id` added after user-to-tenant mapping resolution

### Cloud Firestore
- **Purpose**: Primary database for tenant data
- **Region**: europe-west1 (Belgium) for GDPR compliance
- **Features Used**:
  - Offline persistence with automatic sync
  - Real-time listeners for UI updates
  - Subcollections for hierarchical data (jobs > costs, jobs > advances)
  - Composite indexes for queries
  - Security Rules for authorization
- **Data Model**: Tenant-scoped collections under `/tenants/{tenantId}/`

### Cloud Functions (2nd Gen)
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

### Cloud Storage for Firebase
- **Purpose**: File storage for exports and attachments
- **Region**: europe-west1
- **Features Used**:
  - Signed URLs for secure download
  - Lifecycle policies for automatic cleanup (exports expire after 7 days)
  - Security Rules for access control

## 2.2 System Architecture Diagram Description

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

## 2.3 Data Flow Between Components

### 2.3.1 User Authentication Flow

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

### 2.3.2 Voice Command Flow (Online)

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

### 2.3.3 Offline Data Sync Flow

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

### 2.3.4 Sequential Number Allocation

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

## 2.4 Multi-Tenant Architecture

### 2.4.1 Isolation Strategy

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

### 2.4.2 Authorization Model

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

### 2.4.3 Custom Claims

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

[Back to Index](./index.md) | [Next: Firebase Services →](./firebase-services.md)
