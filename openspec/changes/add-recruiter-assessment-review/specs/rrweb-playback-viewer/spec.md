# rrweb Playback Viewer

## ADDED Requirements

### Requirement: rrweb-player loads recording via signed URL (D-R3, D-R10)

The system SHALL embed the MIT-licensed `rrweb-player` component to render assessment session replays. On mount, the component:
1. Calls `GET /api/sessions/<id>/rrweb` on the runtime backend
2. Receives a 15-minute pre-signed S3 GET URL (per Phase 2 D-RR2)
3. Fetches the rrweb event stream JSON from S3 via that URL
4. Initializes `rrweb-player` with the loaded event stream

The component SHALL NOT cache the rrweb event stream in localStorage / sessionStorage / IndexedDB (per D-R10). Each navigation to the session detail page re-fetches fresh.

#### Scenario: Player loads recording on mount
- **WHEN** a recruiter navigates to the session detail page
- **THEN** the player issues `GET /api/sessions/<id>/rrweb`
- **AND** receives a signed S3 URL within ~500ms
- **AND** fetches the rrweb event stream
- **AND** initializes the player with the events ready to play

#### Scenario: No browser-side caching of recording
- **WHEN** the recruiter navigates away from the session detail page and returns
- **THEN** the player re-issues `GET /api/sessions/<id>/rrweb` (fresh signed URL)
- **AND** re-fetches the event stream from S3 (no localStorage hit)

### Requirement: Recording fetch emits recruiter_access.viewed_replay audit event

The runtime backend SHALL emit `recruiter_access.viewed_replay` audit event (silent per D-AL7) whenever the rrweb signing endpoint is called by an authenticated recruiter, BEFORE returning the signed URL.

The payload includes the recruiter's user_id, email, the session_id, applicant_id, org_id, and the recruiter's IP.

#### Scenario: Replay view emits audit event
- **WHEN** a recruiter loads the session detail page
- **THEN** an audit event `recruiter_access.viewed_replay` appears in the audit-events table within ~1 second
- **AND** the payload includes the recruiter's identity + session + applicant

### Requirement: Player synchronizes with timeline via currentTimestamp prop (D-R7)

The system SHALL synchronize the rrweb player's playback position with the timeline's `currentTimestamp` state via component props. The component takes `currentTimestamp: number` and `onTimestampChange: (ts: number) => void` props. Synchronization rules:
- When `currentTimestamp` changes (e.g., from a timeline click or flagged-moment jump), the player calls `rrwebPlayer.goto(currentTimestamp)` to seek the playback to that point
- When the player auto-advances during normal playback, the component calls `onTimestampChange(currentTime)` so the timeline + audit log update their current-position indicators

#### Scenario: Parent-driven seek jumps the player
- **WHEN** the timeline scrubber's onSeek fires with timestamp T
- **AND** the parent updates the currentTimestamp prop to T
- **THEN** the player calls `rrwebPlayer.goto(T)` and the playback position visibly jumps

#### Scenario: Player-driven progress updates parent
- **WHEN** the player advances during normal playback from T to T+1s
- **THEN** the component calls `onTimestampChange(T+1)` once per second (or at the player's natural update cadence)

### Requirement: Standard playback controls

The system SHALL provide standard rrweb-player controls:
- Play / Pause toggle
- Seek bar (separate from the timeline scrubber — this is the player's own internal control)
- Playback speed selector (1x, 1.5x, 2x)
- Skip forward/back by 10 seconds

Audio controls are N/A (rrweb does not carry audio).

#### Scenario: Speed control works
- **WHEN** the recruiter selects 2x playback speed
- **THEN** the recording plays at twice normal speed

### Requirement: Missing recording graceful fallback

The system SHALL handle the case where the rrweb recording is missing (deleted by right-to-deletion, or never captured for some reason). Display a placeholder:
> *"Session recording is unavailable."*

If the unavailability is due to deletion (the session's applicant has `biometric_deleted_at` set), display:
> *"Session recording was deleted on {biometric_deleted_at} per the applicant's request."*

#### Scenario: Deleted recording shows deletion-aware message
- **WHEN** the session's applicant has `biometric_deleted_at: "2026-08-15T10:00:00Z"`
- **THEN** the player displays the deletion-aware message with that timestamp

#### Scenario: Missing recording (other reasons) shows generic unavailable message
- **WHEN** the session was completed but no rrweb recording was ever captured (e.g., session predates Phase 2 apply)
- **THEN** the player displays the generic unavailable message
