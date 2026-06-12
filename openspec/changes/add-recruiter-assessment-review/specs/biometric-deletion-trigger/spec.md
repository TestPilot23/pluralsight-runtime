# Biometric Deletion Trigger

## ADDED Requirements

### Requirement: Per-applicant deletion via applicant detail page (D-R4)

The system SHALL provide a "Delete biometric data" button on the platform-shell's applicant detail page (`apps/web/src/pages/ApplicantDetail.tsx` per platform-shell proposal #9). The button is PER-APPLICANT (not per-session) per D-P5 — deleting an applicant's biometric data removes ALL artifacts for that applicant across all their sessions:
- The Persona selfie image (S3)
- The face-api.js embedding (DDB attribute)
- All rrweb session recordings for the applicant's sessions (S3)
- IntegritySummary records get a `deleted_at` marker (records retained; biometric content cleared)

#### Scenario: Button appears on applicant detail page
- **WHEN** a recruiter navigates to an applicant detail page for an applicant whose biometric data has NOT been deleted
- **THEN** the page displays the "Delete biometric data" button

#### Scenario: Button is hidden when already deleted
- **WHEN** the applicant has `biometric_deleted_at` set on their record
- **THEN** the "Delete biometric data" button is absent (or disabled with a "Already deleted on {date}" label)

### Requirement: Two-step confirmation modal

Clicking the button SHALL open a confirmation modal with the wording:

> **Are you sure?**
>
> This will permanently delete the applicant's selfie, face embedding, and all session recordings. This cannot be undone.
>
> [Cancel]   [Yes, Delete]

Clicking "Yes, Delete" triggers the deletion flow; "Cancel" closes the modal without action.

#### Scenario: Cancel closes modal without action
- **WHEN** the recruiter clicks "Delete biometric data" → "Cancel" in the modal
- **THEN** the modal closes and no deletion occurs

#### Scenario: Confirm triggers deletion
- **WHEN** the recruiter clicks "Delete biometric data" → "Yes, Delete"
- **THEN** the dashboard calls `POST /api/applicants/<id>/biometric-deletion` on the runtime backend

### Requirement: Backend invokes the Phase 2 biometric-deletion Lambda

The runtime workspace's `POST /api/applicants/<id>/biometric-deletion` endpoint SHALL:
1. Validate the recruiter's Cognito JWT
2. Check the recruiter's `org_id` matches the applicant's `org_id` (cross-org deletion rejected with 403)
3. Emit `biometric.deletion_requested` audit event with `{ recruiter_user_id, recruiter_email, applicant_id, org_id }`
4. Invoke the Phase 2 `hiringiq-pilot-biometric-deletion` Lambda synchronously via `lambda:InvokeFunction`
5. On Lambda success, update the applicant record with `biometric_deleted_at: <now>` + return 200 to the dashboard
6. On Lambda failure, return 500 with `{ code: "DELETION_FAILED", message: <error> }` — the dashboard surfaces this as a failure snackbar

#### Scenario: Successful deletion updates applicant record
- **WHEN** the runtime backend invokes the deletion Lambda for applicant A1
- **AND** the Lambda returns success after deleting all artifacts
- **THEN** A1's applicant record gains `biometric_deleted_at: "<ISO timestamp>"`
- **AND** the runtime backend returns 200 to the dashboard

#### Scenario: Failed deletion does NOT update the applicant record
- **WHEN** the deletion Lambda fails partway through (e.g., S3 timeout)
- **THEN** the applicant record does NOT receive `biometric_deleted_at`
- **AND** the runtime backend returns 500
- **AND** the dashboard displays a failure snackbar with retry option

### Requirement: SessionDetail page reflects deletion state

When an applicant's biometric data has been deleted, the SessionDetail page for any of their sessions SHALL display deletion-aware placeholders:
- The rrweb playback viewer shows *"Session recording was deleted on {biometric_deleted_at} per the applicant's request."*
- Any selfie thumbnail in the session detail (if applicable) shows a "Selfie deleted" placeholder
- The IntegritySummary card still displays (score + behavioral_summary are not biometric)
- The audit log still displays (events are not biometric)
- The "Re-analyze with latest prompt" button is DISABLED with a tooltip *"Re-analysis requires biometric data that has been deleted."*

#### Scenario: Session detail shows deletion placeholders
- **WHEN** a recruiter navigates to a session detail page for a session whose applicant has `biometric_deleted_at`
- **THEN** the rrweb viewer shows the deletion-aware message
- **AND** the IntegritySummary + audit log render normally
- **AND** the Re-analyze button is disabled with the documented tooltip

### Requirement: Deletion event is logged + access logs retained (D-AL7)

The deletion flow SHALL emit `biometric.deletion_requested` (when the recruiter clicks confirm) and `biometric.deleted` (from the Lambda upon successful completion) audit events. Both are retained for the full 180-day window.

Recruiter access events that REFERENCE the deleted data are NOT themselves deleted (per D-AL7 — they're operational records).

#### Scenario: Deletion event sequence is logged
- **WHEN** a recruiter successfully deletes applicant A1's biometric data
- **THEN** the audit log contains `biometric.deletion_requested` + `biometric.deleted` events
- **AND** all prior `recruiter_access.*` events for A1 remain in the audit log unchanged
