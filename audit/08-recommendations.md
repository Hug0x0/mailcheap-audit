# 08 — Prioritised action plan

Effort: **S** < ½ day · **M** ½–2 days · **L** > 2 days.

## Now (security & legal)

| Item | Findings | Effort |
|---|---|---|
| Fix the two withheld issues once privately reported | INJ-01, AUTHZ-01 | S |
| Exempt valid signed tokens from the per-IP limit on one-click unsubscribe | LOGIC-03 | S |
| Route the website signup form through `addSignup({ confirmReturning: true })` | LOGIC-01 | S |
| Always render the postal address; only skip the unsubscribe footer when the placeholder is in a real `href` | LOGIC-02 | S |
| Let `pending` contacts unsubscribe | LOGIC-10 | S |
| Bump `sanitize-html` to ≥ 2.18.1 | DEP-01 | S |

## Next (abuse & trust)

| Item | Findings | Effort |
|---|---|---|
| Count API/MCP-originated confirmation emails against the plan; apply `getDropReason` to API signups | LOGIC-04 | M |
| Don't track test emails; interstitial instead of redirect when the message is gone; expiring link signatures | LOGIC-05 | M |
| Record key creator; revoke an OAuth app's keys on disconnect (or forbid `create_api_key` over OAuth) | AUTH-01 | M |
| Trusted-proxy aware client IP (`TRUSTED_PROXY_HOPS`) | AUTH-02, INJ-02, OPS-05 | S |
| SNS: timestamp window, `MessageId` dedup, SignatureVersion 2 only | AUTH-03 | S |
| QStash: explicit local-bypass flag, verify `sub` against URL | AUTH-04 | S |
| Allowlist-rebuild DCR bodies; test that other registration paths 404 | AUTH-05 | S |
| Per-email OTP failure cap | AUTH-06 | S |
| Gate plan upgrades on paid invoice; check Stripe portal proration settings | LOGIC-06 | M |
| Restrict the done-page "Back to" link to allowed domains | INJ-03 | S |

## Operations

| Item | Findings | Effort |
|---|---|---|
| Commit `vercel.json` with the 3 crons; document list-hygiene | OPS-01 | S |
| Make the OpenAI content check opt-in and documented | OPS-02 | S |
| GitHub Actions: `mongod` service, `npm test`, `npm run lint`, `npm run build`, `npm audit --omit=dev`; Dependabot | TEST-01, DEP-* | S |
| Stronger config checks + deep health check | OPS-03 | S |
| `onRequestError` + optional Sentry/OTel; stop logging whole error objects | OPS-04 | M |
| Disable `autoIndex` in prod, index sync script | OPS-06 | M |
| Dockerfile / compose for self-hosting | OPS-07 | M |

## Scale

| Item | Findings | Effort |
|---|---|---|
| Store and index `emailDomain`; bound list-hygiene restores | PERF-01 | M |
| TTL / drop `Event`; roll up and archive `Message` | PERF-04 | M |
| Index the audience sort; drop the suppression `$nin` | PERF-05 | M |
| `{sentAt:1}` index or daily counters for admin stats | PERF-02 | S |
| Cache contacts-page counts | PERF-03 | M |
| Batch-level status check and `$inc` in the send loop | PERF-06 | M |
| Per-newsletter `revalidateTag` | PERF-07 | S |
| CSV parse in a worker | PERF-08 | S |

## Maintainability

| Item | Findings | Effort |
|---|---|---|
| Split `lib/sending.js` | QUAL-01 | L |
| Deduplicate `escapeHtml` / `escapeRegex` | QUAL-02 | S |
| Validate action outputs in tests | QUAL-03 | S |
| Route tests for SES webhook and untested jobs; coverage reporting; small Playwright suite | TEST-02, TEST-03 | M |
| Harden scoping plugin against `$lookup`/`$unionWith`; extend parity test to `app/welcome`; update SECURITY.md lists | AUTHZ-02…04 | S |
| Purge `Segment` with the account; keep `List-Unsubscribe` on the app domain | LOGIC-11 | S |
