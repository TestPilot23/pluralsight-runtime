# IntegritySummary

The post-session Bedrock analyzer's output. One per terminal session. Drives the recruiter-facing integrity score, behavioral summary, flagged moments, and timeline coloring.

## Storage

- **Table:** `hiringiq-pilot-integrity-summaries`
- **PK:** `session_id` (String, UUID v7)
- **GSI:** `applicant_id`
- **GSI:** `org_id`
- No sort key.

Encrypted at rest. PITR enabled. On-demand billing.

## Attributes (D-RR3)

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `session_id` | String (UUID v7) | Yes | Matches PK |
| `applicant_id` | String (UUID v7) | Yes | GSI for cross-session queries |
| `org_id` | String (UUID v7) | Yes | GSI for org-scoped lists |
| `status` | String | Yes | `"complete"` \| `"analyzer_failed"` (after DLQ exhaustion) |
| `integrity_score` | Number | Yes (when `status="complete"`) | 0–100; higher = cleaner session |
| `behavioral_summary` | String | Yes (when `status="complete"`) | Free-text prose; **no judgment language** (no "cheat" / "fraud" / "dishonest") |
| `flagged_moments` | FlaggedMoment[] | Yes (when `status="complete"`) | Specific timestamps the recruiter should jump to |
| `timeline_segments` | TimelineSegment[] | Yes (when `status="complete"`) | Contiguous coverage of the session timeline, each segment colored green/yellow/red |
| `analyzer_version` | String | Yes | Format: `v1@<bedrock-model-id>` (e.g., `v1@amazon.nova-pro-v1:0`) |
| `created_at` | String | Yes | ISO-8601 UTC |
| `updated_at` | String | Yes | ISO-8601 UTC; updated on re-analysis |

### `FlaggedMoment`

| Field | Type | Notes |
|---|---|---|
| `timestamp` | String | ISO-8601 UTC; "Jump to this moment" target |
| `severity` | Enum | `"yellow"` \| `"red"` |
| `description` | String | Factual description (e.g., `"Out of frame for 12 seconds during item 4"`) |
| `related_event_ids` | String[] | References to the [[AuditEvent]]s that triggered this moment |

### `TimelineSegment`

| Field | Type | Notes |
|---|---|---|
| `start_timestamp` | String | ISO-8601 UTC |
| `end_timestamp` | String | ISO-8601 UTC |
| `severity` | Enum | `"green"` \| `"yellow"` \| `"red"` |
| `reason` | String \| null | Optional short label (e.g., `"sustained paste activity"`); null for green segments |

## Invariants (D-RR3)

- `timeline_segments` cover the **entire** session timeline contiguously — no gaps. If the LLM produces gappy output, the analyzer programmatically fills gaps with green segments before persisting.
- `behavioral_summary` is factual, never accusatory. The prompt template explicitly prohibits judgment language. A regex check in CI on representative samples greps for `"cheat" | "fraud" | "dishonest"` — these must not appear.
- Score bands (SSM-configurable, defaults documented):
  - `green`: ≥ 80
  - `yellow`: 50–79
  - `red`: < 50
- A `DECLINED` [[Session]] produces **no** IntegritySummary (D-AL4). The dashboard handles missing summaries gracefully.
- Re-analyzing a session writes a new record with the same `session_id` PK — the prior summary is overwritten. The `analyzer_version` reflects which prompt + model produced the current value.
- Rate-limited to 3 re-runs per session per 24h via SSM-configurable threshold.

## Production pipeline

```
Session reaches COMPLETED or EXPIRED
        │
        ▼
session.ending event handler ────> SQS message to hiringiq-pilot-analyzer-queue
                                          │
                                          ▼
                                  analyzer-service Lambda
                                          │
                                          ├──> fetch session + items + responses + campaign + Persona result + rrweb pointer
                                          ├──> construct Bedrock prompt (v1 template)
                                          ├──> invoke Bedrock (model from SSM)
                                          ├──> Zod-validate response shape (retry once on parse failure)
                                          ├──> programmatically fill timeline gaps with green
                                          └──> PutItem into integrity-summaries table
```

On Bedrock failure: SQS retries 3× with exponential backoff. Final failure routes to DLQ + CloudWatch alarm. The summary is persisted with `status: "analyzer_failed"` and the recruiter dashboard shows a "Request manual re-analysis" button.

## Related entities

- [[Session]] — one summary per terminal session (except DECLINED)
- [[AuditEvent]] — flagged_moments reference event IDs as evidence
- [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] — GSI access path

## Source proposals

- `add-assessment-runtime-integrity` — defining proposal (table, analyzer Lambda, prompt template, structured-output validation)
- `add-recruiter-assessment-review` — consumes the summary in the dashboard; adds re-analysis trigger + rate limit
