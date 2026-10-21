# 03 — Authorization & multi-tenant isolation

Static review, read-only. **Result: no cross-account read or write was found** through the dashboard, REST, MCP, server actions, signed links or the admin area. One Medium issue concerns what a single account's public, low-privilege key can do within that account.

## Scope

- **Model & plugin:** `SECURITY.md`, `lib/account-scope.js`, `lib/db.js`, every `models/*.js`, all 34 `crossAccount: true` call sites in `lib/` and `app/`.
- **Actions layer:** `lib/actions/*` (define-action, run-action, registry, shared, dashboard, errors, api-keys, connected-apps, billing, campaigns, contacts, domains, newsletters, overview, segments, uploads).
- **Helpers:** `lib/contacts.js`, `segment-queries.js`, `import-jobs.js`, `domains.js`, `profile.js`, `status-changes.js`, `public-newsletters.js`, `archive-data.js`, `website-domains.js`, `list-hygiene.js`, `email-events.js`.
- **Server actions:** `app/(app)/*/actions.js`, `app/(account)/*/actions.js`, `app/{welcome,restore,keep,confirm,archive,signin,summary/[token],oauth/consent}/actions.js`, `app/admin/actions.js`.
- **Pages & routes:** `app/admin/**`, `app/api/admin/postmaster/*`, `app/(app)/{contacts,campaigns}/[id]/page.js`, `app/(app)/newsletters/[id]/route.js`, `app/api/{uploads,views,unsubscribe,summary/unsubscribe,v1/contacts/export.csv,embed/[id]}/route.js`, `app/{c,o}/[token]/route.js`, `app/p/[slug]/route.js`, `app/api/jobs/*`, `app/api/cron/*`.
- **Auth & transport:** `lib/auth.js`, `lib/selected-newsletter.js`, `proxy.js`, `lib/api-keys.js`, `lib/api/rest.js`, `lib/mcp/server.js`, `lib/oauth.js`, `lib/qstash-route.js`.
- **Tokens:** `lib/{contact-token,export-token,summary-token,signup-form-token,tracking,unsubscribe,confirmation,last-chance}.js`.
- **Test:** `tests/parity.test.mjs`.

## Findings

| ID | Title | Severity |
|---|---|---|
| AUTHZ-01 | A public, low-privilege API key can do more than documented — **details withheld** | Medium |
| AUTHZ-02 | Scoping plugin checks that an `accountId` is present, not whose; ignores `$lookup` and raw collection access | Low |
| AUTHZ-03 | Registry-bypass test doesn't scan `app/welcome/actions.js` | Info |
| AUTHZ-04 | `SECURITY.md`'s list of cross-account queries is incomplete | Info |
| AUTHZ-05 | A plain GET switches the selected newsletter | Info |
| AUTHZ-06 | One HMAC uses the raw app secret instead of an HKDF-derived key | Info |

### AUTHZ-01 — Low-privilege key exceeds its documented capabilities (details withheld)

**Severity: Medium.** A key level that is designed to be embedded in public web pages can do more than `SECURITY.md` describes, with privacy and consent implications for the subscribers of the account that published it. It does **not** give access to any other account's data.

Technical details, affected lines and the fix are **withheld from this public report** until the maintainer has been notified through the project's [private vulnerability reporting](https://github.com/marclou/mailcheap/security) and has had time to fix it. This section will be updated after disclosure.

### AUTHZ-02 — Scoping plugin checks presence, not ownership; ignores `$lookup` and raw access

**Severity: Low (hardening).** No exploitable path found today.

**Where:** `lib/account-scope.js:22-29`, `:38-40`, `:41-46`.

- `assertScoped` only checks that `filter.accountId` *looks like* an ObjectId; correctness relies on every caller building filters with `scoped(account, …)`.
- For aggregations only `pipeline()[0].$match` is inspected; later `$lookup` / `$unionWith` / `$graphLookup` stages run unchecked.
- Direct driver calls (`Model.collection.*`, `connection.db.collection()`) bypass the hooks.

Current uses are safe: `lib/contacts.js:39-56` and `lib/overview-data.js:~118-128` join `Contact` by `newsletterId`/`email` without `accountId` (left side is scoped and a `newsletterId` belongs to one account — the `overview-data` join is the more fragile one). `lib/accounts.js:116-133`, `lib/email-events.js:133-142` are admin-only or token-keyed; `lib/oauth.js:45` raw access is always filtered by `userId`.

**Fix:** reject `$lookup`/`$unionWith`/`$graphLookup` on scoped collections unless their sub-pipeline matches `accountId` or the query is `crossAccount`; add `accountId` to the two `Contact` lookups; optionally reject string `accountId` so only `scoped()`-built ObjectIds pass.

### AUTHZ-03 — Registry-bypass test misses `app/welcome/actions.js`

**Severity: Info.** `tests/parity.test.mjs:11-13` only scans `app/(app)` and `app/(account)`. `app/welcome/actions.js` writes directly (`Account.updateOne` at `:43-52`, `:59-62`; `Newsletter.updateOne(scoped(...))` at `:74`). These are session-scoped and set fixed fields — not exploitable — but skip `runAction`'s blocked/plan checks, and `SECURITY.md`'s claim is wider than the test. **Fix:** add `app/welcome` to `DASHBOARD_DIRECTORIES` with an explicit allowlist, or register those writes as dashboard-only actions.

### AUTHZ-04 — Incomplete list of cross-account queries in `SECURITY.md`

**Severity: Info.** All cross-account queries found are justified, but some are undocumented: list-hygiene cron (`lib/list-hygiene.js:39-55,105`); uniqueness checks for slugs (`lib/onboarding.js:47`, `lib/actions/newsletters.js:339`, `lib/archive-slugs.js:10`) and website hosts (`lib/website-domains.js:36,45`); archive host lookups (`lib/host-routing.js:37`, `lib/archive-data.js:52`); admin views (`lib/sending-capacity.js:53`, `admin-stats.js`, `admin-alerts.js`, `content-review-decisions.js`); the public view counter (`app/api/views/route.js:23-26`, anyone can bump `webViews` of any web-published issue, 30/10 min/IP — vanity metric only). **Fix:** update `SECURITY.md`; add a test that diffs `crossAccount: true` call sites against an allowlist.

### AUTHZ-05 — Plain GET switches the selected newsletter

**Severity: Info.** `app/(app)/newsletters/[id]/route.js:7-12` sets the selected-newsletter cookie on GET (`lib/selected-newsletter.js:35-42`); SameSite=Lax allows top-level cross-site GETs, so another site can switch which of the victim's **own** newsletters the dashboard shows. Actions take explicit ids, so impact is confusion only. **Fix:** accept, or switch via POST/server action.

### AUTHZ-06 — Raw app secret used as an HMAC key

**Severity: Info.** `app/api/embed/[id]/route.js:198-200` keys `hashEmail` with `getAppSecret()` directly, whereas every other token derives its own HKDF key. Output is only a rate-limiter key. **Fix:** derive with its own label (e.g. `mailcheap-embed-email-v1`).
