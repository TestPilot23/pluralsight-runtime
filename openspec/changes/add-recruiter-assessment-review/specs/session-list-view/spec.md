# Session List View

## ADDED Requirements

### Requirement: Campaign overview applicant rows display session-state chips

The system SHALL extend the campaign overview page's applicant table (from platform-shell proposal #9) with a session-state chip per applicant row. The chip displays one of: `Assigned`, `Invited`, `Read`, `In-Progress`, `Completed`, `Declined`, `Expired`, `Abandoned`. The first three (`Assigned`, `Invited`, `Read`) come from the platform-shell's existing `assessment_status` enum (proposal #9); `Declined` and `Expired` are added by this proposal corresponding to the runtime's session state machine terminal states (D-SM1, D-IS4); `In-Progress`/`Completed`/`Abandoned` are shared between the two state models. PAUSED.* states from the runtime collapse into `In-Progress` for display purposes.

#### Scenario: Applicant row shows session state
- **WHEN** the campaign overview page renders for a campaign with applicants in mixed states
- **THEN** each applicant row shows a chip with the applicant's current session state
- **AND** an applicant who has not yet started shows `Read` or `Invited`; an applicant mid-assessment shows `In-Progress`

### Requirement: Score badge for completed sessions

The system SHALL display a clickable IntegritySummary score badge on rows where the applicant's session is in COMPLETED or EXPIRED state. The badge displays the integer score (0-100) and is color-coded:
- Green if score ≥80 (threshold from SSM)
- Yellow if 50-79
- Red if <50

Clicking the badge navigates to the session detail page at `/campaigns/:campaignId/sessions/:sessionId`.

If the IntegritySummary is in `analyzer_failed` status (Phase 2 D-AL4 retry-exhaustion), the badge displays "Analysis unavailable" in neutral grey instead of a numeric score, still clickable.

#### Scenario: Completed session shows clickable score badge
- **WHEN** the row's session has `state: COMPLETED` and an IntegritySummary with `integrity_score: 85`
- **THEN** the row displays a green badge with "85"
- **AND** clicking the badge navigates to the session detail page

#### Scenario: Analyzer-failed session shows neutral badge
- **WHEN** the row's session has `state: COMPLETED` but the IntegritySummary has `status: "analyzer_failed"`
- **THEN** the badge displays "Analysis unavailable" in neutral grey
- **AND** clicking the badge still navigates to the session detail page

### Requirement: Filter + sort controls

The system SHALL provide column-header filter controls on the applicant table:
- Filter by session state (multi-select chips)
- Sort by integrity score (ascending or descending)

Filters and sort combinations work together (AND-filtered, then sorted).

#### Scenario: Filter to Completed + sort by score descending
- **WHEN** the recruiter filters to only COMPLETED rows and sorts by score descending
- **THEN** only COMPLETED rows display, ordered from highest score to lowest

### Requirement: "View session" navigation link

The system SHALL provide a "View session" action button or link on each applicant row whose session is in any state other than `not_yet_started`. The link navigates to `/campaigns/:campaignId/sessions/:sessionId`.

For applicants without an assessment session yet, the link is absent (or disabled).

#### Scenario: View session link navigates correctly
- **WHEN** the recruiter clicks "View session" on a completed applicant's row
- **THEN** the browser navigates to `/campaigns/<campaign_id>/sessions/<session_id>`
- **AND** the SessionDetail page renders with that session's data
