[Back to Index](./index.md) | [Previous: Coding Standards](./coding-standards.md) | [Next: Voice Architecture](./voice-architecture.md)

# Security Architecture

This document consolidates all security-related aspects of the FinDogAI system, including authentication, authorization, data isolation, and security best practices.

## Multi-Tenant Isolation

### Tenant Data Isolation

**Core Principle:** Complete data isolation at the database level through Firestore path structure and security rules.

```
/tenants/{tenantId}/
├── members/{uid}        # Tenant-specific user roles
├── jobs/{jobId}         # Tenant jobs
├── invites/{inviteId}   # Tenant invitations
└── resources/           # Tenant resources
    ├── vehicles/
    ├── teamMembers/
    └── machines/
```

### Security Rules for Tenant Isolation

```javascript
// Firestore Security Rules - Tenant Isolation
match /tenants/{tenantId}/{document=**} {
  // Helper function to check tenant membership
  function isTenantMember() {
    return request.auth != null &&
      exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));
  }

  // Helper function to get user's role
  function getUserRole() {
    return get(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role;
  }

  // All reads require tenant membership
  allow read: if isTenantMember();

  // Creates require owner or representative role
  allow create: if isTenantMember() &&
    getUserRole() in ['owner', 'representative'];

  // Updates require membership and cannot change tenantId
  allow update: if isTenantMember() &&
    request.resource.data.tenantId == resource.data.tenantId;

  // Soft deletes (setting deletedAt) allowed for owners/representatives
  allow update: if isTenantMember() &&
    getUserRole() in ['owner', 'representative'] &&
    request.resource.data.keys().hasAll(['deletedAt']);
}
```

## Authentication Architecture

### Firebase Authentication Integration

**Provider Support:**
- Email/Password (primary)
- Google OAuth
- Microsoft OAuth (for enterprise)
- Magic Link (passwordless)

**Token Management:**
```typescript
interface AuthToken {
  uid: string;              // Firebase Auth UID
  email: string;
  email_verified: boolean;
  // Custom claims set by Cloud Functions
  customClaims?: {
    tenantIds: string[];    // Array of tenant memberships
    primaryTenantId: string; // Default tenant
  };
}
```

### Offline Authentication

**Token Caching Strategy:**
```typescript
class OfflineAuthService {
  private readonly TOKEN_CACHE_KEY = 'findogai_auth_token';
  private readonly REFRESH_THRESHOLD = 5 * 60 * 1000; // 5 minutes

  async cacheToken(token: string): Promise<void> {
    const encrypted = await this.encrypt(token);
    await SecureStorage.set({
      key: this.TOKEN_CACHE_KEY,
      value: encrypted
    });
  }

  async validateOfflineToken(): Promise<boolean> {
    const cached = await SecureStorage.get({ key: this.TOKEN_CACHE_KEY });
    if (!cached) return false;

    const token = await this.decrypt(cached.value);
    const decoded = jwt.decode(token);

    // Check expiration with threshold
    const now = Date.now();
    return decoded.exp * 1000 > now + this.REFRESH_THRESHOLD;
  }
}
```

## Authorization Model

### Role-Based Access Control (RBAC)

**Role Hierarchy:**
```typescript
enum MemberRole {
  OWNER = 'owner',              // Full access, billing, delete tenant
  REPRESENTATIVE = 'representative', // Manage jobs, costs, resources
  TEAM_MEMBER = 'teamMember'    // View and add costs only
}

// Permission matrix
const PERMISSIONS = {
  [MemberRole.OWNER]: [
    'tenant:delete',
    'tenant:billing',
    'members:manage',
    'jobs:*',
    'costs:*',
    'resources:*'
  ],
  [MemberRole.REPRESENTATIVE]: [
    'jobs:*',
    'costs:*',
    'resources:*',
    'members:invite'
  ],
  [MemberRole.TEAM_MEMBER]: [
    'jobs:read',
    'costs:create',
    'costs:read',
    'resources:read'
  ]
};
```

### Audit Trail Security

