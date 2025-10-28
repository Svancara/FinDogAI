[Back to Index](./index.md)

# Technology Stack

This document provides a comprehensive overview of all technologies, runtime environments, and supporting libraries used in the FinDogAI backend.

## 6.1 Firebase Services

| Service | Version | Purpose | Region |
|---------|---------|---------|--------|
| Firebase Authentication | v11+ | User identity management | Global |
| Cloud Firestore | v11+ | NoSQL database with offline sync | europe-west1 |
| Cloud Functions (2nd Gen) | v5+ | Serverless backend logic | europe-west1 |
| Cloud Storage for Firebase | v11+ | File storage (exports, PDFs) | europe-west1 |
| Firebase Hosting | v13+ | PWA hosting (Phase 2) | Global CDN |

### Why Firebase?

**Offline-First**: Built-in offline persistence with automatic sync
**Real-Time**: WebSocket-based listeners for live UI updates
**Scalable**: Automatic scaling from 0 to millions of users
**Secure**: Declarative security rules with granular access control
**GDPR Compliant**: EU region hosting (europe-west1)
**Cost Effective**: Generous free tier, pay-as-you-grow pricing

---

## 6.2 Cloud Functions Runtime

### Node.js Version

**Node.js 20 LTS** (Long-Term Support)

**Rationale:**
- LTS release with long-term support until April 2026
- Native ES modules support
- Performance improvements over Node.js 18
- Required by Firebase Functions v5+

### Key Dependencies

```json
{
  "dependencies": {
    "firebase-admin": "^12.0.0",      // Firebase Admin SDK
    "firebase-functions": "^5.0.0",   // Cloud Functions framework (2nd Gen)
    "pdfmake": "^0.2.10"               // PDF generation (Phase 2)
  },
  "devDependencies": {
    "@types/node": "^20.0.0",          // TypeScript type definitions
    "typescript": "^5.3.0",            // TypeScript compiler
    "eslint": "^8.56.0",               // Code linting
    "jest": "^29.7.0",                 // Unit testing framework
    "@firebase/rules-unit-testing": "^3.0.0"  // Security Rules testing
  }
}
```

**firebase-admin**: Server-side SDK for Firestore, Auth, Storage access
**firebase-functions**: Cloud Functions framework with triggers and callable functions
**pdfmake**: Client-side PDF generation library for Node.js

---

## 6.3 TypeScript for Type Safety

### Shared Types Package

**Location:** `/packages/shared-types`

**Purpose:** Single source of truth for data models shared between mobile app and Cloud Functions.

```typescript
// packages/shared-types/src/models/job.ts

export interface Job {
  tenantId: string;
  jobNumber: number | null;
  title: string;
  description?: string;
  status: 'active' | 'completed' | 'archived';
  currency: string;
  vatRate: number;
  budget?: number;
  createdAt: Timestamp;
  createdBy: string;
  updatedAt: Timestamp;
  updatedBy: string;
}

// Imported by both mobile app and Cloud Functions
import { Job } from '@findogai/shared-types';
```

### Benefits

- **Type safety** across monorepo packages
- **Single source of truth** for data models
- **Compile-time validation** of API contracts
- **IDE autocomplete** and inline documentation
- **Refactoring safety** - rename fields with confidence

### TypeScript Configuration

```json
// packages/functions/tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es2020",
    "lib": ["es2020"],
    "outDir": "lib",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

---

## 6.4 Supporting Libraries

### 6.4.1 Voice API SDKs

```json
{
  "dependencies": {
    "@google-cloud/speech": "^6.5.0",           // Google Cloud STT
    "@google-cloud/text-to-speech": "^5.2.0",   // Google Cloud TTS
    "openai": "^4.28.0",                        // OpenAI API (GPT-4, Whisper)
    "@picovoice/porcupine-web": "^3.0.0"        // On-device keyword spotting
  }
}
```

**Google Cloud Speech-to-Text:**
- Czech language support (cs-CZ)
- Real-time streaming transcription
- Custom vocabulary for domain-specific terms

**Google Cloud Text-to-Speech:**
- WaveNet voices for natural speech
- Czech language support (cs-CZ)
- SSML support for pronunciation control

**OpenAI API:**
- GPT-4 for natural language understanding
- Whisper for alternative STT (Phase 2)
- Structured output parsing

**Picovoice Porcupine:**
- On-device wake word detection
- Low power consumption
- Custom wake word training

### 6.4.2 Utility Libraries

```json
{
  "dependencies": {
    "uuid": "^9.0.1",           // UUID generation for document IDs
    "date-fns": "^3.3.1",       // Date manipulation and formatting
    "zod": "^3.22.4"            // Runtime validation and type inference
  }
}
```

**uuid:**
- Generate UUIDv4 for tenant IDs and document IDs
- Cryptographically secure random IDs

**date-fns:**
- Date arithmetic (add/subtract days, months, etc.)
- Locale-aware formatting (Czech, English)
- Timezone-aware calculations

**zod:**
- Runtime validation of Cloud Function inputs
- Type-safe schema definitions
- Error messages for invalid data

**Example: Zod Validation**

```typescript
import { z } from 'zod';

