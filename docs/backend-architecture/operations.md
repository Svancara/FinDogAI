[Back to Index](./index.md)

# Operations

This document covers backup strategies, disaster recovery, database migrations, and operational runbooks for the FinDogAI backend.

## Overview

Operational considerations:
- **Backup**: Automated daily backups with 30-day retention
- **Disaster Recovery**: RPO 24 hours, RTO 4 hours
- **Migrations**: Schema changes via Cloud Functions
- **Runbooks**: Troubleshooting guides for common issues

---

## 11.1 Backup Strategies

### 11.1.1 Automated Firestore Backups

**Cloud Scheduler + Cloud Functions:**

```typescript
// packages/functions/src/scheduled/backupFirestore.ts

import { onSchedule } from 'firebase-functions/v2/scheduler';
import { firestore } from 'firebase-admin';

export const backupFirestore = onSchedule(
  {
    schedule: 'every day 03:00',
    timeZone: 'Europe/Prague',
    region: 'europe-west1'
  },
  async (event) => {
    const projectId = process.env.GCLOUD_PROJECT;
    const timestamp = new Date().toISOString().split('T')[0];
    const bucket = `gs://${projectId}-backups`;

    await firestore().database().exportDocuments({
      outputUriPrefix: `${bucket}/firestore-${timestamp}`,
      collectionIds: [] // Empty = all collections
    });

    console.log(`Firestore backup completed: ${bucket}/firestore-${timestamp}`);
  }
);
```

**Backup Configuration:**
- **Schedule**: Daily at 3:00 AM CET
- **Retention**: 30 days (managed by Cloud Storage lifecycle policy)
- **Location**: `gs://findogai-prod-backups/firestore-YYYY-MM-DD/`
- **Format**: Firestore export format (includes all collections)

**Lifecycle Policy for Backups:**

```bash
# Set 30-day lifecycle on backup bucket
gsutil lifecycle set lifecycle.json gs://findogai-prod-backups

# lifecycle.json
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

### 11.1.2 Manual Export

**Via Firebase Console:**
1. Firestore > Import/Export
2. Select collections (or leave empty for all)
3. Choose Cloud Storage bucket
4. Click "Export"

**Via gcloud CLI:**

```bash
# Export all collections
gcloud firestore export gs://findogai-prod-backups/manual-export-$(date +%Y%m%d)

# Export specific collections
gcloud firestore export gs://findogai-prod-backups/manual-export-$(date +%Y%m%d) \
  --collection-ids=tenants,user_tenants,sequences
```

---

## 11.2 Disaster Recovery

### 11.2.1 Recovery Point Objective (RPO)

**Target RPO: 24 hours** (acceptable data loss)

**Backup Strategy:**
- Automated daily backups at 3:00 AM
- Worst case: lose up to 24 hours of data
- Critical data (costs, jobs) cached locally in mobile app
- Users can re-sync from local cache after recovery

### 11.2.2 Recovery Time Objective (RTO)

**Target RTO: 4 hours** (time to restore service)

**Recovery Steps:**

1. **Assess Damage** (30 minutes)
   - Identify affected collections
   - Determine recovery point (which backup to use)

2. **Restore from Backup** (2 hours)
```bash
# Import from backup
gcloud firestore import gs://findogai-prod-backups/firestore-2025-10-28/

# Monitor import progress
gcloud firestore operations list
```

3. **Verify Data Integrity** (1 hour)
   - Check critical collections (jobs, costs, vehicles)
   - Verify user memberships and permissions
   - Test authentication flow

4. **Resume Service** (30 minutes)
   - Re-deploy Cloud Functions if needed
   - Notify users of service restoration
   - Monitor for errors

### 11.2.3 Disaster Recovery Testing

**Quarterly DR Test:**

```bash
# 1. Create test project
gcloud projects create findogai-dr-test

# 2. Restore backup to test project
gcloud firestore import gs://findogai-prod-backups/firestore-latest/ \
  --project=findogai-dr-test

# 3. Deploy functions to test project
firebase use findogai-dr-test
firebase deploy --only functions

# 4. Verify functionality
npm run test:e2e -- --project=findogai-dr-test

