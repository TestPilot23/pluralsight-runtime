# Assessment Interface

## ADDED Requirements

### Requirement: Option B layout — webcam corner card + full-width content (D-UI1)

The assessment interface SHALL use the Option B layout: a fixed top-left webcam corner card, full-width content area for the question + answer, top-right timer (when applicable), Submit at bottom-right of the content area.

The layout SHALL respect a minimum viewport width of 1024px. Below this, the magic-link validation chain already routes the applicant to the "Use a larger device" page (D-DC5) — so the interface itself does not need to handle smaller viewports.

#### Scenario: Layout renders with documented regions
- **WHEN** the assessment interface mounts on a 1440px desktop browser
- **THEN** the WebcamCard is positioned at top-left at ~260×220px
- **AND** the question text is rendered in a full-width reading column to the right of the WebcamCard
- **AND** the Submit button is positioned at the bottom-right of the content area

### Requirement: WebcamCard with Hide / minimized chip (D-UI2, Phase 1 scope)

The WebcamCard SHALL be a fixed/sticky top-left component, always rendered during the assessment. The card SHALL support two states at Phase 1:

- **Expanded (default):** 240×180 4:3 webcam preview + a "Hide Camera" button below (Material Symbols `visibility_off` icon + label "Hide Camera"). Total card size ~260×220px.
- **Minimized:** ~80×32 chip showing a pulsing red dot + "REC" text. Slow pulse animation on the dot (~2-second cycle, MD3 motion tokens) reinforces that recording IS still happening. Click anywhere on the chip → expand.

The **warning state** (red 3px border on the expanded card during face_not_detected) and the **auto-expand-on-warning** behavior from D-UI2 are NOT in scope at Phase 1 — they require the face-api.js warning system which arrives in Phase 2. At Phase 1, the card has no warning visual.

#### Scenario: Expanded card renders with Hide Camera button
- **WHEN** the assessment interface mounts
- **THEN** the WebcamCard is in expanded state showing the 240×180 preview + "Hide Camera" button below
- **AND** the live webcam stream renders in the preview

#### Scenario: Hide Camera transitions to minimized chip
- **WHEN** the applicant taps the "Hide Camera" button
- **THEN** the card transitions to the minimized chip state showing `● REC`
- **AND** the red dot pulses with a ~2-second cycle
- **AND** the audit event `webcam_view.visibility_changed` fires with payload `{ visible: false, trigger: "user_action" }`

#### Scenario: Click on chip restores expanded state
- **WHEN** the applicant clicks the minimized `● REC` chip
- **THEN** the card transitions back to expanded state
- **AND** the audit event `webcam_view.visibility_changed` fires with payload `{ visible: true, trigger: "user_action" }`

### Requirement: Item-type renderer dispatch (D-UI4)

The system SHALL dispatch the right-pane content area renderer based on `item.type`. Each item type has its own renderer component:

| `item.type` | Renderer | Bound state shape |
|---|---|---|
| `multiple_choice_single` | MD3 Radio button group | `answer: number` (0-based index into choices) |
| `multiple_choice_multiple` | MD3 Checkbox group | `answer: number[]` (UI submits sorted ascending, deduplicated) |
| `short_answer` | MD3 TextArea | `answer: string` |

Item text (the question prompt) SHALL be displayed in a full-width reading column. For MC types, the question text SHALL be non-selectable via `user-select: none` and `contextmenu` blocking. For short-answer items, copy/paste is ALLOWED — every copy and paste event SHALL emit an audit-log event with the content + timestamp + item_id.

The friction primitives (`user-select: none`, contextmenu blocking) are NOT real security boundaries — dev tools defeats them in seconds. The real signal is the audit-log events; selection blocking just raises effort for casual cheaters.

#### Scenario: MC-single item renders with radio group
- **WHEN** the assessment interface displays an item with `type: "multiple_choice_single"` and `choices: ["A", "B", "C", "D"]`
- **THEN** the right pane renders an MD3 Radio button group with 4 options
- **AND** exactly one radio can be selected at a time

#### Scenario: MC-multiple item renders with checkbox group
- **WHEN** the assessment interface displays an item with `type: "multiple_choice_multiple"` and `choices: ["A", "B", "C", "D"]`
- **THEN** the right pane renders an MD3 Checkbox group with 4 options
- **AND** any subset can be selected

