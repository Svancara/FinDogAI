# Epic 1: Foundation & Authentication Infrastructure

**Expanded Goal:** Establish the technical foundation for FinDogAI by setting up the Angular/Ionic/Capacitor monorepo, configuring Firebase services (Auth, Firestore, Functions), implementing multi-tenant authentication with Security Rules, deploying a basic "health check" screen, and proving the complete CI/CD pipeline works. This epic delivers a deployable application that validates the tech stack end-to-end.

### Story 1.1: Project Initialization & Monorepo Setup

**As a** developer,
**I want** a properly configured monorepo with Angular/Ionic/Capacitor structure,
**so that** I can develop the mobile app, Cloud Functions, and shared types in a unified workspace.

**Acceptance Criteria:**

1. Monorepo created using Nx or Turborepo with workspace configuration
2. `/packages/mobile-app` initialized with Angular 20+ and Ionic 8+
3. `/packages/functions` initialized with Cloud Functions (Node.js v20)
4. `/packages/shared-types` created with TypeScript interfaces
5. `/packages/e2e-tests` initialized with Playwright
6. Root-level `package.json` with workspace scripts (`npm run dev`, `npm run build`, `npm test`)
7. `.gitignore` configured (node_modules, .env, build artifacts)
8. `README.md` with setup instructions and architecture overview
9. All packages build successfully without errors
10. Dev server runs locally on `http://localhost:4200`

### Story 1.2: Firebase Project Configuration

**As a** developer,
**I want** Firebase project configured with europe-west1 region and necessary services enabled,
**so that** the app can authenticate users and persist data with GDPR compliance.

**Acceptance Criteria:**

1. Firebase project created via Firebase Console (project ID: `findogai-mvp` or similar)
2. Firestore database initialized in `europe-west1` (Belgium) region
3. Firebase Authentication enabled with Email/Password provider
4. Firebase Hosting enabled for PWA deployment
5. Cloud Functions configured for `europe-west1` deployment
6. Firebase Storage enabled with `europe-west1` location
7. Firebase CLI installed and authenticated locally (`firebase login`)
8. `.firebaserc` file created with project alias
9. `firebase.json` configured for Hosting, Functions, Firestore rules
10. Firebase Emulators installed (Auth, Firestore, Functions, Storage)
11. Emulator configuration in `firebase.json` with ports defined
12. `npm run emulators` starts all emulators successfully


### Story 1.2b: Sequential Number Allocation (Foundational)

**As a** system architect,
**I want** server-assigned sequential numbers for tenant-level resources and per-job ordinal sequences,
**so that** numbers are unique, voice-friendly, and work both online and offline.

**Acceptance Criteria:**

1. Cloud Function (HTTPS callable) `allocateSequence({ tenantId, sequence })` implemented; sequences supported:
   - `jobNumber`, `teamMemberNumber`, `vehicleNumber`, `machineNumber` (tenant-level)
   - `ordinalNumber` per job for: `/jobs/{jobId}/costs/*`, `/jobs/{jobId}/advances/*`, `/jobs/{jobId}/events/*`
2. Sequence allocation uses Firestore transactions to guarantee strictly increasing integers with no duplicates under concurrency.
3. OnCreate assignment triggers backfill missing sequence fields for offline-created documents:
   - Jobs: set `jobNumber` if absent
   - Team Members: set `teamMemberNumber` if absent
   - Vehicles/Machines: set `vehicleNumber` / `machineNumber` if absent
   - Per-job subcollections (costs/advances/events): set `ordinalNumber` if absent
4. Owner bootstrap (Story 1.3): The initial team member for the tenant is reserved as `teamMemberNumber: 1`.
5. Emulator tests: Online (callable) and offline (trigger) paths allocate without duplicates across ≥100 parallel creates per sequence; placeholders ('—') display in UI until numbers are assigned on sync where applicable.
6. Sequence state persisted under `/tenants/{tenantId}/_counters/{sequenceName}` (or equivalent); README documents usage and error handling.

### Story 1.3: Multi-Tenant Authentication & User Registration

**As a** new user,
**I want** to register with email/password and have a tenant automatically created,
**so that** my data is isolated from other users in the system.

**Acceptance Criteria:**

