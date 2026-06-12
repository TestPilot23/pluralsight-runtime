# Recruiter Access Logging

## ADDED Requirements

### Requirement: Backend emits recruiter_access.* events on every dashboard data view (D-AL7, D-R5)

The runtime workspace's backend endpoints SHALL emit `recruiter_access.*` audit events as their FIRST action — before returning any data. The emission happens server-side, not from React component-level lifecycle hooks (D-R5 — frontend emission is bypassable).

Endpoints + event mappings:
| Endpoint | Emitted event_type |
|---|---|
| `GET /api/sessions/<id>` | `recruiter_access.viewed_session` |
| `GET /api/sessions/<id>/rrweb` | `recruiter_access.viewed_replay` |
| `GET /api/sessions/<id>/audit-events` (page 1 only) | `recruiter_access.viewed_audit_log` |
| `GET /api/applicants/<id>/persona-selfie` | `recruiter_access.viewed_persona_selfie` |
| `GET /api/applicants/<id>/face-embedding` (admin/debug only) | `recruiter_access.viewed_embedding` |

#### Scenario: Session view emits event before data return
- **WHEN** a recruiter calls `GET /api/sessions/<id>`
- **THEN** an audit event `recruiter_access.viewed_session` appears in `hiringiq-pilot-audit-events` BEFORE the HTTP response is returned to the client
- **AND** if the audit-log write fails, the HTTP endpoint returns 500 (the data view does NOT succeed unless the access is logged)

### Requirement: Event envelope matches D-AL7 schema

Each `recruiter_access.*` event SHALL conform to the D-AL7 payload shape:

```ts
{
  event_id: string,                  // UUID v7
  schema_version: number,            // 1
  session_id?: string,               // present when the access is session-scoped
  applicant_id: string,              // always — every access is per-applicant
  campaign_id?: string,
  org_id: string,                    // recruiter's org_id from Cognito JWT
  timestamp_server: string,
  timestamp_client?: string,         // absent — emitted server-side, no client clock
  ip: string,                        // recruiter's source IP from the API Gateway request
  user_agent: string,                // recruiter's user agent
  category: "recruiter-access",
  event_type: "recruiter_access.viewed_session" | ...,
  payload: {
    recruiter_user_id: string,       // from Cognito JWT sub claim
    recruiter_email: string,         // denormalized from Cognito JWT email claim
    accessed_resource: {
      type: "session" | "rrweb_recording" | "audit_log" | "selfie" | "embedding",
      id: string,
      applicant_id: string,
      org_id: string,
    },
  },
}
```

#### Scenario: Event envelope contains all required fields
- **WHEN** a recruiter views a session and the `recruiter_access.viewed_session` event is emitted
- **THEN** the persisted event in the audit-events table contains every documented field
- **AND** the `category` is `"recruiter-access"` (distinguishing it from the applicant-side `integrity-signal` / `session-lifecycle` / `identity` / `assessment` events)

### Requirement: Org-isolation enforced before event emission

The backend SHALL validate that the recruiter's `org_id` (from the Cognito JWT) matches the resource's `org_id` BEFORE emitting the access event. On mismatch, the backend returns 403 and emits a SECURITY audit event `recruiter_access.cross_org_attempt` instead of the resource-specific event.

#### Scenario: Same-org access emits viewed event
- **WHEN** a recruiter in org O1 calls `GET /api/sessions/<id>` for a session in org O1
- **THEN** `recruiter_access.viewed_session` is emitted with the documented payload
- **AND** the response is 200 with the session data

#### Scenario: Cross-org access emits security event + 403
- **WHEN** a recruiter in org O1 calls `GET /api/sessions/<id>` for a session in org O2
- **THEN** `recruiter_access.cross_org_attempt` is emitted with `{ recruiter_org_id: "O1", resource_org_id: "O2", endpoint: "GET /api/sessions/<id>" }`
- **AND** the HTTP response is 403 Forbidden
- **AND** no session data is leaked in the response body

### Requirement: 180-day retention on access logs (D-AL7)

The system SHALL retain recruiter access logs for 180 days (matching the biometric retention from D-P5). The retention is enforced via the same DDB TTL + S3 lifecycle that applies to other audit-log events.

When biometric data is deleted via right-to-deletion, the access logs that REFERENCE the deleted data SHALL be retained for the full 180-day window — they're operational records, not biometric data themselves. This preserves the audit trail of who accessed biometric data while the data still existed.

#### Scenario: Access log retained past biometric deletion
- **WHEN** a recruiter views applicant A1's selfie at T=0, emitting `recruiter_access.viewed_persona_selfie`
- **AND** at T=30 days, applicant A1's biometric data is deleted via right-to-deletion
- **THEN** the `recruiter_access.viewed_persona_selfie` event from T=0 remains in the audit log
- **AND** is queryable until T=180 days
- **AND** the audit log shows the access happened while the selfie still existed

### Requirement: Access logs queryable from the dashboard (future-facing — not in this proposal's UI scope)

The system SHALL store recruiter access events in a way that supports future querying by the dashboard (e.g., "show me everyone in our org who has viewed applicant A1's data"). At MVP, no UI for this exists; the data is queryable via the audit-events table directly.

The GSI on `applicant_id` (created in Phase 1) supports this query pattern.

#### Scenario: Query by applicant_id returns all access events
- **WHEN** an admin queries the audit-events table via the GSI on `applicant_id` for applicant A1
- **THEN** all `recruiter_access.*` events for A1 are returned, including which recruiter viewed what and when
