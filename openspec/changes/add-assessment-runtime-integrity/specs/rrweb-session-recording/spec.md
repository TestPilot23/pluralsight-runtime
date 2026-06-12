# rrweb Session Recording

## ADDED Requirements

### Requirement: Always-on rrweb capture for every session (D-RR1)

The system SHALL run `rrweb-record` in the runtime React app for the duration of every assessment session, from `session.started` through any terminal state. The capture is not conditional — every session gets recorded regardless of integrity-signal severity.

The recorder captures: DOM mutations, mouse moves, scrolls, clicks, inputs (with selective input capture — see below), and viewport resize events.

#### Scenario: Recorder starts on session.started
- **WHEN** a session transitions to IN_PROGRESS via Begin Assessment
- **THEN** the rrweb recorder is initialized and begins capturing events
- **AND** the audit log records `rrweb.recording_started`

#### Scenario: Recorder stops at terminal state
- **WHEN** a session transitions to any terminal state (COMPLETED, EXPIRED, DECLINED, ABANDONED)
- **THEN** the rrweb recorder stops
- **AND** the audit log records `rrweb.recording_stopped`
- **AND** any in-memory buffer is flushed to S3

### Requirement: Selective input capture excludes test content

The system SHALL configure rrweb to capture inputs ONLY from the short-answer textarea (the applicant's typed responses). The MC option labels SHALL be excluded from input capture — they contain test content that we don't want leaking outside the bank-content system.

Implementation: rrweb's `inputs` option set to a selector like `'textarea[data-rrweb-capture="true"]'`, and the short-answer textarea component sets `data-rrweb-capture="true"` while MC option labels do NOT.

#### Scenario: Short-answer typing is captured
- **WHEN** an applicant types "binary search" into a short-answer textarea
- **THEN** the rrweb event stream includes the keystroke events with the typed content

#### Scenario: MC option labels are not captured
- **WHEN** an MC item is displayed with labels "ClusterIP", "NodePort", "LoadBalancer", "ExternalName"
- **THEN** the rrweb event stream does NOT contain the option text as input data
- **AND** inspection of the captured stream confirms only the textarea inputs are present

### Requirement: Batched uploads via dedicated HTTPS endpoint (D-RR1, D-I3)

The system SHALL batch rrweb events and upload them via `POST /api/telemetry/rrweb-batch` HTTPS endpoint. Batching parameters:
- Flush every **5 seconds**, OR
- Flush when the buffer reaches **100 events**, whichever first

The upload uses HTTPS, NOT the WebSocket. This prevents high-volume rrweb data from competing with the integrity-signal WS stream for bandwidth.

Authorization: each batch includes the magic-link JWT in the `Authorization: Bearer <jwt>` header. The telemetry-service validates the JWT and looks up the session before persisting.

#### Scenario: Batch upload at 5-second interval
- **WHEN** the recorder accumulates events for 5 seconds
- **THEN** a POST to `/api/telemetry/rrweb-batch` fires with the buffered events
- **AND** the buffer is cleared on successful upload

#### Scenario: Batch upload at 100-event threshold
- **WHEN** the recorder accumulates 100 events within 3 seconds (e.g., during heavy DOM mutation)
- **THEN** the batch flushes immediately at 100 events without waiting for the 5-second mark

### Requirement: Network resilience via in-memory buffering

The system SHALL handle HTTPS upload failures by holding batches in memory and retrying. When an HTTPS upload fails (5xx, network error, timeout):
- The system SHALL hold the batch in memory and retry on the next flush interval
- WS disconnect SHALL NOT cause rrweb data loss because rrweb uses a separate HTTPS path
- After session-end, any remaining buffered events SHALL be uploaded before the recorder fully stops

There is a maximum in-memory buffer size (configurable, default 10 MB) beyond which oldest batches are discarded with an emitted warning event. Acceptable trade-off — losing some rrweb data on a sustained network outage is better than crashing the browser.

#### Scenario: Failed upload retries
- **WHEN** an HTTPS upload returns 503 Service Unavailable
- **THEN** the batch is held in memory
- **AND** on the next 5-second flush, both the held batch and the new events are uploaded together

#### Scenario: Sustained outage caps the buffer
- **WHEN** uploads fail continuously for 5+ minutes and buffered data exceeds 10 MB
- **THEN** the oldest batches are dropped from the buffer
- **AND** an audit log event `rrweb.buffer_overflow` is emitted (silent) with `{ dropped_events_count, total_buffered_bytes }`

### Requirement: S3 storage with KMS and 180-day retention (D-RR2)

The system SHALL store rrweb event streams in S3 bucket `hiringiq-pilot-rrweb-recordings/` with:
- KMS encryption at rest
- Restricted IAM (only `telemetry-service` writes; only Phase-3 dashboard playback proxy reads)
- 180-day lifecycle policy for automated deletion
- Object key pattern: `<org_id>/<session_id>/events-<batch_seq>.json` (per-batch objects)

At session end, the telemetry-service MAY concatenate batches to a single object via S3 multipart upload OR retain batches as separate objects (implementation choice — both work).

#### Scenario: Batches reach S3 with documented key pattern
- **WHEN** a session emits 3 batches before completion
- **THEN** the rrweb-recordings bucket contains objects matching the key pattern
- **AND** all objects are KMS-encrypted

#### Scenario: 180-day retention policy is configured
- **WHEN** the S3 bucket's lifecycle configuration is inspected
- **THEN** the policy expires objects 180 days after creation
- **AND** the policy matches the biometric retention posture from D-P5

### Requirement: Audit-log events on recording lifecycle

The system SHALL emit silent audit-log events on the rrweb capture lifecycle:
- `rrweb.recording_started` at session.started
- `rrweb.recording_stopped` at terminal state transition
- `rrweb.buffer_overflow` on sustained-outage buffer truncation (rare)

#### Scenario: Lifecycle events captured
- **WHEN** an applicant completes an assessment
- **THEN** the audit log contains `rrweb.recording_started` near `session.started` and `rrweb.recording_stopped` near `session.completed`

### Requirement: Playback library is rrweb-player (Phase 3 will consume)

The system SHALL reserve `rrweb-player` (the MIT-licensed playback library) for the Phase 3 recruiter dashboard's session-replay view. Phase 2 does NOT install the player into the runtime-web app (applicants don't play back their own sessions).

#### Scenario: rrweb-player not bundled in runtime-web
- **WHEN** the runtime-web production bundle is inspected
- **THEN** the bundle does NOT contain rrweb-player code
- **AND** only the rrweb-record runtime is included

### Requirement: rrweb-recording S3 key surfaced on IntegritySummary

The system SHALL include the rrweb recording's S3 key in the IntegritySummary produced by the analyzer per D-RR3:

```ts
{
  ...,
  rrweb_recording_s3_key: string,    // "<org_id>/<session_id>/" prefix; the dashboard concatenates batches at read time
}
```

#### Scenario: IntegritySummary references rrweb prefix
- **WHEN** the analyzer produces an IntegritySummary for a completed session
- **THEN** the summary includes `rrweb_recording_s3_key` pointing at the session's rrweb prefix
- **AND** the dashboard in Phase 3 can use this key to fetch and play back the recording
