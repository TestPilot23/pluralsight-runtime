# Assessment Session State

## ADDED Requirements

### Requirement: Session states + terminal states (D-SM1, D-SM7)

The system SHALL implement the following session state machine. Server-authoritative — the server is the source of truth; the client renders the current state.

**States:**
- `NOT_STARTED` — applicant has been on intro screen but has not clicked Begin Assessment
- `IN_PROGRESS` — assessment is actively running; items are being delivered
- `PAUSED.<reason>` — session is paused with a recoverable reason. At Phase 1, only `PAUSED.ws_disconnect` is implemented (D-SM7). Integrity-driven pauses (face_not_detected, etc.) arrive in Phase 2.
- `ABANDONED` — session was PAUSED for >30 min with no resume. **Recoverable** if the applicant returns via the valid magic link.
- `COMPLETED` — terminal state. Applicant finished all items.
- `EXPIRED` — terminal state. Magic-link JWT expired while session was in ABANDONED.
- `DECLINED` — terminal state. Applicant clicked decline on the intro screen and confirmed.

**Terminal states:** `COMPLETED`, `EXPIRED`, `DECLINED`. (ABANDONED is recoverable; not terminal.)

#### Scenario: NOT_STARTED → IN_PROGRESS on Begin Assessment
- **WHEN** the applicant clicks Begin Assessment with consent checked
- **THEN** the session record's state transitions from NOT_STARTED to IN_PROGRESS
- **AND** the audit event `session.started` is emitted

#### Scenario: IN_PROGRESS → PAUSED.ws_disconnect on disconnect
- **WHEN** the WebSocket connection drops while session is IN_PROGRESS
- **THEN** the `$disconnect` Lambda transitions the session to PAUSED.ws_disconnect
- **AND** an EventBridge Scheduler trigger is created to transition to ABANDONED in 30 minutes
- **AND** the audit event `session.paused.ws_disconnect` is emitted

#### Scenario: PAUSED.ws_disconnect → IN_PROGRESS on reconnect
- **WHEN** the applicant reconnects within the 30-minute window
- **THEN** the `$connect` Lambda transitions the session back to IN_PROGRESS
- **AND** the scheduled ABANDONED trigger is cancelled
- **AND** the audit event `session.resumed` is emitted

#### Scenario: PAUSED → ABANDONED on timeout
- **WHEN** a session remains in PAUSED for 30 minutes with no resume
- **THEN** the EventBridge trigger fires and transitions the session to ABANDONED
- **AND** the audit event `session.abandoned` is emitted

#### Scenario: ABANDONED → IN_PROGRESS on applicant return
- **WHEN** a session in ABANDONED state has its magic link clicked again, and the JWT is still valid
- **THEN** the session transitions back to IN_PROGRESS
- **AND** items already seen remain locked per the "seen" taxonomy
- **AND** the audit event `session.resumed` is emitted

#### Scenario: ABANDONED → EXPIRED on JWT expiry
- **WHEN** a session in ABANDONED state has its magic link's JWT TTL expire
- **THEN** the session transitions to the terminal state EXPIRED
- **AND** the audit event `session.expired` is emitted

### Requirement: Item display order is randomized per applicant, deterministic from session_id (D-SM9)

The system SHALL shuffle the campaign's locked `assessment_questions` array per-applicant. The shuffle SHALL be:
- **Fisher-Yates** with a seed derived from SHA-256 of the `session_id`
- **Deterministic** — given the same `session_id`, two runs of the shuffle produce identical orderings (useful for debugging + analyzer reproducibility)
- **Per-applicant unique** — different sessions on the same campaign produce different orderings (since session_id is unique per applicant-per-campaign)

The shuffled order is computed once at `session.started` and persisted on the session record.

#### Scenario: Two applicants on same campaign see different orders
- **WHEN** applicants `A1` and `A2` both take Campaign `C1` (with 5 locked items in fixed positions)
- **THEN** the orders presented to A1 and A2 are not identical (with overwhelmingly high probability)
- **AND** the audit log records each applicant's item-displayed events in the order they were actually shown to that applicant

#### Scenario: Same session_id always produces same order
- **WHEN** the shuffle is run twice in independent processes with the same session_id input
- **THEN** the resulting orderings are identical

### Requirement: "Seen" taxonomy (D-SM4)

The system SHALL track item exposure per applicant with three states:
- `not_served` — server never sent this item to the client
- `served_unanswered` — the `item.displayed` event fired, but no `item.submitted` followed (timer expired without selection, OR window closed, OR session became PAUSED-then-ABANDONED)
- `served_answered` — `item.displayed` followed by `item.submitted` with a real answer

Both `served_*` states count as "seen for exposure" — the applicant CANNOT be presented this item again in a future session resumption (after ABANDONED return) or in any future re-attempt for this campaign.

The taxonomy is enforced server-side. The client does NOT have authority over what counts as "seen."

#### Scenario: Item served but not answered marks served_unanswered
- **WHEN** the assessment-service sends item `I3` to the client
- **AND** the applicant closes the window before submitting
- **THEN** the session record marks `I3` as `served_unanswered`
- **AND** on resume, item `I3` is NOT re-presented; the next un-seen item is presented

#### Scenario: Resume after ABANDONED locks previously-seen items
- **WHEN** an applicant abandons a session after seeing items `[I1, I2, I3]` and submitting `[I1, I2]`
- **AND** returns within JWT validity
- **THEN** the session resumes with items `[I1, I2]` in `served_answered`, item `I3` in `served_unanswered`
- **AND** the next item presented is from the remaining un-seen pool

### Requirement: Intentional close handling (D-SM5)

