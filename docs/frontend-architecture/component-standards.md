[Back to Index](./index.md)

# Component Standards

## Component Template

This is the standard template for Angular 20+ components in the FinDogAI application.

```typescript
import { Component, OnInit, OnDestroy, inject, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonicModule } from '@ionic/angular';
import { Store } from '@ngrx/store';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

// Feature-specific imports
import { JobsState } from '../data-access/jobs.state';
import { JobsActions } from '../data-access/jobs.actions';
import { selectActiveJob } from '../data-access/jobs.selectors';

@Component({
  selector: 'app-job-detail',
  standalone: true,
  imports: [CommonModule, IonicModule],
  template: `
    <ion-header>
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-back-button></ion-back-button>
        </ion-buttons>
        <ion-title>{{ job()?.title || 'Job Detail' }}</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <!-- Component content -->
    </ion-content>
  `,
  styleUrl: './job-detail.component.scss'
})
export class JobDetailComponent implements OnInit, OnDestroy {
  // Dependency injection using inject()
  private readonly store = inject(Store<JobsState>);

  // Signals for reactive state
  protected readonly job = signal<Job | null>(null);
  protected readonly loading = signal(false);
  protected readonly error = signal<string | null>(null);

  // Computed signals for derived state
  protected readonly jobStatus = computed(() =>
    this.job()?.status || 'unknown'
  );

  // Store subscriptions with auto-cleanup
  private readonly job$ = this.store.select(selectActiveJob).pipe(
    takeUntilDestroyed()
  ).subscribe(job => this.job.set(job));

  ngOnInit(): void {
    this.loadJobData();
  }

  ngOnDestroy(): void {
    // takeUntilDestroyed handles subscription cleanup
    // Add any other cleanup logic here
  }

  private loadJobData(): void {
    this.loading.set(true);
    this.store.dispatch(JobsActions.loadJobDetail({ id: this.jobId }));
  }
}
```

## Component Architecture Patterns

### 1. Standalone Components
All components must be standalone (no NgModules):

```typescript
@Component({
  selector: 'app-example',
  standalone: true,
  imports: [CommonModule, IonicModule, ReactiveFormsModule],
  // ...
})
export class ExampleComponent { }
```

### 2. Dependency Injection with inject()
Use the `inject()` function instead of constructor injection:

```typescript
export class ExampleComponent {
  // ✅ Preferred
  private readonly store = inject(Store);
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);

  // ❌ Avoid (old style)
  constructor(
    private store: Store,
    private router: Router
  ) {}
}
```

### 3. Signals for Reactive State
Use Angular signals for component state:

```typescript
export class ExampleComponent {
  // Simple signals
  protected readonly count = signal(0);
  protected readonly loading = signal(false);
  protected readonly user = signal<User | null>(null);

  // Computed signals
  protected readonly isLoggedIn = computed(() => !!this.user());
  protected readonly fullName = computed(() => {
    const user = this.user();
    return user ? `${user.firstName} ${user.lastName}` : '';
  });

  // Signal methods
  increment(): void {
    this.count.update(value => value + 1);
  }

  setUser(user: User): void {
    this.user.set(user);
  }
}
```

### 4. Automatic Subscription Cleanup
Use `takeUntilDestroyed()` for automatic cleanup:

```typescript
export class ExampleComponent {
  private readonly data$ = this.dataService.getData().pipe(
    takeUntilDestroyed()
  ).subscribe(data => {
    this.dataSignal.set(data);
  });
}
```

### 5. Template Syntax
Use Angular 20+ control flow syntax:

