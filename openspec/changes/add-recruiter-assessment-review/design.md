# Design — Recruiter Assessment Review

## Context

Phases 1 and 2 produce the assessment runtime + integrity instrumentation + post-session IntegritySummary. None of it is visible to the recruiter yet. This proposal closes that gap with a recruiter dashboard extension.

Cross-workspace nature: this proposal LIVES in the runtime workspace's OpenSpec changes directory, but the dashboard React app is in the platform-shell workspace at [`/Users/jgrenard/workspace/projects/PluralSight-POC/apps/web/`](../../../../PluralSight-POC/apps/web/). The proposal's tasks explicitly reference files in BOTH workspaces.

Stakeholders: KMS owns implementation. The Pluralsight Technical Lead reviews the recruiter UX (dashboards are PS-facing). Legal reviews the "What we monitor" disclosure for accuracy.

## Goals / Non-Goals

**Goals:**
- A recruiter can view any session's IntegritySummary, replay, and raw audit log from the dashboard
- The timeline scrubber drives navigation across summary + replay + audit-log views
- Every recruiter view of biometric/replay data emits a `recruiter_access.*` event per D-AL7
- Right-to-deletion is a one-click recruiter action with a confirmation modal
- The "What we monitor" page sets honest expectations (capabilities + limitations)

**Non-Goals:**
- New applicant-facing UX
- Self-serve right-to-deletion for applicants
- Cross-org review (the dashboard remains tenant-isolated)
- Bulk export
- Comparative views across applicants
- Mobile-responsive dashboard

## Decisions

### D-R1. Dashboard code lives in the platform-shell workspace, NOT the runtime workspace

**Choice:** New React components, new route (`/campaigns/:campaignId/sessions/:sessionId`), and modifications to existing pages (campaign-overview) all land in `/Users/jgrenard/workspace/projects/PluralSight-POC/apps/web/`. The runtime workspace contributes only the BACKEND endpoints (signing URLs, deletion-trigger, access-event emission).

**Why:** The recruiter dashboard is the platform-shell's UI. Splitting the dashboard React code across two workspaces would create coupling that's harder to maintain than the current "runtime owns its data, platform-shell owns its dashboard" boundary.

**Implication for OpenSpec ownership:** This proposal modifies platform-shell capabilities (`campaign-overview`) but lives in the runtime workspace's `changes/` directory. When applied, the platform-shell's archived `campaign-overview` spec gets MODIFIED requirements added via the cross-workspace pattern described in tasks.md.

**Alternative considered:** Move the dashboard code into the runtime workspace. Rejected — fragments the recruiter UI across two repos.

### D-R2. Backend endpoints for dashboard access live in the runtime workspace

**Choice:** New HTTPS endpoints in the runtime workspace's `assessment-service` and `applicant-service`:
- `GET /api/sessions/<id>` — session record + IntegritySummary
- `GET /api/sessions/<id>/rrweb` — returns a signed S3 URL (15-min TTL) for the rrweb event stream
- `GET /api/sessions/<id>/audit-events` — paginated audit-log events
- `GET /api/applicants/<id>/persona-selfie` — returns a signed S3 URL for the selfie
- `POST /api/applicants/<id>/biometric-deletion` — invokes the Phase 2 deletion Lambda
- `POST /api/sessions/<id>/re-analyze` — enqueues a fresh analyzer SQS message

Each endpoint validates the Cognito JWT, checks the recruiter's `org_id` claim matches the session's `org_id`, and emits the corresponding `recruiter_access.*` event before returning data.

**Why:** Data ownership is in the runtime workspace; access controls + audit emission are easiest to enforce at the data boundary. The dashboard becomes a thin presentation layer.

### D-R3. rrweb signing endpoint uses 15-minute TTL signed URLs

**Choice:** The dashboard requests `GET /api/sessions/<id>/rrweb`; the runtime endpoint validates the JWT, emits the audit event, then returns a 15-minute pre-signed S3 GET URL. The dashboard's React component loads the rrweb event stream from S3 directly via this URL.

