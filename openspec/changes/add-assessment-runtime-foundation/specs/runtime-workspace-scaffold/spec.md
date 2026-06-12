# Runtime Workspace Scaffold

## ADDED Requirements

### Requirement: Sibling AWS workspace with independent Terraform state

The system SHALL provision the assessment runtime as a sibling AWS workspace separate from the recruiter-facing platform shell. The runtime workspace SHALL use:
- A NEW Terraform state file (`pluralsight-poc-runtime-pilot.tfstate`) in the SHARED state S3 bucket created by the platform shell's `add-aws-infrastructure-baseline` (proposal #1)
- The SAME awscli profile `kms-pluralsight` and the SAME AWS account
- The SAME region `us-east-1`
- An INDEPENDENT CloudFront distribution from the recruiter platform (different origin → different CSP → different cookie boundary)

#### Scenario: Bootstrap reuses shared state bucket
- **WHEN** `terraform init` runs in `infra/terraform/envs/pilot/`
- **THEN** the backend configuration points to the shared platform-shell state bucket
- **AND** the state-file key is `pluralsight-poc-runtime-pilot.tfstate` (distinct from the platform-shell's `pluralsight-poc-pilot.tfstate`)

#### Scenario: Apply does not modify platform-shell resources
- **WHEN** `terraform apply` runs in the runtime workspace
- **THEN** no resources owned by the platform shell are modified or destroyed
- **AND** the apply plan shows only runtime-workspace-specific resources

### Requirement: Four Lambda services deployed independently

The system SHALL deploy four logical microservices, each as its own Lambda function with its own IAM execution role, log group, and deployment artifact:
- `hiringiq-pilot-applicant` — applicant-service
- `hiringiq-pilot-assessment` — assessment-service
- `hiringiq-pilot-telemetry` — telemetry-service
- `hiringiq-pilot-analyzer` — analyzer-service (Phase 1: stub handler; Phase 2: SQS-triggered post-session analysis)

Each Lambda SHALL run Node.js 26 on the ARM64 architecture.

#### Scenario: Each service is a separately deployable artifact
- **WHEN** the four service Lambdas are inspected after apply
- **THEN** each has its own ARN, its own log group at `/aws/lambda/<function-name>`, and its own IAM execution role with the documented permissions
- **AND** the four code modules under `services/<name>/` build independently via `npm run -w services/<name> build`

### Requirement: Monorepo structure enforces module boundaries

The system SHALL organize the runtime workspace as an npm-workspaces monorepo with:
- `apps/runtime-web/` — React 18 + Vite + TypeScript + MUI v6 application (the applicant-facing client)
- `services/<name>/` — four Lambda code modules
- `packages/shared/` — cross-service TypeScript types + utilities
- `infra/terraform/` — Terraform modules + the pilot env composition

Each service module SHALL expose its public API exclusively through `services/<name>/src/public.ts`. ESLint rules SHALL block cross-module imports that bypass `public.ts`.

#### Scenario: Cross-module deep import is rejected by lint
- **WHEN** a file in `services/applicant/` attempts to import from `services/assessment/src/state-machine.ts` (a path bypassing `public.ts`)
- **THEN** `npm run lint` returns a non-zero exit code with the `no-restricted-imports` error citing the cross-module rule
- **AND** the legal import path `services/assessment/src/public.ts` (or `@hiringiq/assessment` if package-named) does NOT trigger the rule

### Requirement: Runtime DynamoDB tables provisioned with full encryption + PITR

The system SHALL provision three new DynamoDB tables:
- `hiringiq-pilot-sessions` — PK=`session_id` (String). Holds session state, applicant_id, campaign_id, item-order, seen-state, timer state.
- `hiringiq-pilot-ws-connections` — PK=`connection_id` (String). Maps API Gateway WebSocket connection IDs to session_ids.
- `hiringiq-pilot-audit-events` — PK=`SESSION#<session_id>` (String), SK=`<timestamp_server>#<event_id>` (String). GSI on `applicant_id` for cross-session queries.

Every table SHALL use on-demand billing, PITR enabled, and server-side encryption with the AWS-managed DynamoDB KMS key.

#### Scenario: Tables exist with documented schema
- **WHEN** `aws dynamodb describe-table` runs against each of the three table names
- **THEN** the schema matches the documented PK/SK; PITR is enabled; SSE is configured with the AWS-managed KMS key; billing mode is PAY_PER_REQUEST

### Requirement: Analyzer SQS queue provisioned at Phase 1 (handler is stub until Phase 2)

The system SHALL provision an SQS queue named `hiringiq-pilot-analyzer-queue` at Phase 1, paired with the `hiringiq-pilot-analyzer` Lambda as its event-source trigger. At Phase 1 the Lambda handler is a no-op stub that logs the invocation and returns; Phase 2 replaces the handler with the real Bedrock-driven analyzer.

A Dead-Letter Queue `hiringiq-pilot-analyzer-dlq` SHALL be provisioned alongside, configured as the SQS redrive target after 3 receive failures. CloudWatch alarm on DLQ depth > 0 routes to `hiringiq-pilot-runtime-alerts`.

Provisioning the queue at Phase 1 (even though it's unused) means Phase 2's PR scope is only the Lambda code swap, not new infrastructure.

#### Scenario: SQS queue and DLQ exist with documented configuration
- **WHEN** `aws sqs get-queue-attributes` is called on the analyzer queue ARN
- **THEN** the queue exists with the DLQ configured as redrive target with maxReceiveCount = 3
- **AND** the Lambda function `hiringiq-pilot-analyzer` is wired as the queue's event source

#### Scenario: Phase 1 stub returns without writing IntegritySummary
- **WHEN** a test SQS message is sent to the analyzer queue at Phase 1
- **THEN** the analyzer Lambda is invoked, logs the receipt, and returns success
- **AND** no IntegritySummary record is written (the `hiringiq-pilot-integrity-summaries` table doesn't exist until Phase 2)

### Requirement: Cross-workspace IAM for item-bank reads

The system SHALL grant the `assessment-service` Lambda IAM permission to read from the platform-shell's item-bank tables:
- `dynamodb:BatchGetItem` on `hiringiq-pilot-items`
- `dynamodb:Query` on `hiringiq-pilot-item-skill-index`

The Lambda SHALL NOT have write permissions on these cross-workspace tables. Item data flows only one direction (read).

#### Scenario: IAM policy is scoped to read-only on item-bank tables
- **WHEN** the assessment-service Lambda's IAM policy is rendered
- **THEN** it contains `BatchGetItem` and `Query` on the documented item-bank table ARNs and contains no Write/Update/Delete actions on those ARNs

### Requirement: Magic-link JWT signing secret is shared with the platform-shell (D-F6)

The system SHALL NOT provision a new Secrets Manager secret for magic-link signing. Per Path Z reconciliation (D-F6), the runtime workspace SHARES the platform-shell's EXISTING secret at `hiringiq/<env>/applicant-magic-link-secret` (provisioned by platform-shell proposal #11 — `add-applicant-assessment-landing`).

The system SHALL grant the runtime's `assessment-service` and `applicant-service` Lambda execution roles `secretsmanager:GetSecretValue` permission on the platform-shell's secret ARN. The ARN is read at Terraform apply-time via a `data "aws_secretsmanager_secret"` source lookup by name.

The `telemetry-service` Lambda also requires read access on this secret (for `$connect` JWT validation when the client opens the WebSocket).

#### Scenario: Runtime Lambdas read the shared platform-shell secret
- **WHEN** the runtime `assessment-service` Lambda calls `GetSecretValue` against the platform-shell's secret ARN
- **THEN** the call succeeds and returns the 64-byte hex signing key
- **WHEN** the runtime `telemetry-service` Lambda makes the same call (for `$connect` JWT validation)
- **THEN** the call also succeeds

#### Scenario: No new runtime-owned magic-link secret exists
- **WHEN** Terraform plan is rendered for the runtime workspace
- **THEN** no `aws_secretsmanager_secret` resource is created with a magic-link-related name
- **AND** the only new secret introduced by the runtime workspace is unrelated to magic-link signing

### Requirement: SSM parameter namespace for runtime config

The system SHALL provision an SSM Parameter Store namespace `/hiringiq/pilot/runtime/*` with initial parameters:
- `usePersonaVerification` — boolean, default `true` (controls the Persona feature flag per D-P1)

Additional parameters are added by Phase 2 (`/hiringiq/pilot/runtime/clock-skew/*`, `/hiringiq/pilot/runtime/face-detection/*`, etc.) and are not in scope here.

#### Scenario: SSM parameter is readable by authorized Lambdas
- **WHEN** `aws ssm get-parameter --name /hiringiq/pilot/runtime/use-persona-verification` runs with the runtime Lambda's IAM
- **THEN** the parameter exists with value `"true"` and is readable
- **WHEN** the platform-shell's API Lambda attempts the same call
- **THEN** the call is permitted (SSM parameters are not secrets and don't require Lambda-specific IAM tightening at this layer)

### Requirement: Observability baseline matches platform-shell pattern

The system SHALL provision CloudWatch log groups for each of the four Lambdas at `/aws/lambda/<function-name>`. CloudWatch alarms SHALL fire on:
- Each Lambda's `Errors` metric > 0 over a 5-minute window
- API Gateway HTTP API 5xx rate > 5% over a 5-minute window
- API Gateway WebSocket API 5xx rate > 5% over a 5-minute window

All alarms route to a NEW SNS topic `hiringiq-pilot-runtime-alerts` (distinct from the platform-shell's alarm topic for clean separation). The topic exists with no subscriptions until the runtime-specific alarm distribution email is confirmed.

#### Scenario: Alarms fire on a deliberate Lambda error
- **WHEN** the applicant-service Lambda is invoked with a payload that causes a deliberate uncaught exception
- **THEN** the corresponding CloudWatch alarm transitions to ALARM state within 5-6 minutes
- **AND** the SNS topic shows a pending notification visible in the CloudWatch console even without an active subscription