```typescript
template: `
  <!-- Conditional rendering -->
  @if (loading()) {
    <ion-spinner></ion-spinner>
  } @else if (error()) {
    <div class="error">{{ error() }}</div>
  } @else {
    <div class="content">{{ data() }}</div>
  }

  <!-- Loops -->
  @for (item of items(); track item.id) {
    <app-item-card [item]="item"></app-item-card>
  } @empty {
    <div class="empty-state">No items found</div>
  }

  <!-- Switch statements -->
  @switch (status()) {
    @case ('draft') {
      <ion-badge color="medium">Draft</ion-badge>
    }
    @case ('active') {
      <ion-badge color="primary">Active</ion-badge>
    }
    @case ('completed') {
      <ion-badge color="success">Completed</ion-badge>
    }
    @default {
      <ion-badge>Unknown</ion-badge>
    }
  }
`
```

## Naming Conventions

### File Names

| Type | Pattern | Example |
|------|---------|---------|
| Component | `name.component.ts` | `job-detail.component.ts` |
| Component Template | `name.component.html` | `job-detail.component.html` |
| Component Styles | `name.component.scss` | `job-detail.component.scss` |
| Component Spec | `name.component.spec.ts` | `job-detail.component.spec.ts` |
| Service | `name.service.ts` | `data.service.ts` |
| Guard | `name.guard.ts` | `auth.guard.ts` |
| Pipe | `name.pipe.ts` | `format-date.pipe.ts` |
| Directive | `name.directive.ts` | `highlight.directive.ts` |
| Model | `name.model.ts` | `user.model.ts` |

### Class Names

```typescript
// Components (PascalCase + Component suffix)
export class JobDetailComponent { }
export class JobCardComponent { }

// Services (PascalCase + Service suffix)
export class AuthService { }
export class DataService { }

// Guards (PascalCase + Guard suffix)
export class AuthGuard { }
export class RoleGuard { }

// Pipes (PascalCase + Pipe suffix)
export class FormatDatePipe { }
export class CurrencyFormatPipe { }

// Directives (PascalCase + Directive suffix)
export class HighlightDirective { }
export class AutoFocusDirective { }
```

### Component Selectors

Use `app-` prefix for application components:

```typescript
@Component({
  selector: 'app-job-detail',  // ✅ Correct
  // ...
})
```

Avoid:
```typescript
@Component({
  selector: 'job-detail',      // ❌ No prefix
  selector: 'JobDetail',       // ❌ PascalCase
  // ...
})
```

## Component Types

### 1. Page Components
Routable components that represent full pages:

```typescript
// features/jobs/pages/job-list/job-list.page.ts
@Component({
  selector: 'app-job-list-page',
  standalone: true,
  imports: [CommonModule, IonicModule],
  templateUrl: './job-list.page.html',
  styleUrl: './job-list.page.scss'
})
export class JobListPage { }
```

### 2. Container Components (Smart)
Components that connect to state and pass data to presentational components:

```typescript
@Component({
  selector: 'app-job-list-container',
  standalone: true,
  imports: [CommonModule, JobCardComponent],
  template: `
    @for (job of jobs(); track job.id) {
      <app-job-card
        [job]="job"
        (selected)="onJobSelected($event)"
      ></app-job-card>
    }
  `
})
export class JobListContainerComponent {
  private readonly store = inject(Store);
  protected readonly jobs = signal<Job[]>([]);

  constructor() {
    this.store.select(selectAllJobs).pipe(
      takeUntilDestroyed()
    ).subscribe(jobs => this.jobs.set(jobs));
  }

  protected onJobSelected(job: Job): void {
    this.store.dispatch(JobsActions.selectJob({ id: job.id }));
  }
}
```

### 3. Presentational Components (Dumb)
Pure components that receive data via inputs and emit events:

```typescript
@Component({
  selector: 'app-job-card',
  standalone: true,
  imports: [CommonModule, IonicModule],
  template: `
    <ion-card (click)="selected.emit(job)">
      <ion-card-header>
        <ion-card-title>{{ job.title }}</ion-card-title>
        <ion-card-subtitle>Job #{{ job.jobNumber }}</ion-card-subtitle>
      </ion-card-header>
      <ion-card-content>
        <p>Status: {{ job.status }}</p>
        <p>Budget: {{ job.budget | currency }}</p>
      </ion-card-content>
    </ion-card>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class JobCardComponent {
  @Input({ required: true }) job!: Job;
  @Output() selected = new EventEmitter<Job>();
}
```

## Input and Output Patterns

### Required Inputs
Use `required: true` for mandatory inputs:

```typescript
export class JobCardComponent {
  @Input({ required: true }) job!: Job;
  @Input({ required: true }) tenantId!: string;