**Immutable Audit Logging:**
```typescript
// Cloud Function - Audit Logger (server-side only)
export const createAuditLog = functions.firestore
  .onWrite(async (change, context) => {
    const auditEntry: AuditLog = {
      timestamp: admin.firestore.FieldValue.serverTimestamp(),
      action: !change.before.exists ? 'CREATE' :
               !change.after.exists ? 'DELETE' : 'UPDATE',
      entityType: context.resource.name,
      entityId: context.params.entityId,
      tenantId: context.params.tenantId,
      userId: change.after?.data()?.updatedBy?.uid ||
               change.before?.data()?.updatedBy?.uid,
      changes: change.before.exists && change.after.exists ?
               getDiff(change.before.data(), change.after.data()) : null,
      ipAddress: context.rawRequest?.ip, // If available
      userAgent: context.rawRequest?.headers?.['user-agent']
    };

    // Write to append-only audit collection
    await admin.firestore()
      .collection('tenants')
      .doc(context.params.tenantId)
      .collection('audit_logs')
      .add(auditEntry);
  });
```

## Data Security

### Encryption at Rest
- **Firestore:** Automatic AES256 encryption by Google
- **Cloud Storage:** AES256 encryption for all files
- **Local Storage:** Capacitor Secure Storage for sensitive data

### Encryption in Transit
- **HTTPS Only:** Enforced via Firebase Hosting
- **WebSocket Security:** Firestore real-time connections use WSS
- **Certificate Pinning:** For native apps (optional)

### Sensitive Data Handling

**PII Protection:**
```typescript
// Never log sensitive data
const sanitizeForLogging = (data: any): any => {
  const sensitive = ['email', 'phone', 'ssn', 'password', 'token'];
  const sanitized = { ...data };

  sensitive.forEach(key => {
    if (sanitized[key]) {
      sanitized[key] = '[REDACTED]';
    }
  });

  return sanitized;
};
```

**Secure File Storage:**
```typescript
// Cloud Storage security rules
service firebase.storage {
  match /b/{bucket}/o {
    // Tenant-scoped file access
    match /tenants/{tenantId}/{allPaths=**} {
      allow read: if request.auth != null &&
        firestore.exists(/databases/(default)/documents/tenants/$(tenantId)/members/$(request.auth.uid));

      allow write: if request.auth != null &&
        firestore.get(/databases/(default)/documents/tenants/$(tenantId)/members/$(request.auth.uid)).data.role in ['owner', 'representative'];

      // Prevent executable uploads
      allow write: if request.resource.contentType.matches('image/.*') ||
                     request.resource.contentType == 'application/pdf';
    }
  }
}
```

## API Security

### Cloud Functions Security

**HTTPS Callable Functions:**
```typescript
// Automatic auth context validation
export const secureFunction = functions.https.onCall(async (data, context) => {
  // Check authentication
  if (!context.auth) {
    throw new functions.https.HttpsError(
      'unauthenticated',
      'User must be authenticated'
    );
  }

  // Validate tenant membership
  const member = await admin.firestore()
    .doc(`tenants/${data.tenantId}/members/${context.auth.uid}`)
    .get();

  if (!member.exists) {
    throw new functions.https.HttpsError(
      'permission-denied',
      'User is not a member of this tenant'
    );
  }

  // Check role-based permissions
  const role = member.data().role;
  if (!hasPermission(role, data.requiredPermission)) {
    throw new functions.https.HttpsError(
      'permission-denied',
      `Role ${role} lacks required permission`
    );
  }

  // Execute function logic
  return await performSecureOperation(data);
});
```

### Rate Limiting

**API Rate Limits:**
```typescript
class RateLimiter {
  private limits = {
    'voice:process': { requests: 100, window: 3600000 }, // 100/hour
    'pdf:generate': { requests: 10, window: 3600000 },   // 10/hour
    'invite:create': { requests: 20, window: 86400000 }  // 20/day
  };

  async checkLimit(userId: string, operation: string): Promise<void> {
    const key = `rate_limit:${userId}:${operation}`;
    const limit = this.limits[operation];

    const count = await redis.incr(key);
    if (count === 1) {
      await redis.expire(key, limit.window / 1000);
    }

    if (count > limit.requests) {
      throw new functions.https.HttpsError(
        'resource-exhausted',
        `Rate limit exceeded for ${operation}`
      );
    }
  }
}
```

