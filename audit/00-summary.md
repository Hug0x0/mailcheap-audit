# 00 — Executive summary

**Target:** [marclou/mailcheap](https://github.com/marclou/mailcheap) @ `d4ae7dd` (2026-10-21) · **Audit date:** 2026-10-22 · **Method:** manual static review (5 parallel focus areas) + `npm audit` on the lockfile. Nothing was deployed or executed against a live system.

## Verdict

MailCheap is a **well-engineered codebase with an above-average security posture** for an indie SaaS. The tenant-isolation model (accountId enforced inside Mongoose, one action pipeline shared by dashboard / REST / MCP, a parity test) is strong, and **no cross-account data access was found**. Webhook verification, SSRF protection, HTML sanitization, signed tokens and idempotent sending are all done carefully, and `SECURITY.md` is mostly accurate.

The weaknesses are elsewhere:

1. **Two Medium security issues** reported privately first (details withheld here).
2. **Email-compliance gaps** — opt-outs that can be undone or dropped, a footer that can lose the postal address.
3. **Abuse channels** that could hurt the shared sending reputation (quota-free confirmation emails, long-lived redirect links).
4. **Operations** — no CI, no `vercel.json` for the documented crons, an implicit OpenAI integration, health checks that don't check much.
5. **Scalability** — several full-collection scans and unbounded collections that will hurt as data grows.

## Findings by severity

| Severity | Count | IDs |
|---|---:|---|
| Critical | 0 | — |
| High | 1 | OPS-01 |
| Medium | 21 | AUTHZ-01\*, INJ-01\*, AUTH-01, LOGIC-01…06, OPS-02…04, PERF-01…05, QUAL-01, TEST-01, TEST-02, AUTH-02 (self-host) |
| Low | 25 | AUTHZ-02, AUTH-03…06, INJ-02, INJ-03, LOGIC-07…11, OPS-05, OPS-06, PERF-06…09, QUAL-02…05, TEST-03, TEST-04, DEP-01 |
| Info | 14 | AUTHZ-03…06, AUTH-07…09, INJ-04, INJ-05, LOGIC-12, OPS-07, QUAL-06, DEP-02, DEP-03 |

\* details withheld pending private disclosure — see [DISCLOSURE.md](../DISCLOSURE.md).

## Top 10 to fix first

| # | ID | Why |
|---|---|---|
| 1 | INJ-01 | Unauthenticated DoS of the app (withheld) |
| 2 | AUTHZ-01 | Public key exceeds its documented scope; subscriber privacy & consent (withheld) |
| 3 | LOGIC-03 | Legally required one-click unsubscribes can be silently dropped at scale |
| 4 | LOGIC-01 | Unsubscribed people can be silently re-subscribed by anyone |
| 5 | LOGIC-02 | HTML emails can ship without a postal address (CAN-SPAM) |
| 6 | OPS-01 | Documented crons (incl. list hygiene) never run on Vercel deploys |
| 7 | LOGIC-04 | ~20× plan quota of owner-written email via confirmations |
| 8 | LOGIC-05 | Signed click links usable as a long-lived redirector → domain blocklisting |
| 9 | AUTH-01 | Keys minted by an MCP/OAuth app survive disconnecting it |
| 10 | OPS-02 | `OPENAI_API_KEY` silently enables sending all content to OpenAI |

## Reports

| File | Area |
|---|---|
| [01-inventory.md](01-inventory.md) | Stack, size, history, config |
| [02-dependencies.md](02-dependencies.md) | `npm audit` results |
| [03-authz-tenant-isolation.md](03-authz-tenant-isolation.md) | Authorization & multi-tenancy |
| [04-auth-oauth-webhooks.md](04-auth-oauth-webhooks.md) | Sign-in, API keys, OAuth/MCP, webhooks, jobs |
| [05-input-handling.md](05-input-handling.md) | XSS, injection, SSRF, CSV, headers, uploads |
| [06-business-logic.md](06-business-logic.md) | Sending, billing, compliance, tracking, imports |
| [07-architecture-quality-performance.md](07-architecture-quality-performance.md) | Architecture, performance, ops, tests, frontend |
| [08-recommendations.md](08-recommendations.md) | Prioritised action plan |
