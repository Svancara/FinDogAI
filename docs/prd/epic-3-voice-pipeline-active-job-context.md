# Epic 3: Voice Pipeline & Active Job Context

**Expanded Goal:** Build the complete voice infrastructure (STT → LLM NLU → TTS confirmation pipeline) with configurable providers, implement offline voice mock mode, and deliver the "Set Active Job" voice flow with full confirmation loop. This epic establishes the foundational voice-first interaction pattern that all subsequent voice features will follow, proving the core technical value proposition of hands-free operation.

### Story 3.1: Voice Service Architecture & Provider Configuration

**As a** developer,
**I want** a configurable voice service layer that supports multiple STT/LLM/TTS providers,
**so that** the app can switch providers for cost optimization and testing without code changes.

**Acceptance Criteria:**

1. Voice service module created in `/packages/mobile-app/src/app/services/voice/`
2. Environment configuration file supports provider endpoints: `STT_PROVIDER` (google-cloud | openai-whisper | mock), `LLM_PROVIDER` (openai-gpt4 | groq-llama3 | ollama | mock), `TTS_PROVIDER` (google-cloud | platform-native | mock)
3. Voice service implements three interfaces: `ISpeechToText`, `IIntentRecognition`, `ITextToSpeech`
4. Each interface has multiple implementations (e.g., GoogleCloudSTT, OpenAIWhisperSTT, MockSTT)
5. Provider factory pattern selects implementation based on environment config
6. Mock providers return hardcoded responses for offline development (no API calls)
7. API credentials loaded from `.env` file (GOOGLE_CLOUD_API_KEY, OPENAI_API_KEY, etc.)
8. Voice service handles errors gracefully: network failures, API quota exceeded, invalid credentials
9. Voice service logs provider selection and latency metrics for debugging


10. Unit tests validate each provider implementation independently
11. Integration test validates provider switching via config change (no code recompile needed)
12. README documents environment variable configuration for all providers
13. Voice service gates operations by `personProfile.aiSupportEnabled` and connectivity; exposes status: ENABLED, DISABLED_BY_USER, OFFLINE
14. In DISABLED_BY_USER or OFFLINE status, voice service does not call STT/LLM/TTS providers and returns UNAVAILABLE for voice operations (no audio recording or queuing)

### Story 3.2: Microphone Access & Audio Recording

**As a** user,
**I want** to trigger voice recording via button press (push-to-talk),
**so that** I can capture voice commands hands-free while driving or working.

**Acceptance Criteria:**

