[Back to Index](./index.md)

# Security Rules

This document provides complete Firestore and Cloud Storage security rules with detailed authorization patterns.

## Overview

Security Rules enforce:
- Authentication requirements
- Multi-tenant data isolation
- Role-based access control (RBAC)
- Audit metadata validation
- Server-only operations

**Deployment:**
```bash
# Deploy Firestore rules
firebase deploy --only firestore:rules

# Deploy Storage rules
firebase deploy --only storage
```

---

## 3.3.1 Firestore Security Rules

Complete security rules for all Firestore collections.

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
        && request.resource.data.createdBy.uid == request.auth.uid
        && request.resource.data.updatedAt == request.time
        && request.resource.data.updatedBy.uid == request.auth.uid;
    }

    function updateMetadataValid() {
      return request.resource.data.updatedAt == request.time
        && request.resource.data.updatedBy.uid == request.auth.uid
        && request.resource.data.createdAt == resource.data.createdAt
        && request.resource.data.createdBy.uid == resource.data.createdBy.uid
        && request.resource.data.createdBy.memberNumber == resource.data.createdBy.memberNumber
        && request.resource.data.createdBy.displayName == resource.data.createdBy.displayName;
    }

    // Schema version checking functions
    // These functions ensure data integrity during schema migrations by preventing
    // writes when migrations are in progress and enforcing version compatibility.

    function getSchemaVersion(tenantId) {
      // Retrieves the current schema version for a tenant
      // Returns the schemaVersion field from the tenant document
      // If the field doesn't exist, defaults to version 1
      let tenant = get(/databases/$(database)/documents/tenants/$(tenantId));
      return tenant.data.keys().hasAny(['schemaVersion'])
        ? tenant.data.schemaVersion
        : 1;
    }

    function isMigrationInProgress(tenantId) {
      // Checks if a schema migration is currently running for a tenant
      // Returns true if the migrations.inProgress field is set to true
      // During migrations, most write operations should be blocked to prevent
      // data corruption from schema inconsistencies
      let tenant = get(/databases/$(database)/documents/tenants/$(tenantId));
      return tenant.data.keys().hasAny(['migrations'])
        && tenant.data.migrations.keys().hasAny(['inProgress'])
        && tenant.data.migrations.inProgress == true;
    }

    function isSchemaCompatible(tenantId) {
      // Checks if the tenant's schema version is compatible with client operations
      // Currently allows schema versions 1 and 2
      // Extend this list as new schema versions are released
      // Incompatible versions should be handled by forcing client updates
      let version = getSchemaVersion(tenantId);
      return version >= 1 && version <= 2;
    }

    function canWriteToTenant(tenantId) {
      // Master check for write operations to tenant data
      // Blocks writes if:
      // - A migration is in progress (prevents data corruption)
      // - Schema version is incompatible (prevents client/server mismatch)
      // This should be used in all write rules for tenant collections
      return !isMigrationInProgress(tenantId) && isSchemaCompatible(tenantId);
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
      // Tenant root document
      // Contains metadata including schemaVersion and migrations status
      allow read: if isActiveMember(tenantId);
      allow update: if isOwner(tenantId)
        // Prevent clients from modifying schema-related fields
        // Only Cloud Functions can update schemaVersion and migrations
        && (!request.resource.data.diff(resource.data).affectedKeys().hasAny(['schemaVersion']))
        && (!request.resource.data.diff(resource.data).affectedKeys().hasAny(['migrations']));
      allow create, delete: if false; // Tenants created via Cloud Functions only

      // Members collection
      match /members/{uid} {
        allow read: if isActiveMember(tenantId);
        allow create: if false; // Only via invite consumption (Cloud Function)
        allow update: if isOwner(tenantId)
          && canWriteToTenant(tenantId); // Check migration status and schema version
        allow delete: if isOwner(tenantId)
          && uid != request.auth.uid // Can't delete self
          && canWriteToTenant(tenantId);
      }

      // Invites collection
      match /invites/{inviteId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid()
          && canWriteToTenant(tenantId);
        allow update: if false; // Immutable after creation
        allow delete: if isOwner(tenantId)
          && canWriteToTenant(tenantId);
      }

      // Business profile (single document)
      match /businessProfile {
        allow read: if isActiveMember(tenantId);
        allow write: if isOwner(tenantId)
          && documentHasTenantId(tenantId)
          && canWriteToTenant(tenantId);
      }

      // Person profile (single document)
      match /personProfile {
        allow read: if isActiveMember(tenantId);
        allow write: if isActiveMember(tenantId)
          && canWriteToTenant(tenantId);
      }

      // Jobs collection
      match /jobs/{jobId} {
        allow read: if isOwnerOrRepresentative(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid()
          && canWriteToTenant(tenantId);
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid()
          && canWriteToTenant(tenantId);
        allow delete: if false; // Use soft delete (status: archived)

        // Costs subcollection
        match /costs/{costId} {
          allow read: if isActiveMember(tenantId); // Team members can read
          allow create: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid()
            && canWriteToTenant(tenantId);
          allow update: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid()
            && canWriteToTenant(tenantId);
          allow delete: if isOwnerOrRepresentative(tenantId)
            && canWriteToTenant(tenantId);
        }

        // Advances subcollection
        match /advances/{advanceId} {
          allow read: if isOwnerOrRepresentative(tenantId);
          allow create: if isOwnerOrRepresentative(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid()
            && canWriteToTenant(tenantId);
          allow update: if isOwnerOrRepresentative(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid()
            && canWriteToTenant(tenantId);
          allow delete: if isOwnerOrRepresentative(tenantId)
            && canWriteToTenant(tenantId);
        }

        // Events subcollection
        match /events/{eventId} {
          allow read: if isActiveMember(tenantId);
          allow create: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && auditMetadataValid()
            && canWriteToTenant(tenantId);
          allow update: if isActiveMember(tenantId)
            && documentHasTenantId(tenantId)
            && updateMetadataValid()
            && canWriteToTenant(tenantId);
          allow delete: if isOwnerOrRepresentative(tenantId)
            && canWriteToTenant(tenantId);
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
          && auditMetadataValid()
          && canWriteToTenant(tenantId);
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid()
          && canWriteToTenant(tenantId);
        allow delete: if isOwnerOrRepresentative(tenantId)
          && canWriteToTenant(tenantId);
      }

      // Machines collection
      match /machines/{machineId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid()
          && canWriteToTenant(tenantId);
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid()
          && canWriteToTenant(tenantId);
        allow delete: if isOwnerOrRepresentative(tenantId)
          && canWriteToTenant(tenantId);
      }

      // Team members collection
      match /teamMembers/{teamMemberId} {
        allow read: if isActiveMember(tenantId);
        allow create: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && auditMetadataValid()
          && canWriteToTenant(tenantId);
        allow update: if isOwnerOrRepresentative(tenantId)
          && documentHasTenantId(tenantId)
          && updateMetadataValid()
          && canWriteToTenant(tenantId);
        allow delete: if isOwnerOrRepresentative(tenantId)
          && canWriteToTenant(tenantId);
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

### Helper Functions Explained

#### Schema Version Checking

**`getSchemaVersion(tenantId)`**
- Retrieves the current schema version for a tenant from the tenant document
- Returns the value of the `schemaVersion` field, defaulting to 1 if not present
- Used to check compatibility between client code and database schema

**`isMigrationInProgress(tenantId)`**
- Checks if a schema migration is currently running for the tenant
- Returns `true` if `migrations.inProgress` field is set to `true`
- During migrations, write operations are blocked to prevent data corruption
- Cloud Functions set this flag before starting migrations and clear it when complete

**`isSchemaCompatible(tenantId)`**
- Validates that the tenant's schema version is compatible with current client operations
- Currently allows schema versions 1 and 2
- Should be updated whenever new schema versions are released
- Clients with incompatible versions should be forced to update

**`canWriteToTenant(tenantId)`**
- Master authorization check for all write operations to tenant data
- Blocks writes if:
  - A migration is in progress (prevents data corruption from schema inconsistencies)
  - Schema version is incompatible (prevents client/server version mismatch)
- This check is integrated into all write rules for tenant collections
- Read operations are not blocked during migrations (data remains accessible)

**Why Schema Version Checking Matters:**
- **Data Integrity**: Prevents writes during migrations when data structure is changing
- **Version Compatibility**: Ensures clients don't write data in incompatible formats
- **Smooth Migrations**: Allows migrations to complete without client interference
- **User Experience**: Read access remains available during migrations, minimizing downtime

**Migration Workflow:**
1. Cloud Function sets `migrations.inProgress = true` on tenant document
2. All client write operations are blocked by `canWriteToTenant()` check
3. Migration runs and updates data schema
4. Cloud Function updates `schemaVersion` to new version
5. Cloud Function sets `migrations.inProgress = false`
6. Client write operations resume with new schema

#### Authentication Checks

**`isAuthenticated()`**
- Verifies user has valid Firebase Auth token
- Required for all tenant data access

**`isMember(tenantId)`**
- Checks if user has membership document in tenant
- Does NOT verify status (could be disabled)

**`isActiveMember(tenantId)`**
- Verifies user is an active member of the tenant
- Most common check for read/write access

#### Role Checks

**`getMemberRole(tenantId)`**
- Retrieves user's role from member document
- Returns: 'owner' | 'representative' | 'teamMember'

**`isOwner(tenantId)`**
- Checks if user has owner role
- Required for sensitive operations (audit logs, member management)

**`isOwnerOrRepresentative(tenantId)`**
- Checks if user has owner OR representative role
- Required for most write operations (jobs, vehicles, etc.)

#### Data Validation

**`documentHasTenantId(tenantId)`**
- Ensures document being created/updated has correct tenantId field
- Prevents cross-tenant data pollution

**`auditMetadataValid()`**
- Validates audit fields on document creation:
  - `createdAt` = server timestamp
  - `createdBy.uid` = current user UID
  - `updatedAt` = server timestamp
  - `updatedBy.uid` = current user UID
- Note: Client must provide full identity object including memberNumber and displayName

**`updateMetadataValid()`**
- Validates audit fields on document update:
  - `updatedAt` = server timestamp
  - `updatedBy.uid` = current user UID
  - `createdAt` = unchanged
  - `createdBy` = unchanged (all fields: uid, memberNumber, displayName)

---

## 3.3.2 Storage Security Rules

Complete security rules for Cloud Storage.

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

### Storage Rules Explained

**Data Exports (`/exports/{tenantId}/*`)**
- Read: Owner role only (verified via Firestore lookup)
- Write: Cloud Functions only (client writes denied)
- Use case: GDPR data export downloads

**PDF Reports (`/reports/{tenantId}/*`)** - Phase 2
- Read: Owner role only
- Write: Cloud Functions only
- Use case: Generated job reports

---

## Role-Based Access Control (RBAC)

### Role Permissions Matrix

| Collection | Owner | Representative | Team Member |
|------------|-------|----------------|-------------|
| tenant (root) | Read/Update* | Read | Read |
| members | Read/Update/Delete | Read | Read |
| invites | Read/Create/Delete | Read/Create | Read |
| businessProfile | Read/Write | Read | Read |
| personProfile | Read/Write | Read/Write | Read/Write |
| jobs | Read/Write | Read/Write | Read (jobs_public) |
| costs | Read/Write | Read/Write | Read/Write |
| advances | Read/Write | Read/Write | None |
| events | Read/Write | Read/Write | Read/Write |
| vehicles | Read/Write | Read/Write | Read |
| machines | Read/Write | Read/Write | Read |
| teamMembers | Read/Write | Read/Write | Read |
| audit_logs | Read | None | None |

*Owners can update the tenant document, but cannot modify `schemaVersion` or `migrations` fields (Cloud Functions only).

### Special Cases

**Tenant Document Schema Fields:**
- The tenant root document contains `schemaVersion` and `migrations` fields
- These fields can only be updated by Cloud Functions (server-side)
- Client updates to the tenant document are allowed, but cannot modify these fields
- This ensures schema version integrity and prevents clients from bypassing migration checks

**Team Member Cost Creation:**
- Team members can create costs (high frequency operation)
- Only owners/representatives can delete costs
- This enables field workers to log costs hands-free

**Soft Deletes:**
- Jobs cannot be hard deleted (rule: `allow delete: if false`)
- Use `status: 'archived'` for soft delete
- Preserves audit trail and cost references

**Server-Only Collections:**
- `audit_logs`: Write access denied for all clients
- `sequences`: Read/write access denied for all clients
- `user_tenants`: Write access denied for all clients

---

## Testing Security Rules

### Local Testing with Emulator

```bash
# Start Firestore emulator
firebase emulators:start --only firestore

# Run rules tests
npm test -- packages/functions/test/firestore.rules.test.ts
```

### Rules Unit Tests

See [Testing Strategy - Security Rules Testing](./testing-strategy.md#103-security-rules-testing) for comprehensive test examples.

Example test:

```typescript
import { assertFails, assertSucceeds } from '@firebase/rules-unit-testing';

test('Owner can read audit logs', async () => {
  const db = getAuthorizedFirestore('owner-uid', { tenant_id: 'test-tenant' });
  const doc = db.collection('tenants/test-tenant/audit_logs').doc('log1');
  await assertSucceeds(doc.get());
});

test('Representative cannot read audit logs', async () => {
  const db = getAuthorizedFirestore('rep-uid', { tenant_id: 'test-tenant' });
  const doc = db.collection('tenants/test-tenant/audit_logs').doc('log1');
  await assertFails(doc.get());
});
```

---

## Deployment

### Deploy to Development

```bash
firebase deploy --only firestore:rules,storage --project findogai-dev
```

### Deploy to Production

```bash
# Deploy rules
firebase deploy --only firestore:rules,storage --project findogai-prod

# Verify in Firebase Console
# - Firestore > Rules tab
# - Storage > Rules tab
```

### Rules Validation

Firebase automatically validates rules syntax before deployment. Common errors:

**Syntax Error:**
```
Error: Invalid rule syntax
```
Solution: Check for missing semicolons, unclosed braces, or invalid function names.

**Recursion Limit:**
```
Error: Function call depth exceeded
```
Solution: Reduce nested helper function calls (max depth ~10).

**Document Read Limits:**
```
Error: Too many document reads in security rules
```
Solution: Reduce `get()` calls (max ~20 per request). Cache membership checks.

---

## Performance Considerations

### Document Reads in Rules

Each `get()` or `exists()` call counts as a document read:

```javascript
// Bad: 2 document reads per request
function isActiveMember(tenantId) {
  return exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid))  // Read 1
    && get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.status == 'active';  // Read 2
}

// Better: 1 document read per request
function isActiveMember(tenantId) {
  let member = get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));  // Read 1
  return member.data.status == 'active';
}
```

**Schema Version Check Impact:**
- The `canWriteToTenant()` function adds one additional document read (to fetch the tenant document)
- This read is only performed for write operations, not reads
- The tenant document is relatively small and frequently cached by Firestore
- This overhead is acceptable given the protection against data corruption during migrations
- Read operations do not incur this cost, ensuring fast data access even during migrations

### Custom Claims for Optimization

Consider adding frequently checked data to custom claims to avoid document reads:

```javascript
// Instead of checking Firestore every time
function getTenantId() {
  return request.auth.token.tenant_id; // No document read
}
```

See [Security Considerations](./security-considerations.md) for authentication and authorization best practices.

---

[← Back: Cloud Functions](./cloud-functions.md) | [Back to Index](./index.md) | [Next: API Design →](./api-design.md)
