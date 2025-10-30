# Member Invitation Mechanism - Database & Flow Diagrams

## 1. Complete Invitation Flow Sequence Diagram

```mermaid
sequenceDiagram
    participant Owner as Owner/Rep
    participant App
    participant CF as Cloud Function
    participant DB as Firestore
    participant NewMember as New Team Member
    participant Auth as Firebase Auth

    Note over Owner,Auth: INVITE CREATION PHASE

    Owner->>App: Click "Invite Member"
    App->>Owner: Show invite form
    Owner->>App: Submit form<br/>(role, optional email)

    App->>CF: createInvite({tenantId, role, email?})
    CF->>CF: Validate: caller is Owner/Rep
    CF->>CF: Generate 6-digit code: "123456"
    CF->>CF: Hash: SHA256("123456") → "abc123..."
    CF->>CF: Set expiresAt = now + 7 days

    CF->>DB: Create "/tenants/{tid}/invites/{id}"
    Note right of DB: Store:<br/>- codeHash: "abc123..."<br/>- expiresAt: Timestamp<br/>- presetRole: role<br/>- createdBy: UserIdentity<br/>- email: optional<br/>- consumedAt: null

    DB-->>CF: Document created
    CF-->>App: Return code: "123456"
    App-->>Owner: Display code (copy/QR)

    Owner->>Owner: Share code with team member<br/>(email, SMS, etc.)

    Note over Owner,Auth: INVITE REDEMPTION PHASE

    NewMember->>App: Sign up / Log in
    App->>Auth: Authentication
    Auth-->>App: Return UID + token

    NewMember->>App: Enter "Join Team" screen
    App->>NewMember: Request invite code
    NewMember->>App: Paste/enter code: "123456"

    App->>CF: redeemInvite({code: "123456"})

    CF->>CF: Hash provided code:<br/>SHA256("123456") → "abc123..."

    CF->>DB: Query invites:<br/>WHERE codeHash="abc123..."<br/>AND consumedAt=null
    DB-->>CF: Return matching invite

    CF->>CF: Validate: expiresAt > now
    CF->>CF: Validate: consumedAt == null
    CF->>CF: Validate: email match (if set)
    CF->>CF: Extract: tenantId, presetRole

    Note over CF,DB: ATOMIC TRANSACTION START

    CF->>DB: allocateSequence(tenantId, 'memberNumber')
    DB-->>CF: Return memberNumber (e.g., 3)

    CF->>DB: allocateSequence(tenantId, 'teamMemberNumber')
    DB-->>CF: Return teamMemberNumber (e.g., 5)

    CF->>DB: CREATE /tenants/{tid}/members/{uid}
    Note right of DB: Store:<br/>- memberNumber: 3<br/>- role: presetRole<br/>- status: "active"<br/>- displayName, email<br/>- createdAt, updatedAt

    CF->>DB: CREATE /user_tenants/{uid}/memberships/{tid}
    Note right of DB: Store:<br/>- tenantId<br/>- role: presetRole<br/>- createdAt, updatedAt

    CF->>DB: CREATE /tenants/{tid}/teamMembers/{id}
    Note right of DB: Store:<br/>- teamMemberNumber: 5<br/>- name: displayName<br/>- hourlyRate: 0<br/>- authUserId: uid<br/>- createdBy: inviter identity<br/>- createdAt, updatedAt

    CF->>DB: UPDATE "/tenants/{tid}/invites/{id}"
    Note right of DB: Set:<br/>- consumedAt: now

    Note over CF,DB: ATOMIC TRANSACTION COMMIT

    CF->>CF: setActiveTenantClaim(uid, tenantId)
    Note right of CF: Inject custom claim<br/>"tenant_id" into JWT

    CF-->>App: Success response

    App->>Auth: Force refresh ID token
    Auth-->>App: New token with tenant_id claim

    App->>App: Switch to joined tenant
    App->>DB: Query tenant data<br/>(jobs, resources, etc.)
    DB-->>App: Return accessible data

    App-->>NewMember: Redirect to home screen<br/>(joined successfully)
```

## 2. Database Schema for Invitation Mechanism