# 5. Document results and cleanup
gcloud projects delete findogai-dr-test
```

---

## 11.3 Database Migrations

### 11.3.1 Schema Changes

**Adding a New Field:**

```typescript
// packages/functions/src/migrations/addCurrencyToVehicles.ts

import { onSchedule } from 'firebase-functions/v2/scheduler';
import { firestore } from 'firebase-admin';

export const addCurrencyToVehicles = onSchedule(
  {
    schedule: 'every sunday 02:00', // Run once manually
    region: 'europe-west1'
  },
  async (event) => {
    const db = firestore();
    const tenantsSnapshot = await db.collection('tenants').listDocuments();

    for (const tenantRef of tenantsSnapshot) {
      const vehiclesSnapshot = await db
        .collection(`${tenantRef.path}/vehicles`)
        .get();

      const batch = db.batch();
      let count = 0;

      for (const vehicleDoc of vehiclesSnapshot.docs) {
        const data = vehicleDoc.data();

        // Only update if currency field missing
        if (!data.currency) {
          batch.update(vehicleDoc.ref, {
            currency: 'CZK', // Default currency
            updatedAt: firestore.FieldValue.serverTimestamp()
          });
          count++;

          // Commit batch every 500 operations
          if (count % 500 === 0) {
            await batch.commit();
          }
        }
      }

      // Commit remaining operations
      if (count % 500 !== 0) {
        await batch.commit();
      }

      console.log(`Updated ${count} vehicles in tenant ${tenantRef.id}`);
    }
  }
);
```

**Migration Checklist:**

1. **Test in Development**
   ```bash
   firebase use findogai-dev
   firebase deploy --only functions:addCurrencyToVehicles
   firebase functions:log --only addCurrencyToVehicles
   ```

2. **Backup Production**
   ```bash
   gcloud firestore export gs://findogai-prod-backups/pre-migration-$(date +%Y%m%d)
   ```

3. **Deploy to Production**
   ```bash
   firebase use findogai-prod
   firebase deploy --only functions:addCurrencyToVehicles
   ```

4. **Trigger Migration**
   ```bash
   # Manually trigger via Cloud Scheduler or gcloud
   gcloud scheduler jobs run addCurrencyToVehicles --location=europe-west1
   ```

5. **Monitor Progress**
   ```bash
   firebase functions:log --only addCurrencyToVehicles
   ```

6. **Verify Results**
   ```bash
   # Query Firestore to verify all vehicles have currency field
   ```

7. **Remove Migration Function**
   ```bash
   # After successful migration, remove from index.ts
   firebase deploy --only functions
   ```

### 11.3.2 Data Transformations

**Renaming a Field:**

```typescript
export const renameCostField = onSchedule(
  { schedule: 'every sunday 02:00' },
  async (event) => {
    const db = firestore();

    // Process all tenants
    const tenantsSnapshot = await db.collection('tenants').listDocuments();

    for (const tenantRef of tenantsSnapshot) {
      // Process all jobs
      const jobsSnapshot = await db.collection(`${tenantRef.path}/jobs`).get();

      for (const jobDoc of jobsSnapshot.docs) {
        // Process all costs subcollection
        const costsSnapshot = await db
          .collection(`${jobDoc.ref.path}/costs`)
          .get();

        const batch = db.batch();

        for (const costDoc of costsSnapshot.docs) {
          const data = costDoc.data();

          // Rename field: oldFieldName -> newFieldName
          if (data.oldFieldName !== undefined) {
            batch.update(costDoc.ref, {
              newFieldName: data.oldFieldName,
              oldFieldName: firestore.FieldValue.delete()
            });
          }
        }

        await batch.commit();
      }

      console.log(`Migrated costs for tenant ${tenantRef.id}`);
    }
  }
);
```

---

## 11.4 Operational Runbooks

### 11.4.1 Runbook: High Firestore Read/Write Costs

**Symptoms:**
- Unexpected Firebase bill
- High read/write counts in Firebase Console

**Diagnosis:**

```bash
# 1. Check usage in Firebase Console
# Firestore > Usage tab

# 2. Enable detailed logging
firebase functions:config:set logging.level=debug
firebase deploy --only functions

# 3. Identify hot collections
# Look for collections with high read/write counts

