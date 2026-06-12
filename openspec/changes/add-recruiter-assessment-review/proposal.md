# Proposal — Recruiter Assessment Review

## Why

After Phase 2 (`add-assessment-runtime-integrity`) applies, every assessment session produces:
- A full audit log of integrity signals + lifecycle events
- A face-api.js enrollment baseline + Persona selfie in S3
- A pixel-faithful rrweb session recording in S3
- A post-session `IntegritySummary` with score + behavioral summary + flagged moments + timeline segments

**But none of it is visible to the recruiter yet.** The data is produced and stored; viewing requires a recruiter-facing UI that hasn't been built. This is the gap Phase 3 closes.

This proposal extends the recruiter dashboard (which lives in the platform-shell workspace at [`../PluralSight-POC/apps/web/`](../../../../PluralSight-POC/apps/web/)) with the screens, components, and access controls needed for a recruiter to review any session their org has access to:

- A list of sessions per campaign with at-a-glance integrity-score sortable columns
- A session detail page with the IntegritySummary card up top, the timeline scrubber + rrweb playback in the middle, the raw audit-log viewer below
- The right-to-deletion button that invokes Phase 2's deletion Lambda
- An honest "What we monitor" disclosure page describing the system's capabilities + limitations

This proposal also operationalizes the **recruiter access logging** requirement (D-AL7) — every view of biometric or replay data emits an audit event, building the who-viewed-what compliance trail mandated by BIPA / GDPR Article 9 / CPRA.

What this proposal does NOT deliver:

- Any new applicant-facing UX. The runtime is untouched.
- Self-serve right-to-deletion for applicants (still recruiter-mediated per D-P5).
- Cross-org dashboard views. The dashboard remains tenant-isolated per the platform-shell's existing model.
- Bulk export of session data. Defer to v2 if pilot recruiters need it.

## What Changes

This proposal extends the recruiter platform shell with the review experience. Cross-workspace edits are explicitly called out per task.

