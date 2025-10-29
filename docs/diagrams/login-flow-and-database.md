# User Login Flow and Database Structure

## 1. Complete Login Flow

```mermaid
sequenceDiagram
    participant User
    participant App
    participant FirebaseAuth
    participant CloudFunction
    participant Firestore

    Note over User,Firestore: NEW USER REGISTRATION FLOW
    User->>App: Enter Email & Password
    App->>FirebaseAuth: createUserWithEmailAndPassword()
    FirebaseAuth->>FirebaseAuth: Create Firebase UID
    FirebaseAuth-->>App: Return UID + Auth Token

    FirebaseAuth->>CloudFunction: Trigger: getOrCreateTenantForUser(uid)
    CloudFunction->>Firestore: Create /tenants/{tenantId}
    CloudFunction->>Firestore: Create /tenants/{tenantId}/members/{uid}<br/>(role: owner, memberNumber: 1)
    CloudFunction->>Firestore: Create /user_tenants/{uid}/memberships/{tenantId}
    CloudFunction->>Firestore: Create /tenants/{tenantId}/teamMembers/{memberId}
    CloudFunction->>CloudFunction: allocateSequence('memberNumber')

    CloudFunction->>CloudFunction: setActiveTenantClaim(uid, tenantId)
    CloudFunction-->>App: Custom Claim Set (tenant_id)

    App->>FirebaseAuth: Force refresh ID token
    FirebaseAuth-->>App: New token with custom claim
    App->>App: Parse tenant_id from token
    App-->>User: Redirect to Home Screen (logged in)

    Note over User,Firestore: EXISTING USER LOGIN FLOW
    User->>App: Enter Email & Password
    App->>FirebaseAuth: signInWithEmailAndPassword()
    FirebaseAuth->>FirebaseAuth: Validate credentials
    FirebaseAuth-->>App: Return UID + Token (with tenant_id claim)

    App->>App: Extract tenant_id from custom claim
    App->>Firestore: Read /tenants/{tenantId}/members/{uid}
    Firestore-->>App: Return user role & permissions

    App->>Firestore: Query /tenants/{tenantId}/jobs (with role filter)
    Firestore-->>App: Return accessible data
    App-->>User: Navigate to Home Screen
```

## 2. Database Structure After Login

```mermaid
erDiagram
    FIREBASE_USER ||--o{ USER_TENANTS : "has memberships"
    USER_TENANTS ||--|| TENANT : "references"
    TENANT ||--|{ MEMBERS : "contains"
    TENANT ||--|{ JOBS : "contains"
    TENANT ||--|{ VEHICLES : "contains"
    TENANT ||--|{ MACHINES : "contains"
    TENANT ||--|{ TEAM_MEMBERS : "contains"
    TENANT ||--|{ AUDIT_LOGS : "contains"
    TENANT ||--o| BUSINESS_PROFILE : "has"
    TENANT ||--o| PERSON_PROFILE : "has"
    TENANT ||--o{ INVITES : "contains"

    JOBS ||--o{ COSTS : "has"
    JOBS ||--o{ ADVANCES : "has"
    JOBS ||--o{ EVENTS : "has"

    COSTS }o--|| VEHICLES : "references (snapshot)"
    COSTS }o--|| MACHINES : "references (snapshot)"
    COSTS }o--|| TEAM_MEMBERS : "references (snapshot)"

    EVENTS }o--|| VEHICLES : "references (snapshot)"

    FIREBASE_USER {
        string uid PK "Firebase Auth UID"
        string email
        string displayName
        timestamp createdAt
    }

    USER_TENANTS {
        string uid PK "Firebase UID"
        string tenantId PK "References tenant"
        timestamp joinedAt
    }

    TENANT {
        string tenantId PK "UUID"
        int schemaVersion "Schema version number"
        timestamp createdAt
        timestamp updatedAt
    }

    MEMBERS {
        string uid PK "Firebase UID"
        string tenantId FK
        int memberNumber "Sequential (1,2,3...)"
        string displayName "Cached"
        string email "Cached"
        string role "owner|representative|teamMember"
        string status "active|disabled"
        timestamp lastSeenAt
        timestamp createdAt
        timestamp updatedAt
    }

    JOBS {
        string jobId PK "Auto-generated"
        string tenantId FK
        int jobNumber "Sequential, nullable"
        string title
        string description
        string status "active|completed|archived"
        string currency "ISO 4217"
        number vatRate
        number budget
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    COSTS {
        string costId PK "Auto-generated"
        string tenantId FK
        int ordinalNumber "Sequential per job"
        string category "transport|material|labor|machine|other"
        number amount
        string description
        timestamp date
        object resource "Snapshot data"
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    ADVANCES {
        string advanceId PK "Auto-generated"
        string tenantId FK
        int ordinalNumber "Sequential per job"
        number amount
        timestamp date
        string note
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    EVENTS {
        string eventId PK "Auto-generated"
        string tenantId FK
        int ordinalNumber "Sequential per job"
        string type "journey_start|journey_end|manual_entry"
        timestamp timestamp
        object data "Event-specific data"
        object createdBy "UserIdentity"
        timestamp createdAt
    }

    VEHICLES {
        string vehicleId PK "Auto-generated"
        string tenantId FK
        int vehicleNumber "Sequential, nullable"
        string name
        string distanceUnit "km|miles"
        number ratePerDistanceUnit
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    MACHINES {
        string machineId PK "Auto-generated"
        string tenantId FK
        int machineNumber "Sequential, nullable"
        string name
        number hourlyRate
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    TEAM_MEMBERS {
        string teamMemberId PK "Auto-generated"
        string tenantId FK
        int teamMemberNumber "Sequential, nullable"
        string name
        number hourlyRate
        string authUserId "Firebase UID or null"
        object createdBy "UserIdentity"
        timestamp createdAt
        object updatedBy "UserIdentity"
        timestamp updatedAt
    }

    BUSINESS_PROFILE {
        string tenantId PK
        string currency "ISO 4217"
        number vatRate
        string distanceUnit "km|miles"
        timestamp createdAt
        timestamp updatedAt
    }

    PERSON_PROFILE {
        string tenantId PK
        string displayName
        string email
        string language "cs|en"
        string preferredVoiceProvider
        boolean aiSupportEnabled
        timestamp createdAt
        timestamp updatedAt
    }

    AUDIT_LOGS {
        string logId PK "Auto-generated"
        string operation "CREATE|UPDATE|DELETE"
        string collection
        string documentId
        string tenantId FK
        timestamp timestamp
        object author "UserIdentity"
        object before "Previous state"
        object after "New state"
        timestamp ttl "Auto-delete marker"
    }

    INVITES {
        string inviteId PK "Auto-generated"
        string tenantId FK
        string codeHash "SHA-256 of 6-digit code"
        timestamp expiresAt "7 days from creation"
        object createdBy "UserIdentity"
        string presetRole "representative|teamMember"
        timestamp consumedAt
        string email "Optional"
        timestamp createdAt
        timestamp updatedAt
    }
```

