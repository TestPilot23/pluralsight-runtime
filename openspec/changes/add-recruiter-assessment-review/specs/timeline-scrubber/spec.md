# Timeline Scrubber

## ADDED Requirements

### Requirement: Renders colored bands from timeline_segments (D-R7)

The system SHALL render a horizontal bar spanning the full width of its container, divided into segments according to the IntegritySummary's `timeline_segments` array. Each segment is colored:
- **Green** for `severity: "green"`
- **Yellow** for `severity: "yellow"`
- **Red** for `severity: "red"`

The segments together cover the full session timeline from `session_start` to `session_end` (Phase 2 D-RR3 guarantees full coverage). The segment widths are proportional to their duration relative to the total session length.

#### Scenario: Three-segment timeline renders proportionally
- **WHEN** a session lasts 30 minutes with segments at green [0-20min], yellow [20-25min], red [25-30min]
- **THEN** the timeline displays a green band covering the left 2/3, a yellow band covering 1/6, and a red band covering 1/6
- **AND** the total width fills the container

#### Scenario: Clean session shows single green band
- **WHEN** a session has a single green segment spanning the entire duration
- **THEN** the timeline displays a single green band filling the container

### Requirement: Flagged moments render as pin markers

The system SHALL render the IntegritySummary's `flagged_moments` array as pin markers (small icons or vertical lines) positioned at the moments' timestamps along the bar. Pins are visually distinct from the colored bands behind them.

Pins are clickable; clicking a pin calls `onSeek(timestamp)` to synchronize the linked views.

Hovering a pin SHALL display a tooltip with the moment's `description`.

#### Scenario: Pin renders at correct position
- **WHEN** a session has a flagged_moment at 14:30 (50% through a 29:00 session)
- **THEN** the timeline displays a pin marker at the horizontal midpoint of the bar

#### Scenario: Pin hover shows description
- **WHEN** the recruiter hovers a pin marker
- **THEN** a tooltip appears with the flagged_moment's `description` text

#### Scenario: Pin click triggers seek
- **WHEN** the recruiter clicks a pin
- **THEN** the parent page's `onSeek` handler is called with the pin's timestamp

### Requirement: Click-to-seek on the bar (D-R7)

Clicking anywhere on the timeline bar (not just on pins) SHALL call `onSeek(timestamp)` with the timestamp corresponding to the click's horizontal position.

The timestamp calculation: `clickX / barWidth * (sessionEnd - sessionStart) + sessionStart`.

#### Scenario: Click at 50% width seeks to mid-session
- **WHEN** the recruiter clicks the horizontal middle of the timeline bar of a 30-minute session
- **THEN** `onSeek(...)` is called with the session's midpoint timestamp (sessionStart + 15 minutes)

### Requirement: Current-position indicator updates with playback

The system SHALL render a current-position indicator (e.g., a thin vertical line) on the timeline at the position corresponding to the `currentTimestamp` prop. As the rrweb player advances during playback, `currentTimestamp` updates via the parent state, and the indicator moves accordingly.

#### Scenario: Indicator moves during playback
- **WHEN** the rrweb player plays from session_start
- **THEN** the timeline's current-position indicator moves rightward in real-time
- **WHEN** the recruiter pauses the player
- **THEN** the indicator stops moving

### Requirement: Keyboard navigation (accessibility)

The system SHALL support keyboard navigation on the timeline:
- Left/Right arrow keys seek by 1 second (or 1% of session duration, whichever is larger)
- Home/End keys jump to session start/end
- Tab navigates between pin markers; Enter activates the seek on a focused pin

The timeline element SHALL have appropriate ARIA roles + labels for screen readers.

#### Scenario: Arrow keys seek the timeline
- **WHEN** the timeline has keyboard focus and the recruiter presses the Right arrow
- **THEN** `onSeek` is called with the current timestamp + 1 second