# 4. Check for inefficient queries
# Review Cloud Functions logs for slow queries
firebase functions:log --limit 100 | grep "slow query"
```

**Resolution:**

1. **Add Missing Indexes:**
```bash
firebase deploy --only firestore:indexes
```

2. **Optimize Queries:**
```typescript
// Bad: No limit
const allJobs = await getDocs(collection(firestore, 'jobs'));

// Good: Limited results
const recentJobs = await getDocs(
  query(collection(firestore, 'jobs'), limit(50))
);
```

3. **Increase Cache Usage:**
```typescript
// Enable offline persistence
await enableIndexedDbPersistence(firestore);
```

4. **Review Real-Time Listeners:**
```typescript
// Bad: Global listener never unsubscribed
onSnapshot(collection(firestore, 'jobs'), callback);

// Good: Unsubscribe when component unmounts
const unsubscribe = onSnapshot(collection(firestore, 'jobs'), callback);
ngOnDestroy() { unsubscribe(); }
```

### 11.4.2 Runbook: Security Rules Blocking Legitimate Requests

**Symptoms:**
- Users report "permission denied" errors
- Cloud Functions logs show security rule rejections

**Diagnosis:**

```bash
# 1. Check security rules in Firebase Console
# Firestore > Rules tab

# 2. Enable rules debugging
# Add console.log statements to rules

# 3. Test rules locally
firebase emulators:start --only firestore
npm run test:rules

# 4. Check user membership
# Verify user has active membership in tenant
```

**Resolution:**

1. **Verify User Membership:**
```typescript
const memberDoc = await getDoc(
  doc(firestore, `tenants/${tenantId}/members/${userId}`)
);

if (!memberDoc.exists || memberDoc.data().status !== 'active') {
  console.error('User not an active member');
}
```

2. **Check Custom Claims:**
```typescript
const idTokenResult = await auth.currentUser.getIdTokenResult();
console.log('Custom claims:', idTokenResult.claims);

// Force token refresh if stale
await auth.currentUser.getIdToken(true);
```

3. **Update Security Rules (if needed):**
```javascript
// Test rules change locally first
firebase emulators:start --only firestore
npm run test:rules

// Deploy updated rules
firebase deploy --only firestore:rules
```

### 11.4.3 Runbook: Cloud Function Timeout

**Symptoms:**
- Cloud Functions logs show timeout errors (60s)
- Users report slow operations

**Diagnosis:**

```bash
# 1. Check function execution time
firebase functions:log --only allocateSequence --limit 100

# 2. Look for timeout patterns
# Functions > Metrics in Firebase Console

# 3. Check cold start frequency
# Look for "Function execution started" vs "Function execution took" in logs
```

**Resolution:**

1. **Optimize Function Code:**
```typescript
// Bad: Heavy import
import * as admin from 'firebase-admin';

// Good: Selective import
import { firestore } from 'firebase-admin/firestore';
```

2. **Increase Memory (if CPU-bound):**
```typescript
export const heavyFunction = onCall(
  { memory: '512MB' }, // Increase from 256MB default
  async (request) => { /* ... */ }
);
```

3. **Increase Timeout (if necessary):**
```typescript
export const longRunningFunction = onCall(
  { timeout: 120 }, // Increase from 60s default
  async (request) => { /* ... */ }
);
```

4. **Use Min Instances (Production Only):**
```typescript
export const criticalFunction = onCall(
  { minInstances: 1 }, // Keep warm instance
  async (request) => { /* ... */ }
);
```

### 11.4.4 Runbook: Audit Log Storage Exceeding Budget

**Symptoms:**
- Firestore storage costs increasing
- Audit logs collection growing large

**Diagnosis:**

```bash
# Check audit log count
gcloud firestore collections list-documents \
  --collection-ids=audit_logs \
  --format="value(name)" | wc -l

# Check storage size
# Firebase Console > Firestore > Usage tab
```

**Resolution:**

1. **Verify Cleanup Function Running:**
```bash
# Check last execution
firebase functions:log --only cleanupAuditLogs --limit 10

