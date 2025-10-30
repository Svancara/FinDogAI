[Back to Index](./index.md) | [Data Models](./data-models.md) | [API Specification](./api-specification.md)

# Business Profile Migration Guide

## Overview

This guide covers the migration of existing tenants to support the new Business Profile resource visibility feature. This feature allows business owners to hide UI elements for resource types (machines, vehicles, other expenses) they don't use.

## Database Migration

### New Fields Added to Business Profile

The Business Profile document needs three new boolean fields:

```typescript
interface BusinessProfile {
  // ... existing fields (currency, vatRate, distanceUnit)
  usesMachines: boolean;      // NEW - Default: true
  usesVehicles: boolean;      // NEW - Default: true
  usesOtherExpenses: boolean; // NEW - Default: true
}
```

### Migration Strategy

**Option 1: Firestore Cloud Function (Recommended)**

```typescript
import { onSchedule } from 'firebase-functions/v2/scheduler';
import { getFirestore } from 'firebase-admin/firestore';

export const migrateBusinessProfiles = onSchedule({
  schedule: 'once',  // Run once manually via Firebase Console
  region: 'europe-west1'
}, async () => {
  const db = getFirestore();
  const tenantsSnapshot = await db.collection('tenants').get();

  const batch = db.batch();
  let batchCount = 0;
  let updateCount = 0;

  for (const tenantDoc of tenantsSnapshot.docs) {
    const tenantId = tenantDoc.id;
    const profileRef = db.doc(`tenants/${tenantId}/businessProfile/default`);
    const profileSnapshot = await profileRef.get();

    if (profileSnapshot.exists) {
      const profile = profileSnapshot.data();

      // Only update if fields don't exist
      if (profile?.usesMachines === undefined ||
          profile?.usesVehicles === undefined ||
          profile?.usesOtherExpenses === undefined) {

        batch.update(profileRef, {
          usesMachines: true,      // Default to showing all
          usesVehicles: true,
          usesOtherExpenses: true,
          updatedAt: FieldValue.serverTimestamp()
        });

        batchCount++;
        updateCount++;

        // Firestore batch limit is 500
        if (batchCount === 500) {
          await batch.commit();
          console.log(`Committed batch of ${batchCount} updates`);
          batchCount = 0;
        }
      }
    }
  }

  // Commit remaining updates
  if (batchCount > 0) {
    await batch.commit();
  }

  console.log(`Migration complete. Updated ${updateCount} business profiles.`);
});
```

**Option 2: Firebase Admin Script (For Testing/Dev)**

```typescript
// scripts/migrate-business-profiles.ts
import * as admin from 'firebase-admin';

admin.initializeApp();
const db = admin.firestore();

async function migrateBusinessProfiles() {
  const tenantsSnapshot = await db.collection('tenants').get();
  console.log(`Found ${tenantsSnapshot.size} tenants`);

  for (const tenantDoc of tenantsSnapshot.docs) {
    const tenantId = tenantDoc.id;
    const profileRef = db.doc(`tenants/${tenantId}/businessProfile/default`);

    await profileRef.update({
      usesMachines: true,
      usesVehicles: true,
      usesOtherExpenses: true,
      updatedAt: admin.firestore.FieldValue.serverTimestamp()
    });

    console.log(`✓ Updated tenant ${tenantId}`);
  }

  console.log('Migration complete!');
}

migrateBusinessProfiles()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error('Migration failed:', error);
    process.exit(1);
  });
```

### Running the Migration

**For Cloud Function:**
1. Deploy the function: `firebase deploy --only functions:migrateBusinessProfiles`
2. Trigger manually from Firebase Console → Functions → migrateBusinessProfiles → Run
3. Monitor logs for completion

**For Admin Script:**
```bash
cd functions
npm run build
node dist/scripts/migrate-business-profiles.js
```

### Validation Query

Verify all profiles have been updated:

