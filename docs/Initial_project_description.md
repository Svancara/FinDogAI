# FinDogAI Project Description

## Idea

- The mobile application should help small entrepreneurs and craftsmen record the costs and advances associated with the implementation of their orders (jobs). Currently, a craftsman has to remember in the evening after work what he bought and for how much money, how many kilometers he drove and how long he worked. This application should help him with this. The application should be as voice-controlled as possible so that it is not necessary to type on the mobile keyboard. The application should use AI for the simplest possible data entry.

- The application must be built following an offline-first methodology, so that the user can use it even without an internet connection. The data will be synchronized when the connection is restored.

- The application uses Google Firebase/Firestore for data storage.

- The idea can be summarized as **voice-controlled mobile AI assistant for recording job costs**. The goal is maximum simplicity and minimizing manual data entry.

## Examples

1. A craftsman leaves for an order in the morning.

2. While driving, the craftsman activates the FinDogAI application and says: "I am driving my Transporter to Brno to handle an order for Mr. Smith."

3. The application AI sumarises and repeats this text to confirm that the voice input was correctly recognized. The application will say: "I understand that you are driving your Volkswagen Transporter to Brno to handle an order for Mr. Smith."

4. The application will ask: "What is the odometer reading?".

5. The craftsman answers, the application AI sumarises and repeats the entered data and save it in the database for the order called "Smith, Brno". If the order does not yet exist, the AI ​​will announce this and help with its creation.

The conversation can also take place in such a way that in the morning it is determined that the order "Smith, Brno" will be worked on all day and all input data will then be automatically saved for this order. So craftsman can say: "Set today's active order to Smith, Brno." The application will say: "I understand that you want to set today's active order to Smith, Brno. Is this correct?" The craftsman answers "Yes" and the application saves the active order.

Other details:

- The application should be voice-activated, similar to Siri or Google Assistant, if it is possible.
- The application must have the simplest possible UI.
- The data will be saved in the database, from which an overview of the costs and invoices will later be generated.
- The application is primarily for mobile use, but it should also be possible to use it on the desktop.
- The application must be multi-tenant. Each customer must have their own data isolated from other customers.
- The application must be secure.
- The application must support multiple languages. 
- The application must support multiple currencies.
- The application must run gracefully in the offline mode.



---

## Basic concept and key functions

We can call the application "**FinDogAI**" for example. Its main strength will be in the conversational interface that does not burden the craftsman.

### **A. Voice control and conversational AI (The heart of the application ❤️)**

This is the most important part. The AI ​​must not only convert speech to text, but also understand the context and intent of the user (so-called Natural Language Understanding - NLU).

* **Intent recognition:** The AI ​​must understand what the user wants to do.
* `"I'm starting to work on an order for Mr. Smith."` -> Intent: **Set active job to "Smith"**, **Record date and time of start of work**
* `"I'm going to Brno to Mr. Smith's place in the Transporter."` -> Intent: **Start a journey**. The AI ​​identifies: means of transport, destination, order.
* `"I bought screws in Hornbach for five hundred crowns."` -> Intent: **Record material**. The AI ​​identifies: shop, item, price.
* `"I'm finishing today."` -> Intent: **Record date and time of end of work**.
* **Additional questions:** If the AI ​​doesn't have all the necessary information, it will actively ask.
* User: `"I'm going to Novak."` -> AI: `"What car are you driving and what's the odometer reading?"`
* User: `"I bought paint."` -> AI: `"How much did it cost?"`
* **Working with an active job:** If in the morning the user selects an "active job" (`"Today I'm working on the job Novak, Brno."`).
All other records (journeys, materials, human labor, machine labor, overhead, comments, pictures) are then automatically assigned to it until they change it.

#### Voice activation and wake word detection (true hands‑free)

Some technical researches have been done and the results are in the [ExternalDocs folder](./docs/ExternalDocs).
So the following part needs to be revised. 

- Objective: enable safe, truly hands‑free operation while driving or on site. The app should listen locally for a dedicated wake phrase (e.g., “Hey FinDog”).
- Approach:
  - Always‑on, on‑device keyword spotting (KWS) for low power and privacy. Options: Porcupine (Picovoice), Vosk KWS, or platform‑native capabilities where available.
  - After wake word is detected, switch to streaming Speech‑to‑Text (STT), then NLU/intent recognition.
  - Provide audible chime + visual indicator for states: “Listening…”, “Thinking…”, “Confirming…”.