**Why:** Pre-signed URLs let the dashboard download large rrweb event streams without proxying them through Lambda (which would be slow and expensive). 15-min TTL is long enough for a recruiter to review (typical session-detail visit ≤10 min) and short enough that a leaked URL has bounded impact.

**Alternative considered:** Proxy through Lambda. Rejected — rrweb event streams can be 10-50 MB; Lambda is poorly suited to streaming this size.

### D-R4. Right-to-deletion is per-applicant, not per-session (D-P5)

**Choice:** The "Delete biometric data" button lives on the applicant detail page (NOT the session detail page) and deletes ALL biometric artifacts for that applicant across all their sessions — Persona selfie, face embedding, all rrweb recordings.

The session detail page shows an inline "Biometric data for this applicant has been deleted" indicator when applicable; the replay viewer + selfie thumbnail are replaced with a "Deleted" placeholder.

**Why:** Per D-P5, deletion is a per-applicant compliance event (the applicant's right-to-deletion request applies to all their data, not a single session). Triggering from the applicant detail page makes the scope clear in the UX.

**Alternative considered:** Per-session delete buttons. Rejected — would confuse recruiters about what's actually deleted; would leave embedding/selfie behind if any session's data persisted.

### D-R5. Recruiter access events are emitted from the BACKEND, not the frontend

**Choice:** The `recruiter_access.*` events emit from the runtime's backend endpoints (D-R2), not from React component-level `useEffect` calls in the dashboard.

**Why:** Frontend emission can be bypassed (devtools, modified client). Backend emission on the data-access endpoint is the source of truth. Even if a malicious recruiter manipulates the dashboard, they cannot view data without hitting the backend, which logs the access.

**Implementation:** Each endpoint handler calls a `recruiterAccessEmitter` helper as its first action, before returning data. The helper writes to the same `hiringiq-pilot-audit-events` table the integrity events use, tagged with category `recruiter-access` + the documented event_type.

### D-R6. The IntegritySummary card displays the score with a three-band color treatment

**Choice:** Score 80-100 → green band, 50-79 → yellow band, <50 → red band. The card prominently displays the score + band color in the upper-left, with the behavioral summary prose below.

The score thresholds are SSM-configurable at `/hiringiq/pilot/runtime/dashboard/score-bands/*` (default 80 / 50). Adjustable post-pilot based on recruiter feedback.

**Why:** Quick visual triage — recruiters scanning a list of sessions need an at-a-glance "is this one worth my attention?" signal. The exact score is also displayed numerically; the band color is just an aid.

### D-R7. Timeline scrubber + rrweb + audit-log are linked navigation surfaces

**Choice:** Clicking anywhere on the timeline scrubber (a band, a pin, an empty point) seeks the rrweb player AND scrolls the audit-log viewer to the corresponding timestamp. Clicking a "Jump to this moment" link in the flagged_moments list does the same.

The synchronization happens via a shared `useState` in a parent component holding the "current playback timestamp"; each child (scrubber, player, log viewer) reads from + writes to this state.

**Why:** A unified navigation model means the recruiter never has to manually correlate timestamps across views. Click a flagged moment → see what happened on the timeline + see the replay + see the events around it.

### D-R8. "What we monitor" disclosure is mandatory honesty, not marketing

**Choice:** The disclosure page lists EVERY integrity signal we capture AND every limitation. Specifically:
- Capabilities: face presence, identity matching, attention tracking, audio voice activity, browser visibility/focus, paste/copy, picture-in-picture, media-device fingerprinting, multi-monitor detection (limited), Windows screenshot detection, screen-share software fingerprinting (limited), rrweb session replay
- Limitations: cannot detect external screen capture (HDMI splitter, OBS without virtual camera), cannot detect phone-photo-with-voice-AI, macOS screenshots invisible, hardware capture cards invisible

The disclosure is accessible from a footer link in the recruiter dashboard. Pilot recruiters MUST understand these limits to set candidate-fairness expectations.

**Why:** Over-promising integrity detection is a legal hazard (the recruiter could make hiring decisions based on a "clean" report that missed a sophisticated attack). Honest disclosure protects KMS, Pluralsight, and the recruiter.

### D-R9. Re-analyze action is recruiter-accessible but rate-limited

**Choice:** The "Re-analyze with latest prompt" button is visible to all recruiters in the org. Clicking it enqueues an SQS message; the new IntegritySummary lands as a new version row (Phase 2's versioning model).

Rate limit: each session can be re-analyzed at most 3 times per 24-hour window (SSM-configurable). Prevents accidental loop / cost overrun.

**Why:** Trusting recruiters at pilot scale is fine; rate-limiting is a backstop. Cost per re-run is $0.05-$0.10 — even with abuse, the cost ceiling is bounded.

### D-R10. Dashboard does NOT cache biometric content

**Choice:** Selfie images + rrweb recordings are NEVER cached in the dashboard's localStorage / sessionStorage / IndexedDB. They are fetched fresh via signed URLs each time the recruiter navigates to the session detail page.

**Why:** Biometric data caching in the browser would create a secondary distribution surface — a recruiter's laptop with a compromised browser would have biometric data accessible offline. Fresh-fetch policy + 15-min URL TTL means the browser cache is a non-target.

**Implication:** Loading the session detail page is slightly slower (no cache hits), but that's acceptable for compliance posture.

## Risks / Trade-offs

- **rrweb playback bandwidth could be expensive at scale.** ~10-50 MB per recording × N recruiter views. At pilot scale negligible; at enterprise scale warrants its own follow-up (e.g., bandwidth caching at the CloudFront layer with short TTL).
- **Recruiter-access audit-log volume could grow large.** A diligent reviewer might generate 50+ events per session viewed. At 100 sessions × 10 view sessions × 50 events = 50K events/mo. Within DDB on-demand throughput limits; still worth monitoring.
- **Cross-workspace edits introduce review complexity.** Engineers reviewing this proposal need to check BOTH workspaces. The tasks.md makes this explicit per-task.
- **The dashboard's playback viewer could leak data if signed URLs are forwarded.** Mitigation: 15-min TTL + browser-cache disabled (D-R10) + access-logged so anyone who forwarded a URL is identifiable from the log.

## Migration Plan

This proposal applies after Phases 1 + 2 are live. No data migration — only new endpoints + new UI.

1. **Apply Terraform** in the runtime workspace — IAM updates + new endpoint integrations.
2. **Deploy runtime Lambda code** — new dashboard endpoints + recruiter-access emitter helper.
3. **Apply cross-workspace edits** to the platform-shell repo. This is a normal feature branch + PR in the platform-shell repo, NOT a separate Terraform stack. The platform-shell's React app rebuilds + redeploys via its existing CloudFront pipeline.
4. **Smoke test:** log in as a test recruiter, navigate to a completed test session's detail page, verify the IntegritySummary card renders, the timeline scrubber draws correctly, the rrweb player loads, the audit-log viewer paginates. Verify the access-log events appear in the audit-events table.
5. **Legal review** of the "What we monitor" disclosure content before the page goes live.

**Rollback strategy:**
- Bad dashboard code: revert the platform-shell PR and redeploy the React app.
- Bad backend endpoint: roll back the runtime Lambda zip + re-deploy.
- Bad access-logging change: minor — events stop firing; rectify and redeploy.

## Open Questions

- **OPEN:** rrweb signing endpoint TTL (15 min lean; tactical).
- **OPEN:** Re-analyze rate limit value (3 per 24h lean).
- **OPEN:** Whether to display partial IntegritySummary failures (`status: "analyzer_failed"`) prominently or hide them — leans toward surfacing with a "Request manual re-analysis" CTA.
- **OPEN:** Score thresholds for the green/yellow/red bands (default 80/50; SSM-tuneable).
