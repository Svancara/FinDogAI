[Back to Index](./index.md)

# Security Considerations

This document covers authentication flows, authorization patterns, data encryption, and API security for the FinDogAI backend.

## Overview

Security is implemented through multiple layers:
- **Authentication**: Firebase Authentication with custom claims
- **Authorization**: Firestore Security Rules with RBAC
- **Encryption**: At-rest and in-transit encryption
- **API Security**: HTTPS-only, rate limiting, input validation

---

## 8.1 Authentication Flow

### 8.1.1 Email/Password Authentication (MVP)

**Registration Flow:**

```
1. User submits email/password
2. Firebase Auth creates account → returns UID
3. Cloud Function triggered:
   a. Creates tenant: /tenants/{newTenantId}
   b. Creates member: /tenants/{tenantId}/members/{uid} (role: owner)
   c. Creates mapping: /user_tenants/{uid}/memberships/{tenantId}
   d. Sets custom claim: tenant_id = tenantId
4. User receives email verification (optional)
5. App initializes with tenant context
```

**Login Flow:**

```
1. User submits email/password
2. Firebase Auth validates credentials → returns UID + token
3. App reads custom claim: tenant_id from token
4. App queries /tenants/{tenantId}/members/{uid} for role
5. App initializes with tenant context and user role
```

**Client Implementation:**

```typescript
import { Auth, signInWithEmailAndPassword, createUserWithEmailAndPassword } from '@angular/fire/auth';

async login(email: string, password: string) {
  const userCredential = await signInWithEmailAndPassword(this.auth, email, password);
  const idTokenResult = await userCredential.user.getIdTokenResult();
  const tenantId = idTokenResult.claims['tenant_id'];

  // Load user role from Firestore
  const memberDoc = await getDoc(doc(this.firestore, `tenants/${tenantId}/members/${userCredential.user.uid}`));
  const role = memberDoc.data()?.role;

  return { uid: userCredential.user.uid, tenantId, role };
}
```

### 8.1.2 OAuth Providers (Phase 2)

**Supported Providers:**
- Google Sign-In
- Apple Sign-In

**Implementation:**

```typescript
import { signInWithPopup, GoogleAuthProvider } from '@angular/fire/auth';

async signInWithGoogle() {
  const provider = new GoogleAuthProvider();
  const userCredential = await signInWithPopup(this.auth, provider);
  // Same tenant resolution flow as email/password
}
```

---

## 8.2 Authorization Patterns

### 8.2.1 Role-Based Access Control (RBAC)

**Roles:**
- **Owner**: Full access including audit logs, member management, and data export
- **Representative**: Cannot modify business profile or access audit logs
- **Team Member**: Read-only access to jobs_public; can create costs

**Enforcement:**

Security Rules check role from `/tenants/{tenantId}/members/{uid}` document:

```javascript
function getMemberRole(tenantId) {
  return get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role;
}

function isOwner(tenantId) {
  return isActiveMember(tenantId) && getMemberRole(tenantId) == 'owner';
}
```

**Application-Level Checks:**

```typescript
// Route guards
@Injectable()
export class OwnerGuard implements CanActivate {
  canActivate(route: ActivatedRouteSnapshot): boolean {
    return this.authService.userRole === 'owner';
  }
}

// Component-level
<button *ngIf="userRole === 'owner'" (click)="exportData()">
  Export Data
</button>
```

### 8.2.2 Custom Claims for Tenant Context

**Setting Custom Claims (Cloud Function):**

```typescript
import { auth } from 'firebase-admin';

export const setTenantClaim = onDocumentCreated(
  'user_tenants/{uid}/memberships/{tenantId}',
  async (event) => {
    const { uid, tenantId } = event.params;

    await auth().setCustomUserClaims(uid, {
      tenant_id: tenantId
    });
  }
);
```

**Reading Custom Claims (Client):**

```typescript
const idTokenResult = await this.auth.currentUser.getIdTokenResult();
const tenantId = idTokenResult.claims['tenant_id'];
```

**Force Token Refresh:**

```typescript
// After tenant switching
await this.auth.currentUser.getIdToken(true); // Force refresh
```

---

## 8.3 Data Encryption

### 8.3.1 Encryption at Rest

**Firebase Automatic Encryption:**
- All Firestore data encrypted at rest using AES-256
- Encryption keys managed by Google Cloud KMS
- EU region hosting (europe-west1) for GDPR compliance

**No additional configuration required** - enabled by default.

### 8.3.2 Encryption in Transit