- UX & safety:
  - Initial setup wizard lets user choose: wake word mode vs push‑to‑talk button. Wake word can be disabled per user or per vehicle.
  - Driving mode: larger UI, minimal confirmations, noise suppression, auto‑repeat summaries to reduce eyes‑off‑road time.
  - False accepts/rejects tuning (sensitivity slider per language); easy “Cancel/Stop” voice command.
- Multilingual support:
  - Per‑language wake word models and phonetic variants (Czech, English, German, Polish, Slovak, ...). Allow optional custom wake words (with caution and testing prompts).
- Privacy & power:
  - KWS runs fully on device; no audio sent to cloud until wake word is detected. Auto‑suspend listening on low battery or after N minutes of inactivity.
- Web vs hybrid packaging:
  - PWA fallback: reliable push‑to‑talk and limited KWS feasibility via WebAudio (device‑dependent). For robust wake word support, ship a hybrid build (Capacitor) enabling a native KWS engine.
- Telemetry (opt‑in):
  - Track wake word precision/recall, average time‑to‑first‑token. Provide an opt‑in diagnostics mode to improve models without storing raw audio.
- Failure handling:
  - If mic is busy or KWS fails, show non‑intrusive banner and fallback to push‑to‑talk; keep a local “voice memo” mode for later transcription.

### B. Craftsmen personal data (Personal profile)

1. Name
2. Email
3. Language

### C. Business data (Business profile)

1. Currency (a default value for Jobs)
2. VAT rate (a default value for Jobs)
3. Business description
4. Road Distance Unit (km, miles)

### D. Software application data

1. Subscription type
    - Trial
    - Basic
    - Premium
2. Subscription expiration date
3. Subscription status (active, expired, canceled)
4. Subscription constraints (max jobs, max team members, max vehicles, max machines)
5. Subscription price (monthly, yearly)


### **E. Resources Management**

1. **Team members**. Mandatory. The craftsman himself is a team member.
2. **Vehicles** (optional)
3. **Machines** (optional)

Resources have typicaly these properties:
* Name
* Hourly rate (machines, team members)
* Kilometer rate (vehicles)

### **F. Jobs Management**

- Application maintains a list of jobs. 
- Each job has:
    - JobNumber (Autoincremental, in the Firestore database for the specific user)
    - Title 
    - Description 
    - JobStatus (active, archived)
    - CreatedAt 
    - UpdatedAt 
    - UpdatedBy (user name)
    - VatRate in percent
    - Currency = "EUR"; // ISO 4217 code (validated in application layer)
    - Budget = 0; // Bid budget entered manually by user
    - List of Costs (Transport, Material, Labor, Machine, Other)
    - List of Advances provided by the client
    - List of updating events (Starting work; Starting journey; Finishing work; Finishing journey; Material added; etc.)

### **G. Advances Management**

  - The Advances collection is specific for each job. 
  - It is a list of payments made by the client to the craftsman. 
  - Application automatically assignes an ordinal number to each advance. The numbering is based on the Firestore autoincremental feature.
TADY  - The application screen displays the sum of advances and the sum of costs. The difference is the amount the craftsman is owed.


### **G. Cost Management**

#### There are two methods how to manage costs:

##### A. CRUD operations (e.g. Creating, Updating, Deleting costs). 
 Application will manage a list of costs for each job on teh separate tab/screen. This is the "classic" way how to manage costs. However it should be also possible to add/update/delete costs by voice. A list of costs can be filtered by the cost type (transport, material, labor, machine, other) and updated on the specific page.

###### B. Event-based (e.g. Starting work, Stopping work, Adding material, etc.)

1. **Labor**
* **Recording:** Simple voice commands for start/stop (`"Starting work"`, `"Taking a break"`, `"Finishing for today"`) or by time (`"I worked for 2 hours."`).
* **Rate setting:** User sets his/her hourly rate and the application automatically calculates the price for the work performed.
* **Team** A user can store their team members and set their hourly rates in the database.
* **Team work recording:** The user can say: `"John is working on the Smith order."` and the application will ask: `"How many hours is he working?"`
* **Team members working on the same order:** The application will store worked hours for each team member to the same or different orders. Application automatically calculate the total cost of the work performed by the team members on the order according to their hourly rates.

2. **Material**
* **Recording:** By voice (`"I bought 5 plasterboard sheets for 2500 crowns."`).
* **Smart receipt scanning:** The user could say: `"Write down this receipt."`, take a picture of it and the AI ​​would use OCR (Optical Character Recognition) technology to "read" the individual items, prices and dates and suggest their inclusion in the order.

