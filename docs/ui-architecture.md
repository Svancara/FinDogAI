# FinDogAI Frontend Architecture Document

## Change Log

| Date | Version | Description | Author |
|------|---------|-------------|---------|
| 2025-01-27 | 1.0 | Initial frontend architecture document | Winston (Architect) |

## Frontend Tech Stack

### Technology Stack Table

| Category | Technology | Version | Purpose | Rationale |
|----------|------------|---------|---------|-----------|
| Framework | **Angular** | **20+** | Core application framework | Latest Angular with signals, improved performance, and enhanced developer experience |
| UI Library | Ionic Framework | 8+ | Mobile UI components & PWA | Native-feel components, PWA support critical for project, platform-specific styling |
| State Management | **NgRx** | 18+ | Application state management | Enterprise-grade state management with devtools, perfect for complex voice pipeline flows |
| Routing | Angular Router + Ionic Navigation | 20+ / 8+ | Navigation & deep linking | Built-in lazy loading, guards, PWA routing support |
| Build Tool | Angular CLI + **Nx** + Vite | 20+ / 19+ / 5+ | Monorepo & development | Nx for monorepo management, Vite for fast HMR, Angular schematics |
| Styling | Ionic CSS Variables + SCSS | - | Theming & styles | Platform-adaptive theming, CSS custom properties for runtime theming |
| Testing | Jest + Playwright | 29+ / 1.40+ | Unit & E2E testing | Jest for fast unit tests, Playwright for cross-platform E2E and PWA testing |
| Component Library | Ionic Components | 8+ | UI component system | Pre-built accessible components optimized for touch, mobile, and PWA |
| Form Handling | Angular Reactive Forms | 20+ | Form management | Built-in validation, type-safe forms with signals integration |
| Animation | Angular Animations + Ionic | 20+ | UI animations | Hardware-accelerated animations, gesture-based interactions |
| Dev Tools | Angular DevTools + Redux DevTools | Latest | Development debugging | Component tree inspection, NgRx state debugging, performance profiling |
| PWA | @angular/pwa + Workbox | 20+ / 7+ | Progressive Web App | Offline support, installability, push notifications, app-like experience |

### Additional Frontend Dependencies

**Firebase/Firestore Integration:**
- @angular/fire (v17+) - Angular Firebase integration
- firebase (v10+) - Core Firebase SDK with offline persistence

**Voice Pipeline (Client-side):**
- Web Speech API or Capacitor Speech Recognition plugin
- OpenAI/Anthropic SDK for LLM integration
- Web Speech Synthesis API or native TTS via Capacitor

**Mobile/Native:**
- Capacitor (v5+) - Native runtime and plugins
- Capacitor plugins for device features (filesystem, camera, etc.)

**Monorepo Tooling:**
- Nx for monorepo management
- Shared TypeScript types package (`/packages/shared-types`)

## Project Structure

```plaintext
findogai/                                 # Monorepo root
├── apps/
│   ├── mobile-app/                      # Main Angular/Ionic PWA application
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/               # Singleton services, guards, interceptors
│   │   │   │   │   ├── services/
│   │   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   │   ├── firestore.service.ts
│   │   │   │   │   │   └── voice-pipeline.service.ts
│   │   │   │   │   ├── guards/
│   │   │   │   │   ├── interceptors/
│   │   │   │   │   └── models/
│   │   │   │   ├── features/           # Lazy-loaded feature modules
│   │   │   │   │   ├── jobs/
│   │   │   │   │   │   ├── data-access/    # NgRx state for jobs
│   │   │   │   │   │   ├── pages/
│   │   │   │   │   │   └── components/
│   │   │   │   │   ├── voice/
│   │   │   │   │   │   ├── data-access/    # NgRx state for voice
│   │   │   │   │   │   ├── pages/
│   │   │   │   │   │   └── components/
│   │   │   │   │   ├── costs/
│   │   │   │   │   ├── team/
│   │   │   │   │   └── settings/
│   │   │   │   ├── shared/             # Shared components, directives, pipes
│   │   │   │   │   ├── ui/
│   │   │   │   │   ├── directives/
│   │   │   │   │   └── pipes/
│   │   │   │   └── store/              # Root NgRx store configuration
│   │   │   │       ├── app.state.ts
│   │   │   │       └── app.effects.ts
│   │   │   ├── assets/
│   │   │   ├── environments/
│   │   │   ├── theme/                  # Ionic theme variables
│   │   │   ├── index.html
│   │   │   ├── main.ts
│   │   │   ├── manifest.json           # PWA manifest
│   │   │   └── ngsw-config.json        # Angular service worker config
│   │   ├── capacitor.config.ts
│   │   ├── ionic.config.json
│   │   └── project.json                # Nx project configuration
│   │
│   └── mobile-app-e2e/                 # Playwright E2E tests
│       ├── src/
│       │   ├── fixtures/
│       │   ├── support/
│       │   └── specs/
│       └── project.json
│
├── libs/                                # Shared libraries (Nx convention)
│   ├── shared/
│   │   ├── types/                      # TypeScript interfaces/types
│   │   │   └── src/
│   │   │       ├── lib/
│   │   │       │   ├── models/
│   │   │       │   ├── api/
│   │   │       │   └── firestore/
│   │   │       └── index.ts
│   │   └── utils/                      # Shared utilities
│   │       └── src/
│   │           └── lib/
│   │
│   └── mobile/                         # Mobile-specific shared code
│       ├── ui/                         # Shared Ionic components
│       ├── data-access/                # Shared state/services
│       └── voice/                      # Voice pipeline utilities
│
├── tools/                               # Nx workspace tools
├── .bmad-core/                          # BMAD configuration
├── docs/
│   ├── prd/
│   ├── ui-architecture.md              # This document
│   └── architecture/
├── nx.json                             # Nx configuration
├── package.json
├── tsconfig.base.json                  # Base TypeScript config
└── angular.json                        # Angular workspace config
```

## Component Standards

### Component Template

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

### Naming Conventions

**Files and Folders:**
- `feature-name.component.ts` - Components
- `data.service.ts` - Services
- `auth.guard.ts` - Guards
- `format-date.pipe.ts` - Pipes
- `highlight.directive.ts` - Directives
- `user.model.ts` - Models/Interfaces
- `jobs.actions.ts` - NgRx actions
- `jobs.reducer.ts` - NgRx reducers
- `jobs.effects.ts` - NgRx effects
- `jobs.selectors.ts` - NgRx selectors

### Additional Offline-First Component Patterns

#### Network Status Service
```typescript
// libs/mobile/data-access/src/lib/services/network.service.ts
import { Injectable, inject } from '@angular/core';
import { fromEvent, merge, of, Observable } from 'rxjs';
import { map, startWith, distinctUntilChanged, shareReplay } from 'rxjs/operators';
import { toSignal } from '@angular/core/rxjs-interop';

@Injectable({ providedIn: 'root' })
export class NetworkService {
  public readonly online$: Observable<boolean> = merge(
    of(navigator.onLine),
    fromEvent(window, 'online').pipe(map(() => true)),
    fromEvent(window, 'offline').pipe(map(() => false))
  ).pipe(
    startWith(navigator.onLine),
    distinctUntilChanged(),
    shareReplay({ bufferSize: 1, refCount: true })
  );

  public readonly isOnline = toSignal(this.online$, {
    initialValue: navigator.onLine
  });

  public getConnectionType(): 'wifi' | 'cellular' | 'slow-2g' | 'offline' | 'unknown' {
    if (!navigator.onLine) return 'offline';

    const connection = (navigator as any).connection;
    if (!connection) return 'unknown';

    return connection.effectiveType || 'unknown';
  }
}
```

#### Sync Queue Service (Complete Implementation)
```typescript
// libs/mobile/data-access/src/lib/services/sync-queue.service.ts
import { Injectable, inject } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { map, filter, debounceTime } from 'rxjs/operators';
import { Store } from '@ngrx/store';
import { NetworkService } from './network.service';

export interface SyncOperation {
  id?: string;
  type: string;
  payload: any;
  timestamp: number;
  retryCount?: number;
  maxRetries?: number;
}

@Injectable({ providedIn: 'root' })
export class SyncQueueService {
  private readonly store = inject(Store);
  private readonly network = inject(NetworkService);

  private readonly queue = new BehaviorSubject<SyncOperation[]>([]);
  public readonly pendingCount$ = this.queue.pipe(
    map(ops => ops.length)
  );

  constructor() {
    // Process queue when coming back online
    this.network.online$.pipe(
      filter(online => online),
      debounceTime(1000) // Wait a second for stable connection
    ).subscribe(() => {
      this.processPendingOperations();
    });

    // Load persisted queue from localStorage
    this.loadQueueFromStorage();
  }

  async addOperation(operation: SyncOperation): Promise<void> {
    const ops = this.queue.value;
    const newOp = {
      ...operation,
      id: crypto.randomUUID(),
      retryCount: 0,
      maxRetries: operation.maxRetries || 3
    };

    this.queue.next([...ops, newOp]);
    this.persistQueueToStorage();

    // Try to process immediately if online
    if (this.network.isOnline()) {
      await this.processPendingOperations();
    }
  }

  async processPendingOperations(): Promise<void> {
    const operations = this.queue.value;
    if (operations.length === 0) return;

    for (const op of operations) {
      try {
        await this.processOperation(op);
        this.removeOperation(op.id!);
      } catch (error) {
        console.error(`Failed to sync operation ${op.id}:`, error);
        this.handleOperationError(op);
      }
    }
  }

  private async processOperation(op: SyncOperation): Promise<void> {
    // Dispatch appropriate actions based on operation type
    switch (op.type) {
      case 'UPDATE_JOB':
        this.store.dispatch(JobsActions.syncOfflineUpdate(op.payload));
        break;
      case 'CREATE_COST':
        this.store.dispatch(CostsActions.syncOfflineCreate(op.payload));
        break;
      case 'CREATE_ADVANCE':
        this.store.dispatch(AdvancesActions.syncOfflineCreate(op.payload));
        break;
      case 'START_JOURNEY':
        this.store.dispatch(JourneysActions.syncOfflineStart(op.payload));
        break;
      default:
        console.warn(`Unknown operation type: ${op.type}`);
    }
  }

  private handleOperationError(op: SyncOperation): void {
    op.retryCount = (op.retryCount || 0) + 1;

    if (op.retryCount >= (op.maxRetries || 3)) {
      // Move to dead letter queue or notify user
      this.removeOperation(op.id!);
      console.error(`Operation ${op.id} exceeded max retries`);
    } else {
      // Keep in queue for retry
      this.persistQueueToStorage();
    }
  }

  private removeOperation(id: string): void {
    const ops = this.queue.value.filter(op => op.id !== id);
    this.queue.next(ops);
    this.persistQueueToStorage();
  }

  private persistQueueToStorage(): void {
    localStorage.setItem('sync-queue', JSON.stringify(this.queue.value));
  }

  private loadQueueFromStorage(): void {
    const stored = localStorage.getItem('sync-queue');
    if (stored) {
      try {
        this.queue.next(JSON.parse(stored));
      } catch (e) {
        console.error('Failed to load sync queue from storage');
      }
    }
  }
}
```

