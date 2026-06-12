# Magic Link Validation

## ADDED Requirements

### Requirement: Sequential validation chain at every page load

The system SHALL execute the following sequential validation checks at every magic-link page load. A failure at any step SHALL route the applicant to the corresponding terminal page and SHALL NOT proceed to subsequent checks:

1. **JWT signature** — verify the magic-link JWT signature using the HMAC-SHA256 key from Secrets Manager. Failure → generic "Invalid invitation" page.
2. **JWT TTL** — verify the JWT was issued within the 15-day window. Failure → "Invitation expired" page (D-MAG2).
3. **Campaign active** — verify the referenced campaign is in the Active state per the platform-shell's campaign-lifecycle model. Failure → "Assessment no longer active" page (D-MAG3).
4. **Viewport size** — verify the current viewport is ≥1024px wide. Failure → "Use a larger device" page (D-DC5).
5. **Browser supported** — verify the browser is Chrome, Firefox, or Edge (desktop), with `navigator.mediaDevices.getUserMedia`, Web Audio API, and `MediaDevices.enumerateDevices` available. Failure → "Unsupported browser" page (D-DC6).
6. **Session state allows access** — verify the session is in a state that allows entry (NOT_STARTED, IN_PROGRESS, PAUSED.*, or ABANDONED). Other states route to their terminal pages (COMPLETED → completion screen if not expired; DECLINED → goodbye page; EXPIRED → expired page).

The order is intentionally JWT-first to prevent leaking campaign or session state to bad actors with invalid invitations.

#### Scenario: Valid magic link proceeds to consent landing
- **WHEN** the applicant clicks a magic link with a valid JWT, an active campaign, on a 1440px desktop Chrome browser, and the session is in NOT_STARTED state
- **THEN** the validation chain completes successfully and the applicant lands on the consent landing page

#### Scenario: Invalid JWT short-circuits before campaign check
- **WHEN** the applicant clicks a magic link whose JWT signature does not verify
- **THEN** the system displays the generic "Invalid invitation" page
- **AND** no information is leaked about whether the campaign exists or its state

#### Scenario: Validation order is JWT → campaign → viewport → browser
- **WHEN** an applicant clicks a magic link that is expired AND points to a closed campaign AND is on a mobile device AND in Safari
- **THEN** the displayed page is the "Invitation expired" page (the earliest failure in the chain)
- **AND** the audit log records exactly one event (`session.access_attempted_with_expired_link`) — subsequent checks were not run

### Requirement: Invitation Expired page (D-MAG2)

The system SHALL display a page with the following content when the JWT TTL check fails:

> Your assessment invitation has expired. If you're still interested in this position, please contact {recruiter_first_name} {recruiter_last_name} at {recruiter_email} to request a new invitation.

The page SHALL have no call-to-action buttons. The page SHALL emit a silent audit event `session.access_attempted_with_expired_link` with payload `{ jwt_issued_at, time_elapsed_days, campaign_id, applicant_id }`.

#### Scenario: Page renders with recruiter contact line
- **WHEN** the applicant lands on the Invitation Expired page for a campaign owned by `John Smith <jsmith@foobarcorp.com>`
- **THEN** the page reads `"Your assessment invitation has expired. If you're still interested in this position, please contact John Smith at jsmith@foobarcorp.com to request a new invitation."`

### Requirement: Assessment No Longer Active page (D-MAG3)

The system SHALL display a page with the following content when the campaign-active check fails:

> This assessment is no longer active. Please contact {recruiter_first_name} {recruiter_last_name} at {recruiter_email} for more information.

The page SHALL have no call-to-action buttons. The page SHALL emit a silent audit event `session.access_attempted_to_closed_campaign` with payload `{ campaign_id, campaign_closed_at, applicant_id }`.

The page SHALL trigger on either of these conditions:
- The recruiter manually closed the campaign via the dashboard
- The campaign's `end_at` reached past automatically per the platform-shell's campaign-lifecycle proposal #8

#### Scenario: Campaign closed by recruiter routes to the no-longer-active page
- **WHEN** the recruiter closes a campaign at 14:00 UTC
- **AND** the applicant clicks their valid magic link at 14:05 UTC
- **THEN** the page displays the documented "Assessment no longer active" content
- **AND** the audit event records `campaign_closed_at: "2026-XX-XXT14:00:00Z"`

### Requirement: Use a Larger Device page (D-DC5)

The system SHALL display a page with the following content when the viewport size check fails:

> This assessment requires a desktop or laptop computer. Please return to this email on a larger device to begin your assessment.

