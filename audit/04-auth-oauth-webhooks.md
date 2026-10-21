# 04 — Authentication, API keys, OAuth/MCP, webhooks & jobs

Static review, read-only. `node_modules` was not installed, so library internals (Better Auth `oauth-provider` / `mcp` / `cimd`, `@upstash/qstash`, MCP SDK) were not read; where a control depends on library behaviour, the finding says so and severity is lowered.

## Scope

- `SECURITY.md`
- `lib/auth.js`, `lib/oauth.js`, `lib/oauth-rules.js`, `lib/validation.js` (`toSafeNextPath`)
- `app/api/auth/[...all]/route.js`, `app/signin/page.js`, `app/signin/actions.js`
- `app/oauth/consent/page.js`, `app/oauth/consent/actions.js`
- `app/.well-known/*/route.js`
- `app/mcp/route.js`, `app/api/mcp/route.js`, `lib/mcp/server.js`
- `lib/api-keys.js`, `lib/api-key-tokens.js`, `lib/api/rest.js`, `lib/actions/{run-action,define-action,api-keys,connected-apps}.js`, `lib/account-state.js` (parts)
- `app/api/v1/**`, `lib/export-token.js`
- `lib/rate-limit.js`, `lib/rate-levels.js`
- `app/api/webhooks/{stripe,ses,ses-tenants,postshiba}/route.js`, `lib/sns.js`, `lib/sns-webhook.js`, `lib/postshiba-webhook.js`, `lib/email-provider/postshiba-events.js`, `lib/stripe.js`, `lib/billing.js` (`handleStripeEvent`), `lib/tenant-events.js`, `lib/email-events.js`
- `app/api/jobs/*` (6 routes), `app/api/cron/*` (3 routes), `lib/qstash-route.js`, `lib/cron.js`, `lib/account-purge.js`
- `app/api/admin/postmaster/{connect,callback}/route.js`, `lib/postmaster.js`
- `proxy.js`, `next.config.mjs`
- `tests/qstash-route.test.mjs`, `tests/oauth.test.mjs` (parts)

## Findings

| ID | Title | Severity |
|---|---|---|
| AUTH-01 | API keys created by an OAuth app / MCP agent survive disconnecting the app | Medium |
| AUTH-02 | Per-IP rate limits trust client-supplied `X-Forwarded-For` | Low (Vercel) / Medium (self-host) |
| AUTH-03 | SNS webhooks: no replay protection, SignatureVersion 1 (SHA-1) accepted | Low |
| AUTH-04 | Job endpoints skip signature checks on `NODE_ENV=development`; signature maybe not bound to URL | Low |
| AUTH-05 | Dynamic client registration only partly restricts client metadata | Low |
| AUTH-06 | OTP guesses per address capped only through codes-per-inbox | Low |
| AUTH-07 | Consent screen headlines the client's self-chosen name | Info |
| AUTH-08 | PostShiba event with account but no tenant trusted on unique args alone | Info |
| AUTH-09 | 20-active-key cap can be exceeded by concurrent requests | Info |

### AUTH-01 — API keys an OAuth app or MCP agent creates keep working after it is disconnected

**Severity: Medium.** Breaks the documented guarantee that disconnecting an app cuts it off; reachable by an LLM agent over MCP that a prompt injection can steer. Not unauthenticated: needs a connected app or full key acting maliciously or under injection.

**Where:**
- `lib/actions/api-keys.js:53-74` — `create_api_key` is available to any full-level caller, OAuth connections included; `lib/mcp/server.js:94` gives them `level: "full"`.
- `lib/oauth.js:182-192` — `revokeConnectedApp` deletes only that app's tokens and consent.
- `models/ApiKey` — a key does not record who created it (`api-keys.js:71`).

**Explanation:** A connected app (ChatGPT, Claude, Grok Bot, or any CIMD client) can call `create_api_key` with `level: "full"` and receive a working `mc_…` key. Disconnecting the app later removes its tokens, but the minted key keeps full access until revoked by hand. The consent page does list "Create and revoke API keys", but users won't read that as "can leave a permanent backdoor".

**Fix:** record the creator on each key (`createdBy: oauth|apiKey|dashboard`); revoke a connection's keys in `revokeConnectedApp` / `disconnectAllApps`; better, forbid OAuth connections from `create_api_key` (an `oauthAllowed: false` flag checked in `runAction`) or limit them to `signups` keys; show the creating app on the API keys page.

### AUTH-02 — Per-IP rate limits trust the client's `X-Forwarded-For`

**Severity: Low on Vercel (edge overwrites the header — not verified in code), Medium for self-hosting** behind nginx `$proxy_add_x_forwarded_for` or Node exposed directly.

**Where:** `lib/rate-limit.js:72-75` takes the leftmost `x-forwarded-for` entry, then `x-real-ip`. Used by `CODES_PER_IP`, `CHECKS_PER_IP` (`app/signin/actions.js:16,19,34,61`), `SIGNUPS_PER_IP` (`lib/auth.js:44,128-129`), `REGISTRATIONS_PER_IP` (`lib/auth.js:94,179`). `tests/oauth.test.mjs:246` itself rotates this header to pass the registration limit.

**Explanation:** a random `X-Forwarded-For` per request bypasses every per-IP limit: sign-in code spam to many addresses, code checks, dynamic client registration, and (off Vercel) the 3-accounts-a-day signup limit. Per-inbox limits still cap OTP guessing on one address (AUTH-06): this is an abuse-limit bypass, not an auth bypass.

**Fix:** derive the client IP from a trusted source — on Vercel, the platform header; elsewhere a `TRUSTED_PROXY_HOPS` setting taking the Nth entry from the right; default to the socket address. Document it for self-hosters.

