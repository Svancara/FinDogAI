[Back to Index](./index.md)

# Firebase Services

This document provides detailed information about the Firebase services used in FinDogAI and their configuration.

## Overview

FinDogAI leverages four core Firebase services, all configured in the europe-west1 region for GDPR compliance:

1. **Firebase Authentication** - User identity management
2. **Cloud Firestore** - Primary database
3. **Cloud Functions (2nd Gen)** - Server-side business logic
4. **Cloud Storage for Firebase** - File storage

## Firebase Authentication

### Purpose
User identity management and tenant context injection.

### Configuration

- **Region**: Global (identity tokens)
- **Providers**:
  - Email/password authentication (MVP)
  - OAuth providers (Phase 2: Google, Apple)

### Features Used

#### Email/Password Authentication
- Standard Firebase email/password authentication
- Password requirements enforced by Firebase (minimum 6 characters)
- Email verification available but not required for MVP
- Password reset via email

#### Custom Claims for Multi-Tenancy
Custom claim `tenant_id` is added to user tokens after tenant membership resolution:

```typescript
// Example custom claim structure
{
  "user_id": "firebase-uid-12345",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "email_verified": true
}
```

**Important**: User roles are NOT stored in custom claims. They are enforced by Security Rules checking the `/tenants/{tenantId}/members/{uid}` document.

### Authentication Flow

