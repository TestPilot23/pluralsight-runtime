# Proposal — Assessment Runtime Integrity

## Why

Phase 1 (`add-assessment-runtime-foundation`) delivers an applicant who can complete an assessment end-to-end, but the platform has **no integrity instrumentation** — face-api.js, audio VAD, browser-signal capture, rrweb recording, biometric storage, and the post-session analyzer are all absent. A pilot recruiter looking at a completed session today would see only the session-lifecycle events (started, items submitted, completed) and the applicant's responses — no signals about whether the applicant was actually present, attentive, alone, or honest.

This proposal layers the **full anti-cheat instrumentation + post-session integrity analyzer** on top of the Phase 1 foundation. After this proposal applies, every assessment session captures rich behavioral signals via the WebSocket + dedicated HTTPS endpoints, stores rrweb session recordings + biometric data per the documented retention policies, and produces a post-session `IntegritySummary` via Bedrock LLM analysis that gives the recruiter a numerical score + descriptive behavioral summary + timeline-segmented flagged moments to scrub through.

What this proposal delivers, in recruiter-visible terms (once Phase 3 ships the dashboard):

- Every session has a full audit log of integrity signals (presence, attention, paste/copy, tab/focus, screen-share signals, screenshot attempts, etc.)
- Every session has a pixel-faithful rrweb recording for replay
- Every session has a `IntegritySummary` with a 0-100 score + a behavioral summary describing what happened factually + timestamped segments colored green/yellow/red so the recruiter can scrub through hotspots

What this proposal does NOT deliver:

- The recruiter dashboard UI to view any of the above — that's Phase 3 (`add-recruiter-assessment-review`). At the end of Phase 2, the data is produced and stored; viewing requires Phase 3.

## What Changes

This proposal layers integrity capture + analysis on top of the foundation laid in Phase 1. Sequenced by dependency:

