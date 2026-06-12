# Tasks — Assessment Runtime Integrity

Sequenced by dependency. Phases 1, 2 lay the AWS + library foundations. Phases 3–9 build the integrity capture layers. Phase 10 implements the analyzer. Phase 11 verifies end-to-end.

## 1. AWS provisioning (Terraform updates)

- [ ] 1.1 Add `infra/terraform/modules/biometric-storage/` provisioning S3 bucket `hiringiq-pilot-biometric-raw/` with KMS encryption (customer-managed key), restricted IAM (only `applicant-service` + `biometric-deletion` + `analyzer-service` Lambdas can read), object-level access logging enabled, lifecycle policy for 180-day deletion. **Verify:** `terraform plan` shows the bucket; access from an unauthorized IAM role is denied.
- [ ] 1.2 Add `infra/terraform/modules/rrweb-storage/` provisioning S3 bucket `hiringiq-pilot-rrweb-recordings/` with KMS, IAM scoped to `telemetry-service` (write) + Phase-3 dashboard signer (read), 180-day lifecycle. **Verify:** `terraform plan` shows the bucket.
- [ ] 1.3 Add `infra/terraform/modules/audit-archive/` provisioning S3 bucket `hiringiq-pilot-audit-archive/` with **Object Lock in compliance mode**, default 180-day retention, KMS, lifecycle transition to Glacier at 30 days. **Verify:** Object Lock configuration is `COMPLIANCE` mode; attempting to delete an object within retention window fails even from root.
- [ ] 1.4 Add `infra/terraform/modules/integrity-summaries-table/` provisioning DynamoDB table `hiringiq-pilot-integrity-summaries` with PK=`session_id`, GSI on `applicant_id`, GSI on `org_id`, on-demand billing, PITR enabled, SSE. **Verify:** `terraform plan` shows the table.
- [ ] 1.5 Update existing IAM roles per D-I5:
  - `applicant-service`: `s3:PutObject` on biometric-raw bucket; egress permissions to Persona's API (NAT or VPC endpoint depending on existing networking)
  - `analyzer-service`: `bedrock:InvokeModel` on configured model ARNs; `s3:GetObject` on rrweb-recordings + audit-archive + biometric-raw; `dynamodb:PutItem` on integrity-summaries table
  - `telemetry-service`: `s3:PutObject` on rrweb-recordings + audit-archive
  - **Verify:** rendered policies match documented scopes; no wildcards.
- [ ] 1.6 Add new Lambda `hiringiq-pilot-biometric-deletion` with the cross-resource deletion handler (D-I10). Not exposed via API Gateway at Phase 2. **Verify:** Lambda exists; IAM has Delete permissions on the relevant buckets + DDB tables.
- [ ] 1.7 Add new SSM parameter namespace `/hiringiq/pilot/runtime/integrity/*` with initial values:
  - `analyzer-model = "amazon.nova-pro-v1:0"`
  - `face-detection.cadence-seconds = 5`
  - `face-detection.identity-threshold = 0.4`
  - `face-detection.identity-mismatch-consecutive-min = 3`
  - `face-detection.attention-pitch-degrees-threshold = 30`
  - `face-detection.attention-yaw-degrees-threshold = 45`
  - `face-detection.attention-sustained-seconds = 10`
  - `clock-skew.baseline-threshold-ms = 5000`
  - `clock-skew.drift-threshold-ms = 2000`
  - **Verify:** `aws ssm get-parameters --names ...` returns all values.
- [ ] 1.8 Run `terraform apply`. **Verify:** clean `terraform plan` afterwards.

## 2. Shared library updates (`packages/shared/`)

- [ ] 2.1 Add full event-type enums to `packages/shared/src/audit-events.ts` matching D-AL2's taxonomy. Cover all four categories (identity, session-lifecycle, integrity-signal, assessment) with the documented event types. **Verify:** unit tests cover every event type's payload Zod schema.
- [ ] 2.2 Add IntegritySummary Zod schema to `packages/shared/src/integrity-summary.ts` per D-RR3's documented shape. Includes timeline_segments validation (all three severity strings + full timeline coverage assertion). **Verify:** unit tests cover happy path + segment-coverage validation + payload field types.
- [ ] 2.3 Add face-api.js TypeScript types for the embedding (Float32Array of length 128 or 512 depending on the recognition net used) in `packages/shared/src/biometric-types.ts`. **Verify:** TypeScript compiles; the embedding can be serialized to a DDB-storable string (base64-encoded Float32Array buffer).
- [ ] 2.4 Add `packages/shared/src/clock-skew.ts` with the baseline + drift detection helpers per D-AL1a. Reads SSM thresholds, computes offset using ping/pong latency estimate. **Verify:** unit tests with synthetic timestamps cover baseline-exceeded + drift-detected emission triggers.

