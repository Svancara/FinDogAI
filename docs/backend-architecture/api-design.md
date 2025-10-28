[Back to Index](./index.md)

# API Design and Connectivity

This document covers Firebase SDK integration, real-time listeners, offline persistence, and connectivity patterns for the mobile application.

## 5.1 Firebase SDK Integration

### 5.1.1 AngularFire Configuration

Complete Firebase initialization in Angular application:

```typescript
// packages/mobile-app/src/app/app.config.ts

import { provideFirebaseApp, initializeApp } from '@angular/fire/app';
import { provideAuth, getAuth } from '@angular/fire/auth';
import { provideFirestore, getFirestore, enableIndexedDbPersistence } from '@angular/fire/firestore';
import { provideFunctions, getFunctions } from '@angular/fire/functions';
import { provideStorage, getStorage } from '@angular/fire/storage';

export const appConfig: ApplicationConfig = {
  providers: [
    provideFirebaseApp(() => initializeApp(environment.firebase)),
    provideAuth(() => getAuth()),
    provideFirestore(() => {
      const firestore = getFirestore();
      enableIndexedDbPersistence(firestore).catch((err) => {
        if (err.code === 'failed-precondition') {
          console.warn('Offline persistence failed: multiple tabs open');
        } else if (err.code === 'unimplemented') {
          console.warn('Offline persistence not supported by browser');
        }
      });
      return firestore;
    }),
    provideFunctions(() => getFunctions(undefined, 'europe-west1')),
    provideStorage(() => getStorage()),
    // ... other providers
  ]
};
```

### 5.1.2 Environment Configuration

**Development Environment:**

```typescript
// packages/mobile-app/src/environments/environment.ts

export const environment = {
  production: false,
  firebase: {
    apiKey: 'AIza...',
    authDomain: 'findogai-dev.firebaseapp.com',
    projectId: 'findogai-dev',
    storageBucket: 'findogai-dev.appspot.com',
    messagingSenderId: '123456789',
    appId: '1:123456789:web:abcdef'
  },
  voiceProviders: {
    stt: 'mock', // 'google-cloud' | 'openai-whisper' | 'mock'
    llm: 'mock', // 'openai-gpt4' | 'groq-llama3' | 'ollama' | 'mock'
    tts: 'platform-native' // 'google-cloud' | 'platform-native' | 'mock'
  },
  apiKeys: {
    googleCloud: '',
    openai: '',
    groq: ''
  }
};
```

**Production Environment:**

```typescript
// packages/mobile-app/src/environments/environment.prod.ts

export const environment = {
  production: true,
  firebase: {
    apiKey: 'AIza...',
    authDomain: 'findogai.firebaseapp.com',
    projectId: 'findogai-prod',
    storageBucket: 'findogai-prod.appspot.com',
    messagingSenderId: '987654321',
    appId: '1:987654321:web:fedcba'
  },
  voiceProviders: {
    stt: 'google-cloud',
    llm: 'openai-gpt4',
    tts: 'google-cloud'
  },
  apiKeys: {
    googleCloud: process.env['GOOGLE_CLOUD_API_KEY'],
    openai: process.env['OPENAI_API_KEY'],
    groq: ''
  }
};
```

---

## 5.2 Cloud Function Invocation

### 5.2.1 Firestore Triggers

Firestore triggers run automatically on document changes. No client-side code needed.

**Trigger Types:**
- `onDocumentCreated`: Fires when new document is written
- `onDocumentUpdated`: Fires when document is modified
- `onDocumentDeleted`: Fires when document is removed
- `onDocumentWritten`: Fires on create, update, or delete

**Example: Audit Logging on Job Creation**

