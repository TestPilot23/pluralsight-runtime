# WSConnection

Tracks live WebSocket connections from applicant browsers to the telemetry-service. Lets the runtime drive pause-on-disconnect and resume-on-reconnect without polling.

## Storage

- **Table:** `hiringiq-pilot-ws-connections`
- **PK:** `connection_id` (String, assigned by API Gateway)
- No sort key.

Encrypted at rest with the AWS-managed KMS key. On-demand billing. **PITR not required** — this table is ephemeral by design; rows live ~10–60 minutes per session.

## Attributes

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `connection_id` | String | Yes | API Gateway WebSocket connection ID; matches PK |
| `session_id` | String (UUID v7) | Yes | References [[Session]] |
| `applicant_id` | String (UUID v7) | Yes | Denormalized for fast lookup |
| `org_id` | String (UUID v7) | Yes | Denormalized for tenant filtering |
| `connected_at` | String | Yes | ISO-8601 UTC; set on `$connect` |
| `last_heartbeat_at` | String | Yes | ISO-8601 UTC; updated on every received message |
| `source_ip` | String | Yes | Connection origin IP at `$connect` |

## Lifecycle

```
$connect ────> row written
              │
              │ (WS messages received) ────> last_heartbeat_at updated
              │
              ▼
$disconnect ──> row deleted
              │
              └──> Session transitions to PAUSED.ws_disconnect
                   EventBridge schedules ABANDONED transition for now + 30 min
```

On reconnect within 30 minutes:

```
$connect ────> new row written (new connection_id)
              │
              ▼
              Session transitions back to IN_PROGRESS
              EventBridge cancels the ABANDONED schedule
```

## Invariants

- A session may have **at most one** live `WSConnection` at a time. A second `$connect` for the same `session_id` first deletes any prior row + transitions the session through the disconnect/reconnect cycle.
- The `$connect` handler authenticates via the magic-link JWT in the connection's query string — see [[MagicLinkPayload]].
- API Gateway has a hard 2-hour connection lifetime. Heartbeats DO NOT extend it — assessments shorter than 2 hours never hit this; longer assessments rely on natural reconnects.

## Access patterns

- **Pause-on-disconnect:** the `$disconnect` Lambda looks up the row by `connection_id`, transitions the session, and deletes the row.
- **Reconnect detection:** the `$connect` Lambda queries by `session_id` (via in-memory cache or a small auxiliary index) to detect orphaned rows.
- **Cleanup:** rows older than 4 hours are reaped by a scheduled Lambda (insurance against API Gateway lifecycle bugs).

## Related entities

- [[Session]] — the session this connection serves
- [[AuditEvent]] — `$connect` / `$disconnect` emit `session.paused.ws_disconnect` and `session.resumed` events
- [[MagicLinkPayload]] — token used to authenticate the `$connect` handshake

## Source proposals

- `add-assessment-runtime-foundation` — defining proposal (table, `$connect`/`$disconnect` handlers, pause/resume logic)
