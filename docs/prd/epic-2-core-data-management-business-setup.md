# Epic 2: Core Data Management & Business Setup

**Expanded Goal:** Implement the foundational data entities (Jobs, Vehicles, Machines, Team Members) with full CRUD operations, sequential ID generation via Firestore counters, and enable users to configure Business and Person Profiles. This epic establishes the business context that voice flows will reference, delivering a functional job management system before voice features are added.

**CRITICAL ARCHITECTURAL PRINCIPLE:** Costs and Events collections will NEVER use references to resources (Vehicles, Machines, Team Members). Instead, they will store complete copies of resource data at the time of creation. This enables resource deletion without referential integrity checking and provides audit trail of historical resource states.

### Story 2.1: Jobs CRUD with Sequential Job Numbers

**As a** craftsman,
**I want** to create, view, edit, and archive jobs with auto-assigned job numbers,
**so that** I can organize my work and reference jobs by number in voice commands.

**Acceptance Criteria:**

1. Jobs List screen displays all jobs sorted by status (active, completed, archived), then by jobNumber ascending; records without jobNumber yet (offline-created, pending allocation) display '—' and sort after numbered entries until sync assigns a number
2. Each job card shows: `[jobNumber] Title - Status` with financial summary (total costs, budget, remaining)
3. "Create Job" button navigates to job creation form
4. Job creation form fields: title (required), description, budget (optional), currency (default from businessProfile), vatRate (default from businessProfile)
5. On save (online), `jobNumber` is allocated by HTTPS callable (transactional) and returned to the client; offline, show placeholder '—' and the number is assigned on sync by an onCreate trigger
6. New job saved to `/tenants/{tenantId}/jobs/{jobId}` with: jobNumber, title, description, status: "active", currency, vatRate, budget, createdAt, createdBy (current team member ID)
7. Job detail screen displays all job fields with Edit and Archive buttons
8. Edit job updates: title, description, budget, currency, vatRate, updatedAt, updatedBy
9. Archive job changes status to "archived" (soft delete, not physically removed)
10. Empty state message: "No jobs yet. Create your first job to get started."
11. Offline mode: Job CRUD works without network, syncs when online

### Story 2.2: Vehicles Management with Rate Configuration

**As a** craftsman,
**I want** to manage my vehicles with per-distance rates,
**so that** journey costs can be calculated automatically based on odometer readings.

**Acceptance Criteria:**

1. Resources Management screen has "Vehicles" tab
2. Vehicles list displays: `[vehicleNumber] Name - Rate: X CZK/km` (or /mi based on businessProfile distanceUnit)
3. "Add Vehicle" button navigates to vehicle creation form
4. Vehicle form reads distanceUnit from businessProfile and displays single rate field labeled "Rate per [Km/Mile]" accordingly
5. Vehicle form fields: name (required, e.g., "Transporter", "Škoda Octavia"), rate (required, labeled based on distanceUnit)
6. On save (online), `vehicleNumber` is allocated by HTTPS callable (transactional) and returned to the client; offline, the number is assigned on sync by an onCreate trigger
7. Vehicle saved to `/tenants/{tenantId}/vehicles/{vehicleId}` with: vehicleNumber, name, distanceUnit (copied from businessProfile), ratePerDistanceUnit, createdAt, createdBy
8. Edit vehicle updates: name, rate, updatedAt, updatedBy (distanceUnit remains immutable from creation time)
9. Delete vehicle removes from Firestore immediately without reference checking (costs/events contain full vehicle copies)
10. Empty state: "No vehicles yet. Add your first vehicle to track journey costs."
11. Vehicle deletion shows confirmation: "Delete [vehicleNumber] Name? This cannot be undone."

### Story 2.3: Machines Management with Hourly Rates

**As a** craftsman,
**I want** to manage my machines/equipment with hourly rates,
**so that** machine labor costs can be tracked per job.

**Acceptance Criteria:**

1. Resources Management screen has "Machines" tab
2. Machines list displays: `[machineNumber] Name - Rate: X CZK/hour`
3. "Add Machine" button navigates to machine creation form
4. Machine form fields: name (required, e.g., "Excavator", "Drill"), hourlyRate (required)
5. On save (online), `machineNumber` is allocated by HTTPS callable (transactional) and returned to the client; offline, the number is assigned on sync by an onCreate trigger
6. Machine saved to `/tenants/{tenantId}/machines/{machineId}` with: machineNumber, name, hourlyRate, createdAt, createdBy
7. Edit machine updates: name, hourlyRate, updatedAt, updatedBy
8. Delete machine removes from Firestore immediately without reference checking (costs/events contain full machine copies)
9. Empty state: "No machines yet. Add equipment to track machine labor costs."
10. Machine deletion shows confirmation: "Delete [machineNumber] Name? This cannot be undone."

