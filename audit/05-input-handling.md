# 05 — Input handling: XSS, HTML/email injection, NoSQL, SSRF, CSV, headers

Static review, read-only; nothing was executed. Exploitability is inferred from the code.

## Scope

- **Sanitizing & rendering:** `lib/clean-html.js`, `lib/safe-html.js`, `lib/markdown.js`, `lib/render-email.js`, `lib/merge-tags.js`, `lib/archive-content.js`, `lib/archive-seo.js`, `lib/themes.js`, `lib/colors.js`, `lib/og-image.js`, `lib/thumbnail.js`, `lib/tracking.js`, `lib/summary-email.js`, `lib/telegram.js`.
- **Components:** `FrameIssue.jsx`, `ScriptJsonLd.jsx`, `StyleTheme.jsx`, `PreviewEmail.jsx`, `DialogPreviewEmail.jsx`; every `dangerouslySetInnerHTML`, `srcDoc`, `innerHTML`/`DOMParser` sink grepped.
- **Public pages:** `app/archive/**`, `app/embed/**`, `app/api/embed/[id]`, `public/embed.js`, `lib/signup-form*.js`, `app/p/[slug]`, `app/c/[token]`, `app/api/views`, `app/api/unsubscribe`, admin previews.
- **Headers & sender:** `lib/sender.js`, `lib/sender-rules.js`, `lib/sending-rules.js`, `lib/email-provider/postshiba-rules.js`, `lib/ses.js`, `lib/platform-email.js`.
- **NoSQL & ReDoS:** `lib/validation.js`, `lib/api/rest.js`, `lib/actions/*`, `lib/segments.js`, `lib/segment-queries.js`, `lib/contacts.js`, `lib/search.js`, `lib/email-aliases.js`, `lib/accounts.js`, `app/archive/actions.js`.
- **SSRF:** `lib/remote-image.js`, `lib/og-image.js`, `lib/favicon.js`, `lib/gravatar.js`, `lib/dns-check.js`, `lib/sns*.js`, every server-side `fetch(`.
- **CSV:** `lib/contacts-csv.js`, `app/api/v1/contacts/export.csv/route.js`, `lib/export-token.js`, `lib/csv-import.js`.
- **Redirects, headers, routing:** `next.config.mjs`, `proxy.js`, `lib/host-routing.js`, `lib/archive-hosts.js`, `lib/rate-limit.js`, sign-in / consent / billing redirects.
- **Uploads:** `app/api/uploads/route.js`, `lib/actions/uploads.js`, `lib/image-files.js`, `lib/image-storage.js`, `lib/upload-image.js`.

## Findings

| ID | Title | Severity |
|---|---|---|
| INJ-01 | Server-side resource exhaustion in a public image route — **details withheld** | Medium |
| INJ-02 | Per-IP rate limits key on client-controlled `X-Forwarded-For` (self-host) | Low |
| INJ-03 | Signup "done" page signs and shows a "Back to …" link to any http(s) URL | Low |
| INJ-04 | CSP has no `script-src`; archive rich-issue HTML relies on the sanitizer alone | Info |
| INJ-05 | SSRF blocklist misses a few legacy IPv6 ranges | Info |

### INJ-01 — Resource exhaustion in a public, unauthenticated image route (details withheld)

**Severity: Medium.** A customer-controlled setting is used to build a server-side file path without sufficient containment, which can be abused to make an unauthenticated public endpoint exhaust server memory (denial of service; a single self-hosted process can be taken down). No secrets are disclosed.

Technical details, affected lines and the fix are **withheld from this public report** until the maintainer has been notified through the project's [private vulnerability reporting](https://github.com/marclou/mailcheap/security) and has had time to ship a fix. This section will be updated after disclosure.

### INJ-02 — Per-IP rate limits key on a client-controlled `X-Forwarded-For`

**Severity: Low** (none on Vercel, which overwrites the header). Same root cause as [AUTH-02](04-auth-oauth-webhooks.md#auth-02--per-ip-rate-limits-trust-the-clients-x-forwarded-for).

`lib/rate-limit.js:72-75` uses the first `x-forwarded-for` entry. Behind proxies that append to the header (nginx `proxy_add_x_forwarded_for`, many LBs), that entry is client-chosen, so rotating it bypasses per-IP limits on sign-in codes, `subscribe-ip`, the embed `perIp` / `frameLoads` limits and the `views` counter. Other limits (3 tries per code, per-inbox/newsletter/key limits, BotID on Cloud) keep the impact low.

**Fix:** platform IP on Vercel; a `TRUSTED_PROXY_HOPS` setting for self-hosting.

### INJ-03 — Signup "done" page will sign and show a "Back to …" link to any http(s) URL

**Severity: Low** (needs a click; the target host is visible in the link text).

- `app/api/embed/[id]/route.js:165` — `back = referer || origin` is copied into the sealed done token on every redirect, including error replies.
- `lib/signup-form-token.js:62` only checks it parses as http(s).
- `lib/signup-form-page.js:110,115` renders it as `<a href>` on the app origin.

With a forged `Referer`, anyone can POST to `/api/embed/<public newsletter id>` and get a signed `/embed/<id>/done?s=…` URL on MailCheap's domain saying "Back to evil.example" — a weak phishing aid valid for `MAX_DONE_AGE_MS`. No XSS (scheme restricted, value escaped).

**Fix:** only set `back` when the form passed the host check (`isAllowedHost` with non-empty `allowedDomains`) and the browser `Origin` matches the referer; or only show it for the newsletter's allowed domains.

### INJ-04 — CSP sets no `script-src`; archive HTML relies on the sanitizer alone

**Severity: Info** (documented trade-off, no bypass found). `next.config.mjs:5-12`: `script-src-attr 'none'` blocks inline handlers, but `<script>` elements and `javascript:` URLs aren't restricted by CSP. Archive issues are safe today because `sanitize-html` runs on every render (`lib/archive-content.js:19`, `lib/safe-html.js`) and HTML issues render in a script-less sandbox. On Cloud archives live on `<slug>.mailcheap.ai` (same site as the app); self-hosted without a shared domain they're served on the app origin — one sanitizer regression would be stored XSS.

**Suggestion:** a stricter CSP for `/archive/*` responses (e.g. static archive HTML with `script-src 'self'`); keep the sanitizer test corpus broad (mXSS, `<math>`/`<svg>` nesting, unclosed tags).

### INJ-05 — SSRF blocklist misses a few legacy IPv6 forms

**Severity: Info.** `lib/remote-image.js:28-37` handles IPv4-mapped addresses correctly, but not IPv4-compatible `::/96`, 6to4 `2002::/16`, Teredo `2001::/32`. Not routed to local addresses on modern hosts, and fetches are HTTPS:443 only, ruling out metadata endpoints. **Suggestion:** add those ranges, or allow only public unicast.