## Voice Pipeline Security

### LLM API Security

**API Key Management:**
```typescript
// Never expose API keys to client
// Cloud Functions environment variables only
const LLM_CONFIG = {
  openai: {
    apiKey: process.env.OPENAI_API_KEY,
    organization: process.env.OPENAI_ORG_ID
  },
  anthropic: {
    apiKey: process.env.ANTHROPIC_API_KEY
  }
};

// Input sanitization for LLM
const sanitizeLLMInput = (input: string): string => {
  // Remove potential prompt injection attempts
  const cleaned = input
    .replace(/ignore previous instructions/gi, '')
    .replace(/system:/gi, '')
    .replace(/\[INST\]/gi, '');

  // Limit input length
  return cleaned.substring(0, 1000);
};
```

### Voice Command Validation

```typescript
interface VoiceCommandValidator {
  validateIntent(intent: Intent): boolean {
    // Whitelist allowed actions
    const allowedActions = [
      'navigate', 'create_job', 'add_cost',
      'set_active', 'query_status'
    ];

    return allowedActions.includes(intent.action);
  }

  validateEntityAccess(entityId: string, userId: string): Promise<boolean> {
    // Verify user has access to referenced entity
    return this.checkTenantMembership(entityId, userId);
  }
}
```

## Security Headers

### Firebase Hosting Headers

```json
{
  "hosting": {
    "headers": [
      {
        "source": "**",
        "headers": [
          {
            "key": "Content-Security-Policy",
            "value": "default-src 'self'; script-src 'self' 'unsafe-inline' https://apis.google.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https://*.googleapis.com https://*.cloudfunctions.net wss://*.firebaseio.com"
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
          },
          {
            "key": "Referrer-Policy",
            "value": "strict-origin-when-cross-origin"
          },
          {
            "key": "Permissions-Policy",
            "value": "geolocation=(), microphone=(self), camera=()"
          }
        ]
      }
    ]
  }
}
```

## Compliance and Privacy

### GDPR Compliance

**Data Privacy Measures:**
- Data stored in EU region (europe-west1)
- User consent for data processing
- Right to erasure (soft delete with purge option)
- Data portability (export functions)
- Privacy by design principles

**Data Retention Policy:**
```typescript
// Scheduled function for GDPR compliance
export const purgeOldData = functions.pubsub
  .schedule('every 24 hours')
  .onRun(async () => {
    const cutoffDate = new Date();
    cutoffDate.setFullYear(cutoffDate.getFullYear() - 1); // 1 year retention

    // Purge soft-deleted records older than retention period
    const batch = admin.firestore().batch();

    const oldRecords = await admin.firestore()
      .collectionGroup('audit_logs')
      .where('timestamp', '<', cutoffDate)
      .limit(500)
      .get();

    oldRecords.forEach(doc => {
      batch.delete(doc.ref);
    });

    await batch.commit();
  });
```

## Security Best Practices

### Development Security

1. **Environment Variables:**
   - Never commit `.env` files
   - Use Secret Manager for production
   - Rotate API keys regularly

2. **Dependency Security:**
   - Regular `npm audit` checks
   - Automated dependency updates via Dependabot
   - Lock file integrity checks

3. **Code Security:**
   - Input validation on all user inputs
   - Output encoding to prevent XSS
   - Parameterized queries (though Firestore handles this)
   - Regular security code reviews

### Operational Security

1. **Monitoring:**
   - Failed authentication attempts
   - Unusual data access patterns
   - API rate limit violations
   - Error rate spikes

2. **Incident Response:**
   - Security incident playbook
   - Data breach notification process
   - Regular security drills
   - Audit log preservation

---

*Next: [Voice Architecture](./voice-architecture.md)*