## 3. Data Stored After Successful Login

### Global Collections
```
/user_tenants/{uid}/
  └── memberships/{tenantId}/
      ├── tenantId: string
      ├── joinedAt: Timestamp
```

### Tenant-Scoped Collections
```
/tenants/{tenantId}/
  ├── schemaVersion: number
  ├── createdAt: Timestamp
  ├── updatedAt: Timestamp
  │
  ├── members/{uid}/
  │   ├── tenantId: string
  │   ├── memberNumber: number (1, 2, 3...)
  │   ├── displayName: string
  │   ├── email: string
  │   ├── role: "owner" | "representative" | "teamMember"
  │   ├── status: "active" | "disabled"
  │   ├── lastSeenAt: Timestamp
  │   ├── createdAt: Timestamp
  │   └── updatedAt: Timestamp
  │
  ├── jobs/{jobId}/
  │   ├── tenantId: string
  │   ├── jobNumber: number | null
  │   ├── title: string
  │   ├── description: string
  │   ├── status: "active" | "completed" | "archived"
  │   ├── currency: string (ISO 4217)
  │   ├── vatRate: number
  │   ├── budget: number
  │   ├── createdBy: UserIdentity {uid, memberNumber, displayName}
  │   ├── createdAt: Timestamp
  │   ├── updatedBy: UserIdentity
  │   ├── updatedAt: Timestamp
  │   │
  │   ├── costs/{costId}/
  │   │   ├── tenantId: string
  │   │   ├── ordinalNumber: number | null
  │   │   ├── category: "transport" | "material" | "labor" | "machine" | "other"
  │   │   ├── amount: number
  │   │   ├── description: string
  │   │   ├── date: Timestamp
  │   │   ├── resource: object (snapshot)
  │   │   ├── createdBy: UserIdentity
  │   │   ├── createdAt: Timestamp
  │   │   ├── updatedBy: UserIdentity
  │   │   └── updatedAt: Timestamp
  │   │
  │   ├── advances/{advanceId}/
  │   │   ├── tenantId: string
  │   │   ├── ordinalNumber: number | null
  │   │   ├── amount: number
  │   │   ├── date: Timestamp
  │   │   ├── note: string
  │   │   ├── createdBy: UserIdentity
  │   │   ├── createdAt: Timestamp
  │   │   ├── updatedBy: UserIdentity
  │   │   └── updatedAt: Timestamp
  │   │
  │   └── events/{eventId}/
  │       ├── tenantId: string
  │       ├── ordinalNumber: number | null
  │       ├── type: "journey_start" | "journey_end" | "manual_entry"
  │       ├── timestamp: Timestamp
  │       ├── data: object (event-specific)
  │       ├── createdBy: UserIdentity
  │       └── createdAt: Timestamp
  │
  ├── vehicles/{vehicleId}/
  │   ├── tenantId: string
  │   ├── vehicleNumber: number | null
  │   ├── name: string
  │   ├── distanceUnit: "km" | "miles"
  │   ├── ratePerDistanceUnit: number
  │   ├── createdBy: UserIdentity
  │   ├── createdAt: Timestamp
  │   ├── updatedBy: UserIdentity
  │   └── updatedAt: Timestamp
  │
  ├── machines/{machineId}/
  │   ├── (Similar structure to vehicles)
  │
  ├── teamMembers/{teamMemberId}/
  │   ├── tenantId: string
  │   ├── teamMemberNumber: number | null
  │   ├── name: string
  │   ├── hourlyRate: number
  │   ├── authUserId: string | null (Firebase UID)
  │   ├── createdBy: UserIdentity
  │   ├── createdAt: Timestamp
  │   ├── updatedBy: UserIdentity
  │   └── updatedAt: Timestamp
  │
  ├── audit_logs/{logId}/
  │   ├── operation: "CREATE" | "UPDATE" | "DELETE"
  │   ├── collection: string
  │   ├── documentId: string
  │   ├── tenantId: string
  │   ├── timestamp: Timestamp
  │   ├── author: UserIdentity
  │   ├── before: object (for UPDATE)
  │   ├── after: object (for CREATE/UPDATE)
  │   └── ttl: Timestamp (1 year retention)
  │
  ├── businessProfile/ (single document)
  │   ├── tenantId: string
  │   ├── currency: string (ISO 4217)
  │   ├── vatRate: number
  │   ├── distanceUnit: "km" | "miles"
  │   ├── createdAt: Timestamp
  │   └── updatedAt: Timestamp
  │
  ├── personProfile/ (single document)
  │   ├── tenantId: string
  │   ├── displayName: string
  │   ├── email: string
  │   ├── language: "cs" | "en"
  │   ├── preferredVoiceProvider: string
  │   ├── aiSupportEnabled: boolean
  │   ├── createdAt: Timestamp
  │   └── updatedAt: Timestamp
  │
  └── invites/{inviteId}/
      ├── tenantId: string
      ├── codeHash: string (SHA-256)
      ├── expiresAt: Timestamp
      ├── createdBy: UserIdentity
      ├── presetRole: "representative" | "teamMember"
      ├── consumedAt: Timestamp
      ├── email: string (optional)
      ├── createdAt: Timestamp
      └── updatedAt: Timestamp
```

