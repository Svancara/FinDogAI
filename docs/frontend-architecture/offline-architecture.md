[Back to Index](./index.md)

# Offline Architecture

The FinDogAI application is built with an offline-first architecture, ensuring full functionality without internet connectivity. This is critical for field workers who may operate in areas with poor or no network coverage.

## User Identity in Offline Operations

All documents that are created or modified (both online and offline) must include compound identity objects for audit tracking:

```typescript
// Compound identity object for multi-tenant user attribution
export interface UserIdentity {
  uid: string;              // Firebase Auth UID for security checks
  memberNumber: number;     // Tenant-specific identifier (voice-friendly)
  displayName: string;      // Cached display name for audit display
}
```

When creating or updating documents offline, the application must:
1. Retrieve the cached user identity from auth state (loaded at login)
2. Include the full compound identity object in `createdBy` and `updatedBy` fields
3. Ensure the identity is preserved through the sync queue when the operation is replayed online

## Network Status Service

The Network Status Service monitors the application's connectivity status and provides real-time updates.

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

### Usage in Components

```typescript
import { Component, inject, signal } from '@angular/core';
import { NetworkService } from '@core/services/network.service';

@Component({
  selector: 'app-jobs-list',
  template: `
    @if (!isOnline()) {
      <div class="offline-banner">
        <ion-icon name="cloud-offline"></ion-icon>
        <span>Offline Mode - Changes will sync when connected</span>
      </div>
    }

    <!-- Rest of template -->
  `
})
export class JobsListComponent {
  private readonly network = inject(NetworkService);
  protected readonly isOnline = this.network.isOnline;
}
```

## Sync Queue Service

The Sync Queue Service manages offline operations and synchronizes them when connectivity is restored.

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

## Firestore Sync Service