- **Add a session-list view to each campaign's overview page.** The platform-shell's campaign-overview page (proposal #9 in the shell) already shows applicants assigned to a campaign. This proposal extends those applicant rows with session-state indicator chips (Read / In-Progress / Completed / Declined / Expired / Abandoned) and — when an applicant has a COMPLETED or EXPIRED session — a clickable IntegritySummary score badge that navigates to the session detail page. Filtering by state + sorting by score are added.
- **Build the session detail page** as a new route in the platform-shell React app (`/campaigns/:campaignId/sessions/:sessionId`). The page lays out (top to bottom): assessment metadata header, IntegritySummary card (score, behavioral summary, flagged moments), timeline scrubber + rrweb playback (side-by-side or stacked depending on viewport), raw audit-log viewer, action footer (Delete biometric data button, Re-run analyzer button).
- **Build the IntegritySummary card component.** Renders the score with a color band (green ≥80, yellow 50-79, red <50), the behavioral summary prose with appropriate typography, the flagged_moments list with timestamp + description + "Jump to this moment" links (which drive the timeline + rrweb playback).
- **Build the timeline scrubber component.** Renders a horizontal timeline from session_start to session_end colored by `timeline_segments` severity (green/yellow/red bands), with `flagged_moments` rendered as pin markers on top. Each band + pin is clickable; click → seeks the rrweb player to that timestamp + scrolls the audit-log viewer to the same moment.
- **Build the rrweb playback viewer.** Embeds the MIT-licensed `rrweb-player` library. Fetches the rrweb event-stream JSON via a backend signing endpoint (the dashboard never has direct S3 access — the backend signs short-lived URLs after verifying the recruiter's org access). Standard playback controls (play / pause / scrub / playback speed).
- **Build the raw audit-log viewer.** Shows the full event stream below the IntegritySummary, default-collapsed under a "Show full audit log" expander. Filterable by category (identity / session-lifecycle / integrity-signal / assessment). Sortable by timestamp. Searchable by event_type.
- **Operationalize recruiter access logging** (D-AL7). Every backend endpoint that serves a session's data emits a corresponding `recruiter_access.viewed_*` audit event with `{ recruiter_user_id, recruiter_email, accessed_resource: { type, id, applicant_id, org_id }, timestamp_server, ip }`. Endpoints affected:
  - `GET /api/sessions/<id>` → `recruiter_access.viewed_session`
  - `GET /api/sessions/<id>/rrweb` (signing endpoint) → `recruiter_access.viewed_replay`
  - `GET /api/sessions/<id>/audit-events` (paginated) → `recruiter_access.viewed_audit_log`
  - `GET /api/applicants/<id>/persona-selfie` (signing endpoint) → `recruiter_access.viewed_persona_selfie`
- **Wire the right-to-deletion button.** A "Delete biometric data" button on the session detail page (or applicant detail — TBD per task below) opens a confirmation modal, then on confirm calls `POST /api/applicants/<id>/biometric-deletion` (the runtime workspace's endpoint that invokes the Phase 2 `biometric-deletion` Lambda). On success, the dashboard reflects the deletion (selfie thumbnail removed, embedding indicator goes away, "Recording deleted" placeholder where the rrweb viewer was).
- **Build the "What we monitor" disclosure page.** A new info page accessible from a help link in the dashboard footer. Lists the integrity signals we capture (silent + persistent + escalating), the retention policy, the deletion process, and — importantly — the limitations we CANNOT detect (hardware screen capture, phone-photo-with-voice-AI, etc.). Sets honest expectations for the recruiter so they don't over-trust a "clean" IntegritySummary.
- **Add an analyzer re-run button.** On the session detail page, a small "Re-analyze with latest prompt" button enqueues an SQS message to the analyzer Lambda with the same session_id. Useful for sessions analyzed before a prompt upgrade. Each run produces a new IntegritySummary version row (Phase 2's versioning model); the dashboard defaults to displaying the latest but lets recruiters compare across versions.

## Capabilities

### New Capabilities (in the runtime workspace)

- `recruiter-access-logging`: the backend endpoints + Lambda handlers that emit `recruiter_access.viewed_*` audit events on every recruiter view of biometric or replay data (D-AL7).
- `biometric-deletion-trigger`: the runtime-side HTTP endpoint that exposes the Phase 2 `biometric-deletion` Lambda to the recruiter dashboard (with cognito-JWT + org-isolation validation).

### Cross-workspace capabilities (in the platform-shell workspace)

These are added by this proposal but their code lives in `/Users/jgrenard/workspace/projects/PluralSight-POC/apps/web/`:

- `session-list-view`: campaign-overview extension with session-state chips + score badges + sort/filter.
- `session-detail-view`: new dashboard route + layout for reviewing a single session.
- `integrity-summary-card`: score + summary + flagged moments component.
- `timeline-scrubber`: colored-band timeline with pins + click-to-jump.
- `rrweb-playback-viewer`: embedded rrweb-player component + signing-endpoint integration.
- `raw-audit-log-viewer`: full event-stream viewer below IntegritySummary.
- `what-we-monitor-disclosure`: honest disclosure page accessible from dashboard footer.

### Modified Capabilities

- `campaign-overview` (platform-shell proposal #9): the applicant table per campaign gains session-state chips + score badges per applicant-session pair.

## Impact

- **Code:**
  - Runtime workspace:
    - `services/assessment/src/routes/recruiter-views.ts` — endpoints for the dashboard to fetch session + IntegritySummary + audit-events
    - `services/applicant/src/routes/biometric-deletion-trigger.ts` — invokes Phase 2's deletion Lambda
    - `services/telemetry/src/recruiter-access-emitter.ts` — emits `recruiter_access.*` events from the assessment + applicant services
  - Platform-shell workspace (cross-workspace edits):
    - `apps/web/src/pages/SessionDetail.tsx` — new route
    - `apps/web/src/components/integrity/IntegritySummaryCard.tsx`
    - `apps/web/src/components/integrity/TimelineScrubber.tsx`
    - `apps/web/src/components/integrity/RrwebPlaybackViewer.tsx`
    - `apps/web/src/components/integrity/RawAuditLogViewer.tsx`
    - `apps/web/src/pages/WhatWeMonitor.tsx` — disclosure page
    - `apps/web/src/pages/CampaignOverview.tsx` — extension with session-state chips
- **AWS resources added:**
  - 1 new IAM role for the recruiter-dashboard signing Lambdas (Cognito-JWT validated, scoped to org_id from the JWT)
  - Updated IAM on assessment + applicant services for the new endpoints
- **Cost (pilot scale, Phase 3 on top of Phases 1+2):**
  - Additional Lambda invocations from the recruiter dashboard (~1 per session view × ~100 sessions/mo × ~3 views per session = ~300/mo): ~$0.05/mo
  - rrweb playback bandwidth: ~10-50 MB per playback × ~30 playbacks/mo = ~1.5GB/mo CloudFront egress: ~$0.15/mo
  - Total incremental Phase 3 cost: under $1/mo at pilot scale
- **Dependencies introduced:**
  - `rrweb-player` (MIT) bundled in the platform-shell's React app
  - Updates to the platform-shell's React app dependency tree (MUI components for the timeline scrubber; possibly `d3-scale` or similar for timeline geometry)
- **Cross-workspace coupling:** This proposal LIVES in the runtime workspace's OpenSpec changes directory, but it modifies code in the platform-shell workspace. The tasks.md explicitly lists which files in `/Users/jgrenard/workspace/projects/PluralSight-POC/apps/web/` are touched.
- **Downstream unblocks:** This is the final phase. After Phase 3 applies, the assessment-runtime workstream is feature-complete for pilot. Follow-up proposals (custom domain, CI/CD pipeline for the runtime workspace, self-serve right-to-deletion UI) are smaller incremental changes.
- **Operational risk introduced:**
  - Recruiter dashboard now displays biometric data (selfie thumbnails on session detail). Strict IAM scoping + access logging + 15-min signed URLs mitigate the surface. **Note:** the dashboard displays selfie thumbnails ONLY on the session detail page, not on the campaign overview — to reduce incidental exposure.
  - rrweb playback CAN contain sensitive content (the applicant's typed responses are in the recording). The signing endpoint validates org-level access; recruiters in OrgA cannot fetch OrgB's recordings.
  - **The access-log volume is meaningful.** A diligent recruiter might trigger ~10-50 recruiter_access events per session-detail-page visit (session view + summary view + log view + replay view + multiple panel switches). The audit-events table needs to scale. At pilot scale this is fine; at enterprise scale it warrants its own follow-up consideration.
- **Out of scope (handled in future proposals or deferred):**
  - **Self-serve right-to-deletion UI for applicants** — the applicant emails the recruiter, who triggers via the dashboard. Defer to v2.
  - **Bulk session export** (CSV download of audit events, etc.) — defer to v2.
  - **Comparative views across applicants on a campaign** (side-by-side timeline scrubbers, etc.) — defer to v2.
  - **Notification when a session completes** (email or in-app to the recruiter) — defer to v2; recruiters check the dashboard.
  - **Mobile-friendly recruiter dashboard** — defer to v2; recruiters use desktop per the platform-shell's existing posture.

## Open Questions

- **OPEN:** rrweb signing endpoint TTL. Lean: 15 minutes (long enough for a recruiter to review; short enough that a leaked URL has bounded impact). Tactical — pick during implementation.
- **OPEN:** Where the right-to-deletion button lives — session detail page (per-session deletion) vs applicant detail page (per-applicant whole-history deletion). Lean: **applicant detail** because deletion is per-applicant per D-P5 (deletes all artifacts for that applicant across all their sessions). The session detail page shows an inline "this applicant's biometric data has been deleted" indicator when applicable.
- **OPEN:** Whether the "Re-analyze" button is recruiter-accessible at MVP or admin-only. Lean: recruiter-accessible — it costs $0.05-$0.10 per re-run; trusting the recruiter at pilot scale is acceptable. Add usage-rate-limiting later if needed.
- **OPEN:** Whether the dashboard surfaces partial IntegritySummary failures (`status: "analyzer_failed"` from Phase 2 D-AL4 retry exhaustion) to the recruiter, or hides them. Lean: surface with a "Analysis unavailable" placeholder + "Request manual re-analysis" button — honest beats hidden.
