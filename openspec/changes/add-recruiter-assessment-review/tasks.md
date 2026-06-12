# Tasks — Recruiter Assessment Review

Sequenced by dependency. Phases 1–2 add the runtime workspace's backend endpoints + IAM. Phases 3–8 build the platform-shell dashboard UI. Phase 9 wires the deletion + re-analyze flows. Phase 10 verifies end-to-end.

## 1. Runtime workspace backend endpoints

- [ ] 1.1 Add `services/assessment/src/routes/recruiter-views.ts` with `GET /api/sessions/<id>`. Returns the session record + IntegritySummary + applicant + campaign metadata. Validates Cognito JWT + checks `org_id` claim matches the session's org_id. **Verify:** unit test confirms (a) authenticated request from matching org returns data, (b) cross-org request returns 403, (c) call emits `recruiter_access.viewed_session`.
- [ ] 1.2 Add `GET /api/sessions/<id>/rrweb` to `recruiter-views.ts`. Returns a signed S3 GET URL for the rrweb event stream with 15-min TTL. Validates Cognito JWT + org_id match before signing. Emits `recruiter_access.viewed_replay`. **Verify:** integration test confirms the signed URL fetches the recording successfully within TTL; expires after.
- [ ] 1.3 Add `GET /api/sessions/<id>/audit-events` to `recruiter-views.ts` — paginated audit-log events filterable by category. Default page size 100. Emits `recruiter_access.viewed_audit_log` once per page-1 request (not per page to avoid log spam). **Verify:** unit test confirms pagination + filter behavior.
- [ ] 1.4 Add `GET /api/applicants/<id>/persona-selfie` to `services/applicant/src/routes/recruiter-views.ts`. Returns a signed S3 GET URL for the Persona selfie image with 15-min TTL. Validates Cognito JWT + org_id match. Emits `recruiter_access.viewed_persona_selfie`. **Verify:** integration test confirms the signed URL fetches the selfie.
- [ ] 1.5 Add `POST /api/applicants/<id>/biometric-deletion` to `services/applicant/src/routes/`. Validates the recruiter's Cognito JWT + applicant org match. Invokes the Phase 2 `biometric-deletion` Lambda synchronously via `lambda:InvokeFunction`. Emits `biometric.deletion_requested` audit event. **Verify:** integration test confirms the deletion Lambda is invoked + returns success; all artifacts (selfie + embedding + rrweb) deleted.
- [ ] 1.6 Add `POST /api/sessions/<id>/re-analyze` to `services/assessment/src/routes/`. Validates Cognito JWT + org_id. Checks the rate-limit (3 re-runs per 24h per session, SSM-configurable). Enqueues a new SQS message to `hiringiq-pilot-analyzer-queue` with the session_id. Emits `analyzer.re_run_requested` audit event with `{ recruiter_user_id, session_id }`. **Verify:** integration test confirms a new analyzer SQS message; rate-limit is enforced.
- [ ] 1.7 Implement `services/telemetry/src/recruiter-access-emitter.ts` — shared helper invoked by all dashboard endpoints BEFORE returning data. Writes `recruiter_access.*` events to the audit-events table (DDB + S3 archive) with the documented envelope per D-AL7. **Verify:** unit test confirms each event type's payload shape; integration test confirms write to both stores.
- [ ] 1.8 Update IAM for the assessment + applicant services: add `secretsmanager:GetSecretValue` on the Cognito JWKS, `lambda:InvokeFunction` on the biometric-deletion Lambda ARN, `sqs:SendMessage` on the analyzer queue, S3 `GetObject` permissions for signing rrweb + selfie URLs. **Verify:** rendered policies match documented scopes; no wildcards.

## 2. Runtime workspace Terraform updates

- [ ] 2.1 Add API Gateway routes for the new endpoints (1.1–1.6) to the existing HTTP API. Each new route maps to the appropriate Lambda. **Verify:** `terraform plan` shows the new routes.
- [ ] 2.2 Add SSM parameter `/hiringiq/pilot/runtime/dashboard/reanalysis-rate-limit-per-24h` (default `3`) and `/hiringiq/pilot/runtime/dashboard/score-bands/{green-min, yellow-min}` (default `80`, `50`) for D-R6 + D-R9. **Verify:** parameters exist after apply.
- [ ] 2.3 Apply Terraform changes. **Verify:** clean `terraform plan` after; new routes available via API Gateway invoke URL.

