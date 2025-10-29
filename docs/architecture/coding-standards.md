[Back to Index](./index.md) | [Previous: Deployment](./deployment.md)

# Coding Standards

## Critical Fullstack Rules

These MINIMAL but CRITICAL standards prevent common mistakes in AI-driven development:

- **Type Sharing:** Always define types in `packages/shared-types` and import from there - never duplicate type definitions
- **API Calls:** Never make direct HTTP calls from components - always use the service layer
- **Environment Variables:** Access only through config objects (`environment.ts`), never `process.env` directly
- **Error Handling:** All API operations must use the centralized `ApiErrorHandler` service
- **State Updates:** Never mutate state directly - use NgRx actions for all state changes
- **Offline First:** Always check `navigator.onLine` before external API calls (LLM, STT, TTS)
- **Tenant Isolation:** Never construct Firestore paths manually - use service methods that enforce tenant context
- **Soft Deletes:** DELETE operations must set `deletedAt` timestamp, never remove documents
- **Voice Commands:** Always provide visual feedback for voice interactions (loading, success, error states)
- **Audit Trail:** Never bypass audit metadata (createdBy, updatedBy) when modifying data

## Naming Conventions

| Element | Frontend | Backend | Example |
|---------|----------|---------|---------|
| Components | PascalCase | - | `JobDetailsComponent.ts` |
| Services | PascalCase + Service | PascalCase + Service | `JobsService.ts` |
| Hooks | camelCase with 'use' | - | `useVoiceCommand.ts` |
| Cloud Functions | - | camelCase | `allocateSequence.ts` |
| Firestore Collections | - | snake_case | `team_members` |
| TypeScript Interfaces | PascalCase | PascalCase | `Job`, `Cost`, `Member` |
| File Names | kebab-case | kebab-case | `job-details.component.ts` |

## File Organization Rules

### Import Order
```typescript
// Correct import order
// 1. Angular/Ionic imports
import { Component, inject } from '@angular/core';
import { IonContent, IonHeader } from '@ionic/angular/standalone';

// 2. Third-party libraries
import { Observable } from 'rxjs';
import { Store } from '@ngrx/store';

// 3. Shared packages
import { Job, Cost } from '@findogai/shared-types';

// 4. Local imports
import { JobsService } from '../../services/jobs.service';
```

## TypeScript Standards

### Type Definitions
```typescript
// ✅ GOOD: Define in shared-types package
// packages/shared-types/src/entities/job.ts
export interface Job {
  tenantId: string;
  jobNumber: number | null;
  title: string;
  status: JobStatus;
}

// ❌ BAD: Duplicate definitions in components
interface Job {  // Don't do this!
  // ...
}
```

### Null and Undefined Handling
```typescript
// ✅ GOOD: Explicit null checks
if (job?.jobNumber !== null && job?.jobNumber !== undefined) {
  console.log(job.jobNumber);
}

// ✅ GOOD: Use nullish coalescing
const displayNumber = job?.jobNumber ?? 'Pending';

// ❌ BAD: Truthy checks that fail for 0
if (job?.jobNumber) {  // This fails when jobNumber is 0!
  console.log(job.jobNumber);
}
```

### Promise Handling
```typescript
// ✅ GOOD: Proper async/await with error handling
async function createJob(job: CreateJobDto): Promise<string> {
  try {
    const jobId = await this.jobsService.create(job);
    return jobId;
  } catch (error) {
    this.errorHandler.handle(error);
    throw error;
  }
}

// ❌ BAD: Unhandled promises
function createJob(job: CreateJobDto): Promise<string> {
  return this.jobsService.create(job);  // No error handling!
}
```

## Angular Standards

### Component Structure
```typescript
@Component({
  selector: 'app-job-details',
  standalone: true,
  imports: [CommonModule, IonContent, IonHeader],
  templateUrl: './job-details.component.html',
  styleUrls: ['./job-details.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class JobDetailsComponent {
  // 1. Inputs and Outputs
  @Input() jobId!: string;
  @Output() jobUpdated = new EventEmitter<Job>();

  // 2. Public properties
  job$ = signal<Job | null>(null);

  // 3. Private properties
  private jobsService = inject(JobsService);
  private store = inject(Store);

  // 4. Lifecycle hooks
  ngOnInit(): void {
    this.loadJob();
  }

  // 5. Public methods
  loadJob(): void {
    // Implementation
  }

  // 6. Private methods
  private updateJob(): void {
    // Implementation
  }
}
```

