# Integrity Event Taxonomy

## ADDED Requirements

### Requirement: Audit-log envelope shape (D-AL1)

The system SHALL use the following envelope shape for every audit-log event written to either the DDB hot-path or the S3 archive:

```ts
{
  event_id: string,             // UUID v7, server-assigned
  schema_version: number,       // 1 at this phase; bump only on breaking envelope changes
  session_id: string,
  applicant_id: string,
  campaign_id: string,
  org_id: string,
  timestamp_server: string,     // ISO-8601 UTC, authoritative ordering
  timestamp_client: string,     // ISO-8601 UTC, advisory (skew detection)
  ip: string,                   // captured per-event from the WS connection
  user_agent: string,           // denormalized for searchability
  category: "identity" | "session-lifecycle" | "integrity-signal" | "assessment" | "operational" | "recruiter-access",
  event_type: string,           // e.g., "persona.attempted", "item.displayed"
  payload: Record<string, unknown>,
}
```

The envelope SHALL NOT contain a `severity` field. Per-event severity is a deliberate non-decision — the post-session analyzer applies severity bands via `timeline_segments` (D-RR3).

#### Scenario: Every event written includes the full envelope
- **WHEN** the telemetry-service receives a client-emitted event
- **THEN** the persisted record in `hiringiq-pilot-audit-events` contains every documented field
- **AND** `timestamp_server` is set to the server-side receipt time (not the client-supplied timestamp)

#### Scenario: Envelope rejects events with unknown category
- **WHEN** a client emits an event with `category: "unknown"`
- **THEN** the telemetry-service rejects the event with a 400 response and emits an internal alert; no audit-log entry is created

### Requirement: Event taxonomy across four categories (D-AL2)

The system SHALL emit events from exactly six categories: `identity`, `session-lifecycle`, `integrity-signal`, `assessment`, `operational`, `recruiter-access`. The complete event-type taxonomy at Phase 2 includes:

**identity:**
- `persona.attempted`, `persona.succeeded`, `persona.declined.<reason>`, `persona.api_error`, `persona.bypassed`
- `enrollment.photo_stored`, `enrollment.embedding_computed`, `enrollment.deferred`

**session-lifecycle:**
- `session.landing_page_viewed`, `session.intro_screen_viewed`, `session.started`, `session.paused.ws_disconnect`, `session.paused.face_not_detected`, `session.paused.webcam_stream_unavailable`, `session.resumed`, `session.completed`, `session.abandoned`, `session.expired`, `session.completion_screen_not_confirmed`
- `applicant.declined.initiated`, `applicant.declined.cancelled`, `applicant.declined.confirmed`
- `session.access_attempted_with_expired_link`, `session.access_attempted_to_closed_campaign`, `session.access_attempted_from_small_screen`, `session.access_attempted_from_unsupported_browser`
- `consent.checkbox_checked`, `consent.checkbox_unchecked`, `terms.accepted`, `terms.opened`, `privacy_notice.opened`, `permissions.granted`, `permissions.denied`, `session.landing_continue_clicked`
- `device_check.viewed`, `device_check.webcam_preview_viewed`, `device_check.mic_test_passed`, `device_check.mic_test_skipped`, `device_check.speaker_test_played`, `device_check.speaker_test_self_attest`, `device_check.continue_clicked`
- `webcam_view.visibility_changed`
- `rrweb.recording_started`, `rrweb.recording_stopped`

**integrity-signal:**
- `face.not_detected`, `face.detection_restored`, `face.identity_mismatch`, `face.multiple_detected`, `face.attention_off_screen`, `webcam.stream_unavailable`, `mic.disabled`
- `tab.visibility_lost`, `tab.visibility_restored`, `window.focus_lost`, `window.focus_restored`
- `paste.detected`, `copy.detected`
- `picture_in_picture.entered`, `picture_in_picture.exited`
- `media_devices.changed`, `media_devices.suspicious_present`
- `display.secondary_present`, `display.permission_denied`
- `audio.voice_activity_detected`
- `screenshot.print_screen_detected`
- `ip.changed`
- `clock.skew_detected`
- `extension.suspicious_activity`

**assessment:**
- `item.displayed`, `item.submitted`, `item.time_expired`, `item.null_submission`, `text.selection_attempted`

**operational** (system events not driven by applicant behavior):
- `rrweb.recording_started`, `rrweb.recording_stopped`, `rrweb.buffer_overflow`
- `biometric.deletion_requested`, `biometric.deleted`
- `enrollment.photo_stored`, `enrollment.embedding_computed`, `enrollment.deferred`
- `analyzer.re_run_requested`, `session.completion_screen_not_confirmed`

