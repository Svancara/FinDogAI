[Back to Index](./index.md)

# Development & Deployment

This document covers environment configuration, CI/CD pipelines, deployment processes, and monitoring setup for the FinDogAI backend.

## 7.1 Environment Configuration

### 7.1.1 Firebase Projects

| Environment | Project ID | Purpose |
|-------------|------------|---------|
| Development | findogai-dev | Local development with emulators |
| Staging | findogai-staging | Pre-production testing |
| Production | findogai-prod | Live user data |

### 7.1.2 Environment Variables

**Local Development (.env.local - NOT committed to git):**

```bash
# Firebase configuration (from Firebase Console)
FIREBASE_API_KEY=AIza...
FIREBASE_AUTH_DOMAIN=findogai-dev.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-dev

# Voice API keys
GOOGLE_CLOUD_API_KEY=AIza...
OPENAI_API_KEY=sk-proj-...
GROQ_API_KEY=gsk_...

# Provider selection (for development)
STT_PROVIDER=mock
LLM_PROVIDER=mock
TTS_PROVIDER=platform-native
```

**Production (.env.production - values stored in CI/CD secrets):**

```bash
FIREBASE_PROJECT_ID=findogai-prod
GOOGLE_CLOUD_API_KEY=${GOOGLE_CLOUD_API_KEY}
OPENAI_API_KEY=${OPENAI_API_KEY}
STT_PROVIDER=google-cloud
LLM_PROVIDER=openai-gpt4
TTS_PROVIDER=google-cloud
```

---

## 7.2 CI/CD Pipeline

### 7.2.1 GitHub Actions Workflow

**File:** `.github/workflows/deploy.yml`

```yaml
name: Deploy to Firebase

on:
  push:
    branches:
      - main        # Production deployment
      - staging     # Staging deployment

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:ci

      - name: Run Security Rules tests
        run: npm run test:rules

  deploy-functions:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Firebase CLI
        run: npm install -g firebase-tools

      - name: Deploy Cloud Functions
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
          GOOGLE_CLOUD_API_KEY: ${{ secrets.GOOGLE_CLOUD_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          firebase use ${{ github.ref == 'refs/heads/main' && 'findogai-prod' || 'findogai-staging' }}
          firebase deploy --only functions

      - name: Deploy Firestore Rules and Indexes
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: firebase deploy --only firestore:rules,firestore:indexes

  deploy-hosting:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Build mobile app (PWA)
        run: |
          cd packages/mobile-app
          npm run build:prod

      - name: Deploy to Firebase Hosting
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: |
          firebase use ${{ github.ref == 'refs/heads/main' && 'findogai-prod' || 'findogai-staging' }}
          firebase deploy --only hosting
```

---

## 7.3 Firebase Deployment Process

### 7.3.1 Manual Deployment (Development)

```bash
# Select Firebase project
firebase use findogai-dev

# Deploy all services
firebase deploy

# Deploy specific services
firebase deploy --only functions
firebase deploy --only firestore:rules
firebase deploy --only firestore:indexes
firebase deploy --only hosting
firebase deploy --only storage
```

### 7.3.2 Functions Deployment Best Practices

```bash
# Deploy specific function (for faster iteration)
firebase deploy --only functions:allocateSequence

# Deploy with function config
firebase functions:config:set \
  openai.api_key="sk-proj-..." \
  google.api_key="AIza..."

firebase deploy --only functions
```

**Important Notes:**
- Always test in development environment first
- Use `--only` flag to deploy specific services
- Verify security rules before deploying to production
- Monitor logs after deployment for errors

---

## 7.4 Monitoring and Logging

### 7.4.1 Cloud Functions Logs

**View logs in Firebase Console:**
- Functions > Logs tab
- Filter by function name, severity, or time range

**View logs via CLI:**

```bash
# Stream all function logs
firebase functions:log

# Stream specific function logs
firebase functions:log --only allocateSequence

# View logs for specific time range
firebase functions:log --since 1h
```

