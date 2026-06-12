# Applicant Consent Landing

## ADDED Requirements

### Requirement: Consent landing is the first screen after magic-link validation

The system SHALL display the consent landing page as the first applicant-facing screen after successful magic-link validation. The page SHALL contain (top to bottom):
- Org name from the campaign record
- Recruiter intro line: *"You've been invited by {recruiter_first_name} {recruiter_last_name} to take a skills assessment for {campaign_name}."*
- Assessment overview list: estimated duration, webcam/mic required, government-ID verification required
- Permission status card showing live state for camera + microphone
- T&C checkbox with links to *"Terms & Conditions"* and *"Privacy Notice"*
- Continue CTA (bottom-right) — enabled only when both permissions are granted AND T&C is checked

#### Scenario: Valid landing renders all documented sections
- **WHEN** the consent landing page renders for a campaign with `org_name="FooBarCorp"`, `recruiter="John Smith"`, `campaign_name="Senior Engineer 2026 Q3"`, `assessment_total_time_seconds=1800`
- **THEN** the rendered page contains the literal text `"You've been invited by John Smith to take a skills assessment for Senior Engineer 2026 Q3."`
- **AND** the assessment overview shows `~30 minutes` formatted from the 1800-second total

### Requirement: Permission grants happen on landing, not later

The system SHALL request camera + microphone permissions via `navigator.mediaDevices.getUserMedia({video: true, audio: true})` on landing-page mount. Permissions SHALL NOT be requested at the Persona step or later — Persona's $1.50/check would be incurred for an applicant who would have failed permission grants anyway, so we filter up-front.

If granted: the permission card shows ✓ Granted for both, and the T&C checkbox becomes interactable.

If denied: the card shows the denial state with inline help text: *"This assessment requires camera and microphone access. Please grant permission in your browser settings and refresh this page."* The Continue CTA remains disabled regardless of T&C state.

#### Scenario: Permissions granted enables T&C interaction
- **WHEN** the applicant grants both camera and microphone permissions
- **THEN** the permission status card shows `Camera access ✓ Granted` and `Microphone access ✓ Granted`
- **AND** the T&C checkbox becomes interactable
- **AND** the system emits `permissions.granted` with payload `{ camera: true, mic: true }`

#### Scenario: Permission denied blocks Continue regardless of T&C
- **WHEN** the applicant grants camera but denies microphone
- **AND** then checks the T&C checkbox
- **THEN** the Continue CTA remains disabled
- **AND** the page shows the documented permission-denied help text
- **AND** the system emits `permissions.denied` with payload `{ camera: true, mic: false }`

### Requirement: WebSocket opens on landing-page mount (D-IS5)

The system SHALL open the WebSocket connection to the telemetry-service immediately on consent landing mount. The connection SHALL include the magic-link JWT in the query string for the `$connect` Lambda to validate. This early open ensures (a) connectivity issues surface before the applicant invests time, (b) intro-screen and pre-assessment events flow through the same telemetry pipeline as in-assessment events.

#### Scenario: WS connection established at landing
- **WHEN** the consent landing renders successfully
- **THEN** a WebSocket connection appears in the `hiringiq-pilot-ws-connections` DDB table with the documented schema
- **AND** the connection is bound to the session_id from the JWT

#### Scenario: WS connection failure displays an error state
- **WHEN** the WebSocket connection to telemetry-service fails to open at landing
- **THEN** the page displays an inline error: *"We're having trouble connecting. Please refresh the page or check your internet connection."*
- **AND** Continue remains disabled until reconnection succeeds

### Requirement: T&C checkbox is separate from D-IS2 integrity consent

The landing-page T&C checkbox SHALL cover the general Terms & Conditions for participating in the assessment (e.g., honest participation, no cheating, no sharing of content). The integrity-recording consent (D-IS2 — *"I understand my webcam will be recorded, my microphone will be monitored for activity, and my browser activity will be captured for integrity review."*) is a SEPARATE checkbox on the assessment intro screen, NOT on the landing.

The two consents are intentionally split:
- T&C = legal terms for taking the assessment
- Integrity consent = specific acknowledgement of behavioral capture

#### Scenario: Both consents required before any item is shown
- **WHEN** the applicant checks the landing T&C, clicks Continue, completes Persona + device check, but does NOT check the intro-screen integrity consent
- **THEN** the Begin Assessment CTA on the intro screen is disabled
- **AND** the applicant cannot proceed to the first item

### Requirement: Window Management API permission is NOT requested at landing

The system SHALL NOT request Multi-Screen Window Placement API permission at the consent landing (or any other point at MVP). The Window Management permission prompt (*"This site wants to view the position and size of all your displays"*) is confusing and would raise red flags. Multi-monitor detection at MVP uses the no-permission `window.screen` API and accepts reduced fidelity. Revisit post-pilot.

#### Scenario: No permission prompt for Window Management
- **WHEN** the consent landing renders
- **THEN** no `window.getScreens()` call is attempted
- **AND** the browser does NOT show a Multi-Screen Window Placement permission prompt

### Requirement: Audit events on landing

The system SHALL emit the following silent audit events at the appropriate moments on the consent landing page:
- `session.landing_page_viewed` on initial render
- `permissions.granted` / `permissions.denied` after the `getUserMedia` resolution, with `{ camera: boolean, mic: boolean }` payload
- `terms.accepted` when the T&C checkbox transitions to checked
- `terms.opened` when the applicant clicks the "Terms & Conditions" link
- `privacy_notice.opened` when the applicant clicks the "Privacy Notice" link
- `session.landing_continue_clicked` on Continue button click

#### Scenario: Happy-path event sequence
- **WHEN** an applicant lands, grants permissions, opens the privacy notice, accepts T&C, and clicks Continue
- **THEN** the audit-events table contains events in this order: `session.landing_page_viewed`, `permissions.granted`, `privacy_notice.opened`, `terms.accepted`, `session.landing_continue_clicked`