```mermaid
erDiagram
    INVITES ||--o| MEMBERS : "creates on redemption"
    INVITES ||--o| USER_TENANTS : "creates on redemption"
    INVITES ||--o| TEAM_MEMBERS : "creates on redemption"
    INVITES }o--|| TENANT : "belongs to"
    INVITES }o--|| SEQUENCES : "allocates memberNumber"
    INVITES }o--|| SEQUENCES : "allocates teamMemberNumber"
    MEMBERS }o--|| TENANT : "belongs to"
    TEAM_MEMBERS }o--|| TENANT : "belongs to"
    USER_TENANTS }o--|| FIREBASE_USER : "maps"

    INVITES {
        string inviteId PK "Auto-generated UUID"
        string tenantId FK "Owner tenant"
        string codeHash "SHA-256 of 6-digit code"
        timestamp expiresAt "7 days from creation"
        object createdBy "UserIdentity {uid, memberNumber, displayName}"
        string presetRole "representative | teamMember"
        timestamp consumedAt "NULL until redeemed"
        string email "Optional pre-assignment"
        timestamp createdAt "Server timestamp"
        timestamp updatedAt "Server timestamp"
    }

    MEMBERS {
        string uid PK "Firebase Auth UID"
        string tenantId FK "Joined tenant"
        int memberNumber "Sequential (1,2,3...)"
        string displayName "Cached from profile"
        string email "Cached from profile"
        string role "owner | representative | teamMember"
        string status "active | disabled"
        timestamp lastSeenAt "Last activity"
        timestamp createdAt "Server timestamp"
        timestamp updatedAt "Server timestamp"
    }

    USER_TENANTS {
        string uid PK "Firebase Auth UID"
        string tenantId PK "Tenant membership"
        string role "Cached role"
        timestamp createdAt "Server timestamp"
        timestamp updatedAt "Server timestamp"
    }

    TEAM_MEMBERS {
        string teamMemberId PK "Auto-generated UUID"
        string tenantId FK "Owner tenant"
        int teamMemberNumber "Sequential (1,2,3...)"
        string name "Display name"
        number hourlyRate "Labor cost rate"
        string authUserId "Firebase UID or null"
        object createdBy "UserIdentity"
        timestamp createdAt "Server timestamp"
        object updatedBy "UserIdentity"
        timestamp updatedAt "Server timestamp"
    }

    SEQUENCES {
        string tenantId PK "Tenant owner"
        string counterType PK "memberNumber | teamMemberNumber | etc"
        int currentValue "Last allocated number"
        timestamp updatedAt "Last increment"
    }

    TENANT {
        string tenantId PK "UUID"
        int schemaVersion "Schema version"
        timestamp createdAt "Creation time"
        timestamp updatedAt "Last update"
    }

    FIREBASE_USER {
        string uid PK "Firebase Auth UID"
        string email "User email"
        string displayName "User name"
        timestamp createdAt "Registration time"
    }
```

## 3. Invite States and Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending : Owner creates invite<br/>Cloud Function generates code

    Pending --> Consumed : Team member redeems<br/>with valid code
    Pending --> Expired : 7 days pass<br/>expiresAt < now
    Pending --> Revoked : Owner deletes invite

    Consumed --> [*] : Member joined<br/>consumedAt set
    Expired --> [*] : Cleanup process<br/>or remains archived
    Revoked --> [*] : Deleted from DB

    note right of Pending
        Database State:
        - codeHash: stored
        - expiresAt: future
        - consumedAt: null
        - Status: Active
    end note

    note right of Consumed
        Database State:
        - consumedAt: timestamp
        - Member created
        - TeamMember created
        - UserTenant mapping created
    end note

    note right of Expired
        Database State:
        - expiresAt: past
        - consumedAt: null
        - Cannot be redeemed
    end note