### AUTH-03 — SNS webhooks: no replay protection, accepts SignatureVersion 1 (SHA-1)

**Severity: Low.** Replay needs an AWS-signed SNS message, which normally only AWS and the server see (could leak from logs/proxies).

**Where:** `lib/sns.js:6-9` maps `SignatureVersion "1"` to `RSA-SHA1`; `lib/sns.js:84-99`, `lib/sns-webhook.js:55-67` — `Timestamp` is signed but never compared with the clock, `MessageId` never deduplicated.

**Explanation:** a captured signed bounce/complaint `Notification` stays valid forever. Bounce and complaint rates drive automatic pauses and account blocks; if any handler is not fully idempotent (claimed in a comment, not verified for every SES event type), replays could push an account toward a block.

**Fix:** reject `Timestamp` older than ~1 h; store processed `MessageId`s (TTL index ~1 day); configure topics with `SignatureVersion=2` and only accept `"2"`.

### AUTH-04 — Job endpoints skip signature checks when `NODE_ENV=development`; signatures may not be bound to the URL

**Severity: Low.** Needs a misconfigured deployment or a captured QStash request.

**Where:** `lib/qstash-route.js:27` (`if (process.env.NODE_ENV === "development") return handler(request);`), `lib/qstash-route.js:33`.

1. `SECURITY.md` says the bypass only happens "in `next dev`", but the code keys on `NODE_ENV`. A self-hosted container started with `NODE_ENV=development` (common docker-compose copy-paste) exposes every `/api/jobs/*` endpoint, which `proxy.js:49` makes public. Handlers re-check state (`runPurge` needs a matching `purgeAt`, `send-batch` only sends existing messages), limiting damage — still a fail-open path.
2. Unverified without `node_modules`: `verifySignatureAppRouter` may call `Receiver.verify` without the request `url`, so the JWT `sub` (destination) isn't checked, and no `jti` dedup happens. A captured signed body could then be replayed to the same or another job endpoint within the token lifetime (~5 min). No test covers a mismatched subject.

**Fix:** skip verification only behind an explicit local flag that is refused when `NODE_ENV === "production"` or `APP_URL` isn't localhost; verify with a `Receiver` passing `url: request.url` (or check `sub`); optionally store `jti` 10 min; add a mismatched-subject test.

### AUTH-05 — Dynamic client registration only partly restricts client metadata

**Severity: Low.** Redirect URIs are strictly allowlisted, so codes only reach Cursor. What remains is consent-screen spoofing and library-dependent fields.

**Where:** `lib/auth.js:178-199` (`checkRegistration`), `lib/auth.js:255-257` (hook matches only `ctx.path === "/oauth2/register"`).

- Only `token_endpoint_auth_method != none`, `backchannel_logout_uri` and off-list redirect URIs are refused; every other RFC 7591 field passes through.
- `client_name` is free text (e.g. "Claude", "MailCheap Support") and is the consent page headline (`app/oauth/consent/page.js:49`).
- Depending on the Better Auth version, fields like `post_logout_redirect_uris`, `skip_consent`, `require_pkce`, `grant_types`, `scope` may be accepted unauthenticated — not verifiable without the library.
- If the MCP plugin mounts another registration path (the legacy plugin used `/mcp/register`), it would skip the allowlist; `OAUTH_DISABLED_PATHS` (`lib/auth.js:73-89`) doesn't list one.

**Fix:** rebuild the registration body from an allowlist (`redirect_uris`, `token_endpoint_auth_method: "none"`, length-capped `client_name`, fixed `grant_types`/`response_types`), drop everything else, pin the name of allowlisted clients, and test that any other registration route answers 404.

### AUTH-06 — OTP guesses per address are capped only by codes per inbox

**Severity: Low.** Small odds, noisy (the victim receives every code).

**Where:** `app/signin/actions.js:16-19,34,61`; Better Auth emailOTP allows 3 attempts per code.

**Explanation:** 5 codes / 15 min × 3 tries = 1,440 guesses/day against 10⁶ codes ≈ 0.14 %/day, ~4 %/month against one chosen account, from a single IP. No per-email cap on failures across codes, no lockout, no alert.

**Fix:** per-email failed-verification limit (e.g. 10/day), alert after N failures, or 8-digit codes.

### AUTH-07 — Consent screen leads with the client's self-chosen name

**Severity: Info** (phishing hardening). `app/oauth/consent/page.js:49-50,101`, `lib/oauth-rules.js:98-102`. For CIMD clients the host is correctly derived from the `client_id` URL, but the headline is the document's `name`; the host appears in small text and only when it differs. **Fix:** always show the `client_id` host prominently ("evil.example calls itself 'Claude'") and flag names of well-known apps on a non-matching host.

### AUTH-08 — PostShiba event naming an account but no tenant is trusted on unique args alone

**Severity: Info** (needs a PostShiba-signed event). `lib/postshiba-webhook.js:28-31,47`: when `tenant_id` is missing the event is processed with `isAccountTrusted: true`, so suppressions/unsubscribes apply without a matching message. `SECURITY.md` says args "have to match the PostShiba tenant on the event" — the code is looser. **Fix:** without a tenant, refuse the event or set `isAccountTrusted: false`.

### AUTH-09 — The 20-active-key cap can be exceeded concurrently

**Severity: Info.** `lib/actions/api-keys.js:67-71` counts then creates, non-atomically. The 10/hour key-creation limit caps the overshoot at ~10. **Fix:** conditional counter (`$inc` with `activeKeys < 20`) or a transaction.