#### Firestore Sync Service with Conflict Resolution
```typescript
// libs/mobile/data-access/src/lib/services/firestore-sync.service.ts
import { Injectable, inject } from '@angular/core';
import {
  Firestore,
  doc,
  onSnapshot,
  setDoc,
  serverTimestamp,
  runTransaction
} from '@angular/fire/firestore';

@Injectable({ providedIn: 'root' })
export class FirestoreSyncService {
  private readonly firestore = inject(Firestore);

  /**
   * Firestore Default Conflict Resolution Strategy:
   * - Last Write Wins (LWW) based on server timestamp
   * - Optimistic concurrency control via transactions for critical operations
   * - Automatic retry with exponential backoff for transient failures
   */

  async updateDocument(path: string, data: any): Promise<void> {
    const docRef = doc(this.firestore, path);

    // Firestore automatically handles conflicts with last-write-wins
    // Server timestamp ensures consistent ordering
    await setDoc(docRef, {
      ...data,
      updatedAt: serverTimestamp(), // Server timestamp for ordering
      // Firestore's offline persistence handles the queue
      // When online, changes are synchronized automatically
      // Conflicts resolved by server timestamp (last write wins)
    }, { merge: true });
  }

  // For operations requiring strict consistency, use transactions
  async updateWithTransaction(path: string, updateFn: (data: any) => any): Promise<void> {
    const docRef = doc(this.firestore, path);

    return runTransaction(this.firestore, async (transaction) => {
      const doc = await transaction.get(docRef);
      if (!doc.exists()) {
        throw new Error('Document does not exist');
      }

      const newData = updateFn(doc.data());
      transaction.update(docRef, {
        ...newData,
        updatedAt: serverTimestamp()
      });
    });
  }

  // Listen for remote changes to handle conflicts reactively
  subscribeToDocument(path: string, callback: (data: any) => void): () => void {
    const docRef = doc(this.firestore, path);

    return onSnapshot(docRef, {
      includeMetadataChanges: true // Get notified of local vs server changes
    }, (snapshot) => {
      const source = snapshot.metadata.hasPendingWrites ? 'local' : 'server';

      if (snapshot.exists()) {
        callback({
          data: snapshot.data(),
          source,
          fromCache: snapshot.metadata.fromCache
        });
      }
    });
  }
}
```

## State Management

### Complete NgRx State Implementation

```typescript
// jobs.state.ts - State interface
export interface Job {
  id: string;
  tenantId: string;
  jobNumber: number;
  title: string;
  status: 'draft' | 'active' | 'completed' | 'cancelled';
  budget?: number;
  vatRate: number;
  currency: string;
  createdAt: Timestamp;
  updatedAt: Timestamp;
  _syncStatus?: 'synced' | 'pending' | 'error';
  _localChanges?: boolean;
}

export interface JobsState {
  entities: Record<string, Job>;
  ids: string[];
  selectedId: string | null;
  loading: boolean;
  error: string | null;
  filter: {
    status: string | null;
    searchTerm: string;
  };
  syncStatus: {
    lastSync: number | null;
    pendingChanges: string[]; // IDs of jobs with pending changes
  };
}

// jobs.actions.ts - NgRx actions with offline support
import { createActionGroup, props, emptyProps } from '@ngrx/store';

export const JobsActions = createActionGroup({
  source: 'Jobs',
  events: {
    // Load actions
    'Load Jobs': emptyProps(),
    'Load Jobs Success': props<{ jobs: Job[] }>(),
    'Load Jobs Failure': props<{ error: string }>(),

    // CRUD with offline support
    'Create Job': props<{ job: Omit<Job, 'id'> }>(),
    'Create Job Offline': props<{ job: Job; tempId: string }>(),
    'Create Job Success': props<{ job: Job; tempId?: string }>(),
    'Create Job Failure': props<{ error: string; tempId?: string }>(),

    'Update Job': props<{ id: string; changes: Partial<Job> }>(),
    'Update Job Optimistic': props<{ id: string; changes: Partial<Job> }>(),
    'Update Job Success': props<{ id: string; changes: Partial<Job> }>(),
    'Update Job Failure': props<{ id: string; error: string; previousState?: Job }>(),

    'Delete Job': props<{ id: string }>(),
    'Delete Job Success': props<{ id: string }>(),
    'Delete Job Failure': props<{ id: string; error: string }>(),

    // Sync actions
    'Sync Offline Changes': emptyProps(),
    'Sync Offline Update': props<{ payload: any }>(),
    'Sync Job': props<{ id: string }>(),
    'Sync Success': props<{ id: string }>(),
    'Sync Failure': props<{ id: string; error: string }>(),

    // Selection
    'Select Job': props<{ id: string }>(),
    'Clear Selection': emptyProps(),

    // Filtering
    'Set Filter': props<{ filter: Partial<JobsState['filter']> }>(),
    'Clear Filter': emptyProps(),
  }
});

// jobs.reducer.ts - Reducer with offline support
import { createReducer, on } from '@ngrx/store';
import { createEntityAdapter, EntityAdapter } from '@ngrx/entity';

export const jobsAdapter: EntityAdapter<Job> = createEntityAdapter<Job>({
  selectId: (job) => job.id,
  sortComparer: (a, b) => b.updatedAt.seconds - a.updatedAt.seconds
});

export const initialJobsState: JobsState = jobsAdapter.getInitialState({
  selectedId: null,
  loading: false,
  error: null,
  filter: {
    status: null,
    searchTerm: ''
  },
  syncStatus: {
    lastSync: null,
    pendingChanges: []
  }
});

export const jobsReducer = createReducer(
  initialJobsState,

  // Load operations
  on(JobsActions.loadJobs, (state) => ({
    ...state,
    loading: true,
    error: null
  })),

  on(JobsActions.loadJobsSuccess, (state, { jobs }) =>
    jobsAdapter.setAll(jobs, {
      ...state,
      loading: false,
      syncStatus: {
        ...state.syncStatus,
        lastSync: Date.now()
      }
    })
  ),

  // Optimistic update with rollback support
  on(JobsActions.updateJobOptimistic, (state, { id, changes }) =>
    jobsAdapter.updateOne(
      {
        id,
        changes: {
          ...changes,
          _syncStatus: 'pending',
          _localChanges: true
        }
      },
      {
        ...state,
        syncStatus: {
          ...state.syncStatus,
          pendingChanges: [...new Set([...state.syncStatus.pendingChanges, id])]
        }
      }
    )
  ),

  on(JobsActions.updateJobSuccess, (state, { id }) =>
    jobsAdapter.updateOne(
      {
        id,
        changes: {
          _syncStatus: 'synced',
          _localChanges: false
        }
      },
      {
        ...state,
        syncStatus: {
          ...state.syncStatus,
          pendingChanges: state.syncStatus.pendingChanges.filter(pid => pid !== id)
        }
      }
    )
  ),

  on(JobsActions.updateJobFailure, (state, { id, previousState }) => {
    // Rollback to previous state if provided
    if (previousState) {
      return jobsAdapter.updateOne(
        {
          id,
          changes: {
            ...previousState,
            _syncStatus: 'error'
          }
        },
        state
      );
    }
    return jobsAdapter.updateOne(
      { id, changes: { _syncStatus: 'error' } },
      state
    );
  }),

  // Selection
  on(JobsActions.selectJob, (state, { id }) => ({
    ...state,
    selectedId: id
  })),

  on(JobsActions.clearSelection, (state) => ({
    ...state,
    selectedId: null
  })),

  // Filtering
  on(JobsActions.setFilter, (state, { filter }) => ({
    ...state,
    filter: { ...state.filter, ...filter }
  })),

  on(JobsActions.clearFilter, (state) => ({
    ...state,
    filter: initialJobsState.filter
  }))
);

// jobs.effects.ts - Effects with Firestore and offline support
import { Injectable, inject } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { Store } from '@ngrx/store';
import {
  Firestore,
  collection,
  collectionData,
  doc,
  setDoc,
  updateDoc,
  deleteDoc,
  serverTimestamp
} from '@angular/fire/firestore';
import { from, of, timer } from 'rxjs';
import {
  switchMap,
  map,
  catchError,
  withLatestFrom,
  filter,
  retry,
  mergeMap,
  delay
} from 'rxjs/operators';
import { NetworkService } from '@core/services/network.service';
import { selectJobById, selectPendingChanges } from './jobs.selectors';

@Injectable()
export class JobsEffects {
  private readonly actions$ = inject(Actions);
  private readonly store = inject(Store);
  private readonly firestore = inject(Firestore);
  private readonly network = inject(NetworkService);
  private readonly tenantId = 'tenant-123'; // Get from auth service

  // Load jobs with Firestore real-time updates
  loadJobs$ = createEffect(() =>
    this.actions$.pipe(
      ofType(JobsActions.loadJobs),
      switchMap(() => {
        const jobsCollection = collection(this.firestore, `tenants/${this.tenantId}/jobs`);

        return collectionData(jobsCollection, { idField: 'id' }).pipe(
          map(jobs => JobsActions.loadJobsSuccess({ jobs: jobs as Job[] })),
          catchError(error => of(JobsActions.loadJobsFailure({ error: error.message })))
        );
      })
    )
  );

  // Create job
  createJob$ = createEffect(() =>
    this.actions$.pipe(
      ofType(JobsActions.createJob),
      switchMap(({ job }) => {
        const newJob = {
          ...job,
          id: doc(collection(this.firestore, 'dummy')).id,
          createdAt: serverTimestamp(),
          updatedAt: serverTimestamp()
        };

        const jobDoc = doc(this.firestore, `tenants/${this.tenantId}/jobs/${newJob.id}`);

        return from(setDoc(jobDoc, newJob)).pipe(
          map(() => JobsActions.createJobSuccess({ job: newJob as Job })),
          catchError(error => of(JobsActions.createJobFailure({ error: error.message })))
        );
      })
    )
  );

  // Optimistic update with Firestore sync
  updateJob$ = createEffect(() =>
    this.actions$.pipe(
      ofType(JobsActions.updateJob),
      withLatestFrom(this.store.select(state => state)),
      mergeMap(([action, state]) => {
        const currentJob = selectJobById(action.id)(state);
        const previousState = { ...currentJob };

        // Dispatch optimistic update immediately
        this.store.dispatch(JobsActions.updateJobOptimistic({
          id: action.id,
          changes: action.changes
        }));

        // Perform Firestore update
        const jobDoc = doc(this.firestore, `tenants/${this.tenantId}/jobs/${action.id}`);

        return from(updateDoc(jobDoc, {
          ...action.changes,
          updatedAt: serverTimestamp()
        })).pipe(
          map(() => JobsActions.updateJobSuccess({
            id: action.id,
            changes: action.changes
          })),
          retry({
            delay: (error, retryCount) => {
              // Exponential backoff: 1s, 2s, 4s
              const delay = Math.pow(2, retryCount - 1) * 1000;
              return timer(delay);
            },
            count: 3
          }),
          catchError(error => of(JobsActions.updateJobFailure({
            id: action.id,
            error: error.message,
            previousState
          })))
        );
      })
    )
  );

  // Delete job
  deleteJob$ = createEffect(() =>
    this.actions$.pipe(
      ofType(JobsActions.deleteJob),
      switchMap(({ id }) => {
        const jobDoc = doc(this.firestore, `tenants/${this.tenantId}/jobs/${id}`);

        return from(deleteDoc(jobDoc)).pipe(
          map(() => JobsActions.deleteJobSuccess({ id })),
          catchError(error => of(JobsActions.deleteJobFailure({ id, error: error.message })))
        );
      })
    )
  );

  // Sync offline changes when coming back online
  syncOfflineChanges$ = createEffect(() =>
    this.network.online$.pipe(
      filter(online => online),
      withLatestFrom(this.store.select(selectPendingChanges)),
      filter(([_, pendingIds]) => pendingIds.length > 0),
      switchMap(([_, pendingIds]) =>
        from(pendingIds).pipe(
          mergeMap(id =>
            of(JobsActions.syncJob({ id })).pipe(
              delay(100) // Stagger sync operations
            )
          )
        )
      )
    )
  );

  // Process offline sync
  syncOfflineUpdate$ = createEffect(() =>
    this.actions$.pipe(
      ofType(JobsActions.syncOfflineUpdate),
      switchMap(({ payload }) => {
        const jobDoc = doc(this.firestore, `tenants/${this.tenantId}/jobs/${payload.id}`);

        return from(updateDoc(jobDoc, {
          ...payload.data,
          updatedAt: serverTimestamp()
        })).pipe(
          map(() => JobsActions.syncSuccess({ id: payload.id })),
          catchError(error => of(JobsActions.syncFailure({
            id: payload.id,
            error: error.message
          })))
        );
      })
    )
  );
}

// jobs.selectors.ts - Selectors with offline awareness
import { createFeatureSelector, createSelector } from '@ngrx/store';

export const selectJobsState = createFeatureSelector<JobsState>('jobs');

export const { selectAll, selectEntities, selectIds } = jobsAdapter.getSelectors(selectJobsState);

export const selectJobById = (id: string) => createSelector(
  selectEntities,
  entities => entities[id] || null
);

export const selectSelectedJob = createSelector(
  selectJobsState,
  selectEntities,
  (state, entities) => state.selectedId ? entities[state.selectedId] : null
);

export const selectPendingChanges = createSelector(
  selectJobsState,
  state => state.syncStatus.pendingChanges
);

export const selectJobsWithSyncStatus = createSelector(
  selectAll,
  jobs => jobs.map(job => ({
    ...job,
    hasLocalChanges: job._localChanges || false,
    syncStatus: job._syncStatus || 'synced'
  }))
);

export const selectOfflineJobsCount = createSelector(
  selectAll,
  jobs => jobs.filter(j => j._syncStatus === 'pending').length
);

export const selectFilteredJobs = createSelector(
  selectAll,
  selectJobsState,
  (jobs, state) => {
    let filtered = [...jobs];

    if (state.filter.status) {
      filtered = filtered.filter(job => job.status === state.filter.status);
    }

    if (state.filter.searchTerm) {
      const term = state.filter.searchTerm.toLowerCase();
      filtered = filtered.filter(job =>
        job.title.toLowerCase().includes(term) ||
        job.jobNumber.toString().includes(term)
      );
    }

    return filtered;
  }
);
```