### Service Structure
```typescript
@Injectable({
  providedIn: 'root'
})
export class JobsService {
  // 1. Dependencies
  private db = inject(Firestore);
  private tenantService = inject(TenantService);
  private errorHandler = inject(ApiErrorHandler);

  // 2. Public methods
  async create(job: CreateJobDto): Promise<string> {
    try {
      const tenantId = this.tenantService.currentTenantId;
      const jobsRef = collection(this.db, `tenants/${tenantId}/jobs`);
      const docRef = await addDoc(jobsRef, {
        ...job,
        createdAt: serverTimestamp(),
        createdBy: this.getCurrentUserAudit()
      });
      return docRef.id;
    } catch (error) {
      this.errorHandler.handle(error);
      throw error;
    }
  }

  // 3. Private methods
  private getCurrentUserAudit(): AuditMetadata {
    // Implementation
  }
}
```

### Template Standards
```html
<!-- ✅ GOOD: Use async pipe for observables -->
<div *ngIf="job$ | async as job">
  <h1>{{ job.title }}</h1>
</div>

<!-- ✅ GOOD: TrackBy for lists -->
<ion-item *ngFor="let cost of costs; trackBy: trackByCostId">
  {{ cost.description }}
</ion-item>

<!-- ❌ BAD: Subscribe in component -->
<!-- Don't subscribe manually unless necessary -->
```

## Cloud Functions Standards

### Function Structure
```typescript
import { onCall } from 'firebase-functions/v2/https';
import { logger } from 'firebase-functions/v2';

export const allocateSequence = onCall({
  region: 'europe-west1',
  memory: '256MiB',
  timeoutSeconds: 60
}, async (request) => {
  // 1. Validate authentication
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'User must be authenticated');
  }

  // 2. Validate input
  const { tenantId, sequenceType } = request.data;
  if (!tenantId || !sequenceType) {
    throw new HttpsError('invalid-argument', 'Missing required fields');
  }

  // 3. Log operation
  logger.info('Allocating sequence', { tenantId, sequenceType });

  try {
    // 4. Business logic
    const sequenceNumber = await allocateNextNumber(tenantId, sequenceType);

    // 5. Return result
    return {
      sequenceNumber,
      sequenceType,
      allocatedAt: FieldValue.serverTimestamp()
    };
  } catch (error) {
    // 6. Error handling
    logger.error('Sequence allocation failed', error);
    throw new HttpsError('internal', 'Failed to allocate sequence');
  }
});
```

### Firestore Trigger Structure
```typescript
import { onDocumentCreated } from 'firebase-functions/v2/firestore';
import { logger } from 'firebase-functions/v2';

export const onJobCreate = onDocumentCreated(
  'tenants/{tenantId}/jobs/{jobId}',
  async (event) => {
    const { tenantId, jobId } = event.params;
    const job = event.data?.data();

    if (!job) {
      logger.warn('Job data is null', { tenantId, jobId });
      return;
    }

    logger.info('Job created', { tenantId, jobId });

    try {
      // Assign job number if not set
      if (!job.jobNumber) {
        const jobNumber = await allocateJobNumber(tenantId);
        await event.data?.ref.update({ jobNumber });
      }

      // Create audit log
      await createAuditLog({
        tenantId,
        action: 'job.created',
        entityId: jobId,
        userId: job.createdBy.uid
      });
    } catch (error) {
      logger.error('Job creation processing failed', error);
      // Don't throw - triggers should be idempotent
    }
  }
);
```

## Error Handling Standards