**HTTPS Enforcement:**
- All Firebase SDK connections use HTTPS/TLS 1.3
- WebSocket connections for real-time listeners use WSS
- Cloud Functions require HTTPS (HTTP redirected)

**Firebase Hosting Security Headers:**

```json
// firebase.json
{
  "hosting": {
    "headers": [
      {
        "source": "**",
        "headers": [
          {
            "key": "Strict-Transport-Security",
            "value": "max-age=31536000; includeSubDomains"
          },
          {
            "key": "X-Content-Type-Options",
            "value": "nosniff"
          },
          {
            "key": "X-Frame-Options",
            "value": "DENY"
          },
          {
            "key": "X-XSS-Protection",
            "value": "1; mode=block"
          }
        ]
      }
    ]
  }
}
```

---

## 8.4 API Security

### 8.4.1 Cloud Functions Authentication

**Automatic Token Validation:**

```typescript
import { onCall, HttpsError } from 'firebase-functions/v2/https';

export const allocateSequence = onCall(async (request) => {
  // request.auth is automatically validated by Firebase
  if (!request.auth) {
    throw new HttpsError('unauthenticated', 'User must be authenticated');
  }

  const uid = request.auth.uid;
  const tenantId = request.data.tenantId;

  // Verify tenant membership
  const memberDoc = await admin.firestore()
    .doc(`tenants/${tenantId}/members/${uid}`)
    .get();

  if (!memberDoc.exists || memberDoc.data()?.status !== 'active') {
    throw new HttpsError('permission-denied', 'Not an active member');
  }

  // ... function logic
});
```

**Input Validation with Zod:**

```typescript
import { z } from 'zod';

const AllocateSequenceSchema = z.object({
  tenantId: z.string().uuid(),
  type: z.enum(['jobNumber', 'vehicleNumber', 'machineNumber', 'teamMemberNumber']),
  jobId: z.string().uuid().optional()
});

export const allocateSequence = onCall(async (request) => {
  try {
    const validatedData = AllocateSequenceSchema.parse(request.data);
    // ... use validatedData
  } catch (error) {
    throw new HttpsError('invalid-argument', error.message);
  }
});
```

### 8.4.2 Rate Limiting (Phase 2)

**Firebase App Check:**
- Prevents abuse from bots and unauthorized clients
- Requires valid app attestation token

**Implementation:**

```typescript
import { getAppCheck, initializeAppCheck, ReCaptchaV3Provider } from '@angular/fire/app-check';

// In app.config.ts
provideAppCheck(() => {
  const appCheck = initializeAppCheck(undefined, {
    provider: new ReCaptchaV3Provider('your-recaptcha-site-key'),
    isTokenAutoRefreshEnabled: true
  });
  return appCheck;
});
```

**Cloud Functions Enforcement:**

```typescript
export const allocateSequence = onCall(
  { enforceAppCheck: true }, // Requires valid App Check token
  async (request) => {
    // ... function logic
  }
);
```

---

## Security Best Practices

### 1. Never Expose API Keys in Client Code

**Bad:**
```typescript
const openai = new OpenAI({
  apiKey: 'sk-proj-...' // NEVER do this!
});
```

**Good:**
```typescript
// Use Cloud Functions as proxy
const callable = httpsCallable(functions, 'processVoiceCommand');
const result = await callable({ text: transcription });
```

### 2. Validate All User Input

```typescript
// In Cloud Functions
const schema = z.object({
  amount: z.number().positive().max(1000000),
  category: z.enum(['transport', 'material', 'labor', 'machine', 'other'])
});

const validatedData = schema.parse(request.data);
```

### 3. Use Principle of Least Privilege

- Grant minimum required permissions
- Team members cannot access audit logs
- Representatives cannot modify business profile

### 4. Audit All Sensitive Operations

```typescript
// Log all audit log access
export const onAuditLogRead = onDocumentRead(
  'tenants/{tenantId}/audit_logs/{logId}',
  async (event) => {
    console.log('Audit log accessed:', {
      logId: event.params.logId,
      tenantId: event.params.tenantId,
      userId: event.auth?.uid,
      timestamp: new Date().toISOString()
    });
  }
);
```

### 5. Implement GDPR Data Export/Deletion

See [Cloud Functions - exportData](./cloud-functions.md) for GDPR-compliant data export implementation.

---

For complete security rules implementation, see [Security Rules](./security-rules.md).

---

[← Back: Development & Deployment](./development-deployment.md) | [Back to Index](./index.md) | [Next: Performance Optimization →](./performance-optimization.md)
