# Audio Voice Activity Detection

## ADDED Requirements

### Requirement: Voice activity detection without audio storage (D-AUD1)

The system SHALL run voice activity detection (VAD) in the browser using Web Audio API + AnalyserNode for the duration of the assessment (from `session.started` through any terminal state). The mic stream is continuously analyzed for speech-like signals using energy + spectral analysis.

**Audio waveforms SHALL NEVER be stored.** Detection happens in-memory client-side only; no audio data reaches any server-side store (S3, DDB, CloudWatch, anywhere).

#### Scenario: VAD runs from session start to end
- **WHEN** a session transitions from NOT_STARTED to IN_PROGRESS
- **THEN** the VAD module begins analyzing the mic stream
- **WHEN** the session reaches any terminal state (COMPLETED, EXPIRED, DECLINED, ABANDONED)
- **THEN** the VAD module stops analyzing

#### Scenario: No audio reaches server
- **WHEN** the runtime workspace is searched for any code path that uploads audio data
- **THEN** no such code path exists
- **AND** Lambda CloudWatch logs contain no base64-encoded audio buffers
- **AND** no S3 bucket exists with audio file objects for this session

### Requirement: Sustained-voice threshold emission

The system SHALL emit `audio.voice_activity_detected` when sustained speech-like activity is detected for >2 seconds. Sustained-voice is defined as energy + spectral characteristics consistent with human speech (vs ambient noise like keyboard typing, AC hum, traffic).

Payload: `{ duration_seconds, peak_db, item_id_at_time? }`. The event is silent (no applicant UX).

#### Scenario: 3 seconds of talking emits event
- **WHEN** an applicant talks for 3 seconds while item I7 is displayed
- **THEN** `audio.voice_activity_detected` is emitted ONCE with `{ duration_seconds ≈ 3, peak_db, item_id_at_time: "I7" }`

#### Scenario: 1 second of talking does NOT emit
- **WHEN** an applicant speaks a single word (~1 second) and then is silent
- **THEN** no `audio.voice_activity_detected` event is emitted (under the 2-second threshold)

#### Scenario: Background noise (typing) does NOT emit
- **WHEN** an applicant types continuously for 30 seconds with no speech
- **THEN** no `audio.voice_activity_detected` event is emitted (typing fails the spectral-characteristic check for speech)

### Requirement: VAD pauses with the session

The VAD module SHALL suspend analysis during any PAUSED state. Resumption happens on transition back to IN_PROGRESS.

#### Scenario: PAUSED state suspends VAD
- **WHEN** a session transitions to PAUSED.face_not_detected
- **THEN** the VAD module stops emitting events
- **WHEN** the session transitions back to IN_PROGRESS
- **THEN** the VAD module resumes

### Requirement: Mouth-correlation deferred (D-AUD1)

The system SHALL NOT implement audio-applicant-mouth correlation (`audio.background_voice_likely`) at MVP. This v2 enhancement would combine VAD with face-api.js mouth-landmark detection to distinguish applicant speech from background voices.

#### Scenario: No background_voice_likely event at MVP
- **WHEN** an audit log is inspected for any session at MVP
- **THEN** no event of type `audio.background_voice_likely` exists