#### Scenario: Short-answer item renders with textarea + copy/paste allowed
- **WHEN** the assessment interface displays an item with `type: "short_answer"`
- **THEN** the right pane renders an MD3 TextArea
- **AND** the textarea accepts copy and paste keyboard shortcuts
- **WHEN** the applicant pastes text into the textarea
- **THEN** an audit event `paste.detected` fires with payload `{ text: <pasted-text>, item_id, timestamp_client, timestamp_server }`

#### Scenario: MC item text is not selectable
- **WHEN** an MC item is displayed and the applicant attempts to highlight the question text with their mouse
- **THEN** no text becomes selected
- **AND** right-clicking on the question text produces no context menu

### Requirement: Submit button disabled-until-valid (D-UI5)

The Submit button SHALL be rendered always but disabled until a valid selection exists for the current item type:
- `multiple_choice_single`: enabled when exactly one radio is selected
- `multiple_choice_multiple`: enabled when at least one checkbox is selected
- `short_answer`: enabled when the textarea has at least 1 non-whitespace character

On hover of the disabled state, a tooltip SHALL display *"Make a selection to continue"* (MC types) or *"Enter your answer to continue"* (short answer).

#### Scenario: MC-single Submit enables on selection
- **WHEN** an MC-single item is displayed with no radio selected
- **THEN** Submit is disabled with the documented tooltip on hover
- **WHEN** the applicant selects a radio
- **THEN** Submit becomes enabled

#### Scenario: Short-answer Submit ignores whitespace-only input
- **WHEN** the applicant types only spaces into the textarea
- **THEN** Submit remains disabled

### Requirement: Timer in top-right when item has a time limit (D-UI6)

If `item.time_limit_seconds` is non-null, the system SHALL render a countdown timer in the top-right of the interface in the format `⏱ MM:SS`. The timer is purely visual — server-side enforcement of time-up is authoritative.

When the timer reaches 0:
- Server-side enforcement auto-submits the applicant's current selection (D-UI6)
- For `short_answer` items, the textarea is disabled immediately to prevent further typing
- A 2-second toast appears: *"Time's up. Submitting your answer."*
- After the toast dismisses, the system auto-advances to the next item

If `item.time_limit_seconds` is null, no timer is rendered.

#### Scenario: Timer renders for time-limited item
- **WHEN** an item with `time_limit_seconds: 60` is displayed
- **THEN** the top-right shows a countdown starting at `⏱ 01:00`
- **AND** the timer decrements every second

#### Scenario: Time expiry auto-submits and advances
- **WHEN** the timer reaches `⏱ 00:00` on a `short_answer` item with text "my answer" in the textarea
- **THEN** the system emits `item.submitted` with `{ answer: "my answer", auto_submitted: true }`
- **AND** the textarea is disabled
- **AND** a 2-second toast displays "Time's up. Submitting your answer."
- **AND** after 2 seconds, the next item displays

#### Scenario: Time expiry with no selection submits null
- **WHEN** the timer reaches 0 on an MC item with no radio selected
- **THEN** the system emits `item.null_submission` with `{ item_id, auto_submitted: true }`
- **AND** the system advances to the next item

### Requirement: Copy/paste events on short-answer items

The short-answer textarea SHALL fire audit events on every copy and paste action:
- `copy.detected` — when the applicant copies text from the textarea
- `paste.detected` — when the applicant pastes text into the textarea

Each event SHALL include payload `{ text, item_id_at_time, timestamp_client, timestamp_server }`.

These events are emitted in Phase 1 because the textarea is the originating UI surface. The Phase 2 analyzer is the consumer of these events; at Phase 1 they accumulate in the audit log unused.

#### Scenario: Paste fires audit event with content
- **WHEN** the applicant pastes the string "binary search" into a short-answer textarea on item `I7`
- **THEN** an audit event `paste.detected` is written with `text: "binary search"` and `item_id_at_time: "I7"`

### Requirement: Submit click emits item.submitted

When the applicant clicks Submit on an item, the system SHALL emit `item.submitted` with payload:
```ts
{
  item_id: string,
  answer: number | number[] | string,    // type-specific per discriminated union
  auto_submitted: false,                  // true only on timer expiry
  timestamp_client: string,
  timestamp_server: string,
}
```

After the server confirms the submission and writes the `served_answered` state, the next item is fetched and displayed.

#### Scenario: Submit on MC-single records the index
- **WHEN** the applicant selects option index 2 on an MC-single item `I3` and clicks Submit
- **THEN** the audit event `item.submitted` fires with `{ item_id: "I3", answer: 2, auto_submitted: false, ... }`
- **AND** the next item is fetched and displayed