The page SHALL have no call-to-action buttons. The page SHALL emit a silent audit event `session.access_attempted_from_small_screen` with payload `{ viewport_width, viewport_height, user_agent }`.

#### Scenario: Mobile viewport triggers the device block
- **WHEN** the applicant clicks the magic link on a phone with viewport width 414px
- **THEN** the page displays the documented "larger device" content
- **AND** the audit event payload contains `viewport_width: 414`

### Requirement: Unsupported Browser page (D-DC6)

The system SHALL display a page with the following content when the browser-support check fails:

> This assessment requires Chrome, Firefox, or Microsoft Edge on a desktop or laptop. Please reopen this invitation in a supported browser.

The page SHALL have no call-to-action buttons. The page SHALL emit a silent audit event `session.access_attempted_from_unsupported_browser` with payload `{ user_agent, missing_features: string[] }`.

The check SHALL use BOTH user-agent string detection AND feature detection. A browser passing the user-agent check but failing a feature detection (e.g., an older Chrome without getUserMedia) is treated as unsupported.

#### Scenario: Safari is rejected even on desktop
- **WHEN** the applicant clicks the magic link in Safari on macOS
- **THEN** the page displays the documented "Unsupported Browser" content
- **AND** the audit event payload contains the Safari user-agent string

#### Scenario: Older Chrome without required APIs is rejected
- **WHEN** the applicant clicks the magic link in Chrome 80 (no full getUserMedia support)
- **THEN** the page displays the documented "Unsupported Browser" content
- **AND** the audit event payload contains `missing_features: ["getUserMedia"]` (or similar)

### Requirement: Magic-link JWT structure (matches platform-shell proposal #11)

The JWT payload SHALL contain the following claims, matching the platform-shell's `add-applicant-assessment-landing` (proposal #11) exactly:

| Claim | Type | Value |
|---|---|---|
| `aid` | string | applicant UUID v7 |
| `cid` | string | campaign UUID v7 |
| `oid` | string | org UUID v7 |
| `iat` | number | Unix seconds (issuance) |
| `exp` | number | `min(iat + 15 × 86400, campaign.end_at)` — 15-day cap per D-MAG1, ceiling at campaign natural end |
| `iss` | string | `"hiringiq-pilot"` |
| `aud` | string | `"applicant-assessment"` |

The JWT SHALL be signed with HS256 using the SHARED platform-shell secret at `hiringiq/<env>/applicant-magic-link-secret`. The JWT SHALL be opaque to the client (the React app does not parse the JWT; the server validates and returns the relevant data via `GET /assess?token=<jwt>`).

The `session_id` is NOT a JWT claim. The session is identified server-side by the `{oid, cid, aid}` tuple — the runtime's `assessment-service` looks up (or lazily creates) the session record on the first request after JWT validation.

#### Scenario: JWT contains all documented claims with platform-shell-aligned names
- **WHEN** a magic link is generated for applicant `A1` on campaign `C1` (ending Feb 28) for org `O1` on Jan 5
- **THEN** the resulting JWT decodes to a payload containing `aid: A1`, `cid: C1`, `oid: O1`, `iss: "hiringiq-pilot"`, `aud: "applicant-assessment"`, `iat` (Jan 5), `exp` (Jan 20 — the 15-day cap, EARLIER than Feb 28)

#### Scenario: TTL caps at campaign end when 15 days exceeds it
- **WHEN** a magic link is generated on Jan 5 for a campaign ending Jan 10
- **THEN** the resulting JWT's `exp` is Jan 10 (campaign.end_at, EARLIER than iat+15-days)

#### Scenario: Tampered JWT fails validation
- **WHEN** an attacker modifies the `aid` claim in the JWT body and re-encodes the token without re-signing
- **THEN** the JWT signature verification step fails and the applicant is routed to the generic "Invalid invitation" page

### Requirement: Pre-flight blocks emit audit events even for failed validation

The system SHALL write audit events to the `hiringiq-pilot-audit-events` table for every magic-link access attempt that fails any pre-flight check. Events SHALL be written even though no full session exists yet (the session_id is derived from the JWT's `{oid, cid, aid}` tuple, which the events reference).

#### Scenario: Expired magic link still produces an audit-log entry
- **WHEN** an applicant clicks an expired magic link
- **THEN** an entry appears in the audit-events table at PK = `SESSION#<session_id>`, event_type = `session.access_attempted_with_expired_link`
- **AND** the entry is queryable by the recruiter (the recruiter dashboard's Phase 3 capabilities can surface it)
