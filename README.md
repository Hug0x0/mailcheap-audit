# MailCheap — Independent Audit

Independent code audit of [marclou/mailcheap](https://github.com/marclou/mailcheap), an open-source (AGPL-3.0) newsletter platform built on Next.js 16, MongoDB/Mongoose, Better Auth, AWS SES / PostShiba, Stripe and Upstash.

| | |
|---|---|
| Audited commit | [`d4ae7dd`](https://github.com/marclou/mailcheap/commit/d4ae7dd7256b8162c116441e35f35dbfa960a5e7) (2026-10-21) |
| Source | pinned as a git submodule in [`src/`](src) |
| Method | manual static review of the source, assisted by tooling (`npm audit`, grep-based sweeps). No live system was tested. |

## Reports

Start with the **[executive summary](audit/00-summary.md)**.

| Report | Area |
|---|---|
| [00-summary](audit/00-summary.md) | Verdict, severity breakdown, top 10 |
| [01-inventory](audit/01-inventory.md) | Stack, size, history, configuration |
| [02-dependencies](audit/02-dependencies.md) | `npm audit` on the lockfile |
| [03-authz-tenant-isolation](audit/03-authz-tenant-isolation.md) | Authorization & multi-tenancy |
| [04-auth-oauth-webhooks](audit/04-auth-oauth-webhooks.md) | Sign-in, API keys, OAuth/MCP, webhooks, jobs |
| [05-input-handling](audit/05-input-handling.md) | XSS, injection, SSRF, CSV, uploads |
| [06-business-logic](audit/06-business-logic.md) | Sending, billing, compliance, tracking |
| [07-architecture-quality-performance](audit/07-architecture-quality-performance.md) | Architecture, perf, ops, tests, frontend |
| [08-recommendations](audit/08-recommendations.md) | Prioritised action plan |

**Headline:** 0 critical, 1 high (ops), 21 medium, 25 low, 14 info. No cross-account data access found. Two medium security findings are withheld pending private disclosure — see [DISCLOSURE.md](DISCLOSURE.md).

## Disclaimer

This is an independent, best-effort static review. It is not affiliated with MailCheap or its author. Findings are not guarantees of exploitability, and the absence of a finding does not mean the absence of a bug. Any suspected security vulnerability with real impact should be reported privately through the project's [security policy](https://github.com/marclou/mailcheap/security), not in a public issue.
