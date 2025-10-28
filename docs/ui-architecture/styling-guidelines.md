[Back to Index](./index.md)

# Styling Guidelines

## Theme System Configuration

The FinDogAI application uses Ionic CSS Custom Properties combined with SCSS for a flexible, maintainable theming system with dark mode support.

### Global Theme Variables

```scss
// apps/mobile-app/src/theme/variables.scss

:root {
  /* ===== Color Palette ===== */
  --ion-color-primary: #3880ff;
  --ion-color-primary-rgb: 56, 128, 255;
  --ion-color-primary-contrast: #ffffff;
  --ion-color-primary-contrast-rgb: 255, 255, 255;
  --ion-color-primary-shade: #3171e0;
  --ion-color-primary-tint: #4c8dff;

  --ion-color-secondary: #3dc2ff;
  --ion-color-secondary-rgb: 61, 194, 255;
  --ion-color-secondary-contrast: #ffffff;
  --ion-color-secondary-contrast-rgb: 255, 255, 255;
  --ion-color-secondary-shade: #36abe0;
  --ion-color-secondary-tint: #50c8ff;

  --ion-color-tertiary: #5260ff;
  --ion-color-tertiary-rgb: 82, 96, 255;
  --ion-color-tertiary-contrast: #ffffff;
  --ion-color-tertiary-contrast-rgb: 255, 255, 255;
  --ion-color-tertiary-shade: #4854e0;
  --ion-color-tertiary-tint: #6370ff;

  --ion-color-success: #2dd36f;
  --ion-color-success-rgb: 45, 211, 111;
  --ion-color-success-contrast: #ffffff;
  --ion-color-success-contrast-rgb: 255, 255, 255;
  --ion-color-success-shade: #28ba62;
  --ion-color-success-tint: #42d77d;

  --ion-color-warning: #ffc409;
  --ion-color-warning-rgb: 255, 196, 9;
  --ion-color-warning-contrast: #000000;
  --ion-color-warning-contrast-rgb: 0, 0, 0;
  --ion-color-warning-shade: #e0ac08;
  --ion-color-warning-tint: #ffca22;

  --ion-color-danger: #eb445a;
  --ion-color-danger-rgb: 235, 68, 90;
  --ion-color-danger-contrast: #ffffff;
  --ion-color-danger-contrast-rgb: 255, 255, 255;
  --ion-color-danger-shade: #cf3c4f;
  --ion-color-danger-tint: #ed576b;

  /* ===== App-Specific Colors ===== */
  --app-primary: var(--ion-color-primary);
  --app-primary-contrast: var(--ion-color-primary-contrast);
  --app-success: var(--ion-color-success);
  --app-warning: var(--ion-color-warning);
  --app-danger: var(--ion-color-danger);

  /* Voice UI Colors */
  --voice-idle-color: #92949c;
  --voice-listening-color: #3880ff;
  --voice-processing-color: #ffc409;
  --voice-speaking-color: #2dd36f;
  --voice-error-color: #eb445a;
  --voice-active-pulse: rgba(56, 128, 255, 0.3);
  --voice-recording-pulse: rgba(255, 196, 9, 0.3);

  /* ===== Typography ===== */
  --font-family-base: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --font-family-monospace: 'Courier New', Courier, monospace;

  /* Font Sizes */
  --font-size-xs: 0.75rem;   /* 12px */
  --font-size-sm: 0.875rem;  /* 14px */
  --font-size-base: 1rem;    /* 16px */
  --font-size-md: 1.125rem;  /* 18px */
  --font-size-lg: 1.25rem;   /* 20px */
  --font-size-xl: 1.5rem;    /* 24px */
  --font-size-2xl: 2rem;     /* 32px */

  /* Font Weights */
  --font-weight-light: 300;
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;

  /* Line Heights */
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;

  /* ===== Spacing System ===== */
  --spacing-xs: 0.25rem;  /* 4px */
  --spacing-sm: 0.5rem;   /* 8px */
  --spacing-md: 1rem;     /* 16px */
  --spacing-lg: 1.5rem;   /* 24px */
  --spacing-xl: 2rem;     /* 32px */
  --spacing-2xl: 3rem;    /* 48px */
  --spacing-3xl: 4rem;    /* 64px */

  /* ===== Touch Targets (Accessible) ===== */
  --touch-target-min: 48px;         /* Minimum touch target */
  --touch-target-comfortable: 56px;  /* Comfortable for field work */
  --touch-target-large: 64px;       /* Extra large for critical actions */

  /* ===== Border Radius ===== */
  --radius-none: 0;
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-xl: 1rem;
  --radius-full: 50%;

  /* ===== Shadows ===== */
  --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 20px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 40px rgba(0, 0, 0, 0.15);

  /* ===== Z-Index Scale ===== */
  --z-dropdown: 1000;
  --z-sticky: 1020;
  --z-fixed: 1030;
  --z-modal-backdrop: 1040;
  --z-modal: 1050;
  --z-popover: 1060;
  --z-tooltip: 1070;
  --z-toast: 1080;

  /* ===== Transitions ===== */
  --transition-fast: 150ms ease-in-out;
  --transition-base: 250ms ease-in-out;
  --transition-slow: 350ms ease-in-out;

  /* ===== Offline Mode ===== */
  --offline-bg: #fff7ed;
  --offline-border: #f59e0b;
  --offline-text: #92400e;
}

/* ===== Dark Mode ===== */
@media (prefers-color-scheme: dark) {
  :root {
    /* Dark mode color overrides */
    --ion-color-primary: #428cff;
    --ion-color-primary-rgb: 66, 140, 255;
    --ion-color-primary-contrast: #ffffff;
    --ion-color-primary-contrast-rgb: 255, 255, 255;
    --ion-color-primary-shade: #3a7be0;
    --ion-color-primary-tint: #5598ff;

    /* Background colors for dark mode */
    --ion-background-color: #121212;
    --ion-background-color-rgb: 18, 18, 18;
    --ion-text-color: #ffffff;
    --ion-text-color-rgb: 255, 255, 255;

    /* Card and surface colors */
    --ion-card-background: #1e1e1e;
    --ion-item-background: #1e1e1e;

    /* Dark mode shadows (lighter shadows) */
    --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.3);
    --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.3);
    --shadow-lg: 0 10px 20px rgba(0, 0, 0, 0.3);
    --shadow-xl: 0 20px 40px rgba(0, 0, 0, 0.3);

    /* Dark mode offline indicators */
    --offline-bg: #451a03;
    --offline-border: #f59e0b;
    --offline-text: #fef3c7;
  }
}

/* ===== Utility Classes ===== */
.touch-target {
  min-width: var(--touch-target-min);
  min-height: var(--touch-target-min);
}

.touch-target-large {
  min-width: var(--touch-target-large);
  min-height: var(--touch-target-large);
}

/* Voice UI States */
.voice-listening {
  --ion-color-base: var(--voice-active-color);
  animation: pulse 1.5s infinite;
}

.voice-recording {
  --ion-color-base: var(--voice-recording-pulse);
  animation: recording-pulse 1s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

@keyframes recording-pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

/* Offline Mode Styling */
.offline-banner {
  background: var(--offline-bg);
  border: 1px solid var(--offline-border);
  color: var(--offline-text);
  padding: var(--spacing-sm) var(--spacing-md);
  text-align: center;
  position: sticky;
  top: 0;
  z-index: var(--z-sticky);
}

/* PWA Install Prompt */
.pwa-install-prompt {
  position: fixed;
  bottom: var(--spacing-lg);
  left: var(--spacing-md);
  right: var(--spacing-md);
  background: var(--ion-color-primary);
  color: var(--ion-color-primary-contrast);
  padding: var(--spacing-md);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-lg);
  z-index: var(--z-toast);
}
```