The Firestore Sync Service handles real-time synchronization with conflict resolution.

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

  async updateDocument(path: string, data: any, userIdentity: UserIdentity): Promise<void> {
    const docRef = doc(this.firestore, path);

    // Firestore automatically handles conflicts with last-write-wins
    // Server timestamp ensures consistent ordering
    await setDoc(docRef, {
      ...data,
      updatedAt: serverTimestamp(), // Server timestamp for ordering
      updatedBy: userIdentity,      // Compound identity for audit trail
      // Firestore's offline persistence handles the queue
      // When online, changes are synchronized automatically
      // Conflicts resolved by server timestamp (last write wins)
    }, { merge: true });
  }

  // For operations requiring strict consistency, use transactions
  async updateWithTransaction(
    path: string,
    updateFn: (data: any) => any,
    userIdentity: UserIdentity
  ): Promise<void> {
    const docRef = doc(this.firestore, path);

    return runTransaction(this.firestore, async (transaction) => {
      const doc = await transaction.get(docRef);
      if (!doc.exists()) {
        throw new Error('Document does not exist');
      }

      const newData = updateFn(doc.data());
      transaction.update(docRef, {
        ...newData,
        updatedAt: serverTimestamp(),
        updatedBy: userIdentity  // Include compound identity
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

## Offline UI Components

### Offline Banner Component

```typescript
// shared/ui/offline-banner/offline-banner.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonicModule } from '@ionic/angular';
import { NetworkService } from '@core/services/network.service';
import { SyncQueueService } from '@core/services/sync-queue.service';

@Component({
  selector: 'app-offline-banner',
  standalone: true,
  imports: [CommonModule, IonicModule],
  template: `
    @if (!isOnline()) {
      <div class="offline-banner">
        <ion-icon name="cloud-offline"></ion-icon>
        <span class="offline-banner__text">
          Offline Mode
          @if (pendingCount$ | async; as count) {
            @if (count > 0) {
              <ion-badge color="warning">{{ count }} pending</ion-badge>
            }
          }
        </span>
      </div>
    }
  `,
  styles: [`
    .offline-banner {
      background: var(--offline-bg);
      border-bottom: 1px solid var(--offline-border);
      color: var(--offline-text);
      padding: var(--spacing-sm) var(--spacing-md);
      display: flex;
      align-items: center;
      gap: var(--spacing-sm);
      position: sticky;
      top: 0;
      z-index: var(--z-sticky);

      &__text {
        flex: 1;
        display: flex;
        align-items: center;
        gap: var(--spacing-sm);
      }

      ion-icon {
        font-size: 20px;
      }
    }
  `]
})
export class OfflineBannerComponent {
  private readonly network = inject(NetworkService);
  private readonly syncQueue = inject(SyncQueueService);

  protected readonly isOnline = this.network.isOnline;
  protected readonly pendingCount$ = this.syncQueue.pendingCount$;
}
```

### Sync Status Indicator

```typescript
// shared/ui/sync-status/sync-status.component.ts
import { Component, Input } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonicModule } from '@ionic/angular';

@Component({
  selector: 'app-sync-status',
  standalone: true,
  imports: [CommonModule, IonicModule],
  template: `
    <div class="sync-status" [class]="'sync-status--' + status">
      <div class="sync-status__indicator"></div>
      <span class="sync-status__label">
        @switch (status) {
          @case ('synced') { Synced }
          @case ('pending') { Pending }
          @case ('error') { Error }
        }
      </span>
    </div>
  `,
  styles: [`
    .sync-status {
      display: inline-flex;
      align-items: center;
      gap: var(--spacing-xs);
      font-size: var(--font-size-sm);

      &__indicator {
        width: 8px;
        height: 8px;
        border-radius: var(--radius-full);
      }

      &--synced .sync-status__indicator {
        background: var(--app-success);
      }

      &--pending .sync-status__indicator {
        background: var(--app-warning);
        animation: pulse 1.5s infinite;
      }

      &--error .sync-status__indicator {
        background: var(--app-danger);
      }
    }
  `]
})
export class SyncStatusComponent {
  @Input() status: 'synced' | 'pending' | 'error' = 'synced';
}
```

## Optimistic UI Updates

### Pattern for Optimistic Updates

```typescript
// Example: Optimistic job update with compound identity
export class JobDetailComponent {
  private readonly store = inject(Store);
  private readonly authState$ = this.store.select(selectAuthState);

  protected updateJob(id: string, changes: Partial<Job>): void {
    // Get current user identity from auth state
    this.authState$.pipe(take(1)).subscribe(authState => {
      if (!authState.user?.identity) {
        console.error('User identity not available');
        return;
      }

      // Include updatedBy in the changes
      const changesWithIdentity = {
        ...changes,
        updatedBy: authState.user.identity,
        updatedAt: Timestamp.now() // Will be replaced by serverTimestamp
      };

      // 1. Immediately update UI (optimistic)
      this.store.dispatch(JobsActions.updateJobOptimistic({
        id,
        changes: changesWithIdentity
      }));

      // 2. Attempt to sync with backend
      this.store.dispatch(JobsActions.updateJob({
        id,
        changes: changesWithIdentity
      }));

      // 3. Effects handle success/failure
      // - Success: Mark as synced
      // - Failure: Rollback or mark as error
    });
  }
}
```

## Offline Testing

### Simulating Offline Mode

```typescript
// Development helper for testing offline scenarios
export class OfflineTestHelper {
  static goOffline(): void {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.controller?.postMessage({
        type: 'SIMULATE_OFFLINE'
      });
    }
    // Trigger offline event
    window.dispatchEvent(new Event('offline'));
  }

  static goOnline(): void {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.controller?.postMessage({
        type: 'SIMULATE_ONLINE'
      });
    }
    // Trigger online event
    window.dispatchEvent(new Event('online'));
  }
}
```

## Best Practices

### 1. Offline-First Mindset
- Assume offline by default
- Sync in background when online
- Never block user interactions

### 2. Clear Communication
- Show offline status prominently
- Indicate pending syncs
- Provide feedback on sync errors

### 3. Data Management
- Queue all mutations when offline
- Include compound identity objects in all create/update operations
- Preserve user identity through sync queue operations
- Retry failed operations
- Handle conflicts gracefully (last-write-wins based on server timestamp)

### 4. Performance
- Minimize sync operations
- Batch updates when possible
- Use efficient data structures

### 5. Error Handling
- Log sync failures
- Notify users of persistent errors
- Provide manual retry options

---

[← Previous: Styling Guidelines](./styling-guidelines.md) | [Back to Index](./index.md) | [Next: Testing Strategy →](./testing-strategy.md)
