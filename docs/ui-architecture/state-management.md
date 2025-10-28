[Back to Index](./index.md)

# State Management with NgRx

## Complete NgRx State Implementation

This section provides a comprehensive example of NgRx state management for the Jobs feature, including offline support and optimistic updates.

## State Interface

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
```

## Actions

```typescript
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
```

## Reducer

```typescript
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
```

## Effects

```typescript
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
```

## Selectors

```typescript
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

## Store Configuration

```typescript
// app/store/app.state.ts - Root state
import { JobsState } from '@features/jobs/data-access/jobs.state';
import { CostsState } from '@features/costs/data-access/costs.state';
import { VoiceState } from '@features/voice/data-access/voice.state';

export interface AppState {
  jobs: JobsState;
  costs: CostsState;
  voice: VoiceState;
  // ... other feature states
}

// app.config.ts - Configure NgRx store
import { ApplicationConfig } from '@angular/core';
import { provideStore } from '@ngrx/store';
import { provideEffects } from '@ngrx/effects';
import { provideStoreDevtools } from '@ngrx/store-devtools';
import { jobsReducer } from '@features/jobs/data-access/jobs.reducer';
import { JobsEffects } from '@features/jobs/data-access/jobs.effects';
import { environment } from '@environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    provideStore({
      jobs: jobsReducer,
      // ... other feature reducers
    }),
    provideEffects([
      JobsEffects,
      // ... other effects
    ]),
    provideStoreDevtools({
      maxAge: 25,
      logOnly: environment.production,
      autoPause: true,
      trace: !environment.production
    }),
    // ... other providers
  ]
};
```

## Usage in Components

```typescript
// Component using NgRx store
import { Component, inject, signal } from '@angular/core';
import { Store } from '@ngrx/store';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { JobsActions } from './data-access/jobs.actions';
import { selectAllJobs, selectJobsLoading } from './data-access/jobs.selectors';

@Component({
  selector: 'app-jobs-list',
  standalone: true,
  template: `
    @if (loading()) {
      <ion-spinner></ion-spinner>
    } @else {
      @for (job of jobs(); track job.id) {
        <app-job-card
          [job]="job"
          (selected)="onJobSelected($event)"
        ></app-job-card>
      }
    }
  `
})
export class JobsListComponent {
  private readonly store = inject(Store);

  protected readonly jobs = signal<Job[]>([]);
  protected readonly loading = signal(false);

  constructor() {
    // Subscribe to store
    this.store.select(selectAllJobs).pipe(
      takeUntilDestroyed()
    ).subscribe(jobs => this.jobs.set(jobs));

    this.store.select(selectJobsLoading).pipe(
      takeUntilDestroyed()
    ).subscribe(loading => this.loading.set(loading));

    // Dispatch load action
    this.store.dispatch(JobsActions.loadJobs());
  }

  protected onJobSelected(job: Job): void {
    this.store.dispatch(JobsActions.selectJob({ id: job.id }));
  }

  protected updateJob(id: string, changes: Partial<Job>): void {
    this.store.dispatch(JobsActions.updateJob({ id, changes }));
  }
}
```

## NgRx Best Practices

### 1. Action Naming
- Use descriptive, verb-based names
- Group related actions with `createActionGroup`
- Include Success/Failure pairs for async operations

### 2. State Structure
- Keep state flat and normalized
- Use entity adapters for collections
- Include UI state (loading, error) with data

### 3. Selectors
- Use memoized selectors
- Create derived selectors for computed data
- Keep selectors pure (no side effects)

### 4. Effects
- One effect per action type
- Handle errors gracefully
- Use appropriate operators (switchMap, mergeMap, etc.)

### 5. Testing
- Test reducers with known inputs/outputs
- Test effects with mock actions and services
- Test selectors with mock state

---

[← Previous: Component Standards](./component-standards.md) | [Back to Index](./index.md) | [Next: API Integration →](./api-integration.md)
