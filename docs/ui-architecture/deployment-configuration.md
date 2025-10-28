[Back to Index](./index.md)

# Deployment Configuration

## Environment Configuration

### Development Environment

```typescript
// apps/mobile-app/src/environments/environment.ts
export const environment = {
  production: false,
  appVersion: '1.0.0-dev',

  // Firebase Configuration
  firebase: {
    apiKey: 'AIzaSyDEVELOPMENT_KEY_HERE',
    authDomain: 'findogai-dev.firebaseapp.com',
    projectId: 'findogai-dev',
    storageBucket: 'findogai-dev.appspot.com',
    messagingSenderId: '123456789',
    appId: '1:123456789:web:abcdef123456',
    measurementId: 'G-DEVELOPMENT'
  },

  // Cloud Functions Endpoints
  functionsUrl: 'http://localhost:5001/findogai-dev/us-central1',
  functionsRegion: 'us-central1',

  // Voice Pipeline Configuration
  voice: {
    speechRecognitionLang: 'en-US',
    speechSynthesisVoice: 'Google US English',
    commandTimeout: 5000, // 5 seconds
    wakeWord: 'Hey FinDog', // Optional wake word
    llmProvider: 'openai', // or 'anthropic'
    llmApiKey: process.env['LLM_API_KEY'] || '', // Load from env
    llmModel: 'gpt-4o-mini' // or 'claude-3-haiku'
  },

  // Offline Configuration
  offline: {
    enablePersistence: true,
    cacheSizeBytes: 50 * 1024 * 1024, // 50MB
    syncInterval: 30000, // 30 seconds
    maxRetries: 3,
    retryDelay: 1000 // 1 second base delay
  },

  // PWA Configuration
  pwa: {
    enabled: true,
    updateCheckInterval: 3600000, // 1 hour
    updateMode: 'prompt', // or 'auto'
    offlineGoogleAnalytics: false
  },

  // Feature Flags
  features: {
    voiceCommands: true,
    offlineMode: true,
    pdfGeneration: false, // Phase 2
    advancedReporting: false, // Phase 2
    teamCollaboration: true,
    costTracking: true
  },

  // Logging
  logging: {
    level: 'debug', // 'error' | 'warn' | 'info' | 'debug'
    remoteLogging: false,
    logToConsole: true
  },

  // Development Tools
  devTools: {
    reduxDevTools: true,
    performanceMonitoring: true,
    showDebugInfo: true
  }
};
```

### Production Environment

```typescript
// apps/mobile-app/src/environments/environment.prod.ts
export const environment = {
  production: true,
  appVersion: '1.0.0',

  // Firebase Configuration (Production)
  firebase: {
    apiKey: process.env['FIREBASE_API_KEY'],
    authDomain: 'findogai.firebaseapp.com',
    projectId: 'findogai-prod',
    storageBucket: 'findogai-prod.appspot.com',
    messagingSenderId: process.env['FIREBASE_MESSAGING_SENDER_ID'],
    appId: process.env['FIREBASE_APP_ID'],
    measurementId: process.env['FIREBASE_MEASUREMENT_ID']
  },

  // Cloud Functions Endpoints (Production)
  functionsUrl: 'https://us-central1-findogai-prod.cloudfunctions.net',
  functionsRegion: 'us-central1',

  // Voice Pipeline Configuration
  voice: {
    speechRecognitionLang: 'en-US',
    speechSynthesisVoice: 'Google US English',
    commandTimeout: 5000,
    wakeWord: 'Hey FinDog',
    llmProvider: process.env['LLM_PROVIDER'] || 'openai',
    llmApiKey: process.env['LLM_API_KEY'],
    llmModel: process.env['LLM_MODEL'] || 'gpt-4o-mini'
  },

  // Offline Configuration
  offline: {
    enablePersistence: true,
    cacheSizeBytes: 100 * 1024 * 1024, // 100MB
    syncInterval: 60000, // 1 minute
    maxRetries: 5,
    retryDelay: 2000 // 2 seconds base delay
  },

  // PWA Configuration
  pwa: {
    enabled: true,
    updateCheckInterval: 7200000, // 2 hours
    updateMode: 'prompt',
    offlineGoogleAnalytics: true
  },

  // Feature Flags (Production)
  features: {
    voiceCommands: true,
    offlineMode: true,
    pdfGeneration: false, // Enable in Phase 2
    advancedReporting: false,
    teamCollaboration: true,
    costTracking: true
  },

  // Logging (Production)
  logging: {
    level: 'error',
    remoteLogging: true,
    logToConsole: false,
    sentryDsn: process.env['SENTRY_DSN'] // Error tracking
  },

  // Development Tools (Disabled in Production)
  devTools: {
    reduxDevTools: false,
    performanceMonitoring: true,
    showDebugInfo: false
  }
};
```