```

## 4. Database Collections Modified During Invitation Flow

```mermaid
flowchart TD
    Start([Invite Created]) --> InviteDoc["/tenants/{tid}/invites/{id}"]

    InviteDoc --> |Stored| InviteData["codeHash: SHA-256<br/>expiresAt: +7 days<br/>presetRole: role<br/>createdBy: UserIdentity<br/>email: optional<br/>consumedAt: null"]

    InviteData --> Share[Owner shares code<br/>with team member]

    Share --> Redeem{Team member<br/>redeems code}

    Redeem --> |Invalid/Expired| Error[Error: Invalid invite]
    Redeem --> |Valid| Transaction[Atomic Transaction]

    Transaction --> Seq1["Allocate memberNumber<br/>from /sequences/{tid}/counters"]
    Transaction --> Seq2["Allocate teamMemberNumber<br/>from /sequences/{tid}/counters"]

    Seq1 --> Member["CREATE<br/>/tenants/{tid}/members/{uid}"]
    Seq2 --> TeamMember["CREATE<br/>/tenants/{tid}/teamMembers/{id}"]
    Transaction --> Mapping["CREATE<br/>/user_tenants/{uid}/memberships/{tid}"]
    Transaction --> Update["UPDATE<br/>/tenants/{tid}/invites/{id}"]

    Member --> MemberData["memberNumber: allocated<br/>role: presetRole<br/>status: active<br/>displayName, email"]

    TeamMember --> TMData["teamMemberNumber: allocated<br/>name: displayName<br/>hourlyRate: 0<br/>authUserId: uid"]

    Mapping --> MapData["tenantId: tid<br/>role: presetRole"]

    Update --> InviteUpdate["consumedAt: NOW<br/>(marks as used)"]

    MemberData --> Claim[Set Custom Claim<br/>tenant_id in JWT]
    TMData --> Claim
    MapData --> Claim
    InviteUpdate --> Claim

    Claim --> Complete([Team Member Joined])

    style InviteDoc fill:#e1f5ff
    style Member fill:#c8e6c9
    style TeamMember fill:#c8e6c9
    style Mapping fill:#c8e6c9
    style Update fill:#fff9c4
    style Transaction fill:#f3e5f5
```

## 5. Invite Code Security Flow

```mermaid
flowchart LR
    A[Generate Code] --> B["6-digit random<br/>e.g., 123456"]
    B --> C["SHA-256 Hash<br/>abc123...def456"]
    C --> D["Store ONLY hash<br/>in Firestore"]
    D --> E[Return code to Owner<br/>ONE TIME display]

    E --> F[Owner shares code<br/>via email/SMS]
    F --> G[Team member enters code]

    G --> H["Compute SHA-256<br/>of entered code"]
    H --> I["Query DB:<br/>WHERE codeHash = hash"]

    I --> J{Match found?}
    J --> |No| K[Error: Invalid code]
    J --> |Yes| L{Expired?}

    L --> |Yes| M[Error: Expired]
    L --> |No| N{Already used?}

    N --> |Yes| O[Error: Already redeemed]
    N --> |No| P{Email match?}

    P --> |If set, must match| Q[Validate email]
    P --> |Not set| R[Proceed to redemption]
    Q --> |Match| R
    Q --> |No match| S[Error: Wrong email]

    R --> T[Create memberships<br/>Mark invite consumed]

    style D fill:#ffcdd2
    style E fill:#fff9c4
    style H fill:#c8e6c9
    style T fill:#c8e6c9
```

## 6. Data Structure: Invites Collection Detail

### Collection Path

```text
/tenants/{tenantId}/invites/{inviteId}
```

### Document Schema

```yaml
inviteId: "auto-generated-uuid"
tenantId: "owner-tenant-uuid"

# Security - Code never stored in plaintext
codeHash: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"  # SHA-256

# Expiration
expiresAt: Timestamp  # now + 7 days (604,800 seconds)
createdAt: Timestamp  # Server timestamp
updatedAt: Timestamp  # Server timestamp

# Creator Information (Compound Identity)
createdBy:
  uid: "firebase-auth-uid"
  memberNumber: 1  # Tenant-specific ID
  displayName: "John Doe"

# Invite Configuration
presetRole: "representative"  # or "teamMember"

# Optional Pre-assignment
email: "newmember@example.com"  # Optional - binds invite to specific email

# Redemption Tracking
consumedAt: null  # or Timestamp when redeemed (for single-use enforcement)
```

### Indexes

```yaml
# Single field indexes
- field: expiresAt
  order: ASC
  purpose: Cleanup queries for expired invites

- field: consumedAt
  order: ASC
  purpose: Filter pending vs consumed invites

# Composite indexes
- fields:
    - codeHash: ASC
    - consumedAt: ASC
  purpose: Fast lookup during redemption validation