  // Optional inputs with defaults
  @Input() showActions = true;
  @Input() compact = false;
}
```

### Type-Safe Outputs
Use strongly-typed EventEmitters:

```typescript
export class JobCardComponent {
  @Output() selected = new EventEmitter<Job>();
  @Output() deleted = new EventEmitter<string>(); // job ID
  @Output() statusChanged = new EventEmitter<{ job: Job; status: JobStatus }>();
}
```

### Signal Inputs (Angular 20+)
Use signal inputs for better reactivity:

```typescript
export class JobCardComponent {
  // Signal input
  job = input.required<Job>();

  // Computed from signal input
  protected readonly isActive = computed(() =>
    this.job().status === 'active'
  );
}
```

## Component Communication

### 1. Parent to Child (Input)
```typescript
// Parent
<app-job-card [job]="selectedJob()"></app-job-card>

// Child
@Input({ required: true }) job!: Job;
```

### 2. Child to Parent (Output)
```typescript
// Parent
<app-job-card
  [job]="job()"
  (selected)="onJobSelected($event)"
></app-job-card>

// Child
@Output() selected = new EventEmitter<Job>();

handleClick(): void {
  this.selected.emit(this.job);
}
```

### 3. Sibling Communication (via Service/Store)
```typescript
// Component A
protected saveJob(job: Job): void {
  this.store.dispatch(JobsActions.updateJob({ id: job.id, changes: job }));
}

// Component B
constructor() {
  this.store.select(selectActiveJob).pipe(
    takeUntilDestroyed()
  ).subscribe(job => this.job.set(job));
}
```

## Lifecycle Hooks

### Recommended Hooks
- `ngOnInit()`: Initialization logic, fetch data
- `ngOnDestroy()`: Cleanup (though `takeUntilDestroyed` handles most cases)

### Avoid When Possible
- `ngAfterViewInit()`: Use signals instead
- `ngOnChanges()`: Use signal inputs instead
- `ngDoCheck()`: Performance impact

## Component Best Practices

1. **Single Responsibility**: Each component has one clear purpose
2. **Small and Focused**: < 300 lines per component
3. **OnPush Detection**: Use for presentational components
4. **No Business Logic**: Move to services or effects
5. **Type Safety**: Strong typing for all inputs/outputs
6. **Accessibility**: Proper ARIA labels and keyboard support
7. **Responsive Design**: Mobile-first approach
8. **Error Handling**: Display user-friendly error messages
9. **Loading States**: Show spinners for async operations
10. **Empty States**: Handle no-data scenarios gracefully

## Single Resource Auto-Selection Pattern

When displaying resource selection controls (vehicles, team members, machines), implement auto-selection for single-resource scenarios:

### Pattern Overview

```typescript
@Component({
  selector: 'app-transport-cost-form',
  template: `
    <!-- Single resource: show as read-only text -->
    @if (vehicles().length === 1) {
      <div class="resource-info">
        <ion-label>Vehicle</ion-label>
        <p class="readonly-value">[{{ vehicles()[0].vehicleNumber }}] {{ vehicles()[0].name }}</p>
      </div>
    }

    <!-- Multiple resources: show dropdown -->
    @else if (vehicles().length > 1) {
      <ion-item>
        <ion-label>Vehicle</ion-label>
        <ion-select [(ngModel)]="selectedVehicleId" interface="popover">
          @for (vehicle of vehicles(); track vehicle.id) {
            <ion-select-option [value]="vehicle.id">
              [{{ vehicle.vehicleNumber }}] {{ vehicle.name }}
            </ion-select-option>
          }
        </ion-select>
      </ion-item>
    }

    <!-- No resources: show warning -->
    @else {
      <ion-item color="warning">
        <ion-label>No vehicles available. Add one in Settings.</ion-label>
      </ion-item>
    }
  `
})
export class TransportCostFormComponent {
  protected readonly vehicles = signal<Vehicle[]>([]);
  protected readonly selectedVehicleId = signal<string | null>(null);

