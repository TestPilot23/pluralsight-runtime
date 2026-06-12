# Biometric Artifacts

The Persona selfie image, the face-api.js identity embedding, and the rrweb session recordings. Not a single entity — a coordinated set of artifacts spanning S3 buckets and a DDB attribute on [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]].

## Storage

### Selfie image

- **Bucket:** `hiringiq-pilot-biometric-raw/`
- **Key:** `<org_id>/<applicant_id>/selfie.jpg`
- **Encryption:** KMS, customer-managed key
- **Access:** `applicant-service` (write at enrollment), `biometric-deletion` Lambda (delete), `analyzer-service` (read), Phase 3 dashboard signer (read via 15-min signed URL)
- **Lifecycle:** Auto-delete at 180 days via S3 lifecycle policy
- **Logging:** Object-level access logging enabled

### Face embedding

- **Stored on:** the applicant record in `hiringiq-pilot` (platform-shell table) as an attribute
- **Attribute:** `selfie_embedding`
- **Type:** String — base64-encoded `Float32Array` buffer (128 or 512 dims depending on the face-api.js recognition net version)
- **Size:** < 2 KB per applicant

### rrweb session recordings

- **Bucket:** `hiringiq-pilot-rrweb-recordings/`
- **Key:** `<org_id>/<session_id>/events-<batch_seq>.json` (one object per 5s/100-event batch)
- **Encryption:** KMS, customer-managed key
- **Access:** `telemetry-service` (write), Phase 3 dashboard signer (read)
- **Lifecycle:** Auto-delete at 180 days

### Audit-archive S3 objects

Although not "biometric" in the legal sense, the audit-archive S3 bucket carries the integrity-event trail that references biometric activity (face-detection events, identity-mismatch events). See [[AuditEvent]] for details. The audit-archive uses **Object Lock in compliance mode** for 180-day retention — this is non-deletable even by root within the retention window.

## Production pipeline

```
Persona inquiry succeeds
        │
        ▼
persona-webhook (applicant-service)
        │
        ├──> downloads selfie via Persona's signed URL
        ├──> writes selfie to s3://biometric-raw/<org>/<applicant>/selfie.jpg
        ├──> @vladmandic/face-api generates 128-dim or 512-dim embedding
        ├──> base64-encodes the Float32Array
        └──> writes embedding to applicant record's selfie_embedding attribute
```

```
Assessment session in progress
        │
        ▼
rrweb-capture (runtime-web client) ──> POST /api/telemetry/rrweb-batch (HTTPS) ──> s3://rrweb-recordings/<org>/<session>/
                                       │
                                       └──> service: telemetry-service
                                            JWT-validated via the magic-link secret
                                            batches concatenated at analyzer-read time
```

## Right-to-deletion

A recruiter triggering the deletion flow via `POST /api/applicants/<id>/biometric-deletion` invokes the `biometric-deletion` Lambda, which performs:

1. **Delete selfie** — `s3://biometric-raw/<org>/<applicant>/*`
2. **Delete embedding** — UpdateItem on the applicant record removing the `selfie_embedding` attribute
3. **Delete rrweb recordings** — `s3://rrweb-recordings/<org>/<session>/*` for every session belonging to this applicant
4. **Mark [[IntegritySummary]] records as deleted** — soft-flag, summary content preserved for compliance
5. **Sets `biometric_deleted_at` on the applicant record** — UI hides re-deletion CTAs after this is set
6. **Emits audit events** — `biometric.deletion_requested` + `biometric.deleted` to [[AuditEvent]]

### What's NOT deleted (intentional)

- `recruiter_access.*` events for that applicant — the access audit log remains complete even after deletion. Compliance requires we know who looked at biometric data before it was deleted.
- The Persona-stored biometric on Persona's side — that's governed by Persona's own retention. The runtime doesn't have a delete-from-Persona endpoint at the pilot scale.

## Invariants (D-P5)

- 180-day default retention. Recruiter-triggered deletion happens any time before the 180-day cutoff.
- The audit-archive S3 bucket's Object Lock **prevents** deletion of the access trail during the 180-day window, even by root, even if a deletion is requested.
- Persona biometric data stays with Persona — the runtime never persists the raw Persona-side biometric. Only the face-api.js embedding computed from the Persona-returned selfie is stored on our side.
- The face-api.js embedding alone cannot reconstruct a face image. It's a one-way derivative — useful for live-frame comparison only.

## Related entities

- [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] — embedding lives as an attribute; selfie + recordings index by `applicant_id`
- [[Session]] — rrweb recordings are keyed by `session_id`
- [[AuditEvent]] — every biometric write + read + delete emits an audit event
- [[IntegritySummary]] — soft-flagged as deleted (not hard-deleted) when the applicant's biometric data is deleted

## Source proposals

- `add-assessment-runtime-integrity` — defining proposal (S3 buckets, face-api.js pipeline, biometric-deletion Lambda)
- `add-recruiter-assessment-review` — adds the recruiter-facing deletion trigger UI
