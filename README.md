# MailCheap — Independent Audit

Independent code audit of [marclou/mailcheap](https://github.com/marclou/mailcheap), an open-source (AGPL-3.0) newsletter platform built on Next.js 16, MongoDB/Mongoose, Better Auth, AWS SES / PostShiba, Stripe and Upstash.

| | |
|---|---|
| Audited commit | [`d4ae7dd`](https://github.com/marclou/mailcheap/commit/d4ae7dd7256b8162c116441e35f35dbfa960a5e7) (2026-10-21) |
| Source | pinned as a git submodule in [`src/`](src) |
| Method | manual static review of the source, assisted by tooling (`npm audit`, grep-based sweeps). No live system was tested. |

## Reports

See [`audit/`](audit) — one file per area. Start with [`audit/00-summary.md`](audit/00-summary.md).

## Disclaimer

This is an independent, best-effort static review. It is not affiliated with MailCheap or its author. Findings are not guarantees of exploitability, and the absence of a finding does not mean the absence of a bug. Any suspected security vulnerability with real impact should be reported privately through the project's [security policy](https://github.com/marclou/mailcheap/security), not in a public issue.
