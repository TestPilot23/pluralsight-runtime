# Biometric Storage

## ADDED Requirements

### Requirement: 180-day retention for all biometric artifacts (D-P5)

The system SHALL retain all biometric artifacts for 180 days post-assessment:
- Persona selfie image (raw)
- face-api.js embedding (Float32Array vector)
- rrweb session recording (per D-RR2)

After 180 days, automated S3 lifecycle policies + DDB TTL SHALL delete the artifacts. Per-tenant override is possible (customer requesting a shorter window) — handled via a tenant-specific lifecycle rule applied at apply-time.

180 days matches a typical 6-month review window for hiring decisions. Recruiters may revisit a near-miss candidate, do comparative analyses, or respond to a candidate inquiry within this window.

#### Scenario: Lifecycle policies are configured at 180 days
- **WHEN** the S3 buckets' lifecycle configurations are inspected
- **THEN** `hiringiq-pilot-biometric-raw/` has a policy expiring objects at 180 days
- **AND** `hiringiq-pilot-rrweb-recordings/` has the same policy
- **AND** the DDB applicant table's `face_enrollment_embedding` attribute has a TTL of 180 days (or is paired with a deletion lambda)

#### Scenario: Per-tenant override possible
- **WHEN** a hypothetical customer requests 30-day deletion
- **THEN** a tenant-specific lifecycle rule can override the default to 30 days for that org's objects
- **AND** the override is applied without affecting other tenants

### Requirement: Storage architecture (D-P5)

The system SHALL split biometric storage by artifact type:
- **Persona selfie photo** → S3 bucket `hiringiq-pilot-biometric-raw/` with KMS encryption, restricted IAM, object-level access logging
- **face-api.js embedding** → DynamoDB attribute on the applicant's record (encoded as base64 Float32Array buffer; <2 KB per applicant)
- **rrweb recording** → S3 bucket `hiringiq-pilot-rrweb-recordings/` (D-RR2)

The differentiation rationale:
- Raw selfie is the highest-sensitivity item — separate bucket with tightest controls
- Embedding is smaller and less reconstructable into a face — DDB is fine
- rrweb is operational data with a different access pattern — its own bucket

#### Scenario: Selfie and embedding written to documented locations
- **WHEN** a successful Persona enrollment completes for applicant A1 in org O1
- **THEN** S3 has the object `hiringiq-pilot-biometric-raw/O1/A1/selfie.jpg`
- **AND** the applicant's DDB record has attribute `face_enrollment_embedding` as a base64 string

### Requirement: IAM scoping for biometric access

The system SHALL scope IAM permissions on the biometric-raw S3 bucket strictly:
- Write: `applicant-service` Lambda only (writes the selfie on enrollment)
- Read: `applicant-service` Lambda (re-reads for re-enrollment if needed) + `biometric-deletion` Lambda (for right-to-deletion) + Phase 3's dashboard signing proxy (Lambda that signs short-lived URLs for the dashboard to display)
- Delete: `biometric-deletion` Lambda only

Object-level access logging is enabled — every GetObject + PutObject + DeleteObject is logged to a separate S3 bucket `hiringiq-pilot-biometric-access-logs/` for compliance audit.

No human IAM user SHALL have direct read access to the biometric-raw bucket. Access is mediated through Lambdas with audit-logged invocations.

#### Scenario: Unauthorized Lambda cannot read biometric-raw
- **WHEN** the `telemetry-service` Lambda attempts `GetObject` on a biometric-raw file
- **THEN** the call returns AccessDenied

#### Scenario: Object-level access is logged
- **WHEN** the applicant-service Lambda reads the biometric-raw object for re-enrollment
- **THEN** the access log bucket records the GetObject event with the IAM principal + timestamp

### Requirement: Recruiter-mediated right-to-deletion at MVP (D-P5)

The system SHALL support deletion of an applicant's biometric data via a recruiter-mediated workflow:
1. Applicant emails the recruiter requesting deletion (per GDPR/CPRA right-to-deletion)
2. Recruiter triggers the deletion via the dashboard (Phase 3 — dashboard button)
3. The dashboard invokes the `biometric-deletion` Lambda with `{ applicant_id }`
4. The Lambda atomically deletes:
   - The applicant's selfie from `hiringiq-pilot-biometric-raw/`
   - The applicant's `face_enrollment_embedding` DDB attribute
   - ALL rrweb recordings for sessions belonging to that applicant from `hiringiq-pilot-rrweb-recordings/`
   - The applicant's IntegritySummary records from `hiringiq-pilot-integrity-summaries` (sets a `deleted_at` marker; the row stays for audit-trail integrity but biometric content is cleared)
5. The Lambda emits an audit log event `biometric.deleted` with the deletion timestamp + scope summary

Self-serve right-to-deletion UI (applicant initiates without recruiter) is deferred to v2.

#### Scenario: Deletion Lambda removes all artifact types atomically
- **WHEN** the `biometric-deletion` Lambda is invoked with `{ applicant_id: "A1" }`
- **THEN** S3 GetObject on the applicant's selfie returns 404
- **AND** the DDB applicant record's `face_enrollment_embedding` attribute is null or removed
- **AND** S3 ListObjects on the rrweb-recordings bucket with prefix `<org_id>/<session_id>/` for the applicant's sessions returns empty
- **AND** the IntegritySummary records for the applicant's sessions have `deleted_at` populated
- **AND** the audit log records `biometric.deleted` with full scope payload

#### Scenario: Access-log entries retained post-deletion
- **WHEN** an applicant's biometric data is deleted via the workflow above
- **THEN** the `recruiter_access.viewed_persona_selfie` events for that applicant (from prior recruiter views) REMAIN in the audit log
- **AND** these access-log entries serve as the audit trail that the data was accessed before deletion

### Requirement: Privacy notice content reflects retention + deletion

The privacy notice (linked from the consent landing per D-IS2's *"View privacy notice"* link) SHALL state:
- The system records the applicant's selfie (Persona-derived) + a face embedding + a session replay (rrweb)
- These artifacts are retained for 180 days
- The applicant can request deletion by contacting their recruiter
- Audio is NOT stored (mic is monitored for activity only)
- The list of integrity signals captured (silent + persistent + escalating per D-SM2-pre)

The privacy notice content is finalized in Phase 2 (Phase 1 used a stub). Legal review of this content is a hard precondition before Phase 2 applies to a real applicant.

#### Scenario: Privacy notice link loads the updated content
- **WHEN** an applicant clicks "View privacy notice" on the consent landing
- **THEN** the loaded page contains the documented retention + deletion + audio-not-stored language
- **AND** the page is reviewable in advance by legal counsel
