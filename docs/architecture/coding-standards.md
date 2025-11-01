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
- **Template Syntax:** Use `@if`, `@for`, `@switch` control flow (Angular 17+) instead of `*ngIf`, `*ngFor`, `*ngSwitch`
- **Internationalization (i18n):** ALL user-facing text MUST use translation keys via TranslateService/TranslatePipe - never hardcode UI strings

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
<!-- ✅ GOOD: Use @if control flow (Angular 17+) -->
@if (job$ | async; as job) {
  <h1>{{ job.title }}</h1>
}

<!-- ✅ GOOD: Use @for with track (Angular 17+) -->
@for (cost of costs; track cost.id) {
  <ion-item>{{ cost.description }}</ion-item>
}

<!-- ⚠️ LEGACY: *ngIf and *ngFor (use @if/@for instead) -->
<!-- <div *ngIf="job$ | async as job">...</div> -->

<!-- ❌ BAD: Subscribe in component -->
<!-- Don't subscribe manually unless necessary -->
```

### Conditional UI Rendering Based on Business Profile

**Pattern:** Use Business Profile flags to control UI visibility for resource types

```typescript
// ✅ GOOD: Service-based visibility control with @if control flow
@Component({
  selector: 'app-resources',
  template: `
    <ion-tabs>
      <ion-tab-button tab="team-members">
        <ion-label>Team Members</ion-label>
      </ion-tab-button>

      <!-- Conditionally show vehicle tab -->
      @if (showVehicles$ | async) {
        <ion-tab-button tab="vehicles">
          <ion-label>Vehicles</ion-label>
        </ion-tab-button>
      }

      <!-- Conditionally show machine tab -->
      @if (showMachines$ | async) {
        <ion-tab-button tab="machines">
          <ion-label>Machines</ion-label>
        </ion-tab-button>
      }
    </ion-tabs>
  `
})
export class ResourcesComponent {
  private businessProfileService = inject(BusinessProfileService);

  // Reactive visibility streams
  showVehicles$ = this.businessProfileService.shouldShowResourceType('vehicles');
  showMachines$ = this.businessProfileService.shouldShowResourceType('machines');
  showOtherExpenses$ = this.businessProfileService.shouldShowResourceType('otherExpenses');
}
```

```typescript
// ✅ GOOD: BusinessProfileService implementation
@Injectable({
  providedIn: 'root'
})
export class BusinessProfileService {
  private db = inject(Firestore);
  private tenantService = inject(TenantService);

  private businessProfile$: Observable<BusinessProfile> = this.getBusinessProfile();

  shouldShowResourceType(type: 'machines' | 'vehicles' | 'otherExpenses'): Observable<boolean> {
    return this.businessProfile$.pipe(
      map(profile => {
        switch(type) {
          case 'machines': return profile.usesMachines;
          case 'vehicles': return profile.usesVehicles;
          case 'otherExpenses': return profile.usesOtherExpenses;
          default: return false;
        }
      }),
      distinctUntilChanged()
    );
  }

  private getBusinessProfile(): Observable<BusinessProfile> {
    const tenantId = this.tenantService.currentTenantId;
    const docRef = doc(this.db, `tenants/${tenantId}/businessProfile/default`);
    return docData(docRef).pipe(
      shareReplay(1) // Cache and share among subscribers
    );
  }
}
```

```typescript
// ✅ GOOD: Filter cost types based on business profile
@Component({
  selector: 'app-cost-form',
  template: `
    <ion-select [(ngModel)]="costCategory">
      <ion-select-option value="material">Material</ion-select-option>
      <ion-select-option value="labor">Labor</ion-select-option>
      @if (showVehicles$ | async) {
        <ion-select-option value="transport">Transport</ion-select-option>
      }
      @if (showMachines$ | async) {
        <ion-select-option value="machine">Machine</ion-select-option>
      }
      @if (showOtherExpenses$ | async) {
        <ion-select-option value="expense">Other Expense</ion-select-option>
      }
    </ion-select>
  `
})
export class CostFormComponent {
  private businessProfileService = inject(BusinessProfileService);

