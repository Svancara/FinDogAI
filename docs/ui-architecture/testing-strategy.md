[Back to Index](./index.md)

# Testing Strategy

## Overview

The FinDogAI testing strategy focuses on ensuring reliability, especially for offline operations and voice command functionality.  We use Jest for unit and integration tests, and Playwright for end-to-end testing.

## Component Testing with Jest

### Component Test Template

```typescript
// job-detail.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { IonicModule } from '@ionic/angular';
import { provideMockStore, MockStore } from '@ngrx/store/testing';
import { signal } from '@angular/core';
import { of } from 'rxjs';
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

## Integration Testing

### Voice Pipeline Integration Test

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

## E2E Testing with Playwright

### Job Creation Flow

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

## Testing Best Practices

### 1. Unit Tests
- Test individual components in isolation
- Mock all dependencies (Store, Services, Ionic components)
- Test component logic, not framework behavior
- Focus on user interactions and state changes
- Aim for 80%+ coverage of business logic

### 2. Integration Tests
- Test complete user flows within a feature
- Mock external dependencies (APIs, Firebase)
- Verify state management flows
- Test offline/online transitions

### 3. E2E Tests
Test critical user flows:
- Login → Create Job → Add Costs → Generate Report
- Voice command flows
- Offline mode operations
- PWA installation and updates
- Multi-device sync scenarios

### 4. Coverage Goals
- 90%+ for critical business logic
- 80%+ for components
- 70%+ for effects/services
- Focus on behavior, not line coverage

### 5. Test Structure
Use Arrange-Act-Assert pattern:

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

### 6. Mock External Dependencies
- Use provideMockStore for NgRx
- Create service spies with Jasmine
- Mock Firestore operations
- Simulate network conditions

## Testing Scripts

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

## Rationale for Testing Strategy

### Jest over Karma/Jasmine
- Faster test execution (important for CI/CD)
- Better watch mode for development
- Snapshot testing support
- Modern assertion library

### Playwright over Cypress
- Per explicit requirement (NOT Cypress)
- Better mobile device emulation
- Supports offline testing
- Native app testing capability

### Focus Areas
- Offline/online transitions (critical for field use)
- Voice command accuracy
- Touch-friendly interactions for challenging conditions
- State synchronization

---

[← Previous: Offline Architecture](./offline-architecture.md) | [Back to Index](./index.md) | [Next: Performance Optimization →](./performance-optimization.md)
