[Back to Index](./index.md)

# Project Structure

## Monorepo Organization

The FinDogAI project uses Nx for monorepo management, providing a well-structured codebase with shared libraries and clear separation of concerns.

## Directory Structure

```plaintext
findogai/                                 # Monorepo root
├── apps/
│   ├── mobile-app/                      # Main Angular/Ionic PWA application
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/               # Singleton services, guards, interceptors
│   │   │   │   │   ├── services/
│   │   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   │   ├── firestore.service.ts
│   │   │   │   │   │   └── voice-pipeline.service.ts
│   │   │   │   │   ├── guards/
│   │   │   │   │   ├── interceptors/
│   │   │   │   │   └── models/
│   │   │   │   ├── features/           # Lazy-loaded feature modules
│   │   │   │   │   ├── jobs/
│   │   │   │   │   │   ├── data-access/    # NgRx state for jobs
│   │   │   │   │   │   ├── pages/
│   │   │   │   │   │   └── components/
│   │   │   │   │   ├── voice/
│   │   │   │   │   │   ├── data-access/    # NgRx state for voice
│   │   │   │   │   │   ├── pages/
│   │   │   │   │   │   └── components/
│   │   │   │   │   ├── costs/
│   │   │   │   │   ├── team/
│   │   │   │   │   └── settings/
│   │   │   │   ├── shared/             # Shared components, directives, pipes
│   │   │   │   │   ├── ui/
│   │   │   │   │   ├── directives/
│   │   │   │   │   └── pipes/
│   │   │   │   └── store/              # Root NgRx store configuration
│   │   │   │       ├── app.state.ts
│   │   │   │       └── app.effects.ts
│   │   │   ├── assets/
│   │   │   ├── environments/
│   │   │   ├── theme/                  # Ionic theme variables
│   │   │   ├── index.html
│   │   │   ├── main.ts
│   │   │   ├── manifest.json           # PWA manifest
│   │   │   └── ngsw-config.json        # Angular service worker config
│   │   ├── capacitor.config.ts
│   │   ├── ionic.config.json
│   │   └── project.json                # Nx project configuration
│   │
│   └── mobile-app-e2e/                 # Playwright E2E tests
│       ├── src/
│       │   ├── fixtures/
│       │   ├── support/
│       │   └── specs/
│       └── project.json
│
├── libs/                                # Shared libraries (Nx convention)
│   ├── shared/
│   │   ├── types/                      # TypeScript interfaces/types
│   │   │   └── src/
│   │   │       ├── lib/
│   │   │       │   ├── models/
│   │   │       │   ├── api/
│   │   │       │   └── firestore/
│   │   │       └── index.ts
│   │   └── utils/                      # Shared utilities
│   │       └── src/
│   │           └── lib/
│   │
│   └── mobile/                         # Mobile-specific shared code
│       ├── ui/                         # Shared Ionic components
│       ├── data-access/                # Shared state/services
│       └── voice/                      # Voice pipeline utilities
│
├── tools/                               # Nx workspace tools
├── .bmad-core/                          # BMAD configuration
├── docs/
│   ├── prd/
│   ├── ui-architecture/                # This documentation
│   └── architecture/
├── nx.json                             # Nx configuration
├── package.json
├── tsconfig.base.json                  # Base TypeScript config
└── angular.json                        # Angular workspace config
```

## Directory Responsibilities

### `/apps/mobile-app`
Main application containing the Angular/Ionic PWA.

#### `/app/core`
**Purpose**: Singleton services, guards, and core application logic.

