[Back to Index](./index.md) | [Migration Guide](./business-profile-migration.md) | [Coding Standards](./coding-standards.md)

# Business Profile Implementation Checklist

## Developer Implementation Guide for Resource Visibility Feature

This checklist provides a complete implementation roadmap for the Business Profile resource visibility feature across the FinDogAI application.

---

## Phase 1: Shared Types & Models

### 1.1 Update Shared Types Package
Location: `packages/shared-types/src/entities/business-profile.ts`

- [ ] Add `BusinessProfile` interface with new fields
  ```typescript
  export interface BusinessProfile {
    tenantId: string;
    currency: string;
    vatRate: number;
    distanceUnit: 'km' | 'mi';
    usesMachines: boolean;      // NEW
    usesVehicles: boolean;      // NEW
    usesOtherExpenses: boolean; // NEW
    createdAt: Timestamp;
    updatedAt: Timestamp;
  }
  ```

- [ ] Export `ResourceVisibilityType` union type
  ```typescript
  export type ResourceVisibilityType = 'machines' | 'vehicles' | 'otherExpenses';
  ```

- [ ] Add validation schema (Zod/Joi)
  ```typescript
  export const BusinessProfileSchema = z.object({
    tenantId: z.string(),
    currency: z.enum(['CZK', 'EUR', 'USD']),
    vatRate: z.number().min(0).max(100),
    distanceUnit: z.enum(['km', 'mi']),
    usesMachines: z.boolean().default(true),
    usesVehicles: z.boolean().default(true),
    usesOtherExpenses: z.boolean().default(true),
    createdAt: z.instanceof(Timestamp),
    updatedAt: z.instanceof(Timestamp)
  });
  ```

- [ ] Build and test shared types package
  ```bash
  nx build shared-types
  nx test shared-types
  ```

---

## Phase 2: Backend Implementation

### 2.1 Update Firestore Security Rules
Location: `firestore.rules`

- [ ] Add Business Profile read/update rules
- [ ] Add validation for boolean fields (see migration guide)
- [ ] Test security rules with Firebase Emulator
  ```bash
  firebase emulators:start --only firestore
  ```

### 2.2 Database Migration
Location: `functions/src/migrations/`

- [ ] Create migration Cloud Function (see migration guide)
- [ ] Test migration locally with emulator
- [ ] Create rollback function (optional)
- [ ] Document migration in deployment notes

### 2.3 Tenant Initialization Updates
Location: `functions/src/triggers/onTenantCreate.ts`

- [ ] Update tenant creation to include new fields in Business Profile
  ```typescript
  const businessProfile: BusinessProfile = {
    tenantId: event.params.tenantId,
    currency: 'CZK',
    vatRate: 21,
    distanceUnit: 'km',
    usesMachines: true,      // NEW
    usesVehicles: true,      // NEW
    usesOtherExpenses: true, // NEW
    createdAt: FieldValue.serverTimestamp(),
    updatedAt: FieldValue.serverTimestamp()
  };
  ```

- [ ] Test with new tenant creation in emulator

---

## Phase 3: Frontend Services

### 3.1 Business Profile Service
Location: `apps/mobile/src/app/core/services/business-profile.service.ts`

- [ ] Create `BusinessProfileService` with methods:
  - [ ] `getBusinessProfile(): Observable<BusinessProfile>`
  - [ ] `updateBusinessProfile(updates: Partial<BusinessProfile>): Promise<void>`
  - [ ] `shouldShowResourceType(type: ResourceVisibilityType): Observable<boolean>`
  - [ ] `getDefaultJobSettings(): Observable<JobDefaults>`

- [ ] Implement reactive caching with `shareReplay(1)`
- [ ] Add error handling
- [ ] Write unit tests
  ```bash
  nx test mobile --testFile=business-profile.service.spec.ts
  ```

### 3.2 Update Existing Services
Locations: Various service files

- [ ] **JobsService** - Inject `BusinessProfileService` for defaults
- [ ] **VehiclesService** - Check visibility before operations
- [ ] **MachinesService** - Check visibility before operations
- [ ] **CostsService** - Filter cost categories based on visibility

---

## Phase 4: UI Components

### 4.1 Business Profile Settings Component
Location: `apps/mobile/src/app/features/settings/business-profile/`

- [ ] Create settings form component
- [ ] Add three toggle switches:
  - [ ] "Working with machines" toggle
  - [ ] "Using vehicles" toggle
  - [ ] "Other business expenses" toggle
- [ ] Add help text: "Hide resource types your business doesn't use to simplify the interface. This won't delete existing data."
- [ ] Implement reactive form with validation
- [ ] Add save/cancel actions
- [ ] Add success/error toasts
- [ ] Test component in isolation