**recruiter-access** (Phase 3 dashboard view events; documented here as the canonical taxonomy entry):
- `recruiter_access.viewed_session`
- `recruiter_access.viewed_replay`
- `recruiter_access.viewed_audit_log`
- `recruiter_access.viewed_persona_selfie`
- `recruiter_access.viewed_embedding`
- `recruiter_access.cross_org_attempt`

The taxonomy is EXTENSIBLE — new event types can be added in future proposals WITHOUT bumping `schema_version` (additive change).

#### Scenario: All four categories are valid
- **WHEN** events from each of the four categories are emitted during a session
- **THEN** each event passes envelope validation and is persisted to both DDB + S3

#### Scenario: Adding new event type does not bump schema_version
- **WHEN** a future proposal adds a new event type `face.eye_tracking_anomaly` under category `integrity-signal`
- **THEN** existing audit-log events at `schema_version: 1` remain valid and queryable
- **AND** the new event type writes as `schema_version: 1` until a STRUCTURAL change to the envelope occurs

### Requirement: Persist all events with no client/server filtering (D-AL1)

The system SHALL persist EVERY event the telemetry-service receives to both the DDB hot-path and the S3 archive. The system SHALL NOT filter events by severity, type, or volume. Decisions about what's "important" are reserved for the post-session analyzer (D-AL4).

#### Scenario: High-volume events are not dropped
- **WHEN** an applicant produces 1,500+ `text.selection_attempted` events in a single session by repeatedly trying to select MC item text
- **THEN** every single event appears in the audit log
- **AND** none are dropped or aggregated at the server

### Requirement: Dual-timestamp capture (D-AL1)

The system SHALL capture BOTH `timestamp_server` and `timestamp_client` on every event. `timestamp_server` is set by the telemetry-service Lambda on event receipt; `timestamp_client` is set by the React app at event emission. Server timestamp is authoritative for ordering and time-up enforcement; client timestamp enables clock-skew detection.

#### Scenario: Both timestamps preserved
- **WHEN** an event is emitted at client time `T_c` and received at server time `T_s`
- **THEN** the persisted record has both `timestamp_client: T_c` and `timestamp_server: T_s`
- **AND** the values may differ (network latency + client clock skew)

### Requirement: Clock-skew detection (D-AL1a)

The system SHALL compute clock-skew baseline from the first 3 events of a session: `baseline_offset_ms = median(server_receipt_time - client_emission_time - estimated_one_way_latency)`. Estimated latency comes from WS ping/pong probes.

The system SHALL emit `clock.skew_detected` under two conditions:
- **Baseline exceeded:** `|baseline_offset_ms| > 5000ms` (SSM-configurable). Emitted once at session start.
- **Mid-session drift:** `|current_offset_ms - baseline_offset_ms| > 2000ms` (SSM-configurable). Emitted on the event where the threshold is first exceeded.

Both thresholds are SSM parameters at `/hiringiq/pilot/runtime/integrity/clock-skew/*`.

#### Scenario: Baseline-exceeded emits at session start
- **WHEN** an applicant's first 3 events have a median offset of 8000ms
- **THEN** the system emits `clock.skew_detected` with payload `{ baseline_offset_ms: 8000, threshold: 5000, reason: "baseline_offset_exceeded" }` once
- **AND** subsequent events with the same offset DO NOT trigger duplicate emissions

#### Scenario: Mid-session drift emits on the threshold-crossing event
- **WHEN** session baseline is 200ms and a mid-session event arrives with offset 2500ms (drift of 2300ms)
- **THEN** the system emits `clock.skew_detected` with payload `{ baseline_offset_ms: 200, current_offset_ms: 2500, delta_ms: 2300, item_id_at_time }` on that event
- **AND** the audit log captures this as a leading-indicator signal

### Requirement: S3 immutable archive with Object Lock compliance mode (D-AL3, D-I8)

The system SHALL provision an S3 bucket `hiringiq-pilot-audit-archive/` with Object Lock in **compliance mode**, default retention 180 days, KMS encryption, lifecycle transition to Glacier at 30 days. Every audit-log event written to the DDB hot-path SHALL ALSO be written to this bucket as a separate object at key `<org_id>/<applicant_id>/<session_id>/<timestamp>-<event_id>.json`.

Compliance mode means even an AWS root account user CANNOT delete objects within the retention window — court-defensible tamper-evidence.

#### Scenario: Every DDB event has an S3 archive counterpart
- **WHEN** an integration test writes a known event to the audit log
- **THEN** the DDB record AND the S3 object both exist with identical envelope content
- **AND** the S3 object's bucket-level `ObjectLockConfiguration` is `COMPLIANCE` mode

#### Scenario: Object Lock prevents deletion within retention
- **WHEN** an authenticated admin attempts to `s3:DeleteObject` on an archive entry within the retention window
- **THEN** the call returns AccessDenied even with full IAM rights
- **AND** the object remains
