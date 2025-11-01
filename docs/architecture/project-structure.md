[Back to Index](./index.md) | [Previous: Components](./components.md) | [Next: Development Workflow](./development-workflow.md)

# Unified Project Structure

```
findogai/
├── .github/                         # CI/CD workflows
│   └── workflows/
│       ├── ci.yaml                  # Continuous integration
│       └── deploy.yaml               # Deploy to environments
├── apps/                            # Application packages
│   ├── mobile-app/                  # Angular/Ionic PWA
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/           # Singleton services
│   │   │   │   ├── features/       # Feature modules
│   │   │   │   │   ├── jobs/
│   │   │   │   │   ├── costs/
│   │   │   │   │   ├── resources/
│   │   │   │   │   └── voice/
│   │   │   │   ├── shared/         # Shared components
│   │   │   │   └── state/          # NgRx store
│   │   │   ├── assets/             # Static assets
│   │   │   │   └── i18n/           # Translation files (en.json, cs.json)
│   │   │   ├── environments/       # Environment configs
│   │   │   └── theme/              # Ionic theming
│   │   ├── capacitor.config.ts     # Native app config
│   │   ├── ionic.config.json
│   │   └── package.json
│   └── functions/                   # Cloud Functions
│       ├── src/
│       │   ├── triggers/           # Firestore triggers
│       │   ├── callables/          # HTTPS callable functions
│       │   ├── scheduled/          # Cron jobs
│       │   ├── services/           # Business logic
│       │   └── utils/              # Utilities
│       ├── .env.example
│       └── package.json
├── packages/                        # Shared packages
│   ├── shared-types/               # TypeScript definitions
│   │   ├── src/
│   │   │   ├── entities/          # Data models
│   │   │   ├── api/               # API interfaces
│   │   │   └── constants/         # Shared constants
│   │   └── package.json
│   └── validation/                 # Shared validation
│       ├── src/
│       │   ├── schemas/           # Validation schemas
│       │   └── validators/        # Validation functions
│       └── package.json
├── infrastructure/                 # IaC definitions
│   ├── firebase/
│   │   ├── firestore.rules        # Security rules
│   │   ├── firestore.indexes.json # Database indexes
│   │   └── storage.rules          # Storage security
│   └── terraform/                  # Additional GCP resources
│       ├── main.tf
│       └── variables.tf
├── scripts/                        # Build and utility scripts
│   ├── deploy.sh
│   └── setup-local.sh
├── docs/                           # Documentation
│   ├── architecture.md            # This document
│   ├── prd/                       # Sharded PRD
│   ├── backend-architecture/      # Backend details
│   └── ui-architecture/           # Frontend details
├── .env.example                    # Environment template
├── nx.json                         # Nx configuration
├── package.json                    # Root package.json
├── firebase.json                   # Firebase configuration
└── README.md                       # Project overview
```

## Directory Structure Details

### `/apps/mobile-app/`
The main Progressive Web Application built with Angular and Ionic. Contains all frontend code including:
- **Core**: Singleton services (auth, config, error handling, language service)
- **Features**: Feature modules organized by domain (jobs, costs, resources, voice)
- **Shared**: Reusable UI components and utilities (including language selector)
- **State**: NgRx store configuration and state management
- **Assets/i18n**: Translation JSON files (en.json, cs.json) for multi-language support
- **Environments**: Environment-specific configuration files

### `/apps/functions/`
Firebase Cloud Functions for server-side processing:
- **Triggers**: Firestore event handlers (onCreate, onUpdate, onDelete)
- **Callables**: HTTPS callable functions (allocateSequence, generatePDF)
- **Scheduled**: Cron jobs (cleanup, reporting)
- **Services**: Business logic shared across functions
- **Utils**: Utility functions and helpers

### `/packages/shared-types/`
Type definitions shared across frontend and backend:
- **Entities**: Data model interfaces (Job, Cost, Member, etc.)
- **API**: Request/response types for Cloud Functions
- **Constants**: Shared enums and constants

### `/packages/validation/`
Validation logic used by both frontend and backend:
- **Schemas**: Zod/Joi validation schemas
- **Validators**: Custom validation functions

### `/infrastructure/`
Infrastructure as Code definitions:
- **Firebase**: Security rules and database indexes
- **Terraform**: GCP resource configuration (if needed beyond Firebase)

### `/scripts/`
Build and deployment automation scripts:
- **deploy.sh**: Deployment automation
- **setup-local.sh**: Local environment setup

### `/docs/`
Project documentation:
- **architecture/**: This sharded architecture documentation
- **prd/**: Product Requirements Documentation
- **backend-architecture/**: Detailed backend specifications
- **ui-architecture/**: Detailed frontend specifications

## Key Configuration Files

### Root Level
- **nx.json**: Nx monorepo configuration and build caching
- **package.json**: Root dependencies and workspace scripts
- **firebase.json**: Firebase project configuration
- **.env.example**: Template for environment variables

### Mobile App
- **capacitor.config.ts**: Native app configuration for iOS/Android
- **ionic.config.json**: Ionic CLI configuration
- **angular.json**: Angular CLI configuration

### Functions
- **.env.example**: Template for Cloud Functions environment variables
- **package.json**: Function-specific dependencies

## Dependency Graph

```
shared-types (no dependencies)
    ↓
validation (depends on: shared-types)
    ↓
mobile-app (depends on: shared-types, validation)
functions (depends on: shared-types, validation)
    ↓
e2e-tests (depends on: all)
```

## Build Order

1. **shared-types**: Built first (no dependencies)
2. **validation**: Built second (depends on shared-types)
3. **mobile-app & functions**: Built in parallel (both depend on shared-types and validation)
4. **e2e-tests**: Built last (test-only package)

## File Naming Conventions

- **Components**: `feature-name.component.ts`
- **Services**: `feature-name.service.ts`
- **Models**: `entity-name.model.ts`
- **Interfaces**: `interface-name.interface.ts`
- **Cloud Functions**: `function-name.function.ts`
- **Tests**: `*.spec.ts` (unit), `*.e2e.ts` (end-to-end)

## Module Organization

### Frontend Feature Module Structure
```
features/jobs/
├── components/
│   ├── job-list/
│   ├── job-detail/
│   └── job-form/
├── services/
│   └── jobs.service.ts
├── state/
│   ├── jobs.actions.ts
│   ├── jobs.effects.ts
│   ├── jobs.reducer.ts
│   └── jobs.selectors.ts
├── models/
│   └── job-view.model.ts
└── jobs.routes.ts
```

### Backend Function Organization
```
functions/src/triggers/
├── jobs/
│   ├── on-job-create.ts
│   ├── on-job-update.ts
│   └── on-job-delete.ts
├── costs/
│   └── on-cost-write.ts
└── index.ts (exports all triggers)
```

---

*Next: [Development Workflow](./development-workflow.md)*
