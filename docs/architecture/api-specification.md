[Back to Index](./index.md) | [Previous: Data Models](./data-models.md) | [Next: Components](./components.md)

# API Specification

FinDogAI uses a hybrid API approach: Direct Firestore SDK for real-time data operations with offline support, and Cloud Functions (HTTPS Callable) for server-side operations that require secure backend processing.

## Firestore Direct Access Pattern

The frontend accesses Firestore directly using the Firebase SDK with security rules enforcement. This is NOT a traditional REST API but provides superior offline capabilities.

**TypeScript Service Interface:**
```typescript
// Base Firestore service pattern
interface FirestoreService<T> {
  get(id: string): Promise<T>;
  list(query?: QueryConstraints): Observable<T[]>;
  create(data: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<string>;
  update(id: string, data: Partial<T>): Promise<void>;
  delete(id: string): Promise<void>;  // Soft delete
  subscribe(id: string): Observable<T>;
}
```

**Real-time Subscription Example:**
```typescript
// Jobs service with real-time updates
class JobsService implements FirestoreService<Job> {
  private db = inject(Firestore);
  private tenantId = inject(TenantService).currentTenantId;

  list(status?: JobStatus): Observable<Job[]> {
    const jobsRef = collection(this.db, `tenants/${this.tenantId}/jobs`);
    const q = query(
      jobsRef,
      where('deletedAt', '==', null),
      status ? where('status', '==', status) : null,
      orderBy('createdAt', 'desc')
    ).filter(Boolean);

    return collectionData(q, { idField: 'id' });
  }

  async create(job: CreateJobDto): Promise<string> {
    const jobsRef = collection(this.db, `tenants/${this.tenantId}/jobs`);
    const docRef = await addDoc(jobsRef, {
      ...job,
      jobNumber: null, // Will be assigned by Cloud Function
      createdAt: serverTimestamp(),
      createdBy: this.getCurrentUserAudit(),
      updatedAt: serverTimestamp(),
      updatedBy: this.getCurrentUserAudit()
    });
    return docRef.id;
  }
}
```

## Cloud Functions Callable API

Server-side operations exposed as HTTPS Callable functions with automatic auth context.

### Function: `allocateSequence`

**Purpose:** Atomically allocate next sequential number for tenant resources

**Request:**
```typescript
interface AllocateSequenceRequest {
  tenantId: string;
  sequenceType: 'job' | 'vehicle' | 'teamMember' | 'machine' | 'cost';
  entityId?: string;  // For cost sequences (job-scoped)
}
```

**Response:**
```typescript
interface AllocateSequenceResponse {
  sequenceNumber: number;
  sequenceType: string;
  allocatedAt: Timestamp;
}
```

**Example Call:**
```typescript
const allocateSequence = httpsCallable<AllocateSequenceRequest, AllocateSequenceResponse>(
  functions,
  'allocateSequence'
);

const result = await allocateSequence({
  tenantId: 'tenant123',
  sequenceType: 'job'
});
console.log(`Allocated job number: ${result.data.sequenceNumber}`);
```

### Function: `generatePDF` (Phase 2)

**Purpose:** Generate PDF reports for jobs with cost breakdowns

**Request:**
```typescript
interface GeneratePDFRequest {
  jobId: string;
  includeDetails: {
    costs: boolean;
    journeys: boolean;
    materials: boolean;
    laborHours: boolean;
  };
  format: 'invoice' | 'report' | 'summary';
  language: 'en' | 'cs' | 'de';
}
```

**Response:**
```typescript
interface GeneratePDFResponse {
  pdfUrl: string;         // Signed URL valid for 1 hour
  expiresAt: Timestamp;
  fileName: string;
  sizeBytes: number;
}
```

### Function: `processVoiceCommand`

**Purpose:** Process complex voice commands that require server-side LLM integration

**Request:**
```typescript
interface ProcessVoiceCommandRequest {
  transcript: string;
  context: {
    activeJobId?: string;
    currentView: string;
    recentEntities: string[];  // Recent IDs for context
  };
  language: 'en' | 'cs' | 'de';
}
```

**Response:**
```typescript
interface ProcessVoiceCommandResponse {
  intent: 'navigate' | 'create' | 'update' | 'query' | 'unknown';
  confidence: number;
  action?: {
    type: string;
    payload: any;
  };
  suggestedResponse: string;
  needsConfirmation: boolean;
}
```

### Function: `inviteTeamMember`

**Purpose:** Generate secure invitation code for team members

**Request:**
```typescript
interface InviteTeamMemberRequest {
  role: 'representative' | 'teamMember';
  email?: string;  // Optional pre-assignment
  expirationDays?: number;  // Default 7
}
```

**Response:**
```typescript
interface InviteTeamMemberResponse {
  inviteCode: string;      // 6-digit code
  inviteId: string;
  expiresAt: Timestamp;
  shareUrl: string;        // Deep link for mobile app
}
```

## Firestore Triggers (Automatic Backend Processing)

These Cloud Functions run automatically in response to Firestore changes:

### Trigger: `onJobCreate`