## 3. Platform-shell dashboard scaffold (cross-workspace)

- [ ] 3.1 In the platform-shell workspace (`/Users/jgrenard/workspace/projects/PluralSight-POC/apps/web/`), add the route `/campaigns/:campaignId/sessions/:sessionId` to the React Router config. Page component `SessionDetail.tsx` is a stub at this step. **Verify:** navigating to the route renders the stub.
- [ ] 3.2 Add API client functions in `apps/web/src/lib/api/sessions.ts`: `getSession()`, `getSessionRrweb()`, `getSessionAuditEvents()`, `getApplicantSelfie()`, `deleteBiometric()`, `reAnalyzeSession()`. Each wraps the runtime workspace's HTTPS endpoint with the Cognito JWT in the Authorization header. **Verify:** unit tests mock the HTTPS responses + confirm types.
- [ ] 3.3 Add `rrweb-player` dependency to `apps/web/package.json`. **Verify:** `npm install` succeeds; the player imports without errors.

## 4. Integrity summary card component (cross-workspace)

- [ ] 4.1 Build `apps/web/src/components/integrity/IntegritySummaryCard.tsx`. Props: `summary: IntegritySummary`. Renders: large score number with color band (green ≥80, yellow 50-79, red <50, thresholds from SSM read at page load), behavioral_summary prose, flagged_moments list with timestamp + description + "Jump to this moment" buttons. **Verify:** snapshot tests cover all three score-band states.
- [ ] 4.2 Add the "Jump to this moment" button handlers. Each button takes a timestamp + emits an event to the parent SessionDetail page via prop callback. The parent page synchronizes timeline + rrweb player + audit log to the timestamp. **Verify:** click handler calls the prop callback with the correct timestamp.
- [ ] 4.3 Handle the `status: "analyzer_failed"` case per Phase 2 D-AL4 retry-exhaustion. Display "Analysis unavailable for this session" with a "Request manual re-analysis" button that calls `POST /api/sessions/<id>/re-analyze`. **Verify:** snapshot test covers the failed state.

## 5. Timeline scrubber component (cross-workspace)

- [ ] 5.1 Build `apps/web/src/components/integrity/TimelineScrubber.tsx`. Props: `timelineSegments: TimelineSegment[]`, `flaggedMoments: FlaggedMoment[]`, `currentTimestamp: number`, `onSeek: (timestamp: number) => void`. Renders: horizontal bar from session_start to session_end colored by segments (green/yellow/red CSS variables), with pin markers from flaggedMoments. **Verify:** snapshot test confirms the bar renders with correct band colors + pin positions.
- [ ] 5.2 Click handler: clicking anywhere on the bar OR a pin calls `onSeek(timestamp)`. The parent SessionDetail synchronizes the rrweb player + audit log viewer to the new timestamp. **Verify:** integration test confirms clicks produce the correct callbacks.
- [ ] 5.3 Render a current-position indicator (a thin vertical line on the bar at `currentTimestamp`). This updates as the rrweb player plays. **Verify:** snapshot tests confirm the indicator moves with timestamp updates.
- [ ] 5.4 Accessibility: timeline is keyboard-navigable (left/right arrows seek by 1s; Home/End jump to start/end). Pin markers are focusable; Enter activates the seek. **Verify:** a11y test confirms keyboard navigation.

## 6. rrweb playback viewer component (cross-workspace)