See [High Level Architecture - User Authentication Flow](./high-level-architecture.md#231-user-authentication-flow) for detailed authentication sequence.

## Cloud Firestore

### Purpose
Primary database for all tenant data with offline-first capabilities.

### Configuration

- **Region**: europe-west1 (Belgium) - GDPR compliant
- **Mode**: Native mode (not Datastore mode)
- **Billing**: Pay-as-you-go (Blaze plan required for Cloud Functions)

### Features Used

#### Offline Persistence
- Automatic offline persistence enabled on mobile clients
- IndexedDB storage for web
- Automatic sync when connection restored
- Pending writes queued locally

#### Real-Time Listeners
- WebSocket connections for real-time updates
- Automatic reconnection handling
- Snapshot listeners for live UI updates

#### Hierarchical Data with Subcollections
```
/tenants/{tenantId}/jobs/{jobId}/
  ├─ costs/{costId}      - Job costs
  ├─ advances/{advanceId} - Customer advances
  └─ events/{eventId}     - Journey events
```

#### Composite Indexes
Required indexes for complex queries (defined in `firestore.indexes.json`):
- Status + UpdatedAt for job listings
- CreatedBy + CreatedAt for user activity
- Category + Date for cost filtering

See [Data Model - Required Indexes](./data-model.md#43-required-indexes) for complete index configuration.

#### Security Rules
Firestore Security Rules enforce:
- Authentication requirements
- Multi-tenant data isolation
- Role-based access control
- Audit metadata validation

See [Security Rules](./security-rules.md) for complete rules.

### Data Model

All business data is scoped under `/tenants/{tenantId}/` path prefix for multi-tenant isolation:

```
/tenants/{tenantId}/
  ├─ members/          - User roles and permissions
  ├─ jobs/             - Job records with subcollections
  ├─ vehicles/         - Vehicle resources
  ├─ machines/         - Machine resources
  ├─ teamMembers/      - Team member resources
  ├─ audit_logs/       - Compliance audit trail
  ├─ businessProfile   - Tenant settings (single doc)
  └─ personProfile     - User preferences (single doc)
```

See [Data Model](./data-model.md) for complete schema definitions.

## Cloud Functions (2nd Gen)

### Purpose
Server-side business logic for operations that cannot run client-side.

### Configuration

- **Region**: europe-west1 (co-located with Firestore)
- **Runtime**: Node.js 20
- **Generation**: 2nd Gen (Cloud Run-based)
- **Memory**: 256MB (default)
- **Timeout**: 60s (default)
- **Concurrency**: 80 requests per instance

### Function Categories

#### Firestore Triggers
- **onCreate/onUpdate/onDelete**: Audit logging for all tenant data changes
- **onCreate**: Sequential number assignment for offline-created entities

#### HTTPS Callable Functions
- `allocateSequence` - Transactional ID allocation for online operations
- `exportData` - GDPR-compliant data export
- `generatePDF` - PDF report generation (Phase 2)
- `processVoiceCommand` - LLM integration for voice NLU (Phase 2)

#### Scheduled Functions
- `cleanupAuditLogs` - Daily cleanup of audit logs older than 1 year (runs at 2 AM)

### Deployment Configuration

```json
// firebase.json
{
  "functions": [
    {
      "source": "packages/functions",
      "codebase": "default",
      "runtime": "nodejs20",
      "region": "europe-west1",
      "concurrency": 80,
      "memory": "256MB",
      "timeout": "60s"
    }
  ]
}
```

See [Cloud Functions](./cloud-functions.md) for implementation details.

## Cloud Storage for Firebase

### Purpose
Secure file storage for data exports and generated reports.

### Configuration

- **Region**: europe-west1
- **Access Control**: Security Rules + Signed URLs
- **Lifecycle Policies**: Automatic cleanup

### Features Used

#### Signed URLs for Secure Download
- Time-limited URLs (1 hour expiration)
- No direct public access
- Generated by Cloud Functions

#### Lifecycle Policies
Automatic cleanup of temporary files:
- **Data exports**: 7-day Time-To-Live (TTL)
- **PDF reports**: 30-day TTL (Phase 2)

#### Storage Structure

```
/exports/{tenantId}/
  └─ {timestamp}_data_export.json

/reports/{tenantId}/   (Phase 2)
  └─ {timestamp}_job_{jobNumber}.pdf
```

#### Security Rules

```javascript
// storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Data exports (owner-only)
    match /exports/{tenantId}/{fileName} {
      allow read: if isAuthenticated() && isTenantOwner(tenantId);
      allow write: if false; // Only Cloud Functions can write
    }

    // PDF reports (Phase 2)
    match /reports/{tenantId}/{fileName} {
      allow read: if isAuthenticated() && isTenantOwner(tenantId);
      allow write: if false; // Only Cloud Functions can write
    }
  }
}
```

See [Security Rules - Storage Security Rules](./security-rules.md#332-storage-security-rules) for complete rules.

## External Services Integration

### Google Cloud Speech-to-Text (STT)
- **Purpose**: Voice transcription
- **Language**: Czech (cs-CZ)
- **Integration**: Direct API calls from mobile app
- **Fallback**: OpenAI Whisper API (Phase 2)

### Google Cloud Text-to-Speech (TTS)
- **Purpose**: Voice confirmation playback
- **Language**: Czech (cs-CZ)
- **Voices**: WaveNet voices for natural speech
- **Integration**: Direct API calls from mobile app

### OpenAI API
- **GPT-4**: Natural language understanding for voice commands
- **Whisper**: Alternative STT (Phase 2)
- **Integration**: Backend proxy via Cloud Functions to protect API keys

### Picovoice
- **Purpose**: On-device keyword spotting (KWS)
- **Integration**: Local SDK in mobile app
- **Wake word**: Custom wake word for hands-free activation

## Firebase Project Structure

### Environments

#### Development
- **Project ID**: `findogai-dev`
- **Purpose**: Development and testing
- **Data**: Test data, can be reset
- **Functions**: Test versions of Cloud Functions

#### Production
- **Project ID**: `findogai-prod`
- **Purpose**: Production deployment
- **Data**: Live customer data
- **Functions**: Production-tested Cloud Functions
- **Backups**: Automated daily backups enabled

### Environment Configuration

Environment-specific configuration stored in `.env` files:

```bash
# Development
FIREBASE_PROJECT_ID=findogai-dev
FIREBASE_API_KEY=...
FIRESTORE_EMULATOR_HOST=localhost:8080  # For local development

# Production
FIREBASE_PROJECT_ID=findogai-prod
FIREBASE_API_KEY=...
```

See [Development & Deployment - Environment Configuration](./development-deployment.md#71-environment-configuration) for complete setup.

## Cost Considerations

### Free Tier Limits

- **Firebase Auth**: 50,000 MAU (Monthly Active Users)
- **Firestore**: 50K reads/20K writes/20K deletes per day
- **Cloud Functions**: 2M invocations/month, 400K GB-seconds
- **Cloud Storage**: 5GB storage, 1GB/day downloads

### Estimated Monthly Costs (100 Active Users)

- **Firestore**: ~$5-10 (based on read/write patterns)
- **Cloud Functions**: ~$2-5 (audit logging + sequence allocation)
- **Cloud Storage**: ~$1 (exports and reports)
- **Voice APIs**: ~$10-20 (based on usage)

**Total estimated**: $18-35/month for 100 users

See [Performance Optimization - Cost Optimization](./performance-optimization.md#94-cost-optimization) for detailed breakdown and optimization strategies.

---

[← Back: High Level Architecture](./high-level-architecture.md) | [Back to Index](./index.md) | [Next: Data Model →](./data-model.md)
