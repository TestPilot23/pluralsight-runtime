# MagicLinkPayload

The JWT payload carried in the URL token of every applicant invitation. Not persisted as a record — it's a signed token whose validity is verified at every applicant-facing entry point.

## Storage

**Not persisted.** Signed JWT. The signing secret lives in AWS Secrets Manager at `hiringiq/<env>/applicant-magic-link-secret` (128-char hex, 64-byte random_password). Created by the platform shell; **shared with the runtime** — no separate secret is provisioned (D-F6).

## Claims

| Claim | Type | Notes |
|---|---|---|
| `iss` | String | `"hiringiq-pilot"` |
| `aud` | String | `"applicant-assessment"` |
| `iat` | Number | Unix-seconds at issue |
| `exp` | Number | Unix-seconds at expiry; `min(iat + 15 days, campaign.end_at)` per D-F6 |
| `aid` | String (UUID v7) | Applicant ID — references [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] |
| `cid` | String (UUID v7) | Campaign ID — references [[../../../AssessmentSystems-POC/openspec/data-items/Campaign]] |
| `oid` | String (UUID v7) | Organization ID — references [[../../../AssessmentSystems-POC/openspec/data-items/Organization]] |

## Signing

- **Algorithm:** HS256 (HMAC-SHA256)
- **Secret:** 64-byte random hex from Secrets Manager
- **Library:** `jose`

## Usage

The signed JWT appears in three places:

1. **Invitation email CTA** — `https://<runtime-cloudfront>/?token=<jwt>`
2. **HTTPS authorization** — `GET /assess?token=<jwt>` returns the landing data; `POST /assess/begin` transitions the applicant to In-Progress
3. **WebSocket handshake** — `wss://<runtime-ws>/?token=<jwt>` authenticates the [[WSConnection]] at `$connect`

The same JWT is reused for the full assessment lifetime — there's no token refresh. The token is short-lived enough (15 days max, often shorter) that a reissue cycle isn't worth the complexity.

## Invariants

- `exp` is **never** more than 15 days after `iat`, even if the campaign runs longer. D-F6 caps the lifetime to bound the blast radius of a leaked link.
- Magic links are **not individually revocable**. The only revocation path is rotating the signing secret, which invalidates every live link in the env. Routine rotation is not done — only on suspected compromise.
- Tampered tokens (wrong signature) and stale tokens (`exp` passed) are both rejected with `401 INVALID_MAGIC_LINK`. The 401 response is intentionally indistinguishable between the two cases — no information leakage about whether a given link was once valid.
- The runtime never persists the raw JWT. It verifies, extracts the three IDs, and discards the token.

## Helpers

Exported from `packages/auth/src/magic-link.ts` (platform shell) and mirrored in `packages/shared/src/magic-link.ts` (runtime — D-F6 keeps the implementations in sync):

- `generateMagicLink({ applicantId, campaignId, orgId, exp, secret })` — returns the JWT string
- `verifyMagicLink(token, secret)` — returns the parsed payload or throws `MagicLinkVerificationError`

## Related entities

- [[../../../AssessmentSystems-POC/openspec/data-items/Applicant]] — `aid` claim
- [[../../../AssessmentSystems-POC/openspec/data-items/Campaign]] — `cid` claim
- [[../../../AssessmentSystems-POC/openspec/data-items/Organization]] — `oid` claim
- [[Session]] — created by the assessment-begin handler after token verification
- [[WSConnection]] — `$connect` Lambda verifies the token

## Source proposals

- Platform shell `add-applicant-assessment-landing` — defining proposal (token generation + verification + public routes)
- Runtime `add-assessment-runtime-foundation` — adds the 15-day cap (D-F6); reuses the platform shell's secret
