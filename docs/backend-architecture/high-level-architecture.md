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

## 2.5 Schema Evolution Strategy

### 2.5.1 Overview

FinDogAI implements tenant-level schema versioning to safely evolve the data model over time. This approach enables zero-downtime migrations, gradual rollouts, and backwards compatibility.

### 2.5.2 Tenant-Level Versioning

Each tenant maintains its own schema version independently:

```typescript
// /tenants/{tenantId}
{
  tenantId: "550e8400-e29b-41d4-a716-446655440000",
  schemaVersion: 1,
  createdAt: Timestamp,
  updatedAt: Timestamp,
  migrations: {
    2: {
      appliedAt: Timestamp,
      appliedBy: "admin-uid",
      status: "completed",
      documentsProcessed: 2500
    }
  }
}
```

**Benefits:**
- Migrate tenants incrementally (10-50 at a time)
- Rollback individual tenants if migration fails
- Test migrations in production with pilot tenants
- No forced downtime for all users simultaneously

### 2.5.3 Version Compatibility Requirements

**Client Compatibility Matrix:**

| Client Version | Min Schema | Max Schema | Status |
|----------------|-----------|-----------|--------|
| 1.0.x | v1 | v1 | Current |
| 1.1.x | v1 | v2 | Future: Dual-mode support |
| 2.0.x | v2 | v3 | Future: Drops v1 |

**Version Check Flow:**

```
1. App starts → Check tenant schema version
2. If schemaVersion < minSupported:
   → Show "Please contact support" message
   → Block app usage
3. If schemaVersion > maxSupported:
   → Show "Update required" message
   → Redirect to app store
4. If minSupported ≤ schemaVersion ≤ maxSupported:
   → Proceed with appropriate schema adapter
```

### 2.5.4 Backwards Compatibility Approach

**Read Path (Clients must support N and N+1):**

```typescript
// Client reads both old and new schema formats
interface JobReader {
  read(doc: DocumentSnapshot): Job {
    const data = doc.data();
    const version = data.schemaVersion || 1;

    switch (version) {
      case 1:
        return this.readV1(data);
      case 2:
        return this.readV2(data);
      default:
        throw new Error(`Unsupported schema version: ${version}`);
    }
  }

  private readV1(data: any): Job {
    return {
      id: data.id,
      currency: data.currency || 'CZK',
      // Map v1 fields to common interface
    };
  }

  private readV2(data: any): Job {
    return {
      id: data.id,
      currency: data.primaryCurrency,
      currencies: data.currencies,
      // Map v2 fields to common interface
    };
  }
}
```

**Write Path (Migration function transforms):**

```typescript
// Client writes in its supported version
// Migration function transforms data to new schema
async function migrateV1toV2(job: JobV1): Promise<JobV2> {
  return {
    ...job,
    primaryCurrency: job.currency,
    currencies: [job.currency],
    schemaVersion: 2
  };
}
```

### 2.5.5 Zero-Downtime Migration Strategy

**Migration Process:**

1. **Pre-Migration (T-1 week)**
   - Deploy client v1.1 with dual-schema support (reads v1 and v2)
   - All users gradually update to v1.1
   - Monitor adoption rate (target: 95% within 1 week)

2. **Migration Phase (T+0)**
   - Select pilot tenants (5-10 small tenants)
   - Run `migrateSchema` with `dryRun: true` to estimate scope
   - Execute migration for pilot tenants
   - Monitor for errors, performance issues, user feedback
   - If successful, proceed to batch migration

3. **Batch Migration (T+1 day)**
   - Migrate tenants in batches of 10-50
   - Schedule migrations during off-peak hours (2-4 AM CET)
   - Monitor each batch completion before proceeding
   - Automatic rollback if error rate > 5%

4. **Post-Migration (T+1 week)**
   - Verify all tenants migrated successfully
   - Monitor application errors and performance
   - Notify any remaining tenants on old schema

5. **Deprecation (T+3 months)**
   - Mark v1 schema as deprecated
   - Client v2.0 drops v1 support
   - Force-migrate any remaining v1 tenants

### 2.5.6 Handling Mixed Versions During Migration

**Scenario: Tenant on v1, Migration to v2 in Progress**

