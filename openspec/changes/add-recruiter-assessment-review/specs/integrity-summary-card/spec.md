# Integrity Summary Card

## ADDED Requirements

### Requirement: Score with three-band color treatment (D-R6)

The system SHALL render the IntegritySummary's `integrity_score` (0-100) as a large numeric value with a color band:
- **Green** if score ≥ green-threshold (default 80, SSM-configurable at `/hiringiq/pilot/runtime/dashboard/score-bands/green-min`)
- **Yellow** if 50 ≤ score < 80
- **Red** if score < 50 (default; SSM-configurable at `.../score-bands/yellow-min`)

The color band is displayed as a colored ribbon along the left edge of the card OR as a background tint behind the score number. The exact score number is ALSO displayed numerically (the color is an aid, not the only signal).

#### Scenario: High score renders green
- **WHEN** an IntegritySummary with `integrity_score: 92` is rendered
- **THEN** the card displays "92" with a green color treatment

#### Scenario: Low score renders red
- **WHEN** an IntegritySummary with `integrity_score: 42` is rendered
- **THEN** the card displays "42" with a red color treatment

#### Scenario: Threshold updates change band assignment
- **WHEN** the SSM green-threshold is updated from 80 to 90
- **AND** the page is reloaded
- **THEN** a score of 85 that was previously displayed green now displays yellow

### Requirement: Behavioral summary prose rendering

The system SHALL render the IntegritySummary's `behavioral_summary` text as plain prose below the score. Typography: MD3 body-large size. The text wraps naturally to the card's width. No special parsing (no Markdown, no HTML — the field is plain text from the LLM per Phase 2 D-AL5).

If the prose contains paragraphs separated by double line breaks, the renderer SHALL preserve the paragraph break visually.

#### Scenario: Multi-paragraph summary renders with breaks
- **WHEN** the behavioral_summary contains "Para 1.\n\nPara 2.\n\nPara 3."
- **THEN** the card displays three visually separated paragraphs

#### Scenario: Plain text only — no XSS opportunity
- **WHEN** the behavioral_summary contains `"<script>alert('xss')</script>"`
- **THEN** the card displays the literal text (no script execution)

### Requirement: Flagged moments list with jump-to-moment buttons (D-R7)

The system SHALL render the `flagged_moments` array as a vertical list below the behavioral summary. Each entry displays:
- The moment's `timestamp_server` formatted as `MM:SS` relative to session start
- The moment's `description` text
- A "Jump to this moment" button that, when clicked, calls the parent SessionDetail's `onSeek(timestamp)` handler to synchronize the timeline + replay + audit log to that point

If `flagged_moments` is empty (clean session), the list area shows "No flagged moments — session was clean throughout."

#### Scenario: Flagged moments list renders entries
- **WHEN** the summary has 3 flagged_moments at session-relative timestamps 02:15, 14:42, and 27:09
- **THEN** the list displays 3 entries with those formatted timestamps and the associated descriptions
- **AND** each has a "Jump to this moment" button

#### Scenario: Jump button calls parent seek handler
- **WHEN** the recruiter clicks "Jump to this moment" on a flagged_moment at timestamp 14:42
- **THEN** the parent page's `onSeek` is called with 14:42 (in absolute Unix seconds)
- **AND** the timeline + player + audit log all synchronize

### Requirement: analyzer_failed placeholder

The system SHALL handle the `status: "analyzer_failed"` state by displaying:
> *"Analysis unavailable for this session."*
> *"The integrity analyzer failed to process this session's data. You can request a manual re-analysis from the page footer."*

The IntegritySummaryCard SHALL NOT display a score, behavioral_summary, or flagged_moments list in this state — those fields are absent or partial in the data.

#### Scenario: analyzer_failed renders placeholder
- **WHEN** the IntegritySummary has `status: "analyzer_failed"`
- **THEN** the card displays the documented placeholder text
- **AND** no score number, color band, or flagged-moments list is rendered