  showVehicles$ = this.businessProfileService.shouldShowResourceType('vehicles');
  showMachines$ = this.businessProfileService.shouldShowResourceType('machines');
  showOtherExpenses$ = this.businessProfileService.shouldShowResourceType('otherExpenses');
}
```

**Key Rules:**
1. **Team Members UI is NEVER hidden** - Business owner always exists as team member #1
2. **Use async pipe** - Avoid manual subscriptions for visibility flags
3. **Use @if control flow** - Prefer `@if` over `*ngIf` for all conditional rendering (Angular 17+)
4. **Reactive updates** - UI changes immediately when Business Profile is updated
5. **Existing data preserved** - Hidden resources remain in database, just not visible/selectable in UI
6. **Auto-selection still works** - When exactly one resource exists, auto-select regardless of visibility flags
7. **Apply consistently** - Hide buttons, tabs, filters, and options for disabled resource types across all screens

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

## Internationalization (i18n) Standards

### Language Support Strategy

- **MVP Scope**: English (en) and Czech (cs) only
- **Phase 2 Goal**: Multiple languages
- **Critical Requirement**: ALL UI text must be externalized and translation-ready from day one

### i18n Library Configuration

**Library**: `@ngx-translate/core` with `@ngx-translate/http-loader`
**Translation Files**: `public/i18n/{lang}.json` (e.g., en.json, cs.json)
**Fallback Language**: English (`en`)
**Caching**: Service Worker caches translation files for offline access

### Translation File Structure

```json
{
  "common": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "edit": "Edit",
    "retry": "Retry"
  },
  "auth": {
    "signIn": "Sign In",
    "signOut": "Sign Out",
    "email": "Email",
    "password": "Password",
    "errorInvalidCredentials": "Invalid email or password"
  },
  "jobs": {
    "title": "Jobs",
    "createJob": "Create Job",
    "jobNumber": "Job #{{number}}",
    "status": {
      "draft": "Draft",
      "active": "Active",
      "completed": "Completed"
    }
  }
}
```

### Translation Key Naming Convention

- Use **dot notation** for hierarchical organization
- Group by **feature domain** (auth, jobs, costs, resources, etc.)
- Use **camelCase** for key names
- Prefix error messages with `error`
- Use `common.*` for shared UI elements

**Pattern**: `{domain}.{feature}.{context}`

Examples:
- `jobs.list.title` → "Job List"
- `costs.form.amountLabel` → "Amount"
- `auth.errors.errorInvalidEmail` → "Invalid email format"
- `common.buttons.save` → "Save"

### Template Usage - TranslatePipe

```typescript
// ✅ GOOD: Use translate pipe in templates
template: `
  <ion-header>
    <ion-toolbar>
      <ion-title>{{ 'jobs.list.title' | translate }}</ion-title>
    </ion-toolbar>
  </ion-header>

  <ion-button>{{ 'common.buttons.save' | translate }}</ion-button>

  <!-- With interpolation parameters -->
  <h2>{{ 'jobs.detail.jobNumber' | translate: {number: job.jobNumber} }}</h2>

  <!-- Pluralization -->
  <p>{{ 'jobs.list.itemCount' | translate: {count: jobs.length} }}</p>
`

// ❌ BAD: Hardcoded strings
template: `
  <ion-title>Job List</ion-title>
  <ion-button>Save</ion-button>
`
```

### Component Usage - TranslateService

```typescript
// ✅ GOOD: Use TranslateService for dynamic strings
export class JobFormComponent {
  private readonly translate = inject(TranslateService);
  private readonly toastController = inject(ToastController);

  async saveJob(): Promise<void> {
    try {
      await this.jobsService.save(this.job);
      const message = this.translate.instant('jobs.form.saveSuccess');
      this.showToast(message, 'success');
    } catch (error) {
      const message = this.translate.instant('jobs.form.errorSaveFailed');
      this.showToast(message, 'danger');
    }
  }

  async showToast(message: string, color: string): Promise<void> {
    const toast = await this.toastController.create({
      message,
      duration: 3000,
      color
    });
    await toast.present();
  }
}

// ❌ BAD: Hardcoded strings in component logic
async saveJob(): Promise<void> {
  this.showToast('Job saved successfully', 'success');
}
```

### Language Switching Implementation

```typescript
// LanguageService - wraps TranslateService with persistence
@Injectable({
  providedIn: 'root'
})
export class LanguageService {
  private readonly translate = inject(TranslateService);
  private readonly langSubject = new BehaviorSubject<string>('en');

  public readonly lang$ = this.langSubject.asObservable();

  constructor() {
    this.initializeLanguage();
  }

  private initializeLanguage(): void {
    // Priority: localStorage > browser language > default 'en'
    const storedLang = localStorage.getItem('lang');
    const browserLang = this.translate.getBrowserLang() || 'en';
    const initialLang = storedLang || (browserLang === 'cs' ? 'cs' : 'en');

    this.setLanguage(initialLang);
  }

