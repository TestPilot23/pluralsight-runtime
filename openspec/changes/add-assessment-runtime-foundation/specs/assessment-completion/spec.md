# Assessment Completion

## ADDED Requirements

### Requirement: Completion screen content (D-CS1)

The system SHALL display the completion screen when the session transitions to the COMPLETED terminal state. The screen SHALL contain:
- Animated success ✓ icon (MD3 filled green checkmark with ~600ms scale-up animation on mount)
- *"Thank you for completing your assessment."*
- *"{recruiter_first_name} will be reviewing your submission and will reach out shortly with the next steps."*
- *"Have a question? Contact {recruiter_first_name} {recruiter_last_name} at {recruiter_email}."*
- *"You may now close this window."*

The screen SHALL NOT contain:
- An assessment summary (questions answered, time taken). Applicant doesn't need this data.
- "Your responses are being analyzed" language. The analyzer is for the recruiter, not the applicant.
- A download or copy of the applicant's submission. Deferred to v2.
- A "close window" button — just a text instruction.
- An auto-close countdown.

#### Scenario: Completion screen renders documented content
- **WHEN** an applicant completes their assessment and is routed to the completion screen
- **THEN** the page displays the animated ✓ icon (scaling up from 0 over ~600ms)
- **AND** the page displays the thank-you message, the recruiter review-and-reach-out message, the recruiter contact line, and "You may now close this window."

#### Scenario: Completion screen excludes deprecated content
- **WHEN** the completion screen renders
- **THEN** the page does NOT contain any text like "responses are being analyzed" or "assessment summary"
- **AND** the page does NOT contain a button labeled "close" or similar

### Requirement: Completion sequence + WS close (D-CS2)

The system SHALL execute the following sequence when an assessment completes:

```
1. Applicant submits last item (or timer expires on last item with auto-submit)
2. Server: state transitions IN_PROGRESS → COMPLETED, writes to sessions DDB table, emits SQS message that will trigger the integrity analyzer in Phase 2
3. Server: broadcasts "session_complete" WebSocket message to the client
4. Client: receives "session_complete" → routes to the completion screen
5. Client: completion screen renders. Animated ✓ scales up.
6. Client: emits session.completed event via WS (client view-timestamp captured)
7. Client closes the WS connection
8. Completion screen stays visible until the applicant closes the window manually
```

The analyzer trigger fires at step 2 (server state transition), NOT at step 6 (client event). Reason: if the client crashes between steps 4–6 (network failure, browser crash), the analyzer still runs because the server transitioned independently.

The client-side `session.completed` event is the audit-log record confirming the applicant saw the completion screen. It carries the client-side timestamp of view-time. The server's state-transition timestamp is a different (and earlier) moment. Both are useful for forensics.

**Phase 1 note:** the SQS message in step 2 is published, but the analyzer-service Lambda is a no-op stub. The analyzer's real implementation arrives in Phase 2; provisioning the message-publishing path now ensures Phase 2 only swaps the Lambda code without infrastructure churn.

**Client-initiated WS close.** In step 7, the CLIENT closes the WS, not the server. Cleaner sequencing — the client's last act is firing the event, then closing. Server-initiated close would create a window where the client tries to emit and finds the connection gone.

#### Scenario: Last item submitted triggers full completion sequence
- **WHEN** an applicant submits the last item in their shuffled order
- **THEN** the server transitions session state to COMPLETED
- **AND** the server publishes a message to the analyzer SQS queue (Phase 1: no-op handler; Phase 2: triggers full analysis)
- **AND** the server broadcasts `session_complete` over the WS
- **AND** the client receives it, renders the completion screen, emits `session.completed`, closes the WS

#### Scenario: Analyzer SQS message is published even at Phase 1
- **WHEN** a session reaches COMPLETED at Phase 1
- **THEN** a message appears in the `hiringiq-pilot-analyzer-queue` SQS queue
- **AND** the analyzer-service Lambda is triggered, logs the invocation, and returns (no IntegritySummary produced at Phase 1)

### Requirement: 5-minute fallback for missing client confirmation (D-CS2)

If the server transitions a session to COMPLETED but never receives the client-side `session.completed` event within 5 minutes, the server SHALL emit a server-side audit event:

```
event_type: "session.completion_screen_not_confirmed"
payload: { session_id, completed_at, time_since_completion_seconds }
```

The analyzer (Phase 2) already ran on the COMPLETED transition; this fallback just records that the applicant didn't see the screen. Minor signal — useful for the recruiter to know.

#### Scenario: Client never confirms within 5 minutes
- **WHEN** the server transitions a session to COMPLETED
- **AND** the client closes the browser before the completion screen renders (network failure or browser crash)
- **AND** 5 minutes pass with no `session.completed` event received
- **THEN** the audit log records `session.completion_screen_not_confirmed`

### Requirement: Magic-link JWT stays valid post-COMPLETED (D-CS3)

The magic-link JWT SHALL remain valid until its natural expiry (15 days from issuance per D-MAG1), even after the session reaches COMPLETED. The applicant CAN refresh the completion screen during that window — useful for re-reading the recruiter's contact info.

When the JWT expires, refreshing the URL routes the applicant through the magic-link validation chain (per D-MAG1), which detects the expired JWT and redirects to the Invitation Expired page (D-MAG2). The applicant cannot resume a COMPLETED assessment after JWT expiry; they just see the expiry page.

#### Scenario: Applicant refreshes completion screen within JWT validity
- **WHEN** an applicant has completed their assessment 3 days ago and clicks the magic link again
- **THEN** the magic-link validation chain detects the COMPLETED session state
- **AND** the applicant is routed to the completion screen, NOT to the assessment interface
- **AND** the page renders with the documented thank-you content

#### Scenario: Applicant clicks completion magic link after JWT expiry
- **WHEN** an applicant who completed their assessment 16 days ago clicks the magic link
- **THEN** the JWT TTL check fails first in the validation chain
- **AND** the applicant is routed to the Invitation Expired page (D-MAG2) — NOT to the completion screen

### Requirement: COMPLETED state is terminal but link-readable

The COMPLETED state SHALL be a terminal state — no further transitions out of COMPLETED are possible. Specifically:
- COMPLETED → IN_PROGRESS is NOT allowed (the applicant cannot retake an assessment they already finished)
- COMPLETED → DECLINED is NOT allowed
- The session record stays in COMPLETED forever (until 180-day retention deletes the record per the Phase 2 lifecycle policy)

But the completion SCREEN remains accessible (read-only) while the magic-link JWT is valid. This is purely a UX read-mode, not a state change.

#### Scenario: COMPLETED → anything else is rejected
- **WHEN** code attempts to transition a session out of COMPLETED via the state-machine helper
- **THEN** the transition throws an "illegal state transition" error
- **AND** the session record remains in COMPLETED