3. **Transport**
* **Recording:** By voice at the beginning and end of the trip.
* **Automation:** The application can use GPS to automatically track the distance traveled. The user would just say: `"I'm going to Mr. Smith's"` and the application would measure the route itself and upon arrival ask: `"Have you arrived at the location? Should I write down the 15 km drive to the Smith order?"`.
* **Vehicles:** The user saves their vehicles in the database(e.g. "Transporter", "Fabia") and sets the rate per kilometer for each. The AI ​​then automatically calculates the cost of the trip.

4. **Machines ⚙ ** (optional)
* **Recording:** Simple voice commands for start/stop (`"Starting work with excavator"`, `"Stopping excavator"`, `"Excavator ends for today"`).
* **Rate setting:** User sets the hourly rate of the machine in the database. The application calculates the total cost of the work performed by the machine.

5. **Other costs** (optional)
* **Recording:** By voice (`"I paid 500 crowns for parking."`).


---

## Technological design

To implement such an application, a combination of several technologies will be needed:

* **Subscription model:** The application will be available for free for a trial period. After that, a subscription will be required. The subscription will be billed monthly or yearly. The subscription will include a certain number of jobs, team members, vehicles and machines. If the user wants to add more, they will have to pay extra.
* **Payment processing:** The application will use a payment processing service like Stripe to process payments.

* **Mobile applications:** An Angular PWA (Progressive Web App) can be used to create a mobile application.
* **Database:** **Firebase/Firestore** for storing data about users, jobs, team members, machines, cost items, etc.
* **AI and Voice Services (the most important):**
* **Speech-to-Text:** Device or browser native capabilities where available; Services like **Google Cloud Speech-to-Text** or **Whisper by OpenAI**, which have excellent support for Czech, German, Polish and other languages. (It is possible to use multiple services for different languages.)
* **Natural Language Processing (NLU):** Services like **Google Dialogflow** or **Rasa** for recognizing intentions and entities from text.
* **Text-to-Speech:** Services like **Google Cloud Text-to-Speech** for generating natural-sounding Czech (and other languages) voice responses from the assistant.
* **Image recognition (OCR):** **Google Cloud Vision AI** for the receipt scanning feature.

---

## Data structure

### Firebase Firestore data structure:

We would need a collection of customers and at least the following basic tables in the database for each customer:
* **Vehicles:** List of the user's vehicles with their names and rate per km, last known odometer reading.
* **Team:** List of the user's team members with their names and hourly rates.
* **Machines:** List of the user's machines with their names and hourly rates.
* **Jobs:** Job name, client name, address, status (active, completed, invoiced), notes, images.
* **Costs:** Subcollection of all cost items for the job:
* Type (transport, material, labor, machine)
* Date and time
* Description (e.g. "Journey to Brno", "Plasterboard")
* Amount/units (e.g. 55 km, 2500 CZK, 8 hours)
* Price (e.g. 55 km * 0.50 CZK/km = 27.50 CZK)
* Comment
* Picture


