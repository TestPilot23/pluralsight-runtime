# Proposal — Assessment Runtime Foundation

## Why

The HiringIQ platform-shell workspace ([`../PluralSight-POC/`](../../../PluralSight-POC/)) can invite applicants to assessments, but the actual assessment-taking experience is a placeholder route (`/assess/runtime`) per platform-shell proposal #11. Today an applicant who clicks their magic link lands on a stub page that does nothing.

This proposal builds the **foundation of the assessment runtime**: an applicant can click their magic link, verify their identity via Persona, test their devices, see the assessment intro, complete a set of assessment items, and reach a completion screen. The full instrumentation layer (face-api.js, audio VAD, screen-share signals, rrweb, integrity analyzer) is deferred to the next proposal (`add-assessment-runtime-integrity`); the recruiter-facing review experience is deferred to the third (`add-recruiter-assessment-review`).

The phasing exists because the assessment-runtime work is large (60+ design decisions captured in the sibling shell's [`thoughts/assessment-design-decisions.md`](../../../../PluralSight-POC/thoughts/assessment-design-decisions.md)). Splitting into three vertical-value slices keeps each proposal reviewable and lets the team validate the assessment-completion path end-to-end before adding integrity overhead.

What this proposal delivers, in user-visible terms:

- An applicant clicks a magic link → lands on a consent page → grants camera + mic permissions → completes Persona ID verification → tests their devices → sees the assessment intro → answers the locked questions for their campaign → sees a thank-you completion screen.
- The session is fully server-authoritative (timer enforcement, item delivery, grading happen server-side; the client is presentational).
- The session state machine is wired (NOT_STARTED → IN_PROGRESS → COMPLETED, plus DECLINED and EXPIRED terminal states).
- The four logical microservices (`applicant-service`, `assessment-service`, `telemetry-service`, `analyzer-service`) are deployed as Lambdas with their public API surfaces defined.

What is NOT in this proposal (deferred):

- Behavioral integrity signals (face-api.js, VAD, paste/copy/tab-blur capture, screen-share signals, rrweb recording, biometric storage)
- Post-session integrity analyzer (Bedrock LLM, IntegritySummary production)
- Recruiter dashboard additions (session detail view, timeline scrubber, replay viewer)
- The `telemetry-service` WS endpoint exists but only handles session-lifecycle events at this phase; integrity signal ingestion lands in Phase 2

## What Changes

The full applicant-facing flow from magic-link click through completion. Sequenced top-to-bottom:

- **Provision the new sibling AWS workspace.** Separate CloudFront distribution for the runtime React app (`runtime.hiringiq-pilot.cloudfront.net` until a custom domain is assigned). Separate Terraform stack at `infra/terraform/` reusing modules from the platform shell (`tenant-data-store`, `runtime-secrets`, `runtime-observability`). New DynamoDB tables: `hiringiq-pilot-sessions`, `hiringiq-pilot-ws-connections`, `hiringiq-pilot-audit-events` (the audit-events table is shared with Phase 2; Phase 1 writes session-lifecycle events into it). All resources in `us-east-1` under the existing `kms-pluralsight` awscli profile.
- **Deploy four Lambda services with clear API boundaries.** `applicant-service` and `assessment-service` behind API Gateway HTTP API; `telemetry-service` behind API Gateway WebSocket API + an HTTPS endpoint for future rrweb upload (HTTPS endpoint exists at this phase but is a stub returning 501 Not Implemented — wired in Phase 2); `analyzer-service` is provisioned as an SQS-triggered Lambda but its handler is a no-op stub at this phase (it doesn't run because nothing triggers the SQS queue yet — full implementation in Phase 2).
- **Implement magic-link JWT validation.** Every page load runs the validation chain from D-MAG1: JWT signature → 15-day TTL → campaign-active → viewport ≥ 1024px → supported browser → session state. Failures route to the appropriate terminal page (D-MAG2 expired, D-MAG3 closed, D-DC5 small screen, D-DC6 unsupported browser).
- **Build the consent landing page** (D-DC1). React app at `runtime.<host>/`. Requests camera + mic permissions on mount via `getUserMedia`. Shows the T&C consent checkbox + recruiter intro + brief assessment overview. Continue CTA enabled when both permissions granted AND T&C checked. Opens the WebSocket connection per D-IS5 so the telemetry pipeline is live before Persona.
- **Integrate Persona for identity verification** (D-P1, D-P2, D-P3). Apply the `usePersonaVerification` SSM-controlled feature flag — `true` (default) calls Persona; `false` skips the entire screen + returns `verified: true`. When `true`, the applicant runs through Persona's Inquiry SDK. Backend handles the webhook, evaluates the result, applies the 3-strike counter (API errors don't count), and routes failures to the documented retry paths or to the Unable-to-Verify terminal state. **NOTE:** Persona-selfie-as-enrollment per D-P4 is deferred to Phase 2 — at this phase, Persona's selfie image is not retrieved (face-api.js enrollment is the integrity workstream's concern).
- **Build the device-check screen** (D-DC2). Webcam status card (informational only — Persona verified the camera), microphone live volume meter, speaker self-attest test. The CTA is "Ready" per the original brief. Continuation never blocked.
- **Build the assessment intro screen** (D-IS1 through D-IS5). Assessment metadata card (campaign name, question count, estimated duration from `assessment_total_time_seconds` per item-bank D17), consent checkbox (D-IS2), recruiter contact + accommodations line (D-IS3), Begin Assessment CTA + Decline button with two-step confirmation (D-IS4). Opens the assessment session and transitions state to IN_PROGRESS on Begin Assessment click.
- **Build the assessment interface** (D-UI1 through D-UI8). Option B layout — webcam corner card at top-left (basic version at this phase — no auto-expand-on-warning because there's no warning system yet; the corner card just shows the live preview), single-column reading area for the question, choices/textarea below, Submit at bottom-right, timer top-right when applicable. Item-type renderer dispatches on `item.type` (D-UI4 — handles all three MVP types `multiple_choice_single`, `multiple_choice_multiple`, `short_answer`). Submit-disabled-until-valid (D-UI5). Timer expiry behavior (D-UI6). Instant transition from Begin Assessment (D-UI8).
- **Implement the session state machine** (D-SM1 through D-SM10, minus integrity-driven pauses). States: `NOT_STARTED`, `IN_PROGRESS`, `PAUSED.ws_disconnect`, `ABANDONED`, `COMPLETED`, `DECLINED`, `EXPIRED`. PAUSED.face_not_detected and other integrity-driven pauses land in Phase 2. Server-authoritative timer enforcement. Item display order randomized per applicant, deterministic from `session_id` (D-SM9). "Seen" taxonomy (D-SM4) tracked server-side. Intentional close treated identically to crash (D-SM5). ABANDONED is recoverable; magic link expiry promotes to EXPIRED terminal state (D-SM7).
- **Implement grading.** Server fetches item from item-bank tables (`hiringiq-pilot-items` from platform-shell proposal #7), receives applicant's answer over the WS, grades server-side, persists `served_answered` state on the campaign-session. Answer field of the item record is NEVER sent to the client. For `multiple_choice_multiple`, grading does set-equality on the sorted-unique index array.
- **Build the completion screen** (D-CS1 through D-CS3). Renders when server transitions session to COMPLETED. Animated success icon, thank-you message, recruiter contact info, "you may now close this window" instruction. Client emits final `session.completed` event and closes the WebSocket. Magic-link JWT stays valid until natural expiry; applicant can re-load the screen during that window.
- **Audit-log writes for session-lifecycle events.** The audit log table is populated with the session-lifecycle events from D-AL2 (`session.intro_screen_viewed`, `session.started`, `session.paused.ws_disconnect`, `session.resumed`, `session.completed`, `session.abandoned`, `applicant.declined.confirmed`, etc.) plus the magic-link / pre-flight events (`session.landing_page_viewed`, `session.access_attempted_with_expired_link`, etc.). Per-event IP + dual-timestamp (server + client) per D-AL1 / D-AL1a. **Integrity-signal events** (`face.*`, `tab.*`, `paste.*`, etc.) and the **assessment** category events (`item.displayed`, `item.submitted`) ARE in scope for this proposal — they fire on the natural state transitions; the analyzer just doesn't consume them until Phase 2.

## Capabilities

### New Capabilities

- `runtime-workspace-scaffold`: the sibling AWS workspace itself — separate Terraform stack, CloudFront distribution, four Lambda services as deployable units, three new DDB tables (sessions, ws_connections, audit_events).
- `magic-link-validation`: the sequential gate-check at every magic-link page load (JWT → campaign-active → viewport → browser → session state); the four pre-flight block pages (expired, closed, small-screen, unsupported-browser).
- `applicant-consent-landing`: the first applicant-facing screen — T&C consent + camera/mic permission grants + opening the WS connection.
- `persona-identity-verification`: applicant runs through Persona's Inquiry SDK; backend handles webhook + 3-strike counter + reason-code routing + `usePersonaVerification` feature flag.
- `device-check`: webcam status confirmation + mic live meter + speaker self-attest; "Ready" CTA never blocked.
- `assessment-intro`: pre-assessment metadata + consent + Begin Assessment + Decline flow + DECLINED terminal state.
- `assessment-interface`: the main assessment runtime UI — Option B layout, webcam corner card (basic — no warning system at this phase), item-type renderer dispatch, timer, Submit.
- `assessment-session-state`: server-authoritative session state machine (NOT_STARTED → IN_PROGRESS → COMPLETED + terminal branches); per-applicant random ordering; "seen" taxonomy; pause-on-WS-disconnect + resume; ABANDONED + EXPIRED.
- `assessment-completion`: COMPLETED screen + final WS event + connection close + magic-link-stays-valid.

### Modified Capabilities

None at this phase. The platform-shell's `add-applicant-assessment-landing` (proposal #11) replaces its stub `/assess/runtime` route with a redirect to the new runtime CloudFront distribution. That platform-shell-side modification is captured in **its** OpenSpec workspace under that proposal's MODIFIED section — handled as a cross-workspace edit during apply, not as a modification declared by this proposal.

## Impact

- **Code:** New sibling monorepo at `/Users/jgrenard/workspace/projects/PluralSight-POC-runtime/`. Subdirectories: `apps/runtime-web` (React 18 app), `services/applicant` + `services/assessment` + `services/telemetry` + `services/analyzer` (each a separate Lambda code module with `public.ts` API surface), `infra/terraform/` (modules + the `pilot` env composition), `packages/shared` (cross-service types — session, item, audit-event shapes).
- **AWS account:** Single account, awscli profile `kms-pluralsight`. All resources in `us-east-1`. Reuses the existing Cognito user pool from the platform shell for recruiter org-level access (Phase 3); applicant identity at this phase is magic-link JWT.
- **AWS resources added:**
  - 1 new CloudFront distribution (runtime-web)
  - 1 new S3 bucket (runtime-web static hosting)
  - 1 new API Gateway HTTP API (applicant-service + assessment-service)
  - 1 new API Gateway WebSocket API (telemetry-service)
  - 4 new Lambda functions (applicant, assessment, telemetry, analyzer-stub)
  - 3 new DynamoDB tables (sessions, ws_connections, audit_events)
  - 1 new SQS queue (analyzer queue — provisioned but unused at this phase)
  - 0 new Secrets Manager secrets for magic-link signing (per Path Z reconciliation: runtime SHARES the platform-shell's existing `hiringiq/<env>/applicant-magic-link-secret` — see D-F6). 1 SSM parameter namespace (`/hiringiq/pilot/runtime/*`).
  - 1 new CloudWatch log group per Lambda (4 total)
  - 2 new CloudWatch alarms (Lambda errors, API Gateway 5xx)
  - 1 new SNS topic (runtime-alerts) — no subscriptions until alarm-distribution email is confirmed
- **Cost (pilot scale, Phase 1 only):** Under $1/month. All Lambda + API Gateway + DDB at on-demand pricing; no biometric storage yet (Phase 2). The Persona API cost per inquiry ($1.50/check) is incurred when `usePersonaVerification=true` and an applicant runs the flow — that cost was always going to land at Phase 1.
- **Dependencies introduced:** `@persona/inquiry-sdk` (or Persona's hosted inquiry redirect, depending on Q-PSL-1 resolution), React 18+, Vite, Material UI v6, Zod, `ws` + `aws-lambda-fastify` (or direct Lambda handlers), Terraform `>= 1.7`, AWS provider `~> 5.x`.
- **Cross-workspace dependencies:**
  - Reads from platform-shell's `hiringiq-pilot` table for campaign + applicant records
  - Reads from `hiringiq-pilot-items` + `hiringiq-pilot-item-skill-index` (platform-shell's item-bank proposal #7) to serve items
  - Shares Cognito user pool with platform shell for recruiter org-level access (Phase 3)
- **Downstream unblocks:** `add-assessment-runtime-integrity` (Phase 2) can begin — needs the session state machine + audit-events table + WS connection in place.
- **Operational risk introduced:**
  - Magic-link JWT signing key is a new secret to manage (created in Secrets Manager, rotation policy = manual at MVP).
  - WS service on Lambda has cold-start latency on first connection of a quiet period — mitigated by intro-screen pre-warm (D-IS5).
  - Item-bank table reads cross workspace boundaries (runtime → platform-shell tables). Documented in `infra/terraform/README.md` cross-account access pattern (same account, but acknowledged as a workspace coupling).
- **Out of scope (handled in follow-up proposals):**
  - **Integrity instrumentation** (face-api.js, VAD, browser signals, screen-share detection, screenshot detection, rrweb recording, biometric storage) → `add-assessment-runtime-integrity` (Phase 2)
  - **Post-session integrity analyzer** (Bedrock LLM, IntegritySummary, timeline_segments, flagged_moments) → `add-assessment-runtime-integrity` (Phase 2)
  - **Recruiter dashboard additions** (session detail page, timeline scrubber, rrweb playback viewer, raw audit-log viewer, recruiter access logging, right-to-deletion flow) → `add-recruiter-assessment-review` (Phase 3)
  - **Custom domain** (e.g., `assess.hiringiq.com`) → separate follow-up proposal `add-runtime-custom-domain` when Pluralsight delivers the FQDN, mirrors the platform shell's `add-custom-domain` pattern
  - **GitHub Actions CI/CD** for this workspace → separate follow-up `add-runtime-cicd-pipeline` once the GitHub repo for the runtime workspace is assigned
  - **Persona selfie retrieval as face-api.js enrollment baseline** (D-P4) — defers to Phase 2 because the face-api.js consumer of the enrollment doesn't exist until Phase 2

## Open Questions

- **OPEN:** Q-PSL-1 — confirm Pluralsight's Persona contract is on the Hosted/API-based Flow (default assumption). If Embedded Flow, the Phase 2 Persona-selfie retrieval cannot work; Phase 1 itself is unaffected (Persona inquiry runs identically in either flow).
- **OPEN:** Q-A1 — shared UI tokens / MUI theme distribution. NPM package vs git-subtree from platform shell. Decision needed during the workspace-scaffolding tasks below; defaulting to git-subtree at MVP (simpler than publishing a private NPM package) unless the user prefers NPM.
- **OPEN:** WS service hosting concrete pattern — direct `aws-lambda-fastify` handlers vs lighter-weight per-route handlers. Tactical pick; doesn't affect spec contracts.
