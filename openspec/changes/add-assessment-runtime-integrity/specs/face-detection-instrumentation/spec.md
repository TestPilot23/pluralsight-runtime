# Face Detection Instrumentation

## ADDED Requirements

### Requirement: face-api.js runs at 5-second cadence during assessment

The system SHALL run face-api.js TinyFaceDetector + landmark + recognition nets at a configurable cadence (SSM parameter `face_detection.cadence_seconds`, default 5) from `session.started` through any terminal state. Detection runs on the live webcam stream in the browser, NOT server-side.

The model files (`tiny_face_detector_model`, `face_landmark_68_model`, `face_recognition_model`) SHALL be served from `apps/runtime-web/public/face-api-models/` via the runtime CloudFront distribution.

#### Scenario: Detection runs continuously during IN_PROGRESS
- **WHEN** a session is in IN_PROGRESS state for 30 seconds
- **THEN** face-api.js has been invoked approximately 6 times (at 5-second cadence)
- **AND** the audit log records approximately 6 detection events (any combination of `face.not_detected` / `face.detection_restored` / `face.identity_mismatch` / etc.)

#### Scenario: Detection pauses during PAUSED state
- **WHEN** a session is in PAUSED.ws_disconnect or PAUSED.face_not_detected state
- **THEN** face-api.js detection is suspended (no new events fired)
- **AND** on transition back to IN_PROGRESS, detection resumes

### Requirement: face.not_detected is the escalating-warning pattern (D-SM2)

The system SHALL emit `face.not_detected` events under the escalating-warning pattern. The escalation flow:

- **T=0:** face-api.js fails to detect a face. Emit `face.not_detected` with `{ warning_number: 1 }`. WebcamCard gains red border (D-UI2 warning state). Red toaster appears with the documented text.
- **T+15s:** Recheck. If face detected → emit `face.detection_restored` with `{ recovered_after_seconds: 15, warnings_consumed: 1 }`. Reset warning counter. WebcamCard red border + toaster dismiss. If still missing → emit `face.not_detected` with `{ warning_number: 2 }`. Toaster persists.
- **T+30s:** Recheck. Warning #3 if still missing.
- **T+45s:** Recheck. If still missing → session state transitions to **PAUSED.face_not_detected**. Emit `session.paused.face_not_detected`. Whole screen blurs.

