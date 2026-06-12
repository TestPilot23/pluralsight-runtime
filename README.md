# HiringIQ Assessment Runtime

The applicant-facing proctored assessment system for HiringIQ. Sibling workspace
to the recruiter-facing platform shell at
[`../PluralSight-POC/`](../PluralSight-POC/).

## What lives here

- **Magic-link entry + consent landing + Persona ID verification flow**
- **Device check + assessment intro screen + assessment interface**
- **Item rendering** (mc_single, mc_multiple, short_answer; extensible)
- **WebSocket telemetry pipeline** (event capture, audit log)
- **Browser instrumentation** (face-api.js, VAD, paste/copy, tab visibility,
  screen-sharing signals, screenshot detection)
- **rrweb session recording**
- **Biometric storage** (Persona selfie → face-api.js enrollment, embeddings)
- **Post-session integrity analyzer** (Bedrock LLM → IntegritySummary)
- **Completion screen**

## What does NOT live here

- Recruiter authentication, campaign creation, applicant invitation flows — those
  live in the platform shell (`../PluralSight-POC/`).
- Item bank content — also platform shell (proposal `add-assessment-item-bank`).
- Recruiter dashboard chrome — also platform shell, but the runtime workspace
  contributes the IntegritySummary + rrweb + timeline UI components via Phase 3.

## Architecture

Four logical microservices, all on AWS Lambda at pilot scale (locked D-SVC2):

| Service | Surface | Owns |
|---|---|---|
| `applicant-service` | HTTPS via API Gateway | Applicant profile, Persona integration, biometric storage |
| `assessment-service` | HTTPS via API Gateway | Session state machine, item delivery, grading |
| `telemetry-service` | WebSocket via API Gateway + HTTPS for rrweb batch upload | WS event ingestion, audit log writes, rrweb upload |
| `analyzer-service` | SQS-triggered Lambda (no HTTP surface) | Post-session Bedrock analysis, IntegritySummary production |

Promote individual services from Lambda to Fargate when their CloudWatch metrics
exceed the thresholds documented in
[`thoughts/assessment-design-decisions.md`](../PluralSight-POC/thoughts/assessment-design-decisions.md)
§ D-SVC2.

## Proposal apply order

1. `add-assessment-runtime-foundation` — baseline assessment delivery
2. `add-assessment-runtime-integrity` — anti-cheat instrumentation + analyzer
3. `add-recruiter-assessment-review` — recruiter dashboard additions for review

See [`openspec/changes/`](./openspec/changes/) for in-flight proposals.

## Design history

The design was hammered out across the conversation captured in:
- [`../PluralSight-POC/thoughts/assessment-design-decisions.md`](../PluralSight-POC/thoughts/assessment-design-decisions.md) — locked decisions (A–M, 60+ entries)
- [`../PluralSight-POC/thoughts/open-questions.md`](../PluralSight-POC/thoughts/open-questions.md) — outstanding action items

Every proposal in this workspace references decisions by ID (e.g., D-P3, D-SM2)
so the design rationale is traceable without re-litigating.

## Workspace conventions

See [`openspec/config.yaml`](./openspec/config.yaml) for the full tech-stack
lock + rules. Key conventions:

- Node.js 26, Terraform, AWS Lambda
- Material Design 3 over Pluralsight brand tokens
- React 18+ on a separate CloudFront distribution from the recruiter platform
- 180-day biometric retention (D-P5)
- Conventional commits, feature branches, main = released to prod
- 600-line file ceiling
