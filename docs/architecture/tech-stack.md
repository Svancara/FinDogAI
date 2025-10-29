[Back to Index](./index.md) | [Previous: High-Level Overview](./high-level-overview.md) | [Next: Data Models](./data-models.md)

# Technology Stack

This is the **DEFINITIVE** technology selection for the entire FinDogAI project. All development must use these exact specifications.

## Technology Stack Table

| Category | Technology | Version | Purpose | Rationale |
|----------|------------|---------|---------|-----------|
| Frontend Language | **TypeScript** | 5.3+ | Type-safe development | Shared types across fullstack, better IDE support, reduced runtime errors |
| Frontend Framework | **Angular** | 20+ | Core application framework | Latest with signals for reactivity, standalone components, excellent PWA support |
| UI Component Library | **Ionic Framework** | 8+ | Mobile-optimized UI | Native-feel components, platform-adaptive styling, touch-optimized |
| State Management | **NgRx** | 18+ | Application state | Enterprise-grade with devtools, perfect for complex voice flows |
| Backend Language | **TypeScript** | 5.3+ | Cloud Functions development | Type sharing with frontend, async/await support |
| Backend Framework | **Firebase Functions** | 5+ (2nd Gen) | Serverless compute | Auto-scaling, integrated with Firestore triggers, cost-effective |
| API Style | **Direct Firestore + Callable Functions** | SDK v11+ | Data access pattern | Offline-first with SDK, callable functions for server operations |
| Database | **Cloud Firestore** | v11+ | NoSQL with offline sync | Built-in offline persistence, real-time listeners, automatic sync |
| Cache | **Firestore Offline** | Built-in | Client-side caching | Automatic with enablePersistence(), no additional setup |
| File Storage | **Cloud Storage for Firebase** | v11+ | PDFs and exports | Integrated with Functions, signed URLs for security |
| Authentication | **Firebase Auth** | v11+ | Identity management | Multiple providers, offline token caching, custom claims for roles |
| Frontend Testing | **Jest + Angular Testing** | 29+ / 20+ | Unit and integration tests | Fast execution, component testing utilities |
| Backend Testing | **Jest + Functions Test SDK** | 29+ / 5+ | Cloud Functions testing | Offline function testing, mock Firestore |
| E2E Testing | **Playwright** | 1.40+ | Cross-platform E2E | Web and mobile testing, PWA support |
| Build Tool | **Angular CLI + Nx** | 20+ / 19+ | Build and monorepo | Nx for monorepo, Angular CLI for schematics |
| Bundler | **Vite (via Angular)** | 5+ | Fast HMR and builds | Integrated with Angular 20+, instant hot reload |
| IaC Tool | **Firebase CLI + Terraform** | 13+ / 1.5+ | Infrastructure as code | Firebase for services, Terraform for GCP resources |
| CI/CD | **GitHub Actions** | Latest | Automated deployment | Free for public repos, excellent Firebase integration |
| Monitoring | **Firebase Monitoring + Sentry** | Latest / 7+ | Performance and errors | Firebase for metrics, Sentry for error tracking |
| Logging | **Cloud Logging** | Latest | Centralized logs | Integrated with Cloud Functions, structured logging |
| CSS Framework | **Ionic CSS Variables + SCSS** | 8+ / Latest | Styling and theming | CSS custom properties for runtime theming, SCSS for development |

## Additional Technology Specifications

### Voice Pipeline Technologies
- **STT Online:** Web Speech API (primary), Google Cloud Speech-to-Text (fallback)
- **LLM Integration:** OpenAI GPT-4 / Anthropic Claude via REST APIs
- **TTS Online:** Web Speech Synthesis API (primary), Google Cloud Text-to-Speech (fallback)
- **Offline Fallback:** Basic command recognition with cached responses

### Mobile/Native Technologies
- **Runtime:** Capacitor 5+
- **Plugins:** @capacitor/filesystem, @capacitor/camera, @capacitor/speech-recognition

### Development Tools
- **Package Manager:** npm 10+ (Nx integrated)
- **Linting:** ESLint 8+ with Angular recommended rules
- **Formatting:** Prettier 3+ with Angular configuration
- **Git Hooks:** Husky 8+ with lint-staged

## Version Compatibility Matrix

### Firebase Services Alignment
All Firebase services must use matching major versions to ensure compatibility:
- Firebase SDK: v11.x
- AngularFire: v18.x (compatible with Angular 20)
- Firebase Admin SDK: v12.x
- Firebase CLI: v13.x

### Angular Ecosystem
- Angular 20 requires:
  - TypeScript 5.3+
  - RxJS 7.8+
  - Zone.js 0.14+

### Node.js Requirements
- Cloud Functions: Node.js 20 LTS
- Local Development: Node.js 20 LTS
- CI/CD: Node.js 20 LTS

## Technology Selection Rationale

### Why These Choices?

1. **TypeScript Everywhere**: Single language across stack reduces context switching
2. **Firebase Platform**: Integrated solution with offline-first capabilities
3. **Angular + Ionic**: Mature PWA framework with mobile-optimized components
4. **NgRx**: Predictable state management crucial for offline sync
5. **Playwright**: Modern E2E testing with mobile web support
6. **Nx Monorepo**: Efficient builds and explicit dependency management

### Alternatives Considered and Rejected

| Alternative | Reason for Rejection |
|------------|---------------------|
| React Native | Requires separate native codebase, no PWA |
| Express.js Backend | No built-in offline sync, requires custom implementation |
| MongoDB | No real-time sync, requires custom offline solution |
| Redux Toolkit | Less integrated with Angular ecosystem |
| Cypress | Limited mobile web testing capabilities |

## Upgrade Policy

- **Security Updates**: Applied immediately
- **Minor Updates**: Monthly evaluation
- **Major Updates**: Quarterly evaluation with testing
- **LTS Versions**: Preferred for all critical dependencies

---

*Next: [Data Models](./data-models.md)*