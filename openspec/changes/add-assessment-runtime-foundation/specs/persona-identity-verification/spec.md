# Persona Identity Verification

## ADDED Requirements

### Requirement: usePersonaVerification feature flag controls the entire Persona step (D-P1)

The system SHALL read the SSM parameter `/hiringiq/pilot/runtime/use-persona-verification` (boolean) at session-config time. The runtime returns this flag value to the client via the `GET /assess/config` endpoint.

When `usePersonaVerification = false`:
- The client SHALL skip the entire Persona screen at session-start (no Persona UI rendered)
- The backend SHALL NOT call Persona's Inquiry API
- The session record SHALL be updated with `{ persona_status: "bypassed", bypassed_at: <timestamp> }`
- An audit event `persona.bypassed` SHALL be emitted with payload `{ reason: "flag_disabled" }`
- The applicant SHALL transition directly from consent landing → device check

When `usePersonaVerification = true` (default):
- The client SHALL render the Persona Inquiry flow per the SDK
- The applicant SHALL be required to complete the Persona inquiry before proceeding to device check

#### Scenario: Flag disabled skips Persona screen entirely
- **WHEN** the SSM parameter is set to `false`
- **AND** the applicant clicks Continue on the consent landing
- **THEN** the next screen is the device check, NOT the Persona screen
- **AND** the audit log contains `persona.bypassed` with `reason: "flag_disabled"`
- **AND** no Persona API call has occurred (Persona dashboard shows no inquiries)

#### Scenario: Flag enabled launches Persona inquiry
- **WHEN** the SSM parameter is set to `true` (default)
- **AND** the applicant clicks Continue on the consent landing
- **THEN** the next screen renders Persona's Inquiry SDK with the documented template ID and inquiry-reference equal to the applicant_id
- **AND** the audit log contains `persona.attempted`

### Requirement: API errors do not count toward 3-strike (D-P2)

The system SHALL handle Persona API errors (timeout, 5xx, network failure) without incrementing the 3-strike counter. When `usePersonaVerification = true` AND Persona's API is unreachable:
- The system SHALL log the error to CloudWatch and emit a `persona.api_error` audit event
- The applicant SHALL see the message: *"We're having trouble verifying your ID right now. Please try again in a moment."*
- The applicant SHALL be permitted to retry the inquiry with the SAME information (Persona supports inquiry resumption via inquiry-id)
- API errors SHALL NOT count toward the 3-strike counter
- Retries SHALL be rate-limited to 5 per 10-minute window per applicant to prevent infinite hammering

#### Scenario: API error allows unlimited retries (within rate limit)
- **WHEN** Persona's API times out twice in a row during an applicant's inquiry
- **THEN** the applicant sees the "trouble verifying" message both times
- **AND** the 3-strike counter remains at 0
- **AND** two `persona.api_error` events appear in the audit log

#### Scenario: Rate limit blocks excessive retries
- **WHEN** an applicant attempts a 6th retry within 10 minutes
- **THEN** the system blocks the retry attempt and displays a "please wait" message
- **AND** an audit event records the rate-limit hit

### Requirement: Reason-code routing per D-P3

When `usePersonaVerification = true` AND Persona returns a `declined` or `failed` status, the system SHALL route the applicant to one of three flows based on the reason code:

| Persona reason code | Applicant message | Retry path | Counts toward 3-strike? |
|---|---|---|---|
| `id_expired`, `id_damaged`, `id_unsupported_country` | "The ID you submitted couldn't be accepted. Please try again with a different government-issued ID." | Restart inquiry with new ID upload | Yes |
| `selfie_not_matching_id`, `liveness_check_failed`, `face_not_detected` | "We couldn't match your photo to the ID. Please retake your photo and try again." | Restart inquiry with new selfie (ID upload preserved) | Yes |
| `id_country_blacklist`, sanctions hit | "Verification unsuccessful. Please contact the recruiter." | None — straight to Unable-to-Verify | N/A |
| Other / generic decline | "Verification unsuccessful. Please try again." | Restart inquiry | Yes |

The audit log SHALL preserve the EXACT reason code (high-fidelity record for the recruiter), while the applicant message is intentionally vague to avoid information-disclosure attacks.

