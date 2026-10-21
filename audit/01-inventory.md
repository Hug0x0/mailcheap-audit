# 01 — Inventory & project overview

Audited commit: `d4ae7dd7256b8162c116441e35f35dbfa960a5e7` (2026-10-21).

## Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16.4.1 (App Router, server actions), React 19.2.9 |
| Runtime | Node.js >= 22, ESM (`"type": "module"`), plain JavaScript (no TypeScript) |
| Database | MongoDB via Mongoose 9 |
| Auth | Better Auth 1.7 (Google sign-in, sessions in DB), `@better-auth/oauth-provider`, `@better-auth/mcp`, `@better-auth/cimd` |
| Email | AWS SES v2 or PostShiba (pluggable `lib/email-provider/`), Resend for platform email |
| Payments | Stripe 23 (only on MailCheap Cloud) |
| Queue / rate limit | Upstash QStash, Upstash Ratelimit + Redis |
| Storage | Vercel Blob or S3 + CloudFront |
| Editor | Tiptap 3 |
| Validation | Zod 4 |
| AI integration | MCP server (`@modelcontextprotocol/server`), OAuth for ChatGPT / Claude |

## Size

| Directory | JS files | Lines |
|---|---:|---:|
| `app/` | 135 | 9,372 |
| `lib/` | 239 | 26,692 |
| `components/` | 213 | 19,611 |
| `models/` | 22 | 987 |
| `tests/` | 161 | 27,478 |

~57k lines of application code, ~27k lines of tests (test/code ratio ≈ 0.48).

## History

- 50 commits, single author, first commit 2026-10-19 — the public history is very young (likely squashed/imported from a private repo).
- No `.github/` directory: **no CI pipeline** is defined in the repository (tests and lint are not enforced on push/PR).

## Security posture documentation

The repository ships an unusually detailed [`SECURITY.md`](https://github.com/marclou/mailcheap/blob/d4ae7dd/SECURITY.md) describing the tenant-isolation model, API key design, OAuth/MCP rules, SSRF protections and webhook trust. The audit uses it as the specification and checks the code against it.

## Configuration

`.env.example` documents ~50 variables (MongoDB, Better Auth secret, suppression hash key, Google OAuth, Stripe, SES/PostShiba, QStash, Upstash, storage, Telegram alerts…). Self-hosting is supported; `MAILCHEAP_CLOUD=true` switches on the hosted-only features (billing, etc.).