## API Integration

### Complete API Service Implementation

```typescript
// libs/mobile/data-access/src/lib/services/api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpHeaders, HttpErrorResponse } from '@angular/common/http';
import { Functions, httpsCallable, httpsCallableData } from '@angular/fire/functions';
import { Auth } from '@angular/fire/auth';
import { Observable, from, throwError, of, timer } from 'rxjs';
import { catchError, retry, timeout, switchMap, map } from 'rxjs/operators';
import { NetworkService } from './network.service';
import { SyncQueueService } from './sync-queue.service';

// Type-safe function names
export type CloudFunctionName =
  | 'allocateSequence'
  | 'generatePDF'
  | 'processVoiceCommand'
  | 'exportReport';

// Request/Response interfaces
export interface AllocateSequenceRequest {
  tenantId: string;
  sequenceType: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber' | 'ordinalNumber';
  jobId?: string; // Required for ordinalNumber
}

export interface AllocateSequenceResponse {
  number: number;
  sequenceType: string;
  allocatedAt: string;
}

export interface GeneratePDFRequest {
  jobId: string;
  options?: {
    includePhotos?: boolean;
    includeSignatures?: boolean;
    templateId?: string;
  };
}

export interface ProcessVoiceCommandRequest {
  transcript: string;
  context: {
    activeJobId?: string;
    currentScreen?: string;
    previousCommands?: string[];
    tenantId: string;
    userId: string;
  };
}

export interface ProcessVoiceCommandResponse {
  intent: string;
  action: {
    type: string;
    payload: any;
  };
  response: string;
  confidence: number;
}

export interface ApiResponse<T = any> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  metadata?: {
    timestamp: number;
    requestId: string;
  };
}

@Injectable({ providedIn: 'root' })
export class ApiService {
  private readonly http = inject(HttpClient);
  private readonly functions = inject(Functions);
  private readonly auth = inject(Auth);
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);

  // Cloud Functions base configuration
  private readonly functionsConfig = {
    region: 'us-central1',
    timeout: 30000, // 30 seconds
    retryConfig: {
      count: 3,
      delay: 1000,
      maxDelay: 5000
    }
  };

  /**
   * Call a Cloud Function with automatic retry and offline queue
   */
  callFunction<TRequest, TResponse>(
    functionName: CloudFunctionName,
    data: TRequest,
    options?: {
      requiresAuth?: boolean;
      queueIfOffline?: boolean;
      timeout?: number;
    }
  ): Observable<TResponse> {
    const opts = {
      requiresAuth: true,
      queueIfOffline: false,
      timeout: this.functionsConfig.timeout,
      ...options
    };

    // Check if offline and should queue
    if (!this.network.isOnline() && opts.queueIfOffline) {
      return this.queueFunctionCall(functionName, data);
    }

    // Get the callable function
    const callable = httpsCallable<TRequest, TResponse>(
      this.functions,
      functionName,
      { timeout: opts.timeout }
    );

    return from(callable(data)).pipe(
      retry({
        count: this.functionsConfig.retryConfig.count,
        delay: this.exponentialBackoff
      }),
      catchError(error => this.handleFunctionError(error, functionName))
    );
  }

  /**
   * Queue function call for later execution when online
   */
  private queueFunctionCall<T>(functionName: string, data: any): Observable<T> {
    const operation = {
      type: 'CLOUD_FUNCTION',
      payload: {
        functionName,
        data,
        timestamp: Date.now()
      },
      timestamp: Date.now()
    };

    this.syncQueue.addOperation(operation);

    // Return optimistic response or pending indicator
    return of({
      success: true,
      data: null,
      metadata: {
        queued: true,
        queuedAt: Date.now()
      }
    } as any);
  }

  /**
   * Exponential backoff for retries
   */
  private exponentialBackoff(error: any, retryCount: number): Observable<number> {
    const delay = Math.min(
      this.functionsConfig.retryConfig.delay * Math.pow(2, retryCount - 1),
      this.functionsConfig.retryConfig.maxDelay
    );
    console.warn(`Retry attempt ${retryCount} after ${delay}ms for error:`, error);
    return timer(delay);
  }

  /**
   * Handle Cloud Function errors
   */
  private handleFunctionError(error: any, functionName: string): Observable<never> {
    let errorMessage = 'An unexpected error occurred';
    let errorCode = 'UNKNOWN_ERROR';

    if (error.code) {
      // Firebase Function error codes
      switch (error.code) {
        case 'functions/cancelled':
          errorMessage = 'Request was cancelled';
          errorCode = 'CANCELLED';
          break;
        case 'functions/unknown':
          errorMessage = error.message || 'Unknown error occurred';
          errorCode = 'UNKNOWN';
          break;
        case 'functions/deadline-exceeded':
          errorMessage = 'Request timeout';
          errorCode = 'TIMEOUT';
          break;
        case 'functions/permission-denied':
          errorMessage = 'Permission denied';
          errorCode = 'PERMISSION_DENIED';
          break;
        case 'functions/unauthenticated':
          errorMessage = 'Authentication required';
          errorCode = 'UNAUTHENTICATED';
          break;
        default:
          errorMessage = error.message || errorMessage;
          errorCode = error.code;
      }
    }

    console.error(`Cloud Function ${functionName} error:`, {
      code: errorCode,
      message: errorMessage,
      details: error
    });

    return throwError(() => ({
      code: errorCode,
      message: errorMessage,
      functionName,
      timestamp: Date.now()
    }));
  }

  // ============================================
  // Specific Cloud Function Implementations
  // ============================================

  /**
   * Allocate next sequence number for tenant
   */
  allocateSequence(request: AllocateSequenceRequest): Observable<AllocateSequenceResponse> {
    return this.callFunction<AllocateSequenceRequest, AllocateSequenceResponse>(
      'allocateSequence',
      request,
      { queueIfOffline: true } // Queue this if offline
    );
  }

  /**
   * Generate PDF report (Phase 2)
   */
  generatePDF(jobId: string, options?: {
    includePhotos?: boolean;
    includeSignatures?: boolean;
  }): Observable<{ url: string; expiresAt: string }> {
    return this.callFunction<GeneratePDFRequest, { url: string; expiresAt: string }>(
      'generatePDF',
      { jobId, options },
      {
        timeout: 60000, // 60 seconds for PDF generation
        queueIfOffline: false // Don't queue PDF generation
      }
    );
  }

  /**
   * Process voice command through LLM
   */
  processVoiceCommand(command: {
    transcript: string;
    context: {
      activeJobId?: string;
      currentScreen?: string;
      previousCommands?: string[];
    };
  }): Observable<ProcessVoiceCommandResponse> {
    const tenantId = localStorage.getItem('tenantId') || '';
    const userId = this.auth.currentUser?.uid || '';

    return this.callFunction<ProcessVoiceCommandRequest, ProcessVoiceCommandResponse>(
      'processVoiceCommand',
      {
        ...command,
        context: {
          ...command.context,
          tenantId,
          userId
        }
      },
      {
        timeout: 15000, // 15 seconds for voice processing
        queueIfOffline: false // Real-time only
      }
    );
  }

  /**
   * Export report
   */
  exportReport(params: {
    jobId?: string;
    dateRange?: { start: Date; end: Date };
    format: 'csv' | 'xlsx' | 'pdf';
  }): Observable<{ url: string }> {
    return this.callFunction(
      'exportReport',
      params,
      {
        timeout: 45000,
        queueIfOffline: false
      }
    );
  }
}
```

