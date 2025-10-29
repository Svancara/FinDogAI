[Back to Index](./index.md)

# Data Model

This document provides complete schema definitions for all Firestore collections, entity relationships, and required indexes.

## 4.1 Complete Schema Definitions

### 4.1.1 Tenant Collections

All tenant-scoped collections are stored under `/tenants/{tenantId}/` for multi-tenant isolation.

---

#### Collection: `/tenants/{tenantId}/members/{uid}`

Stores user roles and permissions for tenant access control, including tenant-specific identity.

```typescript
interface Member {
  tenantId: string;           // UUID, matches path parameter
  memberNumber: number;       // Sequential per tenant, used for audit trails
  displayName: string;        // User's display name (cached for performance)
  email: string;              // User's email (cached for performance)
  role: 'owner' | 'representative' | 'teamMember';
  status: 'active' | 'disabled';
  lastSeenAt: Timestamp | null;
  createdAt: Timestamp;       // serverTimestamp
  updatedAt: Timestamp;       // serverTimestamp
}
```

**Indexes:**
- Single field: `status` (ASC)
- Single field: `memberNumber` (ASC) - unique per tenant
- Composite: `status` (ASC) + `role` (ASC)

---

#### Collection: `/tenants/{tenantId}/invites/{inviteId}`

Stores pending team invitations with 7-day expiration.

```typescript
interface Invite {
  tenantId: string;           // UUID
  codeHash: string;           // SHA-256 hash of 6-digit code
  expiresAt: Timestamp;       // 7 days from creation
  createdBy: {
    uid: string;              // Firebase Auth UID for security checks
    memberNumber: number;     // Tenant-specific identifier
    displayName: string;      // Cached for display
  };
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

#### Collection: `/tenants/{tenantId}/jobs/{jobId}`

Primary collection for job records.

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
  createdBy: {
    uid: string;              // Firebase Auth UID for security checks
    memberNumber: number;     // Tenant-specific identifier
    displayName: string;      // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;              // Firebase Auth UID for security checks
    memberNumber: number;     // Tenant-specific identifier
    displayName: string;      // Cached for display
  };
}
```

**Indexes:**
- Single field: `jobNumber` (ASC) - must be unique per tenant
- Single field: `status` (ASC)
- Composite: `status` (ASC) + `updatedAt` (DESC)
- Composite: `createdBy.uid` (ASC) + `createdAt` (DESC)

---

#### Subcollection: `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}`

Stores job costs with category-specific schemas.

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
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
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

#### Subcollection: `/tenants/{tenantId}/jobs/{jobId}/advances/{advanceId}`

Stores customer advance payments.

```typescript
interface Advance {
  tenantId: string;
  ordinalNumber: number | null; // Sequential per job
  amount: number;
  date: Timestamp;
  note?: string;
  createdAt: Timestamp;
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
}
```

**Indexes:**
- Single field: `ordinalNumber` (ASC)
- Single field: `date` (DESC)

---

#### Subcollection: `/tenants/{tenantId}/jobs/{jobId}/events/{eventId}`

Stores journey tracking events.

