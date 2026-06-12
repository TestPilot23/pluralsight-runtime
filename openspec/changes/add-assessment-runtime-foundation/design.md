# Design — Assessment Runtime Foundation

## Context

The applicant-facing assessment runtime was designed end-to-end in a long architectural conversation captured in [`../../../../PluralSight-POC/thoughts/assessment-design-decisions.md`](../../../../PluralSight-POC/thoughts/assessment-design-decisions.md). That document has 60+ locked decisions (A through M sections) covering the full system: Persona, audit log + analyzer, session state machine, UI layout, intro/landing/device-check/completion screens, rrweb capture, recruiter access, and so on.

Rather than restate every decision here, this design.md focuses on the **architectural calls specific to Phase 1**: what's in, what's out, why we phased it this way, and the integration points between this phase's deliverables and the platform-shell workspace.

Stakeholders: KMS owns implementation. The Pluralsight Technical Lead reviews architecture (separate workspace, Lambda-only deployment, multi-service decomposition).

## Goals / Non-Goals

**Goals (this proposal):**
- An applicant can complete an assessment end-to-end through a polished UI without any integrity instrumentation visible to them.
- The four-service architecture (D-SVC1) is deployed and operational, with all four services as Lambda functions (D-SVC2).
- The session state machine is server-authoritative and correctly handles all the pre-assessment branches (DECLINED, EXPIRED), the assessment flow (IN_PROGRESS, PAUSED.ws_disconnect), and the terminal states (COMPLETED, ABANDONED).
- The audit-log table is populated with session-lifecycle events from this phase onwards. Phase 2 layers integrity-signal events on top of the same table.
- The runtime workspace's Terraform stack is independent of the platform shell's — they reuse modules where possible (cross-workspace module references via Terraform's module sources) but each has its own state file.

**Non-Goals (deferred to Phase 2):**
- Any face-api.js, audio VAD, browser-instrumentation, screen-share-detection, or screenshot-detection signal capture.
- rrweb session recording.
- Biometric storage (Persona selfie retrieval, face-api.js enrollment, embeddings) — see D-P4 deferral note below.
- Post-session Bedrock analyzer.
- The auto-expand-on-warning behavior on the webcam card (no warning system yet).
- Pause states for face_not_detected, webcam_stream_unavailable, etc.

**Non-Goals (deferred to Phase 3):**
- Recruiter dashboard additions (session list, session detail, timeline scrubber, rrweb playback, raw audit-log viewer, recruiter access logging events, right-to-deletion flow).

## Decisions

### D-F1. Phase 1 ships the runtime workspace + foundation; integrity layers in Phase 2

**Choice:** Three-phase split with strict dependency order — Phase 1 (foundation) → Phase 2 (integrity) → Phase 3 (recruiter review).

**Why:** Authoring everything as a single OpenSpec proposal would produce a 250+ task megachange. Splitting at the natural architectural seams — what an applicant SEES vs what's CAPTURED for analysis vs what the RECRUITER sees — gives three reviewable proposals each delivering vertical value.

Phase 1 alone proves the assessment can be completed end-to-end, which de-risks the rest. Phase 2 layers in observability without re-touching the user-facing flow. Phase 3 is purely additive — extends the existing recruiter dashboard.

**Alternative considered:** A single mega-proposal. Rejected — exceeds reviewable scope; the failure mode is reviewers skim sections and miss issues.

### D-F2. Persona selfie retrieval (D-P4) defers to Phase 2

**Choice:** Persona inquiry runs at Phase 1 (D-P1, D-P2, D-P3 all apply), but we do NOT fetch the selfie image from Persona's Inquiry API. The face-api.js enrollment baseline doesn't exist as a consumer yet.

**Why:** At Phase 1, there's no in-assessment presence verification (no face-api.js running every 8 minutes), so the enrollment photo has nothing to enroll against. Fetching and storing a selfie at this phase would mean storing biometric data that nothing reads. Better to defer storage until the consumer exists.

**Implication:** The biometric retention policy (D-P5, 180 days) doesn't apply at Phase 1 because we don't store biometric data yet. It activates in Phase 2 when Persona selfie retrieval lands.

**Impact on Phase 1 Persona flow:** The applicant still goes through Persona's Inquiry SDK. We receive the webhook with the verification result (status + confidence) per D-P1. We do NOT call Persona's Inquiry GET API to retrieve assets. Storage on our side: only `{ persona_inquiry_id, status, confidence, attempts_count, last_reason_code }` on the applicant record.

### D-F3. WebSocket service exists at Phase 1 but only handles session-lifecycle events

**Choice:** `telemetry-service` is deployed as a WebSocket-on-Lambda service from Phase 1 (D-SVC2 WS pattern). It accepts client connections, handles `$connect` / `$disconnect` / `$default` routes, writes session-lifecycle events to the audit-log table. Integrity-signal events (face.*, tab.*, paste.*, etc.) are NOT handled in Phase 1 — the client doesn't emit them at this phase because the instrumentation isn't installed.

**Why:** Having the WS infrastructure in place lets us validate the WS connection lifecycle (open on intro screen per D-IS5, pause on disconnect, resume on reconnect) without the operational complexity of high-volume signal capture. Phase 2 turns on the signal capture by enabling the client-side instrumentation; the WS endpoint just accepts more event types.

**Phase 1 events emitted by the client:**
- `session.landing_page_viewed`
- `permissions.granted` / `permissions.denied`
- `terms.accepted`
- `terms.opened` / `privacy_notice.opened`
- `session.landing_continue_clicked`
- `persona.*` events (attempted, succeeded, declined.<reason>, api_error, bypassed)
- `device_check.viewed`
- `device_check.webcam_preview_viewed`
- `device_check.mic_test_passed` / `device_check.mic_test_skipped`
- `device_check.speaker_test_played` / `device_check.speaker_test_self_attest`
- `device_check.continue_clicked`
- `session.intro_screen_viewed`
- `consent.checkbox_checked` / `consent.checkbox_unchecked`
- `applicant.declined.initiated` / `applicant.declined.cancelled` / `applicant.declined.confirmed`
- `session.started` (Begin Assessment click)
- `item.displayed` / `item.submitted` / `item.time_expired` / `item.null_submission`
- `session.paused.ws_disconnect` (server-driven on disconnect)
- `session.resumed` (server-driven on reconnect)
- `session.completed`

That's the **non-integrity** subset of D-AL2's taxonomy. Phase 2 adds the integrity-signal events on top.

### D-F4. Audit-log table is shared with Phase 2

**Choice:** Provision the `hiringiq-pilot-audit-events` table at Phase 1. Schema matches D-AL3 exactly (PK = `SESSION#<session_id>`, SK = `<timestamp_server>#<event_id>`). Phase 1 writes only the non-integrity events; Phase 2 writes integrity-signal events into the same table.

**Why:** One table, one schema, one access pattern. Phase 2 doesn't migrate data — it just adds new event types into the existing flow.

**Storage decision:** S3 immutable archive (D-AL3) is also provisioned at Phase 1 — every event written to DDB is also written to S3. This means the tamper-evident archive exists from day 1, before integrity signals start flowing. Cheap to provision; expensive to retro-fit if we skipped it.

### D-F5. Item-bank table reads are cross-workspace

**Choice:** The runtime workspace reads from the platform shell's `hiringiq-pilot-items` and `hiringiq-pilot-item-skill-index` tables (created by platform-shell proposal #7, `add-assessment-item-bank`). The runtime workspace does NOT replicate these tables.

**Why:** Items are global content, not tenant-scoped. Replicating would create a sync problem and double the source-of-truth surface. Cross-workspace DDB reads are trivially supported (same AWS account, same region, IAM grant on the runtime Lambdas' execution roles).

**Implication for Terraform:** Runtime workspace's Lambda IAM roles must reference the platform-shell's item-bank table ARNs. We achieve this by reading the table ARNs from Terraform `data` sources (looking up tables by name) rather than coupling via remote state. The platform shell's `add-assessment-item-bank` proposal must be applied BEFORE this proposal can run successfully — added to the apply-order checklist.

### D-F6. Magic-link JWT contract is shared with the platform-shell (Path Z hybrid)

**Choice:** The runtime workspace **validates** JWTs that the platform-shell's `add-applicant-assessment-landing` (proposal #11) mints. The two workspaces share the EXISTING platform-shell secret `hiringiq/<env>/applicant-magic-link-secret` (HS256-signed, 64-byte hex key). The runtime's `assessment-service` and `applicant-service` Lambda IAM roles are granted `secretsmanager:GetSecretValue` on the platform-shell's secret ARN (cross-stack IAM grant).

**JWT contract (matches platform-shell #11 exactly):**

| Claim | Type | Value |
|---|---|---|
| `aid` | string | applicant UUID v7 |
| `cid` | string | campaign UUID v7 |
| `oid` | string | org UUID v7 |
| `iat` | number | Unix seconds (issuance) |
| `exp` | number | `min(iat + 15 × 86400, campaign.end_at)` — TTL is the EARLIER of 15-days-from-issuance (per D-MAG1) OR the campaign's natural end date |
| `iss` | string | `"hiringiq-pilot"` |
| `aud` | string | `"applicant-assessment"` |

**TTL reconciliation:** Platform-shell #11 as originally authored used `exp = campaign.end_at` only. D-MAG1 (locked in design conversation) said 15-day TTL. Path Z reconciles via the `min()` rule — applicants get a 15-day window relative to invitation, capped at the campaign's natural end. This requires a small modification to platform-shell #11's spec (changing the `exp` formula); flagged as a cross-workspace edit in Phase 1's tasks.

**`session_id` is NOT a JWT claim.** Per platform-shell #11, the session is identified by the `{oid, cid, aid}` tuple — the runtime's `assessment-service` looks up (or lazily creates) the session record on the first request after JWT validation. The runtime's session record's PK is `session_id` (a server-generated UUID v7), but that ID is NOT propagated through the JWT — the client never needs to know it.

**Helpers:** Reuse `generateMagicLink({ applicantId, campaignId, orgId, exp, secret })` and `verifyMagicLink(token, secret)` from the platform-shell's `packages/auth/src/magic-link.ts`. These already implement the correct HS256 + claims contract. The runtime workspace either imports them via a shared NPM package OR git-subtrees the `packages/auth/` directory.

**Why this design:** Owning the JWT signing in the runtime was premature optimization (rejected). Path Z keeps platform-shell as the issuer (it already implements this correctly), the runtime as the validator (its natural role), shares one secret (no credential proliferation), respects the locked 15-day TTL via the min() formula (preserves the design conversation's intent), and reuses already-authored helpers (no duplication).

**Cross-workspace edit required:** platform-shell #11's spec gains a MODIFIED Requirement (added by this proposal's tasks) changing the `exp` formula from `campaign.end_at` to `min(iat + 15 × 86400, campaign.end_at)`.

### D-F7. Modular code structure with strict internal API boundaries

**Choice:** Each of the four services is its own code module under `services/`. Each exposes a `public.ts` (or `public/index.ts`) file that defines the module's public API. ESLint rule blocks cross-module imports that bypass `public.ts`.

**Why:** D-SVC2's "promote to Fargate later" path requires modules with clean API surfaces. The discipline now means a future split is mechanical, not architectural.

**ESLint rule sketch:** `no-restricted-imports` with patterns like `services/applicant/!(public)*` blocked for non-applicant code. Configured in `eslint.config.js` at the monorepo root.

### D-F8. analyzer-service Lambda is provisioned but no-op at Phase 1

**Choice:** Provision the `analyzer-service` Lambda function + its SQS trigger queue at Phase 1 via Terraform. The Lambda handler is a stub that logs the invocation and returns. The SQS queue receives no messages at Phase 1 because nothing in the session-end transitions emits to it yet.

**Why:** Provisioning the empty service now means Phase 2 only needs to swap the Lambda's code, not stand up new infrastructure. Cleaner Phase 2 PR. Cost is negligible — a Lambda with no invocations costs $0.

**Alternative considered:** Skip provisioning until Phase 2. Rejected — Phase 2's PR scope creeps wider than needed.

### D-F9. Item display order is randomized per applicant

**Choice (already locked in D-SM9, restating for clarity):** The campaign's `assessment_questions` array (locked at campaign create per platform-shell's item-bank D17) is presented to each applicant in a shuffled order. Shuffle seed is derived deterministically from `session_id`.

**Why:** Same items for apples-to-apples comparison across applicants on the same campaign; different order to defeat the "I memorized the position of each question" attack.

**Implementation:** Server-side Fisher-Yates shuffle with the seed derived from `crypto.createHash('sha256').update(session_id).digest()`. Reproducible — given the same `session_id`, two independent runs of the shuffle produce identical order. Useful for debugging + analyzer reproducibility.

## Risks / Trade-offs

- **Phase 1 ships without integrity capture.** A pilot recruiter could in theory adjudicate an assessment without any of the anti-cheat signals the brief promised. **Mitigation:** This proposal explicitly NOT pilot-ready on its own. The apply-checklist documents that Phase 2 must apply before any real applicant takes the assessment.
- **WS-on-Lambda cold-start latency on `$default` route.** First message after a quiet period (~5+ min idle) takes 200-2000ms longer. **Mitigation:** WS opens at intro-screen load (D-IS5), so by the time the applicant clicks Begin Assessment, the Lambda is already warm.
- **Cross-workspace coupling for item-bank reads.** The runtime workspace is fully functional only when the platform shell's item-bank proposal has applied. **Mitigation:** Documented apply-order dependency. Runtime Terraform `data` sources fail loudly if the items table doesn't exist, surfacing the missing dependency clearly at apply time.
- **Magic-link JWT signing secret is operationally critical.** Loss of the secret invalidates every outstanding magic link. **Mitigation:** Standard Secrets Manager rotation policy at v2; manual rotation runbook documented in `infra/terraform/README.md`.
- **The DECLINED + EXPIRED + ABANDONED state proliferation.** Three terminal states for "didn't complete" plus one recoverable state (ABANDONED). Reviewers may confuse them. **Mitigation:** State diagram in `services/assessment/README.md` plus unit tests covering every transition.

## Migration Plan

This is a greenfield proposal in a new workspace. Nothing to migrate from. Apply steps:

1. **Pre-flight:** confirm platform-shell proposal #7 (`add-assessment-item-bank`) has applied. Confirm `aws --profile kms-pluralsight dynamodb describe-table --table-name hiringiq-pilot-items` returns a non-empty result.
2. **Workspace scaffold:** create `apps/runtime-web`, `services/{applicant,assessment,telemetry,analyzer}`, `infra/terraform/`, `packages/shared`. ESLint config with cross-module import restrictions.
3. **Bootstrap Terraform state for the runtime workspace.** Reuse the platform-shell's state bucket + lock table — DIFFERENT state file (`pluralsight-poc-runtime-pilot.tfstate`). Same `kms-pluralsight` profile.
4. **Apply runtime Terraform** — provisions all the AWS resources listed in proposal.md's Impact.
5. **Deploy Lambda code** — initial deploy via `terraform apply` (Lambda functions reference S3 deployment artifacts uploaded by the CI/CD pipeline once added; at MVP via `aws lambda update-function-code` from a developer laptop).
6. **Smoke test:** create a test campaign in the platform shell pointing to seeded items, generate a magic link, click through end-to-end (consent → Persona → device check → intro → 2-3 items → complete). Verify session record in `hiringiq-pilot-sessions` reaches COMPLETED state, audit-log table contains the expected events.

**Rollback strategy:**
- Bad Terraform change: revert the offending commit and re-apply.
- Bad Lambda deploy: redeploy previous zip; functions support versioning.
- The runtime workspace can be wholesale destroyed without impacting the platform shell (separate state file, separate CloudFront distribution, separate DDB tables — but the platform shell's item-bank tables stay because they're owned by the platform shell).

## Open Questions

- **OPEN:** Q-PSL-1 — Pluralsight's Persona contract flow (Hosted/API-based vs Embedded). Phase 1 functions identically in either flow; affects Phase 2's selfie-retrieval feasibility only.
- **OPEN:** Q-A1 — shared UI tokens packaging (NPM vs git-subtree). Decision needed during the workspace scaffolding tasks below.
- **OPEN:** WS service handler library choice — `aws-lambda-fastify` vs lighter weight. Tactical implementation pick; doesn't affect spec contracts. Recommend deferring to the engineer authoring the telemetry-service module.
- **OPEN:** Whether the runtime should reuse the platform-shell's CloudFront alarm distribution SNS topic OR provision its own. Lean: provision its own — `hiringiq-pilot-runtime-alerts` — so platform-shell and runtime alarms route separately. Cleaner separation; minor extra cost (~$0).