```typescript
// Automatically assigns job number when job created without one
export const onJobCreate = onDocumentCreated(
  'tenants/{tenantId}/jobs/{jobId}',
  async (event) => {
    const job = event.data?.data();
    if (!job?.jobNumber) {
      // Allocate and assign job number
      const jobNumber = await allocateNextJobNumber(event.params.tenantId);
      await event.data?.ref.update({ jobNumber });
    }

    // Create audit log entry
    await createAuditLog({
      tenantId: event.params.tenantId,
      action: 'job.created',
      entityId: event.params.jobId,
      userId: job.createdBy.uid,
      metadata: { jobNumber }
    });
  }
);
```

### Trigger: `onCostWrite`

```typescript
// Handles cost creation, update, and deletion with audit logging
export const onCostWrite = onDocumentWritten(
  'tenants/{tenantId}/jobs/{jobId}/costs/{costId}',
  async (event) => {
    const before = event.data?.before.data();
    const after = event.data?.after.data();

    // Determine action type
    const action = !before ? 'created' : !after ? 'deleted' : 'updated';

    // Create audit log
    await createAuditLog({
      tenantId: event.params.tenantId,
      action: `cost.${action}`,
      entityId: event.params.costId,
      parentId: event.params.jobId,
      userId: after?.updatedBy?.uid || before?.updatedBy?.uid,
      changes: action === 'updated' ? getDiff(before, after) : null
    });
  }
);
```

## Business Profile API

Direct Firestore access pattern for business configuration.

### TypeScript Service Interface
```typescript
interface BusinessProfileService {
  get(): Observable<BusinessProfile>;
  update(updates: Partial<BusinessProfile>): Promise<void>;
  shouldShowResourceType(type: 'machines' | 'vehicles' | 'otherExpenses'): Observable<boolean>;
}
```

### Business Profile Document Structure
```typescript
interface BusinessProfile {
  tenantId: string;
  currency: string;              // ISO 4217 code (CZK, EUR, USD)
  vatRate: number;               // Percentage 0-100
  distanceUnit: 'km' | 'mi';
  usesMachines: boolean;         // Default: true
  usesVehicles: boolean;         // Default: true
  usesOtherExpenses: boolean;    // Default: true
  createdAt: Timestamp;
  updatedAt: Timestamp;
}
```

### Validation Rules
```typescript
// Business Profile validation
function validateBusinessProfile(profile: Partial<BusinessProfile>): ValidationResult {
  const errors: string[] = [];

  // Currency validation
  if (profile.currency && !['CZK', 'EUR', 'USD'].includes(profile.currency)) {
    errors.push('Currency must be one of: CZK, EUR, USD');
  }

  // VAT rate validation
  if (profile.vatRate !== undefined) {
    if (profile.vatRate < 0 || profile.vatRate > 100) {
      errors.push('VAT rate must be between 0 and 100');
    }
  }

  // Distance unit validation
  if (profile.distanceUnit && !['km', 'mi'].includes(profile.distanceUnit)) {
    errors.push('Distance unit must be either km or mi');
  }

  // Boolean flag validation
  if (profile.usesMachines !== undefined && typeof profile.usesMachines !== 'boolean') {
    errors.push('usesMachines must be a boolean');
  }
  if (profile.usesVehicles !== undefined && typeof profile.usesVehicles !== 'boolean') {
    errors.push('usesVehicles must be a boolean');
  }
  if (profile.usesOtherExpenses !== undefined && typeof profile.usesOtherExpenses !== 'boolean') {
    errors.push('usesOtherExpenses must be a boolean');
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

## Security Rules Integration

All direct Firestore access is secured by Firebase Security Rules:

```javascript
// Example security rules enforcing tenant isolation
match /tenants/{tenantId}/jobs/{jobId} {
  allow read: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));

  allow create: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)) &&
    get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role in ['owner', 'representative'];

  allow update: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)) &&
    request.resource.data.tenantId == resource.data.tenantId;  // Prevent tenant ID changes
}

// Business Profile security rules
match /tenants/{tenantId}/businessProfile/default {
  // All tenant members can read business profile
  allow read: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));

  // Only owners can update business profile
  allow update: if request.auth != null &&
    exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)) &&
    get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role == 'owner' &&
    request.resource.data.tenantId == resource.data.tenantId &&  // Prevent tenant ID changes
    request.resource.data.vatRate >= 0 && request.resource.data.vatRate <= 100 &&  // Validate VAT rate
    request.resource.data.currency in ['CZK', 'EUR', 'USD'] &&  // Validate currency
    request.resource.data.distanceUnit in ['km', 'mi'];  // Validate distance unit
}
```

## Error Handling

**Firestore SDK Errors:**
```typescript
interface FirestoreError {
  code: 'permission-denied' | 'not-found' | 'already-exists' | 'resource-exhausted';
  message: string;
}
```

**Cloud Functions Errors:**
```typescript
interface CallableError {
  code: 'unauthenticated' | 'permission-denied' | 'invalid-argument' | 'internal';
  message: string;
  details?: any;
}
```

**Client Error Handler:**
```typescript
class ApiErrorHandler {
  handle(error: any): void {
    if (error.code === 'permission-denied') {
      this.toast.error('You do not have permission for this action');
    } else if (error.code === 'resource-exhausted') {
      this.toast.error('Rate limit exceeded. Please try again later.');
    } else if (error.code === 'unauthenticated') {
      this.router.navigate(['/login']);
    } else {
      this.toast.error('An unexpected error occurred');
      console.error('API Error:', error);
    }
  }
}
```

---

*Next: [Components](./components.md)*