Warning counter resets to 0 on `face.detection_restored`. The audit log captures every warning regardless of reset (for the analyzer's pattern recognition).

#### Scenario: Three warnings → pause transition
- **WHEN** a test obscures the camera at T=0
- **AND** does not restore it for 45+ seconds
- **THEN** the audit log contains `face.not_detected` events with warning_number 1, 2, 3 at approximately T=0, T+15, T+30
- **AND** at T=45+, the session state in DDB transitions to PAUSED.face_not_detected
- **AND** the audit log records `session.paused.face_not_detected`

#### Scenario: Face restoration within window resets warnings
- **WHEN** face is missing at T=0 (warning #1 fires)
- **AND** restored at T=10
- **AND** then missing again at T=60 (start of a fresh detection failure)
- **THEN** the T=60 emission has `warning_number: 1` (counter reset, not 2)

### Requirement: face.identity_mismatch is debounced and silent (D-I6, D-SM6)

The system SHALL emit `face.identity_mismatch` ONLY when the face-api.js comparison between the live frame's embedding and the enrollment embedding produces a distance ≥ threshold (SSM `face_detection.identity_threshold`, default 0.4) for **N consecutive checks** (SSM `face_detection.identity_mismatch_consecutive_min`, default 3). The event is silent — no applicant UX is shown.

Identity comparison only runs when the applicant's `face_enrollment_embedding` exists on their DDB record. If `enrollment.deferred` was emitted during Persona webhook handling, identity comparison is SKIPPED and no `face.identity_mismatch` events are produced.

#### Scenario: Single low-confidence frame does NOT emit
- **WHEN** a single face-api.js comparison returns a distance of 0.6 (above threshold)
- **AND** the next two comparisons return 0.2 and 0.25 (below threshold)
- **THEN** NO `face.identity_mismatch` event is emitted

#### Scenario: Three consecutive low-confidence frames emit
- **WHEN** three consecutive comparisons return 0.55, 0.62, 0.58 (all above threshold)
- **THEN** ONE `face.identity_mismatch` event is emitted on the third check
- **AND** the payload includes `{ enrollment_embedding_distance, consecutive_mismatches: 3, confidence_threshold }`
- **AND** NO applicant-facing UX changes (no toaster, no border, no pause)

#### Scenario: Missing enrollment skips identity comparison
- **WHEN** the applicant's record has no `face_enrollment_embedding` (Persona retrieval failed)
- **THEN** face-api.js still runs presence + attention detection
- **AND** identity-mismatch comparison is silently skipped — no events emitted for this signal

### Requirement: face.attention_off_screen targets the screenshot-LLM threat (D-I7)

The system SHALL emit `face.attention_off_screen` when face-api.js's head-pose estimation indicates sustained off-screen attention. Triggers:

- **Pitch down:** head pitch >30° down (SSM `face_detection.attention_pitch_degrees_threshold`, default 30) for ≥10 contiguous seconds (SSM `face_detection.attention_sustained_seconds`, default 10) → emit
- **Yaw to side:** head yaw >45° either left or right (SSM `face_detection.attention_yaw_degrees_threshold`, default 45) for ≥10 contiguous seconds → emit

Payload: `{ pitch_degrees, yaw_degrees, duration_seconds, item_id_at_time }`. Silent event.

This signal targets the threat of "applicant takes phone photo of screen, uploads to ChatGPT, reads answer." A consumer looking down at their phone produces a clear pitch-down head-pose signature.

#### Scenario: Sustained head-pitch-down emits
- **WHEN** an applicant's head pose pitch is sustained at 35° down for 12 seconds while item I7 is displayed
- **THEN** `face.attention_off_screen` is emitted with `{ pitch_degrees ≈ 35, yaw_degrees ≈ 0, duration_seconds ≈ 12, item_id_at_time: "I7" }`

#### Scenario: Brief head movement does NOT emit
- **WHEN** an applicant briefly looks down for 4 seconds (e.g., to type)
- **THEN** no `face.attention_off_screen` event is emitted (under the 10-second threshold)

### Requirement: face.multiple_detected is persistent warning (D-SM6)

The system SHALL emit `face.multiple_detected` whenever face-api.js detects more than one face in the live webcam stream. Persistent-warning pattern per D-SM2-pre — the toaster *"Multiple faces detected. Please ensure you are alone."* persists until only one face is detected again.

Payload: `{ face_count, item_id_at_time }`.

This is a high-value signal for "someone helping the applicant in the room." Toaster informs the applicant; analyzer interprets the duration and clustering of multi-face events post-session.

#### Scenario: Two faces appear and disappear
- **WHEN** a second person enters the camera frame for 8 seconds during item I5
- **THEN** `face.multiple_detected` is emitted on the first detection with `{ face_count: 2, item_id_at_time: "I5" }`
- **AND** the persistent toaster appears
- **WHEN** the second person leaves
- **THEN** the toaster dismisses
- **AND** subsequent face checks return to single-face state (no separate "restored" event needed — the absence of further `face.multiple_detected` events is itself the signal)

### Requirement: webcam.stream_unavailable folds into face_not_detected UX (D-SM6)

The system SHALL emit `webcam.stream_unavailable` when the webcam stream is entirely cut (vs. face_not_detected which is "stream present but no face in frame"). This event triggers the SAME escalating warning UX as face_not_detected — red border, red toaster, 3-warning escalation → PAUSE.webcam_stream_unavailable.

From the applicant's perspective, both look identical (the system "can't see them"). Distinguished in the audit log only.

Payload: `{ stream_lost_at, last_known_resolution, last_known_fps }`.

#### Scenario: Webcam stream killed mid-session triggers escalation
- **WHEN** an applicant disables their webcam at T=0 during the assessment
- **THEN** `webcam.stream_unavailable` is emitted with the payload
- **AND** the WebcamCard transitions to warning state (red border + red toaster)
- **AND** the 15s-recheck escalation proceeds identically to the face_not_detected flow
- **AND** at T=45+ if not restored, session transitions to PAUSED.webcam_stream_unavailable

### Requirement: mic.disabled is persistent warning (D-SM6)

The system SHALL emit `mic.disabled` when the mic stream becomes unavailable (applicant disabled it, OS-level cut, hardware issue). Persistent-warning pattern. Toaster: *"Your microphone has been disabled. Please re-enable it for full assessment monitoring."* Dismisses when mic stream resumes.

**Does NOT pause the assessment.** Mic is secondary signal — only webcam stream loss escalates to PAUSE.

Payload: `{ disabled_at, last_reading_db_level }`.

#### Scenario: Mic disabled fires persistent warning without pause
- **WHEN** an applicant disables their microphone mid-session
- **THEN** `mic.disabled` is emitted with payload
- **AND** the persistent mic-disabled toaster appears
- **AND** the session state remains IN_PROGRESS (NOT transitioned to PAUSED)
- **WHEN** the applicant re-enables the mic
- **THEN** the toaster dismisses
