[Back to Index](./index.md)

# Cloud Functions Architecture

This document provides detailed implementation patterns for Cloud Functions, including triggers, callable functions, and deployment configuration.

## Overview

Cloud Functions handle server-side operations that cannot run client-side:
- Audit logging for compliance
- Sequential number assignment for offline-created entities
- Data export for GDPR compliance
- Transactional sequence allocation

**Configuration:**
- **Region**: europe-west1 (co-located with Firestore)
- **Runtime**: Node.js 20
- **Generation**: 2nd Gen (Cloud Run-based)
- **Performance Target**: <500ms execution time (NFR14)

---

## 3.2.1 Function Categories

### Audit Logging Triggers (High Frequency)

Automatically log all tenant data changes for compliance and audit trail.

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
        author: {
          uid: data.createdBy.uid,
          memberNumber: data.createdBy.memberNumber,
          displayName: data.createdBy.displayName
        },
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

**Implementation Notes:**
- Use batched writes for multiple audit logs
- Keep audit log documents lean (avoid embedding large objects)
- Set TTL to 1 year from creation
- Index on `timestamp` and `collection` for efficient queries

---

### Sequential Number Assignment Triggers (Offline Sync)

Assign sequential numbers to documents created offline without pre-allocated numbers.

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

**Transaction Guarantees:**
- Atomic counter increment
- No duplicate sequence numbers
- Automatic retry on contention

---

### HTTPS Callable Functions

#### allocateSequence

Transactional sequence number allocation for online operations.

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
  type: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber' | 'memberNumber';
  jobId?: string; // Required for ordinalNumber allocation
}
```

**Usage from Client:**

```typescript
// packages/mobile-app/src/app/services/sequence.service.ts

async allocateJobNumber(tenantId: string): Promise<number> {
  const callable = httpsCallable<AllocateSequenceRequest, { sequenceNumber: number }>(
    this.functions,
    'allocateSequence'
  );

  const result = await callable({ tenantId, type: 'jobNumber' });
  return result.data.sequenceNumber;
}
```

---

#### createMember

Handles member creation with sequential memberNumber allocation when users join a tenant.

```typescript
// packages/functions/src/callable/createMember.ts

