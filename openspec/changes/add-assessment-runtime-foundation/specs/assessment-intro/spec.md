# Assessment Intro

## ADDED Requirements

### Requirement: Intro screen content (D-IS1)

The system SHALL display the assessment intro screen as the fourth applicant-facing screen, sequenced after device check. The screen SHALL contain (top to bottom):
- Greeting: *"Welcome, {applicant_first_name}."*
- Assessment metadata card:
  - Campaign name (from the campaign record)
  - Question count + estimated duration formatted from `assessment_total_time_seconds` (campaign record, computed at campaign-create time per platform-shell item-bank D17 — read-only at runtime)
- Heading: *"Before we begin, a few things to know:"*
- Timer line (⏱): *"Some questions have a time limit. The timer will appear in the top-right corner when applicable."*
- Webcam line (📷): *"Your webcam needs to stay on and you need to remain visible in frame throughout the assessment. If we lose sight of you, the assessment will pause until we can see you again."*
- Microphone line (🎤): *"Your microphone needs to remain enabled."*
- Divider
- Consent checkbox (D-IS2)
- Divider
- Recruiter contact line (D-IS3)
- Two CTAs: "I don't wish to continue" (bottom-left, text button) + "Begin Assessment" (bottom-right, filled button — disabled until consent checked)

The wording above is EXACT. The microphone line is intentionally neutral — does NOT promise consequences either way, since mic_disabled is a persistent warning, not a pause trigger (D-SM6).

The screen DOES NOT contain the deprecated "Repeated or extended issues will be noted in the recruiter's review" language from the original brief. Removed per the same telegraphing-avoidance reasoning as the silent integrity-signal events.

#### Scenario: Intro screen content matches exactly
- **WHEN** the intro screen renders for applicant `Jane Smith` on campaign `Senior Engineer 2026 Q3` (20 questions, 30-minute duration)
- **THEN** the page contains the greeting `"Welcome, Jane."`, the assessment card showing the campaign name and `20 questions · ~30 minutes`
- **AND** the page contains all four documented bullet/icon lines verbatim
- **AND** the page does NOT contain the string "noted in the recruiter's review"

### Requirement: Consent checkbox is separate from landing T&C (D-IS2)

The intro screen SHALL display a consent checkbox with the wording:

> *"I understand my webcam will be recorded, my microphone will be monitored for activity, and my browser activity will be captured for integrity review."*

Followed by a link to *"View privacy notice"*.

The checkbox SHALL be unchecked by default. The Begin Assessment CTA SHALL be disabled until the checkbox is checked.

The wording deliberately differentiates:
- Webcam **will be recorded** (selfie in Phase 2, rrweb video in Phase 2, face-api.js stream in Phase 2)
- Microphone **will be monitored for activity** (VAD in Phase 2, NOT recorded — audio waveforms never stored)
- Browser activity **will be captured** (audit-log events)

The differentiated language is intentional and accurate to the system's actual data handling.

#### Scenario: Consent default unchecked, Begin Assessment disabled
- **WHEN** the intro screen renders
- **THEN** the consent checkbox is unchecked
- **AND** the Begin Assessment CTA is disabled

#### Scenario: Checking consent enables Begin Assessment
- **WHEN** the applicant checks the consent checkbox
- **THEN** the Begin Assessment CTA becomes enabled
- **AND** the audit event `consent.checkbox_checked` fires

#### Scenario: Unchecking consent re-disables Begin Assessment
- **WHEN** the applicant unchecks the consent checkbox after previously checking it
- **THEN** the Begin Assessment CTA returns to disabled state
- **AND** the audit event `consent.checkbox_unchecked` fires

### Requirement: Recruiter contact + accommodations line (D-IS3)

The system SHALL display a recruiter contact line near the bottom of the intro screen:

> *"Have a question or need an accommodation? Contact {recruiter_first_name} {recruiter_last_name} at {recruiter_email}."*

The recruiter name + email are pulled from the campaign record. The word "accommodation" is required wording for ADA / accessibility compliance — applicants who need extra time, screen reader support, or alternate input methods have an explicit channel to request it before starting.

#### Scenario: Recruiter contact line displays accurate info
- **WHEN** the intro screen renders for a campaign owned by `John Smith <jsmith@foobarcorp.com>`
- **THEN** the page contains the exact text `"Have a question or need an accommodation? Contact John Smith at jsmith@foobarcorp.com."`

### Requirement: Decline flow with two-step confirmation (D-IS4)

The system SHALL provide an "I don't wish to continue" button at the bottom-left of the intro screen. The button SHALL be styled as a text button (lower visual weight than the primary Begin Assessment CTA).

Clicking the button SHALL open a confirmation modal with the wording:

> **Are you sure?**
>
> If you continue without taking this assessment, you won't be able to come back to it. Your recruiter will be notified that you chose not to proceed.
>
> [Continue with Assessment]   [Yes, I'm sure]

Clicking "Yes, I'm sure" SHALL:
- Emit `applicant.declined.confirmed` audit event
- Transition the session state to **DECLINED** (new terminal state)
- Redirect the applicant to the decline goodbye page

The DECLINED terminal state SHALL NOT trigger the integrity analyzer. No events were captured during an assessment that didn't happen.

#### Scenario: Modal cancel returns to intro
- **WHEN** the applicant clicks "I don't wish to continue" → "Continue with Assessment" in the modal
- **THEN** the modal closes and the applicant remains on the intro screen
- **AND** the session state remains in NOT_STARTED (or whatever the pre-Begin state is)
- **AND** the audit event `applicant.declined.cancelled` fires

#### Scenario: Modal confirm transitions to DECLINED
- **WHEN** the applicant clicks "I don't wish to continue" → "Yes, I'm sure" in the modal
- **THEN** the session state transitions to DECLINED
- **AND** the audit event `applicant.declined.confirmed` fires
- **AND** the applicant is redirected to the decline goodbye page

### Requirement: Decline goodbye page (D-IS4)

The system SHALL display a goodbye page after a confirmed decline. The page SHALL contain:

> *"Thank you for considering this assessment. {Recruiter name} has been notified that you've chosen not to proceed. If you'd like to reach out, you can contact them at {recruiter_email}."*

The page SHALL have no call-to-action buttons. It is reached only from a DECLINED-state session.

#### Scenario: Goodbye page renders with recruiter info
- **WHEN** an applicant lands on the goodbye page after declining for a campaign owned by `John Smith <jsmith@foobarcorp.com>`
- **THEN** the page reads `"Thank you for considering this assessment. John Smith has been notified that you've chosen not to proceed. If you'd like to reach out, you can contact them at jsmith@foobarcorp.com."`

### Requirement: Begin Assessment transitions state and goes instant (D-UI8)

Clicking Begin Assessment SHALL:
- Transition the session state NOT_STARTED → IN_PROGRESS server-side
- Navigate to the assessment interface
- The transition SHALL be instant (no loading spinner) — pre-warming happens on the intro screen mount (D-IS5 + D-UI8)

**Pre-warming on intro mount** (background, transparent to applicant):
- WebSocket is already connected (opened on landing per D-IS5)
- The first item is fetched via `GET /sessions/<id>/items/next` and held in memory
- face-api.js model bytes are downloaded (Phase 2 will use them; Phase 1 only downloads them)
- rrweb recorder is primed (Phase 2 will use; Phase 1 stub)

If pre-warming is incomplete when Begin is clicked (slow network), a single-frame "Starting…" overlay shows until ready. Beyond 5 seconds, a proper loading indicator surfaces.

#### Scenario: Instant transition in normal conditions
- **WHEN** the applicant clicks Begin Assessment with all pre-warming complete
- **THEN** the assessment interface renders within ~50ms with the first item already on screen
- **AND** the session state in DDB is IN_PROGRESS
- **AND** the audit event `session.started` fires

#### Scenario: Slow network shows brief loading state
- **WHEN** the applicant clicks Begin Assessment before the first item has been fetched
- **THEN** a "Starting…" overlay displays for up to 5 seconds
- **AND** once pre-warming completes, the assessment interface renders normally

### Requirement: Audit events on intro screen

The system SHALL emit the following silent audit events at the appropriate moments on the intro screen:
- `session.intro_screen_viewed` on mount
- `consent.checkbox_checked` / `consent.checkbox_unchecked`
- `privacy_notice.opened` if the applicant clicks the View privacy notice link
- `applicant.declined.initiated` (Decline button clicked, modal opens)
- `applicant.declined.cancelled` (modal canceled)
- `applicant.declined.confirmed` (final confirmation, state transitions to DECLINED)
- `session.started` (Begin Assessment clicked, state transitions to IN_PROGRESS)

#### Scenario: Happy-path event sequence on intro screen
- **WHEN** an applicant lands on the intro screen, opens the privacy notice, accepts the consent checkbox, and clicks Begin Assessment
- **THEN** the audit-events table contains events in this order: `session.intro_screen_viewed`, `privacy_notice.opened`, `consent.checkbox_checked`, `session.started`

#### Scenario: Decline event sequence
- **WHEN** an applicant lands on the intro screen, clicks "I don't wish to continue," then confirms "Yes, I'm sure"
- **THEN** the audit-events table contains events in this order: `session.intro_screen_viewed`, `applicant.declined.initiated`, `applicant.declined.confirmed`
- **AND** no `session.started` event fires for this session