### Environment Variable Files

```bash
# .env.local (Development - Git ignored)
FIREBASE_API_KEY=your_dev_api_key_here
FIREBASE_AUTH_DOMAIN=findogai-dev.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-dev
FIREBASE_STORAGE_BUCKET=findogai-dev.appspot.com
FIREBASE_MESSAGING_SENDER_ID=123456789
FIREBASE_APP_ID=1:123456789:web:abcdef
FIREBASE_MEASUREMENT_ID=G-XXXXX

# LLM Configuration
LLM_PROVIDER=openai
LLM_API_KEY=sk-dev-xxxxxxxxxxxxx
LLM_MODEL=gpt-4o-mini

# Development URLs
API_URL=http://localhost:5001
APP_URL=http://localhost:4200

# Feature Flags
ENABLE_DEBUG_MODE=true
ENABLE_MOCK_DATA=true

# .env.production (Production - Secure storage)
FIREBASE_API_KEY=${SECRET_FIREBASE_API_KEY}
FIREBASE_AUTH_DOMAIN=findogai.firebaseapp.com
FIREBASE_PROJECT_ID=findogai-prod
FIREBASE_STORAGE_BUCKET=findogai-prod.appspot.com
FIREBASE_MESSAGING_SENDER_ID=${SECRET_FIREBASE_SENDER_ID}
FIREBASE_APP_ID=${SECRET_FIREBASE_APP_ID}
FIREBASE_MEASUREMENT_ID=${SECRET_MEASUREMENT_ID}

# LLM Configuration
LLM_PROVIDER=openai
LLM_API_KEY=${SECRET_LLM_API_KEY}
LLM_MODEL=gpt-4o-mini

# Production URLs
API_URL=https://api.findogai.com
APP_URL=https://app.findogai.com

# Error Tracking
SENTRY_DSN=${SECRET_SENTRY_DSN}

# Analytics
GA_TRACKING_ID=${SECRET_GA_TRACKING_ID}
```

## Build Configuration

### Nx Project Configuration

```json
// apps/mobile-app/project.json (Nx configuration)
{
  "name": "mobile-app",
  "targets": {
    "build": {
      "configurations": {
        "development": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.ts"
            }
          ],
          "optimization": false,
          "sourceMap": true,
          "namedChunks": true
        },
        "production": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.prod.ts"
            }
          ],
          "optimization": true,
          "sourceMap": false,
          "namedChunks": false,
          "serviceWorker": true,
          "ngswConfigPath": "apps/mobile-app/ngsw-config.json"
        },
        "staging": {
          "fileReplacements": [
            {
              "replace": "apps/mobile-app/src/environments/environment.ts",
              "with": "apps/mobile-app/src/environments/environment.staging.ts"
            }
          ],
          "optimization": true,
          "sourceMap": true
        }
      }
    },
    "serve": {
      "configurations": {
        "development": {
          "buildTarget": "mobile-app:build:development",
          "hmr": true,
          "port": 4200
        },
        "production": {
          "buildTarget": "mobile-app:build:production",
          "port": 4200
        }
      }
    }
  }
}
```

### Angular Build Optimization