# Manually trigger if needed
gcloud scheduler jobs run cleanupAuditLogs --location=europe-west1
```

2. **Adjust TTL (if needed):**
```typescript
// Reduce retention from 1 year to 90 days
ttl: admin.firestore.Timestamp.fromDate(
  new Date(Date.now() + 90 * 24 * 60 * 60 * 1000)
)
```

3. **Manual Cleanup (Emergency):**
```typescript
// Create one-time cleanup function
export const emergencyCleanup = onCall(async (request) => {
  const db = admin.firestore();
  const cutoffDate = admin.firestore.Timestamp.fromDate(
    new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) // 30 days
  );

  // Delete in batches
  const expiredLogs = await db
    .collectionGroup('audit_logs')
    .where('timestamp', '<', cutoffDate)
    .limit(5000)
    .get();

  const batch = db.batch();
  expiredLogs.docs.forEach(doc => batch.delete(doc.ref));
  await batch.commit();

  return { deleted: expiredLogs.size };
});
```

---

## 11.3 Schema Migration Procedures

### 11.3.1 Overview

Schema migrations allow the FinDogAI data model to evolve while maintaining backwards compatibility and zero downtime. This section provides step-by-step procedures for planning, executing, and monitoring schema migrations.

### 11.3.2 Pre-Migration Checklist

**Before Starting Any Migration:**

- [ ] **Document Changes**
  - [ ] Create migration specification document
  - [ ] List all affected collections and fields
  - [ ] Document transformation logic
  - [ ] Identify backwards compatibility requirements

- [ ] **Code Preparation**
  - [ ] Implement migration function (e.g., `MigratorV1toV2`)
  - [ ] Implement rollback function
  - [ ] Add unit tests for migration logic
  - [ ] Update client code to support both old and new schemas

- [ ] **Testing**
  - [ ] Test migration on local emulator
  - [ ] Test migration on staging environment
  - [ ] Verify rollback procedure works
  - [ ] Performance test with realistic data volume
  - [ ] Test client compatibility with both schema versions

- [ ] **Backup**
  - [ ] Create manual Firestore backup
  - [ ] Document backup location
  - [ ] Verify backup restoration procedure

- [ ] **Communication**
  - [ ] Notify stakeholders of migration schedule
  - [ ] Prepare user-facing communication (email, in-app banner)
  - [ ] Schedule during low-traffic hours (2-4 AM CET)

- [ ] **Monitoring**
  - [ ] Set up error rate alerts
  - [ ] Prepare migration progress dashboard
  - [ ] Assign on-call engineer for migration window

### 11.3.3 Step-by-Step Migration Process

#### Phase 1: Preparation (T-1 week)

**1. Deploy Dual-Schema Client Version**

```bash
# Deploy client v1.1 that supports both v1 and v2 schemas
cd packages/mobile-app
npm run build
# Deploy to app stores
```

**2. Monitor Client Adoption**

```typescript
// Track client version adoption
const stats = await analytics.getClientVersionStats();
console.log(`v1.1 adoption: ${stats.v1_1_percentage}%`);

// Target: 95% adoption before proceeding with migration
```

**3. Deploy Migration Cloud Function**

```bash
cd packages/functions
npm run build
firebase deploy --only functions:migrateSchema,functions:checkSchemaMigrations
```

#### Phase 2: Pilot Migration (T+0 days)

**1. Select Pilot Tenants**

```typescript
// Select 5-10 small tenants with low activity
const pilotTenants = await selectPilotTenants({
  criteria: {
    documentCount: { $lt: 1000 },
    activeUsers: { $lt: 5 },
    lastActivity: { $gt: Date.now() - 7 * 24 * 60 * 60 * 1000 }
  },
  count: 10
});
```

**2. Run Dry-Run Estimation**

```bash
# For each pilot tenant, estimate migration scope
firebase functions:call migrateSchema --data='{
  "tenantId": "pilot-tenant-1",
  "targetVersion": 2,
  "dryRun": true
}'

# Output:
# {
#   "estimatedDocuments": 750,
#   "estimatedDuration": 8,  // minutes
#   "collections": {
#     "tenants/pilot-tenant-1/jobs": 250,
#     "tenants/pilot-tenant-1/costs": 500
#   }
# }
```

**3. Execute Pilot Migrations**

```bash
# Migrate first pilot tenant
firebase functions:call migrateSchema --data='{
  "tenantId": "pilot-tenant-1",
  "targetVersion": 2,
  "dryRun": false
}'

# Monitor progress
firebase functions:log --only migrateSchema --follow