- **Build the event taxonomy + envelope** (D-AL1, D-AL2, D-AL1a). Every audit-log event uses the documented envelope shape with dual timestamps (server + client), category, event_type, payload, schema_version. The full taxonomy from D-AL2 is in scope: identity / session-lifecycle / integrity-signal / assessment categories with their concrete event types. The clock-skew detection mechanic (D-AL1a) emits `clock.skew_detected` events on both baseline-exceeded and mid-session-drift thresholds.
- **Provision the S3 audit archive** with Object Lock in compliance mode + KMS encryption + Glacier-after-30-days lifecycle (D-AL3). Every audit event is written to BOTH the DDB hot path (already provisioned in Phase 1) AND this immutable S3 archive. Tamper-evidence is in place from this proposal's apply onward.
- **Persona selfie retrieval + face-api.js enrollment** (D-P4). Persona webhook handler now extends Phase 1's logic to call Persona's Inquiry API after successful verification, download the selfie via signed URL, store it in S3, generate a face-api.js embedding (512-float vector), and persist the embedding as a DDB attribute on the applicant record. **Hard precondition:** Pluralsight's Persona contract is on the Hosted/API-based Flow (Q-PSL-1). If Embedded Flow is confirmed instead, this capability needs ~half a day of swap-back work to capture a separate enrollment photo on the device-check screen.
- **Integrate face-api.js for in-assessment presence + attention + identity monitoring.** The TinyFaceDetector model is downloaded at intro-screen pre-warm (Phase 1 already does this in stub form — this proposal turns on the consumer). During the assessment, face-api.js runs at ~5s cadence emitting the documented events: `face.not_detected` (escalating warning per D-SM2), `face.detection_restored`, `face.identity_mismatch` (silent, debounced — N consecutive low-confidence matches against the enrollment embedding), `face.attention_off_screen` (silent, head pose pitch >30° down for 10+ sec, targets the phone-photo-to-LLM threat), `face.multiple_detected` (persistent warning).
- **Add audio Voice Activity Detection (VAD)** (D-AUD1). Web Audio API + AnalyserNode runs in the browser; sustained voice activity (>2s of speech-like signal) emits `audio.voice_activity_detected` silently. Audio waveforms are **never stored** server-side — VAD is in-memory client-side only.
- **Add browser instrumentation events** (D-SM2-pre full pattern table). Capture and emit silent events for: `tab.visibility_lost` / `tab.visibility_restored` (visibilitychange API), `window.focus_lost` / `window.focus_restored` (blur/focus events), `picture_in_picture.entered` / `picture_in_picture.exited`, `media_devices.changed`, `media_devices.suspicious_present` (matches against known screen-share software fingerprints: OBS, Snap, ManyCam, XSplit, Loopback, BlackHole, Aggregate, NDI), `display.secondary_present` (via `window.screen` — NO Window Management API permission prompt at MVP per D-DC1), `display.permission_denied`, `paste.detected` + `copy.detected` (already partially wired in Phase 1; this phase ensures full coverage including content payload), `ip.changed`, `clock.skew_detected`, `screenshot.print_screen_detected` (Windows-reliable, macOS unreliable), `extension.suspicious_activity` (best-effort DOM-mutation / JS-injection detection).
- **Add the WebcamCard warning system** (D-UI2 in full form, D-UI7). The card from Phase 1 gains the warning state: red 3px border + red toaster with the exact wording *"We can't detect your face on camera. Please recenter or check your webcam — otherwise the assessment will end."* The auto-expand-on-warning behavior activates — if the card is in the minimized chip state when `face.not_detected` fires, the card auto-expands BEFORE the red toaster appears so the applicant can see the framing problem. The 3-warning escalation at 15s intervals → PAUSE.face_not_detected transition. Mic-disabled persistent toaster: *"Your microphone has been disabled. Please re-enable it for full assessment monitoring."* Multiple-faces persistent toaster: *"Multiple faces detected. Please ensure you are alone."*
- **Add integrity-driven pause states** (D-SM2, D-SM6, D-SM7). The session state machine from Phase 1 gains new PAUSED reasons: `PAUSED.face_not_detected`, `PAUSED.webcam_stream_unavailable`. Pause-time budget enforcement (D-SM3, 90s per item) applies across all pause reasons; budget already provisioned in Phase 1, now consumed by the new pause reasons.
- **Add rrweb session recording** (D-RR1, D-RR2). rrweb-record runs in the runtime React app from session.started through session.completed (or terminal state). Batched uploads every 5s OR every 100 events via the dedicated HTTPS endpoint at `POST /api/telemetry/rrweb-batch` (provisioned in Phase 1 as a 501 stub; this phase implements the handler). S3 storage with KMS + 180-day lifecycle.
- **Add biometric storage with 180-day retention** (D-P5). S3 bucket `hiringiq-pilot-biometric-raw/` with KMS encryption + restricted IAM + object-level access logging for the Persona selfie and (via D-RR2) rrweb recordings. Face-api.js embedding stored as DDB attribute on the applicant record. Automated lifecycle policies for 180-day deletion. Recruiter-mediated right-to-deletion flow (the dashboard endpoint that triggers coordinated deletion lands in Phase 3; the underlying deletion Lambda + IAM is provisioned in this phase).
- **Implement the post-session integrity analyzer** (D-AL4, D-AL5, D-AL6, D-RR3, D-RR5). The analyzer-service Lambda from Phase 1 (which was a no-op stub) gets its real implementation. Triggered by SQS messages on session-end transitions (COMPLETED, EXPIRED — but NOT DECLINED). Reads all audit-log events + campaign metadata + items the applicant saw (with `answer` STRIPPED) + applicant responses + Persona result + face-detection confidence trajectory. Calls Bedrock (Nova Pro at MVP; configurable via SSM). Produces `IntegritySummary` with `integrity_score`, `behavioral_summary`, `flagged_moments`, `timeline_segments` (green/yellow/red continuous bands covering the full session timeline), `rrweb_recording_s3_key`. Persists summary to a new DDB table `hiringiq-pilot-integrity-summaries`. LLM prompt explicitly forbids judgment language ("describe, don't judge") and includes specific guidance for the screenshot-to-LLM round-trip threat signature.

## Capabilities

### New Capabilities