**Template Example:**
```typescript
@Component({
  selector: 'app-business-profile-settings',
  template: `
    <form [formGroup]="profileForm" (ngSubmit)="onSave()">
      <!-- Currency, VAT, Distance Unit fields -->

      <ion-list-header>
        <ion-label>My business involves:</ion-label>
      </ion-list-header>

      <ion-item>
        <ion-toggle formControlName="usesMachines">
          Working with machines
        </ion-toggle>
      </ion-item>

      <ion-item>
        <ion-toggle formControlName="usesVehicles">
          Using vehicles
        </ion-toggle>
      </ion-item>

      <ion-item>
        <ion-toggle formControlName="usesOtherExpenses">
          Other business expenses
        </ion-toggle>
      </ion-item>

      <ion-note>
        Hide resource types your business doesn't use to simplify the interface.
        This won't delete existing data.
      </ion-note>

      <ion-button type="submit" expand="block">Save</ion-button>
    </form>
  `
})
export class BusinessProfileSettingsComponent {
  // Implementation
}
```

### 4.2 Resources Management Screen
Location: `apps/mobile/src/app/features/resources/`

- [ ] Update `ResourcesComponent` to inject `BusinessProfileService`
- [ ] Add visibility streams:
  ```typescript
  showVehicles$ = this.businessProfileService.shouldShowResourceType('vehicles');
  showMachines$ = this.businessProfileService.shouldShowResourceType('machines');
  ```
- [ ] Update template with `@if` conditionals:
  ```html
  @if (showVehicles$ | async) {
    <ion-tab-button tab="vehicles">
      <ion-label>Vehicles</ion-label>
    </ion-tab-button>
  }
  ```
- [ ] Test all tab visibility combinations
- [ ] Ensure Team Members tab is ALWAYS visible

### 4.3 Cost Entry Components
Location: `apps/mobile/src/app/features/costs/`

- [ ] **Cost Form Component** - Filter cost type options
  ```typescript
  @if (showVehicles$ | async) {
    <ion-select-option value="transport">Transport</ion-select-option>
  }
  @if (showMachines$ | async) {
    <ion-select-option value="machine">Machine</ion-select-option>
  }
  @if (showOtherExpenses$ | async) {
    <ion-select-option value="expense">Other Expense</ion-select-option>
  }
  ```

- [ ] **Cost List Component** - Filter displayed categories
- [ ] **Cost Filter Component** - Hide disabled cost type filters
- [ ] Test with all visibility combinations

### 4.4 Resource Selection Components
Location: `apps/mobile/src/app/features/resources/`

- [ ] **Add Resource Button** - Conditionally show based on type
  ```typescript
  @if (showMachines$ | async) {
    <ion-button (click)="addMachine()">
      <ion-icon name="add"></ion-icon>
      Add Machine
    </ion-button>
  }
  ```

- [ ] Test auto-selection logic still works with visibility flags

### 4.5 Voice Flow Components
Location: `apps/mobile/src/app/features/voice/`

- [ ] Update voice command processor to check visibility
- [ ] Filter available actions based on Business Profile
- [ ] Update voice prompts to exclude hidden resource types
- [ ] Test voice flows with different visibility combinations

---

## Phase 5: State Management (NgRx)

### 5.1 Business Profile State
Location: `apps/mobile/src/app/state/business-profile/`

- [ ] Create feature state for Business Profile
- [ ] Add actions: `loadProfile`, `updateProfile`, `updateProfileSuccess`, `updateProfileFailure`
- [ ] Add reducer to handle state updates
- [ ] Add effects for Firestore operations
- [ ] Add selectors:
  - [ ] `selectBusinessProfile`
  - [ ] `selectShowMachines`
  - [ ] `selectShowVehicles`
  - [ ] `selectShowOtherExpenses`

### 5.2 Update Existing State
Location: Various state modules

- [ ] Update resources state to use visibility selectors
- [ ] Update costs state to filter categories
- [ ] Test state changes propagate to UI

---

## Phase 6: Testing

### 6.1 Unit Tests

- [ ] Business Profile Service tests
  ```bash
  nx test mobile --testFile=business-profile.service.spec.ts
  ```
- [ ] Component tests for visibility logic
- [ ] State management tests (actions, reducers, effects, selectors)
- [ ] Validation schema tests

### 6.2 Integration Tests

- [ ] Test Business Profile settings form
- [ ] Test resource tabs visibility
- [ ] Test cost form filtering
- [ ] Test voice flow filtering
- [ ] Test auto-selection with visibility flags

### 6.3 E2E Tests (Playwright)
Location: `apps/mobile-e2e/src/`

- [ ] Test toggle switches in Business Profile settings
- [ ] Test UI updates reactively when flags change
- [ ] Test navigation with hidden tabs
- [ ] Test cost creation with filtered options
- [ ] Test all combinations:
  - All enabled (default)
  - Only machines disabled
  - Only vehicles disabled
  - Only expenses disabled
  - Multiple disabled
  - All disabled except team members