### Frontend Error Handling
```typescript
@Injectable({
  providedIn: 'root'
})
export class ApiErrorHandler {
  private toastController = inject(ToastController);
  private router = inject(Router);

  handle(error: any): void {
    // Log to console in development
    if (!environment.production) {
      console.error('API Error:', error);
    }

    // User-friendly messages
    if (error.code === 'permission-denied') {
      this.showToast('You do not have permission for this action');
    } else if (error.code === 'unauthenticated') {
      this.showToast('Session expired. Please log in again.');
      this.router.navigate(['/login']);
    } else if (error.code === 'resource-exhausted') {
      this.showToast('Rate limit exceeded. Please try again later.');
    } else {
      this.showToast('An unexpected error occurred');
    }

    // Report to Sentry in production
    if (environment.production) {
      Sentry.captureException(error);
    }
  }

  private async showToast(message: string): Promise<void> {
    const toast = await this.toastController.create({
      message,
      duration: 3000,
      position: 'bottom',
      color: 'danger'
    });
    await toast.present();
  }
}
```

## Testing Standards

### Unit Test Structure
```typescript
describe('JobsService', () => {
  let service: JobsService;
  let firestore: Firestore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        JobsService,
        { provide: Firestore, useValue: mockFirestore }
      ]
    });
    service = TestBed.inject(JobsService);
    firestore = TestBed.inject(Firestore);
  });

  it('should create job with audit metadata', async () => {
    // Arrange
    const job: CreateJobDto = {
      title: 'Test Job',
      status: 'active'
    };

    // Act
    const jobId = await service.create(job);

    // Assert
    expect(jobId).toBeDefined();
    expect(addDoc).toHaveBeenCalledWith(
      expect.anything(),
      expect.objectContaining({
        title: 'Test Job',
        createdBy: expect.objectContaining({ uid: expect.any(String) })
      })
    );
  });
});
```

## Documentation Standards

### Code Comments
```typescript
/**
 * Allocates the next sequential number for a tenant resource.
 *
 * @param tenantId - The tenant identifier
 * @param sequenceType - Type of sequence (job, vehicle, etc.)
 * @returns The allocated sequence number
 * @throws {HttpsError} If allocation fails
 *
 * @example
 * const jobNumber = await allocateSequence('tenant123', 'job');
 * console.log(jobNumber); // 1
 */
export async function allocateSequence(
  tenantId: string,
  sequenceType: SequenceType
): Promise<number> {
  // Implementation
}
```

### Inline Comments
```typescript
// ✅ GOOD: Explain WHY, not WHAT
// Use transaction to prevent race conditions when multiple jobs are created simultaneously
await runTransaction(db, async (transaction) => {
  // ...
});

// ❌ BAD: Obvious comments
// Increment the counter
counter++;
```

## Security Standards

### Input Validation
```typescript
// ✅ GOOD: Validate and sanitize all inputs
function createJob(job: CreateJobDto): Promise<string> {
  // Validate required fields
  if (!job.title || !job.status) {
    throw new Error('Missing required fields');
  }

  // Sanitize string inputs
  const sanitizedTitle = this.sanitize(job.title);

  // Validate enums
  if (!['active', 'completed', 'archived'].includes(job.status)) {
    throw new Error('Invalid status value');
  }

  return this.jobsService.create({
    ...job,
    title: sanitizedTitle
  });
}
```

### Tenant Isolation
```typescript
// ✅ GOOD: Always use tenant context
class JobsService {
  private getTenantPath(): string {
    const tenantId = this.tenantService.currentTenantId;
    return `tenants/${tenantId}`;
  }

  list(): Observable<Job[]> {
    const path = `${this.getTenantPath()}/jobs`;
    return collectionData(collection(this.db, path));
  }
}

// ❌ BAD: Hardcoded or user-provided paths
list(tenantId: string): Observable<Job[]> {  // Don't trust user input!
  return collectionData(collection(this.db, `tenants/${tenantId}/jobs`));
}
```

## Performance Standards

### Minimize Bundle Size
```typescript
// ✅ GOOD: Import only what you need
import { map } from 'rxjs/operators';

// ❌ BAD: Import entire library
import * as rxjs from 'rxjs';
```

### Optimize Firestore Queries
```typescript
// ✅ GOOD: Use indexes and limit results
const q = query(
  collection(db, 'jobs'),
  where('status', '==', 'active'),
  where('deletedAt', '==', null),
  orderBy('createdAt', 'desc'),
  limit(50)
);

// ❌ BAD: Fetch all and filter in memory
const allJobs = await getDocs(collection(db, 'jobs'));
const activeJobs = allJobs.docs.filter(doc => doc.data().status === 'active');
```

---

*End of Coding Standards. [Return to Index](./index.md)*