## Component-Specific Styling

### BEM Methodology

Use BEM (Block Element Modifier) naming convention for component styles:

```scss
// Example: Job Card Component Styling
// apps/mobile-app/src/app/features/jobs/components/job-card/job-card.component.scss

:host {
  display: block;
}

.job-card {
  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: var(--spacing-md);

    &--active {
      background: var(--app-primary);
      color: var(--app-primary-contrast);
    }
  }

  &__title {
    font-size: var(--font-size-lg);
    font-weight: var(--font-weight-semibold);
    margin: 0;
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: var(--spacing-xs);

    &-indicator {
      width: 8px;
      height: 8px;
      border-radius: var(--radius-full);

      &--synced { background: var(--app-success); }
      &--pending { background: var(--app-warning); }
      &--error { background: var(--app-danger); }
    }
  }

  &__actions {
    display: flex;
    gap: var(--spacing-sm);
    padding: var(--spacing-md);

    ion-button {
      min-height: var(--touch-target-min);
      min-width: var(--touch-target-comfortable);
    }
  }
}

// Responsive adjustments
@media (max-width: 576px) {
  .job-card {
    &__actions {
      flex-direction: column;

      ion-button {
        width: 100%;
      }
    }
  }
}
```

## Styling Best Practices

### 1. CSS Custom Properties First
Use CSS custom properties for all dynamic values:

