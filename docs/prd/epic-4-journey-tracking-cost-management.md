# Epic 4: Journey Tracking & Cost Management

**Expanded Goal:** Implement the "Start Journey" voice flow with odometer reading parsing and vehicle lookup, create manual cost entry screens for all five cost categories (Transport, Material, Labor, Machine, Other), and enable advances tracking. This epic completes the core cost capture functionality, delivering the full MVP value proposition of hands-free expense recording with manual fallback options.

### Story 4.1: Start Journey Intent Recognition & Entity Parsing

**As a** developer,
**I want** LLM to parse "Start Journey" voice commands extracting destination, vehicle, and odometer reading,
**so that** journey costs can be automatically created under the active job.

**Acceptance Criteria:**

1. Intent Recognition prompt updated to include StartJourney intent alongside SetActiveJob
2. Prompt template: "Extract intent and entities from: '{transcription}'. Intents: SetActiveJob, StartJourney. Return JSON: {intent, entities}."
3. For "I'm going to Brno in Transporter, odometer 12345" (or Czech equivalent), LLM returns: `{intent: "StartJourney", entities: {destination: "Brno", vehicleIdentifier: "Transporter", odometerReading: 12345}}`
4. vehicleIdentifier can be: numeric vehicleNumber ("1"), partial vehicle name ("Trans"), or full name ("Transporter")
5. odometerReading extracted as integer (handles various formats: "12345", "twelve thousand three hundred forty-five", "1 2 3 4 5")
6. If vehicleIdentifier is numeric, query Firestore for vehicle with matching vehicleNumber
7. If vehicleIdentifier is text, query Firestore for vehicle with name containing text (case-insensitive)
8. If multiple vehicles match, return first vehicle (sorted by vehicleNumber)
9. If no vehicle matches, return error: "Vehicle not found. Please say vehicle number or name."
10a. Recognize optional inline job override phrase (e.g., "for job 123") and parse as `jobTarget` entity (job number or name)

10. If no Active Job set and no inline job override specified, return error: "No active job. Please set an active job or specify a job in your command."; if inline job override is present, proceed with that job
11. Mock mode returns hardcoded intent: `{intent: "StartJourney", entities: {destination: "Brno", vehicleIdentifier: "1", odometerReading: 12345}}`
12. Parsed intent with matched vehicle data passed to Confirmation Loop

### Story 4.2: Start Journey Voice Flow with Journey Event Creation

**As a** craftsman,
**I want** to say "I'm going to [destination] in [vehicle], odometer [reading]" and have a journey event created,
**so that** I can track transportation costs without manual entry while driving.

**Acceptance Criteria:**

