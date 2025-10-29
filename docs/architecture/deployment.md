[Back to Index](./index.md) | [Previous: Development Workflow](./development-workflow.md) | [Next: Coding Standards](./coding-standards.md)

# Deployment Architecture

## Deployment Strategy

### Frontend Deployment
- **Platform:** Firebase Hosting
- **Build Command:** `nx build mobile-app --prod`
- **Output Directory:** `dist/apps/mobile-app`
- **CDN/Edge:** Global Firebase CDN with automatic SSL

### Backend Deployment
- **Platform:** Cloud Functions (2nd Gen)
- **Build Command:** `nx build functions --prod`
- **Deployment Method:** Firebase CLI with automatic function discovery

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test:all

      - name: Build applications
        run: |
          nx build mobile-app --prod
          nx build functions --prod

      - name: Deploy to Firebase
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
        run: |
          npm run deploy:hosting
          npm run deploy:functions
          npm run deploy:rules
```

### Deployment Stages

1. **Code Checkout**: Pull latest code from repository
2. **Dependencies**: Install npm packages with caching
3. **Testing**: Run all unit and integration tests
4. **Build**: Compile frontend and backend with production optimizations
5. **Deploy**: Deploy to Firebase services sequentially
6. **Verify**: Run smoke tests on deployed environment

## Environments

| Environment | Frontend URL | Backend URL | Purpose |
|------------|--------------|-------------|---------|
| Development | http://localhost:4200 | http://localhost:5001 | Local development with emulators |
| Staging | https://findogai-staging.web.app | https://europe-west1-findogai-staging.cloudfunctions.net | Pre-production testing |
| Production | https://findogai.app | https://europe-west1-findogai-prod.cloudfunctions.net | Live environment |

## Environment Configuration

### Development Environment
```bash
# .env.development
FIREBASE_PROJECT_ID=findogai-dev
FIREBASE_REGION=europe-west1
USE_EMULATORS=true
LOG_LEVEL=debug
```

### Staging Environment
```bash
# .env.staging
FIREBASE_PROJECT_ID=findogai-staging
FIREBASE_REGION=europe-west1
USE_EMULATORS=false
LOG_LEVEL=info
```

### Production Environment
```bash
# .env.production
FIREBASE_PROJECT_ID=findogai-prod
FIREBASE_REGION=europe-west1
USE_EMULATORS=false
LOG_LEVEL=warn
SENTRY_ENABLED=true
```

## Deployment Commands

### Manual Deployment
```bash
# Deploy everything
firebase deploy

# Deploy only hosting
firebase deploy --only hosting

# Deploy only functions
firebase deploy --only functions

# Deploy only Firestore rules
firebase deploy --only firestore:rules

# Deploy to specific project
firebase deploy --project=findogai-prod
```

### Selective Function Deployment
```bash
# Deploy single function
firebase deploy --only functions:allocateSequence

# Deploy function group
firebase deploy --only functions:triggers,functions:callables
```

## Firebase Hosting Configuration

### firebase.json
```json
{
  "hosting": {
    "public": "dist/apps/mobile-app",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(jpg|jpeg|gif|png|svg|webp)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=31536000"
          }
        ]
      },
      {
        "source": "**/*.@(js|css)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=31536000"
          }
        ]
      }
    ]
  }
}
```

## Cloud Functions Configuration

### Function Runtime Settings
```typescript
// apps/functions/src/index.ts
export const allocateSequence = onCall({
  region: 'europe-west1',
  memory: '256MiB',
  timeoutSeconds: 60,
  maxInstances: 100,
  minInstances: 0, // Scale to zero
  cpu: 1
}, async (request) => {
  // Function implementation
});
```

### Function Environment Variables
```bash
# Set function config
firebase functions:config:set openai.key="YOUR_KEY"
firebase functions:config:set anthropic.key="YOUR_KEY"

# Get current config
firebase functions:config:get
```

## Rollback Strategy

### Hosting Rollback
```bash
# List previous releases
firebase hosting:channel:list

# Deploy to channel
firebase hosting:channel:deploy preview

# Rollback to previous version
firebase hosting:rollback
```

### Functions Rollback
```bash
# Functions don't support automatic rollback
# Manual process:
# 1. Checkout previous commit
# 2. Deploy functions from that commit
git checkout <previous-commit-hash>
firebase deploy --only functions
```

## Monitoring and Alerting

### Firebase Performance Monitoring
- Automatically tracks web performance metrics
- Custom traces for critical operations
- Real-time performance data in Firebase Console

### Error Tracking (Sentry)
```typescript
// Initialize Sentry in production
if (environment.production) {
  Sentry.init({
    dsn: environment.sentryDsn,
    environment: environment.name,
    tracesSampleRate: 0.1,
  });
}
```

### Cloud Logging
```typescript
// Structured logging in Cloud Functions
import { logger } from 'firebase-functions/v2';

logger.info('Job created', {
  tenantId: 'tenant123',
  jobId: 'job456',
  userId: 'user789'
});
```

### Alerts Configuration
```yaml
# Alert on high error rate
- alert: HighErrorRate
  condition: error_rate > 5%
  duration: 5m
  severity: critical
  notify: pagerduty

# Alert on slow functions
- alert: SlowFunctions
  condition: p95_latency > 5s
  duration: 10m
  severity: warning
  notify: slack
```

## Security Best Practices

### Secrets Management
- Store API keys in Firebase Functions config
- Use environment-specific secrets
- Rotate secrets regularly
- Never commit secrets to repository

### Firestore Security Rules
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Tenant isolation
    match /tenants/{tenantId}/{document=**} {
      allow read, write: if request.auth != null &&
        exists(/databases/$(database)/documents/tenants/$(tenantId)/members/$(request.auth.uid));
    }
  }
}
```

### Content Security Policy
```html
<!-- meta tag in index.html -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' 'unsafe-inline';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data: https:;">
```

## Performance Optimization

### Frontend Optimization
- **Code splitting**: Lazy load routes and modules
- **Tree shaking**: Remove unused code
- **Minification**: Compress JavaScript and CSS
- **Image optimization**: WebP format with fallbacks
- **Service worker**: Cache static assets

### Backend Optimization
- **Cold start reduction**: Keep functions warm with scheduled pings
- **Connection pooling**: Reuse Firestore connections
- **Caching**: Implement response caching where appropriate
- **Batch operations**: Group Firestore operations

## Cost Management

### Monitoring Costs
- Set up billing alerts in GCP Console
- Monitor function invocations and execution time
- Track Firestore read/write operations
- Review bandwidth usage monthly

### Cost Optimization
- Use Cloud Functions 2nd Gen (cheaper cold starts)
- Implement proper caching to reduce database reads
- Optimize function memory allocation
- Set max instances to prevent runaway costs
- Use Cloud Scheduler for batch operations during off-peak hours

## Disaster Recovery

### Backup Strategy
- **Firestore**: Automated daily backups
- **Storage**: Versioning enabled on all buckets
- **Configuration**: Git repository as source of truth

### Recovery Procedures
1. Identify scope of issue
2. Roll back to last known good deployment
3. Restore data from backups if needed
4. Verify system functionality
5. Post-mortem analysis

---

*Next: [Coding Standards](./coding-standards.md)*