export const createMember = onCall<CreateMemberRequest>(
  async (request) => {
    const { tenantId, role, email, displayName } = request.data;
    const uid = request.auth?.uid;

    if (!uid || !tenantId) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    // For invites: verify inviter has permission
    // For first tenant: allow creation as owner
    const db = admin.firestore();

    // Allocate memberNumber sequentially
    let memberNumber: number;

    await db.runTransaction(async (transaction) => {
      const counterRef = db.doc(`sequences/${tenantId}/counters/memberNumber`);
      const counterDoc = await transaction.get(counterRef);

      memberNumber = (counterDoc.data()?.current || 0) + 1;

      transaction.set(counterRef, { current: memberNumber }, { merge: true });

      // Create member document
      const memberRef = db.doc(`tenants/${tenantId}/members/${uid}`);
      transaction.set(memberRef, {
        tenantId,
        memberNumber,
        displayName,
        email,
        role,
        status: 'active',
        lastSeenAt: null,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      // Create user-tenant mapping for auth flow
      const mappingRef = db.doc(`user_tenants/${uid}/memberships/${tenantId}`);
      transaction.set(mappingRef, {
        tenantId,
        role,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });
    });

    return { memberNumber: memberNumber! };
  }
);

interface CreateMemberRequest {
  tenantId: string;
  role: 'owner' | 'representative' | 'teamMember';
  email: string;
  displayName: string;
}
```

**Member Creation Flows:**

1. **First Tenant Owner** (registration):
   - User signs up with email/password
   - Creates new tenant (UUID generated)
   - Calls `createMember` with `role: 'owner'`
   - Gets `memberNumber: 1`

2. **Invite Acceptance**:
   - User accepts invite with 6-digit code
   - Cloud Function verifies invite validity
   - Calls `createMember` with preset role from invite
   - Gets next sequential memberNumber

3. **Direct Addition by Owner**:
   - Owner adds existing user to tenant
   - Calls `createMember` with specified role
   - Gets next sequential memberNumber

---

#### exportData

GDPR-compliant data export with time-limited download URLs.

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

**Helper Functions:**

```typescript
async function exportCollection(path: string): Promise<any[]> {
  const snapshot = await admin.firestore().collection(path).get();
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}

async function exportJobSubcollections(tenantId: string, subcollection: string): Promise<Record<string, any[]>> {
  const result: Record<string, any[]> = {};
  const jobsSnapshot = await admin.firestore().collection(`tenants/${tenantId}/jobs`).get();

  for (const jobDoc of jobsSnapshot.docs) {
    const subcollectionSnapshot = await jobDoc.ref.collection(subcollection).get();
    result[jobDoc.id] = subcollectionSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
  }

  return result;
}
```

---

#### migrateSchema

Tenant-level schema migration with progress tracking and rollback support.

```typescript
// packages/functions/src/callable/migrateSchema.ts

export const migrateSchema = onCall<MigrateSchemaRequest>(
  async (request) => {
    const { tenantId, targetVersion, dryRun = false } = request.data;
    const uid = request.auth?.uid;

    if (!uid || !tenantId) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    // Verify owner role (only owners can trigger migrations)
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
      throw new HttpsError('permission-denied', 'Owner role required for migrations');
    }

    const db = admin.firestore();
    const tenantRef = db.doc(`tenants/${tenantId}`);
    const tenantDoc = await tenantRef.get();

    if (!tenantDoc.exists) {
      throw new HttpsError('not-found', 'Tenant not found');
    }

    const currentVersion = tenantDoc.data()?.schemaVersion || 1;

    // Validate migration path
    if (targetVersion <= currentVersion) {
      throw new HttpsError('failed-precondition',
        `Cannot migrate to v${targetVersion}. Current version is v${currentVersion}`);
    }

    if (targetVersion - currentVersion > 1) {
      throw new HttpsError('failed-precondition',
        `Cannot skip versions. Migrate to v${currentVersion + 1} first`);
    }

    // Check if migration already in progress
    const migrations = tenantDoc.data()?.migrations || {};
    if (migrations[targetVersion]?.status === 'in_progress') {
      throw new HttpsError('failed-precondition',
        `Migration to v${targetVersion} already in progress`);
    }

    if (dryRun) {
      // Dry run: estimate documents to migrate
      const estimate = await estimateMigration(tenantId, targetVersion);
      return {
        dryRun: true,
        currentVersion,
        targetVersion,
        estimatedDocuments: estimate.totalDocuments,
        estimatedDuration: estimate.estimatedMinutes,
        collections: estimate.collections
      };
    }

    // Mark migration as in_progress
    await tenantRef.update({
      [`migrations.${targetVersion}`]: {
        appliedAt: admin.firestore.FieldValue.serverTimestamp(),
        appliedBy: uid,
        status: 'in_progress',
        documentsProcessed: 0,
        error: null
      }
    });

    // Execute migration (async, non-blocking)
    executeMigration(tenantId, currentVersion, targetVersion, uid)
      .then(async (result) => {
        // Update tenant with new schema version
        await tenantRef.update({
          schemaVersion: targetVersion,
          updatedAt: admin.firestore.FieldValue.serverTimestamp(),
          [`migrations.${targetVersion}.status`]: 'completed',
          [`migrations.${targetVersion}.documentsProcessed`]: result.documentsProcessed
        });
        console.log(`Migration to v${targetVersion} completed for tenant ${tenantId}`);
      })
      .catch(async (error) => {
        // Mark migration as failed
        await tenantRef.update({
          [`migrations.${targetVersion}.status`]: 'failed',
          [`migrations.${targetVersion}.error`]: error.message
        });
        console.error(`Migration to v${targetVersion} failed for tenant ${tenantId}:`, error);
      });

    return {
      dryRun: false,
      currentVersion,
      targetVersion,
      status: 'in_progress',
      message: 'Migration started. Check migration status in tenant document.'
    };
  }
);

interface MigrateSchemaRequest {
  tenantId: string;
  targetVersion: number;
  dryRun?: boolean;
}

// Helper: Estimate migration scope
async function estimateMigration(
  tenantId: string,
  targetVersion: number
): Promise<MigrationEstimate> {
  const db = admin.firestore();
  const migrator = getMigrator(targetVersion);

  const collections = await migrator.getAffectedCollections(tenantId);
  let totalDocuments = 0;

  const collectionCounts: Record<string, number> = {};

  for (const collectionPath of collections) {
    const snapshot = await db.collection(collectionPath).count().get();
    const count = snapshot.data().count;
    collectionCounts[collectionPath] = count;
    totalDocuments += count;
  }

  // Estimate: ~100 documents per second
  const estimatedMinutes = Math.ceil(totalDocuments / 100 / 60);

  return {
    totalDocuments,
    estimatedMinutes,
    collections: collectionCounts
  };
}

interface MigrationEstimate {
  totalDocuments: number;
  estimatedMinutes: number;
  collections: Record<string, number>;
}

// Helper: Execute migration
async function executeMigration(
  tenantId: string,
  fromVersion: number,
  toVersion: number,
  triggeredBy: string
): Promise<{ documentsProcessed: number }> {
  const migrator = getMigrator(toVersion);
  return await migrator.migrate(tenantId, triggeredBy);
}

// Helper: Get migrator for target version
function getMigrator(targetVersion: number): SchemaMigrator {
  switch (targetVersion) {
    case 2:
      return new MigratorV1toV2();
    case 3:
      return new MigratorV2toV3();
    default:
      throw new Error(`No migrator available for version ${targetVersion}`);
  }
}

// Base migrator interface
interface SchemaMigrator {
  migrate(tenantId: string, triggeredBy: string): Promise<{ documentsProcessed: number }>;
  rollback(tenantId: string): Promise<void>;
  getAffectedCollections(tenantId: string): Promise<string[]>;
}

// Example: Migration from v1 to v2 (template)
class MigratorV1toV2 implements SchemaMigrator {
  async migrate(tenantId: string, triggeredBy: string): Promise<{ documentsProcessed: number }> {
    const db = admin.firestore();
    let totalProcessed = 0;

    // Example: Add new field to all jobs
    const jobsRef = db.collection(`tenants/${tenantId}/jobs`);
    const jobsSnapshot = await jobsRef.get();

    const batchSize = 500;
    let batch = db.batch();
    let batchCount = 0;

    for (const jobDoc of jobsSnapshot.docs) {
      const data = jobDoc.data();

      // Migration logic: Add new fields, transform data, etc.
      batch.update(jobDoc.ref, {
        // Example: Add multi-currency support
        primaryCurrency: data.currency || 'CZK',
        currencies: [data.currency || 'CZK'],
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      batchCount++;
      totalProcessed++;

      // Commit batch every 500 operations
      if (batchCount >= batchSize) {
        await batch.commit();
        batch = db.batch();
        batchCount = 0;

        // Update progress in tenant document
        await db.doc(`tenants/${tenantId}`).update({
          'migrations.2.documentsProcessed': totalProcessed
        });
      }
    }

    // Commit remaining operations
    if (batchCount > 0) {
      await batch.commit();
    }

    return { documentsProcessed: totalProcessed };
  }

  async rollback(tenantId: string): Promise<void> {
    // Rollback logic: restore original schema
    const db = admin.firestore();
    const jobsRef = db.collection(`tenants/${tenantId}/jobs`);
    const jobsSnapshot = await jobsRef.get();

    const batch = db.batch();
    for (const jobDoc of jobsSnapshot.docs) {
      batch.update(jobDoc.ref, {
        primaryCurrency: admin.firestore.FieldValue.delete(),
        currencies: admin.firestore.FieldValue.delete()
      });
    }

    await batch.commit();
  }

  async getAffectedCollections(tenantId: string): Promise<string[]> {
    return [
      `tenants/${tenantId}/jobs`,
      `tenants/${tenantId}/vehicles`,
      `tenants/${tenantId}/machines`
    ];
  }
}
```

**Usage from Admin UI:**

```typescript
// Dry run first to estimate scope
const estimate = await migrateSchema({
  tenantId: 'tenant-123',
  targetVersion: 2,
  dryRun: true
});

console.log(`Migration will process ${estimate.estimatedDocuments} documents`);
console.log(`Estimated duration: ${estimate.estimatedDuration} minutes`);

// Trigger actual migration
const result = await migrateSchema({
  tenantId: 'tenant-123',
  targetVersion: 2,
  dryRun: false
});

// Monitor progress by watching tenant document
const unsubscribe = onSnapshot(
  doc(firestore, `tenants/tenant-123`),
  (snapshot) => {
    const migrations = snapshot.data()?.migrations;
    const migration = migrations?.[2];

    if (migration?.status === 'completed') {
      console.log('Migration completed!');
      unsubscribe();
    } else if (migration?.status === 'failed') {
      console.error('Migration failed:', migration.error);
      unsubscribe();
    } else if (migration?.status === 'in_progress') {
      console.log(`Processed: ${migration.documentsProcessed} documents`);
    }
  }
);
```

---

### Scheduled Functions

#### checkSchemaMigrations

Daily check for tenants requiring schema migrations or running migrations that have stalled.

```typescript
// packages/functions/src/scheduled/checkSchemaMigrations.ts

export const checkSchemaMigrations = onSchedule(
  {
    schedule: 'every day 04:00',
    timeZone: 'Europe/Prague',
    region: 'europe-west1'
  },
  async (event) => {
    const db = admin.firestore();
    const CURRENT_SCHEMA_VERSION = 1; // Update as new versions are released
    const STALLED_THRESHOLD_HOURS = 4;

    const tenantsSnapshot = await db.collection('tenants').get();

    for (const tenantDoc of tenantsSnapshot.docs) {
      const tenantId = tenantDoc.id;
      const data = tenantDoc.data();
      const schemaVersion = data.schemaVersion || 1;
      const migrations = data.migrations || {};

      // Check for outdated schema versions
      if (schemaVersion < CURRENT_SCHEMA_VERSION) {
        const versionsOutdated = CURRENT_SCHEMA_VERSION - schemaVersion;
        console.warn(
          `Tenant ${tenantId} is ${versionsOutdated} version(s) behind (current: v${schemaVersion}, latest: v${CURRENT_SCHEMA_VERSION})`
        );

        // Notify tenant owners (send email, in-app notification, etc.)
        await notifyTenantOwners(tenantId, {
          type: 'schema_outdated',
          message: `Your data schema is outdated. Please migrate to v${CURRENT_SCHEMA_VERSION}.`,
          currentVersion: schemaVersion,
          targetVersion: CURRENT_SCHEMA_VERSION
        });
      }

      // Check for stalled migrations
      for (const [version, migration] of Object.entries(migrations)) {
        if (migration.status === 'in_progress') {
          const appliedAt = migration.appliedAt.toDate();
          const hoursSince = (Date.now() - appliedAt.getTime()) / (1000 * 60 * 60);

          if (hoursSince > STALLED_THRESHOLD_HOURS) {
            console.error(
              `Migration to v${version} for tenant ${tenantId} has been in progress for ${hoursSince.toFixed(1)} hours`
            );

            // Mark as failed and notify
            await db.doc(`tenants/${tenantId}`).update({
              [`migrations.${version}.status`]: 'failed',
              [`migrations.${version}.error`]: 'Migration stalled (exceeded 4-hour threshold)'
            });

            await notifyTenantOwners(tenantId, {
              type: 'migration_failed',
              message: `Schema migration to v${version} has failed. Please contact support.`,
              version: parseInt(version),
              error: 'Migration stalled'
            });
          }
        }
      }
    }

    console.log('Schema migration check completed');
  }
);

async function notifyTenantOwners(tenantId: string, notification: any): Promise<void> {
  // Implementation: Send email, push notification, or create in-app notification
  const db = admin.firestore();
  const membersSnapshot = await db
    .collection(`tenants/${tenantId}/members`)
    .where('role', '==', 'owner')
    .get();

  for (const memberDoc of membersSnapshot.docs) {
    // Send notification to owner
    console.log(`Notifying owner ${memberDoc.id} of tenant ${tenantId}:`, notification);
    // TODO: Implement actual notification mechanism (email, FCM, etc.)
  }
}
```

---

#### cleanupAuditLogs

Daily cleanup of audit logs older than 1 year (GDPR compliance).

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

**Schedule**: Runs daily at 2:00 AM (europe-west1 time zone)

**Batch Processing**: Limits to 500 documents per tenant per run to avoid timeouts

---

## 3.2.2 Function Deployment Configuration

### Package Configuration

```json
// packages/functions/package.json
{
  "name": "findogai-functions",
  "version": "1.0.0",
  "engines": {
    "node": "20"
  },
  "main": "lib/index.js",
  "scripts": {
    "build": "tsc",
    "serve": "npm run build && firebase emulators:start --only functions",
    "shell": "npm run build && firebase functions:shell",
    "deploy": "firebase deploy --only functions",
    "logs": "firebase functions:log"
  },
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

### Firebase Configuration

```json
// firebase.json
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

**Configuration Options:**
- **runtime**: Node.js 20 (latest LTS)
- **region**: europe-west1 (GDPR compliance, co-located with Firestore)
- **concurrency**: 80 requests per instance (2nd Gen default)
- **memory**: 256MB (sufficient for most operations)
- **timeout**: 60s (default, increase for long-running operations)

### TypeScript Configuration

```json
// packages/functions/tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es2020",
    "lib": ["es2020"],
    "outDir": "lib",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

---

## Function Organization

```
packages/functions/
├── src/
│   ├── index.ts                 # Export all functions
│   ├── triggers/
│   │   ├── auditLogger.ts       # Audit logging triggers
│   │   └── assignSequence.ts    # Sequential number assignment
│   ├── callable/
│   │   ├── allocateSequence.ts  # Sequence allocation
│   │   ├── createMember.ts      # Member creation with memberNumber
│   │   ├── exportData.ts        # Data export
│   │   └── generatePDF.ts       # PDF generation (Phase 2)
│   ├── scheduled/
│   │   └── cleanupAuditLogs.ts  # Daily audit log cleanup
│   └── utils/
│       ├── validators.ts        # Input validation
│       └── helpers.ts           # Shared utilities
├── package.json
├── tsconfig.json
└── .eslintrc.js
```

### index.ts

```typescript
// packages/functions/src/index.ts

// Triggers
export * from './triggers/auditLogger';
export * from './triggers/assignSequence';

// Callable functions
export * from './callable/allocateSequence';
export * from './callable/createMember';
export * from './callable/exportData';
export * from './callable/migrateSchema';

// Scheduled functions
export * from './scheduled/cleanupAuditLogs';
export * from './scheduled/checkSchemaMigrations';
```

---

## Deployment

### Development Deployment

```bash
# Build and deploy all functions
cd packages/functions
npm run build
firebase deploy --only functions --project findogai-dev

# Deploy specific function
firebase deploy --only functions:allocateSequence --project findogai-dev
```

### Production Deployment

```bash
# Deploy to production (with confirmation)
firebase deploy --only functions --project findogai-prod

# View deployment logs
firebase functions:log --project findogai-prod
```

See [Development & Deployment](./development-deployment.md) for CI/CD pipeline configuration.

---

## Testing

### Local Emulator

```bash
# Start Firebase Emulator Suite
firebase emulators:start --only functions,firestore

# Test callable functions
firebase functions:shell
> allocateSequence({ tenantId: 'test-123', type: 'jobNumber' })
```

### Unit Tests

See [Testing Strategy - Unit Testing for Cloud Functions](./testing-strategy.md#101-unit-testing-for-cloud-functions) for comprehensive testing approach.

---

## Performance Optimization

### Cold Start Mitigation

- Keep function code minimal
- Lazy-load heavy dependencies
- Use global variables for reusable connections
- Consider min instances for critical functions (production only)

### Cost Optimization

- Minimize function invocations (batch operations when possible)
- Use appropriate memory allocation (256MB default)
- Set realistic timeouts (avoid 540s default)
- Leverage Firestore triggers instead of polling

See [Performance Optimization](./performance-optimization.md) for detailed strategies.

---

[← Back: Data Model](./data-model.md) | [Back to Index](./index.md) | [Next: Security Rules →](./security-rules.md)
