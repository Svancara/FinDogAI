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
  stripeCustomerId?: string;  // Stripe customer ID for billing
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
- Parent to all tenant-scoped collections (members, jobs, resources, subscription, payments)
- One-to-one with TenantSubscription
- One-to-many with Member entities
- One-to-many with Job entities
- One-to-many with PaymentRecord entities

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
- One-to-one with UserPreferences

## UserPreferences Model

**Purpose:** Store user-specific preferences including active job context for voice interactions

**Key Attributes:**
- `uid`: string - Firebase Auth UID (document ID, matches Member.uid)
- `tenantId`: string - Tenant scope
- `activeJobId`: string | null - Currently active job for voice context
- `lastActiveJobIds`: string[] - Recent jobs for quick switching (max 5)
- `locale`: string - User's preferred language (cs-CZ or en-US)
- `voiceEnabled`: boolean - Voice features enabled/disabled
- `defaultCurrency`: string | null - Override tenant default if needed
- `defaultVatRate`: number | null - Override tenant default if needed

**TypeScript Interface:**
```typescript
interface UserPreferences {
  uid: string;                    // Firebase Auth UID - document ID
  tenantId: string;               // Tenant scope
  activeJobId: string | null;     // Currently active job for voice context (FR1)
  lastActiveJobIds: string[];     // Recent jobs for quick switching (max 5)
  locale: string;                  // User language preference (cs-CZ or en-US)
  voiceEnabled: boolean;           // Voice features on/off
  defaultCurrency: string | null; // User's preferred currency
  defaultVatRate: number | null;  // User's preferred VAT rate
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Firestore Location:** `/users/{uid}/preferences/default`

**Relationships:**
- One-to-one with Member (same UID)
- References Job entity via activeJobId
- Created automatically on first user login

**Voice Context Usage:**
- When user says "add cost to active job", system uses `activeJobId`
- Updates on "Set Active Job" voice command (FR1)
- Cleared when job is completed or archived

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

## Business Profile Model

**Purpose:** Tenant-wide configuration and defaults for jobs, currency, VAT, and resource type visibility

**Key Attributes:**
- `currency`: string - ISO 4217 currency code (e.g., 'CZK', 'EUR', 'USD')
- `vatRate`: number - Default VAT percentage (0-100)
- `distanceUnit`: 'km' | 'mi' - Preferred distance measurement unit
- `usesMachines`: boolean - Whether business uses machines (controls UI visibility)
- `usesVehicles`: boolean - Whether business uses vehicles (controls UI visibility)
- `usesOtherExpenses`: boolean - Whether business tracks other expenses (controls UI visibility)

**TypeScript Interface:**
```typescript
interface BusinessProfile {
  tenantId: string;
  currency: string;         // ISO 4217 code
  vatRate: number;          // Percentage (0-100)
  distanceUnit: 'km' | 'mi';
  usesMachines: boolean;    // Default: true - Controls UI visibility for machine-related features
  usesVehicles: boolean;    // Default: true - Controls UI visibility for vehicle-related features
  usesOtherExpenses: boolean; // Default: true - Controls UI visibility for other expenses
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Relationships:**
- One-to-one with Tenant (created during tenant initialization)
- Referenced by Job creation for default values
- Referenced by Vehicle creation for distanceUnit default
- Controls conditional UI rendering throughout the application

**UI Visibility Control Logic:**
- When `usesMachines: false`, hide machine-related UI elements (add machine button, machine costs, machine resource tabs)
- When `usesVehicles: false`, hide vehicle-related UI elements (add vehicle button, vehicle/transport costs, vehicle resource tabs)
- When `usesOtherExpenses: false`, hide other expense UI elements (add expense button, expense costs in lists)
- Team member controls are ALWAYS visible (at least one team member must exist - the business owner)
- Existing resources of hidden types remain in database but are not selectable in UI

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

## Subscription Models

### Tenant Subscription Model

**Purpose:** Track subscription plan, billing status, usage limits, and current usage for each tenant

**Location:** `/tenants/{tenantId}/subscription/default` (single document per tenant)

**Key Attributes:**
- `stripeCustomerId`: string - Stripe customer identifier
- `stripeSubscriptionId`: string | null - Stripe subscription ID (null for free plan)
- `plan`: SubscriptionPlan - Current subscription tier
- `status`: SubscriptionStatus - Subscription state
- `limits`: Object - Plan-specific usage limits
- `usage`: Object - Current period usage tracking

**TypeScript Interface:**
```typescript
type SubscriptionPlan = 'free' | 'trial' | 'pro' | 'enterprise';

type SubscriptionStatus =
  | 'active'          // Paid subscription, full access
  | 'trialing'        // In trial period, full access
  | 'past_due'        // Payment failed, grace period (3 days)
  | 'canceled'        // User canceled, access until period end
  | 'incomplete'      // Initial payment pending
  | 'incomplete_expired'  // Trial expired without payment
  | 'unpaid';         // Payment failed, no access

interface TenantSubscription {
  tenantId: string;

  // Stripe Integration
  stripeCustomerId: string;
  stripeSubscriptionId: string | null;
  stripePriceId: string | null;

  // Plan & Status
  plan: SubscriptionPlan;
  status: SubscriptionStatus;

  // Trial Management
  trialStart: Timestamp | null;
  trialEnd: Timestamp | null;

  // Billing Period
  currentPeriodStart: Timestamp;
  currentPeriodEnd: Timestamp;

  // Plan Limits (copied from plan definition for performance)
  limits: {
    maxJobs: number;                   // -1 = unlimited
    maxTeamMembers: number;            // -1 = unlimited
    maxVoiceMinutesPerMonth: number;   // -1 = unlimited
    pdfExport: boolean;
  };

  // Current Usage (reset monthly)
  usage: {
    jobsCreated: number;
    activeTeamMembers: number;
    voiceMinutesThisMonth: number;
    lastVoiceUsageReset: Timestamp;
  };

  // Lifecycle Flags
  cancelAtPeriodEnd: boolean;
  canceledAt: Timestamp | null;

  // Metadata
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

**Relationships:**
- Belongs to one Tenant (one-to-one)
- Referenced by all operations requiring usage limit checks
- Updated by Stripe webhooks and usage tracking triggers

### Payment Record Model

**Purpose:** Historical record of all subscription payments and invoices

**Location:** `/tenants/{tenantId}/payments/{paymentId}`

**Key Attributes:**
- `stripeInvoiceId`: string - Stripe invoice identifier
- `amount`: number - Payment amount in cents
- `status`: PaymentStatus - Payment state
- `invoicePdfUrl`: string - Stripe-hosted invoice PDF URL

**TypeScript Interface:**
```typescript
type PaymentStatus = 'paid' | 'open' | 'void' | 'uncollectible';

interface PaymentRecord {
  tenantId: string;
  paymentId: string;  // Auto-generated Firestore ID

  // Stripe References
  stripeInvoiceId: string;
  stripePaymentIntentId: string;
  stripeChargeId: string | null;

  // Payment Details
  amount: number;     // Amount in cents (2900 = €29.00)
  currency: string;   // 'eur'
  status: PaymentStatus;

  // Invoice
  invoicePdfUrl: string | null;
  invoiceNumber: string | null;

  // Period Covered
  periodStart: Timestamp;
  periodEnd: Timestamp;

  // Failure Information
  failureReason: string | null;
  failureMessage: string | null;

  // Metadata
  createdAt: Timestamp;
  paidAt: Timestamp | null;
}
```

**Relationships:**
- Belongs to one Tenant
- Created by Stripe webhook handlers
- Referenced for billing history and invoice access

**Usage Tracking:**
- Job creation increments `usage.jobsCreated`
- Member changes update `usage.activeTeamMembers`
- Voice commands increment `usage.voiceMinutesThisMonth`
- Usage resets monthly on subscription renewal

**Subscription Plans:**
See [Subscription Billing Architecture](./subscription-billing.md) for detailed plan definitions and limits.

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