The server SHALL NOT attempt to distinguish window-close from browser-crash; both produce the same signal (WebSocket disconnect or no signal at all). Both are treated identically:
- The item open at the time is marked `served_unanswered`
- On reconnect (within the 30-min PAUSED → ABANDONED window OR after ABANDONED → IN_PROGRESS resume), the applicant resumes at the NEXT item they haven't seen
- Per D-SM4, the abandoned item is locked from future exposure

At MVP, there is NO "Quit assessment" CTA inside the assessment interface. Applicants who want to abandon close the window. If pilot data shows applicants want a clean exit path, that's a v2 enhancement.

#### Scenario: Window close mid-item marks served_unanswered
- **WHEN** an applicant has item `I5` displayed and closes their browser
- **THEN** the WebSocket $disconnect Lambda fires
- **AND** session state transitions to PAUSED.ws_disconnect
- **AND** item `I5` is marked `served_unanswered` (in addition to the pause-state transition)
- **WHEN** the applicant reconnects 5 minutes later
- **THEN** the next item presented is the next un-seen item after `I5`'s original position in the shuffled order — NOT `I5` again

### Requirement: Pause-time budget per item (D-SM3)

Each item SHALL have a pause-time budget of 90 seconds. Pause-time accumulates across ALL pause reasons (ws_disconnect today; face_not_detected and other reasons in Phase 2).

After 90 seconds of cumulative pause within a single item, the item's timer KEEPS RUNNING through any further pause. The screen remains blurred and the item hidden, but the time-up enforcement clock no longer halts.

This blocks the "fake disconnect at 0:05 to dodge the timer" attack while giving honest applicants generous recovery time.

The budget value is SSM-configurable at `/hiringiq/pilot/runtime/pause-time-budget-seconds`.

#### Scenario: Pause within budget pauses the timer normally
- **WHEN** an applicant is on a 60-second timed item with 30 seconds remaining
- **AND** the WS disconnects for 45 seconds (within the 90s budget)
- **AND** then reconnects
- **THEN** on reconnect, the item still has 30 seconds remaining

#### Scenario: Pause beyond budget keeps timer running
- **WHEN** an applicant is on a 60-second timed item with 30 seconds remaining
- **AND** the WS disconnects for 120 seconds (90s within budget, then 30s beyond)
- **AND** then reconnects
- **THEN** on reconnect, the item has 0 seconds remaining (the timer continued for the 30 seconds beyond budget)
- **AND** the time-expired auto-submit has fired

### Requirement: Server-side grading without exposing answer to client

The server SHALL grade each `item.submitted` event server-side after fetching the canonical answer from the item-bank table. The item's `answer` field SHALL NEVER be sent to the client.

Per item type:
- `multiple_choice_single`: grading compares integer equality between submitted index and stored answer
- `multiple_choice_multiple`: grading compares set equality between submitted (sorted, unique) array and stored answer array
- `short_answer`: grading is documented post-MVP; at Phase 1, the system records the raw response on the session record without auto-grading (the recruiter or future analyzer interprets)

#### Scenario: MC-single grading is exact integer match
- **WHEN** an item has `answer: 2` and the applicant submits `answer: 2`
- **THEN** the session record marks the item correct
- **WHEN** the applicant submits `answer: 0`
- **THEN** the session record marks the item incorrect

#### Scenario: MC-multiple grading is set equality on sorted answer arrays
- **WHEN** an item has `answer: [0, 2]` (sorted, unique per the item-bank schema)
- **AND** the applicant submits `answer: [0, 2]`
- **THEN** the session record marks the item correct
- **WHEN** the applicant submits `answer: [0, 2, 3]`
- **THEN** the session record marks the item incorrect (extra selection makes the set non-equal)
- **WHEN** the applicant submits `answer: [0]`
- **THEN** the session record marks the item incorrect (missing selection)

#### Scenario: Answer field is not exposed in the API
- **WHEN** the client requests the next item via `GET /sessions/<id>/items/next`
- **THEN** the response contains the item's id, title, text, type, choices (for MC types), time_limit_seconds, but NOT the `answer` field
- **AND** inspecting browser network traffic confirms the `answer` field is absent from every item-fetch response

### Requirement: Session record schema

The system SHALL persist session records in the `hiringiq-pilot-sessions` table with the following attributes:

| Field | Type | Description |
|---|---|---|
| `session_id` | String (PK) | UUID v7 assigned at magic-link generation |
| `applicant_id` | String | From the magic-link JWT |
| `campaign_id` | String | From the magic-link JWT |
| `org_id` | String | From the magic-link JWT (tenant isolation) |
| `state` | String | Current session state per D-SM1 |
| `state_reason` | String OR null | Sub-reason for PAUSED states (e.g., "ws_disconnect") |
| `state_changed_at` | ISO-8601 | Last state transition timestamp |
| `created_at` | ISO-8601 | Session creation timestamp |
| `shuffled_item_ids` | Array<String> | The per-applicant random order of item IDs (computed at session.started) |
| `item_states` | Map<item_id, "not_served" \| "served_unanswered" \| "served_answered"> | Per D-SM4 |
| `item_responses` | Map<item_id, { answer, submitted_at, correct?: boolean }> | The applicant's response per item |
| `pause_time_used_per_item` | Map<item_id, number> | Cumulative pause-time consumed per item (D-SM3) |
| `current_item_id` | String OR null | The item currently being displayed (null between items or in terminal states) |
| `persona_status` | String | "pending" / "approved" / "declined.<reason>" / "bypassed" |
| `persona_attempts_count` | number | Strike counter |

#### Scenario: Session record reflects state machine accurately
- **WHEN** a session reaches COMPLETED state
- **THEN** the session record has `state: "COMPLETED"`, `state_changed_at: <timestamp>`, `item_states` for every item in either `served_answered` or `served_unanswered` (never `not_served` for a COMPLETED session that ran to the end)