## 3. Audit-log envelope + S3 archive (capability: `integrity-event-taxonomy`)

- [ ] 3.1 Update `services/telemetry/src/event-ingest.ts` to write every received event to BOTH the DDB `hiringiq-pilot-audit-events` table AND the S3 `hiringiq-pilot-audit-archive/` bucket. Object key: `<org_id>/<applicant_id>/<session_id>/<timestamp>-<event_id>.json`. **Verify:** integration test confirms a single ingested event appears in both stores.
- [ ] 3.2 Implement the dual-timestamp + IP capture per D-AL1. Server stamps `timestamp_server` on receipt; client stamps `timestamp_client` at emission; the WS connection's source IP is recorded per-event (not just at $connect). **Verify:** integration test confirms an event's payload contains both timestamps and the IP.
- [ ] 3.3 Implement clock-skew detection per D-AL1a. Compute baseline from first 3 events of session. Per-event drift detection with SSM thresholds. Emit `clock.skew_detected` on threshold exceedance. **Verify:** unit test simulates a clock jump and confirms the event fires with the correct payload.
- [ ] 3.4 Implement schema_version handling. Default `schema_version: 1` at this phase. Adding new event types is additive (no version bump). **Verify:** all events emitted at this phase carry `schema_version: 1`.

## 4. Persona selfie enrollment (capability: `persona-selfie-enrollment`)

- [ ] 4.1 Extend `services/applicant/src/persona-webhook.ts` to call Persona's Inquiry API on successful inquiry per D-I5 + D-P4. Extract `selfie-photo` URL from the verification's `attributes` field. **Verify:** unit test with mocked Persona response confirms the URL is extracted; integration test with the Persona sandbox confirms the API call succeeds.
- [ ] 4.2 Implement `services/applicant/src/selfie-retrieval.ts` that downloads the selfie via the signed URL and writes to S3 at `<biometric-raw-bucket>/<org_id>/<applicant_id>/selfie.jpg`. **Verify:** integration test confirms the S3 object exists after a mocked Persona success.
- [ ] 4.3 Implement `services/applicant/src/face-embedding.ts` using `@vladmandic/face-api` to load the selfie + generate a 128-dim or 512-dim embedding. **Verify:** unit test with a known face image produces a stable embedding (same image → same vector within float-precision tolerance).
- [ ] 4.4 Persist the embedding as a DDB attribute on the applicant record. Store as base64-encoded Float32Array buffer for compact storage (<2 KB). **Verify:** integration test confirms the embedding round-trips through DDB write + read.
- [ ] 4.5 Handle Persona Inquiry API failures gracefully per D-I5: log the error, emit `enrollment.deferred` audit event, allow the applicant to proceed (the assessment still runs; identity comparison degrades to "no enrollment, skipped"). **Verify:** unit test simulates an API failure and confirms the applicant can continue.
- [ ] 4.6 Emit audit events: `enrollment.photo_stored`, `enrollment.embedding_computed`, `enrollment.deferred` (on failure). **Verify:** integration test confirms event sequence in the happy path.

## 5. Face-detection instrumentation (capability: `face-detection-instrumentation`)