1. Registration screen with fields: email, password, displayName
2. Form validation: email format, password strength (min 8 chars), required fields
3. On successful registration, Firebase Auth user created
4. On success, a new tenant is created at `/tenants/{tenantId}` (server-generated ID) with `createdBy: {uid: user.uid, memberNumber: 1, displayName}`, `createdAt`
5. Membership document created at `/tenants/{tenantId}/members/{user.uid}` with `{ role: 'owner', status: 'active' }`
6. Index document created at `/user_tenants/{user.uid}/memberships/{tenantId}` for listing/selecting tenants (read-only; maintained by Cloud Functions)
7. Initial profile docs created under `/tenants/{tenantId}`: `personProfile` (displayName, email, language: cs, aiSupportEnabled: true, createdAt) and `businessProfile` (currency: CZK, vatRate: 21%, distanceUnit: km, createdAt); create team member resource `teamMembers/{teamMemberId}` for the owner with `teamMemberNumber: 1`, `authUserId: user.uid`, `name: displayName`
8. Success message displayed: "Welcome, [displayName]! Your account is ready."
9. User automatically logged in and redirected to home screen
10. Registration errors handled gracefully (email already exists, weak password, network failure)

### Story 1.3a: User–Tenant Mapping Service (Get-or-Create)

**As a** system architect,
**I want** a backend service/Cloud Function that resolves or creates the user→tenant mapping on first sign-in,
**so that** every authenticated user is scoped to a valid tenant and Security Rules can enforce isolation via `tenant_id`.

**Acceptance Criteria:**

1. Implement HTTPS callable `getOrCreateTenantForUser({ uid, displayName, email })`.
2. Behavior:
   - If a mapping exists at `user_tenants/{uid}` OR an index exists at `/user_tenants/{uid}/memberships/{tenantId}`, return the existing `tenantId`.
   - If none exists, atomically create:
     - `/tenants/{tenantId}` with `createdBy: {uid, memberNumber: 1, displayName}`, `createdAt`
     - `/tenants/{tenantId}/members/{uid}` with `{ role: 'owner', status: 'active' }`
     - `/user_tenants/{uid}/memberships/{tenantId}` (read-only index for listing/selecting tenants)
     - `user_tenants/{uid}` with `{ tenantId, createdAt }`
     - Team member resource at `/tenants/{tenantId}/teamMembers/{teamMemberId}` with `teamMemberNumber: 1`, `authUserId: uid`, `name: displayName`
3. Idempotency: Safe to call repeatedly without creating duplicates; concurrent calls do not create multiple tenants.
4. After success, client invokes `setActiveTenantClaim({ tenantId })` (Story 1.3b) and force-refreshes the ID token.
5. Emulator tests: First-time call creates tenant + mapping; subsequent calls return the same `tenantId` with no duplicates.



### Story 1.3b: Active Tenant Custom Claim

**As a** system architect,
**I want** a Cloud Function to set/update the user's active tenant custom claim,
**so that** the app and Security Rules can reliably scope access by `tenant_id`.

**Acceptance Criteria:**

1. Cloud Function (HTTPS callable) `setActiveTenantClaim({ tenantId })` validates that the caller has membership at `/tenants/{tenantId}/members/{uid}` and sets a custom Auth claim `tenant_id = tenantId` for the current user.
2. Registration flow (Story 1.3): After creating the tenant + owner membership, client invokes `setActiveTenantClaim` and force-refreshes the ID token so claims are available immediately.
3. Invitation redemption (Story 1.8): After membership creation, client invokes `setActiveTenantClaim` and force-refreshes the ID token.
4. App init: If `tenant_id` claim is missing/stale but `/user_tenants/{uid}/memberships/{tenantId}` exists, the app prompts or selects the appropriate tenant and calls `setActiveTenantClaim` to reconcile.
5. Emulator tests: Non-members cannot set claims (permission denied). Owners and invited members can set claims; after setting, Security Rules allow access to the claimed tenant; switching tenants updates the claim and access accordingly.

### Story 1.4: User Login & Session Management

**As a** registered user,
**I want** to log in with my email/password,
**so that** I can access my data across devices.

**Acceptance Criteria:**

1. Login screen with fields: email, password
2. "Remember me" checkbox (persists session locally)
3. On successful login, Firebase Auth session established
4. Custom claim `tenant_id` retrieved and stored in app state
5. User redirected to home screen after login
6. Login errors handled: "Invalid email/password", "User not found", "Too many attempts"
7. "Forgot password" link triggers Firebase password reset email
8. Session persists across app restarts (if "Remember me" checked)
9. Logout button clears session and returns to login screen
10. Logged-in users cannot access registration screen (redirect to home)

### Story 1.5: Firestore Security Rules for Multi-Tenant Isolation

**As a** system architect,
**I want** Firestore Security Rules enforcing tenant-scoped data access,
**so that** users cannot read or write other tenants' data.

**Acceptance Criteria:**

