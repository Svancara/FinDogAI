---
type: "always_apply"
---

# NgRx Usage Rules

> **SCOPE**: Apply these rules when developing Angular applications with state management needs.

**Description**: Mandatory usage of NgRx Store, Effects, and Entity patterns for all state management in Angular applications to ensure predictable, scalable, and maintainable state architecture.

## Mandatory NgRx Components

- **Store**: ALL application state MUST be managed through NgRx Store
- **Effects**: ALL side effects (HTTP calls, async operations, external interactions) MUST use NgRx Effects
- **Actions**: ALL state changes MUST be triggered through dispatched actions
- **Reducers**: ALL state mutations MUST be handled through pure reducer functions
- **Selectors**: ALL state access MUST use memoized selectors

## State Architecture Requirements

- **No Component State**: Components MUST NOT manage local state for business data
- **No Service State**: Services MUST NOT hold stateful data - use Store instead
- **Centralized State**: ALL application state MUST be centralized in the NgRx Store
- **Immutable Updates**: ALL state updates MUST be immutable using NgRx patterns

## Required NgRx Patterns

- **Entity State**: Use `@ngrx/entity` for collections and normalized data
- **Feature States**: Organize state by feature modules using `createFeature()`
- **Typed Actions**: Use `createAction()` with typed payloads
- **Typed Selectors**: Use `createSelector()` with proper typing
- **Effect Patterns**: Use `createEffect()` for all async operations

## Implementation Standards

- **Store Configuration**: Configure store with devtools in development
- **Effect Error Handling**: ALL effects MUST handle errors gracefully
- **Action Naming**: Use consistent action naming: `[Feature] Action Description`
- **State Shape**: Design normalized, flat state structures
- **Selector Composition**: Compose complex selectors from simpler ones

## Prohibited Patterns

- **Direct HTTP in Components**: Components MUST NOT make direct HTTP calls
- **Shared Mutable State**: NEVER share mutable objects between components
- **Manual Subscriptions**: Avoid manual subscriptions - use async pipe and selectors
- **State Mutations**: NEVER mutate state directly - always dispatch actions

## Required Setup

```typescript
// App configuration MUST include:
provideStore({ /* feature reducers */ }),
provideEffects([/* effect classes */]),
provideStoreDevtools({ maxAge: 25, logOnly: !isDevMode() })
```

## Exception Policy

- **No Exceptions**: NgRx usage is mandatory for ALL state management
- **Authorization Override**: Only skip NgRx if human explicitly states: "I AUTHORIZE YOU TO SKIP NGRX THIS TIME"
- **Simple UI State**: Temporary UI state (form validation, loading spinners) may use component state if it doesn't affect business logic