1. Voice Command Hub (home screen) displays large circular microphone button (Safety Orange #FF6B35)
2. Button shows microphone icon (Material Design solid fill) with label "Tap to Speak"
3. Tap button requests microphone permission (iOS/Android via Capacitor)
4. If permission denied, show error: "Microphone access required. Enable in Settings."
5. Once permission granted, tap starts audio recording (visual feedback: button pulses, label changes to "Listening...")
6. Audio captured via Web Audio API (browser) or Capacitor Audio plugin (native)
7. Recording stops automatically after 10 seconds of silence OR user taps button again
8. Manual stop: Tap button again, label changes to "Processing...", recording ends
9. Recorded audio converted to format required by STT provider (WAV, FLAC, or OGG)
10. While recording, display real-time audio level indicator (waveform or volume bars)
11. Offline (production): Voice interactions are disabled; no audio is recorded or queued; mic tap plays platform-native TTS to announce unavailability
12. Error handling: "Recording failed. Check microphone connection."

13. If personProfile.aiSupportEnabled is false, tapping microphone plays platform-native TTS: "AI support is disabled. Use manual entry." No STT/LLM/TTS requests are made; header shows "AI Disabled" badge
14. If offline (production) with aiSupportEnabled=true, tapping microphone plays platform-native TTS: "Offline: AI support unavailable." No STT/LLM/TTS requests are made; no audio is recorded or queued; header shows "Offline · AI Unavailable"
15. Header text format uses middle dot (U+00B7): "Offline · AI Unavailable" (exact string)
### Story 3.3: Speech-to-Text Integration

**As a** developer,
**I want** recorded audio transcribed to text via STT provider,
**so that** voice commands can be parsed into structured intents.

**Acceptance Criteria:**

1. Voice service sends recorded audio to configured STT provider (Google Cloud Speech-to-Text primary)
2. STT request includes language parameter (cs-CZ or en-US based on personProfile.language)
3. STT request configured for short utterances (single phrase mode, not streaming for MVP)
4. Response returns transcribed text: e.g., "Set active job to 123"
5. Transcription displayed in real-time on Voice Confirmation Modal as "Heard: [transcription]"
6. If STT fails (network error, API error), display: "Could not transcribe audio. Try again."
7. Retry button allows user to re-record without dismissing modal
8. STT latency tracked: Target ≤3.0 seconds median and ≤5.0 seconds P95 from recording end to transcription display
9. Mock mode returns predefined transcription instantly (e.g., "set active job to one two three")
10. Czech language transcription tested with sample phrases: "Nastav aktivní práci na 123"
11. Empty transcription (silence detected) shows: "No speech detected. Please try again."
12. Transcription text passed to Intent Recognition service (Story 3.4)

13. STT quality evaluated on curated domain test set (≥100 phrases per language) → report WER (median, P95) for cs-CZ and en-US
14. Numeric transcription accuracy: digits/amounts exact-match ≥90% in controlled conditions; log misrecognitions with ground truth for analysis
### Story 3.4: Intent Recognition & Slot Filling with LLM

**As a** developer,
**I want** transcribed text parsed into structured intents and entities via LLM,
**so that** voice commands can be executed as app actions.

**Acceptance Criteria:**

1. Intent Recognition service sends transcription to configured LLM provider with structured prompt
2. Prompt template: "Extract intent and entities from: '{transcription}'. Intents: SetActiveJob. Return JSON: {intent, entities}."
3. For "Set active job to 123" (or Czech equivalent), LLM returns: `{intent: "SetActiveJob", entities: {jobIdentifier: "123"}}`
4. jobIdentifier can be: numeric jobNumber ("123"), partial job title ("Smith"), or full title ("Smith, Brno")
5. If jobIdentifier is numeric, query Firestore for job with matching jobNumber
6. If jobIdentifier is text, query Firestore for job with title containing text (case-insensitive, partial match)
7. If multiple jobs match (by partial title), require disambiguation: TTS lists top 2–3 candidates or prompt for job number; only auto-select when an exact numeric jobNumber is provided.
8. If no job matches, return error: "Job not found. Please say job number or title."
9. Intent recognition latency target: <1 second for LLM response
10. Mock mode returns hardcoded intent: `{intent: "SetActiveJob", entities: {jobIdentifier: "123"}}`
13. Intent quality evaluated on curated domain test set → report precision, recall, and F1 per intent; F1 ≥0.85 for SetActiveJob and StartJourney
14. Numeric entity extraction (jobNumber, odometer) exact-match ≥90% in controlled conditions; log confusion cases for analysis
15. Unrecognized intent returns: "I didn't understand. Try: 'Set active job to [number or name]'"
16. Parsed intent and matched job data passed to Confirmation Loop (Story 3.5)

### Story 3.5: Text-to-Speech Confirmation Loop

**As a** user,
**I want** to hear the app read back my parsed voice command via TTS,
**so that** I can confirm accuracy before data is saved.

**Acceptance Criteria:**

1. Voice Confirmation Modal displays parsed intent in structured format: "✓ Job: **123 - Smith, Brno**"
2. TTS service generates audio from confirmation text: "Setting active job to one two three, Smith, Brno. Say 'yes' to confirm or 'no' to retry."
3. TTS uses configured provider (Google Cloud Text-to-Speech primary) with language from personProfile
4. Czech TTS voice: cs-CZ-Wavenet-A (female) or cs-CZ-Wavenet-B (male) - user preference in future
5. TTS audio plays automatically when modal displays parsed result
6. After TTS completes, wake-word detection (KWS) remains active to listen for "yes" or "no" voice responses
7. Voice response "yes" (or Czech "ano") → triggers Accept action (same as tapping Accept button)
8. Voice response "no" (or Czech "ne") → triggers Retry action (same as tapping Retry button)
9. Touch buttons (Accept/Retry/Cancel) remain enabled throughout as fallback for noisy environments or when hands are free
10. User cannot dismiss modal during TTS playback (prevents accidental interruption)
11. Accept action (voice "yes" or button tap): Executes action (sets active job), dismisses modal, shows success message
12. Retry action (voice "no" or button tap): Clears modal, returns to microphone button ready state (user can re-record)
13. Cancel button: Dismisses modal, no action taken (no voice equivalent for safety—requires deliberate touch)
14. TTS latency target: <2 seconds from intent parse to audio playback start
15. Voice response timeout: If no "yes"/"no" detected within 10 seconds after TTS ends, buttons remain available (no auto-dismiss)
16. Mock mode: Simulates TTS playback instantly; voice responses can be simulated via test UI controls
13. Mock mode plays platform-native TTS (Web Speech API) or skips audio playback
14. Offline (production): TTS is not invoked; platform-native TTS may be used only to announce AI unavailability; no queuing

### Story 3.6: Set Active Job Voice Flow (End-to-End)

**As a** craftsman,
**I want** to say "Set active job to [number or name]" and have the app confirm and set the active job,
**so that** subsequent costs are auto-assigned without manual selection.

**Acceptance Criteria:**

1. User taps microphone button, says: "Set active job to 123" (or Czech: "Nastav aktivní práci na 123")
2. STT transcribes → LLM parses → Job #123 queried from Firestore → TTS confirms
3. Voice Confirmation Modal displays: "✓ Job: **123 - Smith, Brno**" with Accept/Retry/Cancel buttons
4. TTS plays: "Setting active job to one two three, Smith, Brno. Say 'yes' to confirm or 'no' to retry."
5. User responds "yes" (voice) OR taps Accept button → Active job state saved to app state (NgRx/Signals) and localStorage for persistence
6. Active Job Banner (persistent, top of screen) updates to show: "[123] Smith, Brno" with Deep Blue background
7. Success message (toast): "Job 123 set as active" (in user's language)
8. Modal dismisses, user returns to Voice Command Hub
9. Active job persists across app restarts (loaded from localStorage on app init)
10. If no active job set, banner shows: "No active job. Tap to select." (tapping opens jobs list)
11. Tapping active job banner opens quick "Change Active Job" modal with jobs list (tap to select new job)
12. Offline development mode only: transcription/intent/TTS use mock providers; in production offline, voice interactions are disabled (no recording, no NLU/TTS); use manual flow.
13. End-to-end flow (tap mic → accept) completes in ≤10 seconds (median) and ≤14 seconds (P95) on typical 4G network conditions with real providers
14. Voice flow tested with: numeric job IDs ("123"), partial titles ("Smith"), full titles ("Smith, Brno"), Czech phrases, voice confirmation responses ("yes"/"no", "ano"/"ne")
15. Error handling: "Job 999 not found. Say a valid job number or name."
16. Hands-free confirmation: After TTS ends, KWS listens for "yes"/"no"; if detected, executes corresponding action without requiring touch


### Story 3.7: Voice Error Handling & Recovery Flows

**As a** user,
**I want** clear feedback and recovery options when voice recognition fails,
**so that** I can complete my task despite errors.

**Acceptance Criteria:**

1. **Background Noise Interference:** If STT confidence score <0.6 (provider-specific threshold), show warning: "Audio unclear. Try again in a quieter location." with Retry/Cancel buttons
2. **Network Timeout (STT/LLM/TTS):** If request exceeds 10 seconds, show: "Network timeout. Check connection and retry." with Retry/Switch to Manual Entry/Cancel options
3. **Unrecognized Intent:** If LLM returns `intent: "unknown"` or confidence <0.7, show: "I didn't understand that. Try rephrasing or use manual entry." with Retry/Manual Entry buttons
4. **Firestore Write Failure:** If cost/event creation fails (security rules, network error), show: "Could not save. [Error reason]. Retry or save manually." with Retry/Manual Entry/Cancel
5. **Device Storage Full:** If offline queue write fails due to storage, show: "Device storage full. Free up space to continue." with link to device settings; disable voice recording until resolved
6. **Battery Critical (<10%):** Show warning banner: "Low battery. Voice features may be unreliable. Consider manual entry." Voice remains enabled but user is warned
7. **Wrong Language Detection:** If user speaks English but personProfile.language = Czech (or vice versa), LLM may fail to parse; show: "Language mismatch detected. Switch to [detected language]?" with Switch/Retry/Cancel
8. **Technical Terms Not Recognized:** If STT transcribes incorrectly (e.g., "plasterboard" → "plastic board"), user sees incorrect transcription in modal and can tap Retry before Accept
9. **Missing Required Entities:** If LLM parses intent but missing critical entity (e.g., `AddMaterialCost` without `amount`), prompt: "Please say the amount." and re-activate microphone for additional input
10. **Accent/Pronunciation Issues:** If STT consistently fails for specific user, provide feedback: "Having trouble? Try speaking more slowly or use manual entry." after 3 consecutive failures
11. **All Error States:** Include "Switch to Manual Entry" button that dismisses voice modal and opens corresponding manual form (pre-filled with any successfully parsed data)
12. **Error Logging:** Log minimal metadata only (timestamp, error category/code, non-PII context, user action). Do not store raw transcriptions or intent text in audit logs.
13. **Mock Mode:** Errors can be simulated via developer settings (e.g., force STT timeout, force low confidence, force write failure) for testing
14. **Offline Production:** Voice disabled entirely; attempting to tap microphone shows: "Voice features require internet connection. Use manual entry." with link to manual form

---

### **Rationale for Epic 3 Stories:**

**Story Sequencing:**
- 3.1: Infrastructure first (provider architecture must exist before implementing features)
- 3.2: Microphone access (can't transcribe without audio input)
- 3.3: STT (converts audio to text)
- 3.4: LLM NLU (converts text to structured intent)
- 3.5: TTS (confirmation loop pattern with hands-free voice confirmation)
- 3.6: End-to-end integration (validates entire pipeline with Set Active Job)
- 3.7: Error handling (comprehensive recovery flows for production readiness)

**Vertical Slice Validation:**
- After 3.2: Audio recording works (can capture voice)
- After 3.3: Transcription works (can see text from voice)
- After 3.4: Intent parsing works (can extract job references)
- After 3.5: Confirmation loop pattern established (reusable for all voice flows)
- After 3.6: First complete voice flow delivers user value (hands-free job switching)

**Mock Mode Importance:**
- Enables offline development (no API costs during feature work)
- Faster iteration (no network latency)
- Playwright tests can validate voice UX without real API calls

**Why Set Active Job First (not Start Journey)?**
- Simpler: No odometer parsing, no vehicle lookup, fewer entities
- Establishes pattern: All voice flows follow same STT→LLM→TTS→Confirm structure
- Foundation for Epic 4: Start Journey requires active job context to work

---