```scss
/* ✅ Good - Uses custom properties */
.button {
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  background: var(--app-primary);
}

/* ❌ Avoid - Hardcoded values */
.button {
  padding: 16px;
  border-radius: 8px;
  background: #3880ff;
}
```

### 2. Touch-Friendly Design

Ensure all interactive elements meet minimum touch targets:

```scss
.button {
  /* Minimum 48x48px for touch targets */
  min-width: var(--touch-target-min);
  min-height: var(--touch-target-min);

  /* Comfortable size for field work */
  &--comfortable {
    min-width: var(--touch-target-comfortable);
    min-height: var(--touch-target-comfortable);
  }

  /* Extra large for critical actions */
  &--critical {
    min-width: var(--touch-target-large);
    min-height: var(--touch-target-large);
  }
}
```

### 3. Responsive Design

Mobile-first approach with responsive breakpoints:

```scss
// Mobile first (default)
.container {
  padding: var(--spacing-sm);
}

// Tablet
@media (min-width: 768px) {
  .container {
    padding: var(--spacing-md);
  }
}

// Desktop
@media (min-width: 1024px) {
  .container {
    padding: var(--spacing-lg);
  }
}
```

### 4. Consistent Spacing

Use the spacing scale for margins and padding:

```scss
.card {
  margin-bottom: var(--spacing-md);
  padding: var(--spacing-md);

  &__header {
    margin-bottom: var(--spacing-sm);
  }

  &__footer {
    margin-top: var(--spacing-lg);
  }
}
```

### 5. Color Usage

Use semantic color variables:

```scss
.status {
  &--success {
    background: var(--app-success);
    color: var(--ion-color-success-contrast);
  }

  &--warning {
    background: var(--app-warning);
    color: var(--ion-color-warning-contrast);
  }

  &--error {
    background: var(--app-danger);
    color: var(--ion-color-danger-contrast);
  }
}
```

## Rationale for Styling Approach

### CSS Custom Properties
- **Runtime theming** without rebuilds
- **Dark mode switching** instant
- **Consistent spacing/sizing** across app
- **Easy white-labeling** for future clients

### Touch-Friendly Targets
- **Minimum 48px** targets (Material Design guideline)
- **Comfortable 56px** for primary actions
- **Large 64px** for critical voice/safety actions

### Voice UI Visual Feedback
- **Distinct colors** for listening/recording/processing
- **Animations** for active states
- **Clear visual hierarchy** for voice commands

### Offline Mode Indicators
- **Sticky positioning** for constant awareness
- **Distinct color scheme** from normal operations
- **Non-intrusive** but noticeable

## Accessibility Considerations

### 1. Color Contrast
Ensure WCAG AA compliance:

```scss
// Check contrast ratios
.text-on-primary {
  color: var(--ion-color-primary-contrast); // White on blue
}

.text-on-secondary {
  color: var(--ion-color-secondary-contrast); // Meets 4.5:1 ratio
}
```

### 2. Focus Indicators
Visible focus states for keyboard navigation:

```scss
.focusable {
  &:focus {
    outline: 2px solid var(--ion-color-primary);
    outline-offset: 2px;
  }

  &:focus:not(:focus-visible) {
    outline: none; // Remove for mouse users
  }
}
```

### 3. Screen Reader Support
Use visually hidden utility for screen readers:

```scss
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## Ionic-Specific Styling

### Using Ionic CSS Utilities

```html
<!-- Spacing utilities -->
<div class="ion-padding">Padded content</div>
<div class="ion-margin-top">Top margin</div>

<!-- Text utilities -->
<p class="ion-text-center">Centered text</p>
<p class="ion-text-uppercase">Uppercase</p>

<!-- Color utilities -->
<ion-badge color="success">Success</ion-badge>
<ion-button color="primary">Primary</ion-button>
```

### Custom Ionic Colors

```scss
// Add custom color to Ionic palette
ion-button.custom-color {
  --background: #ff6b6b;
  --background-hover: #ff5252;
  --background-activated: #ff3838;
  --color: #ffffff;
}
```

---

[← Previous: Routing](./routing.md) | [Back to Index](./index.md) | [Next: Offline Architecture →](./offline-architecture.md)
