# Runtime data items

One markdown file per persisted entity owned by this workspace. Each file describes:

- **Storage** — table or bucket, partition key, sort key.
- **Attributes** — the full attribute table, annotated with the proposal that introduced each field.
- **Lifecycle** — when records are created, transitioned, and cleaned up.
- **Source proposals** — the OpenSpec changes that define or modify the shape.

## Entity index

| Entity | Storage | Owns / extends |
|---|---|---|
| [Session](./Session.md) | `hiringiq-pilot-sessions` table | Per-applicant assessment session + state machine |
| [AuditEvent](./AuditEvent.md) | `hiringiq-pilot-audit-events` table + S3 archive | Telemetry + integrity events |
| [WSConnection](./WSConnection.md) | `hiringiq-pilot-ws-connections` table | Live WebSocket connection tracking |
| [IntegritySummary](./IntegritySummary.md) | `hiringiq-pilot-integrity-summaries` table | Post-session Bedrock analyzer output |
| [MagicLinkPayload](./MagicLinkPayload.md) | Not persisted — JWT shape only | Applicant entry token |
| [BiometricArtifacts](./BiometricArtifacts.md) | S3 + DDB attribute on Applicant | Persona selfie image + face-api.js embedding + rrweb recordings |

## Entities owned by the platform shell

`Organization`, `User`, `Campaign`, `Skill`, `AssessmentQuestion`, `Applicant`, `Item`, `ItemSkillIndex` live in the platform shell. See `../../../AssessmentSystems-POC/openspec/data-items/`.

The runtime **reads** Campaign, Applicant, and Item records; it **writes** biometric attributes onto Applicant records via the `applicant-service`. All other entities listed above are owned end-to-end by this workspace.