### API Client Configuration (Complete Interceptors)

```typescript
// libs/mobile/data-access/src/lib/interceptors/auth.interceptor.ts
import { Injectable, inject } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError, from } from 'rxjs';
import { catchError, switchMap } from 'rxjs/operators';
import { Auth } from '@angular/fire/auth';
import { Router } from '@angular/router';
import { Capacitor } from '@capacitor/core';
import { environment } from '@environments/environment';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private readonly auth = inject(Auth);
  private readonly router = inject(Router);

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Skip auth for public endpoints
    if (this.isPublicEndpoint(req.url)) {
      return next.handle(req);
    }

    // Add auth token to requests
    return from(this.auth.currentUser?.getIdToken() ?? Promise.resolve(null)).pipe(
      switchMap(token => {
        if (!token) {
          // No token available, redirect to login
          this.router.navigate(['/auth/login']);
          return throwError(() => new Error('No authentication token'));
        }

        // Clone request with auth header
        const authReq = req.clone({
          setHeaders: {
            Authorization: `Bearer ${token}`,
            'X-Tenant-Id': this.getTenantId(),
            'X-Client-Version': this.getAppVersion(),
            'X-Platform': this.getPlatform()
          }
        });

        return next.handle(authReq);
      }),
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Token expired or invalid
          this.handleAuthError();
        }
        return throwError(() => error);
      })
    );
  }

  private isPublicEndpoint(url: string): boolean {
    const publicEndpoints = ['/api/public', '/api/health'];
    return publicEndpoints.some(endpoint => url.includes(endpoint));
  }

  private handleAuthError(): void {
    // Sign out and redirect to login
    this.auth.signOut();
    this.router.navigate(['/auth/login']);
  }

  private getTenantId(): string {
    // Get from user claims or local storage
    return localStorage.getItem('tenantId') || '';
  }

  private getAppVersion(): string {
    return environment.appVersion;
  }

  private getPlatform(): string {
    return Capacitor.getPlatform(); // 'ios', 'android', or 'web'
  }
}

// libs/mobile/data-access/src/lib/interceptors/offline.interceptor.ts
import { Injectable, inject } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpResponse } from '@angular/common/http';
import { Observable, throwError, of } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { NetworkService } from '../services/network.service';
import { SyncQueueService } from '../services/sync-queue.service';

@Injectable()
export class OfflineInterceptor implements HttpInterceptor {
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Check if we're offline before making request
    if (!this.network.isOnline()) {
      // Check if this request can be queued
      if (this.canQueueRequest(req)) {
        return this.queueRequest(req);
      }

      // Return offline error
      return throwError(() => ({
        status: 0,
        statusText: 'Offline',
        error: {
          message: 'No network connection available',
          code: 'OFFLINE'
        }
      }));
    }

    return next.handle(req).pipe(
      catchError(error => {
        // Network error occurred during request
        if (error.status === 0 && this.canQueueRequest(req)) {
          return this.queueRequest(req);
        }
        return throwError(() => error);
      })
    );
  }

  private canQueueRequest(req: HttpRequest<any>): boolean {
    // Only queue POST/PUT/PATCH requests
    const queuableMethods = ['POST', 'PUT', 'PATCH'];
    return queuableMethods.includes(req.method) && !req.url.includes('auth');
  }

  private queueRequest(req: HttpRequest<any>): Observable<HttpEvent<any>> {
    // Add to sync queue
    this.syncQueue.addOperation({
      type: 'HTTP_REQUEST',
      payload: {
        url: req.url,
        method: req.method,
        body: req.body,
        headers: req.headers.keys().reduce((acc, key) => ({
          ...acc,
          [key]: req.headers.get(key)
        }), {})
      },
      timestamp: Date.now()
    });

    // Return success response for optimistic update
    return of(new HttpResponse({
      status: 202,
      statusText: 'Accepted',
      body: {
        queued: true,
        message: 'Request queued for sync when online'
      }
    }));
  }
}

// app.config.ts - Configure interceptors
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptorsFromDi } from '@angular/common/http';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptorsFromDi()),
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    },
    {
      provide: HTTP_INTERCEPTORS,
      useClass: OfflineInterceptor,
      multi: true
    },
    // ... other providers
  ]
};
```

## Routing (Complete Implementation)

### Full Route Configuration

```typescript
// app/app.routes.ts - Main application routes
import { Routes } from '@angular/router';
import { AuthGuard } from '@core/guards/auth.guard';
import { TenantGuard } from '@core/guards/tenant.guard';
import { OnboardingGuard } from '@core/guards/onboarding.guard';
import { RoleGuard } from '@core/guards/role.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/tabs/jobs',
    pathMatch: 'full'
  },
  {
    path: 'auth',
    loadChildren: () => import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES)
  },
  {
    path: 'onboarding',
    loadChildren: () => import('./features/onboarding/onboarding.routes').then(m => m.ONBOARDING_ROUTES),
    canActivate: [AuthGuard]
  },
  {
    path: 'tabs',
    loadComponent: () => import('./layouts/tabs/tabs.page').then(m => m.TabsPage),
    canActivate: [AuthGuard, TenantGuard],
    children: [
      {
        path: 'jobs',
        loadChildren: () => import('./features/jobs/jobs.routes').then(m => m.JOBS_ROUTES),
        data: { preload: true }
      },
      {
        path: 'voice',
        loadChildren: () => import('./features/voice/voice.routes').then(m => m.VOICE_ROUTES),
        data: { preload: true }
      },
      {
        path: 'costs',
        loadChildren: () => import('./features/costs/costs.routes').then(m => m.COSTS_ROUTES)
      },
      {
        path: 'team',
        loadChildren: () => import('./features/team/team.routes').then(m => m.TEAM_ROUTES),
        canActivate: [RoleGuard],
        data: { roles: ['owner', 'representative'] }
      },
      {
        path: 'settings',
        loadChildren: () => import('./features/settings/settings.routes').then(m => m.SETTINGS_ROUTES)
      },
      {
        path: '',
        redirectTo: '/tabs/jobs',
        pathMatch: 'full'
      }
    ]
  },
  {
    path: '**',
    loadComponent: () => import('./pages/not-found/not-found.page').then(m => m.NotFoundPage)
  }
];

// features/jobs/jobs.routes.ts - Feature-specific routes
import { Routes } from '@angular/router';
import { JobsListPage } from './pages/jobs-list/jobs-list.page';
import { JobDetailPage } from './pages/job-detail/job-detail.page';
import { JobEditPage } from './pages/job-edit/job-edit.page';
import { JobCreatePage } from './pages/job-create/job-create.page';
import { PendingChangesGuard } from '@core/guards/pending-changes.guard';
import { JobResolver } from './resolvers/job.resolver';

export const JOBS_ROUTES: Routes = [
  {
    path: '',
    component: JobsListPage,
    title: 'Jobs'
  },
  {
    path: 'create',
    component: JobCreatePage,
    canDeactivate: [PendingChangesGuard],
    title: 'Create Job'
  },
  {
    path: ':id',
    component: JobDetailPage,
    resolve: {
      job: JobResolver
    },
    title: 'Job Details'
  },
  {
    path: ':id/edit',
    component: JobEditPage,
    canDeactivate: [PendingChangesGuard],
    resolve: {
      job: JobResolver
    },
    title: 'Edit Job'
  },
  {
    path: ':id/costs',
    loadChildren: () => import('./features/job-costs/job-costs.routes').then(m => m.JOB_COSTS_ROUTES)
  }
];
```

### Complete Guard Implementations