**Structured Logging:**

```typescript
// Use console.log for INFO level
console.log('Job created:', { jobId, tenantId });

// Use console.error for ERROR level
console.error('Failed to allocate sequence:', error);

// Use console.warn for WARNING level
console.warn('Offline persistence unavailable');
```

### 7.4.2 Firestore Monitoring

**Firebase Console:**
- Firestore > Usage tab
- Monitor reads, writes, deletes per day
- Track document count and storage size

**Cloud Monitoring (Google Cloud Console):**
- Create custom dashboards
- Set up alerts for quota limits
- Monitor query performance

**Metrics to Monitor:**
- Document reads/writes per day
- Average query latency
- Failed security rule checks
- Storage size growth

### 7.4.3 Performance Monitoring

**Cloud Functions Performance:**
- Execution time (target: <500ms per NFR14)
- Cold start frequency
- Error rate
- Invocation count

**Set up Alerts:**

```bash
# Example: Alert on high error rate
gcloud alpha monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High Function Error Rate" \
  --condition-threshold-value=0.05 \
  --condition-threshold-duration=300s \
  --condition-display-name="Error rate > 5%"
```

---

## Local Development with Emulators

### 7.5.1 Firebase Emulator Suite

**Start emulators:**

```bash
# Start all emulators
firebase emulators:start

# Start specific emulators
firebase emulators:start --only firestore,functions,auth
```

**Emulator Configuration (firebase.json):**

```json
{
  "emulators": {
    "auth": {
      "port": 9099
    },
    "firestore": {
      "port": 8080
    },
    "functions": {
      "port": 5001
    },
    "storage": {
      "port": 9199
    },
    "ui": {
      "enabled": true,
      "port": 4000
    }
  }
}
```

**Connect App to Emulators:**

```typescript
// packages/mobile-app/src/app/app.config.ts

import { connectAuthEmulator } from '@angular/fire/auth';
import { connectFirestoreEmulator } from '@angular/fire/firestore';
import { connectFunctionsEmulator } from '@angular/fire/functions';

export const appConfig: ApplicationConfig = {
  providers: [
    provideFirebaseApp(() => initializeApp(environment.firebase)),
    provideAuth(() => {
      const auth = getAuth();
      if (environment.useEmulators) {
        connectAuthEmulator(auth, 'http://localhost:9099');
      }
      return auth;
    }),
    provideFirestore(() => {
      const firestore = getFirestore();
      if (environment.useEmulators) {
        connectFirestoreEmulator(firestore, 'localhost', 8080);
      }
      return firestore;
    }),
    provideFunctions(() => {
      const functions = getFunctions(undefined, 'europe-west1');
      if (environment.useEmulators) {
        connectFunctionsEmulator(functions, 'localhost', 5001);
      }
      return functions;
    })
  ]
};
```

### 7.5.2 Seed Data for Development

```bash
# Import seed data to emulator
firebase emulators:start --import=./seed-data

# Export emulator data for reuse
firebase emulators:export ./seed-data
```

---

## Troubleshooting

### Common Deployment Issues

**Issue: Functions deployment timeout**

```bash
# Increase timeout
firebase deploy --only functions --force

# Deploy functions one at a time
firebase deploy --only functions:allocateSequence
```

**Issue: Security rules validation error**

```bash
# Test rules locally first
firebase emulators:start --only firestore
npm run test:rules
```

**Issue: Out of quota errors**

Check Firebase Console > Usage and Billing
- Firestore: 50K reads/day free tier
- Functions: 2M invocations/month free tier
- Upgrade to Blaze plan if exceeded

---

For detailed CI/CD pipeline configuration and monitoring setup, see the [original backend architecture document](../backend-architecture.md).

---

[← Back: Technology Stack](./technology-stack.md) | [Back to Index](./index.md) | [Next: Security Considerations →](./security-considerations.md)
