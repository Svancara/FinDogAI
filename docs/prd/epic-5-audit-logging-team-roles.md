# Epic 5: Audit Logging & Team Roles

**Expanded Goal:** Implement Cloud Functions audit triggers (onCreate/onUpdate/onDelete) for all collections, scheduled TTL cleanup function, and enforce team member privilege checks in UI and Security Rules. This epic delivers compliance features and multi-user access control, preparing the MVP for team use and regulatory requirements.

### Story 5.1: Cloud Functions Audit Triggers for Jobs & Costs

**As a** system architect,
**I want** Firestore triggers to automatically log all CRUD operations on jobs and costs,
**so that** we have a complete audit trail for compliance and debugging.

**Acceptance Criteria:**

1. Cloud Function `onJobCreate` triggers on `/tenants/{tenantId}/jobs/{jobId}` onCreate
2. Function writes to `/tenants/{tenantId}/audit_logs/{logId}` with: operation: "CREATE", collection: "jobs", documentId: jobId, tenantId, timestamp (server time), author: {compound identity object from job.createdBy}, after: {full job document}, ttl: (timestamp + 1 year)
3. Cloud Function `onJobUpdate` triggers on `/tenants/{tenantId}/jobs/{jobId}` onUpdate
4. Function writes audit log with: operation: "UPDATE", before: {old job document}, after: {new job document}
5. Cloud Function `onJobDelete` triggers on `/tenants/{tenantId}/jobs/{jobId}` onDelete
6. Function writes audit log with: operation: "DELETE", before: {deleted job document}, after: null
7. Same pattern for Cost triggers: `onCostCreate`, `onCostUpdate`, `onCostDelete` on `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}`
8. Audit log documents immutable (Security Rules: write-only from Cloud Functions, no client write access)
9. Triggers execute asynchronously (<500ms execution time, non-blocking to client operations)
10. Local emulator testing: Triggers fire when creating/updating/deleting jobs/costs via app
11. Error handling: If audit write fails, log error but don't block original operation (audit is supplementary)
12. Deployed to Firebase: `firebase deploy --only functions`

### Story 5.2: Cloud Functions Audit Triggers for Resources & Advances

**As a** system architect,
**I want** audit triggers for vehicles, machines, team members, and advances,
**so that** all resource changes are logged for historical tracking.

**Acceptance Criteria:**

1. Cloud Functions created for Vehicles: `onVehicleCreate`, `onVehicleUpdate`, `onVehicleDelete` on `/tenants/{tenantId}/vehicles/{vehicleId}`
2. Cloud Functions created for Machines: `onMachineCreate`, `onMachineUpdate`, `onMachineDelete` on `/tenants/{tenantId}/machines/{machineId}`
3. Cloud Functions created for TeamMembers: `onTeamMemberCreate`, `onTeamMemberUpdate`, `onTeamMemberDelete` on `/tenants/{tenantId}/teamMembers/{teamMemberId}`
4. Cloud Functions created for Advances: `onAdvanceCreate`, `onAdvanceUpdate`, `onAdvanceDelete` on `/tenants/{tenantId}/jobs/{jobId}/advances/{advanceId}`
5. All triggers follow same pattern as Story 5.1: operation, collection, documentId, tenantId, timestamp, author (compound identity object), before/after, ttl
6. Subcollections (costs, advances, events) include parent context in audit log: jobId field for traceability
7. Triggers deployed and tested in emulator: Create/update/delete vehicle → audit log entry created
8. Production deployment: All triggers live and logging operations
9. Monitoring: Cloud Functions console shows successful executions, minimal errors
10. Cost optimization: Triggers use minimal compute (no external API calls, just Firestore writes)

### Story 5.3: Scheduled Audit Log TTL Cleanup Function

**As a** system architect,
**I want** a daily scheduled function to delete audit logs older than 1 year,
**so that** storage costs are controlled while meeting compliance retention requirements.

**Acceptance Criteria:**

1. Cloud Function `scheduledAuditLogCleanup` configured with Cloud Scheduler: runs daily at 2:00 AM UTC
2. Function queries `/tenants/{tenantId}/audit_logs` where `ttl < now()` (finds expired logs)
3. Function deletes expired logs in batches of 500 (Firestore batch write limit)
4. If >500 expired logs exist, function continues in batches until all deleted
5. Function logs summary: "Deleted X audit logs older than 1 year"
6. Function execution time: <30 seconds for typical load (<10,000 expired logs)
7. Error handling: If batch delete fails, retry with exponential backoff (3 retries max)
8. Scheduled function testable via manual trigger: `firebase functions:shell` or HTTP callable wrapper
9. Production deployment: Function scheduled and running automatically
10. Monitoring: Check Cloud Functions logs weekly to confirm cleanup executes successfully

### Story 5.4: Privilege Enforcement in UI (Cost Entry & Financial Visibility)

**As a** business owner,
**I want** UI to hide/disable features based on team member privileges,
**so that** restricted users cannot access unauthorized functionality.

**Acceptance Criteria:**