```typescript
// core/guards/auth.guard.ts - Authentication guard
import { Injectable, inject } from '@angular/core';
import { CanActivate, Router, UrlTree } from '@angular/router';
import { Auth, authState } from '@angular/fire/auth';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  private readonly auth = inject(Auth);
  private readonly router = inject(Router);

  canActivate(): Observable<boolean | UrlTree> {
    return authState(this.auth).pipe(
      take(1),
      map(user => {
        if (user) {
          return true;
        }
        // Store intended URL for redirecting after login
        const returnUrl = window.location.pathname;
        return this.router.createUrlTree(['/auth/login'], {
          queryParams: { returnUrl }
        });
      })
    );
  }
}

// core/guards/tenant.guard.ts - Ensure user has tenant association
import { Injectable, inject } from '@angular/core';
import { CanActivate, Router, UrlTree } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';
import { selectCurrentTenant } from '@store/auth/auth.selectors';

@Injectable({ providedIn: 'root' })
export class TenantGuard implements CanActivate {
  private readonly store = inject(Store);
  private readonly router = inject(Router);

  canActivate(): Observable<boolean | UrlTree> {
    return this.store.select(selectCurrentTenant).pipe(
      take(1),
      map(tenant => {
        if (tenant) {
          return true;
        }
        // Redirect to onboarding if no tenant
        return this.router.createUrlTree(['/onboarding']);
      })
    );
  }
}

// core/guards/pending-changes.guard.ts - Prevent navigation with unsaved changes
import { Injectable } from '@angular/core';
import { CanDeactivate } from '@angular/router';
import { Observable } from 'rxjs';
import { AlertController } from '@ionic/angular';

export interface ComponentCanDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable({ providedIn: 'root' })
export class PendingChangesGuard implements CanDeactivate<ComponentCanDeactivate> {
  constructor(private alertController: AlertController) {}

  async canDeactivate(component: ComponentCanDeactivate): Promise<boolean> {
    if (!component.canDeactivate || component.canDeactivate()) {
      return true;
    }

    const alert = await this.alertController.create({
      header: 'Unsaved Changes',
      message: 'You have unsaved changes. Do you want to leave without saving?',
      buttons: [
        {
          text: 'Stay',
          role: 'cancel',
          cssClass: 'secondary'
        },
        {
          text: 'Leave',
          role: 'confirm',
          cssClass: 'danger'
        }
      ]
    });

    await alert.present();
    const { role } = await alert.onDidDismiss();

    return role === 'confirm';
  }
}

// core/guards/role.guard.ts - Role-based access control
import { Injectable, inject } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router, UrlTree } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';
import { selectUserRole } from '@store/auth/auth.selectors';

@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  private readonly store = inject(Store);
  private readonly router = inject(Router);

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean | UrlTree> {
    const requiredRoles = route.data['roles'] as string[];

    return this.store.select(selectUserRole).pipe(
      take(1),
      map(userRole => {
        if (requiredRoles.includes(userRole)) {
          return true;
        }
        // Redirect to unauthorized page or home
        return this.router.createUrlTree(['/tabs/jobs'], {
          queryParams: { unauthorized: true }
        });
      })
    );
  }
}

// core/resolvers/job.resolver.ts - Preload job data
import { Injectable, inject } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable, of } from 'rxjs';
import { take, tap, filter, switchMap } from 'rxjs/operators';
import { JobsActions } from '@features/jobs/data-access/jobs.actions';
import { selectJobById } from '@features/jobs/data-access/jobs.selectors';
import { Job } from '@shared/types';

@Injectable({ providedIn: 'root' })
export class JobResolver implements Resolve<Job> {
  private readonly store = inject(Store);

  resolve(route: ActivatedRouteSnapshot): Observable<Job> {
    const jobId = route.params['id'];

    // Check if job is already in store
    return this.store.select(state => selectJobById(jobId)(state)).pipe(
      take(1),
      switchMap(job => {
        if (job) {
          return of(job);
        }

        // Load job if not in store
        this.store.dispatch(JobsActions.loadJob({ id: jobId }));

        // Wait for job to be loaded
        return this.store.select(state => selectJobById(jobId)(state)).pipe(
          filter(job => !!job),
          take(1)
        );
      })
    );
  }
}

// app.config.ts - Configure routing with preloading strategy
import { ApplicationConfig } from '@angular/core';
import {
  provideRouter,
  withPreloading,
  PreloadAllModules,
  withComponentInputBinding,
  withInMemoryScrolling,
  withRouterConfig
} from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules), // Preload all lazy modules for PWA
      withComponentInputBinding(), // Enable router input binding
      withInMemoryScrolling({
        scrollPositionRestoration: 'enabled', // Restore scroll position
        anchorScrolling: 'enabled'
      }),
      withRouterConfig({
        paramsInheritanceStrategy: 'always' // Inherit params from parent routes
      })
    ),
    // ... other providers
  ]
};
```

## Styling Guidelines

### Theme System Configuration

```scss
// apps/mobile-app/src/theme/variables.scss

:root {
  /* ===== Color Palette ===== */
  --ion-color-primary: #3880ff;
  --ion-color-primary-rgb: 56, 128, 255;
  --ion-color-primary-contrast: #ffffff;
  --ion-color-primary-contrast-rgb: 255, 255, 255;
  --ion-color-primary-shade: #3171e0;
  --ion-color-primary-tint: #4c8dff;

  --ion-color-secondary: #3dc2ff;
  --ion-color-secondary-rgb: 61, 194, 255;
  --ion-color-secondary-contrast: #ffffff;
  --ion-color-secondary-contrast-rgb: 255, 255, 255;
  --ion-color-secondary-shade: #36abe0;
  --ion-color-secondary-tint: #50c8ff;

  --ion-color-tertiary: #5260ff;
  --ion-color-tertiary-rgb: 82, 96, 255;
  --ion-color-tertiary-contrast: #ffffff;
  --ion-color-tertiary-contrast-rgb: 255, 255, 255;
  --ion-color-tertiary-shade: #4854e0;
  --ion-color-tertiary-tint: #6370ff;

  --ion-color-success: #2dd36f;
  --ion-color-success-rgb: 45, 211, 111;
  --ion-color-success-contrast: #ffffff;
  --ion-color-success-contrast-rgb: 255, 255, 255;
  --ion-color-success-shade: #28ba62;
  --ion-color-success-tint: #42d77d;

  --ion-color-warning: #ffc409;
  --ion-color-warning-rgb: 255, 196, 9;
  --ion-color-warning-contrast: #000000;
  --ion-color-warning-contrast-rgb: 0, 0, 0;
  --ion-color-warning-shade: #e0ac08;
  --ion-color-warning-tint: #ffca22;

  --ion-color-danger: #eb445a;
  --ion-color-danger-rgb: 235, 68, 90;
  --ion-color-danger-contrast: #ffffff;
  --ion-color-danger-contrast-rgb: 255, 255, 255;
  --ion-color-danger-shade: #cf3c4f;
  --ion-color-danger-tint: #ed576b;

  /* ===== App-Specific Colors ===== */
  --app-primary: var(--ion-color-primary);
  --app-primary-contrast: var(--ion-color-primary-contrast);
  --app-success: var(--ion-color-success);
  --app-warning: var(--ion-color-warning);
  --app-danger: var(--ion-color-danger);

  /* Voice UI Colors */
  --voice-idle-color: #92949c;
  --voice-listening-color: #3880ff;
  --voice-processing-color: #ffc409;
  --voice-speaking-color: #2dd36f;
  --voice-error-color: #eb445a;
  --voice-active-pulse: rgba(56, 128, 255, 0.3);
  --voice-recording-pulse: rgba(255, 196, 9, 0.3);

  /* ===== Typography ===== */
  --font-family-base: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-family-monospace: 'Courier New', Courier, monospace;

  /* Font Sizes */
  --font-size-xs: 0.75rem;   /* 12px */
  --font-size-sm: 0.875rem;  /* 14px */
  --font-size-base: 1rem;    /* 16px */
  --font-size-md: 1.125rem;  /* 18px */
  --font-size-lg: 1.25rem;   /* 20px */
  --font-size-xl: 1.5rem;    /* 24px */
  --font-size-2xl: 2rem;     /* 32px */

  /* Font Weights */
  --font-weight-light: 300;
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;

  /* Line Heights */
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;

  /* ===== Spacing System ===== */
  --spacing-xs: 0.25rem;  /* 4px */
  --spacing-sm: 0.5rem;   /* 8px */
  --spacing-md: 1rem;     /* 16px */
  --spacing-lg: 1.5rem;   /* 24px */
  --spacing-xl: 2rem;     /* 32px */
  --spacing-2xl: 3rem;    /* 48px */
  --spacing-3xl: 4rem;    /* 64px */

  /* ===== Touch Targets (Accessible) ===== */
  --touch-target-min: 48px;         /* Minimum touch target */
  --touch-target-comfortable: 56px;  /* Comfortable for field work */
  --touch-target-large: 64px;       /* Extra large for critical actions */

  /* ===== Border Radius ===== */
  --radius-none: 0;
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-xl: 1rem;
  --radius-full: 50%;

  /* ===== Shadows ===== */
  --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 20px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 40px rgba(0, 0, 0, 0.15);

  /* ===== Z-Index Scale ===== */
  --z-dropdown: 1000;
  --z-sticky: 1020;
  --z-fixed: 1030;
  --z-modal-backdrop: 1040;
  --z-modal: 1050;
  --z-popover: 1060;
  --z-tooltip: 1070;
  --z-toast: 1080;

  /* ===== Transitions ===== */
  --transition-fast: 150ms ease-in-out;
  --transition-base: 250ms ease-in-out;
  --transition-slow: 350ms ease-in-out;

  /* ===== Offline Mode ===== */
  --offline-bg: #fff7ed;
  --offline-border: #f59e0b;
  --offline-text: #92400e;
}

/* ===== Dark Mode ===== */
@media (prefers-color-scheme: dark) {
  :root {
    /* Dark mode color overrides */
    --ion-color-primary: #428cff;
    --ion-color-primary-rgb: 66, 140, 255;
    --ion-color-primary-contrast: #ffffff;
    --ion-color-primary-contrast-rgb: 255, 255, 255;
    --ion-color-primary-shade: #3a7be0;
    --ion-color-primary-tint: #5598ff;

    /* Background colors for dark mode */
    --ion-background-color: #121212;
    --ion-background-color-rgb: 18, 18, 18;
    --ion-text-color: #ffffff;
    --ion-text-color-rgb: 255, 255, 255;

    /* Card and surface colors */
    --ion-card-background: #1e1e1e;
    --ion-item-background: #1e1e1e;

    /* Dark mode shadows (lighter shadows) */
    --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.3);
    --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.3);
    --shadow-lg: 0 10px 20px rgba(0, 0, 0, 0.3);
    --shadow-xl: 0 20px 40px rgba(0, 0, 0, 0.3);

    /* Dark mode offline indicators */
    --offline-bg: #451a03;
    --offline-border: #f59e0b;
    --offline-text: #fef3c7;
  }
}

/* ===== Utility Classes ===== */
.touch-target {
  min-width: var(--touch-target-min);
  min-height: var(--touch-target-min);
}

.touch-target-large {
  min-width: var(--touch-target-large);
  min-height: var(--touch-target-large);
}

/* Voice UI States */
.voice-listening {
  --ion-color-base: var(--voice-active-color);
  animation: pulse 1.5s infinite;
}

.voice-recording {
  --ion-color-base: var(--voice-recording-pulse);
  animation: recording-pulse 1s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

@keyframes recording-pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

/* Offline Mode Styling */
.offline-banner {
  background: var(--offline-bg);
  border: 1px solid var(--offline-border);
  color: var(--offline-text);
  padding: var(--spacing-sm) var(--spacing-md);
  text-align: center;
  position: sticky;
  top: 0;
  z-index: var(--z-sticky);
}

/* PWA Install Prompt */
.pwa-install-prompt {
  position: fixed;
  bottom: var(--spacing-lg);
  left: var(--spacing-md);
  right: var(--spacing-md);
  background: var(--ion-color-primary);
  color: var(--ion-color-primary-contrast);
  padding: var(--spacing-md);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-lg);
  z-index: var(--z-toast);
}
```