# Verify completion
gcloud firestore documents get tenants/pilot-tenant-1 --format=json
```

**4. Verify Pilot Results**

```typescript
// Validation script
async function validateMigration(tenantId: string): Promise<ValidationResult> {
  const tenant = await firestore.doc(`tenants/${tenantId}`).get();
  const migration = tenant.data()?.migrations?.[2];

  if (migration?.status !== 'completed') {
    return { success: false, error: `Migration status: ${migration?.status}` };
  }

  // Sample check: verify 10 random jobs have new schema
  const jobs = await firestore
    .collection(`tenants/${tenantId}/jobs`)
    .limit(10)
    .get();

  for (const job of jobs.docs) {
    const data = job.data();
    if (!data.primaryCurrency || !data.currencies) {
      return {
        success: false,
        error: `Job ${job.id} missing v2 fields`
      };
    }
  }

  return { success: true };
}

// Run validation for all pilot tenants
for (const tenant of pilotTenants) {
  const result = await validateMigration(tenant.id);
  console.log(`Tenant ${tenant.id}: ${result.success ? 'PASS' : 'FAIL'}`);
  if (!result.success) {
    console.error(`  Error: ${result.error}`);
  }
}
```

**5. Monitor Pilot Tenants**

```bash
# Monitor for 24-48 hours
# Check error rates, user feedback, performance metrics

# If issues found, rollback:
firebase functions:call rollbackMigration --data='{
  "tenantId": "pilot-tenant-1",
  "fromVersion": 2,
  "toVersion": 1
}'
```

#### Phase 3: Batch Migration (T+2 days)

**1. Prepare Batch List**

```typescript
// Get all remaining tenants on v1
const tenantsToMigrate = await firestore
  .collection('tenants')
  .where('schemaVersion', '==', 1)
  .get();

// Group into batches of 10-50 by document count
const batches = groupTenantsBySize(tenantsToMigrate.docs, {
  batchSize: 20,
  maxDocumentsPerBatch: 50000
});

console.log(`Total batches: ${batches.length}`);
```

**2. Schedule Batch Migrations**

```typescript
// Schedule migrations during off-peak hours
const migrationSchedule = batches.map((batch, index) => ({
  batchId: index + 1,
  tenants: batch,
  scheduledTime: new Date(Date.now() + (index * 2 * 60 * 60 * 1000)), // 2 hours apart
  estimatedDuration: estimateBatchDuration(batch)
}));

// Save schedule
await firestore.collection('migration_schedule').add({
  version: 2,
  schedule: migrationSchedule,
  createdAt: admin.firestore.FieldValue.serverTimestamp()
});
```

**3. Execute Batch Migrations**

```typescript
// Automated batch migration script
async function executeBatchMigration(batchId: number) {
  const batch = migrationSchedule[batchId - 1];

  console.log(`Starting batch ${batchId} with ${batch.tenants.length} tenants`);

  const results = [];

  for (const tenant of batch.tenants) {
    try {
      const result = await migrateSchema({
        tenantId: tenant.id,
        targetVersion: 2,
        dryRun: false
      });

      results.push({ tenantId: tenant.id, status: 'started' });

      // Wait 5 seconds between tenant migrations to avoid throttling
      await sleep(5000);
    } catch (error) {
      console.error(`Failed to start migration for ${tenant.id}:`, error);
      results.push({ tenantId: tenant.id, status: 'error', error: error.message });
    }
  }

  // Monitor batch completion
  await monitorBatchCompletion(batch.tenants);

  return results;
}