1. On app init, load membership at `/tenants/{tenantId}/members/{currentUser.uid}` (role, status), and load the team member resource from `/tenants/{tenantId}/teamMembers` where `authUserId == currentUser.uid` for display
2. Store role and status in app state (NgRx/Signals) for reactive UI updates
3. **membership.status != 'active':** Hide/disable "Add Cost" button, "Add Advance" button, voice microphone button (costs-related commands)
4. **membership.status != 'active':** Cost/Advance edit/delete buttons hidden in lists
5. **role = teamMember:** Job financial summary hidden (display placeholders like "Budget: —", "Costs: —", "Net Due: —")
6. **role = teamMember:** Costs tab displays normally (descriptions/dates/amounts); only the "Costs Summary" section is hidden
7. **role = teamMember:** Advances tab hidden; **role = representative:** Advances tab visible (create/edit/delete enabled)
8. **role = teamMember:** Job budget field hidden in job creation/edit forms; **role = representative:** Jobs are read-only (no edits), including budget
9. Privilege tooltips: When feature disabled, tooltip explains: "Your account doesn't have permission to add costs. Contact business owner."
10. Role-based access checks are reactive: If business owner updates a member's role or status, affected user's UI updates on next app data refresh (within 1 minute or on manual refresh)
11. Team member #1 (owner) always has full access regardless of database values
12. If no team member record found for current user, default to no privileges (safety fallback)

### Story 5.5: Firestore Security Rules for Privilege Enforcement

**As a** system architect,
**I want** Security Rules to enforce privilege checks on server-side,
**so that** users cannot bypass UI restrictions via direct API calls.

**Acceptance Criteria:**

1. Security Rules updated to load membership document for the request.auth user
2. Rule helper: `function member(t) { return get(/databases/$(database)/documents/tenants/$(t)/members/$(request.auth.uid)).data }`
3. **Write access to costs:** `allow create, update, delete: if member(t).status == 'active'`
4. **Read/write access to advances:** `allow read, create, update, delete: if member(t).role in ['owner','representative'] && member(t).status == 'active'`
5. **Owner bypass:** `allow read, write: if member(t).role == 'owner'` (owner always has access)
6. **Job writes (incl. budget):** `allow update, delete: if member(t).role == 'owner'` (representative and teamMember cannot modify jobs)
7. Security Rules tested via emulator with multiple test users (owner, representative, teamMember, disabled)
8. Test: User with `status != 'active'` attempting to create cost → permission denied
9. Test: User with `role = teamMember` attempting to read advance → permission denied
10. Rules deployed: `firebase deploy --only firestore:rules`
11. Production validation: Create test user with restricted role or disabled status, verify API calls respect rules
12. Rules commented explaining privilege logic for future maintainers

### Story 5.6: Audit Log Viewer (Owner-only)

**As a** business owner,
**I want** to review audit logs to see who changed what and when,
**so that** I can verify changes and diagnose issues quickly.

**Acceptance Criteria:**

1. Navigation: Entry point in Tenant menu (e.g., Settings › Admin › Audit Logs) is visible only to the owner; representative and teamMember do not see it.
2. Route: `/audit-logs` (tenant-scoped) with breadcrumb showing tenant name.
3. Data source: Read from `/tenants/{tenantId}/audit_logs` ordered by `timestamp` desc; queries use `limit` (default 50) and support pagination (infinite scroll or "Load more").
4. List item fields: Timestamp (localized), Operation badge (CREATE/UPDATE/DELETE), Collection, Document ID, Author (display as "[memberNumber] displayName" from author compound identity object; fallback to author.uid if memberNumber/displayName missing), and a "View" action.
5. Filters: Quick date ranges (Today, Last 7 days, Last 30 days, Custom), Collection (Jobs, Costs, Vehicles, Machines, TeamMembers, Advances, Events), Operation (CREATE/UPDATE/DELETE), Author, and free-text search (Document ID contains).
6. Empty state: "No audit log entries for the selected filters."
7. Permissions: Only owner can open the page. If a non-owner navigates directly (deep link), show 403-style message: "You don't have permission to view audit logs" with a Back button.
8. Offline handling: Viewer disabled offline with banner: "Requires internet connection"; no cached audit log data is shown.
9. Detail view (drawer or modal) opens on "View":
   - CREATE → show "After" snapshot (formatted JSON) with copy-to-clipboard
   - UPDATE → show side-by-side "Before" vs "After" with inline diff highlighting; changed fields expanded by default
   - DELETE → show "Before" snapshot (deleted document), "After: —"
10. Deep link: If the referenced entity still exists, show "Open [Collection] › [#number Title/Name]" button to navigate to that screen; hide when not applicable.
11. PII limits: No voice transcripts or AI intermediate artifacts are displayed (not logged per FR8); only entity snapshots/metadata appear.
12. Export: No export/download from this view (per FR23, audit logs are excluded from exports).
13. Performance: First paint ≤1.5s on broadband for initial 50 items; diff rendering remains responsive for typical documents (<10 KB each).
14. Accessibility: Keyboard navigable; operation badges have accessible labels; timestamps include human-readable and absolute formats (tooltip).
15. Telemetry (optional): Log viewer opens and filter changes for diagnostics only (no PII).
16. Testing:
   - Create/Update/Delete a Job → entries appear with correct metadata
   - UPDATE shows only changed fields highlighted in diff
   - Representative/teamMember denied access (UI hidden + direct-route guard shows 403 message)
   - Date/collection/operation filters narrow results correctly; pagination appends more entries

**Security Rules alignment:**
- Reads: `allow read: if member(t).role == 'owner'`
- Writes: none from client; audit logs are created exclusively by Cloud Functions triggers
- Queries must include `orderBy('timestamp','desc')` and `limit` on the client for performance

---
