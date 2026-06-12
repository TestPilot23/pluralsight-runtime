# Session

One assessment session per applicant per attempt. Holds the deterministic item ordering, per-item progress, and the runtime state machine.

## Storage

- **Table:** `hiringiq-pilot-sessions`
- **PK:** `session_id` (String, UUID v7)
- No sort key.

Encrypted at rest with the AWS-managed KMS key. PITR enabled. On-demand billing.

## Attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `session_id` | String (UUID v7) | Yes | Matches PK |
| `applicant_id` | String (UUID v7) | Yes | References the platform-shell [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] |
| `campaign_id` | String (UUID v7) | Yes | References the [[../../../AssessmentSystems-POC/openspec/data-items/Campaign]] |
| `org_id` | String (UUID v7) | Yes | Denormalized from applicant; used for tenant isolation in recruiter queries |
| `state` | String | Yes | See state machine below |
| `state_changed_at` | String | Yes | ISO-8601 UTC; updated on every transition |
| `state_change_reason` | String \| null | No | Optional context attached to the most recent transition (e.g., `"ws_disconnect"`) |
| `item_order` | String[] | Yes | Deterministic ordering of item IDs computed once at `session.started` via SHA-256(session_id) Fisher-Yates seed |
| `item_progress` | Object | Yes | Per-item `served_unanswered` \| `served_answered` \| `not_served` taxonomy (D-SM4) |
| `current_item_id` | String \| null | No | The item the applicant is on right now; null between items |
| `current_item_displayed_at` | String \| null | No | ISO-8601 UTC; used for server-side timer enforcement |
| `pause_time_consumed_ms` | Number | Yes | Accumulated pause-time against the per-item 90s budget (D-SM3) |
| `created_at` | String | Yes | ISO-8601 UTC; equals `session.started` time |
| `updated_at` | String | Yes | ISO-8601 UTC |

## State machine

Per D-SM1 / D-SM7. Allowed transitions are enforced by the `services/assessment/src/state-machine.ts` `transition()` function.

```
NOT_STARTED ─(begin)──> IN_PROGRESS ─(last item submitted)──> COMPLETED
                            │ ▲
                            │ │ (reconnect within 30 min)
                            ▼ │
                         PAUSED.<reason>
                            │
                            │ (> 30 min in PAUSED)
                            ▼
                        ABANDONED

NOT_STARTED ─(applicant declines)──> DECLINED
Any non-terminal ─(magic-link JWT exp passes)──> EXPIRED
```

Pause reasons:

- `PAUSED.ws_disconnect` — WebSocket disconnected
- `PAUSED.face_not_detected` — Face-detection escalation hit 3 warnings (D-SM2)

Terminal states (no outgoing transitions): `COMPLETED`, `ABANDONED`, `DECLINED`, `EXPIRED`.

## Invariants

- `item_order.length === campaign.assessment_question_count` at `IN_PROGRESS` entry.
- Two sessions with the same `session_id` produce identical orderings (Fisher-Yates is seeded by SHA-256 of `session_id`).
- Item is "seen" if its progress is `served_unanswered` or `served_answered`. The "seen" taxonomy preserves item exposure even when the applicant closes the window mid-item.
- The platform-shell [[../../../AssessmentSystems-POC/openspec/data-items/Item]]'s `answer` field never enters the session record. Grading writes only the candidate's answer to `item_progress`.
- Pause-time budget is **per-item**, not session-wide. Crossing 90s in one item progresses the timer through the pause.

## Triggers (post-session)

- On transition to `COMPLETED` or `EXPIRED`: enqueue an SQS message to `hiringiq-pilot-analyzer-queue`. The analyzer-service produces an [[IntegritySummary]].
- On transition to `DECLINED`: **no analyzer message** (D-AL4). Declined sessions are intentionally not analyzed.

## Related entities

- [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] — one applicant ↔ one session (per attempt)
- [[../../../AssessmentSystems-POC/openspec/data-items/Campaign]] — source of `item_order` candidates
- [[AuditEvent]] — every state transition emits a `session.*` event
- [[WSConnection]] — pause-on-disconnect is driven by WS lifecycle
- [[IntegritySummary]] — produced by the analyzer after a terminal `COMPLETED`/`EXPIRED`
- [[MagicLinkPayload]] — the JWT carries `aid` and `cid` that resolve to this session at landing time

## Source proposals

- `add-assessment-runtime-foundation` — defining proposal (state machine, item ordering, server-side grading)
- `add-assessment-runtime-integrity` — adds the `PAUSED.face_not_detected` reason
