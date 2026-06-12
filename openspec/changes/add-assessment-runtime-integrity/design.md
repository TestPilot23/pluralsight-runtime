# Design — Assessment Runtime Integrity

## Context

Phase 1 (`add-assessment-runtime-foundation`) laid the runtime workspace + four Lambda services + session state machine + audit-log table, but capturing only session-lifecycle and pre-assessment events. This proposal turns on the **full integrity instrumentation** documented across the platform-shell's `thoughts/assessment-design-decisions.md` sections C (audit log + analyzer), D (state machine + face detection + pause behaviors), and I (rrweb), plus the D-P4 selfie retrieval that was explicitly deferred from Phase 1.

Stakeholders: KMS owns implementation. The Pluralsight Technical Lead reviews the LLM analyzer prompt and the IntegritySummary shape (because the recruiter dashboard in Phase 3 consumes it). Compliance / legal review is needed for the privacy-notice update + biometric retention + right-to-deletion flow.

## Goals / Non-Goals

**Goals (this proposal):**
- Every applicant session captures the full audit log per D-AL1's envelope + D-AL2's taxonomy
- Persona selfie is retrieved and stored as face-api.js enrollment baseline (D-P4)
- face-api.js, audio VAD, browser instrumentation, screenshot detection all run during assessments and emit silent + persistent + escalating events per the three UX patterns (D-SM2-pre)
- rrweb session recording is captured for every session
- The post-session integrity analyzer produces an `IntegritySummary` with score + behavioral summary + flagged moments + timeline segments + rrweb pointer
- Biometric data (selfie, embedding, rrweb) lives in S3/DDB with 180-day retention + KMS encryption + the documented IAM scoping

**Non-Goals (deferred to Phase 3):**
- Any recruiter-facing UI for the data produced by this proposal
- The recruiter-initiated right-to-deletion dashboard button (the underlying Lambda is provisioned here; the trigger is in Phase 3)
- Sharing/exporting of rrweb recordings or audit logs outside the dashboard

**Non-Goals (deferred or out of scope entirely):**
- Voice biometric identification ("is this the applicant's voice?")
- Mouth-correlation for VAD
- Audio waveform storage or transcription
- Hardware-capture / external-screen-share detection (impossible from the browser — documented in design)
- Self-serve right-to-deletion UI

## Decisions

### D-I1. Phase 2 adds capability layers without rewriting Phase 1

**Choice:** Phase 2 strictly EXTENDS Phase 1. The WebcamCard from Phase 1 gains a warning state. The session state machine gains new PAUSED reasons. The Persona webhook handler gains a post-success retrieval step. No Phase 1 contract is broken; every existing requirement still holds.

**Why:** The phased split was designed for this. Modifying Phase 1's specs (vs adding to them) would invalidate previously-applied work — exactly the kind of churn the staged-proposal model avoids.

**Implementation:** Phase 2's `MODIFIED Requirements` blocks (where they appear in capability specs) describe the additions to Phase 1's existing requirements without rewriting them.

### D-I2. face-api.js model files served from the runtime CloudFront origin

**Choice:** TinyFaceDetector + face landmarks + face recognition net (~6 MB total compressed) are served as static assets from `apps/runtime-web/public/face-api-models/`, behind the same CloudFront distribution as the React app. Downloaded once at intro-screen pre-warm + cached by the browser for subsequent assessments by the same applicant.

