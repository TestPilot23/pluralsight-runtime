# WebcamCard Warning State

## ADDED Requirements

### Requirement: Three UX patterns formalized (D-SM2-pre)

The system SHALL classify every applicant-facing integrity signal into exactly one of three UX patterns. Phase 2 implementation honors these patterns:

| Pattern | UX | Escalates? | Pauses? |
|---|---|---|---|
| **Escalating warning** | Red border on WebcamCard + red MD3 error toaster, persists until resolved | Yes — 3 warnings at 15s intervals → PAUSE | Yes |
| **Persistent warning** | Yellow/orange MD3 warning toaster, dismisses when condition resolves | No | No |
| **Silent event** | No applicant UX | No | No |

**Pattern assignments at Phase 2:**
- Escalating: `face.not_detected`, `webcam.stream_unavailable`
- Persistent: `mic.disabled`, `face.multiple_detected`
- Silent: everything else (`face.identity_mismatch`, `face.attention_off_screen`, `tab.*`, `window.*`, `paste.*`, `copy.*`, `picture_in_picture.*`, `media_devices.*`, `display.*`, `audio.voice_activity_detected`, `screenshot.*`, `ip.changed`, `clock.skew_detected`, `extension.suspicious_activity`)

#### Scenario: Pattern assignment is enforced
- **WHEN** a `face.not_detected` event is emitted
- **THEN** the WebcamCard transitions to its warning state (red border + red toaster) — escalating pattern
- **WHEN** a `tab.visibility_lost` event is emitted
- **THEN** no UI change occurs — silent pattern

### Requirement: Escalating warning visuals on face_not_detected

The WebcamCard SHALL transition to its warning state when `face.not_detected` fires:
- The expanded card border becomes **3px solid red** (MD3 error color)
- A red MD3 error-variant toaster appears at the bottom-center of the assessment interface with the text:

> *"We can't detect your face on camera. Please recenter or check your webcam — otherwise the assessment will end."*

The toaster persists for the full 45-second warning window (until either face restoration OR pause transition). Same toaster shown during all three warnings (text is identical).

#### Scenario: First warning shows red border + toaster
- **WHEN** `face.not_detected` is emitted with `warning_number: 1`
- **THEN** the WebcamCard's expanded preview gains a 3px solid red border
- **AND** the documented red toaster appears at the bottom of the interface

#### Scenario: Restoration removes border + toaster
- **WHEN** `face.detection_restored` is emitted
- **THEN** the red border is removed
- **AND** the toaster dismisses

### Requirement: Auto-expand-on-warning behavior (D-UI2)

When `face.not_detected` (warning #1) fires AND the WebcamCard is currently in the minimized chip state, the card SHALL automatically transition to the expanded state BEFORE the red toaster renders. This ensures the applicant can see their framing problem.

The auto-expand emits `webcam_view.visibility_changed` with `{ visible: true, trigger: "auto_expand_on_warning" }` for audit traceability.

The card SHALL stay expanded after the warning condition clears. It does NOT auto-re-minimize. The applicant must explicitly hide it again if they want.

#### Scenario: Minimized card auto-expands on warning
- **WHEN** the WebcamCard is in minimized chip state (applicant hid it earlier)
- **AND** `face.not_detected` (warning #1) fires
- **THEN** the card transitions to expanded state BEFORE the red toaster appears
- **AND** the audit log records `webcam_view.visibility_changed` with `trigger: "auto_expand_on_warning"`
- **AND** then the red toaster renders

#### Scenario: After restoration, card stays expanded
- **WHEN** face detection is restored after a warning sequence
- **THEN** the WebcamCard remains in expanded state (does NOT auto-re-minimize)
- **AND** the applicant can manually re-hide it if desired

### Requirement: Pause-state screen blur (D-SM2 + D-SM10)

The system SHALL blur the assessment interface whenever the session is in any PAUSED state. When the session transitions to ANY PAUSED state (`PAUSED.ws_disconnect`, `PAUSED.face_not_detected`, `PAUSED.webcam_stream_unavailable`):
- The entire assessment interface SHALL blur (CSS filter: blur on the main content container)
- Items SHALL be hidden (text + choices not readable through the blur)
- The WebcamCard expanded preview SHALL also blur
- The pause-state overlay text SHALL appear: *"We're having trouble connecting. Please check your internet — the assessment will resume automatically when reconnected."* (text is the WS-disconnect variant per D-SM10; the integrity-driven pauses use the same overlay pattern for now)

On transition back to IN_PROGRESS, the blur is removed and items become visible again.

#### Scenario: Pause blurs everything
- **WHEN** the session transitions to PAUSED.face_not_detected
- **THEN** the assessment interface main content area has `filter: blur(...)` applied
- **AND** the item text + choices are unreadable through the blur

#### Scenario: Resume unblurs
- **WHEN** the session transitions back from PAUSED to IN_PROGRESS
- **THEN** the blur is removed and items are visible

### Requirement: Persistent warning toasters for mic.disabled + face.multiple_detected (D-SM6)

The system SHALL display persistent warning toasters (MD3 warning variant, yellow/orange) for the two persistent-pattern signals:

- `mic.disabled` → *"Your microphone has been disabled. Please re-enable it for full assessment monitoring."*
- `face.multiple_detected` → *"Multiple faces detected. Please ensure you are alone."*

Toasters dismiss when the underlying condition resolves (mic re-enabled, only-one-face-again).

The persistent-warning visual is DISTINCT from the escalating-warning red MD3 error variant — yellow/orange MD3 warning color signals "fix this but no escalation."

#### Scenario: Mic-disabled toaster appears and dismisses
- **WHEN** the applicant disables their mic at T=0
- **THEN** the persistent mic-disabled toaster appears immediately
- **WHEN** the applicant re-enables the mic at T=20s
- **THEN** the toaster dismisses

#### Scenario: Multiple-faces toaster lifecycle
- **WHEN** `face.multiple_detected` is emitted (face_count: 2)
- **THEN** the persistent multiple-faces toaster appears
- **WHEN** subsequent face checks return face_count: 1 sustained for the next check cycle
- **THEN** the toaster dismisses

### Requirement: Silent events produce NO UX changes

The system SHALL NOT produce any toaster, banner, border change, blur, modal, or other visible UI artifact for events classified as Silent per the pattern table. This includes the high-value silent events: `face.identity_mismatch`, `face.attention_off_screen`, `tab.visibility_lost`, `window.focus_lost`, `paste.detected`, `screenshot.print_screen_detected`, `audio.voice_activity_detected`, `clock.skew_detected`, etc.

#### Scenario: Silent events log without UX
- **WHEN** a `face.identity_mismatch` event is emitted
- **THEN** the audit log contains the event with full payload
- **AND** no toaster, no border change, no banner appears on the applicant's screen
- **AND** the applicant has no indication the event fired
