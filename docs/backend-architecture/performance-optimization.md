[Back to Index](./index.md)

# Performance Optimization

This document covers query optimization, cold start mitigation, caching strategies, and cost optimization for the FinDogAI backend.

## Overview

Performance targets (from NFRs):
- **NFR14**: Backend API response time <500ms (excluding network latency)
- **NFR3**: App launch time <2 seconds with cached data
- **NFR5**: Offline-first architecture with automatic sync

---

## 9.1 Firestore Query Optimization

### 9.1.1 Efficient Query Patterns

**Use Composite Indexes:**

```typescript
// Requires composite index: status (ASC) + updatedAt (DESC)
const activeJobsQuery = query(
  collection(firestore, `tenants/${tenantId}/jobs`),
  where('status', '==', 'active'),
  orderBy('updatedAt', 'desc'),
  limit(20)
);
```

**Always Use Limit:**

```typescript
// Bad: Fetches all documents
const allJobs = await getDocs(collection(firestore, `tenants/${tenantId}/jobs`));

// Good: Limits to most recent 50
const recentJobs = await getDocs(
  query(
    collection(firestore, `tenants/${tenantId}/jobs`),
    orderBy('createdAt', 'desc'),
    limit(50)
  )
);
```

**Avoid Collection Group Queries When Possible:**

```typescript
// Bad: Expensive collection group query
const allCosts = collectionGroup(firestore, 'costs');

// Good: Query specific job's costs
const jobCosts = collection(firestore, `tenants/${tenantId}/jobs/${jobId}/costs`);
```

**Use Pagination with startAfter:**

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

### 9.1.2 Subcollection vs Root Collection Trade-offs

**Use Subcollections For:**
- Costs under jobs (fetch job + costs in single offline query)
- Advances under jobs
- Events under jobs

**Use Root Collections For:**
- Jobs (filtered by tenant, frequently queried independently)
- Vehicles (referenced across multiple jobs)
- Machines (referenced across multiple jobs)

**Example: Why costs are subcollections**

```typescript
// Single offline query fetches job AND all costs
const jobRef = doc(firestore, `tenants/${tenantId}/jobs/${jobId}`);
const costsRef = collection(jobRef, 'costs');

// Both available offline after initial sync
const job = await getDoc(jobRef);
const costs = await getDocs(costsRef);
```

---

## 9.2 Cloud Function Cold Starts

### 9.2.1 Cold Start Mitigation

**Minimize Function Size:**

```typescript
// Bad: Import entire library
import * as admin from 'firebase-admin';

// Good: Import only what you need
import { firestore } from 'firebase-admin';
import { onDocumentCreated } from 'firebase-functions/v2/firestore';
```

**Lazy Load Heavy Dependencies:**

```typescript
export const generatePDF = onCall(async (request) => {
  // Only load pdfmake when function is invoked
  const pdfMake = await import('pdfmake');

  // ... PDF generation logic
});
```

**Use Global Variables for Reusable Connections:**

```typescript
import { initializeApp } from 'firebase-admin/app';
import { getFirestore } from 'firebase-admin/firestore';

// Initialize once per container instance
const app = initializeApp();
const db = getFirestore(app);

export const allocateSequence = onCall(async (request) => {
  // Reuse db connection
  const counterRef = db.doc(`sequences/${tenantId}/counters/${type}`);
  // ...
});
```

**Consider Min Instances for Critical Functions (Production Only):**

```typescript
export const allocateSequence = onCall(
  {
    region: 'europe-west1',
    minInstances: 1, // Keep 1 instance warm (costs ~$6/month)
    maxInstances: 100
  },
  async (request) => {
    // ... function logic
  }
);
```

**Cold Start Metrics:**
- Target: <1 second for critical functions
- Typical: 500ms-2s for Node.js 20 functions
- Min instances eliminate cold starts but add cost

### 9.2.2 Optimize Function Size

**Bundle Size Analysis:**

```bash
# Check function bundle size
cd packages/functions
npm run build
du -sh lib/
```

**Target:** <10MB for fast cold starts

**Optimization Strategies:**
- Use tree-shaking (import only what you need)
- Avoid large dependencies (e.g., use `date-fns` instead of `moment`)
- Use dynamic imports for infrequently-used code paths

---

## 9.3 Caching Strategies

### 9.3.1 Firestore Client-Side Cache

**Offline Persistence:**

```typescript
import { enableIndexedDbPersistence } from '@angular/fire/firestore';

const firestore = getFirestore();
await enableIndexedDbPersistence(firestore);
```

**Cache Size Limits:**
- Default: 40MB for web
- Can be increased to 100MB
- Automatic LRU eviction when full

**What Gets Cached:**
- All documents from real-time listeners
- Documents from `getDoc()` and `getDocs()` calls
- Pending writes (queued for sync)

**Cache Benefits:**
- Instant app startup with cached data (NFR3: <2s)
- Offline functionality
- Reduced Firestore read costs

### 9.3.2 Application-Level Caching

**Cache Resource Definitions:**