**Why:** Avoids the operational complexity of a separate model-serving endpoint. CloudFront caching makes the 6MB cost a one-time hit per applicant (per session, in practice — applicants don't take repeat assessments often enough for cache to matter much).

**Alternative considered:** Pull from a CDN like jsdelivr. Rejected — third-party CDN availability + content-integrity verification + CSP complexity outweigh the bandwidth savings.

### D-I3. rrweb upload uses a dedicated HTTPS endpoint, NOT the WebSocket

**Choice (already locked in D-RR1):** rrweb event batches are uploaded via `POST /api/telemetry/rrweb-batch` (HTTPS). Batches are 5s/100-events whichever first.

**Why:** rrweb generates high-volume events (~1000+ per minute). Pushing them through the WS would compete with the integrity-signal stream for bandwidth and could trigger WS backpressure. Separate HTTPS path isolates the load.

**Why not stream rrweb in real-time:** rrweb's library defaults to batched callbacks; there's no per-event-streaming mode that would work well over WS. Even if there were, the analyzer consumes the recording post-session, not in real-time — so streaming has no consumer-side benefit.

**Implementation:** the telemetry-service Lambda's HTTP route at `POST /api/telemetry/rrweb-batch` receives the batch, validates the JWT, and writes to S3 via the Lambda's IAM permissions. No DDB write for rrweb data — only the S3 object.

### D-I4. Bedrock model is configurable via SSM; default Nova Pro

**Choice:** SSM parameter `/hiringiq/pilot/runtime/integrity/analyzer-model` defaults to `amazon.nova-pro-v1:0`. Alternative `anthropic.claude-3-5-sonnet-20241022-v2:0` for higher-quality runs on flagged sessions (post-pilot, the analyzer can do a two-pass system — Nova first, Claude on flagged).

**Why:** Nova Pro at MVP cost per assessment is ~$0.02-0.05 (well under the $0.05-$0.10 estimate from project memory). If pilot recruiters report the analysis isn't fine-grained enough, swap to Claude 3.5 Sonnet without redeploy.

**Versioning:** the analyzer_version field on IntegritySummary captures `{model_id}@{prompt_version}`. Re-runs against improved prompts coexist as separate version records (D-AL4).

### D-I5. Persona selfie retrieval happens server-side after webhook receipt

**Choice:** After Phase 1's Persona webhook handler receives a successful inquiry result, the applicant-service Lambda calls Persona's Inquiry API (`GET /api/v1/inquiries/{id}`) using the API key from Secrets Manager. Extracts the `selfie-photo` URL from the verification's attributes. Downloads the image. Uploads to S3 at `<biometric-raw-bucket>/<org_id>/<applicant_id>/selfie.jpg` with KMS encryption. Generates the face-api.js embedding from the image (server-side via the same library used in the browser). Persists the embedding as a DDB attribute on the applicant record.

**Why:** Server-side handling means the embedding is computed in a trusted environment; the client never sees the embedding (which is biometric data) directly. The signed URL from Persona is short-lived (~15 min per Persona's docs) — we download promptly during the webhook handler invocation.

**Cost:** No Persona API charge — Persona's per-inquiry fee covers the inquiry, including subsequent API calls to retrieve assets. Lambda cold-start overhead for the embedding generation is ~3-5 seconds; runs during the webhook callback which isn't user-facing-blocking.

**Failure handling:** If selfie retrieval fails (network, Persona API outage, image-decoding error), the applicant's Persona verification is still considered successful (the verdict came from Persona, not from us). The face-api.js enrollment is marked "deferred" and a CloudWatch alarm fires. The recruiter dashboard in Phase 3 can show this state for affected sessions. The face-detection presence + identity checks during assessment degrade gracefully — presence still works against the live stream alone; identity verification reports "no enrollment, comparison skipped."

### D-I6. Identity-mismatch detection is debounced to suppress false positives

**Choice (refines D-SM6's silent-event posture for face.identity_mismatch):** `face.identity_mismatch` event emits ONLY when the face-api.js confidence between the live frame embedding and the enrollment embedding falls below the SSM-configurable threshold (default 0.4 — Euclidean distance, lower = more similar) for **N consecutive checks** (default N=3, ~15 seconds at 5s cadence).

**Why:** A single low-confidence frame might be lighting glitch, head turn, or partial occlusion. Three consecutive low-confidence frames is statistically much more likely to be a real identity change.

**Alternative considered:** Lower threshold, single-frame emission. Rejected — false-positive rate makes the signal noisy.

**Tuning:** SSM parameters `face_detection.identity_threshold` and `face_detection.identity_mismatch_consecutive_min`. Defaults are best-guess; pilot data will refine.

### D-I7. Attention-off-screen is a head-pose-derived signal

**Choice (D-P5 / D-RR / per design § D-AL):** `face.attention_off_screen` emits when face-api.js's head pose estimation shows pitch >30° down (looking at phone in lap) OR sustained yaw >45° in either direction (looking at a side monitor), for ≥10 contiguous seconds. Silent event.

**Why:** Targets the specific threat of "applicant takes phone photo of screen, uploads to Claude/ChatGPT, reads answer." A consumer who's looking down at their phone produces a clear head-pose-pitch signal that's distinguishable from normal screen reading (which is pitch ≈0° to slightly negative).

**Limitations (honestly acknowledged):** Voice-only AI assistance with the applicant maintaining eye contact CANNOT be detected by this signal. The analyzer's correlated-signals approach (timing anomalies + face-confidence drops + answer-quality mismatch) is the broader defense.

**Implementation:** face-api.js's landmark detector provides 68 facial landmarks; head pose is estimated from a subset of those (nose tip, chin, eye corners). The math is in the face-api.js library; we just consume the pose values.

### D-I8. The audit log's S3 archive uses Object Lock in compliance mode

**Choice (D-AL3):** S3 bucket `hiringiq-pilot-audit-archive/` is configured with Object Lock in **compliance mode** (not governance mode). Default retention: 180 days. Even an AWS root account user cannot delete objects within the retention window — court-defensible.

**Why:** Audit logs must be tamper-evident. Compliance mode is the strongest S3 guarantee available. The 180-day window matches the biometric retention; after expiration, the lifecycle policy permits deletion (and Glacier transition starts at 30 days for cost reasons).

**Trade-off accepted:** If a misconfiguration causes wrong data to land in this bucket, we cannot delete it for 180 days. Acceptable — the bucket only receives append-only event records via the telemetry-service Lambda, not user-content.

### D-I9. LLM prompt is versioned and templated

**Choice (D-AL4 + D-AL6):** The analyzer-service maintains its prompt as a versioned template in `services/analyzer/src/prompts/v1.ts`. The prompt is constructed by interpolating the session's metadata + audit log + items (with `answer` stripped) + responses + IntegritySummary template into the template. The full constructed prompt is sent to Bedrock; the model produces structured JSON output (validated via Zod against the IntegritySummary schema before persisting).

**Why:** Prompts will change. Versioning each prompt as `v1`, `v2`, etc. means past summaries reference their generating prompt, and a re-run with `v2` coexists with `v1` summaries — useful for A/B comparisons and pilot iteration.

**Prompt structure (high level):**
```
SYSTEM: You are an assessment-integrity analyzer. Your job is to describe what
happened during an assessment session FACTUALLY without judging cheating intent.
DO NOT use language like "appears to be cheating" or "evidence of cheating."
Describe events neutrally and let the reviewer draw conclusions.

USER: Here is the assessment context: <campaign metadata, item count, total time>.
Here are the items the applicant saw (answer keys stripped): <items>.
Here are the applicant's responses with timing: <responses>.
Here is the full audit-log event stream: <events>.
Here is the Persona result: <persona status + confidence>.

Produce an IntegritySummary with the following schema: ...

Look for indicators of external assistance: items where the applicant's response
time was significantly longer than typical, particularly when (a) there was no
input activity during the elapsed time, (b) the final answer appeared in a short
burst, (c) face-detection confidence or head-pose indicated reduced screen
attention during the gap, and (d) the answer quality contrasts notably with
other answers in the same session. Describe each such pattern factually.

Identify continuous time ranges with clusters of integrity signals. Assign each
range a severity band (green, yellow, red). Cover the entire session timeline —
fill the gaps between anomaly ranges with green segments.
```

**Why this structure:** the SYSTEM message anchors the no-judgment-language rule. The USER message provides all context. The output schema is enforced via Zod parsing post-response.

### D-I10. Right-to-deletion flow provisions the deletion Lambda in Phase 2

**Choice:** A new Lambda `hiringiq-pilot-biometric-deletion` is provisioned in Phase 2. It accepts a `{ applicant_id }` input and atomically deletes:
- The applicant's Persona selfie from S3
- The applicant's face-api.js embedding from the applicant DDB record
- The applicant's rrweb recordings from S3 (all sessions)
- The applicant's IntegritySummary records from DDB (sets a deletion marker but retains audit-log integrity)

The Lambda is NOT exposed via API Gateway at Phase 2 — only the recruiter-dashboard call from Phase 3 will invoke it.

**Why:** Provisioning the deletion machinery now means Phase 3's dashboard work is a wiring exercise, not a deletion-Lambda-from-scratch exercise.

**Note on audit log retention vs deletion:** Per D-AL7 + D-P5, audit-log entries that REFERENCE the deleted biometric data are retained (they're operational records, not biometric). The biometric data itself is deleted; the records of who-accessed-it remain.

## Risks / Trade-offs

- **face-api.js model bundle adds initial load time.** 6 MB compressed model files. **Mitigation:** intro-screen pre-warm + CloudFront caching + browser cache.
- **Bedrock LLM analysis can fail or be slow.** Network failures, model unavailability, output that doesn't validate against the schema. **Mitigation:** SQS DLQ + retry policy (3 attempts with exponential backoff); on permanent failure, IntegritySummary record is written with `status: "analyzer_failed"` and the dashboard shows "Analysis unavailable for this session" — recruiter can request manual re-analysis from the dashboard in Phase 3.
- **Persona selfie retrieval depends on the Hosted/API contract flow (Q-PSL-1).** Action item open. **Mitigation:** if Embedded Flow is confirmed, swap to local selfie capture on the device-check screen — ~½ day of work.
- **rrweb recording size unpredictable for very long sessions.** A 4-hour session could produce ~50-100 MB. **Mitigation:** S3 multipart upload supported by the rrweb upload handler; max session length capped at 4 hours via session-state machine timeout (PAUSED → ABANDONED at 30 min × 8 cycles maxes out around there anyway).
- **Biometric storage triggers BIPA + GDPR Article 9 + CPRA compliance burden.** **Mitigation:** 180-day retention with automated lifecycle, KMS encryption, restricted IAM, recruiter-mediated right-to-deletion, the privacy notice update lands in Phase 2 so applicants are informed before any biometric data is captured.
- **Identity-mismatch false positives at the start of the assessment.** Initial face-api.js comparisons against the just-captured enrollment can have brief low-confidence windows as the applicant settles in. **Mitigation:** debounce (D-I6) — 3 consecutive low-confidence frames required.
- **Cost overrun on Bedrock at high volume.** ~$0.05-$0.10 per assessment is fine at pilot; at 10,000/mo that's $500-$1000/mo. **Mitigation:** monitor; consider Nova Lite for cost reduction or Claude for higher-quality re-analysis on flagged sessions.

## Migration Plan

This proposal applies after Phase 1 is live. No data migration — only additive deployment.

1. **Apply Terraform** — new buckets, new DDB table, new IAM grants, SSM parameters.
2. **Deploy Lambda code** — analyzer-service goes from stub to real; applicant-service gains selfie retrieval; telemetry-service gains the rrweb upload handler + new event-type handlers.
3. **Deploy runtime-web** — new integrity modules + face-api.js + rrweb-record bundles.
4. **Smoke test end-to-end:** create a test campaign, generate a magic link, run through the assessment. **Verify:** audit-events table contains integrity signals beyond just session-lifecycle; biometric-raw bucket contains the Persona selfie + embedding written to applicant record; rrweb-recordings bucket contains the session's event stream; IntegritySummary appears in the new table within ~2 min of session completion.
5. **Privacy notice update:** the runtime-web privacy notice link target gets the new content (biometric retention, right-to-deletion, etc.) — important because applicants who land on the consent landing AFTER this proposal applies need to see the updated notice.

**Rollback strategy:**
- Bad Bedrock model behavior: change the SSM parameter to a known-good model; existing IntegritySummary records preserved.
- Bad analyzer prompt: roll back the prompt template, increment `prompt_version`. The DLQ catches in-flight failures.
- Bad face-api.js model: roll back the model files in `apps/runtime-web/public/face-api-models/`; browsers re-download on next session.
- Catastrophic: terraform destroy the integrity-specific resources (buckets, DDB table). Phase 1 continues to function.

## Open Questions

- **OPEN:** Q-PSL-1 — Persona contract flow confirmation. Highest-priority action item.
- **OPEN:** Bedrock model choice (Nova Pro vs Claude). Default Nova Pro; SSM-tuneable.
- **OPEN:** face-api.js library variant (`face-api.js` vs `@vladmandic/face-api`). Lean: `@vladmandic/face-api`. Tactical.
- **OPEN:** Clock-skew threshold initial values (default 5000ms baseline, 2000ms drift). Pilot data will refine.
- **OPEN:** Audit-log archive retention vs cost — 180 days hot is generous; should we shorten archival access?
