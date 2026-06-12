# Post-Session Integrity Analyzer

## ADDED Requirements

### Requirement: SQS-triggered Lambda on session-end transitions (D-AL4)

The system SHALL trigger the analyzer-service Lambda via SQS messages on session-end state transitions:
- `session.completed` → enqueue analyzer message
- `session.expired` → enqueue analyzer message (partial data; recruiter sees "Session never completed; partial analysis available")
- `applicant.declined.confirmed` → DO NOT enqueue (no events to analyze)
- `session.abandoned` → DO NOT enqueue (recoverable state; analyzer waits for resume OR expiration)

The SQS queue `hiringiq-pilot-analyzer-queue` was provisioned in Phase 1 with the no-op stub Lambda. This phase replaces the stub with the real implementation.

#### Scenario: COMPLETED triggers analyzer
- **WHEN** a session transitions to COMPLETED
- **THEN** an SQS message lands in the analyzer queue with `{ session_id, transition_reason: "completed" }`
- **AND** the analyzer Lambda is invoked

#### Scenario: DECLINED does NOT trigger analyzer
- **WHEN** an applicant clicks "Yes, I'm sure" on the intro screen's decline modal
- **AND** the session transitions to DECLINED
- **THEN** NO SQS message is enqueued
- **AND** the audit log records `applicant.declined.confirmed` but no analyzer invocation occurs

#### Scenario: ABANDONED does NOT trigger analyzer
- **WHEN** a session transitions to ABANDONED (30-min PAUSED timeout)
- **THEN** NO SQS message is enqueued (waiting for either resume or JWT expiry → EXPIRED → then analyzer)

### Requirement: Analyzer consumes full session context (D-AL5)

The analyzer-service Lambda SHALL fetch all of the following for the session being analyzed:
- All audit-log events from `hiringiq-pilot-audit-events` (paginated read until exhausted)
- The campaign record from the platform-shell's tables
- All items the applicant saw (queried via `BatchGetItem` from `hiringiq-pilot-items`, with the `answer` field STRIPPED from each item before inclusion in the LLM prompt)
- The applicant's responses + timing per item (from the session record)
- The Persona result + confidence + attempts (from the applicant record)
- The face-detection confidence trajectory (derived from audit-log events)
- The rrweb recording S3 key (for reference in the IntegritySummary; the analyzer does NOT play back the recording itself — that's the recruiter's job in Phase 3)

**The analyzer NEVER sees correct answers.** Item answers are stripped before the prompt is constructed — this preempts prompt-injection attacks where an applicant's short-answer text could include "the correct answer to this question would be…"

#### Scenario: Analyzer fetches all documented data
- **WHEN** the analyzer Lambda runs for a session
- **THEN** the constructed LLM prompt contains the campaign metadata, item count, audit-log events, applicant responses, and Persona result
- **AND** the prompt does NOT contain any item's `answer` field

#### Scenario: Answer-stripping is enforced
- **WHEN** an item with `answer: 2` is included in the prompt context
- **THEN** the serialized item in the prompt has no `answer` field at all
- **AND** a code-review grep confirms the answer-stripping helper is invoked before prompt construction

### Requirement: LLM prompt forbids judgment language (D-AL4, D-AL6, D-I9)

The analyzer-service prompt SHALL include explicit instructions to the LLM to DESCRIBE events factually and NOT characterize cheating intent. The exact anchor language:

> *"You are an assessment-integrity analyzer. Your job is to describe what happened during an assessment session FACTUALLY without judging cheating intent. DO NOT use language like 'appears to be cheating' or 'evidence of cheating.' Describe events neutrally and let the reviewer draw conclusions."*

The prompt SHALL also instruct the LLM to look for the screenshot-to-LLM round-trip threat pattern:

> *"Look for indicators of external assistance: items where the applicant's response time was significantly longer than typical for similar items in this assessment, particularly when (a) there was no input activity during the elapsed time, (b) the final answer appeared in a short burst, (c) face-detection confidence or head-pose indicated reduced screen attention during the gap, and (d) the answer quality contrasts notably with other answers in the same session. Describe each such pattern factually without asserting cheating intent."*

#### Scenario: Prompt contains the anchor language
- **WHEN** the analyzer prompt template is rendered
- **THEN** the rendered prompt contains the literal SYSTEM message + the round-trip detection guidance verbatim

#### Scenario: Behavioral summaries contain no judgment language
- **WHEN** 10 representative IntegritySummary records are produced and inspected
- **THEN** the `behavioral_summary` fields contain no strings like "cheat", "fraud", "dishonest", "evidence of cheating"
- **AND** descriptions use factual language ("out of frame for 47 seconds", "paste of 23-character string into short_answer field at 12:15")

### Requirement: IntegritySummary shape with timeline_segments (D-RR3)

The analyzer SHALL produce an `IntegritySummary` record with the documented shape:

```ts
{
  session_id: string,
  generated_at: string,
  analyzer_version: string,           // e.g., "v1@amazon.nova-pro-v1:0"

  integrity_score: number,            // 0-100, neutral numerical
  behavioral_summary: string,         // descriptive LLM prose

  flagged_moments: Array<{
    event_ids: string[],
    timestamp_server: string,
    description: string,              // factual only
  }>,

  timeline_segments: Array<{
    start_timestamp: string,
    end_timestamp: string,
    severity: "green" | "yellow" | "red",
    description: string,
    related_event_ids: string[],
  }>,

  raw_events_count: number,
  rrweb_recording_s3_key: string,
}
```