```typescript
type EventType = 'journey_start' | 'journey_end' | 'manual_entry';

interface EventBase {
  tenantId: string;
  ordinalNumber: number | null;
  type: EventType;
  timestamp: Timestamp;
  createdAt: Timestamp;
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
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

#### Collection: `/tenants/{tenantId}/jobs_public/{jobId}`

Sanitized read-only view for team members (FR11).

```typescript
interface JobPublic {
  jobNumber: number;
  title: string;
  status: 'active' | 'completed' | 'archived';
}
```

**Purpose**: Team members can see active jobs without access to financial details.

**Maintenance**: Maintained by Cloud Function triggers on job create/update.

---

#### Collection: `/tenants/{tenantId}/vehicles/{vehicleId}`

Stores vehicle resource definitions.

```typescript
interface Vehicle {
  tenantId: string;
  vehicleNumber: number | null; // Sequential
  name: string;                  // e.g., "Transporter VW"
  distanceUnit: 'km' | 'miles';  // Copied from business profile at creation
  ratePerDistanceUnit: number;   // e.g., 8.50 CZK/km
  createdAt: Timestamp;
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
}
```

**Indexes:**
- Single field: `vehicleNumber` (ASC) - unique per tenant
- Single field: `name` (ASC)

---

#### Collection: `/tenants/{tenantId}/machines/{machineId}`

Stores machine/equipment resource definitions.

```typescript
interface Machine {
  tenantId: string;
  machineNumber: number | null;
  name: string;                 // e.g., "Concrete Mixer"
  hourlyRate: number;
  createdAt: Timestamp;
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
}
```

**Indexes:**
- Single field: `machineNumber` (ASC) - unique per tenant

---

#### Collection: `/tenants/{tenantId}/teamMembers/{teamMemberId}`

Stores team member resource definitions.

```typescript
interface TeamMember {
  tenantId: string;
  teamMemberNumber: number | null;
  name: string;
  hourlyRate: number;
  authUserId: string | null;    // Firebase Auth UID for login mapping
  createdAt: Timestamp;
  createdBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  updatedAt: Timestamp;
  updatedBy: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
}
```

**Indexes:**
- Single field: `teamMemberNumber` (ASC) - unique per tenant
- Single field: `authUserId` (ASC) - for auth lookup

---

#### Document: `/tenants/{tenantId}/businessProfile`

Single document storing tenant business settings.

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

#### Document: `/tenants/{tenantId}/personProfile`

Single document storing user personal preferences.

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

#### Collection: `/tenants/{tenantId}/audit_logs/{logId}`

Stores audit trail for compliance (1-year retention).

```typescript
type AuditOperation = 'CREATE' | 'UPDATE' | 'DELETE';

interface AuditLog {
  operation: AuditOperation;
  collection: string;           // e.g., "jobs", "costs", "vehicles"
  documentId: string;
  tenantId: string;
  timestamp: Timestamp;
  author: {
    uid: string;                // Firebase Auth UID for security checks
    memberNumber: number;       // Tenant-specific identifier
    displayName: string;        // Cached for display
  };
  before?: any;                 // Old data for UPDATE operations
  after?: any;                  // New data for CREATE/UPDATE operations
  ttl: Timestamp;               // Auto-delete after this date (1 year)
}
```

**Indexes:**
- Single field: `ttl` (ASC) - for cleanup queries
- Composite: `timestamp` (DESC) + `collection` (ASC)
- Composite: `author.memberNumber` (ASC) + `timestamp` (DESC)

---

### 4.1.2 Tenant Root Document

#### Document: `/tenants/{tenantId}`

Root tenant document storing tenant-level metadata and schema versioning information.

```typescript
interface Tenant {
  tenantId: string;           // UUID, matches path parameter
  schemaVersion: number;      // Current schema version (starts at 1)
  createdAt: Timestamp;       // serverTimestamp
  updatedAt: Timestamp;       // serverTimestamp