```typescript
// Check for profiles missing new fields
const tenantsRef = db.collection('tenants');
const tenantsSnapshot = await tenantsRef.get();

for (const tenantDoc of tenantsSnapshot.docs) {
  const profileRef = db.doc(`tenants/${tenantDoc.id}/businessProfile/default`);
  const profile = await profileRef.get();
  const data = profile.data();

  if (!data ||
      data.usesMachines === undefined ||
      data.usesVehicles === undefined ||
      data.usesOtherExpenses === undefined) {
    console.warn(`Missing fields for tenant: ${tenantDoc.id}`);
  }
}
```

## Frontend Migration

### No Breaking Changes

The frontend changes are **backward compatible**:
- UI defaults to showing all resource types when fields are missing
- Observable streams handle `undefined` gracefully
- Progressive enhancement approach

### Recommended Deployment Sequence

1. **Deploy Database Migration** (Cloud Function or Script)
2. **Deploy Backend Updates** (Security Rules, if any)
3. **Deploy Frontend Updates** (UI components)

This sequence ensures:
- No runtime errors from missing fields
- Smooth transition for active users
- Offline clients can sync when reconnected

## Security Rules Update

Add validation to security rules:

```javascript
match /tenants/{tenantId}/businessProfile/default {
  allow read: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));

  allow update: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)) &&
    get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role == 'owner' &&
    request.resource.data.tenantId == resource.data.tenantId &&
    request.resource.data.vatRate >= 0 && request.resource.data.vatRate <= 100 &&
    request.resource.data.currency in ['CZK', 'EUR', 'USD'] &&
    request.resource.data.distanceUnit in ['km', 'mi'] &&
    // NEW: Validate boolean fields
    (request.resource.data.usesMachines == null || request.resource.data.usesMachines is bool) &&
    (request.resource.data.usesVehicles == null || request.resource.data.usesVehicles is bool) &&
    (request.resource.data.usesOtherExpenses == null || request.resource.data.usesOtherExpenses is bool);
}
```

Deploy security rules:
```bash
firebase deploy --only firestore:rules
```

## Testing Checklist

### Pre-Migration Testing

- [ ] Test migration script in dev/staging environment
- [ ] Verify no data loss occurs
- [ ] Test with empty tenant collection
- [ ] Test with single tenant
- [ ] Test with 100+ tenants

### Post-Migration Testing

- [ ] Verify all tenants have new fields set to `true`
- [ ] Confirm UI shows all resource types by default
- [ ] Test toggling each flag in Business Profile UI
- [ ] Verify reactive UI updates work correctly
- [ ] Test offline sync with updated profiles
- [ ] Verify security rules allow updates only for owners

### Edge Cases

- [ ] Tenant with zero resources of each type
- [ ] Tenant with one resource of each type (auto-selection)
- [ ] Tenant with multiple resources
- [ ] Offline tenant syncing after migration
- [ ] New tenant creation includes default fields

## Rollback Plan

If issues occur, rollback is simple:

1. **Frontend Rollback:** Deploy previous frontend version
   - Old UI ignores new fields
   - No data corruption

2. **Database Rollback:** Remove fields (optional, not required)
```typescript
// Only if necessary
await profileRef.update({
  usesMachines: admin.firestore.FieldValue.delete(),
  usesVehicles: admin.firestore.FieldValue.delete(),
  usesOtherExpenses: admin.firestore.FieldValue.delete()
});
```

3. **Security Rules:** Redeploy previous version
```bash
git checkout previous-commit -- firestore.rules
firebase deploy --only firestore:rules
```

## Timeline

**Recommended Migration Schedule:**

1. **Week 1:** Deploy to dev environment, test thoroughly
2. **Week 2:** Deploy to staging environment, UAT testing
3. **Week 3:** Run migration in production (low-traffic hours)
4. **Week 4:** Monitor, address any issues

## Support

For issues during migration:
- Check logs: Firebase Console → Functions → Logs
- Verify security rules: Firebase Console → Firestore → Rules
- Test queries: Firestore Console → Query builder

---

*Next: [Developer Implementation Checklist](./business-profile-implementation-checklist.md)*