```

## 7. Database State Changes Timeline

```mermaid
gantt
    title Invitation Mechanism - Database Changes Timeline
    dateFormat X
    axisFormat %s

    section Invite Creation
    Owner clicks "Invite"                    :0, 1
    Cloud Function generates code            :1, 2
    Hash code (SHA-256)                      :2, 1
    Create /invites/{id} document            :3, 2
    Return code to UI                        :5, 1

    section Sharing Phase
    Owner shares code                        :6, 10
    Team member receives code                :16, 5

    section Redemption
    Team member enters code                  :21, 2
    redeemInvite() validates                 :23, 3
    Hash comparison                          :26, 1
    Expiration check                         :27, 1

    section Atomic Transaction
    Allocate memberNumber                    :28, 2
    Allocate teamMemberNumber                :30, 2
    CREATE /members/{uid}                    :32, 2
    CREATE /user_tenants/{uid}/memberships   :34, 2
    CREATE /teamMembers/{id}                 :36, 2
    UPDATE /invites/{id} (consumedAt)        :38, 1
    Transaction commit                       :39, 1

    section Post-Redemption
    Set custom claim (tenant_id)             :40, 2
    Refresh ID token                         :42, 2
    User switches to tenant                  :44, 2
    Load tenant data                         :46, 3
```

## 8. Security Rules for Invites Collection

```javascript
// Firestore Security Rules
match /tenants/{tenantId}/invites/{inviteId} {

  // READ: All active members can see pending/consumed invites
  allow read: if isActiveMember(tenantId);

  // CREATE: Only owner or representative can create invites
  allow create: if isOwnerOrRepresentative(tenantId)
    && documentHasTenantId(tenantId)
    && auditMetadataValid()
    && canWriteToTenant(tenantId);

  // UPDATE: Immutable after creation (only Cloud Function can mark consumed)
  allow update: if false;

  // DELETE: Only owner can revoke (delete) pending invites
  allow delete: if isOwner(tenantId)
    && canWriteToTenant(tenantId);
}

// Helper functions
function isActiveMember(tenantId) {
  return exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid))
    && get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.status == 'active';
}

function isOwnerOrRepresentative(tenantId) {
  let memberDoc = get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));
  return memberDoc.data.status == 'active'
    && (memberDoc.data.role == 'owner' || memberDoc.data.role == 'representative');
}

function isOwner(tenantId) {
  let memberDoc = get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));
  return memberDoc.data.status == 'active' && memberDoc.data.role == 'owner';
}
```

## 9. Cloud Functions for Invitation Mechanism

### Function 1: createInvite

```typescript
// HTTPS Callable Function
exports.createInvite = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    // Validate authentication
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
    }

    const { tenantId, presetRole, email } = data;
    const uid = context.auth.uid;

    // Validate caller is owner or representative
    const memberDoc = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (!memberDoc.exists || !['owner', 'representative'].includes(memberDoc.data().role)) {
      throw new functions.https.HttpsError('permission-denied', 'Insufficient permissions');
    }

    // Generate 6-digit code
    const code = Math.floor(100000 + Math.random() * 900000).toString();

    // Hash the code
    const codeHash = crypto.createHash('sha256').update(code).digest('hex');

    // Create invite document
    const inviteRef = admin.firestore()
      .collection(`tenants/${tenantId}/invites`)
      .doc();

    const inviteData = {
      tenantId,
      codeHash,
      expiresAt: admin.firestore.Timestamp.fromDate(
        new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
      ),
      createdBy: {
        uid,
        memberNumber: memberDoc.data().memberNumber,
        displayName: memberDoc.data().displayName
      },
      presetRole,
      email: email || null,
      consumedAt: null,
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      updatedAt: admin.firestore.FieldValue.serverTimestamp()
    };

    await inviteRef.set(inviteData);

    // Return code (one-time display)
    return {
      inviteId: inviteRef.id,
      code,  // Original code returned ONCE
      expiresAt: inviteData.expiresAt.toDate().toISOString()
    };
  });