// Execute manually or schedule with Cloud Scheduler
await executeBatchMigration(1);
```

**4. Monitor Batch Progress**

```typescript
// Real-time monitoring dashboard
async function monitorBatchCompletion(tenants: string[]): Promise<BatchStatus> {
  const status = {
    total: tenants.length,
    completed: 0,
    inProgress: 0,
    failed: 0,
    errors: []
  };

  const checkInterval = setInterval(async () => {
    status.completed = 0;
    status.inProgress = 0;
    status.failed = 0;
    status.errors = [];

    for (const tenantId of tenants) {
      const tenant = await firestore.doc(`tenants/${tenantId}`).get();
      const migration = tenant.data()?.migrations?.[2];

      if (migration?.status === 'completed') {
        status.completed++;
      } else if (migration?.status === 'in_progress') {
        status.inProgress++;
      } else if (migration?.status === 'failed') {
        status.failed++;
        status.errors.push({ tenantId, error: migration.error });
      }
    }

    console.log(`Batch progress: ${status.completed}/${status.total} completed, ${status.inProgress} in progress, ${status.failed} failed`);

    if (status.completed + status.failed === status.total) {
      clearInterval(checkInterval);
      console.log('Batch migration complete');

      if (status.failed > 0) {
        console.error('Failed migrations:', status.errors);
      }
    }
  }, 30000); // Check every 30 seconds

  return status;
}
```

**5. Handle Failures**

```bash
# If error rate > 5%, pause and investigate
if [ $ERROR_RATE -gt 5 ]; then
  echo "Error rate exceeds threshold, pausing migration"
  # Stop scheduled migrations
  # Investigate failures
  # Fix issues before continuing
fi

# Retry failed migrations
firebase functions:call migrateSchema --data='{
  "tenantId": "failed-tenant-id",
  "targetVersion": 2,
  "dryRun": false
}'
```

#### Phase 4: Post-Migration Verification (T+1 week)

**1. Verify All Tenants Migrated**

```bash
# Check for any remaining v1 tenants
gcloud firestore documents query tenants \
  --filter='schemaVersion=1' \
  --format='value(name)'

# Expected: Empty list
```

**2. Monitor Error Rates**

```bash
# Check Cloud Functions logs for schema-related errors
firebase functions:log --only migrateSchema,checkSchemaMigrations \
  --since '1 week ago' | grep ERROR

# Check client error logs for compatibility issues
```

**3. Notify Remaining Tenants**

```typescript
// Send email to tenants still on old schema
const oldSchemaTenants = await firestore
  .collection('tenants')
  .where('schemaVersion', '<', 2)
  .get();

for (const tenant of oldSchemaTenants.docs) {
  await notifyTenantOwners(tenant.id, {
    type: 'migration_required',
    message: 'Please migrate to the latest schema version',
    targetVersion: 2
  });
}
```

### 11.3.4 Rollback Procedures

**When to Rollback:**
- Error rate exceeds 10% in any batch
- Critical data corruption detected
- Performance degradation observed
- User-reported issues spike

**Rollback Steps:**

**1. Pause Ongoing Migrations**

```typescript
// Stop scheduled migrations
await cloudScheduler.pauseJob('schema-migration-batch');

// Mark in-progress migrations as paused
const inProgressMigrations = await firestore
  .collectionGroup('tenants')
  .where('migrations.2.status', '==', 'in_progress')
  .get();