```typescript
@Injectable({ providedIn: 'root' })
export class VehicleService {
  private vehiclesCache$ = new BehaviorSubject<Vehicle[]>([]);
  private cacheExpiry = 0;

  getVehicles(tenantId: string): Observable<Vehicle[]> {
    // Return cached data if fresh (<5 minutes old)
    if (Date.now() < this.cacheExpiry) {
      return this.vehiclesCache$.asObservable();
    }

    // Fetch from Firestore and cache
    return collectionData(
      collection(this.firestore, `tenants/${tenantId}/vehicles`)
    ).pipe(
      tap(vehicles => {
        this.vehiclesCache$.next(vehicles);
        this.cacheExpiry = Date.now() + 5 * 60 * 1000; // 5 minutes
      })
    );
  }
}
```

**Cache Business Profile:**

```typescript
// Business profile changes rarely - cache for session
private businessProfile$ = new ReplaySubject<BusinessProfile>(1);

getBusinessProfile(tenantId: string): Observable<BusinessProfile> {
  if (!this.businessProfile$.observers.length) {
    const profileRef = doc(this.firestore, `tenants/${tenantId}/businessProfile`);
    docData(profileRef).pipe(take(1)).subscribe(this.businessProfile$);
  }

  return this.businessProfile$.asObservable();
}
```

---

## 9.4 Cost Optimization

### 9.4.1 Firestore Cost Breakdown

**Pricing (as of 2025):**
- Reads: $0.06 per 100K documents
- Writes: $0.18 per 100K documents
- Deletes: $0.02 per 100K documents
- Storage: $0.18 per GB/month

**Cost Estimation (100 active users, 20 jobs/month each):**

```
Daily Operations:
- Job reads: 100 users × 20 jobs × 2 views/day = 4,000 reads
- Cost reads: 100 users × 100 costs × 5 views/day = 50,000 reads
- New costs: 100 users × 10 costs/day = 1,000 writes
- Job updates: 100 users × 2 updates/day = 200 writes

Monthly costs:
- Reads: (4,000 + 50,000) × 30 = 1.62M reads = $0.97
- Writes: (1,000 + 200) × 30 = 36K writes = $0.06
- Storage: ~1GB = $0.18

Total: ~$1.21/month for Firestore
```

**Cost Optimization Strategies:**

1. **Use Offline Cache:**
```typescript
// Bad: Fetch every time
await getDocs(collection(firestore, 'vehicles'));

// Good: Use real-time listener with cache
collectionData(collection(firestore, 'vehicles')); // Cached automatically
```

2. **Limit Query Results:**
```typescript
// Always use limit
query(collection, where(...), limit(50));
```

3. **Batch Writes:**
```typescript
// Use batch writes instead of individual writes
const batch = writeBatch(firestore);
costs.forEach(cost => {
  batch.set(doc(collection(firestore, 'costs')), cost);
});
await batch.commit(); // Single write operation
```

4. **Avoid Collection Group Queries:**
```typescript
// Expensive: Scans all subcollections
collectionGroup(firestore, 'costs');

// Cheaper: Query specific job
collection(firestore, `tenants/${tid}/jobs/${jid}/costs`);
```

### 9.4.2 Cloud Functions Cost Breakdown

**Pricing:**
- Invocations: $0.40 per million
- Compute time: $0.000024 per GB-second

**Free Tier:**
- 2M invocations/month
- 400K GB-seconds/month

**Estimated Usage (100 users):**
- Audit logging: 36K invocations/month (creates/updates/deletes)
- Sequence allocation: 1.2K invocations/month
- Data export: 10 invocations/month

**Total: Well within free tier**

**Optimization:**
- Minimize trigger invocations (batch operations)
- Use 256MB memory (smallest needed)
- Keep execution time <500ms

### 9.4.3 Voice API Cost Estimates

**Google Cloud STT:**
- $0.006 per 15 seconds
- Average cost: 30 sec/command × 10 commands/day × 100 users = $12/month

**Google Cloud TTS:**
- $16.00 per 1M characters (WaveNet)
- Average cost: 50 chars/response × 10 commands/day × 100 users = $0.80/month

**OpenAI GPT-4:**
- $0.03 per 1K input tokens
- Average cost: 100 tokens/command × 10 commands/day × 100 users = $9/month

**Total Voice API costs: ~$22/month for 100 users**

---

## Performance Monitoring

### Key Metrics to Track

1. **Firestore Query Latency**
   - Target: <100ms for indexed queries
   - Monitor: Firebase Performance Monitoring

2. **Cloud Function Execution Time**
   - Target: <500ms (NFR14)
   - Monitor: Cloud Functions logs

3. **App Cold Start Time**
   - Target: <2s with cached data (NFR3)
   - Monitor: Firebase Performance Monitoring

4. **Offline Sync Time**
   - Target: <5s for pending writes
   - Monitor: Custom analytics

### Performance Testing

```typescript
// Performance trace for critical operations
import { trace } from '@angular/fire/performance';

async createJobWithPerformanceTracking(job: Job) {
  const traceRef = trace(this.performance, 'create_job');
  traceRef.start();

  try {
    await this.createJob(job);
    traceRef.stop();
  } catch (error) {
    traceRef.stop();
    throw error;
  }
}
```

---

[← Back: Security Considerations](./security-considerations.md) | [Back to Index](./index.md) | [Next: Testing Strategy →](./testing-strategy.md)