- [ ] 6.1 Build `apps/web/src/components/integrity/RrwebPlaybackViewer.tsx`. Props: `rrwebS3Key: string`, `currentTimestamp: number`, `onTimestampChange: (ts: number) => void`. On mount: fetches the signed URL via `getSessionRrweb()`, then fetches the rrweb event stream from S3, then initializes `rrweb-player`. **Verify:** snapshot test with a mocked event stream renders the player.
- [ ] 6.2 Implement the parent-driven seek: when `currentTimestamp` prop changes (from timeline click or flagged-moment jump), call `rrwebPlayer.goto(currentTimestamp)`. **Verify:** integration test confirms the player jumps when the prop changes.
- [ ] 6.3 Implement the player-driven timestamp update: when the player auto-advances during normal playback, call `onTimestampChange(currentTime)`. This updates the timeline's current-position indicator. **Verify:** integration test confirms timestamp propagation.
- [ ] 6.4 Standard playback controls: play/pause/seek bar, playback speed (1x, 1.5x, 2x), volume (N/A — rrweb doesn't carry audio). **Verify:** controls function as expected.
- [ ] 6.5 Handle errors gracefully: if the rrweb event stream is missing or corrupted, show "Recording unavailable" placeholder. **Verify:** integration test with a missing S3 object covers the failure path.

## 7. Raw audit log viewer component (cross-workspace)

- [ ] 7.1 Build `apps/web/src/components/integrity/RawAuditLogViewer.tsx`. Props: `sessionId: string`, `currentTimestamp: number`. Renders a paginated table of audit events. Default-collapsed behind an MD3 "Show full audit log" expander. **Verify:** snapshot test covers collapsed + expanded states.
- [ ] 7.2 Implement filtering by category (identity / session-lifecycle / integrity-signal / assessment) via MD3 chip toggles. Filter combinations are AND. **Verify:** unit test confirms filter logic.
- [ ] 7.3 Implement sort by timestamp (default ascending) + search by event_type (text input). **Verify:** snapshot test covers sorted + searched states.
- [ ] 7.4 Synchronization with the timeline: when `currentTimestamp` changes, the viewer scrolls to (and highlights) the events nearest to that timestamp. **Verify:** integration test confirms scroll behavior.
- [ ] 7.5 Pagination: 100 events per page. Default sort by timestamp. Page navigation via MD3 Pagination component. **Verify:** integration test confirms paging across multi-page sessions.

## 8. SessionDetail page wiring (cross-workspace)

- [ ] 8.1 Build the full `apps/web/src/pages/SessionDetail.tsx` page layout. Top: assessment metadata header (campaign name, applicant name, completion timestamp). Below: IntegritySummaryCard. Then side-by-side (or stacked on narrower viewports): TimelineScrubber on left, RrwebPlaybackViewer on right. Below: RawAuditLogViewer. Footer: action bar (re-analyze button). **Verify:** snapshot test covers the layout with mocked props.
- [ ] 8.2 Implement the shared `currentTimestamp` state at the page level. All four children (summary card, timeline, player, log viewer) read from + write to this state via props. **Verify:** integration test confirms clicking a flagged moment jumps all four child views to the same timestamp.
- [ ] 8.3 Implement the re-analyze button + confirmation modal. Calls `POST /api/sessions/<id>/re-analyze`. On success, shows a "Re-analysis in progress…" snackbar; the IntegritySummary card auto-refreshes ~30s later via polling (or via a backend webhook in v2). **Verify:** E2E test covers the click → request → snackbar sequence.

## 9. Session-list extension on campaign overview (cross-workspace)

- [ ] 9.1 Modify `apps/web/src/pages/CampaignOverview.tsx` (the existing page from platform-shell proposal #9). Add a new "Status" chip in the applicant rows showing the session state (Read / In-Progress / Completed / Declined / Expired / Abandoned). When the session is COMPLETED or EXPIRED, add a clickable IntegritySummary score badge. **Verify:** snapshot test covers the extended row.
- [ ] 9.2 The score badge color matches the IntegritySummary card's band colors (green/yellow/red). Hover tooltip shows the score number + analyzer version. **Verify:** snapshot test confirms hover state.
- [ ] 9.3 Add column-header filters: filter by status (multi-select chips), sort by score (ascending/descending). **Verify:** unit test confirms filter + sort behavior.
- [ ] 9.4 Add a "View session" link on each row navigating to `/campaigns/:campaignId/sessions/:sessionId`. **Verify:** click navigation confirmed via E2E test.

## 10. Right-to-deletion trigger (cross-workspace)

- [ ] 10.1 Add a "Delete biometric data" button to the APPLICANT detail page (in the platform-shell — `apps/web/src/pages/ApplicantDetail.tsx` per platform-shell proposal #9). Button is hidden if the applicant's biometric data has already been deleted (signaled by a `biometric_deleted_at` attribute on the applicant record). **Verify:** snapshot test covers the visible + hidden states.
- [ ] 10.2 Two-step confirmation modal: *"Are you sure? This will permanently delete the applicant's selfie, face embedding, and all session recordings. This cannot be undone."* + two buttons (Cancel, Yes, Delete). **Verify:** snapshot test confirms wording.
- [ ] 10.3 On confirm, call `POST /api/applicants/<id>/biometric-deletion`. On success, update the applicant record locally (set `biometric_deleted_at`), refresh the page. **Verify:** E2E test covers the deletion flow.
- [ ] 10.4 In the SessionDetail page, when an applicant has deleted biometric data, display "Biometric data for this applicant has been deleted" placeholder where the selfie thumbnail + rrweb viewer would otherwise be. **Verify:** snapshot test covers this state.

## 11. "What we monitor" disclosure page (cross-workspace)

- [ ] 11.1 Build `apps/web/src/pages/WhatWeMonitor.tsx`. Content per D-R8: capabilities list + limitations list + retention policy + deletion process. **Verify:** snapshot test confirms documented sections present.
- [ ] 11.2 Add a footer link from every dashboard page to the disclosure page. **Verify:** integration test confirms the link is present + navigable from at least 3 distinct dashboard pages.
- [ ] 11.3 Legal review of the disclosure content before the link goes live. **Verify:** doc reviewed + signed off by legal counsel (action item, not a code task).

## 12. End-to-end verification

- [ ] 12.1 Smoke test: log in as a test recruiter, navigate to a completed session's detail page. **Verify:** IntegritySummary card renders with the documented data; timeline scrubber displays colored bands; rrweb player loads and plays; audit-log viewer is collapsible + filterable.
- [ ] 12.2 Click a flagged_moment "Jump to this moment" button. **Verify:** timeline + rrweb player + audit-log viewer all synchronize to the timestamp.
- [ ] 12.3 Filter the audit log to a single category. **Verify:** only events in that category are displayed.
- [ ] 12.4 Click the re-analyze button. **Verify:** SQS message is enqueued; ~30s later, the IntegritySummary card shows the updated content (new analyzer_version).
- [ ] 12.5 Test the deletion flow end-to-end. **Verify:** clicking "Delete biometric data" → confirmation → success → selfie thumbnail + rrweb viewer replaced with the "deleted" placeholder on the session detail page; audit log shows `biometric.deletion_requested` + `biometric.deleted` events.
- [ ] 12.6 Test cross-org isolation. **Verify:** a recruiter from OrgA attempting to fetch a session in OrgB returns 403 from the runtime backend endpoint; no data leaks.
- [ ] 12.7 Test the `analyzer_failed` state. **Verify:** a session with `status: "analyzer_failed"` displays the "Analysis unavailable" placeholder + "Request manual re-analysis" button.
- [ ] 12.8 Test access logging end-to-end. **Verify:** viewing a session generates `recruiter_access.viewed_session`, viewing the replay generates `recruiter_access.viewed_replay`, etc. — all visible in the audit-events table tagged with category `recruiter-access`.
- [ ] 12.9 Test the "What we monitor" disclosure. **Verify:** the page is accessible from the footer link; content includes both capabilities + limitations as documented.

## 13. Documentation

- [ ] 13.1 Update `apps/web/README.md` (platform-shell): document the new SessionDetail route + the four new integrity components + the dashboard-runtime API surface. **Verify:** README enumerates these.
- [ ] 13.2 Update `services/assessment/README.md` and `services/applicant/README.md` (runtime): document the new dashboard endpoints + the recruiter-access-emitter helper. **Verify:** READMEs documented.
- [ ] 13.3 Add `docs/recruiter-dashboard-usage.md` to the platform-shell documenting how recruiters interact with the session-review experience: what each panel shows, how to interpret the integrity score, how to deal with flagged sessions. **Verify:** the doc exists with at least 3 worked examples.

## Dependencies on other proposals

**Hard dependencies:**
- This workspace's Phase 1 + Phase 2 (provides the data the dashboard consumes).
- Platform-shell proposal #3 (`add-recruiter-authentication`) — Cognito user pool for recruiter login (already in place).
- Platform-shell proposal #4 (`add-react-app-shell-and-branding`) — the React app + MUI v6 + brand tokens.
- Platform-shell proposal #9 (`add-campaign-overview-and-applicant-assignment`) — the campaign overview page this proposal extends + the ApplicantDetail page for the deletion button.

**Cross-workspace edits:**
- Modifies platform-shell `apps/web/` source files (new components, new pages, extension of CampaignOverview + ApplicantDetail).
- Does NOT modify any platform-shell OpenSpec proposals (the modification is to React code, not specs).
- The platform-shell #9's `campaign-overview` capability is conceptually MODIFIED by this proposal but the modification is captured in this proposal's spec (`session-list-view`), not retroactively edited into proposal #9.

**Recommended apply order:**
After all platform-shell proposals + runtime Phase 1 + runtime Phase 2 apply.

**Unblocks:** Final phase. After this, the runtime workstream is feature-complete for pilot.
