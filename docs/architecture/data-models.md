[Back to Index](./index.md) | [Previous: Tech Stack](./tech-stack.md) | [Next: API Specification](./api-specification.md)

# Data Models

These core data models are shared between frontend and backend through the `/packages/shared-types` package, ensuring type consistency across the fullstack application.

## Tenant Model

**Purpose:** Multi-tenant organization with schema versioning and migration tracking

**Key Attributes:**
- `tenantId`: string (UUID) - Unique tenant identifier
- `schemaVersion`: number - Current data schema version
- `migrations`: Record<string, MigrationStatus> - Applied migration history
- `createdAt`: Timestamp - Tenant creation timestamp
- `updatedAt`: Timestamp - Last modification timestamp

**TypeScript Interface:**
```typescript
interface Tenant {
  tenantId: string;
  schemaVersion: number;
  migrations: {
    [version: string]: {
      appliedAt: Timestamp;
      status: 'completed' | 'failed';
    }
  };
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Relationships:**
- Parent to all tenant-scoped collections (members, jobs, resources)
- One-to-many with Member entities
- One-to-many with Job entities

## Member Model

**Purpose:** User roles and permissions within a tenant, including tenant-specific identity for audit trails

**Key Attributes:**
- `uid`: string - Firebase Auth UID (primary identifier)
- `memberNumber`: number - Sequential identifier per tenant (for display)
- `displayName`: string - Cached user display name
- `email`: string - Cached user email
- `role`: MemberRole - Access level within tenant
- `status`: MemberStatus - Active/disabled state
- `lastSeenAt`: Timestamp | null - Last activity timestamp
- `deletedAt`: Timestamp | null - Soft delete timestamp

**TypeScript Interface:**
```typescript
type MemberRole = 'owner' | 'representative' | 'teamMember';
type MemberStatus = 'active' | 'disabled';

