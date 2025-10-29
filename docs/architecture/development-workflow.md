[Back to Index](./index.md) | [Previous: Project Structure](./project-structure.md) | [Next: Deployment](./deployment.md)

# Development Workflow

## Local Development Setup

### Prerequisites
```bash
# Required tools
node --version  # v20.x LTS
npm --version   # v10.x
firebase --version  # v13.x
nx --version    # v19.x

# Install Firebase CLI
npm install -g firebase-tools

# Install Nx globally (optional)
npm install -g nx
```

### Initial Setup
```bash
# Clone repository
git clone https://github.com/yourorg/findogai.git
cd findogai

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
cp apps/functions/.env.example apps/functions/.env

# Initialize Firebase (select existing project)
firebase use findogai-dev

# Enable Firestore offline persistence (runs automatically)
nx run mobile-app:setup-offline
```

### Development Commands
```bash
# Start all services (frontend + emulators)
npm run dev

# Start frontend only
nx serve mobile-app

# Start backend emulators only
npm run emulators

# Run specific package
nx serve mobile-app
nx serve functions

# Run tests
nx test mobile-app
nx test functions
nx test shared-types

# Build for production
nx build mobile-app --prod
nx build functions --prod
```

## Environment Configuration

### Required Environment Variables
```bash
# Frontend (.env.local)
VITE_FIREBASE_API_KEY=xxx
VITE_FIREBASE_AUTH_DOMAIN=xxx
VITE_FIREBASE_PROJECT_ID=findogai-prod
VITE_FIREBASE_STORAGE_BUCKET=xxx
VITE_FIREBASE_MESSAGING_SENDER_ID=xxx
VITE_FIREBASE_APP_ID=xxx
VITE_OPENAI_API_KEY=xxx  # For voice processing

# Backend (.env)
OPENAI_API_KEY=xxx
ANTHROPIC_API_KEY=xxx
SENTRY_DSN=xxx

# Shared
FIREBASE_PROJECT_ID=findogai-prod
FIREBASE_REGION=europe-west1
```

## Development Best Practices

### Code Organization
- **Feature-based structure**: Organize code by feature, not by file type
- **Lazy loading**: Use Angular lazy-loaded modules for better performance
- **Shared packages**: Put reusable code in shared packages, not apps

### Testing Strategy
- **Unit tests**: Test individual functions and components in isolation
- **Integration tests**: Test service interactions with Firestore emulators
- **E2E tests**: Test critical user flows with Playwright

### Git Workflow
```bash
# Create feature branch
git checkout -b feature/job-management

# Make changes and commit
git add .
git commit -m "feat: add job creation form"

# Push to remote
git push origin feature/job-management

# Create pull request on GitHub
```

### Branch Naming Convention
- **feature/**: New features (`feature/voice-commands`)
- **fix/**: Bug fixes (`fix/offline-sync-error`)
- **docs/**: Documentation updates (`docs/api-specification`)
- **refactor/**: Code refactoring (`refactor/job-service`)
- **test/**: Test additions/updates (`test/job-creation`)

## Local Emulator Usage

### Firebase Emulators Suite
```bash
# Start all emulators
firebase emulators:start

# Start specific emulators
firebase emulators:start --only firestore,auth,functions

# Export emulator data
firebase emulators:export ./emulator-data

# Import emulator data
firebase emulators:start --import=./emulator-data
```

### Emulator Ports
- **Firestore**: http://localhost:8080
- **Auth**: http://localhost:9099
- **Functions**: http://localhost:5001
- **Hosting**: http://localhost:5000
- **Emulator UI**: http://localhost:4000

### Testing with Emulators
```typescript
// Configure app to use emulators in development
if (environment.useEmulators) {
  connectFirestoreEmulator(firestore, 'localhost', 8080);
  connectAuthEmulator(auth, 'http://localhost:9099');
  connectFunctionsEmulator(functions, 'localhost', 5001);
}
```

## Debugging

### Frontend Debugging
```bash
# Serve with source maps
nx serve mobile-app --source-map

# Run in Chrome DevTools
# Open chrome://inspect and select the application
```

### Backend Debugging
```bash
# Debug Cloud Functions locally
firebase emulators:start --inspect-functions

# Attach debugger in VS Code
# Use "Attach to Node Process" configuration
```

### VS Code Debug Configuration
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Functions",
      "port": 9229,
      "restart": true
    }
  ]
}
```

## Code Quality Tools

### Linting
```bash
# Lint all projects
nx run-many --target=lint --all

# Lint specific project
nx lint mobile-app

# Auto-fix linting errors
nx lint mobile-app --fix
```

### Formatting
```bash
# Format all files
npx prettier --write .

# Check formatting
npx prettier --check .
```

### Type Checking
```bash
# Check types across all projects
nx run-many --target=typecheck --all
```

## Monorepo Commands

### Nx Commands
```bash
# Run target for all projects
nx run-many --target=build --all

# Run target for affected projects only
nx affected --target=test

# Show dependency graph
nx graph

# Clear Nx cache
nx reset
```

### Workspace Management
```bash
# Add new library
nx g @nx/angular:library my-lib

# Add new application
nx g @nx/angular:application my-app

# Generate component
nx g @nx/angular:component my-component --project=mobile-app
```

## Common Development Tasks

### Adding a New Feature
1. Create feature branch
2. Generate feature module: `nx g @nx/angular:module features/my-feature --project=mobile-app`
3. Add components, services, and state management
4. Write tests
5. Update documentation
6. Create pull request

### Adding a New Cloud Function
1. Create function file in `apps/functions/src/`
2. Add types to `packages/shared-types`
3. Write unit tests
4. Test with emulators
5. Deploy to staging

### Updating Data Models
1. Update interface in `packages/shared-types`
2. Update Firestore security rules
3. Add migration function if needed
4. Update affected services
5. Run tests to ensure compatibility

## Performance Optimization

### Frontend Optimization
- Use OnPush change detection strategy
- Implement virtual scrolling for long lists
- Lazy load images and modules
- Use trackBy functions in ngFor loops
- Minimize bundle size with tree shaking

### Backend Optimization
- Set appropriate function memory limits
- Use connection pooling for Firestore
- Implement caching strategies
- Optimize cold start times
- Monitor function execution times

## Troubleshooting

### Common Issues

**Firestore Offline Persistence Conflicts**
```bash
# Clear IndexedDB cache
# Open DevTools > Application > Storage > Clear site data
```

**Function Deployment Failures**
```bash
# Check function logs
firebase functions:log

# Verify environment variables
firebase functions:config:get
```

**Build Errors**
```bash
# Clear Nx cache and rebuild
nx reset
npm run build
```

**Dependency Conflicts**
```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install
```

---

*Next: [Deployment](./deployment.md)*