- `integrity-event-taxonomy`: the full audit-log envelope shape, dual timestamps, the 4-category event taxonomy, clock-skew detection mechanic, S3 immutable archive with Object Lock.
- `persona-selfie-enrollment`: D-P4 implementation — Persona Inquiry API retrieval of selfie + S3 storage + face-api.js embedding generation + DDB attribute persistence.
- `face-detection-instrumentation`: face-api.js TinyFaceDetector running at ~5s cadence emitting presence / identity-mismatch / attention-off-screen / multiple-faces signals.
- `audio-voice-activity-detection`: in-browser Web Audio API VAD emitting `audio.voice_activity_detected` without storing waveforms.
- `browser-instrumentation`: tab/focus/PiP/media_devices/screenshot/copy/paste/clock-skew/extension signals.
- `webcam-card-warning-state`: red border + red toaster + auto-expand-on-warning behaviors layered onto the Phase 1 WebcamCard; the three-UX-pattern taxonomy (Escalating / Persistent / Silent) per D-SM2-pre.
- `rrweb-session-recording`: rrweb-record capture + dedicated HTTPS batch upload + S3 storage + 180-day retention.
- `biometric-storage`: S3 bucket + KMS + IAM + lifecycle + right-to-deletion infrastructure for Persona selfie + face-api.js embedding + rrweb recording.
- `post-session-integrity-analyzer`: SQS-triggered Bedrock-powered analyzer producing the IntegritySummary with score + behavioral_summary + flagged_moments + timeline_segments + rrweb pointer.

### Modified Capabilities

- `assessment-session-state` (from Phase 1): adds `PAUSED.face_not_detected` and `PAUSED.webcam_stream_unavailable` as new pause reasons; pause-time budget (D-SM3) now consumed by these.
- `assessment-interface` (from Phase 1): the WebcamCard gains the warning state visuals (red border + red toaster) and the auto-expand-on-warning behavior. The runtime now instantiates rrweb-record from `session.started`. The Item rendering chrome installs the keyboard/clipboard event listeners that drive `paste.detected` / `copy.detected` / `screenshot.print_screen_detected`.
- `applicant-consent-landing` (from Phase 1): the privacy notice link target gets fleshed out with biometric-retention + right-to-deletion language (180 days; deletion via recruiter request).
- `persona-identity-verification` (from Phase 1): the webhook handler now extends to retrieve the selfie via the Persona Inquiry API and trigger the face-api.js enrollment pipeline.

## Impact

- **Code:**
  - `apps/runtime-web/src/integrity/` — new directory for face-api.js, VAD, browser-instrumentation, rrweb modules.
  - `apps/runtime-web/public/face-api-models/` — face-api.js TinyFaceDetector model files (~6 MB compressed) served from S3 origin via the runtime CloudFront distribution.
  - `services/telemetry/src/event-handlers/` — handlers for each event type; the rrweb HTTPS endpoint handler.
  - `services/applicant/src/persona-enrollment.ts` — selfie retrieval + embedding generation pipeline (uses `@vladmandic/face-api` or `face-api.js` on the Lambda — TinyFaceDetector + 68-point landmarks + face recognition net).
  - `services/analyzer/src/handler.ts` — the real implementation, replacing the Phase 1 stub. Bedrock SDK call, prompt construction, IntegritySummary persistence.
  - `services/analyzer/src/prompts/` — the LLM prompt template with the judgment-language prohibition + screenshot-round-trip signature guidance.
- **AWS resources added:**
  - 1 new S3 bucket `hiringiq-pilot-rrweb-recordings/` (KMS, 180-day lifecycle)
  - 1 new S3 bucket `hiringiq-pilot-biometric-raw/` (KMS, restricted IAM, object-level access logging, 180-day lifecycle)
  - 1 new S3 bucket `hiringiq-pilot-audit-archive/` (Object Lock in compliance mode, KMS, Glacier-after-30-days)
  - 1 new DynamoDB table `hiringiq-pilot-integrity-summaries` (PK=session_id, GSI on applicant_id, GSI on org_id for cross-applicant queries)
  - 1 new EventBridge Scheduler schedule group for clock-skew threshold updates (the SSM parameters are read; scheduler not strictly needed at MVP — provisioned for v2)
  - 1 IAM role addition: analyzer-service Lambda gets `bedrock:InvokeModel` permission scoped to the documented model ARNs
  - 1 IAM role addition: applicant-service Lambda gets new S3 PutObject permission on the biometric-raw bucket + a Persona Inquiry API egress allow rule (NAT or VPC endpoint configuration depending on the AWS networking decision in Phase 1)
  - 1 new SSM parameter namespace `/hiringiq/pilot/runtime/integrity/*` for tuneable thresholds (face-detection cadence, identity-mismatch confidence threshold, attention-off-screen pitch threshold, clock-skew baseline + drift thresholds, etc.)