  // Computed: auto-select if single resource
  protected readonly selectedVehicle = computed(() => {
    const vehicleList = this.vehicles();
    if (vehicleList.length === 1) {
      return vehicleList[0]; // Auto-select single vehicle
    }
    return vehicleList.find(v => v.id === this.selectedVehicleId()) || null;
  });

  ngOnInit(): void {
    // Auto-select when vehicles load
    this.vehiclesService.getVehicles(this.tenantId).subscribe(vehicles => {
      this.vehicles.set(vehicles);
      if (vehicles.length === 1) {
        this.selectedVehicleId.set(vehicles[0].id);
      }
    });
  }
}
```

### Usage Guidelines

1. **Check Resource Count**: Always check if `resources.length === 1` before rendering UI
2. **Auto-Select on Load**: Set selected ID automatically when single resource detected
3. **Display as Read-Only**: Show resource info as read-only text (not disabled dropdown)
4. **Consistent Format**: Use `[resourceNumber] Name` format for all resources
5. **Apply to All Resource Types**: Use for vehicles, team members, and machines
6. **Edit Forms**: Apply same logic when editing costs (single → read-only, multiple → dropdown)
7. **Empty State**: If zero resources, show helpful message with link to settings

### Complete Example

```typescript
export class LaborCostFormComponent {
  private readonly teamMembersService = inject(TeamMembersService);

  protected readonly teamMembers = signal<TeamMember[]>([]);
  protected readonly selectedMemberId = signal<string | null>(null);

  // Computed properties
  protected readonly hasSingleMember = computed(() =>
    this.teamMembers().length === 1
  );

  protected readonly selectedMember = computed(() => {
    const members = this.teamMembers();
    if (members.length === 1) {
      return members[0];
    }
    return members.find(m => m.id === this.selectedMemberId()) || null;
  });

  ngOnInit(): void {
    this.teamMembersService.getTeamMembers(this.tenantId).pipe(
      takeUntilDestroyed()
    ).subscribe(members => {
      this.teamMembers.set(members);
      // Auto-select if single member
      if (members.length === 1) {
        this.selectedMemberId.set(members[0].id);
      }
    });
  }

  protected onSave(): void {
    const member = this.selectedMember();
    if (!member) {
      this.showError('Please select a team member');
      return;
    }

    const cost: Partial<LaborCost> = {
      category: 'labor',
      teamMember: {
        teamMemberNumber: member.teamMemberNumber,
        name: member.name,
        hourlyRate: member.hourlyRate
      },
      hours: this.hoursControl.value,
      amount: this.hoursControl.value * member.hourlyRate,
      // ... other fields
    };

    this.saveCost(cost);
  }
}
```

## Role-Based UI Visibility Pattern

Implement privilege-based feature visibility based on user role (FR11):

### Pattern Overview

```typescript
@Component({
  selector: 'app-team-members',
  template: `
    <!-- Owner-only: Invite Team Member button -->
    @if (userRole() === 'owner') {
      <ion-button (click)="openInviteDialog()">
        <ion-icon slot="start" name="person-add"></ion-icon>
        Add Team Member
      </ion-button>
    }

    <!-- All members can view team list -->
    <ion-list>
      @for (member of teamMembers(); track member.id) {
        <ion-item>
          <ion-label>
            <h2>[{{ member.teamMemberNumber }}] {{ member.name }}</h2>
            <p>{{ member.role | titleCase }}</p>
          </ion-label>
        </ion-item>
      }
    </ion-list>
  `
})
export class TeamMembersComponent {
  private readonly store = inject(Store);