#### Scenario: id_expired routes to "try a different ID"
- **WHEN** Persona returns `declined` with reason `id_expired`
- **THEN** the applicant sees "The ID you submitted couldn't be accepted. Please try again with a different government-issued ID."
- **AND** the audit log records `persona.declined.id_expired`
- **AND** the 3-strike counter increments by 1

#### Scenario: selfie_not_matching_id allows retake without re-uploading ID
- **WHEN** Persona returns `declined` with reason `selfie_not_matching_id`
- **THEN** the applicant sees "We couldn't match your photo to the ID. Please retake your photo and try again."
- **AND** the Persona inquiry is resumed (not restarted from scratch — ID upload preserved)
- **AND** the audit log records `persona.declined.selfie_not_matching_id`

#### Scenario: Sanctions hit goes straight to Unable-to-Verify
- **WHEN** Persona returns `declined` with reason `id_country_blacklist`
- **THEN** the applicant transitions immediately to Unable-to-Verify regardless of strike count
- **AND** the applicant sees "Verification unsuccessful. Please contact the recruiter." with the recruiter contact line
- **AND** the audit log records `persona.declined.id_country_blacklist`

### Requirement: 3-strike counter and Unable-to-Verify terminal state (D-P3)

The system SHALL maintain a per-session strike counter incrementing on Persona-evaluated-and-rejected outcomes. At 3 strikes:
- The applicant transitions to the Unable-to-Verify terminal state for the session
- The applicant sees: *"Verification failed. Please reach out to your recruiter — {recruiter_first_name} {recruiter_last_name} at {recruiter_email}."*
- The session state remains in the pre-assessment portion (the assessment never starts)
- The audit log records the full sequence of attempts with their reason codes

The Unable-to-Verify state is reached via either:
- Three accumulated strikes (any combination of reason codes that count)
- An immediate sanctions-hit reason code (no strike accumulation needed)

#### Scenario: Three strikes triggers Unable-to-Verify
- **WHEN** an applicant fails Persona three times with `id_expired`, `id_damaged`, `selfie_not_matching_id` (all counting)
- **THEN** after the third attempt, the applicant transitions to Unable-to-Verify
- **AND** the applicant sees the documented message with the recruiter's name and email
- **AND** the assessment never starts

### Requirement: Persona webhook handler (server-side)

The system SHALL expose an HTTPS endpoint `POST /webhooks/persona` registered with Persona for inquiry-completion notifications. The handler SHALL:
- Verify the webhook HMAC signature using Persona's signing secret (loaded from Secrets Manager)
- Parse the inquiry result
- Update the applicant record with `{ persona_inquiry_id, status, confidence, attempts_count, last_reason_code }`
- Apply the strike-counter and reason-code routing logic
- Emit the appropriate audit event(s)

The handler SHALL NOT fetch the selfie image from Persona's Inquiry API at Phase 1 — that retrieval (D-P4) is deferred to Phase 2.

#### Scenario: Webhook signature verification rejects forged requests
- **WHEN** a request is posted to `/webhooks/persona` with a forged or missing HMAC signature
- **THEN** the handler returns HTTP 401 and does NOT update any applicant record
- **AND** an alert event is logged for the security team

#### Scenario: Webhook persists Persona result to applicant record
- **WHEN** Persona posts a webhook for a successful inquiry with `confidence: 0.97`
- **THEN** the applicant record is updated with `{ persona_inquiry_id: <id>, status: "approved", confidence: 0.97, attempts_count: 1 }`
- **AND** the audit log records `persona.succeeded`

### Requirement: Persona persists only pass/fail + confidence at Phase 1 (D-P4 deferral)

At Phase 1, the system SHALL NOT fetch the selfie image, ID images, or any other asset from Persona's Inquiry API. Only the inquiry RESULT (`status`, `confidence`, `attempts_count`, `last_reason_code`, `persona_inquiry_id`) is persisted to the applicant record.

The face-api.js enrollment baseline does not exist at Phase 1 (no consumer for it). Persona-selfie retrieval lands in Phase 2 along with face-api.js integration.

#### Scenario: No biometric asset retrieval at Phase 1
- **WHEN** an applicant completes Persona at Phase 1
- **THEN** no S3 object is created in any biometric storage bucket
- **AND** the applicant record contains only the pass/fail + confidence fields documented above
- **AND** no GET request was made to Persona's `/api/v1/inquiries/{id}` endpoint
