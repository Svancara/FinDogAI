[Back to Index](./index.md)

# Frontend Tech Stack

## Technology Stack Table

| Category | Technology | Version | Purpose | Rationale |
|----------|------------|---------|---------|-----------|
| Framework | **Angular** | **20+** | Core application framework | Latest Angular with signals, improved performance, and enhanced developer experience |
| UI Library | Ionic Framework | 8+ | Mobile UI components & PWA | Native-feel components, PWA support critical for project, platform-specific styling |
| State Management | **NgRx** | 18+ | Application state management | Enterprise-grade state management with devtools, perfect for complex voice pipeline flows |
| Routing | Angular Router + Ionic Navigation | 20+ / 8+ | Navigation & deep linking | Built-in lazy loading, guards, PWA routing support |
| Build Tool | Angular CLI + **Nx** + Vite | 20+ / 19+ / 5+ | Monorepo & development | Nx for monorepo management, Vite for fast HMR, Angular schematics |
| Styling | Ionic CSS Variables + SCSS | - | Theming & styles | Platform-adaptive theming, CSS custom properties for runtime theming |
| Testing | Jest + Playwright | 29+ / 1.40+ | Unit & E2E testing | Jest for fast unit tests, Playwright for cross-platform E2E and PWA testing |
| Component Library | Ionic Components | 8+ | UI component system | Pre-built accessible components optimized for touch, mobile, and PWA |
| Form Handling | Angular Reactive Forms | 20+ | Form management | Built-in validation, type-safe forms with signals integration |
| Animation | Angular Animations + Ionic | 20+ | UI animations | Hardware-accelerated animations, gesture-based interactions |
| Dev Tools | Angular DevTools + Redux DevTools | Latest | Development debugging | Component tree inspection, NgRx state debugging, performance profiling |
| PWA | @angular/pwa + Workbox | 20+ / 7+ | Progressive Web App | Offline support, installability, push notifications, app-like experience |

## Additional Frontend Dependencies

### Firebase/Firestore Integration
- @angular/fire (v17+) - Angular Firebase integration
- firebase (v10+) - Core Firebase SDK with offline persistence

### Voice Pipeline (Client-side)
- Web Speech API or Capacitor Speech Recognition plugin
- OpenAI/Anthropic SDK for LLM integration
- Web Speech Synthesis API or native TTS via Capacitor

### Mobile/Native
- Capacitor (v5+) - Native runtime and plugins
- Capacitor plugins for device features (filesystem, camera, etc.)

### Monorepo Tooling
- Nx for monorepo management
- Shared TypeScript types package (`/packages/shared-types`)

## Technology Rationale

### Angular 20+
**Why Angular?**
- **Enterprise-grade framework** with strong TypeScript support
- **Signals** for fine-grained reactivity and improved performance
- **Standalone components** reduce boilerplate and improve tree-shaking
- **Built-in PWA support** through @angular/pwa
- **Excellent tooling** with Angular CLI and DevTools
- **Long-term support** and regular updates from Google

**Key Features Used:**
- Signals for reactive state management
- Standalone components and directives
- `inject()` function for cleaner dependency injection
- Control flow syntax (@if, @for, @switch)
- Improved hydration for better initial load

### Ionic Framework 8+
**Why Ionic?**
- **Native-feel UI components** optimized for mobile
- **PWA capabilities** built-in
- **Platform-specific styling** (iOS vs Android)
- **Touch-optimized components** crucial for field work
- **Capacitor integration** for native features
- **Consistent design language** across platforms

**Key Components Used:**
- ion-tabs for bottom navigation
- ion-card for content containers
- ion-fab for floating action buttons
- ion-button with proper touch targets
- ion-modal for overlays
- ion-toast for notifications

### NgRx 18+
**Why NgRx?**
- **Predictable state management** with Redux pattern
- **Time-travel debugging** with Redux DevTools
- **Entity adapters** for normalized state
- **Effect handling** for async operations
- **Excellent TypeScript support** with typed actions and selectors
- **Offline-first patterns** with optimistic updates

**State Architecture:**
- Feature-based state organization
- Entity adapters for collections (jobs, costs, etc.)
- Effects for Firestore synchronization
- Selectors with memoization
- Offline queue management

### Nx Monorepo
**Why Nx?**
- **Monorepo management** for shared code
- **Dependency graph** visualization
- **Affected commands** for efficient CI/CD
- **Code generation** with schematics
- **Build caching** for faster builds
- **Task orchestration** across projects

**Monorepo Structure:**
- `/apps/mobile-app` - Main Angular/Ionic application
- `/libs/shared/types` - Shared TypeScript interfaces
- `/libs/shared/utils` - Shared utility functions
- `/libs/mobile/ui` - Shared Ionic components
- `/libs/mobile/data-access` - Shared services and state

### Jest + Playwright
**Why Jest?**
- **Fast execution** with parallel test running
- **Watch mode** for rapid development
- **Snapshot testing** for UI components
- **Excellent mocking** capabilities
- **Built-in coverage** reporting

**Why Playwright?**
- **Multi-browser support** (Chromium, Firefox, WebKit)
- **Mobile emulation** for testing on different devices
- **Network control** for offline testing
- **Screenshot and video** recording
- **Accessibility testing** built-in

### Firebase + Firestore
**Why Firebase/Firestore?**
- **Offline persistence** out of the box
- **Real-time synchronization** when online
- **Conflict resolution** with last-write-wins
- **Scalable** NoSQL database
- **Authentication** integrated
- **Cloud Functions** for backend logic

**Offline Features:**
- Automatic local caching
- Optimistic UI updates
- Queue pending operations
- Sync when connection restored
- Conflict detection and resolution

### Capacitor 5+
**Why Capacitor?**
- **Web-first approach** with native capabilities
- **Plugin ecosystem** for device features
- **iOS and Android support** from single codebase
- **Live reload** during development
- **PWA compatibility** maintained
- **TypeScript support** throughout

**Plugins Used:**
- @capacitor/core - Core runtime
- @capacitor/filesystem - File operations
- @capacitor/camera - Photo capture
- @capacitor/geolocation - GPS tracking
- @capacitor/network - Network status
- @capacitor/speech-recognition - Voice input

## Development Environment

### Required Tools
- Node.js 20+ LTS
- npm 10+ or pnpm 8+
- Git
- VS Code (recommended) with extensions:
  - Angular Language Service
  - Prettier
  - ESLint
  - Nx Console

### Optional Tools
- Android Studio (for Android development)
- Xcode (for iOS development, macOS only)
- Firebase CLI (for local emulator)
- Chrome DevTools
- Redux DevTools extension

## Browser Support

### Desktop Browsers
- Chrome/Edge (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (latest 2 versions)

### Mobile Browsers
- iOS Safari 15+
- Chrome for Android (latest)
- Samsung Internet (latest)

### PWA Support
- All modern browsers supporting Service Workers
- iOS 16.4+ for improved PWA features
- Android with Chrome 100+

## Version Management

### Package Updates
- Regular updates monthly for patch versions
- Quarterly updates for minor versions
- Major version updates planned and tested
- Security updates applied immediately

### Angular Update Strategy
- Use `ng update` for Angular migrations
- Test thoroughly in staging environment
- Follow Angular update guide
- Update Nx workspace with `nx migrate`

---

[Back to Index](./index.md) | [Next: Project Structure →](./project-structure.md)