interface Member {
  uid: string;                 // Firebase Auth UID - document ID
  tenantId: string;
  memberNumber: number;
  displayName: string;
  email: string;
  role: MemberRole;
  status: MemberStatus;
  lastSeenAt: Timestamp | null;
  deletedAt: Timestamp | null; // Soft delete
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Relationships:**
- Belongs to one Tenant
- UID referenced in audit metadata across all entities
- One-to-many with created/updated Jobs and Costs

## Job Model

**Purpose:** Core business entity representing field service jobs with financial tracking

**Key Attributes:**
- `jobNumber`: number | null - Sequential number (null for offline creation)
- `title`: string - Job identifier/description
- `status`: JobStatus - Workflow state
- `currency`: string - ISO 4217 currency code
- `vatRate`: number - VAT percentage
- `budget`: number | undefined - Optional budget constraint
- `deletedAt`: Timestamp | null - Soft delete timestamp

**TypeScript Interface:**
```typescript
type JobStatus = 'active' | 'completed' | 'archived';

interface Job {
  tenantId: string;
  jobNumber: number | null;
  title: string;
  description?: string;
  status: JobStatus;
  currency: string;
  vatRate: number;
  budget?: number;
  deletedAt: Timestamp | null;  // Soft delete
  createdAt: Timestamp;
  createdBy: AuditMetadata;
  updatedAt: Timestamp;
  updatedBy: AuditMetadata;
}

interface AuditMetadata {
  uid: string;           // Firebase Auth UID (primary)
  memberNumber: number;  // For display purposes
  displayName: string;   // Cached for display
}
```

**Relationships:**
- Belongs to one Tenant
- One-to-many with Cost entities
- One-to-many with Journey entities
- Created/updated by Member entities (via UID)

## Cost Model

**Purpose:** Track all job-related expenses including materials, labor, transport, and equipment

**Key Attributes:**
- `ordinalNumber`: number | null - Sequential per job
- `category`: CostCategory - Type classification
- `amount`: number - Cost value in job currency
- `resource`: ResourceSnapshot - Denormalized resource data at creation time
- `deletedAt`: Timestamp | null - Soft delete timestamp

**TypeScript Interface:**
```typescript
type CostCategory = 'material' | 'transport' | 'labor' | 'machine' | 'expense';

interface Cost {
  tenantId: string;
  jobId: string;
  ordinalNumber: number | null;
  category: CostCategory;
  amount: number;
  description: string;
  resource?: ResourceSnapshot;
  deletedAt: Timestamp | null;  // Soft delete
  createdAt: Timestamp;
  createdBy: AuditMetadata;
  updatedAt: Timestamp;
  updatedBy: AuditMetadata;
}

// Resource snapshots (denormalized at creation)
interface TransportSnapshot {
  vehicleNumber: number;
  name: string;
  distanceUnit: 'km' | 'mi';
  ratePerDistanceUnit: number;
}

interface LaborSnapshot {
  teamMemberNumber: number;
  name: string;
  hourlyRate: number;
}

interface MachineSnapshot {
  machineNumber: number;
  name: string;
  hourlyRate: number;
}

type ResourceSnapshot = TransportSnapshot | LaborSnapshot | MachineSnapshot;
```

**Relationships:**
- Belongs to one Job
- References snapshot of Resource at creation time
- Created/updated by Member entities (via UID)

## Resource Models

**Purpose:** Reusable entities for vehicles, team members, and machines used across jobs

**Key Attributes:**
- Sequential numbers (vehicleNumber, teamMemberNumber, machineNumber)
- Name and rate information
- `status`: 'active' | 'inactive' - Availability state
- `deletedAt`: Timestamp | null - Soft delete (removed from system)

**TypeScript Interfaces:**
```typescript
interface Vehicle {
  tenantId: string;
  vehicleNumber: number | null;
  name: string;
  distanceUnit: 'km' | 'mi';
  ratePerDistanceUnit: number;
  status: 'active' | 'inactive';  // Availability
  deletedAt: Timestamp | null;      // Soft delete
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

interface TeamMember {
  tenantId: string;
  teamMemberNumber: number | null;
  name: string;
  hourlyRate: number;
  status: 'active' | 'inactive';  // Availability
  deletedAt: Timestamp | null;      // Soft delete
  createdAt: Timestamp;
  updatedAt: Timestamp;
}

interface Machine {
  tenantId: string;
  machineNumber: number | null;
  name: string;
  hourlyRate: number;
  status: 'active' | 'inactive';  // Availability
  deletedAt: Timestamp | null;      // Soft delete
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Relationships:**
- Belong to one Tenant
- Snapshot copied to Cost entities when referenced
- Many-to-many with Jobs through Cost entities

## Data Management Patterns

**Soft Delete Pattern:**
- All entities include `deletedAt: Timestamp | null`
- DELETE operations set `deletedAt` to current timestamp
- Queries filter out soft-deleted records by default: `.where('deletedAt', '==', null)`
- Soft-deleted records retained for audit trails and potential recovery

**Resource Availability Pattern:**
- Resources have `status: 'active' | 'inactive'` independent of deletion
- Active = available for selection in new costs
- Inactive = temporarily unavailable (vacation, maintenance, etc.)
- Soft-deleted = removed from system but retained for history

---

## Detailed Rationale

**Trade-offs and Choices:**

1. **Denormalized Resource Snapshots**: Copying resource data into costs preserves historical accuracy when rates change. Trade-off: data duplication for immutability.

2. **Nullable Sequential Numbers**: Allows offline entity creation with server-side assignment. Trade-off: complexity in handling null states for better offline support.

3. **Audit Metadata with UID + Display Data**: UID provides immutable reference while cached display data improves performance. Trade-off: redundancy for better UX and traceability.

4. **Dual Status/Delete Pattern**: Separate concerns of availability and deletion. Trade-off: additional field complexity for more flexible data lifecycle management.

**Key Assumptions:**
- Resource rates change infrequently
- Historical cost accuracy more important than storage efficiency
- Audit trails required for regulatory compliance
- Soft-deleted data retention period defined by business policy

**Design Decisions:**
- Using Firebase Auth UID as primary identifier for audit trails (globally unique and immutable)
- Resource snapshots instead of foreign keys (NoSQL denormalization pattern)
- Combining soft deletes with status enums (separate availability from deletion concerns)
- Caching display data in audit metadata (performance optimization)

---

*Next: [API Specification](./api-specification.md)*
