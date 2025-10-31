# Testing Strategy

**ABOUTME:** Comprehensive testing strategy for FinDogAI covering unit, integration, E2E, performance, security, accessibility, and visual regression testing with detailed patterns, test data management, and implementation guidelines.

## Table of Contents

1. [Overview](#1-overview)
2. [Unit Testing](#2-unit-testing)
3. [Integration Testing](#3-integration-testing)
4. [End-to-End (E2E) Testing](#4-end-to-end-e2e-testing)
5. [Performance Testing](#5-performance-testing)
6. [Security Testing](#6-security-testing)
7. [Accessibility Testing](#7-accessibility-testing)
8. [Visual Regression Testing](#8-visual-regression-testing)
9. [Test Data Management](#9-test-data-management)
10. [Frontend Testing Patterns](#10-frontend-testing-patterns)
11. [Backend Testing Patterns](#11-backend-testing-patterns)
12. [Testing Best Practices](#12-testing-best-practices)
13. [CI/CD Integration](#13-cicd-integration)
14. [Coverage Goals and Metrics](#14-coverage-goals-and-metrics)

---

## 1. Overview

### 1.1 Testing Philosophy

FinDogAI follows a comprehensive testing strategy that ensures:
- **Reliability**: Especially critical for offline operations and data synchronization
- **Security**: Thorough validation of multi-tenant isolation and RBAC
- **Performance**: Meeting NFR targets for voice pipeline latency and offline operations
- **Accessibility**: WCAG 2.1 AA compliance validation
- **User Experience**: Voice command accuracy and touch-friendly interactions

### 1.2 Testing Pyramid

```
         /\
        /  \        E2E Tests (5-10%)
       /____\       - Critical user journeys
      /      \      - Voice pipeline flows
     /________\     Integration Tests (20-30%)
    /          \    - Feature-level flows
   /____________\   - API integrations
  /______________\  Unit Tests (60-75%)
                    - Component logic
                    - Business logic
                    - Utility functions
```

### 1.3 Technology Stack

| Test Type | Tools | Purpose |
|-----------|-------|---------|
| **Unit Tests** | Jest 29+ + Angular Testing Library | Component and service testing |
| **Backend Unit Tests** | Jest 29+ + Firebase Functions Test SDK | Cloud Functions testing |
| **Integration Tests** | Jest + Firebase Emulator Suite | Multi-component flows |
| **E2E Tests** | Playwright 1.40+ | Full user journey testing |
| **Performance Tests** | Lighthouse CI + k6 | Performance benchmarking |
| **Security Tests** | OWASP ZAP + npm audit + Firestore Rules Testing | Security validation |
| **Accessibility Tests** | axe-core + Lighthouse | WCAG compliance |
| **Visual Regression** | Playwright Screenshots + Percy | UI consistency |

---

## 2. Unit Testing

### 2.1 Frontend Unit Testing

#### 2.1.1 Component Testing Structure

**File naming:** `*.component.spec.ts`

**Template:**
```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { IonicModule } from '@ionic/angular';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { signal } from '@angular/core';

describe('[ComponentName]Component', () => {
  let component: [ComponentName]Component;
  let fixture: ComponentFixture<[ComponentName]Component>;
  let store: MockStore;

  // Mock data
  const mockData = {
    // Define test data
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        [ComponentName]Component, // Standalone component
        IonicModule.forRoot()
      ],
      providers: [
        provideMockStore({
          initialState: {
            // Define initial state
          }
        }),
        provideRouter([])
        // Mock services
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent([ComponentName]Component);
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
  });

  // Additional test suites
});
```

#### 2.1.2 Service Testing Structure

**File naming:** `*.service.spec.ts`

**Template:**
```typescript
import { TestBed } from '@angular/core/testing';
import { Firestore } from '@angular/fire/firestore';

describe('[ServiceName]Service', () => {
  let service: [ServiceName]Service;
  let firestoreMock: jasmine.SpyObj<Firestore>;

  beforeEach(() => {
    const firestoreSpy = jasmine.createSpyObj('Firestore', [
      'collection',
      'doc',
      'collectionData',
      'docData'
    ]);

    TestBed.configureTestingModule({
      providers: [
        [ServiceName]Service,
        { provide: Firestore, useValue: firestoreSpy }
      ]
    });

    service = TestBed.inject([ServiceName]Service);
    firestoreMock = TestBed.inject(Firestore) as jasmine.SpyObj<Firestore>;
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  // Test service methods
});
```

#### 2.1.3 NgRx Effects Testing

**File naming:** `*.effects.spec.ts`

**Example:**
```typescript
import { TestBed } from '@angular/core/testing';
import { provideMockActions } from '@ngrx/effects/testing';
import { Observable, of, throwError } from 'rxjs';
import { JobsEffects } from './jobs.effects';
import { JobsService } from '../services/jobs.service';
import { JobsActions } from './jobs.actions';

describe('JobsEffects', () => {
  let actions$: Observable<any>;
  let effects: JobsEffects;
  let jobsService: jasmine.SpyObj<JobsService>;

  beforeEach(() => {
    const jobsServiceSpy = jasmine.createSpyObj('JobsService', [
      'getAll',
      'create',
      'update',
      'delete'
    ]);

    TestBed.configureTestingModule({
      providers: [
        JobsEffects,
        provideMockActions(() => actions$),
        { provide: JobsService, useValue: jobsServiceSpy }
      ]
    });

    effects = TestBed.inject(JobsEffects);
    jobsService = TestBed.inject(JobsService) as jasmine.SpyObj<JobsService>;
  });

  describe('loadJobs$', () => {
    it('should return loadJobsSuccess on successful load', (done) => {
      const mockJobs = [
        { id: '1', title: 'Job 1' },
        { id: '2', title: 'Job 2' }
      ];
      jobsService.getAll.and.returnValue(of(mockJobs));

      actions$ = of(JobsActions.loadJobs());

      effects.loadJobs$.subscribe(action => {
        expect(action).toEqual(JobsActions.loadJobsSuccess({ jobs: mockJobs }));
        done();
      });
    });

    it('should return loadJobsFailure on error', (done) => {
      const error = new Error('Failed to load');
      jobsService.getAll.and.returnValue(throwError(() => error));

      actions$ = of(JobsActions.loadJobs());

      effects.loadJobs$.subscribe(action => {
        expect(action).toEqual(JobsActions.loadJobsFailure({ error: error.message }));
        done();
      });
    });
  });
});
```

### 2.2 Backend Unit Testing

#### 2.2.1 Cloud Functions Testing

**File naming:** `*.test.ts`

**Example:**
```typescript
import { describe, test, expect, jest, beforeEach } from '@jest/globals';
import { CallableContext } from 'firebase-functions/v2/https';
import { allocateSequence } from '../callable/allocateSequence';

// Mock firebase-admin
jest.mock('firebase-admin', () => ({
  firestore: jest.fn(() => ({
    doc: jest.fn(),
    runTransaction: jest.fn()
  }))
}));

describe('allocateSequence', () => {
  let mockContext: Partial<CallableContext>;

  beforeEach(() => {
    jest.clearAllMocks();
    mockContext = {
      auth: {
        uid: 'test-user-123',
        token: {}
      }
    };
  });

  test('should allocate first sequence number', async () => {
    // Test implementation
  });

  test('should increment existing counter', async () => {
    // Test implementation
  });

  test('should throw error for unauthenticated user', async () => {
    mockContext.auth = undefined;

    await expect(
      allocateSequence(
        { tenantId: 'test-123', type: 'jobNumber' },
        mockContext as CallableContext
      )
    ).rejects.toThrow('User must be authenticated');
  });
});
```

### 2.3 Coverage Requirements

| Code Type | Minimum Coverage | Target Coverage |
|-----------|------------------|-----------------|
| Business Logic (Services) | 90% | 95% |
| Components | 80% | 85% |
| NgRx Effects | 85% | 90% |
| NgRx Reducers | 95% | 100% |
| Utility Functions | 90% | 95% |
| Cloud Functions | 80% | 85% |

---

## 3. Integration Testing

### 3.1 Frontend Integration Testing

#### 3.1.1 Feature-Level Integration Tests

**Purpose:** Test complete user flows within a feature module

**Example: Job Creation with Cost Entry**
```typescript
// jobs-feature.integration.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { JobsService } from './services/jobs.service';
import { CostsService } from './services/costs.service';
import { JobsActions } from './data-access/jobs.actions';
import { CostsActions } from './data-access/costs.actions';
import { of } from 'rxjs';

describe('Jobs Feature Integration', () => {
  let store: MockStore;
  let jobsService: JobsService;
  let costsService: CostsService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideMockStore({ initialState: {} }),
        JobsService,
        CostsService
      ]
    });

    store = TestBed.inject(MockStore);
    jobsService = TestBed.inject(JobsService);
    costsService = TestBed.inject(CostsService);
  });

  it('should create job and add cost in sequence', async () => {
    // Arrange
    spyOn(store, 'dispatch');
    spyOn(jobsService, 'create').and.returnValue(of('job-123'));
    spyOn(costsService, 'create').and.returnValue(of('cost-456'));

    // Act - Create job
    const job = {
      title: 'Test Job',
      budget: 10000
    };
    await jobsService.create(job).toPromise();

    // Act - Add cost to job
    const cost = {
      jobId: 'job-123',
      category: 'material',
      amount: 500
    };
    await costsService.create(cost).toPromise();

    // Assert
    expect(jobsService.create).toHaveBeenCalledWith(job);
    expect(costsService.create).toHaveBeenCalledWith(
      jasmine.objectContaining({ jobId: 'job-123' })
    );
  });

  it('should handle offline job creation and sync', async () => {
    // Test offline queue and sync behavior
  });
});
```

#### 3.1.2 Voice Pipeline Integration Testing

**File:** `voice-pipeline.integration.spec.ts`

```typescript
import { TestBed } from '@angular/core/testing';
import { VoiceCommandService } from '@core/services/voice-command.service';
import { SpeechRecognitionService } from '@core/services/speech-recognition.service';
import { LlmService } from '@core/services/llm.service';
import { Store } from '@ngrx/store';
import { provideMockStore } from '@ngrx/store/testing';
import { of } from 'rxjs';

describe('Voice Pipeline Integration', () => {
  let voiceCommandService: VoiceCommandService;
  let speechService: SpeechRecognitionService;
  let llmService: LlmService;
  let store: MockStore;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        VoiceCommandService,
        SpeechRecognitionService,
        LlmService,
        provideMockStore({ initialState: {} })
      ]
    });

    voiceCommandService = TestBed.inject(VoiceCommandService);
    speechService = TestBed.inject(SpeechRecognitionService);
    llmService = TestBed.inject(LlmService);
    store = TestBed.inject(MockStore);
  });

  describe('Complete Voice Flow', () => {
    it('should process "add material cost" command', async () => {
      // Arrange
      const mockTranscript = 'add material cost 500 euros for concrete';
      const mockIntent = {
        action: 'CREATE_COST',
        category: 'material',
        amount: 500,
        currency: 'EUR',
        description: 'concrete'
      };

      spyOn(speechService, 'recognize').and.returnValue(of(mockTranscript));
      spyOn(llmService, 'extractIntent').and.returnValue(of(mockIntent));
      spyOn(store, 'dispatch');

      // Act
      await voiceCommandService.processVoiceCommand().toPromise();

      // Assert
      expect(speechService.recognize).toHaveBeenCalled();
      expect(llmService.extractIntent).toHaveBeenCalledWith(mockTranscript);
      expect(store.dispatch).toHaveBeenCalledWith(
        jasmine.objectContaining({
          type: '[Costs] Create Cost',
          cost: jasmine.objectContaining({
            category: 'material',
            amount: 500
          })
        })
      );
    });

    it('should handle offline voice command queueing', async () => {
      // Test offline voice command behavior
    });

    it('should retry failed STT with fallback provider', async () => {
      // Test fallback mechanisms
    });
  });
});
```

### 3.2 Backend Integration Testing with Firebase Emulator

#### 3.2.1 Emulator Setup

**File:** `firebase.json`
```json
{
  "emulators": {
    "auth": { "port": 9099 },
    "firestore": { "port": 8080 },
    "functions": { "port": 5001 },
    "storage": { "port": 9199 },
    "ui": { "enabled": true, "port": 4000 }
  }
}
```

#### 3.2.2 Full Flow Integration Test

**Example: Job Creation with Triggers**
```typescript
// job-lifecycle.integration.test.ts
import { initializeApp, deleteApp } from 'firebase/app';
import { getFirestore, collection, addDoc, onSnapshot } from 'firebase/firestore';
import { getAuth, signInWithEmailAndPassword } from 'firebase/auth';
import { getFunctions, httpsCallable } from 'firebase/functions';

describe('Job Lifecycle Integration', () => {
  let app;
  let db;
  let auth;
  let functions;

  beforeAll(async () => {
    // Setup Firebase with emulators
    app = initializeApp({ projectId: 'demo-test', apiKey: 'fake-key' });
    db = getFirestore(app);
    auth = getAuth(app);
    functions = getFunctions(app, 'europe-west1');

    // Connect to emulators
    connectFirestoreEmulator(db, 'localhost', 8080);
    connectAuthEmulator(auth, 'http://localhost:9099');
    connectFunctionsEmulator(functions, 'localhost', 5001);

    await signInWithEmailAndPassword(auth, 'test@example.com', 'password123');
  });

  afterAll(async () => {
    await deleteApp(app);
  });

  test('should create job, trigger audit log, and allocate number', async () => {
    const tenantId = 'test-tenant-123';

    // Create job without jobNumber
    const jobRef = await addDoc(collection(db, `tenants/${tenantId}/jobs`), {
      tenantId,
      title: 'Integration Test Job',
      status: 'active',
      jobNumber: null
    });

    // Wait for Cloud Function to allocate jobNumber
    await new Promise(resolve => setTimeout(resolve, 2000));

    // Verify jobNumber was assigned
    const jobSnap = await getDoc(jobRef);
    expect(jobSnap.data().jobNumber).toBeGreaterThan(0);

    // Verify audit log was created
    const auditLogs = await getDocs(
      collection(db, `tenants/${tenantId}/audit_logs`)
    );
    expect(auditLogs.size).toBeGreaterThan(0);

    const auditLog = auditLogs.docs[0].data();
    expect(auditLog.operation).toBe('CREATE');
    expect(auditLog.collection).toBe('jobs');
  });
});
```

---

## 4. End-to-End (E2E) Testing

### 4.1 Playwright Configuration

**File:** `playwright.config.ts`
```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/specs',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  use: {
    baseURL: 'http://localhost:4200',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Tablet',
      use: { ...devices['iPad (gen 7)'] }
    }
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env.CI,
    timeout: 120000
  }
});
```

### 4.2 E2E Test Patterns

#### 4.2.1 Page Object Model

**File:** `e2e/pages/job-list.page.ts`
```typescript
import { Page, Locator } from '@playwright/test';

export class JobListPage {
  readonly page: Page;
  readonly createJobButton: Locator;
  readonly jobCards: Locator;
  readonly searchInput: Locator;

  constructor(page: Page) {
    this.page = page;
    this.createJobButton = page.locator('[data-testid="create-job-fab"]');
    this.jobCards = page.locator('[data-testid="job-card"]');
    this.searchInput = page.locator('[data-testid="search-input"]');
  }

  async goto() {
    await this.page.goto('/tabs/jobs');
  }

  async createJob() {
    await this.createJobButton.click();
  }

  async searchJobs(query: string) {
    await this.searchInput.fill(query);
  }

  async getJobCount() {
    return await this.jobCards.count();
  }
}
```

#### 4.2.2 Critical User Journey Test

**File:** `e2e/specs/critical-job-flow.spec.ts`
```typescript
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { JobListPage } from '../pages/job-list.page';
import { JobFormPage } from '../pages/job-form.page';
import { CostEntryPage } from '../pages/cost-entry.page';

test.describe('Critical Job Flow', () => {
  let loginPage: LoginPage;
  let jobListPage: JobListPage;
  let jobFormPage: JobFormPage;
  let costEntryPage: CostEntryPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    jobListPage = new JobListPage(page);
    jobFormPage = new JobFormPage(page);
    costEntryPage = new CostEntryPage(page);

    // Login
    await loginPage.goto();
    await loginPage.login('test@example.com', 'password123');
  });

  test('should complete full job lifecycle', async ({ page }) => {
    // Navigate to jobs
    await jobListPage.goto();

    // Create new job
    await jobListPage.createJob();
    await jobFormPage.fillJobDetails({
      title: 'E2E Test Job',
      budget: 25000,
      currency: 'EUR'
    });
    await jobFormPage.save();

    // Verify redirect to job list
    await expect(page).toHaveURL('/tabs/jobs');

    // Verify job appears
    await expect(jobListPage.jobCards.first()).toContainText('E2E Test Job');

    // Click on job
    await jobListPage.jobCards.first().click();

    // Add cost
    await costEntryPage.addCost({
      category: 'material',
      amount: 500,
      description: 'Concrete'
    });

    // Verify cost appears in list
    await expect(page.locator('[data-testid="cost-item"]').first())
      .toContainText('Material');
    await expect(page.locator('[data-testid="cost-amount"]').first())
      .toContainText('500');
  });

  test('should work offline', async ({ page, context }) => {
    // Go to jobs page
    await jobListPage.goto();

    // Go offline
    await context.setOffline(true);

    // Create job offline
    await jobListPage.createJob();
    await jobFormPage.fillJobDetails({
      title: 'Offline Job',
      budget: 10000
    });
    await jobFormPage.save();

    // Verify offline indicator
    await expect(page.locator('.offline-banner')).toBeVisible();

    // Verify optimistic update
    await expect(jobListPage.jobCards).toHaveCount(1);

    // Go back online
    await context.setOffline(false);

    // Wait for sync
    await expect(page.locator('.sync-indicator')).toBeVisible();
    await expect(page.locator('.sync-indicator')).toBeHidden({ timeout: 10000 });

    // Verify data persisted
    await page.reload();
    await expect(jobListPage.jobCards).toHaveCount(1);
  });
});
```

#### 4.2.3 Voice Command E2E Test

**File:** `e2e/specs/voice-commands.spec.ts`
```typescript
import { test, expect } from '@playwright/test';

test.describe('Voice Commands', () => {
  test.beforeEach(async ({ page }) => {
    // Mock voice APIs
    await page.route('**/speech.googleapis.com/**', route => {
      route.fulfill({
        status: 200,
        body: JSON.stringify({
          results: [{
            alternatives: [{
              transcript: 'add material cost 500 euros for cement'
            }]
          }]
        })
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
                currency: 'EUR',
                description: 'cement'
              })
            }
          }]
        })
      });
    });

    // Login and navigate
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/tabs/jobs');
  });

  test('should create cost via voice command', async ({ page }) => {
    // Navigate to job detail
    await page.click('[data-testid="job-card"]');

    // Activate voice command
    await page.click('[data-testid="voice-button"]');
    await expect(page.locator('[data-testid="recording-indicator"]')).toBeVisible();

    // Simulate voice input (mocked above)
    await page.click('[data-testid="stop-recording"]');

    // Verify confirmation dialog
    await expect(page.locator('[data-testid="voice-confirmation"]'))
      .toContainText('Material cost: 500 EUR for cement');

    // Confirm
    await page.click('[data-testid="confirm-voice-action"]');

    // Verify cost created
    await expect(page.locator('[data-testid="cost-item"]').first())
      .toContainText('Material');
    await expect(page.locator('[data-testid="cost-amount"]').first())
      .toContainText('500');
  });

  test('should handle voice command rejection', async ({ page }) => {
    await page.click('[data-testid="job-card"]');
    await page.click('[data-testid="voice-button"]');
    await page.click('[data-testid="stop-recording"]');

    // Reject voice command
    await page.click('[data-testid="reject-voice-action"]');

    // Verify no cost created
    await expect(page.locator('[data-testid="cost-item"]')).toHaveCount(0);
  });
});
```

---

## 5. Performance Testing

### 5.1 Lighthouse CI Integration

#### 5.1.1 Configuration

**File:** `.lighthouserc.js`
```javascript
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:4200/',
        'http://localhost:4200/tabs/jobs',
        'http://localhost:4200/tabs/costs',
        'http://localhost:4200/tabs/profile'
      ],
      numberOfRuns: 3,
      settings: {
        preset: 'desktop',
        throttling: {
          rttMs: 40,
          throughputKbps: 10240,
          cpuSlowdownMultiplier: 1
        }
      }
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }]
      }
    },
    upload: {
      target: 'temporary-public-storage'
    }
  }
};
```

### 5.2 Load Testing with k6

#### 5.2.1 Voice Pipeline Load Test

**File:** `performance/voice-pipeline.k6.js`
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 10 },  // Ramp up to 10 users
    { duration: '3m', target: 10 },  // Stay at 10 users
    { duration: '1m', target: 50 },  // Spike to 50 users
    { duration: '3m', target: 50 },  // Stay at 50 users
    { duration: '1m', target: 0 },   // Ramp down to 0
  ],
  thresholds: {
    'http_req_duration': ['p(95)<3000'], // 95% of requests < 3s (NFR1)
    'errors': ['rate<0.1'],              // Error rate < 10%
  },
};

export default function () {
  const url = 'https://europe-west1-PROJECT_ID.cloudfunctions.net/processVoiceCommand';
  const payload = JSON.stringify({
    transcript: 'add material cost 500 euros for cement',
    jobId: 'test-job-123'
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ' + __ENV.AUTH_TOKEN,
    },
  };

  const response = http.post(url, payload, params);

  const success = check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 3s': (r) => r.timings.duration < 3000,
    'has intent': (r) => JSON.parse(r.body).intent !== undefined,
  });

  errorRate.add(!success);
  sleep(1);
}
```

#### 5.2.2 Offline Sync Performance Test

**File:** `performance/offline-sync.k6.js`
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    sync_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 20 },
        { duration: '1m', target: 20 },
        { duration: '30s', target: 0 },
      ],
      gracefulRampDown: '30s',
    },
  },
  thresholds: {
    'http_req_duration{scenario:sync_test}': ['p(95)<2000'], // NFR3: Offline data persists within 2s
  },
};

export default function () {
  const syncUrl = 'https://firestore.googleapis.com/v1/projects/PROJECT_ID/databases/(default)/documents:batchWrite';

  // Simulate batch sync of offline operations
  const operations = Array.from({ length: 10 }, (_, i) => ({
    update: {
      name: `projects/PROJECT_ID/databases/(default)/documents/costs/cost-${i}`,
      fields: {
        amount: { integerValue: 500 },
        category: { stringValue: 'material' }
      }
    }
  }));

  const payload = JSON.stringify({ writes: operations });

  const response = http.post(syncUrl, payload, {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ' + __ENV.AUTH_TOKEN,
    },
  });

  check(response, {
    'sync successful': (r) => r.status === 200,
    'sync time < 2s': (r) => r.timings.duration < 2000,
  });

  sleep(5);
}
```

### 5.3 Performance Testing Scenarios

| Scenario | Tool | NFR Target | Threshold |
|----------|------|------------|-----------|
| **Voice Pipeline Latency** | k6 | ≤3s (online) | p95 < 3000ms |
| **Offline Voice Latency** | k6 | ≤8s (offline) | p95 < 8000ms |
| **Offline Data Persistence** | k6 | ≤2s | p95 < 2000ms |
| **PWA Load Time** | Lighthouse | <2s FCP | FCP < 2000ms |
| **Firestore Query Performance** | Firebase Emulator | <500ms | avg < 500ms |

---

## 6. Security Testing

### 6.1 OWASP ZAP Automated Scanning

#### 6.1.1 ZAP Configuration

**File:** `.zap/rules.conf`
```yaml
# OWASP ZAP Security Testing Configuration
env:
  contexts:
    - name: FinDogAI
      urls:
        - http://localhost:4200
      includePaths:
        - http://localhost:4200/.*
      excludePaths:
        - http://localhost:4200/assets/.*
      authentication:
        method: json
        loginUrl: http://localhost:4200/api/login
        loginRequestData: '{"email":"test@example.com","password":"password123"}'
      sessionManagement:
        method: cookie

jobs:
  - type: spider
    parameters:
      context: FinDogAI
      maxDuration: 10

  - type: passiveScan-wait
    parameters:
      maxDuration: 10

  - type: activeScan
    parameters:
      context: FinDogAI
      policy: Default Policy
      maxRuleDurationInMins: 10
      maxScanDurationInMins: 30

  - type: report
    parameters:
      template: traditional-html
      reportDir: /zap/reports
      reportFile: security-report.html
```

#### 6.1.2 CI Integration Script

**File:** `scripts/security-scan.sh`
```bash
#!/bin/bash
# Run OWASP ZAP security scan

# Start application
npm run start &
APP_PID=$!

# Wait for application to be ready
sleep 30

# Run ZAP scan
docker run -v $(pwd):/zap/wrk/:rw \
  -t owasp/zap2docker-stable \
  zap-full-scan.py \
  -t http://host.docker.internal:4200 \
  -r security-report.html \
  -J security-report.json

# Stop application
kill $APP_PID

# Check for high/medium vulnerabilities
HIGH_VULNS=$(jq '.site[0].alerts[] | select(.riskcode=="3") | length' security-report.json)
MEDIUM_VULNS=$(jq '.site[0].alerts[] | select(.riskcode=="2") | length' security-report.json)

if [ "$HIGH_VULNS" -gt 0 ]; then
  echo "❌ Found $HIGH_VULNS high-severity vulnerabilities"
  exit 1
fi

if [ "$MEDIUM_VULNS" -gt 5 ]; then
  echo "⚠️  Found $MEDIUM_VULNS medium-severity vulnerabilities (threshold: 5)"
  exit 1
fi

echo "✅ Security scan passed"
```

### 6.2 Firestore Security Rules Testing

#### 6.2.1 Comprehensive Rules Test Suite

**File:** `firestore-rules.spec.ts`
```typescript
import {
  assertFails,
  assertSucceeds,
  initializeTestEnvironment,
  RulesTestEnvironment
} from '@firebase/rules-unit-testing';
import { readFileSync } from 'fs';
import { describe, test, beforeAll, afterAll, beforeEach } from '@jest/globals';

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

  describe('Multi-Tenant Isolation', () => {
    test('User cannot read jobs from other tenants', async () => {
      const aliceContext = testEnv.authenticatedContext('alice', {
        tenant_id: 'tenant-a'
      });
      const bobContext = testEnv.authenticatedContext('bob', {
        tenant_id: 'tenant-b'
      });

      // Setup: Alice's job in tenant-a
      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/tenant-a/members/alice'), {
          role: 'owner',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/tenant-a/jobs/job-1'), {
          tenantId: 'tenant-a',
          title: 'Alice Job'
        });
      });

      // Test: Bob cannot read Alice's job
      await assertFails(
        getDoc(doc(bobContext.firestore(), 'tenants/tenant-a/jobs/job-1'))
      );
    });

    test('User cannot create job in unauthorized tenant', async () => {
      const aliceContext = testEnv.authenticatedContext('alice', {
        tenant_id: 'tenant-a'
      });

      // Alice tries to create job in tenant-b (unauthorized)
      await assertFails(
        setDoc(doc(aliceContext.firestore(), 'tenants/tenant-b/jobs/job-1'), {
          tenantId: 'tenant-b',
          title: 'Unauthorized Job'
        })
      );
    });
  });

  describe('Role-Based Access Control', () => {
    test('Owner can delete jobs', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          deletedAt: null
        });
      });

      // Owner can soft-delete (set deletedAt)
      await assertSucceeds(
        updateDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          deletedAt: new Date()
        })
      );
    });

    test('Representative cannot delete jobs', async () => {
      const repContext = testEnv.authenticatedContext('rep-uid', {
        tenant_id: 'test-tenant'
      });

      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/rep-uid'), {
          role: 'representative',
          status: 'active'
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant'
        });
      });

      // Representative cannot delete
      await assertFails(
        updateDoc(doc(repContext.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          deletedAt: new Date()
        })
      );
    });

    test('Team member has read-only access to assigned jobs', async () => {
      const teamContext = testEnv.authenticatedContext('team-uid', {
        tenant_id: 'test-tenant'
      });

      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/team-uid'), {
          role: 'teamMember',
          status: 'active',
          assignedJobs: ['job-1']
        });
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          tenantId: 'test-tenant',
          title: 'Assigned Job'
        });
      });

      // Team member can read assigned job
      await assertSucceeds(
        getDoc(doc(teamContext.firestore(), 'tenants/test-tenant/jobs/job-1'))
      );

      // Team member cannot update job
      await assertFails(
        updateDoc(doc(teamContext.firestore(), 'tenants/test-tenant/jobs/job-1'), {
          title: 'Updated Title'
        })
      );
    });
  });

  describe('Data Validation', () => {
    test('Cost amount must be positive', async () => {
      const ownerContext = testEnv.authenticatedContext('owner-uid', {
        tenant_id: 'test-tenant'
      });

      await testEnv.withSecurityRulesDisabled(async (context) => {
        await setDoc(doc(context.firestore(), 'tenants/test-tenant/members/owner-uid'), {
          role: 'owner',
          status: 'active'
        });
      });

      // Negative amount should fail
      await assertFails(
        setDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/costs/cost-1'), {
          tenantId: 'test-tenant',
          amount: -500,
          category: 'material'
        })
      );

      // Positive amount should succeed
      await assertSucceeds(
        setDoc(doc(ownerContext.firestore(), 'tenants/test-tenant/costs/cost-1'), {
          tenantId: 'test-tenant',
          amount: 500,
          category: 'material',
          createdAt: new Date(),
          createdBy: 'owner-uid'
        })
      );
    });
  });
});
```

### 6.3 Dependency Security Scanning

#### 6.3.1 npm audit Integration

**File:** `scripts/security-audit.sh`
```bash
#!/bin/bash
# Run dependency security audit

echo "🔍 Running npm audit..."

# Run audit and capture output
npm audit --json > audit-report.json

# Check for vulnerabilities
CRITICAL=$(jq '.metadata.vulnerabilities.critical' audit-report.json)
HIGH=$(jq '.metadata.vulnerabilities.high' audit-report.json)
MODERATE=$(jq '.metadata.vulnerabilities.moderate' audit-report.json)

echo "Critical: $CRITICAL"
echo "High: $HIGH"
echo "Moderate: $MODERATE"

# Fail if critical or high vulnerabilities
if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
  echo "❌ Security audit failed: Found $CRITICAL critical and $HIGH high vulnerabilities"
  exit 1
fi

echo "✅ Security audit passed"
```

### 6.4 Security Testing Checklist

- [ ] OWASP Top 10 vulnerabilities tested
- [ ] Multi-tenant isolation verified
- [ ] RBAC permissions validated
- [ ] Input validation tested
- [ ] SQL/NoSQL injection prevention verified
- [ ] XSS protection validated
- [ ] CSRF protection tested
- [ ] Authentication bypass attempts blocked
- [ ] Authorization bypass attempts blocked
- [ ] Rate limiting verified
- [ ] Sensitive data exposure prevented
- [ ] Dependency vulnerabilities scanned

---

## 7. Accessibility Testing

### 7.1 axe-core Integration

#### 7.1.1 Automated Accessibility Tests

**File:** `src/app/shared/testing/accessibility.spec.ts`
```typescript
import { ComponentFixture } from '@angular/core/testing';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

export async function testAccessibility(fixture: ComponentFixture<any>) {
  const element = fixture.nativeElement;
  const results = await axe(element);
  expect(results).toHaveNoViolations();
}

// Usage in component tests
describe('JobDetailComponent Accessibility', () => {
  let component: JobDetailComponent;
  let fixture: ComponentFixture<JobDetailComponent>;

  beforeEach(async () => {
    // Setup component
  });

  it('should have no accessibility violations', async () => {
    await testAccessibility(fixture);
  });
});
```

#### 7.1.2 Component-Specific Accessibility Tests

**File:** `job-detail.component.accessibility.spec.ts`
```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { JobDetailComponent } from './job-detail.component';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('JobDetailComponent Accessibility', () => {
  let component: JobDetailComponent;
  let fixture: ComponentFixture<JobDetailComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [JobDetailComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(JobDetailComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should have no WCAG violations', async () => {
    const results = await axe(fixture.nativeElement, {
      rules: {
        'color-contrast': { enabled: true },
        'label': { enabled: true },
        'button-name': { enabled: true },
        'link-name': { enabled: true }
      }
    });

    expect(results).toHaveNoViolations();
  });

  it('should have proper ARIA labels', () => {
    const saveButton = fixture.nativeElement.querySelector('[aria-label="Save job"]');
    expect(saveButton).toBeTruthy();

    const backButton = fixture.nativeElement.querySelector('[aria-label="Go back"]');
    expect(backButton).toBeTruthy();
  });

  it('should support keyboard navigation', () => {
    const form = fixture.nativeElement.querySelector('form');
    const inputs = form.querySelectorAll('input, button, ion-select');

    // All interactive elements should be keyboard-accessible
    inputs.forEach((element: HTMLElement) => {
      expect(element.tabIndex).toBeGreaterThanOrEqual(0);
    });
  });

  it('should have minimum touch target size', () => {
    const buttons = fixture.nativeElement.querySelectorAll('ion-button, button');

    buttons.forEach((button: HTMLElement) => {
      const rect = button.getBoundingClientRect();
      expect(rect.width).toBeGreaterThanOrEqual(48); // 48px minimum
      expect(rect.height).toBeGreaterThanOrEqual(48);
    });
  });
});
```

### 7.2 Playwright Accessibility Testing

**File:** `e2e/specs/accessibility.spec.ts`
```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility E2E Tests', () => {
  test('Job list page should be accessible', async ({ page }) => {
    await page.goto('/tabs/jobs');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('Voice command button should be accessible', async ({ page }) => {
    await page.goto('/tabs/jobs');

    const voiceButton = page.locator('[data-testid="voice-button"]');

    // Check ARIA attributes
    await expect(voiceButton).toHaveAttribute('aria-label', 'Activate voice command');
    await expect(voiceButton).toHaveAttribute('role', 'button');

    // Check keyboard accessibility
    await voiceButton.focus();
    await expect(voiceButton).toBeFocused();

    // Check touch target size
    const box = await voiceButton.boundingBox();
    expect(box?.width).toBeGreaterThanOrEqual(56);
    expect(box?.height).toBeGreaterThanOrEqual(56);
  });

  test('Forms should have proper labels', async ({ page }) => {
    await page.goto('/jobs/new');

    // All inputs should have labels
    const inputs = page.locator('input, ion-input, ion-select');
    const count = await inputs.count();

    for (let i = 0; i < count; i++) {
      const input = inputs.nth(i);
      const id = await input.getAttribute('id');
      const label = page.locator(`label[for="${id}"]`);
      await expect(label).toBeVisible();
    }
  });
});
```

### 7.3 Accessibility Testing Tools Configuration

| Tool | Purpose | Compliance Level |
|------|---------|------------------|
| **axe-core** | Automated WCAG testing | WCAG 2.1 Level AA |
| **jest-axe** | Unit test accessibility | WCAG 2.1 Level AA |
| **@axe-core/playwright** | E2E accessibility | WCAG 2.1 Level AA |
| **Lighthouse** | Performance + A11y audits | WCAG 2.1 Level AA |
| **Manual Testing** | Screen reader compatibility | WCAG 2.1 Level AA |

---

## 8. Visual Regression Testing

### 8.1 Playwright Screenshots

#### 8.1.1 Visual Regression Test Suite

**File:** `e2e/specs/visual-regression.spec.ts`
```typescript
import { test, expect } from '@playwright/test';

test.describe('Visual Regression Tests', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/tabs/jobs');
  });

  test('Job list page visual snapshot', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('job-list.png', {
      fullPage: true,
      animations: 'disabled'
    });
  });

  test('Job detail page visual snapshot', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.click('[data-testid="job-card"]');
    await page.waitForLoadState('networkidle');

    await expect(page).toHaveScreenshot('job-detail.png', {
      fullPage: true,
      animations: 'disabled'
    });
  });

  test('Cost entry form visual snapshot', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.click('[data-testid="job-card"]');
    await page.click('[data-testid="add-cost-button"]');
    await page.waitForSelector('[data-testid="cost-form"]');

    await expect(page).toHaveScreenshot('cost-form.png', {
      animations: 'disabled'
    });
  });

  test('Offline mode banner visual snapshot', async ({ page, context }) => {
    await page.goto('/tabs/jobs');
    await context.setOffline(true);
    await page.waitForSelector('.offline-banner');

    await expect(page).toHaveScreenshot('offline-banner.png', {
      fullPage: true
    });
  });

  test.describe('Mobile viewport', () => {
    test.use({ viewport: { width: 375, height: 667 } }); // iPhone SE

    test('Mobile job list visual snapshot', async ({ page }) => {
      await page.goto('/tabs/jobs');
      await page.waitForLoadState('networkidle');

      await expect(page).toHaveScreenshot('mobile-job-list.png', {
        fullPage: true,
        animations: 'disabled'
      });
    });
  });
});
```

### 8.2 Percy Integration (Optional)

#### 8.2.1 Percy Configuration

**File:** `.percy.yml`
```yaml
version: 2
snapshot:
  widths:
    - 375  # Mobile
    - 768  # Tablet
    - 1280 # Desktop
  min-height: 1024
  enable-javascript: true
  discovery:
    allowed-hostnames:
      - localhost
    network-idle-timeout: 750
```

#### 8.2.2 Percy Test Integration

**File:** `e2e/specs/percy-visual.spec.ts`
```typescript
import { test } from '@playwright/test';
import percySnapshot from '@percy/playwright';

test.describe('Percy Visual Tests', () => {
  test('Capture job list', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await percySnapshot(page, 'Job List Page');
  });

  test('Capture job detail', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.click('[data-testid="job-card"]');
    await percySnapshot(page, 'Job Detail Page');
  });

  test('Capture voice confirmation dialog', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.click('[data-testid="job-card"]');
    await page.click('[data-testid="voice-button"]');
    await page.click('[data-testid="stop-recording"]');
    await percySnapshot(page, 'Voice Confirmation Dialog');
  });
});
```

---

## 9. Test Data Management

### 9.1 Test Data Strategy

#### 9.1.1 Factory Pattern for Test Data

**File:** `src/testing/factories/job.factory.ts`
```typescript
import { Job, CreateJobDto } from '@findogai/shared-types';
import { faker } from '@faker-js/faker';

export class JobFactory {
  static create(overrides?: Partial<Job>): Job {
    return {
      id: faker.string.uuid(),
      tenantId: 'test-tenant-123',
      jobNumber: faker.number.int({ min: 1000, max: 9999 }),
      title: faker.company.catchPhrase(),
      status: 'active',
      budget: faker.number.int({ min: 5000, max: 50000 }),
      currency: 'EUR',
      vatRate: 20,
      createdAt: faker.date.past(),
      createdBy: faker.string.uuid(),
      updatedAt: faker.date.recent(),
      updatedBy: faker.string.uuid(),
      deletedAt: null,
      ...overrides
    };
  }

  static createDto(overrides?: Partial<CreateJobDto>): CreateJobDto {
    return {
      title: faker.company.catchPhrase(),
      budget: faker.number.int({ min: 5000, max: 50000 }),
      currency: 'EUR',
      vatRate: 20,
      ...overrides
    };
  }

  static createMany(count: number, overrides?: Partial<Job>): Job[] {
    return Array.from({ length: count }, () => this.create(overrides));
  }
}
```

**File:** `src/testing/factories/cost.factory.ts`
```typescript
import { Cost, CreateCostDto } from '@findogai/shared-types';
import { faker } from '@faker-js/faker';

export class CostFactory {
  static create(overrides?: Partial<Cost>): Cost {
    return {
      id: faker.string.uuid(),
      tenantId: 'test-tenant-123',
      jobId: faker.string.uuid(),
      category: faker.helpers.arrayElement(['material', 'labor', 'equipment', 'subcontractor', 'other']),
      subcategory: faker.commerce.productName(),
      amount: faker.number.float({ min: 100, max: 5000, fractionDigits: 2 }),
      currency: 'EUR',
      quantity: faker.number.int({ min: 1, max: 100 }),
      unit: faker.helpers.arrayElement(['pcs', 'kg', 'm', 'm2', 'm3', 'hours']),
      description: faker.commerce.productDescription(),
      date: faker.date.recent(),
      createdAt: faker.date.past(),
      createdBy: faker.string.uuid(),
      updatedAt: faker.date.recent(),
      updatedBy: faker.string.uuid(),
      deletedAt: null,
      ...overrides
    };
  }

  static createDto(overrides?: Partial<CreateCostDto>): CreateCostDto {
    return {
      jobId: faker.string.uuid(),
      category: faker.helpers.arrayElement(['material', 'labor', 'equipment']),
      amount: faker.number.float({ min: 100, max: 5000, fractionDigits: 2 }),
      description: faker.commerce.productName(),
      ...overrides
    };
  }

  static createMany(count: number, overrides?: Partial<Cost>): Cost[] {
    return Array.from({ length: count }, () => this.create(overrides));
  }
}
```

#### 9.1.2 Test Data Builders

**File:** `src/testing/builders/job.builder.ts`
```typescript
import { Job } from '@findogai/shared-types';

export class JobBuilder {
  private job: Partial<Job>;

  constructor() {
    this.job = {
      id: 'job-123',
      tenantId: 'test-tenant',
      jobNumber: 1001,
      title: 'Test Job',
      status: 'active',
      budget: 10000,
      currency: 'EUR',
      vatRate: 20,
      createdAt: new Date(),
      createdBy: 'user-123',
      updatedAt: new Date(),
      updatedBy: 'user-123',
      deletedAt: null
    };
  }

  withId(id: string): this {
    this.job.id = id;
    return this;
  }

  withTitle(title: string): this {
    this.job.title = title;
    return this;
  }

  withBudget(budget: number): this {
    this.job.budget = budget;
    return this;
  }

  withStatus(status: 'active' | 'on_hold' | 'completed' | 'canceled'): this {
    this.job.status = status;
    return this;
  }

  asDeleted(): this {
    this.job.deletedAt = new Date();
    return this;
  }

  build(): Job {
    return this.job as Job;
  }
}

// Usage
const job = new JobBuilder()
  .withTitle('Custom Job')
  .withBudget(25000)
  .withStatus('on_hold')
  .build();
```

### 9.2 Test Data Seeding for E2E Tests

#### 9.2.1 Firebase Emulator Seeding

**File:** `e2e/seed/seed-data.ts`
```typescript
import { initializeApp } from 'firebase/app';
import { getFirestore, collection, doc, setDoc } from 'firebase/firestore';
import { JobFactory } from '../../src/testing/factories/job.factory';
import { CostFactory } from '../../src/testing/factories/cost.factory';

export async function seedTestData() {
  const app = initializeApp({
    projectId: 'demo-test',
    apiKey: 'fake-key'
  });

  const db = getFirestore(app);
  const tenantId = 'test-tenant-123';

  // Seed jobs
  const jobs = JobFactory.createMany(10, { tenantId });
  for (const job of jobs) {
    await setDoc(doc(db, `tenants/${tenantId}/jobs/${job.id}`), job);
  }

  // Seed costs for first job
  const costs = CostFactory.createMany(20, {
    tenantId,
    jobId: jobs[0].id
  });
  for (const cost of costs) {
    await setDoc(doc(db, `tenants/${tenantId}/costs/${cost.id}`), cost);
  }

  console.log('✅ Test data seeded');
}

// Run if called directly
if (require.main === module) {
  seedTestData();
}
```

#### 9.2.2 E2E Test with Seeded Data

**File:** `e2e/specs/with-seeded-data.spec.ts`
```typescript
import { test, expect } from '@playwright/test';
import { seedTestData } from '../seed/seed-data';

test.describe('Tests with Seeded Data', () => {
  test.beforeAll(async () => {
    await seedTestData();
  });

  test('should display seeded jobs', async ({ page }) => {
    await page.goto('/tabs/jobs');

    const jobCards = page.locator('[data-testid="job-card"]');
    await expect(jobCards).toHaveCount(10);
  });

  test('should display seeded costs', async ({ page }) => {
    await page.goto('/tabs/jobs');
    await page.click('[data-testid="job-card"]');

    const costItems = page.locator('[data-testid="cost-item"]');
    await expect(costItems).toHaveCount(20);
  });
});
```

### 9.3 Test Data Cleanup

**File:** `e2e/helpers/cleanup.ts`
```typescript
import { initializeApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';
import { clearFirestoreData } from '@firebase/rules-unit-testing';

export async function cleanupTestData() {
  try {
    await clearFirestoreData({ projectId: 'demo-test' });
    console.log('✅ Test data cleaned up');
  } catch (error) {
    console.error('❌ Failed to cleanup test data:', error);
  }
}
```

---

## 10. Frontend Testing Patterns

### 10.1 Container vs Presentational Component Testing

#### 10.1.1 Container Component Test

**Container components** connect to NgRx store and handle business logic.

```typescript
// job-list-container.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { JobListContainerComponent } from './job-list-container.component';
import { JobsActions } from '../data-access/jobs.actions';
import { selectAllJobs, selectJobsLoading } from '../data-access/jobs.selectors';

describe('JobListContainerComponent', () => {
  let component: JobListContainerComponent;
  let fixture: ComponentFixture<JobListContainerComponent>;
  let store: MockStore;

  const mockJobs = [
    { id: '1', title: 'Job 1' },
    { id: '2', title: 'Job 2' }
  ];

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [JobListContainerComponent],
      providers: [
        provideMockStore({
          selectors: [
            { selector: selectAllJobs, value: mockJobs },
            { selector: selectJobsLoading, value: false }
          ]
        })
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent(JobListContainerComponent);
    component = fixture.componentInstance;
  });

  it('should dispatch loadJobs on init', () => {
    spyOn(store, 'dispatch');

    component.ngOnInit();

    expect(store.dispatch).toHaveBeenCalledWith(JobsActions.loadJobs());
  });

  it('should select jobs from store', (done) => {
    component.jobs$.subscribe(jobs => {
      expect(jobs).toEqual(mockJobs);
      done();
    });
  });

  it('should handle delete action', () => {
    spyOn(store, 'dispatch');

    component.onDelete('job-1');

    expect(store.dispatch).toHaveBeenCalledWith(
      JobsActions.deleteJob({ id: 'job-1' })
    );
  });
});
```

#### 10.1.2 Presentational Component Test

**Presentational components** receive data via @Input and emit events via @Output.

```typescript
// job-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { JobListComponent } from './job-list.component';

describe('JobListComponent', () => {
  let component: JobListComponent;
  let fixture: ComponentFixture<JobListComponent>;

  const mockJobs = [
    { id: '1', title: 'Job 1', budget: 10000 },
    { id: '2', title: 'Job 2', budget: 20000 }
  ];

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [JobListComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(JobListComponent);
    component = fixture.componentInstance;
  });

  it('should display jobs', () => {
    component.jobs = mockJobs;
    fixture.detectChanges();

    const jobCards = fixture.nativeElement.querySelectorAll('[data-testid="job-card"]');
    expect(jobCards.length).toBe(2);
    expect(jobCards[0].textContent).toContain('Job 1');
  });

  it('should emit delete event', () => {
    spyOn(component.delete, 'emit');

    component.onDeleteClick('job-1');

    expect(component.delete.emit).toHaveBeenCalledWith('job-1');
  });

  it('should display empty state when no jobs', () => {
    component.jobs = [];
    fixture.detectChanges();

    const emptyState = fixture.nativeElement.querySelector('[data-testid="empty-state"]');
    expect(emptyState).toBeTruthy();
    expect(emptyState.textContent).toContain('No jobs found');
  });
});
```

### 10.2 Testing Signals (Angular 20+)

```typescript
// signal-component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { signal } from '@angular/core';

describe('SignalComponent', () => {
  let component: SignalComponent;
  let fixture: ComponentFixture<SignalComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [SignalComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(SignalComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should update signal value', () => {
    // Initial value
    expect(component.count()).toBe(0);

    // Update signal
    component.increment();
    expect(component.count()).toBe(1);

    // Update again
    component.increment();
    expect(component.count()).toBe(2);
  });

  it('should trigger view update when signal changes', () => {
    component.increment();
    fixture.detectChanges();

    const countElement = fixture.nativeElement.querySelector('[data-testid="count"]');
    expect(countElement.textContent).toBe('1');
  });

  it('should work with computed signals', () => {
    component.count.set(5);

    expect(component.doubleCount()).toBe(10);
  });
});
```

### 10.3 Testing Offline Behavior

```typescript
// offline-behavior.spec.ts
describe('Offline Behavior', () => {
  let component: JobDetailComponent;
  let fixture: ComponentFixture<JobDetailComponent>;
  let networkService: jasmine.SpyObj<NetworkService>;
  let syncQueueService: jasmine.SpyObj<SyncQueueService>;

  beforeEach(async () => {
    const networkSpy = jasmine.createSpyObj('NetworkService', [], {
      isOnline: signal(true),
      online$: of(true)
    });

    const syncQueueSpy = jasmine.createSpyObj('SyncQueueService',
      ['addOperation', 'processPendingOperations']
    );

    await TestBed.configureTestingModule({
      imports: [JobDetailComponent],
      providers: [
        { provide: NetworkService, useValue: networkSpy },
        { provide: SyncQueueService, useValue: syncQueueSpy }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(JobDetailComponent);
    component = fixture.componentInstance;
    networkService = TestBed.inject(NetworkService) as jasmine.SpyObj<NetworkService>;
    syncQueueService = TestBed.inject(SyncQueueService) as jasmine.SpyObj<SyncQueueService>;
  });

  it('should show offline indicator when offline', () => {
    networkService.isOnline = signal(false);
    fixture.detectChanges();

    const offlineChip = fixture.nativeElement.querySelector('.offline-chip');
    expect(offlineChip).toBeTruthy();
    expect(offlineChip.textContent).toContain('Offline');
  });

  it('should queue operations when offline', async () => {
    networkService.isOnline = signal(false);

    await component.save({ title: 'Updated Title' });

    expect(syncQueueService.addOperation).toHaveBeenCalledWith(
      jasmine.objectContaining({
        type: 'UPDATE_JOB',
        payload: jasmine.objectContaining({
          title: 'Updated Title'
        })
      })
    );
  });

  it('should process queue when coming back online', async () => {
    networkService.isOnline = signal(false);
    networkService.online$ = new BehaviorSubject(false);

    // Queue operations while offline
    await component.save({ title: 'Update 1' });
    await component.save({ title: 'Update 2' });

    // Come back online
    (networkService.online$ as BehaviorSubject<boolean>).next(true);

    await fixture.whenStable();

    expect(syncQueueService.processPendingOperations).toHaveBeenCalled();
  });
});
```

---

## 11. Backend Testing Patterns

### 11.1 Cloud Functions Unit Testing

```typescript
// allocateSequence.test.ts
import { describe, test, expect, jest, beforeEach } from '@jest/globals';
import { allocateSequence } from '../callable/allocateSequence';

describe('allocateSequence', () => {
  let mockFirestore: any;
  let mockTransaction: any;

  beforeEach(() => {
    mockTransaction = {
      get: jest.fn(),
      set: jest.fn()
    };

    mockFirestore = {
      runTransaction: jest.fn((callback) => callback(mockTransaction))
    };
  });

  test('should allocate first sequence number', async () => {
    mockTransaction.get.mockResolvedValue({
      exists: false
    });

    const result = await allocateSequence(
      { tenantId: 'test-123', type: 'jobNumber' },
      { auth: { uid: 'user-123' } }
    );

    expect(result.sequenceNumber).toBe(1);
    expect(mockTransaction.set).toHaveBeenCalled();
  });

  test('should increment existing counter', async () => {
    mockTransaction.get.mockResolvedValue({
      exists: true,
      data: () => ({ current: 42 })
    });

    const result = await allocateSequence(
      { tenantId: 'test-123', type: 'jobNumber' },
      { auth: { uid: 'user-123' } }
    );

    expect(result.sequenceNumber).toBe(43);
  });

  test('should throw for unauthenticated user', async () => {
    await expect(
      allocateSequence(
        { tenantId: 'test-123', type: 'jobNumber' },
        { auth: undefined }
      )
    ).rejects.toThrow('User must be authenticated');
  });
});
```

### 11.2 Testing Firestore Triggers

```typescript
// onJobCreated.test.ts
import { describe, test, expect, jest } from '@jest/globals';
import { onJobCreated } from '../triggers/onJobCreated';
import * as functions from 'firebase-functions-test';

const testEnv = functions();

describe('onJobCreated trigger', () => {
  test('should create audit log on job creation', async () => {
    const snap = testEnv.firestore.makeDocumentSnapshot(
      {
        id: 'job-123',
        title: 'Test Job',
        createdBy: 'user-123'
      },
      'tenants/test-tenant/jobs/job-123'
    );

    const context = {
      params: {
        tenantId: 'test-tenant',
        jobId: 'job-123'
      }
    };

    await onJobCreated(snap, context);

    // Verify audit log was created
    // (Mock or use Firebase Emulator for actual verification)
  });
});
```

---

## 12. Testing Best Practices

### 12.1 General Best Practices

1. **Follow AAA Pattern**: Arrange-Act-Assert
2. **One assertion per test**: Focus on single behavior
3. **Test behavior, not implementation**: Focus on what, not how
4. **Use descriptive test names**: `should [do something] when [condition]`
5. **Keep tests isolated**: No dependencies between tests
6. **Mock external dependencies**: Don't test Firebase, test your code
7. **Use factories for test data**: Consistent, reusable test data
8. **Clean up after tests**: Reset mocks, clear state

### 12.2 Test Naming Conventions

```typescript
// ✅ GOOD - Descriptive, behavior-focused
it('should display error message when login fails', () => {});
it('should queue operation when offline', () => {});
it('should retry STT request with fallback provider', () => {});

// ❌ BAD - Implementation-focused, unclear
it('should call authService.login', () => {});
it('should set error to true', () => {});
it('test retry logic', () => {});
```

### 12.3 Test Organization

```
src/
├── app/
│   ├── features/
│   │   ├── jobs/
│   │   │   ├── components/
│   │   │   │   ├── job-list.component.ts
│   │   │   │   ├── job-list.component.spec.ts       # Unit tests
│   │   │   ├── data-access/
│   │   │   │   ├── jobs.effects.ts
│   │   │   │   ├── jobs.effects.spec.ts             # Effect tests
│   │   │   ├── services/
│   │   │   │   ├── jobs.service.ts
│   │   │   │   ├── jobs.service.spec.ts             # Service tests
│   │   │   ├── jobs.integration.spec.ts             # Integration tests
e2e/
├── specs/
│   ├── critical-flows.spec.ts                       # E2E critical paths
│   ├── voice-commands.spec.ts                       # Voice E2E
│   ├── accessibility.spec.ts                        # A11y E2E
│   ├── visual-regression.spec.ts                    # Visual tests
├── pages/                                           # Page objects
├── seed/                                            # Test data seeding
```

---

## 13. CI/CD Integration

### 13.1 GitHub Actions Workflow

**File:** `.github/workflows/test.yml`
```yaml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:ci

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Start Firebase Emulators
        run: |
          npm install -g firebase-tools
          firebase emulators:start --only firestore,auth,functions &
          sleep 30

      - name: Run integration tests
        run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Run E2E tests
        run: npm run e2e:ci

      - name: Upload E2E results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/

  performance-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun

  security-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: npm audit --audit-level=high

      - name: Run OWASP ZAP scan
        run: ./scripts/security-scan.sh

      - name: Run Firestore Rules tests
        run: npm run test:rules

  accessibility-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Run accessibility tests
        run: npm run test:a11y

      - name: Upload accessibility report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: accessibility-report
          path: accessibility-report/
```

### 13.2 Test Scripts in package.json

```json
{
  "scripts": {
    "test": "nx test",
    "test:ci": "nx test --coverage --browsers=ChromeHeadless --watch=false",
    "test:watch": "nx test --watch",
    "test:coverage": "nx test --coverage",
    "test:integration": "nx run-many --target=test --all --configuration=integration",
    "test:rules": "firebase emulators:exec --only firestore 'npm run test:rules:run'",
    "test:rules:run": "jest --config=jest.rules.config.js",
    "e2e": "playwright test",
    "e2e:ci": "playwright test --config=playwright.ci.config.ts",
    "e2e:debug": "playwright test --debug",
    "e2e:ui": "playwright test --ui",
    "test:a11y": "playwright test --grep @accessibility",
    "test:performance": "k6 run performance/*.k6.js",
    "test:security": "./scripts/security-scan.sh",
    "test:all": "npm run test:ci && npm run test:integration && npm run e2e:ci && npm run test:a11y"
  }
}
```

---

## 14. Coverage Goals and Metrics

### 14.1 Coverage Targets

| Metric | Target | Minimum |
|--------|--------|---------|
| **Overall Line Coverage** | 85% | 80% |
| **Branch Coverage** | 80% | 75% |
| **Function Coverage** | 85% | 80% |
| **Statement Coverage** | 85% | 80% |

### 14.2 Component-Level Coverage

| Component Type | Target Coverage |
|----------------|-----------------|
| Business Logic (Services) | 95% |
| NgRx Effects | 90% |
| NgRx Reducers | 100% |
| Container Components | 85% |
| Presentational Components | 80% |
| Utility Functions | 95% |
| Cloud Functions | 85% |
| Security Rules | 100% |

### 14.3 Testing Metrics Dashboard

Track the following metrics in CI/CD:

- **Unit Test Pass Rate**: 100% required
- **Integration Test Pass Rate**: 100% required
- **E2E Test Pass Rate**: 95% minimum
- **Code Coverage**: 85% target
- **Performance Tests**: Meet NFR thresholds
- **Security Vulnerabilities**: 0 critical/high
- **Accessibility Violations**: 0 WCAG 2.1 AA violations
- **Visual Regression**: No unintended changes

### 14.4 Quality Gates

**Pre-Commit:**
- Unit tests must pass
- Linting must pass
- Formatting must pass

**Pre-Push:**
- Unit tests must pass
- Code coverage ≥ 80%
- No console.log statements

**Pull Request:**
- All unit tests pass
- All integration tests pass
- E2E tests pass
- Code coverage ≥ 85%
- No critical security vulnerabilities
- No accessibility violations
- Visual regression approved

**Pre-Deployment:**
- All tests pass
- Performance tests meet NFR targets
- Security scan passes
- Accessibility audit passes

---

## Summary

This comprehensive testing strategy ensures **high-quality, reliable, and secure** delivery of FinDogAI with:

✅ **Complete test coverage** across all layers (unit, integration, E2E)
✅ **Performance validation** meeting NFR targets
✅ **Security assurance** through automated scanning and rules testing
✅ **Accessibility compliance** with WCAG 2.1 Level AA
✅ **Visual consistency** through regression testing
✅ **Robust test data management** with factories and builders
✅ **Frontend-specific patterns** for Angular/Ionic components
✅ **Backend validation** for Cloud Functions and Firestore
✅ **CI/CD integration** with automated quality gates

This strategy addresses **all gaps identified** in the Architecture Checklist (Sections 7.2 and 7.3).

---

**Related Documentation:**
- [Frontend Testing Strategy](../frontend-architecture/testing-strategy.md) - Frontend-specific patterns
- [Backend Testing Strategy](../backend-architecture/testing-strategy.md) - Backend-specific patterns
- [Coding Standards](./coding-standards.md) - Testing standards and conventions
- [Deployment](./deployment.md) - CI/CD pipeline configuration
- [Development Workflow](./development-workflow.md) - Running tests locally

[Back to Architecture Index](./index.md)
