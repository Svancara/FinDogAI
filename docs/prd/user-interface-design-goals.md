# User Interface Design Goals

### Overall UX Vision

FinDogAI prioritizes a **voice-first, eyes-free interaction model** optimized for mobile craftsmen operating vehicles and working on-site. The UI serves as a visual confirmation layer and fallback for manual operations, not the primary interaction paradigm. Design emphasizes **large touch targets, minimal text entry, clear visual feedback for offline/sync status, and audio-centric workflows**. The application should feel like a conversational assistant that happens to have a visual interface, rather than a traditional form-heavy app with voice bolted on.

### Key Interaction Paradigms

1. **Voice-First with Dual Confirmation Modes:** Primary flow is speak → hear TTS readback → respond "yes"/"no" (hands-free) OR tap Accept/Retry/Cancel (fallback). Voice commands trigger immediate visual feedback showing parsed entities (job ID, amounts, odometer readings). TTS plays confirmation prompt: "Say 'yes' to confirm or 'no' to retry." Wake-word detection remains active during confirmation to capture voice responses.

2. **Active Job Context Banner:** Persistent visual indicator at top of every screen showing currently active job (number + title) with quick-tap to change. Banner is always visible (non-dismissible) to reinforce the mental model that all actions apply to the active job unless specified otherwise.

3. **Sequential ID Shortcuts:** All lists (jobs, costs, resources) display prominent numeric IDs as primary visual identifiers, supporting voice commands like "Set active job to 123" or "Delete cost 45".


4. **Inline Job Targeting & Ephemeral Overrides:** Voice commands may include a job override (e.g., "Material 200 for job 123"). The action applies to that job without changing the Active Job. TTS readback always includes the targeted job (e.g., "for job one two three, Smith, Brno"). Use "Set Active Job" to change context; inline targeting is per-operation only.

5. **Offline-First Status Visibility:** Persistent connectivity indicator with sync queue count. Offline mode is presented as normal operation with standard UI appearance (no special color theme)—no blocking warnings, just informative badges.

6. **Touch-Friendly Ergonomics:** Minimum 48dp touch targets, high contrast colors, avoid swipe gestures that require precision. Favor large buttons and voice over complex navigation hierarchies. Haptic feedback optional (user-toggle).

### Core Screens and Views

1. **Voice Command Hub (Home Screen):** Central microphone button (wake-word optional), active job display, recent activity feed, quick stats (costs today, sync status). Primary entry point for voice interactions.

2. **Jobs List:** Scrollable job cards showing `[jobNumber] Title - Status`. Financial summary (costs vs budget) visible to owner and representative; team member sees only active jobs and no financial summary. Sorted by status first, then by jobNumber ascending. Tap to view details, long-press for quick "Set Active" action.

3. **Job Detail View:** Tabbed interface: Costs (breakdown by category with sequential IDs), Advances (ordinalNumber + amount), Events timeline, PDF export action (Phase 2).

4. **Cost Entry Form (Manual Fallback):** Category-specific forms (Transport/Material/Labor/Machine/Other) with large number pads, vehicle/machine picker, voice dictation for descriptions. Pre-filled with active job context.

5. **Resources Management:** Three tabs (Team Members, Vehicles, Machines). List view with sequential IDs and key properties (rates). Owner can set the member role on the Team Member detail screen.

6. **Settings/Business Profile:** One-time setup wizard style for currency, VAT, distance unit, language preference, voice provider configuration (for developers).

7. **Voice Confirmation Modal:** Full-screen overlay during voice interaction showing real-time transcription, parsed entities in structured format, and Accept/Retry/Cancel actions. TTS readback plays confirmation prompt ending with "Say 'yes' to confirm or 'no' to retry." Wake-word detection remains active to capture voice responses ("yes" = Accept, "no" = Retry). Touch buttons remain enabled as fallback. Modal is non-dismissible during TTS playback to prevent accidental interruption.

### Accessibility: None

- No formal WCAG compliance target for MVP
- High contrast color ratios for readability
- Screen reader support for visual feedback during voice flows (announce "Job 123 set as active")
- TTS readback serves as primary accessibility feature for vision-impaired users
- Large text mode support (iOS/Android system settings)
- Voice commands provide alternative to all touch interactions for core workflows

### Branding

**Visual Identity:** Clean, utilitarian design avoiding "tech startup" aesthetics. Think rugged toolbox rather than sleek dashboard.

**Color Palette (Functional High-Contrast):**

| Purpose | Color | Hex Code | Usage |
|---------|-------|----------|--------|
| Primary Action | Safety Orange | `#FF6B35` | Main CTA buttons, microphone button, active states |
| Success/Sync | Forest Green | `#2D6A4F` | Sync success indicators, confirmation checkmarks |
| Warning/Attention | Amber | `#F4A261` | Validation warnings, pending sync queue badge |
| Error/Destructive | Signal Red | `#D62828` | Delete actions, error messages, failed operations |
| Background | Off-White | `#F8F9FA` | Main screen background (reduces eye strain vs pure white) |
| Surface/Cards | White | `#FFFFFF` | Cards, modals, input fields |
| Text Primary | Charcoal | `#212529` | Body text, headings (AAA contrast on off-white) |
| Text Secondary | Slate Gray | `#6C757D` | Secondary info, metadata, timestamps |
| Border/Divider | Light Gray | `#DEE2E6` | Borders, dividers, disabled states |
| Active Job Banner | Deep Blue | `#1D3557` | Background for persistent active job banner (high contrast with white text) |

**Typography:**
- Font Family: Roboto (Android), SF Pro (iOS), fallback to system sans-serif for web
- Body Text: 16px minimum (1rem)
- Headings: Bold weight, 20px-28px depending on hierarchy
- Job Numbers/Sequential IDs: Monospace variant (Roboto Mono) at 18px for scannability

**Iconography:** Material Design Icons (solid fill), minimum 24x24dp, not line art—better visibility in bright outdoor light.

**Tone:** Confident, direct, unpretentious. Messages like "Job 23 set. Ready to record costs." not "Great! You've successfully configured your active job context!"

### Target Device and Platforms: Web Responsive (Mobile-First)

- **Primary:** Mobile phones (iOS 15+, Android 10+) in portrait orientation—5.5" to 6.7" screens
- **Secondary:** Tablets (iPad, Android tablets) for office/desk use cases (viewing reports, bulk data entry)
- **Tertiary:** Desktop browsers (Chrome/Edge/Safari) for admin tasks, but not optimized—mobile-first design scales up
- **Native Builds:** Capacitor builds for iOS and Android with platform-specific permissions (microphone, filesystem) and optional OS integrations (Siri Shortcuts future consideration). Note: iOS builds require macOS/Xcode.
- **Network Conditions:** Optimized for 4G with offline-first, gracefully handles 3G degradation (slower voice API responses but fully functional)
