# 07 — Architecture, quality & performance

Static review, read-only (nothing built or run).

## 1. Architecture & conventions

- `app/` = routes, pages, thin server-action files; `lib/` = domain logic (`lib/actions/*` = customer actions); `models/` = Mongoose schemas; `components/` = UI. `AGENTS.md` describes this and the code mostly follows it.
- **Single action registry.** Every customer action is a `defineAction` (`lib/actions/define-action.js:25-86`), listed in `lib/actions/registry.js:14-28` (duplicate names rejected), run only through `runAction` (`lib/actions/run-action.js:34-64`): permission → blocked account → plan → per-caller and per-account rate limits → zod validation. Dashboard server actions consistently call `runDashboardAction`. REST (`lib/api/rest.js`), MCP (`lib/mcp/server.js:59`), OpenAPI (`lib/docs/openapi.js`) and `llms.txt` are generated from the same schemas, enforced by `tests/parity.test.mjs`.
- **Tenant scoping enforced in the model layer** (`lib/account-scope.js:31-47`): any query/aggregate/bulkWrite without `accountId` throws unless `crossAccount: true`. Unusually strong.
- **Cloud vs self-host** via `isCloud()` (`lib/config.js:9`) and aliased site extensions (`next.config.mjs:37-45`).

**Deviations**

- Pages query models directly (11 non-API files under `app/` import `@/models/*`, e.g. `app/(app)/contacts/[id]/page.js:55-84`). Read logic is spread across pages, `lib/*-queries.js` and actions; pagination differs (cursor in API `lib/actions/contacts.js:280`, `skip` in dashboard `lib/contacts.js:114-130`).
- `lib/health.js` is not a health check but an 8-line UI helper (`rateTone`).
- Security-sensitive helpers duplicated: `escapeHtml` ×3 (`lib/render-email.js:26`, `lib/signup-form.js:127`, `lib/summary-email.js:30`), `escapeRegex` ×4 (`lib/contacts.js:58`, `lib/segments.js:391`, `lib/email-aliases.js:43`, inline in `lib/accounts.js:90`).
- Action `output` schemas feed MCP `outputSchema` and docs but are never validated against REST responses — docs can drift silently.

## 2. Size hotspots

| # | File | Lines | Note |
|---|---|---:|---|
| 1 | `lib/sending.js` | 1307 | Audience, queueing, first-send holding, batch loop (`sendMessageBatch` ~940-1120: one ~180-line loop, ~10 exits), signup and test emails, QStash publishing. Main split candidate. |
| 2 | `lib/actions/campaigns.js` | 1078 | Schemas + handlers with business logic. |
| 3 | `components/FormNewsletter.jsx` | 959 | Client component, 17 functions, 3 client-side `renderEmail` previews. |
| 4 | `lib/actions/contacts.js` | 956 | No direct test import. |
| 5 | `components/FormCampaign.jsx` | 759 | Sensible `useDeferredValue`/`useMemo` preview. |
| 6 | `lib/actions/newsletters.js` | 641 | |
| 7 | `components/DialogImportContacts.jsx` | 584 | CSV parsed on main thread (PERF-08). |
| 8 | `lib/csv-import.js` | 543 | |
| 9 | `components/DialogPreviewEmail.jsx` | 526 | |
| 10 | `components/DialogSegment.jsx` | 515 | |
| 11 | `lib/content-review.js` | 505 | Large hand-written prompt-injection regex set (`:82-98`) — brittle. |
| 12 | `components/CampaignReport.jsx` | 419 | |
| 13 | `lib/themes.js` | 415 | |
| 14 | `lib/contacts.js` | 415 | |
| 15 | `lib/auth.js` | 415 | |

Worst page: `app/(app)/contacts/[id]/page.js` (647 lines: loading + timeline + JSX). Functions are otherwise small, and comments explain constraints rather than restate code.

## 3. Data model & performance

**Connections** (`lib/db.js`): promise cached on `globalThis`, reset on failure, `maxPoolSize: 10`, `maxIdleTimeMS: 5000`, `attachDatabasePool` for Vercel — right for serverless, but causes reconnect churn on a long-running self-hosted `npm start`. No `autoIndex` setting: Mongoose calls `createIndex` for every model on every cold start, and nothing migrates or drops indexes.

**Unbounded growth**
- `Message` (one doc per recipient per send) and `Event` (raw provider `data` + email) have no TTL/archival; only the account purge deletes them.
- `Event` is **write-only** — created at `lib/email-events.js:187`, never read in `lib/` or `app/` — yet carries 4 indexes (`models/Event.js:7,21-23`).
- Bounded arrays done well: `statusChanges` (100), alert occurrences, dismissals via `$slice`; TTLs on `ImportChunk` and `GlobalSuppression`.

**Missing indexes / full scans**
- **Admin dashboard: 3 full scans of `Message` per load** — `lib/sending-capacity.js:41-52` matches on `sentAt` only (no index leads with it); `lib/admin-stats.js:89-100` uses `accountId: {$nin: admins}`; `:100` counts every message ever sent.
- **List-hygiene cron (every 6 h) scans every contact of every account** — groups on a computed domain (`$split` of email) (`lib/list-hygiene.js:40-57`), `restoreContacts` uses `Number.MAX_SAFE_INTEGER` as limit (`:80`), plus `findDelistedSpamDomains` (`:101`).
- **Audience query on every send:** six-key sort `AUDIENCE_SORT` (`lib/audience.js:2-9`) with no matching index (closest `{newsletterId,status,lastOpenAt}`), plus the account's **whole suppression list in a `$nin`** (`lib/audience.js:19`, `lib/sending.js:387-389`), which grows toward the 16 MB BSON limit.
- **Contacts page: up to ~110 queries per view** — 6 status counts, 1 + up to 100 segment counts (some case-insensitive `$regex`), an `$unwind` tag aggregation, a `skip` find (`app/(app)/contacts/page.js:66-71`, `lib/segment-queries.js:75-82`, `lib/contacts.js:118-123,318-326`). The detail page repeats the tag aggregation. No caching.
- **Campaign report** runs a `$facet` over all of a campaign's messages on every view (`lib/campaign-report.js:10-18`).

**Send loop** (`lib/sending.js:993-1110`): per message a `Campaign.exists` (`:1000`), a claim `updateOne` (`:1057`), and `recordSent` (Message update + Campaign `$inc` + Contact update) — ~9 DB ops per email (acknowledged in `lib/config.js:128`), with the per-message `$inc` serialising on one hot Campaign document. `findPreviousMessages` (`:376-385`) and `holdFirstSendRest` (`:498-503`) load whole campaigns' ids into memory.

**Cache invalidation is global:** `revalidateArchive()` (`lib/archive-data.js:146-148`) revalidates the `/archive/[slug]` layout, so any account publishing invalidates every archive.

**Done well:** compound indexes on Contact with `_id` tiebreakers matching paginated sorts (`models/Contact.js:88-96`); partial unique indexes for idempotent sends (`models/Message.js:46-54`); partial index on queued messages (`:63`).