- [ ] 5.1 Implement `apps/runtime-web/src/integrity/face-detection.ts` using `@vladmandic/face-api`. Loads the TinyFaceDetector + landmark + recognition nets from `/face-api-models/`. **Verify:** unit test confirms the library loads; integration test confirms detection runs on a webcam stream.
- [ ] 5.2 Implement the polling loop at 5s cadence (SSM-configurable). On each tick: detect face → if absent, increment warning counter, emit `face.not_detected`; if present, reset counter, emit `face.detection_restored` (if applicable). **Verify:** integration test simulates an absent → present → absent sequence and confirms the events + counter behavior.
- [ ] 5.3 Implement the 3-warning escalation per D-SM2: warning #1 at first detection failure → 15s recheck → warning #2 → 15s recheck → warning #3 → 15s recheck → PAUSE.face_not_detected. **Verify:** integration test confirms the timing + state transition.
- [ ] 5.4 Implement identity-mismatch detection per D-I6. Compute distance between live frame embedding and enrollment embedding. Emit `face.identity_mismatch` after N consecutive checks below threshold (default N=3). **Verify:** unit test with synthetic embeddings simulates persistent mismatch and confirms the event fires.
- [ ] 5.5 Implement attention-off-screen per D-I7. Head-pose estimation from face landmarks. Emit `face.attention_off_screen` when pitch >30° down (SSM-configurable) OR yaw >45° in either direction sustained for ≥10 sec. **Verify:** unit test with synthetic landmarks confirms pose calculation; integration test with a recorded video stream covers the head-tilt-down case.
- [ ] 5.6 Implement multiple-face detection. Emit `face.multiple_detected` when the face detector returns >1 face. Pattern: Persistent warning (D-SM2-pre). **Verify:** integration test with a multi-face stream confirms the event fires; toaster appears (handled in capability 7 below).
- [ ] 5.7 All face-detection event handlers emit through the WS telemetry pipeline using the envelope from D-AL1. **Verify:** events appear in the audit-events table with the correct envelope.

## 6. Audio voice activity detection (capability: `audio-voice-activity-detection`)

- [ ] 6.1 Implement `apps/runtime-web/src/integrity/voice-activity.ts` using Web Audio API + AnalyserNode. Spectral analysis distinguishes speech from ambient noise. **Verify:** unit test with synthetic audio buffers covers speech-detected and silence-detected cases.
- [ ] 6.2 Sustained-voice threshold: >2s of speech-like signal triggers `audio.voice_activity_detected` emission. Payload includes `{ duration_seconds, peak_db, item_id_at_time }`. **Verify:** integration test confirms the emission threshold + payload shape.
- [ ] 6.3 **Confirm no audio storage.** The AnalyserNode operates on in-memory audio data only; no waveform reaches the server. **Verify:** grep across the codebase confirms there is no code path that uploads audio buffers anywhere.
- [ ] 6.4 The VAD module runs from session.started through any terminal state. Pauses (PAUSED.*) suspend VAD detection to avoid noise. **Verify:** integration test confirms VAD emits during IN_PROGRESS and is silent during PAUSED.

## 7. Browser instrumentation (capability: `browser-instrumentation`)

- [ ] 7.1 Implement `apps/runtime-web/src/integrity/tab-visibility.ts` handling visibilitychange + blur/focus events. Emit `tab.visibility_lost` / `tab.visibility_restored` / `window.focus_lost` / `window.focus_restored`. All silent per D-SM2-pre. **Verify:** unit test with synthetic events confirms emission.
- [ ] 7.2 Implement `apps/runtime-web/src/integrity/picture-in-picture.ts` handling enterpictureinpicture + leavepictureinpicture events. Silent. **Verify:** unit test confirms emission.
- [ ] 7.3 Implement `apps/runtime-web/src/integrity/media-devices.ts` listening on `navigator.mediaDevices.devicechange`. On session-start: enumerate devices via `enumerateDevices()`. On change: re-enumerate + diff + emit `media_devices.changed` with added/removed device list. **Verify:** integration test with a USB-headset plug/unplug emits the event.
- [ ] 7.4 Implement screen-share fingerprint matching. After each devicechange, match new device labels against a regex set of known screen-share software (`OBS`, `Snap`, `ManyCam`, `XSplit`, `Loopback`, `BlackHole`, `Aggregate`, `NDI`). Emit `media_devices.suspicious_present` with the matched fingerprint. **Verify:** unit test with synthetic device lists confirms detection.
- [ ] 7.5 Implement `apps/runtime-web/src/integrity/display-detection.ts` using `window.screen` for secondary display detection. NO Window Management API permission request per D-DC1. Emit `display.secondary_present` at session start if `window.screen.isExtended` is true OR the screen width exceeds typical primary-display widths. **Verify:** unit test with mocked window.screen properties covers detection cases.
- [ ] 7.6 Implement `apps/runtime-web/src/integrity/clipboard.ts` listening for copy + paste events on the document. On paste in a short-answer textarea, emit `paste.detected` with `{ text, item_id_at_time, timestamp }`. On copy from anywhere, emit `copy.detected`. **Verify:** integration test confirms events fire with content payload.
- [ ] 7.7 Implement `apps/runtime-web/src/integrity/screenshot-detection.ts` listening for PrintScreen keydown events (Windows-reliable, macOS unreliable — Cmd+Shift+3/4/5 is intercepted by the OS). Silent event `screenshot.print_screen_detected`. **Verify:** unit test simulates the keydown event and confirms emission.
- [ ] 7.8 Implement `apps/runtime-web/src/integrity/ip-monitor.ts`. The telemetry-service stamps the IP per-event server-side (per task 3.2); the client-side module's job is to detect IP CHANGES — query a known public IP echo service (e.g., the runtime's own `/whoami` endpoint that returns the connecting IP) on a 5-min cadence + on visibility-restored. Emit `ip.changed` when the value differs from prior. **Verify:** integration test with a mocked /whoami response confirms emission on change.
- [ ] 7.9 Implement `apps/runtime-web/src/integrity/extension-detection.ts` with best-effort signals: MutationObserver on document.body for unexpected DOM mutations not from our app's React render cycle; window-level checks for known extension footprints (e.g., `window.Grammarly`). Silent event `extension.suspicious_activity` with payload describing the detected signal. **Verify:** unit test with synthetic DOM mutations triggers emission; explicit caveat documented that this is best-effort.