const AllocateSequenceSchema = z.object({
  tenantId: z.string().uuid(),
  type: z.enum(['jobNumber', 'vehicleNumber', 'machineNumber', 'teamMemberNumber']),
  jobId: z.string().uuid().optional()
});

export const allocateSequence = onCall(async (request) => {
  const { tenantId, type, jobId } = AllocateSequenceSchema.parse(request.data);
  // ... validated data
});
```

---

## 6.5 Testing Stack

### Unit Testing

**Jest** - JavaScript testing framework
- Fast test execution
- Snapshot testing for UI components
- Mock support for Firebase SDK

**@firebase/rules-unit-testing** - Security Rules testing
- Emulator-based testing
- Assertion helpers for rules validation
- Multi-user simulation

### Integration Testing

**Firebase Emulator Suite**
- Local Firestore emulator
- Local Functions emulator
- Local Auth emulator
- Offline testing without cloud costs

### End-to-End Testing

**Playwright** (Phase 2)
- Cross-browser testing
- Mobile emulation
- Screenshot comparison
- Network mocking

See [Testing Strategy](./testing-strategy.md) for detailed testing approach.

---

## 6.6 Development Tools

### Code Quality

```json
{
  "devDependencies": {
    "eslint": "^8.56.0",                         // Linting
    "@typescript-eslint/parser": "^6.19.0",      // TypeScript support
    "@typescript-eslint/eslint-plugin": "^6.19.0",
    "prettier": "^3.2.4",                        // Code formatting
    "eslint-config-prettier": "^9.1.0"           // Prettier integration
  }
}
```

**ESLint Configuration:**

```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier'
  ],
  rules: {
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/explicit-module-boundary-types': 'off'
  }
};
```

### Git Hooks

**Husky** - Git hooks manager
**lint-staged** - Run linters on staged files

```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "pre-push": "npm run test:ci"
    }
  },
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"],
    "*.json": ["prettier --write"]
  }
}
```

---

## 6.7 Monorepo Structure

### Nx Workspace

**Nx** - Monorepo build system
- Incremental builds
- Dependency graph
- Affected commands (test/build only changed packages)
- Code generators

**Workspace Structure:**

```
findogai/
├── packages/
│   ├── mobile-app/           # Ionic/Angular app
│   ├── functions/            # Cloud Functions
│   ├── shared-types/         # TypeScript interfaces
│   └── e2e/                  # End-to-end tests (Phase 2)
├── nx.json                   # Nx configuration
├── package.json              # Root package.json
└── tsconfig.base.json        # Base TypeScript config
```

**Benefits:**
- Shared dependencies (single node_modules)
- Consistent tooling across packages
- Efficient CI/CD (only build affected packages)

---

## 6.8 External Services

### Google Cloud Platform

**Google Cloud Speech-to-Text**
- API: `speech.googleapis.com`
- Pricing: $0.006/15 seconds (standard model)
- Czech language support

**Google Cloud Text-to-Speech**
- API: `texttospeech.googleapis.com`
- Pricing: $16.00 per 1M characters (WaveNet voices)
- Czech WaveNet voices

### OpenAI

**GPT-4**
- API: `api.openai.com/v1/chat/completions`
- Pricing: $0.03/1K input tokens, $0.06/1K output tokens
- Structured output parsing

**Whisper** (Phase 2)
- API: `api.openai.com/v1/audio/transcriptions`
- Pricing: $0.006/minute
- Multilingual support

---

## 6.9 Version Control and Deployment

### Git Workflow

**Branching Strategy:**
- `main` - Production branch
- `staging` - Pre-production testing
- `feature/*` - Feature branches
- `bugfix/*` - Bug fix branches

### CI/CD

**GitHub Actions**
- Automated testing on pull requests
- Automated deployment to Firebase
- Security scanning (Dependabot)

See [Development & Deployment](./development-deployment.md) for CI/CD pipeline details.

---

## 6.10 Monitoring and Logging

### Firebase Monitoring

**Cloud Functions Logs**
- Automatic logging to Cloud Logging
- Structured logging with severity levels
- Log-based metrics and alerts

**Performance Monitoring**
- Function execution time tracking
- Cold start monitoring
- Error rate tracking

### Third-Party Tools (Optional)

**Sentry** - Error tracking and monitoring
**LogRocket** - Session replay and debugging
**Mixpanel** - User analytics (Phase 2)

---

## Version Compatibility Matrix

| Package | Minimum Version | Recommended | Notes |
|---------|----------------|-------------|-------|
| Node.js | 20.0.0 | 20.11.0 | LTS release |
| npm | 10.0.0 | 10.2.4 | Bundled with Node 20 |
| Firebase CLI | 13.0.0 | 13.1.0 | For deployment |
| TypeScript | 5.0.0 | 5.3.3 | Shared across packages |
| Angular | 17.0.0 | 17.1.0 | Mobile app framework |
| Ionic | 7.0.0 | 7.6.5 | Mobile UI components |

---

[← Back: API Design](./api-design.md) | [Back to Index](./index.md) | [Next: Development & Deployment →](./development-deployment.md)