### Component-Specific Styling Patterns

```scss
// Example: Job Card Component Styling
// apps/mobile-app/src/app/features/jobs/components/job-card/job-card.component.scss

:host {
  display: block;
}

.job-card {
  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: var(--spacing-md);

    &--active {
      background: var(--app-primary);
      color: var(--app-primary-contrast);
    }
  }

  &__title {
    font-size: var(--font-size-lg);
    font-weight: var(--font-weight-semibold);
    margin: 0;
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: var(--spacing-xs);

    &-indicator {
      width: 8px;
      height: 8px;
      border-radius: var(--radius-full);

      &--synced { background: var(--app-success); }
      &--pending { background: var(--app-warning); }
      &--error { background: var(--app-danger); }
    }
  }

  &__actions {
    display: flex;
    gap: var(--spacing-sm);
    padding: var(--spacing-md);

    ion-button {
      min-height: var(--touch-target-min);
      min-width: var(--touch-target-comfortable);
    }
  }
}

// Responsive adjustments
@media (max-width: 576px) {
  .job-card {
    &__actions {
      flex-direction: column;

      ion-button {
        width: 100%;
      }
    }
  }
}
```

### Rationale for Styling Approach

**CSS Custom Properties First:**
- Runtime theming without rebuilds
- Dark mode switching instant
- Consistent spacing/sizing across app
- Easy white-labeling for future clients

**Touch-Friendly Targets:**
- Minimum 48px targets (Material Design guideline)
- Comfortable 56px for primary actions
- Large 64px for critical voice/safety actions

**Voice UI Visual Feedback:**
- Distinct colors for listening/recording/processing
- Animations for active states
- Clear visual hierarchy for voice commands

**Offline Mode Indicators:**
- Sticky positioning for constant awareness
- Distinct color scheme from normal operations
- Non-intrusive but noticeable

---

*Document Version: 1.0*
*Last Updated: 2025-01-27*
*Created by: Winston (Architect)*

## Testing Requirements

### Component Test Template

Here's the Jest test template for Angular 20+ components with Ionic and offline support:

```typescript
// job-detail.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { IonicModule } from '@ionic/angular';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { signal } from '@angular/core';
import { JobDetailComponent } from './job-detail.component';
import { NetworkService } from '@core/services/network.service';
import { SyncQueueService } from '@core/services/sync-queue.service';
import { selectActiveJob, selectJobLoading } from '../data-access/jobs.selectors';
import { JobsActions } from '../data-access/jobs.actions';

describe('JobDetailComponent', () => {
  let component: JobDetailComponent;
  let fixture: ComponentFixture<JobDetailComponent>;
  let store: MockStore;
  let networkService: jasmine.SpyObj<NetworkService>;
  let syncQueueService: jasmine.SpyObj<SyncQueueService>;

  const mockJob = {
    id: 'job-123',
    title: 'Test Job',
    status: 'active',
    jobNumber: 1001,
    budget: 5000,
    currency: 'USD',
    vatRate: 20,
    tenantId: 'tenant-123',
    createdAt: new Date(),
    updatedAt: new Date()
  };

  beforeEach(async () => {
    // Create service spies
    const networkSpy = jasmine.createSpyObj('NetworkService', [], {
      isOnline: signal(true),
      online$: of(true)
    });

    const syncQueueSpy = jasmine.createSpyObj('SyncQueueService',
      ['addOperation', 'processPendingOperations'],
      { pendingCount$: of(0) }
    );

    await TestBed.configureTestingModule({
      imports: [
        JobDetailComponent, // Standalone component
        IonicModule.forRoot()
      ],
      providers: [
        provideMockStore({
          initialState: {
            jobs: {
              entities: { [mockJob.id]: mockJob },
              ids: [mockJob.id],
              selectedId: mockJob.id,
              loading: false,
              error: null
            }
          },
          selectors: [
            { selector: selectActiveJob, value: mockJob },
            { selector: selectJobLoading, value: false }
          ]
        }),
        provideRouter([]),
        { provide: NetworkService, useValue: networkSpy },
        { provide: SyncQueueService, useValue: syncQueueSpy }
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    networkService = TestBed.inject(NetworkService) as jasmine.SpyObj<NetworkService>;
    syncQueueService = TestBed.inject(SyncQueueService) as jasmine.SpyObj<SyncQueueService>;

    fixture = TestBed.createComponent(JobDetailComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  afterEach(() => {
    store.resetSelectors();
  });

  describe('Component Initialization', () => {
    it('should create', () => {
      expect(component).toBeTruthy();
    });

    it('should load job data on init', () => {
      spyOn(store, 'dispatch');
      component.ngOnInit();

      expect(store.dispatch).toHaveBeenCalledWith(
        JobsActions.loadJobDetail({ id: mockJob.id })
      );
    });

    it('should display job title', () => {
      const titleElement = fixture.nativeElement.querySelector('ion-title');
      expect(titleElement.textContent).toContain(mockJob.title);
    });
  });

  describe('Offline Behavior', () => {
    it('should show offline indicator when network is unavailable', () => {
      networkService.isOnline = signal(false);
      fixture.detectChanges();

      const offlineChip = fixture.nativeElement.querySelector('.offline-chip');
      expect(offlineChip).toBeTruthy();
      expect(offlineChip.textContent).toContain('Offline');
    });

    it('should queue updates when offline', async () => {
      networkService.isOnline = signal(false);
      const updateData = { title: 'Updated Title' };

      await component.save(updateData);

      expect(syncQueueService.addOperation).toHaveBeenCalledWith(
        jasmine.objectContaining({
          type: 'UPDATE_JOB',
          payload: jasmine.objectContaining({
            id: mockJob.id,
            data: updateData
          })
        })
      );
    });

    it('should show pending sync count', () => {
      syncQueueService.pendingCount$ = of(3);
      fixture.detectChanges();

      const syncBadge = fixture.nativeElement.querySelector('ion-badge');
      expect(syncBadge.textContent).toContain('3');
    });
  });

  describe('User Interactions', () => {
    it('should navigate back when back button is clicked', () => {
      const backButton = fixture.nativeElement.querySelector('ion-back-button');
      backButton.click();

      // Verify navigation occurred (Ionic handles this internally)
      expect(backButton).toBeTruthy();
    });

    it('should handle form submission', () => {
      spyOn(store, 'dispatch');
      const formData = { title: 'New Title', budget: 10000 };

      component.save(formData);

      expect(store.dispatch).toHaveBeenCalledWith(
        JobsActions.updateJob({
          id: mockJob.id,
          changes: formData
        })
      );
    });

    it('should show unsaved changes warning', () => {
      component.hasUnsavedChanges.set(true);

      const canDeactivate = component.canDeactivate();
      expect(canDeactivate).toBeFalsy();
    });
  });

  describe('State Management', () => {
    it('should update job signal when store emits new value', () => {
      const updatedJob = { ...mockJob, title: 'Updated Job' };
      store.overrideSelector(selectActiveJob, updatedJob);
      store.refreshState();

      expect(component.job()).toEqual(updatedJob);
    });

    it('should handle loading state', () => {
      store.overrideSelector(selectJobLoading, true);
      store.refreshState();

      const progressBar = fixture.nativeElement.querySelector('ion-progress-bar');
      expect(progressBar).toBeTruthy();
    });

    it('should handle error state', () => {
      const errorMessage = 'Failed to load job';
      component.error.set(errorMessage);
      fixture.detectChanges();

      const errorElement = fixture.nativeElement.querySelector('.error-message');
      expect(errorElement.textContent).toContain(errorMessage);
    });
  });

  describe('Accessibility', () => {
    it('should have proper ARIA labels', () => {
      const saveButton = fixture.nativeElement.querySelector('[aria-label="Save job"]');
      expect(saveButton).toBeTruthy();
    });

    it('should support keyboard navigation', () => {
      const firstInput = fixture.nativeElement.querySelector('ion-input');
      firstInput.focus();

      expect(document.activeElement).toBe(firstInput);
    });
  });
});
```

### Integration Test Example

```typescript
// voice-pipeline.integration.spec.ts
import { TestBed } from '@angular/core/testing';
import { VoicePipelineService } from '@core/services/voice-pipeline.service';
import { Store } from '@ngrx/store';
import { provideMockStore } from '@ngrx/store/testing';

describe('Voice Pipeline Integration', () => {
  let voiceService: VoicePipelineService;
  let store: MockStore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        VoicePipelineService,
        provideMockStore({ initialState: {} }),
        // Mock Speech Recognition
        {
          provide: 'SpeechRecognition',
          useValue: jasmine.createSpyObj('SpeechRecognition', [
            'start', 'stop', 'addEventListener'
          ])
        }
      ]
    });

    voiceService = TestBed.inject(VoicePipelineService);
    store = TestBed.inject(MockStore);
  });

  describe('Voice Command Flow', () => {
    it('should process "set active job" command', async () => {
      const mockTranscript = 'set active job number 1234';
      spyOn(store, 'dispatch');

      // Simulate STT result
      await voiceService.processTranscript(mockTranscript);

      expect(store.dispatch).toHaveBeenCalledWith(
        jasmine.objectContaining({
          type: '[Voice] Set Active Job',
          jobNumber: 1234
        })
      );
    });

    it('should handle "start journey" command', async () => {
      const mockTranscript = 'start journey to client site';

      const result = await voiceService.processTranscript(mockTranscript);

      expect(result.intent).toBe('START_JOURNEY');
      expect(result.destination).toBe('client site');
    });

    it('should queue commands when offline', async () => {
      // Set offline state
      voiceService.setOfflineMode(true);

      const mockTranscript = 'add material cost concrete 500 dollars';
      const result = await voiceService.processTranscript(mockTranscript);

      expect(result.queued).toBeTruthy();
      expect(result.willProcessWhenOnline).toBeTruthy();
    });
  });
});
```