The summary is persisted to `hiringiq-pilot-integrity-summaries` DDB table (PK=session_id).

**No categorical determination field.** The output explicitly omits the `determination` enum (`clean | concerning | flagged_cheating`) — the recruiter draws conclusions from the score + summary + flagged moments + raw audit log.

#### Scenario: Summary has all documented fields
- **WHEN** the analyzer completes for a test session
- **THEN** the persisted IntegritySummary record contains every documented field
- **AND** no extra fields (specifically no `determination` enum) are present

### Requirement: Timeline segments cover the entire session (D-RR3, D-I9)

The `timeline_segments` array SHALL cover the entire session from start to end with no gaps. The analyzer's prompt instructs the LLM to produce this; if the LLM emits gappy output, the analyzer post-processes by inserting `green` segments to fill gaps before persisting.

The three severity bands:
- `green` — no notable signals during this range
- `yellow` — signals consistent with brief inattention or honest hardware issues
- `red` — signals that strongly correlate with cheating patterns described in the prompt schema

#### Scenario: LLM produces gappy output → analyzer fills with green
- **WHEN** the LLM produces three flagged ranges with gaps between them
- **THEN** the analyzer's post-processor inserts `green` segments covering the gaps
- **AND** the final persisted timeline_segments cover the entire session_start to session_end timeline with no gaps

#### Scenario: Clean session has all-green timeline
- **WHEN** a session with no integrity signals is analyzed
- **THEN** the timeline_segments contain a single green segment covering session_start to session_end
- **AND** the flagged_moments array is empty
- **AND** the integrity_score is high (>80 typically)

### Requirement: Versioned + re-runnable (D-AL4)

The analyzer SHALL include the `analyzer_version` field in every IntegritySummary, in the format `v<prompt_version>@<model_id>` (e.g., `v1@amazon.nova-pro-v1:0`).

When the prompt or model changes, re-running the analyzer on a historical session creates a NEW summary record at the new version. Past versions are NOT overwritten — they coexist for A/B comparison and audit.

The DDB table's PK is `session_id` (only one row per session at a time), but historical versions are stored in a sibling table `hiringiq-pilot-integrity-summary-versions` (PK=session_id, SK=analyzer_version) to support re-runs. This table is provisioned in Phase 2.

#### Scenario: Re-run with new prompt creates new version row
- **WHEN** the analyzer is invoked twice on the same session — first with `v1`, then with `v2`
- **THEN** `hiringiq-pilot-integrity-summaries` reflects the latest version (`v2`)
- **AND** `hiringiq-pilot-integrity-summary-versions` contains BOTH `v1` and `v2` rows
- **AND** the dashboard can fetch a specific historical version on demand

### Requirement: Bedrock model is SSM-configurable (D-I4)

The analyzer SHALL read the Bedrock model ID from SSM parameter `/hiringiq/pilot/runtime/integrity/analyzer-model` at Lambda cold-start (cached for the Lambda's lifetime). Default: `amazon.nova-pro-v1:0`.

Changing the SSM value + Lambda cold-start (e.g., via redeploy or after a quiet period) picks up the new model without code change.

#### Scenario: Model switch via SSM
- **WHEN** the SSM parameter is updated from `amazon.nova-pro-v1:0` to `anthropic.claude-3-5-sonnet-20241022-v2:0`
- **AND** the next analyzer invocation is a cold start
- **THEN** the analyzer uses the Claude model for that invocation
- **AND** the resulting IntegritySummary has `analyzer_version: "v1@anthropic.claude-3-5-sonnet-20241022-v2:0"`

### Requirement: Structured-output validation + retry

The analyzer SHALL validate the Bedrock model's JSON response against the IntegritySummary Zod schema. On validation failure:
- The analyzer retries with a follow-up prompt asking the model to fix the output to match the schema
- After 3 retries without valid output, the analyzer persists an IntegritySummary record with `status: "analyzer_failed"` + the partial Bedrock response in a `raw_model_output` field
- A CloudWatch alarm fires on `analyzer_failed` records exceeding 5% over a 24-hour window

#### Scenario: Malformed JSON triggers retry
- **WHEN** the Bedrock response is not valid JSON
- **THEN** the analyzer constructs a follow-up prompt
- **AND** retries the call
- **AND** if the retry produces valid JSON, the analyzer proceeds normally

#### Scenario: Persistent failures persist with analyzer_failed status
- **WHEN** Bedrock fails to produce valid output after 3 retries
- **THEN** the IntegritySummary record has `status: "analyzer_failed"` and the partial output in `raw_model_output`
- **AND** the recruiter dashboard in Phase 3 shows "Analysis unavailable for this session — request manual re-analysis"

### Requirement: SQS DLQ for poison messages

The analyzer queue SHALL have a Dead-Letter Queue (DLQ) configured. Messages that fail the analyzer Lambda processing 3 times move to the DLQ. A CloudWatch alarm fires on DLQ depth > 0.

The DLQ is `hiringiq-pilot-analyzer-dlq` SQS queue.

#### Scenario: Repeated processing failure routes to DLQ
- **WHEN** the analyzer Lambda fails (uncaught exception) 3 consecutive times on the same SQS message
- **THEN** the message moves to the DLQ
- **AND** the CloudWatch alarm transitions to ALARM state
- **AND** the SNS topic `hiringiq-pilot-runtime-alerts` receives the alarm notification