```json
// angular.json
{
  "projects": {
    "mobile-app": {
      "architect": {
        "build": {
          "options": {
            "outputPath": "dist/apps/mobile-app",
            "index": "apps/mobile-app/src/index.html",
            "main": "apps/mobile-app/src/main.ts",
            "polyfills": ["zone.js"],
            "tsConfig": "apps/mobile-app/tsconfig.app.json",
            "assets": [
              "apps/mobile-app/src/favicon.ico",
              "apps/mobile-app/src/assets",
              "apps/mobile-app/src/manifest.json"
            ],
            "styles": [
              "apps/mobile-app/src/theme/variables.scss",
              "apps/mobile-app/src/global.scss"
            ],
            "scripts": []
          },
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "4kb",
                  "maximumError": "8kb"
                }
              ],
              "outputHashing": "all",
              "optimization": {
                "scripts": true,
                "styles": {
                  "minify": true,
                  "inlineCritical": true
                },
                "fonts": true
              }
            }
          }
        }
      }
    }
  }
}
```

## Capacitor Configuration

```typescript
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.findogai.app',
  appName: 'FinDogAI',
  webDir: 'dist/apps/mobile-app',
  bundledWebRuntime: false,
  server: {
    // Development server config
    url: process.env['CAP_SERVER_URL'], // For live reload
    cleartext: true // Allow HTTP in development
  },
  plugins: {
    App: {
      iosScheme: 'findogai',
      androidScheme: 'findogai'
    },
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#3880ff',
      showSpinner: true,
      spinnerColor: '#ffffff'
    },
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert']
    }
  },
  ios: {
    contentInset: 'automatic',
    limitsNavigationsToAppBoundDomains: false
  },
  android: {
    minWebViewVersion: 80,
    enableJetifier: true
  }
};

export default config;
```

## Build Scripts

```json
// package.json
{
  "scripts": {
    "start": "nx serve mobile-app",
    "build": "nx build mobile-app",
    "build:prod": "nx build mobile-app --configuration=production",
    "build:staging": "nx build mobile-app --configuration=staging",

    "test": "nx test mobile-app",
    "test:coverage": "nx test mobile-app --coverage",
    "e2e": "nx e2e mobile-app-e2e",

    "lint": "nx lint mobile-app",
    "lint:fix": "nx lint mobile-app --fix",

    "cap:sync": "npx cap sync",
    "cap:open:ios": "npx cap open ios",
    "cap:open:android": "npx cap open android",
    "cap:build:ios": "npm run build:prod && npx cap sync ios && npx cap open ios",
    "cap:build:android": "npm run build:prod && npx cap sync android && npx cap open android",

    "deploy:web": "npm run build:prod && firebase deploy --only hosting",
    "deploy:functions": "firebase deploy --only functions",
    "deploy:all": "npm run build:prod && firebase deploy"
  }
}
```

## PWA Manifest

```json
// apps/mobile-app/src/manifest.json
{
  "name": "FinDogAI",
  "short_name": "FinDogAI",
  "theme_color": "#3880ff",
  "background_color": "#ffffff",
  "display": "standalone",
  "orientation": "portrait",
  "scope": "/",
  "start_url": "/",
  "icons": [
    {
      "src": "assets/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "assets/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

## CI/CD Pipeline (GitHub Actions Example)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm run test:coverage

      - name: E2E Tests
        run: npm run e2e:ci

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build:prod
        env:
          FIREBASE_API_KEY: ${{ secrets.FIREBASE_API_KEY }}
          FIREBASE_PROJECT_ID: ${{ secrets.FIREBASE_PROJECT_ID }}
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: dist

      - name: Deploy to Firebase
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: '${{ secrets.GITHUB_TOKEN }}'
          firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}'
          channelId: live
          projectId: findogai-prod
```

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Lint checks passing
- [ ] Code coverage meets threshold (80%+)
- [ ] E2E tests passing
- [ ] Performance benchmarks met
- [ ] Security audit completed
- [ ] Environment variables configured
- [ ] Firebase configuration updated

### Production Build
- [ ] Bundle size within limits
- [ ] Source maps disabled
- [ ] Service worker enabled
- [ ] PWA manifest validated
- [ ] Icons and assets optimized
- [ ] Compression enabled

### Post-Deployment
- [ ] Smoke tests passing
- [ ] Performance monitoring active
- [ ] Error tracking configured
- [ ] Analytics working
- [ ] PWA installable
- [ ] Offline mode functional
- [ ] Voice commands working

---

[← Previous: Performance Optimization](./performance-optimization.md) | [Back to Index](./index.md)
