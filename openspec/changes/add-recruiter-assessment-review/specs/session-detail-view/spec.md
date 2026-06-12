# Session Detail View

## ADDED Requirements

### Requirement: New route + page layout (D-R1, D-R7)

The system SHALL provide a new React route at `/campaigns/:campaignId/sessions/:sessionId` in the platform-shell dashboard. The page SHALL render the following sections in order (top to bottom):

1. **Header:** campaign name + applicant name + session completion timestamp
2. **IntegritySummaryCard** (see separate capability spec): score + behavioral summary + flagged moments
3. **TimelineScrubber + RrwebPlaybackViewer:** side-by-side on viewports ≥1280px wide; stacked on narrower viewports
4. **RawAuditLogViewer:** default collapsed, expandable
5. **Action footer:** "Re-analyze with latest prompt" button (D-R9)

All four child views (summary, timeline, player, log) SHALL share a `currentTimestamp` state at the page level. Updating the state in one view propagates to the others (D-R7).

#### Scenario: Page renders all sections
- **WHEN** a recruiter navigates to a valid session URL
- **THEN** the page displays all 5 documented sections in order
- **AND** the IntegritySummary data, timeline, replay, audit log all populate

#### Scenario: Linked navigation across child views
- **WHEN** the recruiter clicks a flagged_moment "Jump to this moment" button in the IntegritySummaryCard
- **THEN** the TimelineScrubber's position indicator moves to that timestamp
- **AND** the RrwebPlaybackViewer seeks to that timestamp
- **AND** the RawAuditLogViewer scrolls to and highlights events at that timestamp

### Requirement: Page fetches session data with org-isolation enforcement

The page SHALL call `GET /api/sessions/<id>` on mount via the runtime's HTTPS endpoint. The request SHALL include the recruiter's Cognito JWT in the Authorization header. The backend validates the JWT's `org_id` claim matches the session's `org_id`; on mismatch, the backend returns 403 and the page displays "You do not have access to this session" with no data leak.

#### Scenario: Same-org request succeeds
- **WHEN** a recruiter in org O1 navigates to a session belonging to org O1
- **THEN** the backend returns 200 with the session data
- **AND** the page renders normally

#### Scenario: Cross-org request returns 403
- **WHEN** a recruiter in org O1 attempts to navigate to a session belonging to org O2
- **THEN** the backend returns 403 Forbidden
- **AND** the page displays the access-denied message
- **AND** no session data is exposed in the response payload

### Requirement: Re-analyze button enqueues new analyzer SQS message (D-R9)

The system SHALL provide a "Re-analyze with latest prompt" button in the action footer. Clicking the button:
- Shows a confirmation modal: *"Re-analyze this session? A new IntegritySummary will be generated using the current analyzer prompt and model."*
- On confirm, calls `POST /api/sessions/<id>/re-analyze`
- On success, displays a snackbar: *"Re-analysis in progress. The summary will update in approximately 30 seconds."*
- Auto-polls the IntegritySummary endpoint every 10 seconds for up to 2 minutes, refreshing the IntegritySummaryCard when a new analyzer_version is detected

Rate limit: 3 re-runs per 24h per session (SSM-configurable). If exceeded, the backend returns 429 and the dashboard displays *"This session has been re-analyzed the maximum number of times in the past 24 hours."*

#### Scenario: Re-analyze flow succeeds
- **WHEN** the recruiter clicks Re-analyze, confirms, and the backend returns success
- **THEN** the snackbar displays the documented message
- **AND** within ~30 seconds, the IntegritySummary card updates with a new analyzer_version

#### Scenario: Rate limit exceeded
- **WHEN** the recruiter attempts a 4th re-run within 24 hours
- **THEN** the backend returns 429
- **AND** the dashboard displays the rate-limit message

### Requirement: Page handles analyzer_failed state gracefully

The system SHALL detect when the IntegritySummary has `status: "analyzer_failed"` and display appropriate placeholder content:
- The IntegritySummaryCard shows "Analysis unavailable for this session — request manual re-analysis"
- The TimelineScrubber renders empty (no segments to display)
- The RrwebPlaybackViewer still functions (the recording exists even if analysis failed)
- The RawAuditLogViewer still functions (the audit log exists)

The "Re-analyze with latest prompt" button is prominently displayed in this state.

#### Scenario: Analyzer-failed session displays appropriate placeholders
- **WHEN** the session's IntegritySummary has `status: "analyzer_failed"`
- **THEN** the IntegritySummaryCard shows the analysis-unavailable placeholder
- **AND** the timeline scrubber is empty
- **AND** the rrweb player and audit log are still functional
- **AND** the Re-analyze button is prominently displayed
