[Back to Index](./index.md)

# Testing Strategy

This document covers unit testing for Cloud Functions, integration testing with Firebase Emulator, security rules testing, and end-to-end testing strategies.

## Overview

Testing pyramid for FinDogAI backend:
- **Unit Tests**: Cloud Functions business logic
- **Integration Tests**: Firebase Emulator-based testing
- **Security Rules Tests**: Firestore and Storage rules validation
- **E2E Tests**: Full workflow testing (Phase 2)

---

## 10.1 Unit Testing for Cloud Functions

### 10.1.1 Jest Configuration

**File:** `packages/functions/jest.config.js`

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/test'],
  testMatch: ['**/*.test.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/index.ts',
    '!src/**/*.d.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

### 10.1.2 Example: Sequence Allocation Tests

**File:** `packages/functions/test/allocateSequence.test.ts`

```typescript
import { describe, test, expect, beforeEach } from '@jest/globals';
import { firestore } from 'firebase-admin';
import { allocateSequence } from '../src/callable/allocateSequence';

// Mock Firebase Admin
jest.mock('firebase-admin', () => ({
  firestore: jest.fn(() => ({
    doc: jest.fn(),
    runTransaction: jest.fn()
  }))
}));

describe('allocateSequence', () => {
  let mockFirestore: jest.Mocked<firestore.Firestore>;
  let mockTransaction: jest.Mocked<firestore.Transaction>;

  beforeEach(() => {
    mockFirestore = firestore() as jest.Mocked<firestore.Firestore>;
    mockTransaction = {
      get: jest.fn(),
      set: jest.fn(),
      update: jest.fn()
    } as any;
  });

  test('should allocate first job number', async () => {
    const tenantId = 'test-tenant-123';

    // Mock counter document doesn't exist
    mockTransaction.get.mockResolvedValue({
      exists: false
    } as any);

    // Mock transaction
    mockFirestore.runTransaction.mockImplementation(async (callback) => {
      return await callback(mockTransaction);
    });

    const result = await allocateSequence({
      data: { tenantId, type: 'jobNumber' },
      auth: { uid: 'user-123' }
    } as any);

    expect(result.sequenceNumber).toBe(1);
    expect(mockTransaction.set).toHaveBeenCalledWith(
      expect.anything(),
      { current: 1 },
      { merge: true }
    );
  });

  test('should increment existing counter', async () => {
    const tenantId = 'test-tenant-123';

    // Mock counter document exists with current = 42
    mockTransaction.get.mockResolvedValue({
      exists: true,
      data: () => ({ current: 42 })
    } as any);

    mockFirestore.runTransaction.mockImplementation(async (callback) => {
      return await callback(mockTransaction);
    });

    const result = await allocateSequence({
      data: { tenantId, type: 'jobNumber' },
      auth: { uid: 'user-123' }
    } as any);

    expect(result.sequenceNumber).toBe(43);
    expect(mockTransaction.set).toHaveBeenCalledWith(
      expect.anything(),
      { current: 43 },
      { merge: true }
    );
  });

  test('should throw error for unauthenticated user', async () => {
    await expect(
      allocateSequence({
        data: { tenantId: 'test-123', type: 'jobNumber' },
        auth: undefined
      } as any)
    ).rejects.toThrow('User must be authenticated');
  });

  test('should throw error for invalid tenant membership', async () => {
    // Mock member document doesn't exist
    mockFirestore.doc.mockReturnValue({
      get: jest.fn().mockResolvedValue({ exists: false })
    } as any);

    await expect(
      allocateSequence({
        data: { tenantId: 'test-123', type: 'jobNumber' },
        auth: { uid: 'user-123' }
      } as any)
    ).rejects.toThrow('Not an active member');
  });
});
```

**Run Tests:**

```bash
cd packages/functions
npm test

# With coverage
npm test -- --coverage

# Watch mode
npm test -- --watch
```

---

## 10.2 Integration Testing with Firebase Emulator

### 10.2.1 Emulator Configuration

**File:** `firebase.json`

```json
{
  "emulators": {
    "auth": {
      "port": 9099
    },
    "firestore": {
      "port": 8080
    },
    "functions": {
      "port": 5001
    },
    "storage": {
      "port": 9199
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
```

### 10.2.2 Example: End-to-End Sequence Allocation Test

**File:** `packages/functions/test/e2e/sequence.e2e.test.ts`

```typescript
import { initializeApp, deleteApp } from 'firebase/app';
import { getFirestore, collection, addDoc, serverTimestamp } from 'firebase/firestore';
import { getAuth, signInWithEmailAndPassword } from 'firebase/auth';
import { getFunctions, httpsCallable, connectFunctionsEmulator } from 'firebase/functions';
import { describe, test, expect, beforeAll, afterAll } from '@jest/globals';

describe('Sequence Allocation E2E', () => {
  let app;
  let firestore;
  let auth;
  let functions;

  beforeAll(async () => {
    // Initialize Firebase app connected to emulators
    app = initializeApp({
      projectId: 'demo-test-project',
      apiKey: 'fake-api-key'
    });

    firestore = getFirestore(app);
    auth = getAuth(app);
    functions = getFunctions(app, 'europe-west1');

    // Connect to emulators
    connectFirestoreEmulator(firestore, 'localhost', 8080);
    connectAuthEmulator(auth, 'http://localhost:9099');
    connectFunctionsEmulator(functions, 'localhost', 5001);

    // Sign in test user
    await signInWithEmailAndPassword(auth, 'test@example.com', 'password123');
  });

  afterAll(async () => {
    await deleteApp(app);
  });

  test('should allocate sequential job numbers', async () => {
    const tenantId = 'test-tenant-123';
    const allocate = httpsCallable(functions, 'allocateSequence');

    // Allocate first job number
    const result1 = await allocate({ tenantId, type: 'jobNumber' });
    expect(result1.data.sequenceNumber).toBe(1);

    // Allocate second job number
    const result2 = await allocate({ tenantId, type: 'jobNumber' });
    expect(result2.data.sequenceNumber).toBe(2);

    // Allocate third job number
    const result3 = await allocate({ tenantId, type: 'jobNumber' });
    expect(result3.data.sequenceNumber).toBe(3);
  });

  test('should auto-assign number to offline-created job', async () => {
    const tenantId = 'test-tenant-123';

    // Create job without jobNumber (simulating offline creation)
    const jobRef = await addDoc(
      collection(firestore, `tenants/${tenantId}/jobs`),
      {
        tenantId,
        jobNumber: null,
        title: 'Test Job',
        status: 'active',
        createdAt: serverTimestamp(),
        createdBy: auth.currentUser.uid,
        updatedAt: serverTimestamp(),
        updatedBy: auth.currentUser.uid
      }
    );

    // Wait for Cloud Function trigger to assign number
    await new Promise(resolve => setTimeout(resolve, 2000));

    // Verify jobNumber was assigned
    const jobDoc = await getDoc(jobRef);
    expect(jobDoc.data().jobNumber).toBeGreaterThan(0);
  });
});
```

**Run E2E Tests:**

```bash
# Start emulators
firebase emulators:start

# In another terminal, run E2E tests
npm run test:e2e
```

---

## 10.3 Security Rules Testing

### 10.3.1 Rules Test Configuration

**File:** `packages/functions/test/firestore.rules.test.ts`

```typescript
import {
  assertFails,
  assertSucceeds,
  initializeTestEnvironment,
  RulesTestEnvironment
} from '@firebase/rules-unit-testing';
import { describe, test, beforeAll, afterAll } from '@jest/globals';
import { doc, getDoc, setDoc, updateDoc, deleteDoc } from 'firebase/firestore';

describe('Firestore Security Rules', () => {
  let testEnv: RulesTestEnvironment;

  beforeAll(async () => {
    testEnv = await initializeTestEnvironment({
      projectId: 'demo-test-project',
      firestore: {
        rules: readFileSync('firestore.rules', 'utf8'),
        host: 'localhost',
        port: 8080
      }
    });
  });

  afterAll(async () => {
    await testEnv.cleanup();
  });

  beforeEach(async () => {
    await testEnv.clearFirestore();
  });

  describe('Job Collection', () => {
    test('Owner can create job', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup: Create member document
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
      });

      // Test: Owner can create job
      await assertSucceeds(
        setDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          jobNumber: 1,
          title: 'Test Job',
          status: 'active',
          createdAt: new Date(),
          createdBy: 'owner-uid',
          updatedAt: new Date(),
          updatedBy: 'owner-uid'
        })
      );
    });

    test('Representative can read jobs', async () => {
      const repContext = testEnv.authenticatedContext('rep-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/rep-uid'), {
          role: 'representative',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          jobNumber: 1,
          title: 'Test Job'
        });
      });

      // Test: Representative can read job
      await assertSucceeds(
        getDoc(doc(repContext.firestore(), 'tenants/test-tenant/jobs/job-1'))
      );
    });

    test('Team member cannot read full jobs', async () => {
      const teamContext = testEnv.authenticatedContext('team-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/team-uid'), {
          role: 'teamMember',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          jobNumber: 1,
          title: 'Test Job'
        });
      });

      // Test: Team member cannot read full job
      await assertFails(
        getDoc(doc(teamContext.firestore(), 'tenants/test-tenant/jobs/job-1'))
      );
    });

    test('Unauthenticated user cannot read jobs', async () => {
      const unauthContext = testEnv.unauthenticatedContext();

      // Test: Unauthenticated user cannot read
      await assertFails(
        getDoc(doc(unauthContext.firestore(), 'tenants/test-tenant/jobs/job-1'))
      );
    });

    test('Cannot hard delete jobs', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          jobNumber: 1
        });
      });

      // Test: Even owner cannot hard delete
      await assertFails(
        deleteDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/jobs/job-1'))
      );
    });
  });

  describe('Audit Logs', () => {
    test('Owner can read audit logs', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/audit_logs/log-1'), {
          operation: 'CREATE',
          collection: 'jobs',
          timestamp: new Date()
        });
      });

      // Test: Owner can read audit logs
      await assertSucceeds(
        getDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/audit_logs/log-1'))
      );
    });

    test('Representative cannot read audit logs', async () => {
      const repContext = testEnv.authenticatedContext('rep-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/rep-uid'), {
          role: 'representative',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/audit_logs/log-1'), {
          operation: 'CREATE'
        });
      });

      // Test: Representative cannot read audit logs
      await assertFails(
        getDoc(doc(repContext.firestore(), 'tenants/test-tenant/audit_logs/log-1'))
      );
    });

    test('Client cannot write audit logs', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      // Setup
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
      });

      // Test: Even owner cannot write audit logs
      await assertFails(
        setDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/audit_logs/log-1'), {
          operation: 'CREATE'
        })
      );
    });
  });
});
```

### 10.3.2 Running Rules Tests

```bash
# Start Firestore emulator
firebase emulators:start --only firestore

# Run rules tests
npm run test:rules

# With coverage
npm run test:rules -- --coverage
```

---

## 10.4 End-to-End Testing (Phase 2)

### 10.4.1 Playwright Configuration

**File:** `packages/e2e/playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] }
    }
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env.CI
  }
});
```

### 10.4.2 Example: Voice Flow E2E Test (with Mocks)

**File:** `packages/e2e/tests/voice-cost-creation.spec.ts`

```typescript
import { test, expect } from '@playwright/test';

test('should create cost via voice command', async ({ page }) => {
  // Mock voice APIs
  await page.route('**/speech.googleapis.com/**', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({ results: [{ alternatives: [{ transcript: 'Material cost 500 crowns for cement' }] }] })
    });
  });

  await page.route('**/api.openai.com/**', route => {
    route.fulfill({
      status: 200,
      body: JSON.stringify({
        choices: [{
          message: {
            content: JSON.stringify({
              action: 'create_cost',
              category: 'material',
              amount: 500,
              description: 'cement'
            })
          }
        }]
      })
    });
  });

  // Navigate to job detail page
  await page.goto('/jobs/job-123');

  // Click voice button
  await page.click('[data-testid="voice-button"]');

  // Wait for microphone permission and recording
  await page.waitForSelector('[data-testid="recording-indicator"]');

  // Simulate voice command (in real test, this would be actual audio)
  await page.click('[data-testid="stop-recording"]');

  // Wait for cost to appear in list
  await expect(page.locator('[data-testid="cost-item"]')).toContainText('Material');
  await expect(page.locator('[data-testid="cost-amount"]')).toContainText('500');
});
```

---

## Test Coverage Goals

| Test Type | Target Coverage | Actual |
|-----------|----------------|--------|
| Unit Tests (Functions) | 80% | TBD |
| Integration Tests | Key flows covered | TBD |
| Security Rules Tests | All rules validated | TBD |
| E2E Tests (Phase 2) | Happy paths + critical errors | TBD |

---

[← Back: Performance Optimization](./performance-optimization.md) | [Back to Index](./index.md) | [Next: Operations →](./operations.md)
