# Raw Audit Log Viewer

## ADDED Requirements

### Requirement: Default-collapsed paginated audit log

The system SHALL render the session's raw audit-log events as a paginated table BELOW the IntegritySummary card on the session detail page. The viewer SHALL be default-COLLAPSED behind an MD3 expander labeled "Show full audit log."

On expand, the viewer fetches the first page of events via `GET /api/sessions/<id>/audit-events`.

#### Scenario: Default collapsed state
- **WHEN** the session detail page first renders
- **THEN** the audit log viewer shows the "Show full audit log" expander
- **AND** no events are fetched (lazy load on expand)

#### Scenario: Expanding triggers fetch
- **WHEN** the recruiter clicks "Show full audit log"
- **THEN** the viewer expands and fetches `GET /api/sessions/<id>/audit-events?page=1&size=100`
- **AND** the first 100 events display in a table

### Requirement: Pagination (D-R2)

The system SHALL paginate audit events 100 per page. Page navigation controls are MD3 Pagination component at the bottom of the table. Each new page-1 click is a fresh request; clicking pages 2+ does NOT emit a new `recruiter_access.viewed_audit_log` event (one per session viewed, to avoid log spam per D-R2 / D-AL7).

#### Scenario: Pagination loads page 2
- **WHEN** the recruiter clicks "Page 2" in the pagination control
- **THEN** the table re-renders with events 101-200 from the session
- **AND** a NEW `recruiter_access.viewed_audit_log` event is NOT emitted (only the first view counts)

### Requirement: Filter by category

The system SHALL provide MD3 chip toggles to filter events by category:
- `identity`
- `session-lifecycle`
- `integrity-signal`
- `assessment`

Filters combine with OR semantics — multiple chips active = events matching ANY of the selected categories appear in the table. Selecting no chips means "show all categories."

#### Scenario: Filter to integrity-signal only
- **WHEN** the recruiter clicks the "integrity-signal" chip and deselects others
- **THEN** only events with `category: "integrity-signal"` display in the table

#### Scenario: Multiple categories
- **WHEN** the recruiter selects both "integrity-signal" AND "session-lifecycle"
- **THEN** events from both categories display

### Requirement: Search by event_type

The system SHALL provide an MD3 search input that filters events by event_type substring match (case-insensitive).

#### Scenario: Search for "face"
- **WHEN** the recruiter types "face" in the search box
- **THEN** the table filters to events whose event_type contains "face" (case-insensitive) — `face.not_detected`, `face.detection_restored`, `face.identity_mismatch`, `face.attention_off_screen`, `face.multiple_detected`

### Requirement: Sort by timestamp (default ascending)

The system SHALL sort events by `timestamp_server` ascending by default. The column header SHALL be clickable to toggle ascending/descending.

#### Scenario: Sort descending
- **WHEN** the recruiter clicks the timestamp column header
- **THEN** the events re-sort to descending order (newest first)

### Requirement: Synchronization with current timestamp (D-R7)

The system SHALL accept `currentTimestamp` as a prop. When the prop changes (from timeline click, flagged-moment jump, or rrweb playback advancement), the viewer SHALL scroll to and highlight the events nearest to that timestamp.

Highlighting: a subtle MD3 background tint on rows within ±2 seconds of the currentTimestamp.

#### Scenario: Timeline click scrolls audit log
- **WHEN** the recruiter clicks a flagged moment at timestamp 14:30
- **THEN** the audit log viewer scrolls to events around 14:30
- **AND** events within ±2 seconds of 14:30 are highlighted

### Requirement: Event row displays envelope fields

Each row in the table SHALL display:
- `timestamp_server` (formatted as session-relative MM:SS + absolute datetime tooltip)
- `category` (chip)
- `event_type` (text)
- A "Details" expander that, when clicked, reveals the full `payload` JSON inline

#### Scenario: Row expands to show payload
- **WHEN** the recruiter clicks the "Details" expander on a `paste.detected` row
- **THEN** the row expands to show the full payload JSON including the pasted text + item_id_at_time