```typescript
import { onDocumentCreated } from 'firebase-functions/v2/firestore';

export const onJobCreated = onDocumentCreated(
  {
    document: 'tenants/{tenantId}/jobs/{jobId}',
    region: 'europe-west1'
  },
  async (event) => {
    const snapshot = event.data;
    const { tenantId, jobId } = event.params;

    // Write audit log
    await event.firestore
      .collection(`tenants/${tenantId}/audit_logs`)
      .add({
        operation: 'CREATE',
        collection: 'jobs',
        documentId: jobId,
        tenantId,
        timestamp: event.time,
        authorId: snapshot?.data()?.createdBy,
        after: snapshot?.data(),
        ttl: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000)
      });
  }
);
```

### 5.2.2 HTTPS Callable Functions

**Client-Side Invocation:**

```typescript
// packages/mobile-app/src/app/services/sequence.service.ts

import { Injectable, inject } from '@angular/core';
import { Functions, httpsCallable } from '@angular/fire/functions';

@Injectable({ providedIn: 'root' })
export class SequenceService {
  private functions = inject(Functions);

  async allocateJobNumber(tenantId: string): Promise<number> {
    const callable = httpsCallable<AllocateSequenceRequest, AllocateSequenceResponse>(
      this.functions,
      'allocateSequence'
    );

    const result = await callable({ tenantId, type: 'jobNumber' });
    return result.data.sequenceNumber;
  }

  async allocateOrdinalNumber(tenantId: string, jobId: string): Promise<number> {
    const callable = httpsCallable<AllocateSequenceRequest, AllocateSequenceResponse>(
      this.functions,
      'allocateSequence'
    );

    const result = await callable({ tenantId, type: 'ordinalNumber', jobId });
    return result.data.sequenceNumber;
  }
}

interface AllocateSequenceRequest {
  tenantId: string;
  type: 'jobNumber' | 'vehicleNumber' | 'machineNumber' | 'teamMemberNumber' | 'ordinalNumber';
  jobId?: string;
}

interface AllocateSequenceResponse {
  sequenceNumber: number;
}
```

**Error Handling:**

```typescript
try {
  const jobNumber = await this.sequenceService.allocateJobNumber(tenantId);
  console.log('Allocated job number:', jobNumber);
} catch (error: any) {
  if (error.code === 'unauthenticated') {
    console.error('User not authenticated');
  } else if (error.code === 'permission-denied') {
    console.error('Insufficient permissions');
  } else {
    console.error('Function error:', error.message);
  }
}
```

---

## 5.3 Real-Time Listeners

### 5.3.1 Collection Queries with Real-Time Updates

**Job Service Example:**

```typescript
// packages/mobile-app/src/app/services/job.service.ts

import { Injectable, inject } from '@angular/core';
import { Firestore, collection, query, where, orderBy, collectionData } from '@angular/fire/firestore';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class JobService {
  private firestore = inject(Firestore);

  getActiveJobs(tenantId: string): Observable<Job[]> {
    const jobsRef = collection(this.firestore, `tenants/${tenantId}/jobs`);
    const activeQuery = query(
      jobsRef,
      where('status', '==', 'active'),
      orderBy('updatedAt', 'desc')
    );

    return collectionData(activeQuery, { idField: 'id' }) as Observable<Job[]>;
  }

  getJobCosts(tenantId: string, jobId: string): Observable<Cost[]> {
    const costsRef = collection(this.firestore, `tenants/${tenantId}/jobs/${jobId}/costs`);
    const costsQuery = query(costsRef, orderBy('date', 'desc'));

    return collectionData(costsQuery, { idField: 'id' }) as Observable<Cost[]>;
  }
}
```

**Usage in Component:**

```typescript
export class JobListComponent implements OnInit {
  jobs$ = this.jobService.getActiveJobs(this.tenantId);

  ngOnInit() {
    // Observable automatically updates UI on changes
    this.jobs$.subscribe(jobs => {
      console.log('Jobs updated:', jobs);
    });
  }
}
```

### 5.3.2 Document Snapshots

**Real-time sync for business profile:**

```typescript
import { doc, docData } from '@angular/fire/firestore';

getBusinessProfile(tenantId: string): Observable<BusinessProfile> {
  const profileRef = doc(this.firestore, `tenants/${tenantId}/businessProfile`);
  return docData(profileRef) as Observable<BusinessProfile>;
}
```

