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
