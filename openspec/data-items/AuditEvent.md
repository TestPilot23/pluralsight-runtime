# AuditEvent

Every observable event during a session — state transitions, identity signals, integrity signals, UI interactions, recruiter-access events. Written to both DDB (fast query) and S3 (compliance archive with Object Lock).

## Storage

Dual-write:

- **DDB:** `hiringiq-pilot-audit-events`
  - **PK:** `SESSION#<session_id>`
  - **SK:** `<ts>#<event_id>` (ISO-8601 timestamp + UUID for tiebreak)
  - **GSI:** `applicant_id` for cross-session queries by applicant
- **S3:** `hiringiq-pilot-audit-archive/<org_id>/<applicant_id>/<session_id>/<timestamp>-<event_id>.json`
  - Object Lock in **compliance mode**, 180-day retention.
  - Glacier transition at 30 days.

The dual write is part of the same `services/telemetry` ingest handler. A failure to write to either store rolls back the other.

## Envelope (D-AL1)

| Field | Type | Notes |
|---|---|---|
| `event_id` | String (UUID v7) | Idempotency key |
| `session_id` | String (UUID v7) | Owning session |
| `applicant_id` | String (UUID v7) | Denormalized for cross-session GSI queries |
| `org_id` | String (UUID v7) | Denormalized for recruiter-side tenant filtering |
| `event_type` | String | Dotted category (see taxonomy below) |
| `category` | String | `"identity"` \| `"session-lifecycle"` \| `"integrity-signal"` \| `"assessment"` \| `"recruiter-access"` |
| `timestamp_client` | String | ISO-8601 UTC stamped by the browser at emission |
| `timestamp_server` | String | ISO-8601 UTC stamped by the telemetry Lambda on receipt |
| `source_ip` | String | Captured per-event from the WS connection, not just at `$connect` |
| `schema_version` | Integer | `1` at present; bump only on breaking changes |
| `payload` | Object | Event-type-specific payload (see taxonomy) |

## Taxonomy (D-AL2)

| Category | Example event types |
|---|---|
| `identity` | `persona.attempted`, `persona.succeeded`, `persona.declined.<reason>`, `persona.api_error`, `persona.bypassed`, `enrollment.photo_stored`, `enrollment.embedding_computed`, `enrollment.deferred` |
| `session-lifecycle` | `session.landing_page_viewed`, `terms.accepted`, `session.started`, `session.paused.<reason>`, `session.resumed`, `session.completed`, `session.completion_screen_not_confirmed`, `session.access_attempted_with_expired_link`, `applicant.declined.confirmed` |
| `integrity-signal` | `face.not_detected`, `face.identity_mismatch`, `face.attention_off_screen`, `face.multiple_detected`, `audio.voice_activity_detected`, `tab.visibility_lost`, `paste.detected`, `copy.detected`, `screenshot.print_screen_detected`, `media_devices.suspicious_present`, `display.secondary_present`, `ip.changed`, `clock.skew_detected`, `extension.suspicious_activity`, `webcam_view.visibility_changed` |
| `assessment` | `item.displayed`, `item.submitted`, `item.time_expired`, `item.null_submission` |
| `recruiter-access` | `recruiter_access.viewed_session`, `recruiter_access.viewed_replay`, `recruiter_access.viewed_audit_log`, `recruiter_access.viewed_persona_selfie`, `biometric.deletion_requested`, `analyzer.re_run_requested` |

## Invariants

- Every event has both timestamps. Drift > SSM threshold emits `clock.skew_detected`.
- Events are **never overwritten**. Adding new event types is additive — no `schema_version` bump.
- S3 archive objects are **immutable** for 180 days due to Object Lock compliance mode. Even root cannot delete during the retention window.
- Recruiter-access events count against right-to-deletion **retention** — they are kept even when an applicant's biometric data is deleted, so the access audit log remains complete.

## Access patterns

- **Per-session timeline** (recruiter dashboard): Query `PK = SESSION#<id>` with paginated results.
- **Per-applicant history**: Query the `applicant_id` GSI (used by the right-to-deletion auditor).
- **Analyzer ingest**: SQS triggers the analyzer-service which reads the full session's events via PK Query before producing an [[IntegritySummary]].

## Related entities

- [[Session]] — partition key
- [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] — GSI access path
- [[IntegritySummary]] — analyzer consumes events to produce summaries

## Source proposals

- `add-assessment-runtime-foundation` — defining proposal (envelope, DDB table, basic event types)
- `add-assessment-runtime-integrity` — expands taxonomy to the integrity-signal category, adds S3 archive with Object Lock
- `add-recruiter-assessment-review` — adds the recruiter-access category