1. User taps microphone button, says: "I'm going to Brno in Transporter, odometer 12345" (or Czech: "Jedu do Brna v Transporteru, kilometr 12345")
2. STT transcribes → LLM parses → Vehicle queried → TTS confirms
3. Voice Confirmation Modal displays: "✓ Journey to **Brno** | Vehicle: **[1] Transporter** | Odometer: **12,345 [unit]**" (unit label from businessProfile)
4. TTS plays: "Starting journey to Brno in Transporter, odometer one two three four five. Say 'yes' to confirm or 'no' to retry."
5. User responds "yes" (voice) OR taps Accept button → Journey event created under targeted job (inline override if present, else Active Job) at `/tenants/{tenantId}/jobs/{jobId}/events/{eventId}`
6. Event document: ordinalNumber (sequence), type: "journey_start", timestamp (now), data: {destination, vehicle: {full vehicle object copy}, odometerStart: 12345, odometerEnd: null, calculatedDistance: null, calculatedCost: null}, createdAt, createdBy: {uid, memberNumber, displayName}
7. Success message (toast): "Journey to Brno started" (in user's language)
8. Journey event displayed in Job Detail → Events timeline: "[ordinalNumber] Journey to Brno - Transporter - 12,345 [unit]" (unit label from businessProfile)
9. Offline development mode only: mock pipeline saves event locally; in production offline, voice interactions are disabled (no recording); manual Start Journey entry remains available.
10. End-to-end flow completes in <10 seconds on 4G
11. Error handling: "No vehicles found. Add a vehicle in settings first."
12. If odometer reading missing from transcription, prompt: "Please say odometer reading."
13. Hands-free confirmation: After TTS ends, KWS listens for "yes"/"no"; voice response triggers corresponding action
14. Optional job override: Saying "for job [id/name]" targets that job for this operation only and does not change the Active Job
16. If inline job override specified, TTS and modal include the targeted job ("for job [number] [title]"); event is created under that job; Active Job remains unchanged

15. If no Active Job and no job override specified, prompt: "Say job number or set an Active Job first"


### Story 4.3: Manual Cost Entry - Transport Category

**As a** craftsman,
**I want** to manually enter transportation costs with vehicle and distance details,
**so that** I can record journey costs when voice is unavailable or make corrections.

**Acceptance Criteria:**

1. Job Detail screen has "Add Cost" button → opens category selector (Transport, Material, Labor, Machine, Other)
2. Select "Transport" → opens Transport Cost entry form (job context is implicit - current job from Job Detail screen)
3. Form fields: Vehicle (dropdown showing `[vehicleNumber] Name`), **Input Mode toggle (Distance / Manual Amount)**, Description (optional text), Date/Time (default: now)
4. **Single Vehicle Auto-Selection:** If tenant has exactly one vehicle, skip vehicle dropdown entirely and auto-select that vehicle; display vehicle info as read-only text "[vehicleNumber] Name" below form header; vehicle is pre-populated in cost data; user can still access all vehicles via Settings if they want to add more
5. **Distance Mode:** Distance in km/miles (number with unit label from businessProfile), Amount auto-calculated (distance × vehicle.ratePerDistanceUnit) displayed read-only
6. **Manual Amount Mode:** Amount (number with currency), Distance field hidden, Vehicle still required (or auto-selected if single vehicle)
7. On save, cost created in `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}` where jobId is the current job context
8. Cost document: ordinalNumber (sequence), category: "transport", amount, vehicle: {full vehicle object copy with vehicleNumber, name, distanceUnit, ratePerDistanceUnit}, distance (null if manual amount mode), description, timestamp, createdAt, createdBy: {uid, memberNumber, displayName}
9. Cost displayed in Job Detail → Costs tab: "[ordinalNumber] Transport - [vehicleNumber] Vehicle Name - X [unit] - Y CZK" (or "- Y CZK" if distance is null; unit label from businessProfile)
10. Edit cost: Tap cost item → opens form pre-filled with original mode (distance/manual), allows updates to distance/amount/description/vehicle (vehicle field is dropdown if multiple vehicles, read-only if single vehicle), job context remains unchanged
11. Delete cost: Swipe left or long-press → confirmation: "Delete cost [ordinalNumber]?" → removes from Firestore
12. Job financial summary updates: Total costs recalculated, budget remaining updated
13. Offline mode: Cost CRUD works without network, syncs when online
14. Validation: If Distance Mode, distance must be positive number and vehicle must be selected; if Manual Amount Mode, amount must be positive number and vehicle must be selected
15. **Use case clarification:** Distance Mode allows user to enter "I drove 50 [unit] today" (without odometer readings); unit label from businessProfile; system calculates cost from vehicle rate

### Story 4.4: Manual Cost Entry - Material, Labor, Machine, Other Categories

**As a** craftsman,
**I want** to manually enter costs for materials, labor, machines, and miscellaneous expenses,
**so that** I can track all job-related costs comprehensively.

**Acceptance Criteria:**

1. "Add Cost" category selector includes: Transport (Story 4.3), **Material**, **Labor**, **Machine**, **Other** (job context is implicit - current job from Job Detail screen)
2. **Material form:** **Input Mode toggle (Quantity + Unit Price / Total Amount Only)**, Description (required), Supplier (optional), Date/Time
3. **Material Quantity Mode:** Quantity (number of units), Unit Price (price per unit), Amount auto-calculated (quantity × unitPrice) displayed read-only
4. **Material Total Amount Mode:** Amount (number with currency), Quantity and Unit Price fields hidden
5. Material cost document: ordinalNumber, category: "material", amount, quantity (null if total amount mode), unitPrice (null if total amount mode), description, supplier, timestamp, createdAt, createdBy: {uid, memberNumber, displayName}
6. **Labor form:** Team Member (dropdown `[teamMemberNumber] Name`), **Input Mode toggle (Hours / Manual Amount)**, Description (optional), Date/Time
7. **Single Team Member Auto-Selection:** If tenant has exactly one team member, skip team member dropdown entirely and auto-select that member; display member info as read-only text "[teamMemberNumber] Name" below form header; member is pre-populated in cost data
8. **Labor Hours Mode:** Hours (number), Amount auto-calculated (hours × teamMember.hourlyRate) displayed read-only
9. **Labor Manual Amount Mode:** Amount (number with currency), Hours field hidden, Team Member still required (or auto-selected if single member)
10. Labor cost document: ordinalNumber, category: "labor", amount, teamMember: {full team member object copy}, hours (null if manual amount mode), description, timestamp, createdAt, createdBy: {uid, memberNumber, displayName}
11. **Machine form:** Machine (dropdown `[machineNumber] Name`), **Input Mode toggle (Hours / Manual Amount)**, Description (optional), Date/Time
12. **Single Machine Auto-Selection:** If tenant has exactly one machine, skip machine dropdown entirely and auto-select that machine; display machine info as read-only text "[machineNumber] Name" below form header; machine is pre-populated in cost data
13. **Machine Hours Mode:** Hours (number), Amount auto-calculated (hours × machine.hourlyRate) displayed read-only
14. **Machine Manual Amount Mode:** Amount (number with currency), Hours field hidden, Machine still required (or auto-selected if single machine)
15. Machine cost document: ordinalNumber, category: "machine", amount, machine: {full machine object copy}, hours (null if manual amount mode), description, timestamp, createdAt, createdBy: {uid, memberNumber, displayName}
16. **Other form:** Amount (number with currency), Description (required), Date/Time
17. Other cost document: ordinalNumber, category: "other", amount, description, timestamp, createdAt, createdBy: {uid, memberNumber, displayName}
18. All costs saved to `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}` where jobId is the current job context
19. All categories support Edit (update amount/description/resource, mode preserved from creation, job context remains unchanged); on Edit, resource field is dropdown if multiple resources exist, read-only if single resource exists
20. Job Detail → Costs tab displays costs grouped by category with sequential ordinalNumbers and totals per category
21. Cost display shows calculated details when available: "Transport - 50 km - 250 CZK" vs "Transport - 250 CZK" (manual), "Material - 10 units × 5 CZK - 50 CZK" vs "Material - 50 CZK" (manual)
22. Privilege enforcement: If membership.status != 'active', "Add Cost" button hidden/disabled

### Story 4.5: Advances Tracking

**As a** craftsman,
**I want** to record client advance payments per job,
**so that** I can track net amounts owed after subtracting advances from total costs.

**Acceptance Criteria:**

1. Job Detail screen has "Advances" tab (alongside Costs, Events)
2. Advances tab displays list: "[ordinalNumber] Amount - Date - Note" with total sum at bottom
3. "Add Advance" button opens advance entry form (job context is implicit - current job from Job Detail screen)
4. Form fields: Amount (required, number with currency from job), Date (default: today), Note (optional, e.g., "Initial deposit", "Payment #2")
5. On save, advance created in `/tenants/{tenantId}/jobs/{jobId}/advances/{advanceId}` where jobId is the current job context
6. Advance document: ordinalNumber (sequence), amount, date, note, createdAt, createdBy: {uid, memberNumber, displayName}
7. Edit advance: Tap item → opens form pre-filled, allows updates to amount/date/note, updatedAt, updatedBy: {uid, memberNumber, displayName}
8. Delete advance: Swipe left → confirmation: "Delete advance [ordinalNumber]?" → removes from Firestore
9. Job Detail header displays financial summary: "Costs: X CZK | Advances: Y CZK | **Net Due: (X - Y) CZK**" (green if positive, red if negative)
10. Net Due calculation updates in real-time as costs/advances added/removed
11. Privilege enforcement: Advances visible and editable to owner and owner’s representative; hidden for team member
12. Offline mode: Advance CRUD works without network, syncs when online

### Story 4.6: Job Costs Summary & Category Breakdown

**As a** craftsman,
**I want** to view job costs broken down by category with totals,
**so that** I can understand where money is being spent per job.

**Acceptance Criteria:**

1. Job Detail → Costs tab displays costs grouped by category: Transport, Material, Labor, Machine, Other
2. Each category section shows: **Category Total: X CZK** with expandable list of individual costs
3. Costs list format: "[ordinalNumber] Description - Amount - Date" with resource info (vehicle/machine/team member if applicable)
4. Tap cost item → opens edit form (Story 4.3/4.4)
5. Top of Costs tab shows **Overall Total: X CZK** (sum of all categories)
6. Budget indicator: If job has budget defined, show progress bar: "X CZK of Y CZK budget (Z% used)" with color coding (green <75%, amber 75-90%, red >90%)
7. Empty state per category: "No transport costs yet. Add or record via voice."
8. Costs sortable by: Date (newest first - default), Amount (highest first), Category
9. Search/filter: Text input filters costs by description/resource name
10. Privilege enforcement: Costs Summary (category totals and overall total) visible to owner and owner’s representative; team member sees the list of costs but not the summary section
11. Export to CSV button (future enhancement - show disabled with tooltip: "Coming soon")
12. Offline mode: Costs display from local cache, sync status indicator if pending writes

---
### Story 4.7: End Journey Voice Flow (Close Journey + Create Transport Cost)

**As a** craftsman,
**I want** to say "End journey, odometer [reading]" and have the app close the current journey and record the cost,
**so that** distance and transport cost are captured without manual entry.

**Acceptance Criteria:**

1. Intent added: `EndJourney` with entities: `odometerReading` (integer), optional `jobTarget` (job number or name)
2. Precondition: An open journey exists for the targeted job (inline override if present, else Active Job) — the most recent `journey_start` event with `odometerEnd == null`
3. On Accept: Update that event with `odometerEnd`, compute `calculatedDistance = max(0, odometerEnd - odometerStart)`, and `calculatedCost = calculatedDistance * vehicle.ratePerDistanceUnit` (rounded per currency rules)
4. Also create a Transport cost at `/tenants/{tenantId}/jobs/{jobId}/costs/{costId}` with: `category: "transport"`, `mode: "distance"`, `distance`, `vehicle` (full copy), `amount: calculatedCost`, ordinalNumber sequence, `createdAt/By`
5. Validation: If no open journey, show "No ongoing journey to end"; if `odometerEnd < odometerStart`, prompt to re-enter
6. TTS confirmation reads back: destination (if known), vehicle, end reading, calculated distance, and cost, ending with "Say 'yes' to confirm or 'no' to retry."; user responds via voice ("yes"/"no") or taps Accept/Retry/Cancel
10. Inline override behavior: Saying "for job [id/name]" targets that job for this operation only and does not change the Active Job

7. Error handling: Missing odometer → prompt; missing rate on vehicle → block completion and prompt to set vehicle rate on the selected vehicle (or choose another vehicle); after rate is set, retry End Journey.
8. Offline: Voice disabled in production offline; manual End Journey remains available (enter end odometer in Events)
9. Tests (mock providers): StartJourney then EndJourney → event updated and cost created; negative/zero distance handled; missing open journey shows error

### Story 4.8: Add Material Cost Voice Flow

**As a** craftsman,
**I want** to say "Material [amount] for [description]" or "Material [qty] by [unit price] [description]",
**so that** I can quickly record material purchases hands-free.

**Acceptance Criteria:**

1. Intent added: `AddMaterialCost` with entities: `amount` OR (`quantity`, `unitPrice`), optional `description`, optional `jobTarget` (job number or name)
2. Amount parsing supports integers/decimals; currency defaults to businessProfile.currency
3. If `quantity` and `unitPrice` present, compute `amount = quantity * unitPrice` (rounded); store all three
4. On Accept: Create cost under targeted job (inline override if present, else Active Job) with `category: "Material"`, fields: `amount`, optional `quantity`, `unitPrice`, `description`, `createdAt/By`, sequential `ordinalNumber`
5. TTS confirmation summarizes parsed values: "Material, 10 × 200 = 2,000 CZK, 'plasterboard'. Say 'yes' to confirm or 'no' to retry."
6. Validation: If parsed amount missing, prompt: "Please say amount"; if no Active Job and no inline job override specified, prompt: "Say job number or name, or set an Active Job first"
7. Privileges: Requires active membership (any role); otherwise show "Permission denied"
8. Offline: Voice disabled in production offline; manual Material entry available in Story 4.4
9. Tests: Variants with amount only, qty×price, with/without description; Czech phrases recognized
10. Inline override behavior: Saying "for job [id/name]" targets that job for this operation only and does not change the Active Job
11. TTS readback includes the targeted job: "for job [number] [title]"
12. Tests: Include inline job override scenario and verify Active Job remains unchanged


### Story 4.9: Record Work Hours (Labor) Voice Flow

**As a** craftsman,
**I want** to say "Labor [hours] hours" (optionally "for [member]" and/or "at [rate]")
**so that** labor time is recorded with correct cost.

**Acceptance Criteria:**

1. Intent added: `AddLaborHours` with entities: `hours` (decimal), optional `teamMemberIdentifier` (name/number), optional `hourlyRate`, optional `jobTarget` (job number or name)
2. Default team member: current user’s team member resource (`teamMembers` where `authUserId == uid`); if a different member is specified, resolve by number or name
3. Rate selection: use provided `hourlyRate` if present; otherwise teamMember.hourlyRate; compute `amount = hours * rate` (rounded)
4. On Accept: Create cost under targeted job (inline override if present, else Active Job) with `category: "Labor"`, `hours`, `rate`, `amount`, and embedded `teamMember` copy (id, number, name, hourlyRate); set `createdAt/By`, `ordinalNumber`
5. TTS reads back: "Labor, 3.5 hours for Petr at 400 CZK/h = 1,400 CZK. Say 'yes' to confirm or 'no' to retry."
10. Inline override behavior: Saying "for job [id/name]" targets that job for this operation only and does not change the Active Job
11. TTS readback includes the targeted job: "for job [number] [title]"
12. Tests: Include inline job override scenario and verify Active Job remains unchanged

6. Validation: Missing hours → prompt; unknown member → prompt to repeat or default to current user
7. Privileges: Requires active membership (any role); otherwise show "Permission denied"
8. Offline: Voice disabled in production offline; manual Labor entry available in Story 4.4
9. Tests: Current user default, specified member, with/without explicit rate; Czech command variants

### Story 4.10: Quick Expense (Other) Voice Flow

**As a** craftsman,
**I want** to say "Expense [amount] [short note]",
**so that** I can capture small miscellaneous costs quickly.

**Acceptance Criteria:**

1. Intent added: `QuickExpense` with entities: `amount`, optional `description`, optional `jobTarget` (job number or name)
2. On Accept: Create cost under targeted job (inline override if present, else Active Job) with `category: "Other"`, `amount`, optional `description`, `createdAt/By`, `ordinalNumber`
3. TTS confirmation: "Other expense, 50 CZK, 'parking'. Say 'yes' to confirm or 'no' to retry."
4. Validation: If amount missing, prompt; if no Active Job and no inline job override, prompt to specify job or set one
8. Inline override behavior: Saying "for job [id/name]" targets that job for this operation only and does not change the Active Job
9. TTS readback includes the targeted job: "for job [number] [title]"
10. Tests: Include inline job override scenario and verify Active Job remains unchanged

5. Privileges: Requires active membership (any role); amounts are visible to all roles
6. Offline: Voice disabled in production offline; manual Other entry available in Story 4.4
7. Tests: Amount only, amount + note; for teamMember role verify Costs Summary is hidden (but cost amounts remain visible)