**Snapshot Metadata:**

```typescript
import { doc, docSnapshots } from '@angular/fire/firestore';

getJobWithMetadata(tenantId: string, jobId: string) {
  const jobRef = doc(this.firestore, `tenants/${tenantId}/jobs/${jobId}`);

  return docSnapshots(jobRef).pipe(
    map(snapshot => ({
      data: snapshot.data(),
      exists: snapshot.exists(),
      fromCache: snapshot.metadata.fromCache,
      hasPendingWrites: snapshot.metadata.hasPendingWrites
    }))
  );
}
```

---

## 5.4 Offline Persistence Configuration

### 5.4.1 Persistence Settings

**Enable offline persistence (IndexedDB):**

```typescript
import { enableIndexedDbPersistence } from '@angular/fire/firestore';

const firestore = getFirestore();
enableIndexedDbPersistence(firestore, {
  synchronizeTabs: true // Sync across browser tabs
}).catch((err) => {
  if (err.code === 'failed-precondition') {
    console.warn('Multiple tabs open; persistence enabled in first tab only');
  } else if (err.code === 'unimplemented') {
    console.warn('Browser does not support offline persistence');
  }
});
```

**Multi-Tab Behavior:**
- `synchronizeTabs: true`: Share cache across tabs (default)
- `synchronizeTabs: false`: Independent caches per tab

### 5.4.2 Offline Behavior

**Write Operations:**
- Writes succeed immediately against local cache
- Queued automatically for server sync
- `hasPendingWrites` metadata tracks sync status

**Read Operations:**
- Reads always succeed from cache if data exists
- `fromCache` metadata indicates source
- Listeners receive updates when syncing completes

**Example: Pending Writes Detection**

```typescript
import { doc, onSnapshot } from '@angular/fire/firestore';

const jobRef = doc(firestore, `tenants/${tenantId}/jobs/${jobId}`);

onSnapshot(jobRef, (snapshot) => {
  const data = snapshot.data();
  const isPending = snapshot.metadata.hasPendingWrites;
  const isFromCache = snapshot.metadata.fromCache;

  console.log('Job data:', data);
  console.log('Pending sync:', isPending);
  console.log('Offline mode:', isFromCache);

  // Show sync indicator in UI
  if (isPending) {
    this.showSyncIndicator();
  }
});
```

**Sync Error Handling:**

```typescript
import { enableNetwork, disableNetwork } from '@angular/fire/firestore';

// Manually disable network (for testing)
await disableNetwork(firestore);

// Re-enable network
await enableNetwork(firestore);

// Listen for sync errors
import { onSnapshot } from '@angular/fire/firestore';

onSnapshot(
  query,
  (snapshot) => {
    // Success callback
    console.log('Data synced:', snapshot.docs);
  },
  (error) => {
    // Error callback
    console.error('Sync error:', error);
    this.showSyncError(error.message);
  }
);
```

---

## Data Write Patterns

### Create with Auto-ID

```typescript
import { addDoc, collection, serverTimestamp } from '@angular/fire/firestore';

async createJob(tenantId: string, jobData: Partial<Job>) {
  const jobsRef = collection(this.firestore, `tenants/${tenantId}/jobs`);

  const docRef = await addDoc(jobsRef, {
    ...jobData,
    tenantId,
    createdAt: serverTimestamp(),
    createdBy: this.auth.currentUser.uid,
    updatedAt: serverTimestamp(),
    updatedBy: this.auth.currentUser.uid
  });

  return docRef.id;
}
```

### Create with Custom ID

```typescript
import { doc, setDoc, serverTimestamp } from '@angular/fire/firestore';

async createJobWithId(tenantId: string, jobId: string, jobData: Partial<Job>) {
  const jobRef = doc(this.firestore, `tenants/${tenantId}/jobs/${jobId}`);

  await setDoc(jobRef, {
    ...jobData,
    tenantId,
    createdAt: serverTimestamp(),
    createdBy: this.auth.currentUser.uid,
    updatedAt: serverTimestamp(),
    updatedBy: this.auth.currentUser.uid
  });
}
```