## 4. Security & Access Control

### Role-Based Permissions

| Collection | Owner | Representative | Team Member |
|------------|-------|----------------|-------------|
| members | R/W | R | R (own only) |
| jobs | R/W | R/W | R (public view) |
| costs | R/W | R/W | R/W |
| advances | R/W | R/W | R |
| events | R/W | R/W | R |
| vehicles | R/W | R/W | R |
| machines | R/W | R/W | R |
| teamMembers | R/W | R/W | R |
| audit_logs | R | ❌ | ❌ |
| businessProfile | R/W | R | ❌ |
| personProfile | R/W | R/W | R/W |
| invites | R/W | ❌ | ❌ |

### Authentication Token Structure
```
Firebase ID Token:
{
  "iss": "https://securetoken.google.com/{project-id}",
  "aud": "{project-id}",
  "auth_time": 1234567890,
  "user_id": "{firebase-uid}",
  "sub": "{firebase-uid}",
  "iat": 1234567890,
  "exp": 1234571490,
  "email": "user@example.com",
  "email_verified": true,
  "firebase": {
    "identities": {
      "email": ["user@example.com"]
    },
    "sign_in_provider": "password"
  },
  "tenant_id": "{tenantId}"  ← Custom Claim Set After Login
}
```

## 5. Key Implementation Details

### UserIdentity Object (used throughout database)
```typescript
interface UserIdentity {
  uid: string;              // Firebase Auth UID (for security)
  memberNumber: number;     // Tenant-specific ID (for display)
  displayName: string;      // Cached display name
}
```

### Sequential Number Allocation
- **Online**: Cloud Function `allocateSequence(tenantId, counterType)` atomically increments
- **Offline**: Documents created with `null`, server assigns on sync via onCreate trigger
- **Counter Types**: jobNumber, vehicleNumber, machineNumber, teamMemberNumber, memberNumber, job_{jobId}_ordinalNumber

### Multi-Tenant Isolation
- All data scoped under `/tenants/{tenantId}/`
- Custom claim `tenant_id` in JWT for efficient access control
- Firestore Security Rules enforce tenant boundaries
- All documents include `tenantId` field for validation

### Offline Support
- IndexedDB persistence enabled on web clients
- Automatic sync when connection restored
- Nullable sequential numbers for offline-created documents
- Client-side pending write queue management