### 6.4 Manual Testing Checklist

- [ ] Create new tenant → Verify defaults are all `true`
- [ ] Toggle off machines → Verify UI hides machine elements
- [ ] Toggle off vehicles → Verify UI hides vehicle elements
- [ ] Toggle off expenses → Verify UI hides expense elements
- [ ] Toggle all off → Verify only team member and material options remain
- [ ] Create costs with filtered options
- [ ] Test auto-selection with one resource
- [ ] Test offline sync with profile changes
- [ ] Test on iOS device
- [ ] Test on Android device
- [ ] Test on web browser

---

## Phase 7: Documentation

### 7.1 Code Documentation

- [ ] Add JSDoc comments to all public methods
- [ ] Document component inputs/outputs
- [ ] Add inline comments for complex logic
- [ ] Update README files in affected modules

### 7.2 User Documentation

- [ ] Update user guide with Business Profile settings
- [ ] Add screenshots of toggle switches
- [ ] Explain what each toggle does
- [ ] Document that existing data is preserved

### 7.3 Developer Documentation

- [ ] Update architecture docs (✓ Already completed)
- [ ] Update coding standards (✓ Already completed)
- [ ] Update API specification (✓ Already completed)
- [ ] Create migration guide (✓ Already completed)

---

## Phase 8: Deployment

### 8.1 Pre-Deployment

- [ ] Code review completed
- [ ] All tests passing (unit, integration, e2e)
- [ ] Manual testing completed
- [ ] Documentation updated
- [ ] Migration script tested in staging

### 8.2 Deployment Sequence

1. [ ] **Deploy shared-types package**
   ```bash
   nx build shared-types
   npm publish packages/shared-types
   ```

2. [ ] **Deploy backend (Cloud Functions)**
   ```bash
   firebase deploy --only functions
   ```

3. [ ] **Deploy security rules**
   ```bash
   firebase deploy --only firestore:rules
   ```

4. [ ] **Run database migration**
   - Execute migration Cloud Function
   - Monitor logs
   - Verify completion

5. [ ] **Deploy frontend (Mobile App)**
   ```bash
   nx build mobile --configuration production
   firebase deploy --only hosting
   ```

6. [ ] **Deploy to app stores** (if applicable)
   - Build iOS app
   - Build Android app
   - Submit to stores

### 8.3 Post-Deployment

- [ ] Verify all tenants have new fields
- [ ] Monitor error logs for 24 hours
- [ ] Test in production with test tenant
- [ ] Notify users of new feature
- [ ] Monitor support channels for issues

---

## Phase 9: Validation & Monitoring

### 9.1 Production Validation

- [ ] Check Firebase Console → Firestore for data integrity
- [ ] Verify security rules working correctly
- [ ] Test with real user account
- [ ] Confirm offline sync working
- [ ] Verify performance (no degradation)

### 9.2 Monitoring Setup

- [ ] Add analytics events for toggle changes
  ```typescript
  this.analytics.logEvent('business_profile_updated', {
    usesMachines,
    usesVehicles,
    usesOtherExpenses
  });
  ```
- [ ] Set up alerts for errors related to Business Profile
- [ ] Monitor query performance on Business Profile collection
- [ ] Track feature adoption rate

---

## Rollback Procedure

If critical issues are discovered:

1. [ ] Redeploy previous frontend version
2. [ ] Restore previous security rules (if needed)
3. [ ] Document issues encountered
4. [ ] Plan fixes for next deployment

---

## Success Criteria

Feature is complete when:

- [ ] All checklist items completed
- [ ] All tests passing (100% for new code)
- [ ] Code review approved
- [ ] Documentation complete
- [ ] Successfully deployed to production
- [ ] Zero critical bugs in first 7 days
- [ ] User feedback positive
- [ ] Performance metrics unchanged

---

## Estimated Effort

| Phase | Estimated Time |
|-------|----------------|
| Phase 1: Shared Types | 2 hours |
| Phase 2: Backend | 4 hours |
| Phase 3: Services | 6 hours |
| Phase 4: UI Components | 12 hours |
| Phase 5: State Management | 6 hours |
| Phase 6: Testing | 8 hours |
| Phase 7: Documentation | 4 hours |
| Phase 8: Deployment | 3 hours |
| Phase 9: Validation | 2 hours |
| **Total** | **47 hours (~6 days)** |

---

## Key Contacts

- **Architecture Questions:** Winston (Architect Agent)
- **Backend Issues:** Backend Team Lead
- **Frontend Issues:** Frontend Team Lead
- **Testing Issues:** QA Lead
- **Deployment Issues:** DevOps Team

---

*For implementation questions, refer to [Coding Standards](./coding-standards.md) or [Migration Guide](./business-profile-migration.md)*