  protected readonly userRole = signal<string>('');
  protected readonly teamMembers = signal<TeamMember[]>([]);

  constructor() {
    // Subscribe to user role from auth state
    this.store.select(selectUserRole).pipe(
      takeUntilDestroyed()
    ).subscribe(role => this.userRole.set(role));

    // Load team members
    this.store.select(selectAllTeamMembers).pipe(
      takeUntilDestroyed()
    ).subscribe(members => this.teamMembers.set(members));
  }

  protected openInviteDialog(): void {
    // Only callable if button is visible (owner-only)
    this.modalCtrl.create({
      component: InviteTeamMemberModal,
      // ...
    }).then(modal => modal.present());
  }
}
```

### Role Permission Guidelines

Based on FR11, implement these UI visibility rules:

| Feature | Owner | Representative | Team Member |
|---------|-------|----------------|-------------|
| Invite Team Members | ✅ Visible | ❌ Hidden | ❌ Hidden |
| View Audit Logs | ✅ Visible | ❌ Hidden | ❌ Hidden |
| Export Data | ✅ Visible | ❌ Hidden | ❌ Hidden |
| Edit Business Profile | ✅ Enabled | ❌ Disabled | ❌ Hidden |
| Create/Edit Jobs | ✅ Enabled | ❌ Read-only | ❌ Hidden |
| Add/Edit Costs | ✅ Enabled | ✅ Enabled | ✅ Enabled* |
| View Job Budget | ✅ Visible | ✅ Visible | ❌ Hidden |
| Manage Advances | ✅ Enabled | ✅ Enabled | ❌ Hidden |

*Active status required

### Implementation Pattern

```typescript
export class SomeFeatureComponent {
  protected readonly userRole = signal<string>('');
  protected readonly isOwner = computed(() => this.userRole() === 'owner');
  protected readonly isOwnerOrRep = computed(() =>
    ['owner', 'representative'].includes(this.userRole())
  );

  constructor() {
    this.store.select(selectUserRole).pipe(
      takeUntilDestroyed()
    ).subscribe(role => this.userRole.set(role));
  }
}
```

### Template Examples

```typescript
// Owner-only button
@if (isOwner()) {
  <ion-button (click)="inviteTeamMember()">Invite Team Member</ion-button>
}

// Owner or Representative
@if (isOwnerOrRep()) {
  <ion-button (click)="createJob()">Create Job</ion-button>
}

// Disabled for non-owners with tooltip
<ion-button
  [disabled]="!isOwner()"
  (click)="editBusinessProfile()">
  Edit Profile
</ion-button>
@if (!isOwner()) {
  <p class="hint">Only the business owner can edit the profile</p>
}
```

## Internationalization in Components

All components MUST use translation keys for user-facing text. Never hardcode strings in templates or component logic.

### Import TranslateModule

```typescript
@Component({
  selector: 'app-example',
  standalone: true,
  imports: [CommonModule, IonicModule, TranslateModule], // Always include TranslateModule
  template: `...`
})
export class ExampleComponent { }
```

### Use TranslatePipe in Templates

```typescript
@Component({
  selector: 'app-job-list',
  template: `
    <!-- ✅ GOOD: Use translate pipe -->
    <ion-header>
      <ion-toolbar>
        <ion-title>{{ 'jobs.list.title' | translate }}</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-button>
      <ion-icon slot="start" name="add"></ion-icon>
      {{ 'jobs.list.createButton' | translate }}
    </ion-button>

    <!-- With parameters -->
    <h2>{{ 'jobs.detail.jobNumber' | translate: {number: job().jobNumber} }}</h2>

    <!-- Status badges with translation -->
    @switch (job().status) {
      @case ('draft') {
        <ion-badge>{{ 'jobs.status.draft' | translate }}</ion-badge>
      }
      @case ('active') {
        <ion-badge color="primary">{{ 'jobs.status.active' | translate }}</ion-badge>
      }
      @case ('completed') {
        <ion-badge color="success">{{ 'jobs.status.completed' | translate }}</ion-badge>
      }
    }

    <!-- ❌ BAD: Hardcoded strings -->
    <ion-title>Job List</ion-title>
    <ion-button>Create Job</ion-button>
  `
})
```

### Use TranslateService in Component Logic

```typescript
export class JobFormComponent {
  private readonly translate = inject(TranslateService);
  private readonly toastController = inject(ToastController);