```
Time T0: Tenant starts on v1
├─ App reads v1 data normally
├─ App writes v1 data normally
│
Time T1: Migration triggered
├─ Tenant document updated: migrations.2.status = 'in_progress'
├─ Client detects migration in progress
├─ App shows banner: "Updating data format..."
├─ Migration function processes documents in batches
│  └─ Each batch: 500 documents updated
│     └─ Progress tracked: migrations.2.documentsProcessed
│
Time T2: Migration completes
├─ Tenant document updated: schemaVersion = 2
├─ migrations.2.status = 'completed'
├─ Client switches to v2 read/write adapters
├─ Banner dismissed: "Update complete"
│
Time T3: Normal operation on v2
└─ App reads v2 data
   └─ App writes v2 data
```

**Client Behavior During Migration:**

```typescript
async function checkMigrationStatus(tenantId: string): Promise<MigrationStatus> {
  const tenantDoc = await firestore.doc(`tenants/${tenantId}`).get();
  const data = tenantDoc.data();
  const schemaVersion = data.schemaVersion || 1;
  const migrations = data.migrations || {};

  // Check if any migration is in progress
  for (const [version, migration] of Object.entries(migrations)) {
    if (migration.status === 'in_progress') {
      return {
        inProgress: true,
        targetVersion: parseInt(version),
        currentVersion: schemaVersion,
        progress: migration.documentsProcessed || 0
      };
    }
  }

  return {
    inProgress: false,
    currentVersion: schemaVersion
  };
}

// Periodic check (every 30 seconds during migration)
const migrationStatus = await checkMigrationStatus(tenantId);
if (migrationStatus.inProgress) {
  showMigrationBanner(`Updating data... ${migrationStatus.progress} documents processed`);
} else {
  hideMigrationBanner();
}
```

### 2.5.7 Testing Migration Procedures

**Pre-Production Testing:**

1. **Unit Tests**: Test migration logic with sample data
2. **Integration Tests**: Test migration on emulator with full dataset
3. **Staging Tests**: Run migration on staging environment
4. **Pilot Tenants**: Migrate 5-10 real tenants in production
5. **Validation**: Compare data before/after migration

**Test Checklist:**

```typescript
// packages/functions/src/tests/migration.test.ts

describe('Migration v1 to v2', () => {
  it('should transform v1 job to v2 format', async () => {
    const v1Job = {
      id: 'job-1',
      currency: 'CZK',
      // ... other v1 fields
    };

    const v2Job = await migrateJobV1toV2(v1Job);

    expect(v2Job.primaryCurrency).toBe('CZK');
    expect(v2Job.currencies).toEqual(['CZK']);
    expect(v2Job.schemaVersion).toBe(2);
  });

  it('should handle missing fields gracefully', async () => {
    const v1Job = { id: 'job-1' }; // Missing currency

    const v2Job = await migrateJobV1toV2(v1Job);

    expect(v2Job.primaryCurrency).toBe('CZK'); // Default
    expect(v2Job.currencies).toEqual(['CZK']);
  });

  it('should process 1000 documents in <10 seconds', async () => {
    const startTime = Date.now();
    await migrateBatch(tenantId, 1000);
    const duration = Date.now() - startTime;

    expect(duration).toBeLessThan(10000);
  });
});
```

### 2.5.8 Version Deprecation Policy

**Lifecycle Timeline:**

| Phase | Timeline | Action |
|-------|----------|--------|
| **Release** | T+0 | v2 released, v1 fully supported |
| **Adoption** | T+1 week | Pilot migrations, monitor feedback |
| **Batch Migration** | T+2 weeks | Migrate remaining tenants in batches |
| **Deprecation Notice** | T+3 months | Mark v1 deprecated, encourage migration |
| **Migration Window** | T+6 months | All tenants should be on v2 |
| **End of Support** | T+12 months | Drop v1 client support, force-migrate remaining |

**Communication Strategy:**

1. **Email Notifications**:
   - T+0: "New features available, update recommended"
   - T+3 months: "Your schema version is deprecated, please migrate"
   - T+6 months: "Final reminder: Migration required by T+12 months"
   - T+11 months: "Forced migration scheduled in 1 month"

2. **In-App Banners**:
   - T+3 months: Dismissible info banner
   - T+6 months: Persistent warning banner
   - T+11 months: Blocking modal with countdown

3. **Admin Dashboard**:
   - Display current schema version
   - Show migration status for all tenants (admin-only)
   - One-click migration trigger (with dry-run preview)

---

[Back to Index](./index.md) | [Next: Firebase Services →](./firebase-services.md)