```

### Function 2: redeemInvite

```typescript
// HTTPS Callable Function
exports.redeemInvite = functions
  .region('europe-west1')
  .https.onCall(async (data, context) => {
    // Validate authentication
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'User must be authenticated');
    }

    const { code } = data;
    const uid = context.auth.uid;
    const userEmail = context.auth.token.email;
    const displayName = context.auth.token.name || userEmail;

    // Hash provided code
    const codeHash = crypto.createHash('sha256').update(code).digest('hex');

    // Query for matching invite
    const invitesQuery = await admin.firestore()
      .collectionGroup('invites')
      .where('codeHash', '==', codeHash)
      .where('consumedAt', '==', null)
      .limit(1)
      .get();

    if (invitesQuery.empty) {
      throw new functions.https.HttpsError('not-found', 'Invalid or expired invitation');
    }

    const inviteDoc = invitesQuery.docs[0];
    const inviteData = inviteDoc.data();
    const tenantId = inviteData.tenantId;

    // Validate expiration
    if (inviteData.expiresAt.toDate() < new Date()) {
      throw new functions.https.HttpsError('failed-precondition', 'Invitation has expired');
    }

    // Validate email binding (if set)
    if (inviteData.email && inviteData.email !== userEmail) {
      throw new functions.https.HttpsError('permission-denied', 'Invitation is for a different email');
    }

    // Check if user is already a member
    const existingMember = await admin.firestore()
      .doc(`tenants/${tenantId}/members/${uid}`)
      .get();

    if (existingMember.exists) {
      throw new functions.https.HttpsError('already-exists', 'You are already a member of this team');
    }

    // Atomic transaction to create memberships
    await admin.firestore().runTransaction(async (transaction) => {
      // Allocate memberNumber
      const memberNumber = await allocateSequence(transaction, tenantId, 'memberNumber');

      // Allocate teamMemberNumber
      const teamMemberNumber = await allocateSequence(transaction, tenantId, 'teamMemberNumber');

      // Create member document
      const memberRef = admin.firestore().doc(`tenants/${tenantId}/members/${uid}`);
      transaction.set(memberRef, {
        tenantId,
        memberNumber,
        displayName,
        email: userEmail,
        role: inviteData.presetRole,
        status: 'active',
        lastSeenAt: null,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      // Create user-tenant mapping
      const mappingRef = admin.firestore().doc(`user_tenants/${uid}/memberships/${tenantId}`);
      transaction.set(mappingRef, {
        tenantId,
        role: inviteData.presetRole,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      // Create team member resource
      const teamMemberRef = admin.firestore()
        .collection(`tenants/${tenantId}/teamMembers`)
        .doc();
      transaction.set(teamMemberRef, {
        tenantId,
        teamMemberNumber,
        name: displayName,
        hourlyRate: 0,
        authUserId: uid,
        createdBy: inviteData.createdBy,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedBy: inviteData.createdBy,
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      // Mark invite as consumed
      transaction.update(inviteDoc.ref, {
        consumedAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
      });
    });

    // Set custom claim for tenant
    await admin.auth().setCustomUserClaims(uid, {
      tenant_id: tenantId
    });

    return {
      success: true,
      tenantId,
      role: inviteData.presetRole
    };
  });
```

## 10. Summary: Key Database Collections in Invitation Flow

| Collection | Action | When | Data Stored |
|------------|--------|------|-------------|
| `"/tenants/{tid}/invites/{id}"` | **CREATE** | Owner creates invite | codeHash, expiresAt, presetRole, createdBy, email? |
| `"/tenants/{tid}/invites/{id}"` | **UPDATE** | Member redeems invite | consumedAt = now |
| `/sequences/{tid}/counters/memberNumber` | **INCREMENT** | During redemption | currentValue++ |
| `/sequences/{tid}/counters/teamMemberNumber` | **INCREMENT** | During redemption | currentValue++ |
| `/tenants/{tid}/members/{uid}` | **CREATE** | Member redeems invite | memberNumber, role, status, displayName, email |
| `/user_tenants/{uid}/memberships/{tid}` | **CREATE** | Member redeems invite | tenantId, role |
| `/tenants/{tid}/teamMembers/{id}` | **CREATE** | Member redeems invite | teamMemberNumber, name, hourlyRate, authUserId |
| `/tenants/{tid}/audit_logs/{id}` | **CREATE** | After each operation | operation, collection, documentId, author, before/after |

### Total Database Operations Per Redemption: 8 writes

1. Read invite document (validate)
2. Increment memberNumber counter
3. Increment teamMemberNumber counter
4. Create member document
5. Create user-tenant mapping
6. Create team member resource
7. Update invite (set consumedAt)
8. Audit log entries (automatic triggers)