  // Migration tracking
  migrations?: {
    [version: number]: {
      appliedAt: Timestamp;
      appliedBy: string;      // UID of admin who triggered migration
      status: 'completed' | 'failed' | 'in_progress';
      error?: string;
      documentsProcessed?: number;
    }
  };
}
```

**Purpose**: Track schema version per tenant to enable zero-downtime migrations and maintain backwards compatibility.

**Schema Versions:**
- `v1`: Initial schema with compound identity model (current)
- `v2+`: Future schema versions documented as they are released

**Access**: Read by all authenticated members, write by Cloud Functions only.

---

### 4.1.3 Global Collections

#### Collection: `/user_tenants/{uid}/memberships/{tenantId}`

Enables user-to-tenant resolution during auth flow.

```typescript
interface UserTenantMapping {
  tenantId: string;
  role?: 'owner' | 'representative' | 'teamMember'; // Optional snapshot
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Purpose**: Maps Firebase Auth UIDs to tenant memberships for initial tenant context loading.

**Access**: Read-only for authenticated users (their own UID only).

---

#### Collection: `/sequences/{tenantId}/counters/{counterType}`

Server-side sequential number generators.

```typescript
interface SequenceCounter {
  current: number;  // Last allocated number
}
```

**Counter types:**
- `jobNumber`
- `vehicleNumber`
- `machineNumber`
- `teamMemberNumber`
- `memberNumber` (sequential member identification per tenant)
- `job_{jobId}_ordinalNumber` (per-job costs/advances/events)

**Access**: Cloud Functions only (Security Rules deny client access).

---

## 4.2 Entity Relationships

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

### Key Design Decisions

1. **Subcollections for Job-Related Data**: Costs, advances, and events are subcollections under `/tenants/{tenantId}/jobs/{jobId}/` to optimize offline queries (fetch job + all costs in single query).

2. **Resource Snapshots**: Costs and events embed resource data (vehicle name, hourly rate) at creation time. This prevents historical data corruption if resource definitions change later.

3. **Soft Deletes**: Jobs use `status: archived` instead of document deletion to preserve audit trail and cost references.

4. **Nullable Sequential Numbers**: Documents created offline have `null` sequential numbers until server-side assignment completes.

5. **Single Resource Auto-Selection**: Client applications should query resource collections (vehicles, team members, machines) and automatically select the resource if exactly one exists. This improves UX by eliminating unnecessary selection steps. Backend remains unchanged - this is purely a client-side optimization based on collection count queries.

---

## 4.3 Required Indexes

### 4.3.1 Composite Indexes (firestore.indexes.json)

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

### 4.3.2 Index Deployment

Deploy indexes to your Firebase project:

```bash
# Deploy indexes
firebase deploy --only firestore:indexes

# Indexes build asynchronously; monitor progress:
# Firebase Console > Firestore > Indexes tab
```

**Note**: Index creation can take several minutes to hours depending on the size of your dataset. Newly deployed indexes will be in "Building" status until complete.

### 4.3.3 Collection Access Patterns

Understanding access patterns helps optimize queries and indexes:

| Collection | Read Pattern | Write Pattern | Offline Support | Notes |
|------------|--------------|---------------|-----------------|-------|
| tenants/{tid}/members | Startup, auth check | Admin only | No (requires online) | |
| tenants/{tid}/jobs | List view, filters | CRUD by owner/rep | Yes (full offline) | |
| tenants/{tid}/jobs/{jid}/costs | Job detail view | High frequency | Yes (full offline) | |
| tenants/{tid}/vehicles | Resource selector | Low frequency | Yes (cached) | Count query for auto-selection (FR25) |
| tenants/{tid}/machines | Resource selector | Low frequency | Yes (cached) | Count query for auto-selection (FR25) |
| tenants/{tid}/teamMembers | Resource selector | Low frequency | Yes (cached) | Count query for auto-selection (FR25) |
| tenants/{tid}/audit_logs | Admin export only | Auto (triggers) | No (server-side only) | |
| user_tenants/{uid} | Auth flow only | Cloud Function | No (custom claims) | |
| sequences/{tid} | Never (client) | Cloud Function only | No (server transaction) | |

**Note on Single Resource Auto-Selection (FR25):** Client applications should perform simple count queries on resource collections. If `count === 1`, the UI automatically selects that resource and displays it as read-only text. This requires no backend changes - it's a client-side UX optimization.

---

## 4.4 Multi-Tenant Identity Model

### 4.4.1 Overview

The FinDogAI system uses a compound identity model to properly handle multi-tenant scenarios where a single Firebase Auth user (identified by UID) can be a member of multiple tenants, each with their own tenant-specific identity.

### 4.4.2 Identity Components

**Firebase Auth UID**
- Global identifier across the entire system
- Used for authentication and security rule authorization checks
- Stored in `request.auth.uid` in security rules
- Never changes, persists across tenant memberships

**Member Number**
- Sequential, tenant-specific identifier (1, 2, 3, ...)
- Allocated when user joins a tenant
- Human-readable identifier for audit trails and display
- Unique within a tenant, not globally unique

**Display Name & Email**
- Cached from user profile at time of action
- Stored with each audit record for performance
- Avoids additional lookups when displaying history

### 4.4.3 Identity Object Structure

All documents that track who created or updated them use this compound identity object:

```typescript
interface UserIdentity {
  uid: string;              // Firebase Auth UID for security checks
  memberNumber: number;     // Tenant-specific identifier for display
  displayName: string;      // Cached display name
}

// Used in createdBy and updatedBy fields
interface AuditableDocument {
  createdAt: Timestamp;
  createdBy: UserIdentity;
  updatedAt: Timestamp;
  updatedBy: UserIdentity;
  // ... other fields
}
```

### 4.4.4 Why This Design?

**Problem with UID-only approach:**
```typescript
// ❌ WRONG: Using Firebase UID directly
interface Job {
  createdBy: string;  // "abc123xyz" - not tenant-specific
  updatedBy: string;  // "abc123xyz" - same UID across tenants
}
```

Issues:
1. No tenant-specific identity for display
2. Requires lookup to member document for every display
3. UID is not human-readable in audit logs
4. Cannot distinguish same user across different tenants

**Solution with compound identity:**
```typescript
// ✅ CORRECT: Using compound identity object
interface Job {
  createdBy: {
    uid: "abc123xyz",       // For security rules
    memberNumber: 1,        // Tenant-specific, human-readable
    displayName: "John Doe" // Cached for display
  },
  updatedBy: {
    uid: "abc123xyz",
    memberNumber: 1,
    displayName: "John Doe"
  }
}
```

Benefits:
1. Security rules use `createdBy.uid` for authorization
2. UI displays `createdBy.displayName` without lookups
3. Audit logs show human-readable `Member #1 (John Doe)`
4. Tenant-specific identity preserved in historical records

### 4.4.5 Member Creation Flow

When a user joins a tenant (via invite, registration, or owner addition):

1. **Allocate memberNumber** (Cloud Function):
   ```typescript
   const memberNumber = await allocateSequence(tenantId, 'memberNumber');
   ```

2. **Create member document**:
   ```typescript
   await db.collection(`tenants/${tenantId}/members`).doc(uid).set({
     tenantId,
     memberNumber,        // Sequential: 1, 2, 3, ...
     displayName,         // From user profile
     email,               // From user profile
     role,                // owner | representative | teamMember
     status: 'active',
     // ... timestamps
   });
   ```

3. **Client caches identity** (for creating documents):
   ```typescript
   const currentUserIdentity = {
     uid: auth.currentUser.uid,
     memberNumber: memberDoc.memberNumber,
     displayName: memberDoc.displayName
   };
   ```

4. **Use identity in document creation**:
   ```typescript
   await db.collection(`tenants/${tenantId}/jobs`).add({
     // ... job fields
     createdBy: currentUserIdentity,
     updatedBy: currentUserIdentity,
     createdAt: serverTimestamp(),
     updatedAt: serverTimestamp()
   });
   ```

### 4.4.6 Security Rule Validation

Security rules validate the compound identity object:

```javascript
function auditMetadataValid() {
  return request.resource.data.createdAt == request.time
    && request.resource.data.createdBy.uid == request.auth.uid  // Verify UID
    && request.resource.data.updatedAt == request.time
    && request.resource.data.updatedBy.uid == request.auth.uid; // Verify UID
}

function updateMetadataValid() {
  return request.resource.data.updatedAt == request.time
    && request.resource.data.updatedBy.uid == request.auth.uid  // Verify UID
    && request.resource.data.createdAt == resource.data.createdAt
    && request.resource.data.createdBy.uid == resource.data.createdBy.uid  // Preserve UID
    && request.resource.data.createdBy.memberNumber == resource.data.createdBy.memberNumber  // Preserve memberNumber
    && request.resource.data.createdBy.displayName == resource.data.createdBy.displayName;   // Preserve displayName
}
```

**Key Points:**
- Authorization checks use `.uid` field (matches `request.auth.uid`)
- Client must provide full identity object (including memberNumber and displayName)
- Security rules verify UID matches authenticated user
- On updates, original `createdBy` object must remain unchanged

### 4.4.7 Display and Audit Trail Usage

**In UI components:**
```typescript
// Display job creator
<span>Created by: {job.createdBy.displayName} (Member #{job.createdBy.memberNumber})</span>

// Display in audit log
<li>Job #{job.jobNumber} created by {job.createdBy.displayName} at {job.createdAt}</li>
```

**In audit logs:**
```typescript
interface AuditLog {
  author: {
    uid: string;           // For filtering by user
    memberNumber: number;  // For display: "Member #5"
    displayName: string;   // For display: "Jane Smith"
  }
  // ... other fields
}

// Query audit logs by member
const logs = await db.collection(`tenants/${tenantId}/audit_logs`)
  .where('author.memberNumber', '==', 5)
  .orderBy('timestamp', 'desc')
  .get();
```

### 4.4.8 Migration Notes

If upgrading from UID-only approach:

1. **Add memberNumber to existing members**:
   - Run migration Cloud Function to allocate sequential memberNumbers
   - Update all existing member documents

2. **Transform audit fields**:
   - Option A: Leave historical data as-is (UID strings)
   - Option B: Migrate to compound objects (requires backfilling memberNumber/displayName)

3. **Update client code**:
   - Cache current user identity on login
   - Use compound identity object when creating/updating documents
   - Update UI to display from cached identity

---

## 4.5 Schema Versioning

### 4.5.1 Overview

FinDogAI uses tenant-level schema versioning to enable safe evolution of the data model while maintaining backwards compatibility. Each tenant tracks its current schema version and migration history, allowing for zero-downtime updates and gradual rollouts.

### 4.5.2 Versioning Strategy

**Key Principles:**
1. **Tenant-Level Versioning**: Each tenant has its own schema version, enabling gradual migrations
2. **Backwards Compatibility**: Clients must handle multiple schema versions gracefully
3. **Zero-Downtime**: Migrations run in background without service interruption
4. **Explicit Migrations**: Schema changes require explicit migration functions
5. **Rollback Support**: Failed migrations can be rolled back per tenant

### 4.5.3 Schema Version History

#### Version 1 (Current - Initial Release)
**Features:**
- Compound identity model (UID + memberNumber + displayName)
- Tenant-scoped collections with multi-tenant isolation
- Sequential numbering for jobs, resources, and ordinals
- Audit logging with 1-year retention
- Offline-first with nullable sequential numbers

**Collections Structure:**
```
/tenants/{tenantId}/
  - members/
  - jobs/
    - costs/
    - advances/
    - events/
  - vehicles/
  - machines/
  - teamMembers/
  - audit_logs/
  - businessProfile (doc)
  - personProfile (doc)
```

#### Version 2+ (Future)
Future schema versions will be documented here as they are released. Each version entry should include:
- Version number and release date
- Breaking changes (if any)
- New fields or collections
- Migration requirements
- Deprecation notices

### 4.5.4 Version Compatibility Matrix

| Client Version | Min Schema Version | Max Schema Version | Notes |
|----------------|-------------------|-------------------|-------|
| 1.0.x | 1 | 1 | Initial release |
| 1.1.x | 1 | 2 | Supports v1 and v2 schemas |
| 2.0.x | 2 | 3 | Drops v1 support, adds v3 |

### 4.5.5 Client Version Checking

**On App Startup:**

```typescript
// packages/mobile-app/src/app/services/schema-version.service.ts

export class SchemaVersionService {
  private readonly SUPPORTED_VERSIONS = [1]; // Update as new versions are supported
  private readonly MINIMUM_VERSION = 1;
  private readonly MAXIMUM_VERSION = 1;

  async checkCompatibility(tenantId: string): Promise<CompatibilityCheck> {
    const tenantDoc = await this.firestore
      .collection('tenants')
      .doc(tenantId)
      .get();

    const schemaVersion = tenantDoc.data()?.schemaVersion || 1;

    // Check if client supports this schema version
    if (schemaVersion < this.MINIMUM_VERSION) {
      return {
        compatible: false,
        action: 'UPGRADE_REQUIRED',
        message: 'Your data schema is outdated. Please contact support.',
        currentVersion: schemaVersion,
        supportedVersions: this.SUPPORTED_VERSIONS
      };
    }

    if (schemaVersion > this.MAXIMUM_VERSION) {
      return {
        compatible: false,
        action: 'UPDATE_APP',
        message: 'Please update to the latest version of FinDogAI.',
        currentVersion: schemaVersion,
        supportedVersions: this.SUPPORTED_VERSIONS
      };
    }

    return {
      compatible: true,
      action: 'PROCEED',
      message: 'Schema version compatible',
      currentVersion: schemaVersion,
      supportedVersions: this.SUPPORTED_VERSIONS
    };
  }
}

interface CompatibilityCheck {
  compatible: boolean;
  action: 'PROCEED' | 'UPDATE_APP' | 'UPGRADE_REQUIRED' | 'MIGRATION_IN_PROGRESS';
  message: string;
  currentVersion: number;
  supportedVersions: number[];
}
```

### 4.5.6 Handling Mixed Versions During Migration

**Strategy:**
1. **Read Path**: Clients must handle both old and new schemas
2. **Write Path**: Clients write in their supported schema version
3. **Migration Function**: Transforms data from old to new schema
4. **Gradual Rollout**: Migrate tenants incrementally (batches of 10-50)

**Example: Reading with Version Handling**

```typescript
// Handle both v1 (single currency) and v2 (multi-currency) schemas
function readJob(jobDoc: DocumentSnapshot): Job {
  const data = jobDoc.data();
  const schemaVersion = data.schemaVersion || 1;

  if (schemaVersion === 1) {
    return {
      ...data,
      currency: data.currency || 'CZK', // v1: single currency
      currencies: undefined // v2 field not present
    };
  } else if (schemaVersion === 2) {
    return {
      ...data,
      currency: data.primaryCurrency, // v2: primary currency
      currencies: data.currencies // v2: multi-currency support
    };
  }

  throw new Error(`Unsupported schema version: ${schemaVersion}`);
}
```

### 4.5.7 Migration Status Tracking

**During Migration:**

```typescript
// Tenant document during migration
{
  tenantId: "550e8400-e29b-41d4-a716-446655440000",
  schemaVersion: 1, // Still on v1 during migration
  migrations: {
    2: {
      appliedAt: Timestamp(2025-11-01 10:30:00),
      appliedBy: "admin-uid-12345",
      status: "in_progress",
      documentsProcessed: 1250,
      error: null
    }
  }
}
```

**After Successful Migration:**

```typescript
{
  tenantId: "550e8400-e29b-41d4-a716-446655440000",
  schemaVersion: 2, // Updated to v2
  migrations: {
    2: {
      appliedAt: Timestamp(2025-11-01 10:30:00),
      appliedBy: "admin-uid-12345",
      status: "completed",
      documentsProcessed: 2500,
      error: null
    }
  }
}
```

**After Failed Migration:**

```typescript
{
  tenantId: "550e8400-e29b-41d4-a716-446655440000",
  schemaVersion: 1, // Remains on v1
  migrations: {
    2: {
      appliedAt: Timestamp(2025-11-01 10:30:00),
      appliedBy: "admin-uid-12345",
      status: "failed",
      documentsProcessed: 750,
      error: "Transaction timeout at document batch 15"
    }
  }
}
```

### 4.5.8 Version Deprecation Policy

**Timeline:**
1. **Release New Version**: v2 released, v1 still fully supported
2. **Deprecation Notice (T+3 months)**: v1 marked deprecated, migration recommended
3. **Migration Window (T+6 months)**: All tenants should migrate to v2
4. **End of Support (T+12 months)**: v1 client support dropped, forced migration for remaining tenants

**Communication:**
- Email notifications to tenant owners at each milestone
- In-app banners for deprecated versions
- Admin dashboard showing migration status

---

[← Back: Firebase Services](./firebase-services.md) | [Back to Index](./index.md) | [Next: Cloud Functions →](./cloud-functions.md)