### E2E Test with Playwright

```typescript
// e2e/specs/job-creation.spec.ts
import { test, expect, devices } from '@playwright/test';

// Test on multiple devices including mobile
test.describe('Job Creation Flow', () => {
  test.use({ ...devices['iPhone 13'] }); // Mobile viewport

  test.beforeEach(async ({ page }) => {
    // Start with PWA installed
    await page.goto('/');

    // Login
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');

    // Wait for navigation
    await page.waitForURL('/tabs/jobs');
  });

  test('should create a new job', async ({ page }) => {
    // Navigate to job creation
    await page.click('[data-testid="create-job-fab"]');

    // Fill job form
    await page.fill('[data-testid="job-title-input"]', 'Renovation Project');
    await page.fill('[data-testid="job-budget-input"]', '25000');
    await page.selectOption('[data-testid="job-currency-select"]', 'USD');

    // Save job
    await page.click('[data-testid="save-job-button"]');

    // Verify navigation back to list
    await expect(page).toHaveURL('/tabs/jobs');

    // Verify job appears in list
    const jobCard = page.locator('[data-testid="job-card"]').first();
    await expect(jobCard).toContainText('Renovation Project');
  });

  test('should work offline', async ({ page, context }) => {
    // Go offline
    await context.setOffline(true);

    // Try to create a job
    await page.click('[data-testid="create-job-fab"]');
    await page.fill('[data-testid="job-title-input"]', 'Offline Job');
    await page.click('[data-testid="save-job-button"]');

    // Verify offline indicator
    await expect(page.locator('.offline-banner')).toBeVisible();
    await expect(page.locator('.offline-banner')).toContainText('Offline Mode');

    // Verify optimistic update
    await expect(page.locator('[data-testid="job-card"]')).toContainText('Offline Job');

    // Go back online
    await context.setOffline(false);

    // Verify sync indicator appears and disappears
    await expect(page.locator('.sync-indicator')).toBeVisible();
    await expect(page.locator('.sync-indicator')).toBeHidden({ timeout: 5000 });
  });

  test('should handle imprecise touch interaction', async ({ page }) => {
    // Simulate touch with larger area for field work conditions
    const button = page.locator('[data-testid="voice-command-button"]');
    const box = await button.boundingBox();

    // Verify button meets minimum touch target size
    expect(box?.width).toBeGreaterThanOrEqual(56); // min touch target
    expect(box?.height).toBeGreaterThanOrEqual(56);

    // Tap with offset to simulate imprecise tap in challenging conditions
    await button.tap({ position: { x: 10, y: 10 } });

    // Verify action still triggered
    await expect(page.locator('.voice-listening')).toBeVisible();
  });
});
```

### Testing Best Practices

1. **Unit Tests:** Test individual components in isolation
   - Mock all dependencies (Store, Services, Ionic components)
   - Test component logic, not framework behavior
   - Focus on user interactions and state changes
   - Aim for 80%+ coverage of business logic

2. **Integration Tests:** Test component interactions
   - Test complete user flows within a feature
   - Mock external dependencies (APIs, Firebase)
   - Verify state management flows
   - Test offline/online transitions

3. **E2E Tests:** Test critical user flows (using Playwright)
   - Login → Create Job → Add Costs → Generate Report
   - Voice command flows
   - Offline mode operations
   - PWA installation and updates
   - Multi-device sync scenarios

4. **Coverage Goals:** Aim for 80% code coverage
   - 90%+ for critical business logic
   - 80%+ for components
   - 70%+ for effects/services
   - Focus on behavior, not line coverage

5. **Test Structure:** Arrange-Act-Assert pattern
   ```typescript
   it('should update job when save is clicked', () => {
     // Arrange
     const mockData = { title: 'Updated' };
     spyOn(store, 'dispatch');

     // Act
     component.save(mockData);

     // Assert
     expect(store.dispatch).toHaveBeenCalledWith(expectedAction);
   });
   ```

6. **Mock External Dependencies:** API calls, routing, state management
   - Use provideMockStore for NgRx
   - Create service spies with Jasmine
   - Mock Firestore operations
   - Simulate network conditions

### Testing Scripts

```json
// package.json
{
  "scripts": {
    "test": "nx test mobile-app",
    "test:watch": "nx test mobile-app --watch",
    "test:coverage": "nx test mobile-app --coverage",
    "test:ci": "nx test mobile-app --coverage --browsers=ChromeHeadless",
    "e2e": "nx e2e mobile-app-e2e",
    "e2e:ci": "nx e2e mobile-app-e2e --config=ci",
    "e2e:debug": "nx e2e mobile-app-e2e --debug",
    "test:all": "nx run-many --target=test --all --parallel"
  }
}
```

### Rationale for Testing Strategy

**Jest over Karma/Jasmine:**
- Faster test execution (important for CI/CD)
- Better watch mode for development
- Snapshot testing support
- Modern assertion library

**Playwright over Cypress:**
- Per explicit requirement (NOT Cypress)
- Better mobile device emulation
- Supports offline testing
- Native app testing capability

**Focus Areas:**
- Offline/online transitions (critical for field use)
- Voice command accuracy
- Touch-friendly interactions for challenging conditions
- State synchronization

## Environment Configuration

Here are the required environment variables for your Angular/Ionic PWA:

### Development Environment

```typescript
// apps/mobile-app/src/environments/environment.ts
export const environment = {
  production: false,
  appVersion: '1.0.0-dev',

  // Firebase Configuration
  firebase: {
    apiKey: 'AIzaSyDEVELOPMENT_KEY_HERE',
    authDomain: 'findogai-dev.firebaseapp.com',
    projectId: 'findogai-dev',
    storageBucket: 'findogai-dev.appspot.com',
    messagingSenderId: '123456789',
    appId: '1:123456789:web:abcdef123456',
    measurementId: 'G-DEVELOPMENT'
  },

  // Cloud Functions Endpoints
  functionsUrl: 'http://localhost:5001/findogai-dev/us-central1',
  functionsRegion: 'us-central1',

  // Voice Pipeline Configuration
  voice: {
    speechRecognitionLang: 'en-US',
    speechSynthesisVoice: 'Google US English',
    commandTimeout: 5000, // 5 seconds
    wakeWord: 'Hey FinDog', // Optional wake word
    llmProvider: 'openai', // or 'anthropic'
    llmApiKey: process.env['LLM_API_KEY'] || '', // Load from env
    llmModel: 'gpt-4o-mini' // or 'claude-3-haiku'
  },

  // Offline Configuration
  offline: {
    enablePersistence: true,
    cacheSizeBytes: 50 * 1024 * 1024, // 50MB
    syncInterval: 30000, // 30 seconds
    maxRetries: 3,
    retryDelay: 1000 // 1 second base delay
  },

  // PWA Configuration
  pwa: {
    enabled: true,
    updateCheckInterval: 3600000, // 1 hour
    updateMode: 'prompt', // or 'auto'
    offlineGoogleAnalytics: false
  },

  // Feature Flags
  features: {
    voiceCommands: true,
    offlineMode: true,
    pdfGeneration: false, // Phase 2
    advancedReporting: false, // Phase 2
    teamCollaboration: true,
    costTracking: true
  },

  // Logging
  logging: {
    level: 'debug', // 'error' | 'warn' | 'info' | 'debug'
    remoteLogging: false,
    logToConsole: true
  },

  // Development Tools
  devTools: {
    reduxDevTools: true,
    performanceMonitoring: true,
    showDebugInfo: true
  }
};
```

### Production Environment

```typescript
// apps/mobile-app/src/environments/environment.prod.ts
export const environment = {
  production: true,
  appVersion: '1.0.0',

  // Firebase Configuration (Production)
  firebase: {
    apiKey: process.env['FIREBASE_API_KEY'],
    authDomain: 'findogai.firebaseapp.com',
    projectId: 'findogai-prod',
    storageBucket: 'findogai-prod.appspot.com',
    messagingSenderId: process.env['FIREBASE_MESSAGING_SENDER_ID'],
    appId: process.env['FIREBASE_APP_ID'],
    measurementId: process.env['FIREBASE_MEASUREMENT_ID']
  },

  // Cloud Functions Endpoints (Production)
  functionsUrl: 'https://us-central1-findogai-prod.cloudfunctions.net',
  functionsRegion: 'us-central1',

  // Voice Pipeline Configuration
  voice: {
    speechRecognitionLang: 'en-US',
    speechSynthesisVoice: 'Google US English',
    commandTimeout: 5000,
    wakeWord: 'Hey FinDog',
    llmProvider: process.env['LLM_PROVIDER'] || 'openai',
    llmApiKey: process.env['LLM_API_KEY'],
    llmModel: process.env['LLM_MODEL'] || 'gpt-4o-mini'
  },

  // Offline Configuration
  offline: {
    enablePersistence: true,
    cacheSizeBytes: 100 * 1024 * 1024, // 100MB
    syncInterval: 60000, // 1 minute
    maxRetries: 5,
    retryDelay: 2000 // 2 seconds base delay
  },

  // PWA Configuration
  pwa: {
    enabled: true,
    updateCheckInterval: 7200000, // 2 hours
    updateMode: 'prompt',
    offlineGoogleAnalytics: true
  },

  // Feature Flags (Production)
  features: {
    voiceCommands: true,
    offlineMode: true,
    pdfGeneration: false, // Enable in Phase 2
    advancedReporting: false,
    teamCollaboration: true,
    costTracking: true
  },

  // Logging (Production)
  logging: {
    level: 'error',
    remoteLogging: true,
    logToConsole: false,
    sentryDsn: process.env['SENTRY_DSN'] // Error tracking
  },

  // Development Tools (Disabled in Production)
  devTools: {
    reduxDevTools: false,
    performanceMonitoring: true,
    showDebugInfo: false
  }
};
```