**Contents:**
- **services/**: Authentication, Firestore, Voice Pipeline, Network Status
- **guards/**: Route protection (Auth, Tenant, Role, PendingChanges)
- **interceptors/**: HTTP interceptors (Auth, Offline)
- **models/**: Core domain models and interfaces

**Rules:**
- Services are provided in root (`providedIn: 'root'`)
- Only imported once in the app
- No feature-specific logic

#### `/app/features`
**Purpose**: Lazy-loaded feature modules organized by domain.

**Structure Pattern (per feature):**
```plaintext
feature-name/
├── data-access/           # NgRx state management
│   ├── feature.state.ts   # State interface
│   ├── feature.actions.ts # Action creators
│   ├── feature.reducer.ts # Reducers
│   ├── feature.effects.ts # Side effects
│   └── feature.selectors.ts # Selectors
├── pages/                 # Routable page components
│   ├── feature-list/
│   ├── feature-detail/
│   └── feature-edit/
└── components/            # Presentational components
    ├── feature-card/
    ├── feature-form/
    └── feature-item/
```

**Features:**
- **jobs/**: Job management (list, detail, create, edit)
- **voice/**: Voice command interface and history
- **costs/**: Cost tracking and categorization
- **team/**: Team member management
- **settings/**: Application settings and preferences

#### `/app/shared`
**Purpose**: Reusable components, directives, and pipes used across features.

**Contents:**
- **ui/**: Generic UI components (loading spinners, empty states, etc.)
- **directives/**: Custom directives (highlight, auto-focus, etc.)
- **pipes/**: Custom pipes (date formatting, currency, etc.)

**Rules:**
- No feature-specific logic
- No state management
- Pure presentational components

#### `/app/store`
**Purpose**: Root-level state configuration and app-wide effects.

**Contents:**
- `app.state.ts`: Root state interface
- `app.effects.ts`: App-wide effects (auth, sync, etc.)
- Store configuration and meta-reducers

### `/libs/shared/types`
**Purpose**: Shared TypeScript interfaces and types used across the application.

**Organization:**
```plaintext
types/
├── models/          # Domain models (Job, Cost, User, etc.)
├── api/             # API request/response types
└── firestore/       # Firestore document interfaces
```

**Usage:**
```typescript
import { Job, Cost } from '@shared/types';
```

### `/libs/shared/utils`
**Purpose**: Shared utility functions and helpers.

**Examples:**
- Date formatting utilities
- Currency conversion
- String manipulation
- Validation helpers

### `/libs/mobile/ui`
**Purpose**: Shared Ionic components specific to mobile app.

**Examples:**
- Custom Ionic wrappers
- Composite components
- Layout components

### `/libs/mobile/data-access`
**Purpose**: Shared services and state management for mobile features.

**Examples:**
- Network service
- Sync queue service
- Firestore sync service

### `/libs/mobile/voice`
**Purpose**: Voice pipeline utilities and services.

**Examples:**
- Speech recognition wrapper
- TTS service
- Voice command parser

## Naming Conventions

### Files

| Type | Pattern | Example |
|------|---------|---------|
| Component | `name.component.ts` | `job-detail.component.ts` |
| Service | `name.service.ts` | `auth.service.ts` |
| Guard | `name.guard.ts` | `auth.guard.ts` |
| Interceptor | `name.interceptor.ts` | `auth.interceptor.ts` |
| Pipe | `name.pipe.ts` | `format-date.pipe.ts` |
| Directive | `name.directive.ts` | `highlight.directive.ts` |
| Model | `name.model.ts` | `job.model.ts` |
| Interface | `name.interface.ts` | `user.interface.ts` |
| Actions | `name.actions.ts` | `jobs.actions.ts` |
| Reducer | `name.reducer.ts` | `jobs.reducer.ts` |
| Effects | `name.effects.ts` | `jobs.effects.ts` |
| Selectors | `name.selectors.ts` | `jobs.selectors.ts` |
| State | `name.state.ts` | `jobs.state.ts` |

### Classes and Interfaces

```typescript
// Components
export class JobDetailComponent { }

// Services
export class AuthService { }

// Guards
export class AuthGuard { }

// Pipes
export class FormatDatePipe { }

// Directives
export class HighlightDirective { }

// Interfaces
export interface Job { }
export interface UserProfile { }

// Types
export type JobStatus = 'draft' | 'active' | 'completed' | 'cancelled';
```

### Selectors and CSS Classes

```scss
// Component selector (kebab-case)
@Component({
  selector: 'app-job-detail'
})

// CSS classes (BEM notation)
.job-card { }
.job-card__header { }
.job-card__title { }
.job-card--active { }
```

## Import Aliases

Configure path aliases in `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@app/*": ["apps/mobile-app/src/app/*"],
      "@core/*": ["apps/mobile-app/src/app/core/*"],
      "@features/*": ["apps/mobile-app/src/app/features/*"],
      "@shared/*": ["apps/mobile-app/src/app/shared/*"],
      "@environments/*": ["apps/mobile-app/src/environments/*"],
      "@libs/shared/types": ["libs/shared/types/src/index.ts"],
      "@libs/shared/utils": ["libs/shared/utils/src/index.ts"],
      "@libs/mobile/ui": ["libs/mobile/ui/src/index.ts"],
      "@libs/mobile/data-access": ["libs/mobile/data-access/src/index.ts"]
    }
  }
}
```

**Usage:**
```typescript
import { AuthService } from '@core/services/auth.service';
import { Job, Cost } from '@libs/shared/types';
import { JobsState } from '@features/jobs/data-access/jobs.state';
import { environment } from '@environments/environment';
```

## Module Organization

### Standalone Components
All components are standalone (Angular 20+ best practice):

```typescript
@Component({
  selector: 'app-job-detail',
  standalone: true,
  imports: [CommonModule, IonicModule, ReactiveFormsModule],
  templateUrl: './job-detail.component.html'
})
export class JobDetailComponent { }
```

### Feature Routes
Each feature exports its routes:

```typescript
// features/jobs/jobs.routes.ts
export const JOBS_ROUTES: Routes = [
  { path: '', component: JobsListPage },
  { path: ':id', component: JobDetailPage }
];
```

## Code Organization Best Practices

1. **Single Responsibility**: Each file has one clear purpose
2. **Feature-First**: Organize by feature, not by file type
3. **Colocation**: Keep related files together
4. **Shallow Structure**: Avoid deep nesting (max 4 levels)
5. **Lazy Loading**: Feature modules loaded on demand
6. **Shared Code**: Common code in shared libraries
7. **Type Safety**: Strong typing throughout the codebase

## File Size Guidelines

- **Components**: < 300 lines (split if larger)
- **Services**: < 400 lines (split by responsibility)
- **Effects**: < 200 lines (one effect class per feature)
- **Reducers**: < 300 lines (use entity adapters)
- **Selectors**: < 150 lines (memoized selectors)

---

[← Previous: Tech Stack](./tech-stack.md) | [Back to Index](./index.md) | [Next: Component Standards →](./component-standards.md)