1. Security Rules file `firestore.rules` created
2. Rule: Users can access `/tenants/{tenantId}/**` only if a membership exists at `/tenants/{tenantId}/members/{request.auth.uid}`; authorization is role-based:
   - owner: full read/write
   - representative: jobs read-only; no audit_logs read; no exports
   - teamMember: read of jobs via public fields only (jobNumber, title; active jobs only); no advances read; no resources write; no exports; may create/update/delete costs
2a. Rule: Team member job reads use sanitized collection `/tenants/{tenantId}/jobs_public/{jobId}` (fields: jobNumber, title, status); allow read only when `member(t).role == 'teamMember' && resource.data.status == 'active'`.
3. Rule: Unauthenticated users have no read/write access (except public collections if needed in future)
4. Rule: Audit logs (`/tenants/{tenantId}/audit_logs/{logId}`) are readable in-app by owner only; representative and team member have no read access. Writes occur only via Cloud Functions triggers; no client writes.
5. Rules deployed to Firebase via `firebase deploy --only firestore:rules`
6. Manual testing: User A cannot read User B's jobs (returns permission denied)
7. Manual testing: Unauthenticated request to Firestore returns permission denied
8. Security Rules validated via Firebase Emulator Suite (unit tests for rules)
9. Rules include comments explaining tenant isolation logic
10. Rules version controlled in git repository
11. Rule: All documents under `/tenants/{tenantId}/**` must include a `tenantId` field equal to the path `tenantId`
12. Rule: Membership documents (`/tenants/{tenantId}/members/{uid}`) can only be created/updated/deleted by owners; members may only update benign self-fields (e.g., `lastSeenAt`)
13. Rule: `/user_tenants/{uid}/memberships/{tenantId}` is read-only to the client (writes via Cloud Functions only)

### Story 1.6: Basic Health Check Screen & CI/CD Validation

**As a** developer,
**I want** a deployable "health check" screen that validates Firebase connectivity,
**so that** I can confirm the deployment pipeline works end-to-end.

**Acceptance Criteria:**

1. Home screen displays after login: "FinDogAI Health Check"
2. Screen shows: Firebase Auth status (✓ Authenticated as [email])
3. Screen shows: Firestore connectivity status (✓ Connected to [region])
4. Screen shows: Current tenant ID (✓ Tenant: [tenantId])
5. Screen shows: App version and build timestamp
6. Health check writes a test document to `/tenants/{tenantId}/healthCheck` and reads it back
7. If write/read succeeds, display "✓ Firestore Read/Write OK"
8. If offline, display "⚠ Offline Mode - Sync pending"
9. PWA deployed to Firebase Hosting via `firebase deploy --only hosting`
10. Deployed app accessible at `https://findogai-mvp.web.app` (or custom domain)
11. Health check screen functional in deployed PWA
12. Capacitor Android build generates APK successfully (`npx cap build android`)

### Story 1.7: Tenant Invitations (Owner)

**As a** business owner,
**I want** to invite team members with specific roles,
**so that** they can securely join my tenant and collaborate.

**Acceptance Criteria:**

1. "Invite Member" dialog available from Team Members screen with fields: Email (optional), Role: representative | teamMember
2. On "Create Invite", an HTTPS callable Cloud Function creates `/tenants/{tenantId}/invites/{inviteId}` with: `codeHash` (hashed server-side), `expiresAt` (TTL ≥72h), `createdBy: {uid, memberNumber, displayName}`, `presetRole`, `status: pending`
3. Function returns a single-use invite code/link; code itself is not stored in Firestore (only hash)
4. Invites list shows pending/consumed/expired with basic metadata; owners can revoke (delete) pending invites
5. Cloud Function protected by App Check and rate limiting
6. Firebase Emulator tests validate invite creation and listing

### Story 1.8: Invitation Redemption & Membership Linking

**As a** team member,
**I want** to redeem an invitation code/link and join the correct tenant,
**so that** I can access and write data for that business.

**Acceptance Criteria:**

1. Onboarding flow supports "Join a team" with invite code/link input
2. HTTPS callable `redeemInvite(code)` validates `codeHash`, TTL, single-use, and optional email binding, and is protected by App Check and rate limiting; on success it:
   - Creates membership at `/tenants/{tenantId}/members/{uid}` with the preset role
   - Creates index at `/user_tenants/{uid}/memberships/{tenantId}` (read-only; maintained by Functions)
   - Creates team member resource at `/tenants/{tenantId}/teamMembers/{teamMemberId}` with `teamMemberNumber` allocated via `allocateSequence` (HTTPS callable if online; onCreate trigger if offline) and `authUserId: uid`
3. On success, app switches to the joined tenant and displays confirmation
4. If invalid/expired/already-used, show appropriate error and retry option; do not leak tenant existence
5. Emulator tests validate end-to-end redemption and Security Rules for membership-gated access