- **Cost (pilot scale, Phase 2 on top of Phase 1):**
  - Persona selfie retrieval: same per-applicant $1.50 cost as Phase 1 (no extra inquiry; we just call the Inquiry API after the existing inquiry completes — Persona doesn't charge for that)
  - Bedrock LLM (Nova Pro): ~$0.05–$0.10 per assessment per project memory; ~$5-10/mo at pilot scale
  - S3 storage (biometric raw + rrweb + audit archive): ~60GB hot S3 + Glacier transitions → ~$2-3/mo
  - Additional DDB writes (audit events at higher rate from integrity signals): negligible at on-demand pricing
  - Total incremental Phase 2 cost: ~$10-15/mo at pilot scale
- **Dependencies introduced:**
  - `face-api.js` or `@vladmandic/face-api` (MIT) — bundled in the runtime React app + Lambda
  - `rrweb` + `rrweb-player` (MIT) — bundled in the runtime React app; player will be used in Phase 3
  - `@aws-sdk/client-bedrock-runtime` — analyzer-service Lambda
- **Cross-workspace dependencies:** none new — all data flows within the runtime workspace at Phase 2. Phase 3 will read the IntegritySummary table from the recruiter dashboard.
- **Downstream unblocks:** Phase 3 (`add-recruiter-assessment-review`) needs the IntegritySummary table + rrweb recordings + audit-events table populated.
- **Operational risk introduced:**
  - face-api.js TinyFaceDetector ~6MB model bundle adds initial load latency. Mitigated by intro-screen pre-warm.
  - Bedrock LLM analysis is async; if the Lambda fails, the recruiter dashboard shows "Analysis pending…" until retry succeeds. SQS DLQ catches messages that fail repeatedly.
  - Biometric storage triggers BIPA/CPRA/GDPR Article 9 compliance considerations. The 180-day retention + right-to-deletion flow + access logging together meet the documented compliance posture. Privacy notice on the consent landing must reflect this.
- **Out of scope (handled in Phase 3 or deferred):**
  - **Recruiter dashboard UI** for viewing the IntegritySummary + audit log + rrweb playback → Phase 3
  - **Recruiter-initiated right-to-deletion flow** (the dashboard button + cross-bucket deletion endpoint) → Phase 3 (Phase 2 provisions the underlying Lambda + IAM but doesn't expose the trigger)
  - **Voice biometric identification** ("is this the applicant's voice?") → defer past pilot
  - **Mouth-correlation for VAD** (audio detected + applicant's mouth closed = strong background-voice signal) → defer past pilot
  - **Audio waveform storage / transcription** → never in scope (privacy + compliance posture)
  - **Hardware capture / external screen-share detection** → not possible from the browser (documented in design)
  - **Self-serve right-to-deletion UI for applicants** → defer to v2; pilot uses recruiter-mediated

## Open Questions

- **OPEN:** Q-PSL-1 — Pluralsight's Persona contract flow (Hosted/API-based vs Embedded). This proposal HARD-DEPENDS on Hosted/API-based; if Embedded is confirmed, ~½ day swap-back to capture our own enrollment photo on the device-check screen instead.
- **OPEN:** Bedrock model choice — Nova Pro vs Claude 3.5 Sonnet vs Claude Haiku for the analyzer. Cost vs quality tradeoff. Default Nova Pro at MVP; SSM-configurable. Decision deferrable to the analyzer-service implementation tasks below.
- **OPEN:** face-api.js library variant — original `face-api.js` (less actively maintained) vs `@vladmandic/face-api` (active fork with better TypeScript support). Lean: `@vladmandic/face-api`. Implementation-time pick; doesn't affect spec.
- **OPEN:** Clock-skew SSM parameter initial values. Locked design says baseline threshold = 5000ms, mid-session drift threshold = 2000ms (D-AL1a). Tuneable in production based on pilot data.