### Update Document

```typescript
import { doc, updateDoc, serverTimestamp } from '@angular/fire/firestore';

async updateJob(tenantId: string, jobId: string, updates: Partial<Job>) {
  const jobRef = doc(this.firestore, `tenants/${tenantId}/jobs/${jobId}`);

  await updateDoc(jobRef, {
    ...updates,
    updatedAt: serverTimestamp(),
    updatedBy: this.auth.currentUser.uid
  });
}
```

### Soft Delete (Archive)

```typescript
async archiveJob(tenantId: string, jobId: string) {
  const jobRef = doc(this.firestore, `tenants/${tenantId}/jobs/${jobId}`);

  await updateDoc(jobRef, {
    status: 'archived',
    archivedAt: serverTimestamp(),
    archivedBy: this.auth.currentUser.uid,
    updatedAt: serverTimestamp(),
    updatedBy: this.auth.currentUser.uid
  });
}
```

---

## Batch Operations

### Batch Writes

```typescript
import { writeBatch, doc } from '@angular/fire/firestore';

async createMultipleCosts(tenantId: string, jobId: string, costs: Partial<Cost>[]) {
  const batch = writeBatch(this.firestore);

  costs.forEach(cost => {
    const costRef = doc(collection(this.firestore, `tenants/${tenantId}/jobs/${jobId}/costs`));
    batch.set(costRef, {
      ...cost,
      tenantId,
      createdAt: serverTimestamp(),
      createdBy: this.auth.currentUser.uid,
      updatedAt: serverTimestamp(),
      updatedBy: this.auth.currentUser.uid
    });
  });

  await batch.commit();
}
```

**Batch Limits:**
- Maximum 500 operations per batch
- All operations succeed or all fail (atomic)

---

## Transaction Patterns

### Atomic Counter Increment

```typescript
import { runTransaction, doc } from '@angular/fire/firestore';

async incrementCounter(tenantId: string, counterType: string) {
  const counterRef = doc(this.firestore, `sequences/${tenantId}/counters/${counterType}`);

  const newCount = await runTransaction(this.firestore, async (transaction) => {
    const counterDoc = await transaction.get(counterRef);
    const currentCount = counterDoc.exists() ? counterDoc.data()['current'] : 0;
    const newCount = currentCount + 1;

    transaction.set(counterRef, { current: newCount }, { merge: true });

    return newCount;
  });

  return newCount;
}
```

**Transaction Limits:**
- Maximum 500 document reads
- Automatic retry on contention
- Use Cloud Functions for high-contention counters

---

## Query Optimization

### Use Composite Indexes

```typescript
// Requires composite index: tenantId (ASC) + status (ASC) + updatedAt (DESC)
const activeJobsQuery = query(
  collection(this.firestore, `tenants/${tenantId}/jobs`),
  where('tenantId', '==', tenantId),
  where('status', '==', 'active'),
  orderBy('updatedAt', 'desc'),
  limit(50)
);
```

### Limit Results

```typescript
import { limit } from '@angular/fire/firestore';

// Always use limit to reduce reads
const recentJobs = query(
  collection(this.firestore, `tenants/${tenantId}/jobs`),
  orderBy('createdAt', 'desc'),
  limit(20)
);
```

### Pagination with startAfter

```typescript
async loadMoreJobs(lastJob: Job) {
  const nextPage = query(
    collection(this.firestore, `tenants/${this.tenantId}/jobs`),
    orderBy('createdAt', 'desc'),
    startAfter(lastJob.createdAt),
    limit(20)
  );

  const snapshot = await getDocs(nextPage);
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}
```

See [Performance Optimization](./performance-optimization.md) for advanced query optimization strategies.

---

[← Back: Security Rules](./security-rules.md) | [Back to Index](./index.md) | [Next: Technology Stack →](./technology-stack.md)