## 8. WebcamCard warning state (capability: `webcam-card-warning-state`)

- [ ] 8.1 Update `apps/runtime-web/src/components/WebcamCard.tsx` to subscribe to face-detection events. On `face.not_detected`, apply red 3px border (MD3 error color) to the expanded card. **Verify:** snapshot test confirms the red border appears during warning state.
- [ ] 8.2 Implement the red toaster (MD3 error variant, positioned at the bottom of the assessment interface) with the exact text *"We can't detect your face on camera. Please recenter or check your webcam — otherwise the assessment will end."* Persists across the 45s warning window. **Verify:** snapshot test confirms wording; integration test confirms persistence until face restoration.
- [ ] 8.3 Implement auto-expand-on-warning per D-UI2. If the WebcamCard is in minimized state when `face.not_detected` fires, the card auto-expands BEFORE the red toaster appears. Emits `webcam_view.visibility_changed` with `{ visible: true, trigger: "auto_expand_on_warning" }`. **Verify:** integration test confirms the auto-expand timing.
- [ ] 8.4 Implement the mic-disabled persistent toaster (D-SM2-pre, D-SM6). MD3 warning variant (yellow/orange), positioned at the bottom. Text: *"Your microphone has been disabled. Please re-enable it for full assessment monitoring."* Dismisses when the mic stream resumes. **Verify:** integration test confirms the toaster appears on mic disable and dismisses on re-enable.
- [ ] 8.5 Implement the multiple-faces persistent toaster (D-SM6). MD3 warning variant. Text: *"Multiple faces detected. Please ensure you are alone."* Dismisses when only one face is detected again. **Verify:** integration test confirms the toaster lifecycle.
- [ ] 8.6 Implement the pause-state UI per D-SM2 + D-SM10. When session transitions to PAUSED.face_not_detected (after 3 warnings), the entire screen blurs, items hide, and the existing WS-disconnect overlay text appears. On face restoration, the screen un-blurs. **Verify:** integration test confirms the pause + resume UX.

## 9. rrweb session recording (capability: `rrweb-session-recording`)