Customer document shape (top-level under customers/{tenantId}):
```json
{
  "type": "customer",
  "profile": {
    "name": "Acme Corp",
    "email": "ops@acme.com",
    "address": "123 Main St",
    "phone": "+1 555 1234",
    "notes": "VIP"
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

Note: Personal information fields are grouped under the `profile` object (name, email, address, phone, notes). Backend API requests and responses also use this nested `profile` structure for customers.


customers/ (collection)
├──  customer_ID_001 (Document)
│   └── Vehicles (subcollection)
│   └── Team (subcollection)
│   └── Machines (subcollection)
│   └── Jobs (subcollection)
│    ├──  job_ID_001 (Document)
│    │    ├── name: "Smith, Brno"
│    │    ├── client: "Smith"
│    │    ├── address: "Brno"
│    │    ├── status: "active"
│    │    ├── notes: "Painting and plastering"
│    │    ├── images: ["image1.jpg", "image2.jpg"]
│    │    └── costs (subcollection)
│    ├──  job_ID_002 (Document)
│    │    ├── name: "John Dow, Prague"
│    │    ├── client: "Smith"
│    │    ├── address: "Brno"
│    │    ├── status: "active"
│    │    ├── notes: "Painting and plastering"
│    │    ├── images: ["image1.jpg", "image2.jpg"]
│    │    └── costs (subcollection)
│    └── ... next jobs
├──  customer_ID_002 (Document)
│   └── ...
└── ... next customers

---

## Implementation

This project contains three main parts:
- Backend
- User UI
- Administrator UI

- Both UIs are built with Angular and Ionic https://ionicframework.com/
- User UI supports multiple languages. Specifically:
    - Czech
    - English
    - German
    - Polish
    - Slovak

- Czech is a default UI language.
- Programming is done in English.
- The currency is defined by the user (Crowns, Euros, Dollars, etc.).


### Backend
- Backend is built using C# and .NET 8.
- It follows the latest monitoring and logging standards and best practices.
- Backend is responsible for:
    - Authentication and authorization
    - Data storage and retrieval
    - Business logic
    - Communication with external services
- Backend communicates with external services. Specifically with:
    - External services like Google Cloud Text-to-Speech or similar
    - Firestore
    - AI providers like OpenAI, Groq, OpenRouter, Cohere, etc.
    - Other services as needed

### User UI

- User UI is primarily for mobile.
- User UI communicates with backend using REST API.
- User UI is built using Angular and Ionic.
- The User UI must support complete control and navigation through voice commands.
- In addition User UI must support also text input for users who prefer to use text input.
- User UI must support also disconnected operations.

#### Offline mode and sync strategy (emphasized)

- Critical actions available offline:
  - Start/stop work, set active job, add material (name, qty, price), add journey (odometer start/end), add notes, capture photos/voice memos.
- Local storage & caching:
  - Use IndexedDB (e.g., localForage/ngx-indexed-db) for an Outbox queue + local replicas of Jobs and recent Costs.
  - Service Worker (Workbox) caches shell + API responses for read-through when offline. Media (photos) stored in Cache Storage with metadata in IndexedDB.
- Sync engine:
  - Deterministic IDs (UUID v4) with tenant and device prefix to prevent collisions and enable idempotent server upserts.
  - Background sync when connectivity resumes; exponential backoff, per-item retry limits, and clear per-item status (Queued, Syncing, Synced, Error).
  - Conflict resolution: last-writer-wins by updatedAt for simple fields; per-field merge for additive lists; mark "Needs review" if concurrent changes detected.
- Voice in offline:
  - PWA baseline: record audio as voice memo and transcribe when online; provide a small on-device keyword grammar for core intents ("start work", "stop work", "add material").
  - Hybrid build (Capacitor): leverage Android offline recognizer / iOS on-device dictation when available for richer offline STT.
- UX behavior:
  - Visible "Offline" banner; core buttons remain enabled; show queue length and last sync time; allow manual "Sync now".
  - Totals in Job detail computed from local + synced items; warn if totals may be incomplete while offline.
- Security offline:
  - Minimize PII in local cache; encrypt at rest via WebCrypto; if packaged, store key material in platform secure storage; wipe local cache on logout.
- Testing & observability:
  - E2E tests in airplane mode; synthetic conflict scenarios; metrics: median time-to-sync, queue failure rate, storage size guardrails.


Given the target group, the UI must be **extremely simple and clear**.

* **Main screen:** Displays the currently active job and the sum of costs entered.
* **Voice assistant activation:** The main screen contains a large button to activate the voice assistant.
* **Costs inputs:** A set of large buttons for the costs inputs (general chat, start/stop work, start/stop journey, add material, add machine work, add comment, add picture).
* **Job detail:** Sum of costs for the job. After clicking on it, the user would see a clear summary of all costs - divided into labor, materials and transportation. Each item should be easy to manually edit or delete.
* **Recent activities:** A list of recent activities (costs) with a short summary.
* **Export:** A key function at the end. The ability to export the complete list of costs for the job as a simple PDF file, which will serve as **a basis for invoicing**.
* **Jobs list:** A simple list of running and completed jobs.

#### Description of basic operations

- Example 1:
1. Large main button indicating application readiness. When clicked, the application is activated. A user can start talking.
2. The application listens to the user's voice input.
3. A user can say: "I'm going to Brno to handle an order for Mr. Smith. Odometer reading is 123456.". The applcation understands the intent and asks for confirmation.
4. The application asks: "Is the odometer reading correct?"
5. The user answers "Yes" or "No".
6. If the user answers "Yes", the application sumarises and repeats the entered data and asks for confirmation. If the user answers "Yes", the application confirms the data and saves the data. If the user answers "No", the application asks for the correct odometer reading.
7. A user can also cancel the operation at any time by saying "Cancel" or "Stop" or "Back" or "Start again" or similar.
8. If the job does not exist, the application asks: "Should I create a new job for Mr. Smith in Brno?"
9. The user answers "Yes" or "No".
10. If the user answers "Yes", the application creates a new job. If the user answers "No", the application asks for the name of the existing job.

- Example 2:
1. Large main button indicating application readiness. When clicked, the application is activated. A user can start talking.
2. A user says: "Set active job to Smith, Brno."
3. The application asks: "Is the job Smith, Brno correct?"
4. The user answers "Yes" or "No".
5. If the user answers "Yes", the application confirms the data and sets the active job and saves it to the database. All following costs will be automatically assigned to the data.If the user answers "No", the application asks for the correct job name.
6. When an active job is set, the application can enter costs for the active job. It will read labels of the buttons ("Add material", "Add machine work", "Add comment", "Add picture".) For example, if the user says "Add material", the application will ask for the name of the material and the price.
7. The user says: "I bought screws in Hornbach for five hundred crowns." The application understands the intent, repeats the summary of the entered data and asks for confirmation.
8. The user answers "Yes" or "Save" or "No" or "Cancel" or "Don't save" or a similar answer.
9. Depending on the user's answer, the application either saves the data or answers "Cancelled" and returns to point 1.
10. Similarly for other costs.


#### Other parts of the UI

1. Display the list of jobs.
2. Implementation of all CRUD operations, which must be controllable by voice and touch.
This means that the "Save", "Change", "Delete" buttons must respond to touch and voice instructions.

The same for other entities: team members, machines, etc.

---

### Administrator UI
- Administrator UI is primarily for desktop.
- Administrator UI is used for configuration and management of the system.
- Administrator UI is used for:
    - Configuration of the system
    - Management of the system
    - Monitoring of the system
    - Reporting
    - Backup and restore
    - User management
    - Role management
    - Permission management
    - Audit log
    - System log
    - etc.
- Administrator UI is built using Angular and Ionic.
- Administrator can perform all operations that are available in the User UI for any customer.
- It communicates with backend using REST API.
- UI is built using Angular and Ionic.
- The Administrator UI is for configuration and management of the system. No voice commands are needed for the Administrator UI.

## Security and multi-tenant architecture

Robust tenant isolation is a hard requirement. Every request, record, and resource must be scoped to a single tenant (customer) and enforced across client, backend, database, and storage layers.

- Tenancy model
  - Each customer has a unique tenantId. All entities (Jobs, Costs, Vehicles, Team members, Machines) carry tenantId and are stored under customer-specific paths or with an indexed tenantId field.
  - Recommended Firestore shape: customers/{tenantId}/(Jobs, Vehicles, TeamMembers, Machines, ...).

- Identity, authN/Z, and token design
  - Use short-lived access tokens (JWT) issued by the backend/identity provider; include claims: sub, tenant_id, roles, exp, iat.
  - Roles per tenant: owner, admin, worker. A separate superadmin role exists only in the Admin UI service for cross-tenant support.

- Backend enforcement (never trust the client)
  - On every request, derive tenantId exclusively from the authenticated token. Ignore any tenantId present in payload.
  - Apply tenantId filters to all queries and writes. Validate resource ownership pre- and post-DB call.

- Firestore Security Rules (defense-in-depth)
  - Example rule enforcing tenant scoping in customer subtrees:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /customers/{tenantId}/{document=**} {
      allow read, write: if request.auth != null
        && request.auth.token.tenant_id == tenantId;
    }
  }
}
```

