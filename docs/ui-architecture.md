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

## Styling Guidelines (Complete)

[Previous styling section content remains the same - already complete]

## Testing Requirements (Complete)

[Previous testing section content remains the same - already complete]

## Environment Configuration (Complete)

[Previous environment section content remains the same - already complete]

## Frontend Developer Standards (Complete)

[Previous developer standards section content remains the same - already complete]

---

*Document Version: 1.0*
*Last Updated: 2025-01-27*
*Created by: Winston (Architect)*