  async saveJob(): Promise<void> {
    try {
      await this.jobsService.save(this.job);

      // ✅ GOOD: Translate toast messages
      const message = this.translate.instant('jobs.form.saveSuccess');
      await this.showToast(message, 'success');
    } catch (error) {
      // ✅ GOOD: Translate error messages
      const message = this.translate.instant('jobs.form.errorSaveFailed');
      await this.showToast(message, 'danger');
    }
  }

  async showConfirmDialog(): Promise<boolean> {
    const alert = await this.alertController.create({
      header: this.translate.instant('jobs.delete.confirmTitle'),
      message: this.translate.instant('jobs.delete.confirmMessage'),
      buttons: [
        {
          text: this.translate.instant('common.buttons.cancel'),
          role: 'cancel'
        },
        {
          text: this.translate.instant('common.buttons.delete'),
          role: 'destructive'
        }
      ]
    });

    await alert.present();
    const { role } = await alert.onDidDismiss();
    return role === 'destructive';
  }
}
```

### Language Selector Integration

```typescript
@Component({
  selector: 'app-settings',
  template: `
    <ion-list>
      <ion-item>
        <ion-label>{{ 'settings.language' | translate }}</ion-label>
        <app-language-selector></app-language-selector>
      </ion-item>
    </ion-list>
  `
})
export class SettingsComponent {
  // Language selector handles language switching
}
```

### Empty States and Placeholders

```typescript
template: `
  @if (jobs().length === 0) {
    <div class="empty-state">
      <ion-icon name="briefcase-outline"></ion-icon>
      <h3>{{ 'jobs.list.emptyTitle' | translate }}</h3>
      <p>{{ 'jobs.list.emptyMessage' | translate }}</p>
      <ion-button (click)="createJob()">
        {{ 'jobs.list.createFirstJob' | translate }}
      </ion-button>
    </div>
  }
`
```

### Form Labels and Validation Messages

```typescript
template: `
  <form [formGroup]="jobForm">
    <ion-item>
      <ion-label position="floating">
        {{ 'jobs.form.titleLabel' | translate }}
      </ion-label>
      <ion-input formControlName="title" type="text"></ion-input>
    </ion-item>

    @if (jobForm.get('title')?.hasError('required') && jobForm.get('title')?.touched) {
      <div class="error-message">
        {{ 'jobs.form.errorTitleRequired' | translate }}
      </div>
    }

    <ion-item>
      <ion-label position="floating">
        {{ 'jobs.form.budgetLabel' | translate }}
      </ion-label>
      <ion-input formControlName="budget" type="number"></ion-input>
    </ion-item>
  </form>
`
```

## Component Checklist

Before committing a component, ensure:

- [ ] Standalone component with all imports
- [ ] Uses `inject()` for dependencies
- [ ] Signals for local state
- [ ] `takeUntilDestroyed()` for subscriptions
- [ ] Type-safe inputs and outputs
- [ ] Angular 20+ control flow syntax
- [ ] **TranslateModule imported and all UI text uses translation keys**
- [ ] **No hardcoded strings in templates or component logic**
- [ ] Proper error handling
- [ ] Loading and empty states
- [ ] Accessible (ARIA labels, keyboard nav)
- [ ] Touch-friendly (48px+ targets)
- [ ] Role-based UI visibility implemented (if applicable)
- [ ] Unit tests written
- [ ] Documentation comments

---

[← Previous: Project Structure](./project-structure.md) | [Back to Index](./index.md) | [Next: State Management →](./state-management.md)