  setLanguage(lang: string): void {
    this.translate.use(lang);
    localStorage.setItem('lang', lang);
    this.langSubject.next(lang);
  }

  getCurrentLanguage(): string {
    return this.langSubject.value;
  }
}
```

### Language Selector Component

```typescript
// ✅ GOOD: Language selector with persistence
@Component({
  selector: 'app-language-selector',
  standalone: true,
  imports: [IonicModule, CommonModule],
  template: `
    <ion-select
      [value]="currentLang()"
      (ionChange)="onLanguageChange($event)"
      interface="popover">
      <ion-select-option value="en">English</ion-select-option>
      <ion-select-option value="cs">Čeština</ion-select-option>
    </ion-select>
  `
})
export class LanguageSelectorComponent {
  private readonly languageService = inject(LanguageService);
  protected readonly currentLang = signal<string>('en');

  constructor() {
    this.languageService.lang$.pipe(
      takeUntilDestroyed()
    ).subscribe(lang => this.currentLang.set(lang));
  }

  protected onLanguageChange(event: CustomEvent): void {
    this.languageService.setLanguage(event.detail.value);
  }
}
```

### i18n Bootstrap Configuration

```typescript
// main.ts - configure ngx-translate on app bootstrap
import { provideHttpClient } from '@angular/common/http';
import { importProvidersFrom } from '@angular/core';
import { TranslateModule, TranslateLoader } from '@ngx-translate/core';
import { TranslateHttpLoader } from '@ngx-translate/http-loader';
import { HttpClient } from '@angular/common/http';

export function HttpLoaderFactory(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(),
    importProvidersFrom(
      TranslateModule.forRoot({
        defaultLanguage: 'en',
        loader: {
          provide: TranslateLoader,
          useFactory: HttpLoaderFactory,
          deps: [HttpClient]
        }
      })
    ),
    // ... other providers
  ]
});
```

### Offline i18n Considerations

```typescript
// Service Worker configuration (ngsw-config.json)
{
  "assetGroups": [
    {
      "name": "i18n",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/assets/i18n/*.json"
        ]
      }
    }
  ]
}
```

### i18n Validation

```typescript
// ✅ GOOD: CI script to validate key consistency
// scripts/validate-i18n.js
const fs = require('fs');

const languages = ['en', 'cs'];
const baseKeys = getKeys(JSON.parse(fs.readFileSync('public/i18n/en.json')));

languages.forEach(lang => {
  if (lang === 'en') return;
  const langKeys = getKeys(JSON.parse(fs.readFileSync(`public/i18n/${lang}.json`)));
  const missing = baseKeys.filter(key => !langKeys.includes(key));
  if (missing.length > 0) {
    console.error(`Missing keys in ${lang}.json:`, missing);
    process.exit(1);
  }
});

function getKeys(obj, prefix = '') {
  let keys = [];
  for (const [key, value] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === 'object' && value !== null) {
      keys = keys.concat(getKeys(value, fullKey));
    } else {
      keys.push(fullKey);
    }
  }
  return keys;
}
```

### i18n Best Practices

1. **Never hardcode UI text** - Use translation keys for all user-facing strings
2. **Group by feature** - Organize keys by domain (auth, jobs, costs, etc.)
3. **Use parameters** - For dynamic content: `{{ 'key' | translate: {param: value} }}`
4. **Validate keys** - Run validation script in CI to catch missing translations
5. **Fallback gracefully** - Always provide English fallback for missing keys
6. **Cache translations** - Service Worker must cache i18n JSON files
7. **Test both languages** - Include Czech translations in E2E tests
8. **Keep files small** - Split by feature if i18n files become too large (>50KB)
9. **Use instant() sparingly** - Prefer async pipe in templates over instant() in components
10. **Document context** - Add comments in translation files for translators

### Common i18n Mistakes to Avoid

```typescript
// ❌ BAD: String concatenation
const message = 'You have ' + count + ' items';

// ✅ GOOD: Use interpolation
const message = this.translate.instant('items.count', { count });

// ❌ BAD: Conditional text in component
const status = job.isActive ? 'Active' : 'Inactive';

// ✅ GOOD: Translate in template
@if (job.isActive) {
  {{ 'jobs.status.active' | translate }}
} @else {
  {{ 'jobs.status.inactive' | translate }}
}

// ❌ BAD: Hardcoded date formats
const date = new Date().toLocaleDateString('en-US');

// ✅ GOOD: Use Angular date pipe with locale
{{ date | date: 'short' }}
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
