# Persona Selfie Enrollment

## ADDED Requirements

### Requirement: Persona selfie retrieval after successful inquiry (D-P4, D-I5)

The system SHALL retrieve the applicant's selfie from Persona's Inquiry API after a successful inquiry result. The retrieval SHALL happen server-side inside the applicant-service Lambda's Persona webhook handler.

**Process:**
1. The webhook handler receives a successful inquiry-completion event from Persona.
2. The handler calls `GET https://api.withpersona.com/api/v1/inquiries/{inquiry-id}` with the Persona API key (from Secrets Manager).
3. The response contains the inquiry's `verifications` array; the `Government ID` verification's `attributes.selfie-photo` field is a signed URL to the selfie image.
4. The handler downloads the image via the signed URL.
5. The image is uploaded to S3 at `hiringiq-pilot-biometric-raw/<org_id>/<applicant_id>/selfie.jpg` with KMS encryption.

The retrieval depends on Pluralsight's Persona contract being on the Hosted / API-based Flow (Q-PSL-1). If Embedded Flow is confirmed, this requirement does not apply — see the alternative-path note in proposal.md.

#### Scenario: Successful inquiry triggers selfie retrieval
- **WHEN** Persona posts a webhook for a successful inquiry on applicant `A1` in org `O1`
- **THEN** the applicant-service calls Persona's Inquiry API
- **AND** the selfie image is downloaded
- **AND** the S3 object `hiringiq-pilot-biometric-raw/O1/A1/selfie.jpg` exists with KMS encryption
- **AND** the audit log records `enrollment.photo_stored`

#### Scenario: API failure emits enrollment.deferred and allows the applicant to proceed
- **WHEN** Persona's Inquiry API is unreachable during the webhook handler invocation (timeout, 5xx)
- **THEN** the system logs the error to CloudWatch
- **AND** emits `enrollment.deferred` to the audit log with payload `{ reason, error_code }`
- **AND** the applicant proceeds to device check normally (Persona's verification verdict still applies)
- **AND** subsequent face-detection identity comparisons during assessment report "no enrollment, skipped"

### Requirement: Face-api.js embedding generation server-side

The system SHALL generate the face-api.js embedding from the downloaded selfie inside the applicant-service Lambda, server-side. The embedding is a 128-dim or 512-dim Float32Array depending on the recognition net variant used (`@vladmandic/face-api` library; specific net documented in implementation).

The embedding SHALL be persisted as a DDB attribute on the applicant record, encoded as a base64 Float32Array buffer. The embedding SHALL NOT be sent to the client — the recognition model runs server-side; the client never sees the embedding.

#### Scenario: Embedding is generated and persisted
- **WHEN** the selfie image is successfully uploaded to S3
- **THEN** the applicant-service loads the image, runs face detection + recognition, produces an embedding
- **AND** the embedding is base64-encoded and written to the applicant's DDB record as the attribute `face_enrollment_embedding`
- **AND** the audit log records `enrollment.embedding_computed`

#### Scenario: Embedding stable for the same image
- **WHEN** the same selfie image is processed twice independently
- **THEN** the resulting embeddings are identical within floating-point precision tolerance

#### Scenario: No-face-detected in selfie produces deferral
- **WHEN** the face detection step on the downloaded selfie finds zero faces (corrupt image, unusual lighting, etc.)
- **THEN** the system emits `enrollment.deferred` with `{ reason: "no_face_in_selfie" }`
- **AND** the applicant proceeds; identity comparison during assessment is skipped

### Requirement: Biometric storage isolation

The system SHALL store the Persona selfie ONLY in the dedicated biometric-raw S3 bucket (`hiringiq-pilot-biometric-raw/`). The selfie SHALL NEVER be:
- Written to the runtime-web static-asset bucket (CloudFront origin) — would make the photo publicly accessible
- Written to the rrweb-recordings bucket — different retention and access posture
- Logged to CloudWatch via Lambda print statements (image content is biometric data)

The embedding SHALL be stored ONLY in the applicant's DDB record's `face_enrollment_embedding` attribute. It SHALL NOT be:
- Logged to CloudWatch
- Cached in any non-DDB store
- Sent to the client via any API response

#### Scenario: Audit confirms no selfie content in CloudWatch logs
- **WHEN** the applicant-service processes a selfie + embedding for a test applicant
- **THEN** a grep of CloudWatch Logs for the test applicant's ID returns ONLY metadata-level entries (e.g., "Selfie retrieved for A1", "Embedding computed: 128 floats") and NO image bytes or embedding values

#### Scenario: Embedding is not exposed via any API endpoint
- **WHEN** the client requests `GET /api/applicants/<id>` or any other applicant-data endpoint
- **THEN** the response object does NOT contain the `face_enrollment_embedding` field
- **AND** the response is sanitized to exclude any biometric data

### Requirement: Persona API key in Secrets Manager

The system SHALL store Persona's API key in AWS Secrets Manager at `hiringiq/pilot/runtime/persona-api-key`. The applicant-service Lambda SHALL have IAM permission to read this secret; no other component (Lambdas, services) SHALL have access.

#### Scenario: API key access is scoped
- **WHEN** the applicant-service Lambda reads the Persona API key
- **THEN** the call succeeds
- **WHEN** any other Lambda (assessment, telemetry, analyzer) attempts the same call
- **THEN** the call returns AccessDenied