- [ ] 9.1 Add `rrweb` (or `@rrweb/record`) as a runtime-web dependency. Initialize the recorder in `apps/runtime-web/src/integrity/rrweb-capture.ts`. **Verify:** the recorder starts on session.started and stops at any terminal state.
- [ ] 9.2 Implement batching per D-RR1: collect events into a buffer; flush every 5s OR every 100 events whichever first. **Verify:** unit test with a synthetic stream of events confirms batching timing.
- [ ] 9.3 Implement HTTPS upload to the runtime's `POST /api/telemetry/rrweb-batch` endpoint. Include the magic-link JWT in the Authorization header for validation. **Verify:** integration test confirms uploaded batches reach S3.
- [ ] 9.4 Implement in-memory buffering on upload failure. If the HTTPS upload returns 5xx or times out, hold the batch + retry on next interval. WS disconnect does NOT lose rrweb data because upload uses HTTPS not WS. **Verify:** integration test with a flaky network confirms eventual delivery.
- [ ] 9.5 Implement `services/telemetry/src/rrweb-upload.ts` — the HTTPS handler that receives a batch, validates the JWT, looks up the session, writes the batch to S3 at `<rrweb-bucket>/<org_id>/<session_id>/events-<batch_seq>.json`. Concatenate batches to a single object at session-end via S3 multipart, OR keep batches as separate objects + concatenate at analyzer-read time. **Verify:** integration test confirms multiple batches across a session end up readable as a single contiguous event stream.
- [ ] 9.6 Configure rrweb to capture: DOM mutations, mouse moves, scrolls, clicks, inputs (with `inputs: { include: ['textarea[data-rrweb-capture="true"]'] }` to only capture the short-answer textarea content — exclude the MC option labels which contain test content we don't want leaking out). **Verify:** captured event stream excludes MC choice text.
- [ ] 9.7 Emit `rrweb.recording_started` and `rrweb.recording_stopped` events to the audit log. **Verify:** integration test confirms emission.

## 10. Post-session integrity analyzer (capability: `post-session-integrity-analyzer`)

- [ ] 10.1 Replace the `services/analyzer/src/handler.ts` Phase 1 stub with the real implementation. SQS message → fetch all session data (audit log + items + responses + campaign + Persona result + rrweb pointer) → construct LLM prompt → call Bedrock → validate response → persist IntegritySummary. **Verify:** unit test with mocked Bedrock response covers happy path.
- [ ] 10.2 Implement the LLM prompt template in `services/analyzer/src/prompts/v1.ts` per D-I9. Includes the SYSTEM message (judgment-language prohibition), the USER message (full session context), the output schema specification, the screenshot-round-trip detection guidance, the timeline-segments coverage requirement. **Verify:** the prompt text contains all documented anchor phrases; manual review confirms no judgment language anywhere.
- [ ] 10.3 Implement structured-output validation. The Bedrock response is parsed via Zod against the IntegritySummary schema. On parse failure, retry with a "fix your output to match the schema" follow-up. **Verify:** unit test with a malformed Bedrock response covers the retry path.
- [ ] 10.4 Implement timeline-segment coverage validation per D-RR3. The segments must cover the entire session timeline — fill gaps with `green` segments. If the LLM produces gappy output, the analyzer programmatically fills gaps with green segments before persisting. **Verify:** unit test with deliberately gappy LLM output confirms the post-processing fills correctly.
- [ ] 10.5 Persist the IntegritySummary to `hiringiq-pilot-integrity-summaries` table with `analyzer_version: "v1@{model_id}"`. **Verify:** integration test confirms the record appears with the documented schema.
- [ ] 10.6 Implement SQS DLQ + retry policy. 3 retry attempts with exponential backoff. On final failure, the SQS message moves to a DLQ; a CloudWatch alarm fires. **Verify:** integration test with a deliberately-failing Bedrock invocation confirms DLQ delivery + alarm.
- [ ] 10.7 Skip analyzer for DECLINED sessions per D-AL4. The session.ending event handler only enqueues the analyzer SQS message when state transitions to COMPLETED or EXPIRED. **Verify:** integration test confirms a DECLINED session does NOT produce an analyzer SQS message.
- [ ] 10.8 Implement model-configurability via SSM. Read `/hiringiq/pilot/runtime/integrity/analyzer-model` at Lambda init; cache for the Lambda lifetime. **Verify:** changing the SSM value + cold-starting the Lambda picks up the new model.

## 11. Privacy notice update + consent landing tweaks

- [ ] 11.1 Update the privacy notice content (stub from Phase 1) with biometric-retention + right-to-deletion language. 180-day retention + recruiter-mediated deletion + Persona biometric stays with Persona + face-api.js embedding lives in our DDB + rrweb recording in S3. **Verify:** notice is reviewable by legal; the consent landing's link target loads the new content.
- [ ] 11.2 Update the runtime-web bundle to load `face-api.js` models at the consent landing instead of at intro pre-warm (Phase 1 had the pre-warm placeholder). Earlier load means the model is ready before assessment-interface render. **Verify:** Network tab in browser shows model files loading on landing.

## 12. End-to-end verification

- [ ] 12.1 Smoke test full integrity flow: complete an assessment with deliberate cheating signals (out of frame for 20s + paste into short-answer + tab-switch + screenshot attempt + clock jump). **Verify:** the IntegritySummary produced reflects all these signals in `flagged_moments`; the timeline_segments include yellow/red bands at the appropriate timestamps; the rrweb recording exists and is playable via the rrweb-player library.
- [ ] 12.2 Test the Persona selfie enrollment pipeline end-to-end. **Verify:** the applicant's selfie appears in S3; the embedding appears as a DDB attribute on the applicant record; face-api.js comparison during the assessment works (i.e., a low-confidence frame fires `face.identity_mismatch` after 3 consecutive low-confidence checks).
- [ ] 12.3 Test the 3-warning face-not-detected escalation. Cover camera, wait 5s → warning #1 toaster. Wait 15s → warning #2. Wait 15s → warning #3. Wait 15s → PAUSE.face_not_detected, screen blurs. Uncover camera → un-pauses, screen un-blurs. **Verify:** the state transitions + toaster sequence + pause-time-budget consumption are all correct.
- [ ] 12.4 Test the screenshot detection on Windows + macOS. **Verify:** Windows PrintScreen fires `screenshot.print_screen_detected`; macOS Cmd+Shift+4 does NOT (acknowledged limitation).
- [ ] 12.5 Test the rrweb recording playback. After a completed session, fetch the rrweb event stream from S3 + load into rrweb-player. **Verify:** the playback faithfully reconstructs the session including the WebcamCard, item rendering, mouse movements, and timer.
- [ ] 12.6 Test the analyzer on a CLEAN session (no integrity signals). **Verify:** IntegritySummary has `integrity_score` near 100, `behavioral_summary` describes a clean session factually, `flagged_moments` empty, `timeline_segments` all green.
- [ ] 12.7 Test the analyzer DOES NOT use judgment language. **Verify:** grep the `behavioral_summary` from 10 representative sessions for the strings "cheat", "fraud", "dishonest" — they should not appear; only factual descriptions like "out of frame", "paste detected", "long idle period followed by burst of input."
- [ ] 12.8 Test SQS DLQ + alarm path. **Verify:** poisoned message → DLQ → CloudWatch alarm fires.
- [ ] 12.9 Test biometric-deletion Lambda. Invoke it manually with a test applicant_id. **Verify:** selfie deleted from S3; embedding deleted from DDB attribute; rrweb recordings deleted; IntegritySummary marked deleted; recruiter_access events for that applicant retained.
- [ ] 12.10 Test all silent integrity events appear in the audit log but do NOT surface in the applicant UX. **Verify:** integration test confirms `tab.visibility_lost`, `screenshot.print_screen_detected`, `face.identity_mismatch`, `face.attention_off_screen`, `audio.voice_activity_detected`, `media_devices.suspicious_present`, `display.secondary_present`, `extension.suspicious_activity` all log without any toaster or UI change.

## 13. Documentation

- [ ] 13.1 Update `services/analyzer/README.md`: the analyzer's responsibility, the prompt template versioning model, the Bedrock model SSM parameter, the DLQ runbook. **Verify:** README enumerates these.
- [ ] 13.2 Update `services/telemetry/README.md`: the event taxonomy, the dual-timestamp + IP capture, the S3 archive pattern, the rrweb upload endpoint. **Verify:** README documents these.
- [ ] 13.3 Update `infra/terraform/README.md`: the biometric-storage + rrweb-storage + audit-archive modules, the Object Lock compliance-mode commitment, the right-to-deletion Lambda invocation runbook. **Verify:** README documents these.
- [ ] 13.4 Add a `services/applicant/RUNBOOK.md` documenting the Persona selfie retrieval failure modes + manual remediation (the recruiter dashboard in Phase 3 will provide a self-service version). **Verify:** runbook covers the documented failure paths.

## Dependencies on other proposals

**Hard dependencies:**
- This workspace's Phase 1 (`add-assessment-runtime-foundation`) — provides the four Lambda services, the WS endpoint, the sessions + audit-events tables, the Persona Phase-1 implementation.
- Platform-shell proposals #1–#11 — same as Phase 1 (this proposal does not introduce additional cross-workspace coupling beyond Phase 1's).

**Recommended apply order:**
Same chain as Phase 1, then `add-assessment-runtime-foundation`, then **this proposal**, then `add-recruiter-assessment-review` (Phase 3).

**Unblocks:**
- `add-recruiter-assessment-review` (Phase 3): consumes IntegritySummary + rrweb recordings + audit events produced by this proposal.