- Storage isolation (images, receipts, audio)
  - Partition cloud storage by tenant prefix (e.g., /tenants/{tenantId}/media/...).
  - Generate short-lived, signed URLs on the backend; do not expose bucket keys to clients.

- Encryption and secrets
  - Encrypt data in transit (TLS) and at rest (provider default). Consider field-level encryption for sensitive fields (client names, addresses) with per-tenant keys managed via KMS.
  - Manage secrets only in backend (no API keys in the client). Rotate keys regularly.

- Authorization policy (least privilege)
  - Worker: create/update own job entries and costs within tenant; read-only access to selected entities.
  - Admin: full CRUD within tenant; manage members and rates.
  - Owner: billing/export, tenant config, data export/delete requests.

- Auditing and monitoring
  - Immutable audit log per tenant (who, what, when, from where). Correlate with traceId/requestId. Ship logs to centralized monitoring (SIEM) and alert on anomalies.

- Rate limiting and DoS protection
  - Per-tenant and per-user rate limits on critical endpoints (login, cost creation, media upload). Web Application Firewall (WAF) in front of the APIs.

- Backups, data lifecycle, and compliance
  - Regular encrypted backups with per-tenant restore capability. Data retention schedules. GDPR/DSGVO support: export and deletion upon verified request; data residency configuration.

- Example server-side scoping (illustrative)

```
// C# pseudo-code
var tenantId = userToken.TenantId;
return db.Jobs.Where(o => o.TenantId == tenantId);
```

- Testing strategy
  - Unit tests for auth middlewares and repository filters (tenant must match).
  - Integration tests that attempt cross-tenant access (must be denied) and verify Security Rules.
  - Penetration testing for multi-tenant breakout and IDOR vulnerabilities.

