# Device Check

## ADDED Requirements

### Requirement: Device check is the screen between Persona and the assessment intro

The system SHALL display the device check screen as the third applicant-facing screen, sequenced after Persona verification (or Persona bypass per D-P1) and before the assessment intro screen. The screen SHALL contain (top to bottom):
- Webcam status card (informational only — verified during Persona's selfie capture; if Persona was bypassed, the card shows "Camera detected ✓" based on the prior permission grant)
- Microphone live volume meter
- Speaker self-attest test
- "Ready" call-to-action (bottom-right)

The "Ready" CTA SHALL be always enabled. Outcomes of mic and speaker tests are captured as audit events but DO NOT gate progression.

#### Scenario: Screen renders all three test cards
- **WHEN** the device check screen mounts
- **THEN** the rendered page contains a webcam status card, a microphone test card, and a speaker test card
- **AND** the "Ready" CTA is visible at the bottom-right and is enabled from the moment of render

### Requirement: Webcam card is informational, with "View camera" toggle

The webcam card SHALL show `📷 Webcam ✓ OK` with the explanatory text *"Verified during identity check."* (or *"Camera detected."* if Persona was bypassed). A button labeled "View camera" (with Material Symbols `videocam` icon) SHALL toggle an inline live preview of the webcam stream.

Tapping "View camera" SHALL emit `device_check.webcam_preview_viewed` (silent audit event). The preview is purely for the applicant's reassurance — framing check before the assessment begins.

#### Scenario: View camera toggle opens inline preview
- **WHEN** the applicant taps "View camera"
- **THEN** an inline 240×180 webcam preview appears below the card title
- **AND** the audit event `device_check.webcam_preview_viewed` fires once

#### Scenario: View camera toggle closes the preview on second tap
- **WHEN** the applicant taps "View camera" a second time
- **THEN** the inline preview is hidden
- **AND** no additional `device_check.webcam_preview_viewed` event fires (the event captures only opens, not closes)

### Requirement: Microphone test with live volume meter

The microphone card SHALL display a live volume meter showing the current audio level detected on the applicant's microphone stream. The meter SHALL update in real-time using Web Audio API's AnalyserNode.

**Pass threshold:** sustained audio above the noise floor for ~2 seconds. On pass, the meter SHALL show a green ✓ state indicating the mic is working.

**Failure handling:** If no audio is detected after 30 seconds, the card SHALL display inline troubleshooting tips: *"Try unmuting your mic, check the OS sound settings, or try a different mic from your system's audio settings."* The Ready CTA remains enabled — the applicant can proceed without passing the mic test.

The page SHALL NOT include language like "your recruiter will be notified" — same telegraphing-avoidance reasoning as the intro screen.

#### Scenario: Mic detects audio and shows pass state
- **WHEN** the applicant speaks for 3 seconds at normal volume
- **THEN** the live meter rises and shows the pass state with a green ✓
- **AND** the audit event `device_check.mic_test_passed` fires once

#### Scenario: Silent mic for 30 seconds surfaces troubleshooting
- **WHEN** the applicant remains silent for 30 seconds
- **THEN** the card displays the documented troubleshooting text
- **AND** the audit event `device_check.mic_test_skipped` fires (applicant did not pass the test, but can still proceed)

### Requirement: Speaker self-attest test

The speaker card SHALL provide a `[▶ Play sound]` button and a yes/no question: *"Did you hear it?"* with radio buttons `○ Yes` / `○ No`. Clicking Play SHALL play a short audio file (~3-second tone or chord) at standard volume.

If the applicant selects "No," the card SHALL display inline troubleshooting tips: *"Check your volume, try headphones, or test sound in another tab."* The applicant can replay the test by clicking Play again.

The speaker test SHALL NOT block continuation. The Ready CTA remains enabled regardless of self-attest answer.

#### Scenario: Yes attestation captures the affirmative
- **WHEN** the applicant clicks Play, hears the sound, and selects Yes
- **THEN** the audit event `device_check.speaker_test_self_attest` fires with payload `{ heard: true }`

#### Scenario: No attestation surfaces troubleshooting without blocking
- **WHEN** the applicant clicks Play, doesn't hear the sound, and selects No
- **THEN** the card displays the documented troubleshooting text
- **AND** the audit event `device_check.speaker_test_self_attest` fires with payload `{ heard: false }`
- **AND** the Ready CTA remains enabled

### Requirement: Persistent WebcamCard is NOT shown on this screen (D-DC3)

The persistent corner WebcamCard from D-UI2 SHALL NOT be rendered on the device check screen. The device check has its OWN webcam status card (purpose-built for framing confirmation). Duplicating the persistent corner card would create two webcam UIs on one screen — confusing.

The persistent WebcamCard first appears on the assessment interface (not on landing, Persona, device-check, or intro screens). Webcam recording IS still happening during pre-assessment screens (per D-IS5 — WS opened on landing, audit-events captured); the persistent UI element just isn't there.

#### Scenario: No persistent corner card on device-check render
- **WHEN** the device check screen mounts
- **THEN** the rendered DOM contains exactly ONE webcam-related UI element (the device-check screen's status card)
- **AND** the persistent corner card (the one introduced by D-UI2 for the assessment interface) is NOT rendered

### Requirement: CTA label is "Ready"

The continuation CTA on the device check screen SHALL be labeled **"Ready"** (per the original brief — NOT "Continue" or any other variant). Placement: bottom-right. Always enabled.

#### Scenario: CTA renders with exact "Ready" label
- **WHEN** the device check screen renders
- **THEN** the bottom-right button contains the exact text "Ready" (no period, no decoration)

### Requirement: Audit events on device check

The system SHALL emit the following silent audit events on the device check screen:
- `device_check.viewed` on mount
- `device_check.webcam_preview_viewed` when the View camera toggle is opened (D-DC2)
- `device_check.mic_test_passed` when the mic threshold is met (or `device_check.mic_test_skipped` if 30s elapses with no detection)
- `device_check.speaker_test_played` on each click of the Play sound button
- `device_check.speaker_test_self_attest` with `{ heard: boolean }` payload
- `device_check.continue_clicked` (which the spec internally maps to "Ready" — name preserved for consistency with the broader event taxonomy)

#### Scenario: Happy-path event sequence on device check
- **WHEN** an applicant views the screen, opens the webcam preview, passes the mic test, plays the speaker sound, attests Yes, and clicks Ready
- **THEN** the audit-events table contains events in this order: `device_check.viewed`, `device_check.webcam_preview_viewed`, `device_check.mic_test_passed`, `device_check.speaker_test_played`, `device_check.speaker_test_self_attest` (with `heard: true`), `device_check.continue_clicked`