### Environment Variable Files (.env)

```bash
# .env.local (Development - Git ignored)
FIREBASE_API_KEY=your_dev_api_key_here
FIREBASE_AUTH_DOMAIN=findogai-dev.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-dev
FIREBASE_STORAGE_BUCKET=findogai-dev.appspot.com
FIREBASE_MESSAGING_SENDER_ID=123456789
FIREBASE_APP_ID=1:123456789:web:abcdef
FIREBASE_MEASUREMENT_ID=G-XXXXX

# LLM Configuration
LLM_PROVIDER=openai
LLM_API_KEY=sk-dev-xxxxxxxxxxxxx
LLM_MODEL=gpt-4o-mini

# Development URLs
API_URL=http://localhost:5001
APP_URL=http://localhost:4200

# Feature Flags
ENABLE_DEBUG_MODE=true
ENABLE_MOCK_DATA=true

# .env.production (Production - Secure storage)
FIREBASE_API_KEY=${SECRET_FIREBASE_API_KEY}
FIREBASE_AUTH_DOMAIN=findogai.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-prod
FIREBASE_STORAGE_BUCKET=findogai-prod.appspot.com
FIREBASE_MESSAGING_SENDER_ID=${SECRET_FIREBASE_SENDER_ID}
FIREBASE_APP_ID=${SECRET_FIREBASE_APP_ID}
FIREBASE_MEASUREMENT_ID=${SECRET_MEASUREMENT_ID}

# LLM Configuration
LLM_PROVIDER=openai
LLM_API_KEY=${SECRET_LLM_API_KEY}
LLM_MODEL=gpt-4o-mini

# Production URLs
API_URL=https://api.findogai.com
APP_URL=https://app.findogai.com

# Error Tracking
SENTRY_DSN=${SECRET_SENTRY_DSN}

# Analytics
GA_TRACKING_ID=${SECRET_GA_TRACKING_ID}
```

### Build Configuration

```json
// apps/mobile-app/project.json (Nx configuration)
{
  "name": "mobile-app",
  "targets": {
    "build": {
      "configurations": {
        "development": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.ts"
            }
          ],
          "optimization": false,
          "sourceMap": true,
          "namedChunks": true
        },
        "production": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.prod.ts"
            }
          ],
          "optimization": true,
          "sourceMap": false,
          "namedChunks": false,
          "serviceWorker": true,
          "ngswConfigPath": "apps/mobile-app/ngsw-config.json"
        },
        "staging": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.staging.ts"
            }
          ],
          "optimization": true,
          "sourceMap": true
        }
      }
    },
    "serve": {
      "configurations": {
        "development": {
          "buildTarget": "mobile-app:build:development",
          "hmr": true,
          "port": 4200
        },
        "production": {
          "buildTarget": "mobile-app:build:production",
          "port": 4200
        }
      }
    }
  }
}
```

### Environment Usage in Code

```typescript
// Example: Using environment variables in a service
import { Injectable } from '@angular/core';
import { environment } from '@environments/environment';

@Injectable({ providedIn: 'root' })
export class ConfigService {
  get isProduction(): boolean {
    return environment.production;
  }

  get firebaseConfig() {
    return environment.firebase;
  }

  get isVoiceEnabled(): boolean {
    return environment.features.voiceCommands;
  }

  get llmApiKey(): string {
    if (!environment.voice.llmApiKey) {
      throw new Error('LLM API key not configured');
    }
    return environment.voice.llmApiKey;
  }

  get offlineConfig() {
    return environment.offline;
  }

  shouldShowDebugInfo(): boolean {
    return !environment.production && environment.devTools.showDebugInfo;
  }
}
```

### Capacitor Environment Configuration

```typescript
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.findogai.app',
  appName: 'FinDogAI',
  webDir: 'dist/apps/mobile-app',
  bundledWebRuntime: false,
  server: {
    // Development server config
    url: process.env['CAP_SERVER_URL'], // For live reload
    cleartext: true // Allow HTTP in development
  },
  plugins: {
    App: {
      iosScheme: 'findogai',
      androidScheme: 'findogai'
    },
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#3880ff',
      showSpinner: true,
      spinnerColor: '#ffffff'
    },
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert']
    }
  },
  ios: {
    contentInset: 'automatic',
    limitsNavigationBarChanges: false
  },
  android: {
    minWebViewVersion: 80,
    enableJetifier: true
  }
};

export default config;
```

### Rationale for Environment Configuration

**Environment Separation:**
- Clear distinction between dev/staging/prod
- Secure handling of sensitive keys
- Feature flag control per environment

**Voice Pipeline Configuration:**
- Configurable LLM providers for cost optimization
- Language settings for internationalization
- Timeout controls for poor connectivity

**Offline Settings:**
- Tunable cache sizes for different devices
- Configurable sync intervals
- Retry strategies per environment

**PWA Configuration:**
- Update strategies (prompt vs auto)
- Analytics integration
- Service worker control

## Frontend Developer Standards

### Code Quality Standards

1. **TypeScript Strict Mode**
   ```json
   // tsconfig.base.json
   {
     "compilerOptions": {
       "strict": true,
       "noImplicitAny": true,
       "noImplicitReturns": true,
       "noFallthroughCasesInSwitch": true,
       "noUnusedLocals": true,
       "noUnusedParameters": true,
       "strictNullChecks": true,
       "strictFunctionTypes": true,
       "strictBindCallApply": true,
       "strictPropertyInitialization": true
     }
   }
   ```

2. **ESLint Configuration**
   ```json
   // .eslintrc.json
   {
     "extends": [
       "eslint:recommended",
       "plugin:@typescript-eslint/recommended",
       "plugin:@angular-eslint/recommended"
     ],
     "rules": {
       "@typescript-eslint/explicit-function-return-type": "error",
       "@typescript-eslint/no-explicit-any": "error",
       "no-console": ["error", { "allow": ["warn", "error"] }],
       "max-len": ["error", { "code": 120 }]
     }
   }
   ```

3. **Prettier Configuration**
   ```json
   // .prettierrc
   {
     "singleQuote": true,
     "trailingComma": "none",
     "arrowParens": "avoid",
     "printWidth": 120,
     "tabWidth": 2,
     "semi": true
   }
   ```

### Development Workflow

1. **Branch Naming Convention**
   - `feature/JIRA-123-description`
   - `bugfix/JIRA-456-description`
   - `hotfix/critical-issue`
   - `chore/update-dependencies`

2. **Commit Message Format**
   ```
   <type>(<scope>): <subject>

   <body>

   <footer>
   ```
   Types: feat, fix, docs, style, refactor, test, chore

3. **Pull Request Template**
   ```markdown
   ## Description
   Brief description of changes

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Breaking change
   - [ ] Documentation update

   ## Testing
   - [ ] Unit tests pass
   - [ ] E2E tests pass
   - [ ] Manual testing completed

   ## Checklist
   - [ ] Code follows style guidelines
   - [ ] Self-review completed
   - [ ] Comments added for complex code
   - [ ] Documentation updated
   ```

### Performance Guidelines

1. **Bundle Size Optimization**
   - Use lazy loading for all feature modules
   - Implement code splitting
   - Tree-shake unused imports
   - Monitor bundle size with webpack-bundle-analyzer

2. **Runtime Performance**
   - Use OnPush change detection strategy
   - Implement virtual scrolling for long lists
   - Use trackBy functions in *ngFor
   - Debounce user input events
   - Memoize expensive computations

3. **PWA Optimization**
   - Precache critical assets
   - Implement proper cache strategies
   - Optimize images (WebP format)
   - Minimize initial load time

### Accessibility Standards

1. **WCAG 2.1 Level AA Compliance**
   - All interactive elements keyboard accessible
   - Proper ARIA labels and roles
   - Color contrast ratios meet standards
   - Focus indicators visible

2. **Screen Reader Support**
   - Semantic HTML elements
   - Proper heading hierarchy
   - Alt text for images
   - Form labels associated with inputs

3. **Mobile Accessibility**
   - Touch targets minimum 48x48px
   - Gesture alternatives provided
   - Orientation support (portrait/landscape)
   - Font scaling support

### Security Best Practices

1. **Data Protection**
   - Never store sensitive data in localStorage
   - Use secure Firebase rules
   - Implement proper authentication checks
   - Sanitize user input

2. **API Security**
   - Always use HTTPS
   - Implement CSRF protection
   - Add request rate limiting
   - Validate all inputs

3. **Code Security**
   - Regular dependency updates
   - Security audit with `npm audit`
   - No hardcoded credentials
   - Use environment variables for secrets

### Documentation Standards

1. **Code Documentation**
   ```typescript
   /**
    * Processes voice command and updates application state
    * @param transcript - The text from speech recognition
    * @returns Promise with command result
    * @throws {VoiceCommandError} If command cannot be processed
    */
   async processVoiceCommand(transcript: string): Promise<CommandResult> {
     // Implementation
   }
   ```

2. **Component Documentation**
   - Purpose and usage
   - Input/Output properties
   - Events emitted
   - Example usage

3. **README Files**
   - Setup instructions
   - Development workflow
   - Testing procedures
   - Deployment process

### Monitoring and Logging

1. **Error Tracking**
   - Sentry integration for production
   - Structured error logging
   - User context in error reports
   - Performance monitoring

2. **Analytics**
   - Google Analytics for user behavior
   - Custom events for key actions
   - Performance metrics tracking
   - Offline usage statistics

3. **Debug Logging**
   ```typescript
   import { LogLevel } from '@core/services/logger.service';

   logger.debug('Component initialized', { componentName: 'JobDetail' });
   logger.info('Job saved successfully', { jobId: job.id });
   logger.error('Failed to sync data', { error, context });
   ```

### Team Collaboration

1. **Code Reviews**
   - All code must be reviewed
   - At least one approval required
   - Focus on functionality, performance, security
   - Constructive feedback

2. **Knowledge Sharing**
   - Weekly tech talks
   - Documentation updates
   - Pair programming for complex features
   - Maintain decision log

3. **Communication**
   - Daily standup updates
   - Sprint planning participation
   - Retrospective feedback
   - Slack/Teams for async communication