### Story 2.4: Team Members Management with Roles

**As a** business owner,
**I want** to manage team members and assign roles,
**so that** I can control who can add costs and view financial details.

**Acceptance Criteria:**

1. Resources Management screen has "Team Members" tab
2. Team members list displays: `[teamMemberNumber] Name - Rate: X CZK/hour` with role badge (Owner | Representative | Team Member)
3. Registering user auto-created as team member #1 (resource) and as an owner membership (from Story 1.3) — this is the business owner
4. "Add Team Member" opens an "Invite Member" dialog
5. Invite form fields: email (optional), role select: representative | teamMember; after the invite is redeemed, the owner can set name/hourlyRate on the team member resource
6. On "Create Invite", a Cloud Function creates `/tenants/{tenantId}/invites/{inviteId}` and returns a single-use code/link (TTL; single-use)
7. Upon invite redemption, membership is created at `/tenants/{tenantId}/members/{uid}` with selected role and `status: 'active'`; a team member resource is created at `/tenants/{tenantId}/teamMembers/{teamMemberId}` with `teamMemberNumber` and `authUserId: uid`
8. Edit team member updates: name, hourlyRate, updatedAt, updatedBy (role is managed on the membership and editable by owners only; team member #1 cannot change their owner status)
9. Delete team member removes the resource document; membership removal is a separate owner-only action (not implied by resource deletion)
10. **Owner protection:** Team member #1 cannot be deleted - delete button hidden/disabled with tooltip: "Business owner (Member #1) cannot be deleted"
11. Non-owner team member deletion shows confirmation: "Delete [teamMemberNumber] Name? This cannot be undone."
12. Empty state: "No team members yet" should never occur (owner is always present as #1)

### Story 2.5: Business Profile Configuration

**As a** business owner,
**I want** to configure business-wide defaults (currency, VAT rate, distance unit),
**so that** new jobs inherit these settings and I don't repeat data entry.

**Acceptance Criteria:**

1. Settings screen has "Business Profile" section
2. Fields displayed: Currency (dropdown: CZK, EUR, USD with ISO 4217 codes), VAT Rate (percentage, default 21%), Distance Unit (radio: Kilometers / Miles)
3. On first access, fields pre-filled from `/tenants/{tenantId}/businessProfile` (created in Story 1.3)
4. Save button updates businessProfile document with: currency, vatRate, distanceUnit, updatedAt
5. Success message: "Business profile updated"
6. New jobs created after update use new defaults
7. Existing jobs retain their original settings (not retroactively changed)
8. Validation: VAT rate must be 0-100%, currency must be valid ISO code
9. Offline mode: Changes saved locally, sync when online
10. Settings persist across app restarts

### Story 2.6: Person Profile & Language Selection

**As a** user,
**I want** to configure my personal preferences (display name, language),
**so that** the app UI and voice interactions use my preferred language.

**Acceptance Criteria:**

1. Settings screen has "Person Profile" section
2. Fields displayed: Display Name (text), Email (read-only, from Firebase Auth), Language (dropdown: Czech / English), AI Support (toggle: On/Off; default On)
3. On first access, fields pre-filled from `/tenants/{tenantId}/personProfile` (created in Story 1.3)
4. Language change immediately saves to Firestore (`personProfile.language`) before UI update to ensure persistence
5. After save completes, language change immediately updates current screen (Person Profile) UI text (Czech ↔ English using Angular i18n)
6. Language preference applied to all subsequent screens and app restarts
7. Voice provider configuration field: preferredVoiceProvider (dropdown: Google Cloud / OpenAI Whisper / Platform Native) for developers
8. Language preference passed to STT/TTS providers (cs-CZ or en-US locale)
9. Save button (for displayName/voiceProvider/AI Support) updates personProfile document with: displayName, preferredVoiceProvider, aiSupportEnabled, updatedAt
10. Success message: "Profile updated" (in newly selected language)
11. Offline mode: Changes saved locally, sync when online
12. Logout button present at bottom of settings screen
13. AI Support toggle persists to `personProfile.aiSupportEnabled` (boolean). Default: true
14. When AI Support is Off: Mic tap triggers platform-native TTS: "AI support is disabled. Use manual entry." No STT/LLM/TTS requests are made; header shows "AI Disabled"
15. When AI Support is On but the device is offline: Mic tap triggers platform-native TTS: "Offline: AI support unavailable." Header shows "Offline · AI Unavailable"; no audio is recorded or queued