for (const tenant of inProgressMigrations.docs) {
  await tenant.ref.update({
    'migrations.2.status': 'paused'
  });
}
```

**2. Execute Rollback for Affected Tenants**

```typescript
// Rollback function
export const rollbackMigration = onCall<RollbackRequest>(
  async (request) => {
    const { tenantId, fromVersion, toVersion } = request.data;
    const uid = request.auth?.uid;

    if (!uid) {
      throw new HttpsError('unauthenticated', 'User must be authenticated');
    }

    // Verify owner role
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberDoc.exists || memberDoc.data()?.role !== 'owner') {
      throw new HttpsError('permission-denied', 'Owner role required');
    }

    const db = admin.firestore();
    const migrator = getMigrator(fromVersion);

    try {
      // Execute rollback
      await migrator.rollback(tenantId);

      // Update tenant document
      await db.doc(`tenants/${tenantId}`).update({
        schemaVersion: toVersion,
        [`migrations.${fromVersion}.status`]: 'rolled_back',
        [`migrations.${fromVersion}.rolledBackAt`]: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      return { success: true, message: 'Rollback completed' };
    } catch (error) {
      console.error(`Rollback failed for tenant ${tenantId}:`, error);
      throw new HttpsError('internal', `Rollback failed: ${error.message}`);
    }
  }
);

interface RollbackRequest {
  tenantId: string;
  fromVersion: number;
  toVersion: number;
}
```

**3. Verify Rollback Success**

```typescript
// Verify tenant restored to v1 schema
const tenant = await firestore.doc(`tenants/${tenantId}`).get();
const schemaVersion = tenant.data()?.schemaVersion;

if (schemaVersion !== 1) {
  console.error(`Rollback verification failed: schemaVersion is ${schemaVersion}, expected 1`);
}

// Sample check jobs for v1 schema
const jobs = await firestore
  .collection(`tenants/${tenantId}/jobs`)
  .limit(10)
  .get();

for (const job of jobs.docs) {
  const data = job.data();
  if (data.primaryCurrency || data.currencies) {
    console.error(`Job ${job.id} still has v2 fields after rollback`);
  }
}
```

**4. Restore from Backup (Last Resort)**

```bash
# If rollback function fails, restore from pre-migration backup
gcloud firestore import gs://findogai-prod-backups/pre-migration-20251028/ \
  --collection-ids=tenants

# Verify restoration
gcloud firestore documents get tenants/affected-tenant-id --format=json
```

### 11.3.5 Migration Monitoring Dashboard

**Key Metrics to Track:**

```typescript
interface MigrationMetrics {
  // Overall progress
  totalTenants: number;
  tenantsCompleted: number;
  tenantsInProgress: number;
  tenantsFailed: number;
  tenantsRemaining: number;

  // Performance
  avgMigrationDuration: number; // minutes
  documentsPerSecond: number;
  errorsPerHour: number;

  // Current batch
  currentBatch: number;
  batchStartTime: Timestamp;
  batchProgress: number; // 0-100%
  estimatedCompletion: Timestamp;
}

// Real-time dashboard query
async function getMigrationMetrics(): Promise<MigrationMetrics> {
  const tenants = await firestore.collection('tenants').get();

  const metrics: MigrationMetrics = {
    totalTenants: tenants.size,
    tenantsCompleted: 0,
    tenantsInProgress: 0,
    tenantsFailed: 0,
    tenantsRemaining: 0,
    avgMigrationDuration: 0,
    documentsPerSecond: 0,
    errorsPerHour: 0,
    currentBatch: 0,
    batchStartTime: null,
    batchProgress: 0,
    estimatedCompletion: null
  };

  for (const tenant of tenants.docs) {
    const data = tenant.data();
    const migration = data.migrations?.[2];

    if (data.schemaVersion === 2 && migration?.status === 'completed') {
      metrics.tenantsCompleted++;
    } else if (migration?.status === 'in_progress') {
      metrics.tenantsInProgress++;
    } else if (migration?.status === 'failed') {
      metrics.tenantsFailed++;
    } else {
      metrics.tenantsRemaining++;
    }
  }

  return metrics;
}
```

### 11.3.6 Post-Migration Cleanup

**After Successful Migration (T+1 month):**

**1. Remove Deprecated Code**

```bash
# Remove v1 schema support from client code
# Update minimum schema version
cd packages/mobile-app/src/app/services
# Edit schema-version.service.ts
# Set MINIMUM_VERSION = 2

# Deploy updated client
npm run build
# Release to app stores
```

**2. Archive Migration Functions**

```bash
# Move migration functions to archive
cd packages/functions/src
mkdir -p archive/migrations
mv migrations/migratorV1toV2.ts archive/migrations/

# Remove from index.ts exports
# Redeploy functions
firebase deploy --only functions
```

**3. Update Documentation**

```bash
# Update schema version history
# Mark v1 as deprecated
# Document lessons learned
```

---

## Monitoring and Alerts

### Key Metrics to Monitor

1. **Firestore Usage:**
   - Document reads/writes per day
   - Storage size
   - Query latency

2. **Cloud Functions:**
   - Invocation count
   - Error rate
   - Execution time (p50, p95, p99)
   - Cold start frequency

3. **Authentication:**
   - Sign-in success rate
   - Token refresh rate
   - Failed authentication attempts

4. **Application Errors:**
   - Client-side errors (captured in logs)
   - Unhandled promise rejections
   - Security rule violations

### Alerting Policies

```bash
# Example: Alert on high error rate
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High Function Error Rate" \
  --condition-threshold-value=0.05 \
  --condition-threshold-duration=300s
```

---

For comprehensive operational procedures and incident response, refer to the [original backend architecture document](../backend-architecture.md).

---

[← Back: Testing Strategy](./testing-strategy.md) | [Back to Index](./index.md)
