# 06 — Business logic: sending, billing, compliance, tracking, imports

Static review, read-only. Totals: 0 Critical, 0 High, **6 Medium**, 5 Low, 1 Info.

The core send pipeline is solid: no way was found to send a message twice or to exceed the monthly plan limit. The most significant issues are **compliance gaps** (opt-out handling, CAN-SPAM footer) and **abuse channels** (quota-free confirmation emails, a long-lived redirector).

## Scope

- **Sending:** `lib/sending.js`, `send-rate.js`, `send-spread.js`, `sending-capacity.js`, `sending-rules.js`, `schedule.js`, `scheduling.js`, `first-send.js`, `presend-checks.js`, `content-review*.js`, `signup-email-review.js`, `qstash-route.js`, `app/api/jobs/*`, `models/{Campaign,Message}.js`, `lib/email-provider/*`, `lib/actions/campaigns.js`.
- **Billing:** `lib/billing.js`, `stripe.js`, `plans.js`, `usage.js`, `usage-period.js`, `account-state.js`, `lib/actions/billing.js`, `app/(account)/billing/**`, Stripe webhook.
- **Compliance:** `lib/unsubscribe.js`, `app/api/unsubscribe/route.js`, `confirmation.js`, `app/confirm/*`, `last-chance.js`, `global-suppressions.js`, `models/Suppression.js`, `ses-events.js`, `email-events.js`, `email-event-rules.js`, `postal-address.js`, `render-email.js` (footer), `account-deletion.js`, `account-purge.js`, `image-storage.js`, `signups.js`, `contacts.js`, `app/archive/actions.js`, `app/api/embed/[id]/route.js`.
- **Tracking:** `lib/tracking.js`, `app/o/[token]`, `app/c/[token]`, `lib/archive-hosts.js`, `proxy.js`.
- **Imports:** `lib/contact-import.js`, `import-jobs.js`, `import-checks.js`, `models/ImportChunk.js`.

## Findings

| ID | Title | Severity |
|---|---|---|
| LOGIC-01 | Website signup form silently re-subscribes people who unsubscribed | Medium |
| LOGIC-02 | HTML-format emails can go out without a postal address | Medium |
| LOGIC-03 | Gmail/Yahoo one-click unsubscribes can be dropped by a global per-IP rate limit | Medium |
| LOGIC-04 | Confirmation emails are a quota-free channel to unvouched addresses | Medium |
| LOGIC-05 | Signed click links can be turned into a long-lived redirector | Medium |
| LOGIC-06 | Plan upgrade while `past_due` may grant higher limits without payment (unverified) | Medium |
| LOGIC-07 | Concurrent queue / send-more calls can widen the audience or leave a campaign stuck | Low |
| LOGIC-08 | No sweeper for campaigns whose batch jobs died | Low |
| LOGIC-09 | Reserved plan room can leak; month-boundary refunds can go negative | Low |
| LOGIC-10 | Pending contacts can't unsubscribe, page still says "done" | Low |
| LOGIC-11 | Segments survive the account purge; unsubscribe links depend on the customer's domain | Low |
| LOGIC-12 | No opt-out for open/click tracking; content check fails open | Info |

### LOGIC-01 — Website signup form silently re-subscribes people who unsubscribed (Medium)

**Where:** `app/archive/actions.js:61-67`, `lib/contacts.js:206-224`, `lib/signups.js:9-23`.

The archive (website) signup action calls `upsertContact({ resubscribe: true })` without the `confirmReturning` guard the embed endpoint uses. With `requireConfirmation: false` (welcome-email mode), an `unsubscribed` or `dormant` contact goes straight back to `active`, and no email is sent (the welcome email only goes out when `created`). Anyone who knows a former subscriber's address can opt them back in from the newsletter's site.

**Why it matters:** contradicts CAN-SPAM/GDPR expectations about honouring an opt-out, and the project's own rule in `AGENTS.md` ("bringing someone back takes their consent"). BotID and 5/IP/10 min slow bulk abuse, not targeted abuse.

**Fix:** treat returning unsubscribed/dormant contacts as `pending` with a confirmation email, like `addSignup({ confirmReturning: true })` — simplest is to route this action through `addSignup`.

### LOGIC-02 — HTML-format emails can go out without a postal address (Medium)

**Where:** `lib/render-email.js:207`, `lib/sending-rules.js:125`.

`withFooter` skips the whole footer (unsubscribe link **and** postal address) whenever the HTML contains the literal `{{unsubscribe_url}}` anywhere — even in an HTML comment or hidden element. `getSendProblem` only checks that the newsletter *has* an address, not that the email shows it. So HTML issues with their own unsubscribe link never print the physical address; one that hides the placeholder has no visible opt-out at all (only `List-Unsubscribe`).

**Fix:** always append the postal address; only skip the unsubscribe footer when the placeholder is in a real `href`; optionally refuse a send whose rendered HTML lacks the address.

### LOGIC-03 — One-click unsubscribes can be dropped by a global per-IP rate limit (Medium)

**Where:** `app/api/unsubscribe/route.js:14`, `:87`.

The RFC 8058 POST endpoint is limited to 60/min per client IP, a bucket shared by every account. Gmail/Yahoo one-click requests come from a small pool of provider IPs; a big campaign (or several customers at once) exceeds 60/min, extra requests get `429`, and providers don't reliably retry → legally required opt-outs silently lost, and a hit on Gmail/Yahoo bulk-sender compliance. Self-hosted without `x-forwarded-for`, everyone shares the `"unknown"` bucket.

**Fix:** don't IP-limit requests carrying a valid HMAC token (the action is idempotent); limit only invalid-token requests, or key by token/contact.

### LOGIC-04 — Confirmation emails are a quota-free channel to unvouched addresses (Medium)

**Where:** `lib/sending.js:98-100`, `:1250-1276`; `lib/actions/contacts.js:58-62`, `:489-503`.

`add_contact` via API/MCP creates a `pending` contact for any address with no consent flag, which triggers the newsletter's **customisable** confirmation email. Confirmation emails are excluded from the plan quota (`PLAN_KINDS`), the first-send slice and campaign rate pauses. Remaining caps (600 signups/h × plan scale, 100/10 min per newsletter × 3 newsletters, 3 confirmations per address) allow roughly 14k emails/day on Starter — ~20× its 20k/month quota — with owner-written content, gated only by a fail-open content check and the 7-day account health check.

**Fix:** count API/MCP-originated confirmation emails against the plan (or a lower cap); apply `getDropReason` (role/disposable/spam-domain) to API signups as imports do; apply first-send slicing/rate checks to confirmation volume.

### LOGIC-05 — Signed click links can be turned into a long-lived redirector (Medium)

**Where:** `lib/tracking.js:57-71`, `app/c/[token]/route.js:40-52`, `lib/sending.js` (tracked kinds, test sends), `lib/account-purge.js`.

Signed click-tracking links never expire, and some email kinds that bypass content review are still link-tracked. A paying account can therefore mint signed links that 302 from MailCheap's domain to arbitrary URLs. Blocking the account disables them, but the route's fallback when the original message no longer exists redirects again, so the protection doesn't survive account deletion. Phishers routinely abuse ESP redirectors; this can land the shared domain on URIBL/SURBL blocklists, hurting every customer.

**Fix:** don't track test emails (or content-review them); show the interstitial (`continuePage`) instead of redirecting when the message is gone; consider a timestamp in the signature and an interstitial after N months.

### LOGIC-06 — Plan upgrade while `past_due` may grant higher limits without payment (Medium, unverified)

**Where:** `lib/plans.js:61` (`past_due` counts as paid), `lib/billing.js:47-61`, `lib/stripe.js:94-120`.

`applySubscription` sets `plan` from the subscription item's price as soon as Stripe reports the change, regardless of the latest invoice. If the Customer Portal uses deferred prorations / `allow_incomplete`, a Starter customer could switch to Max and get its limit for the period; if the proration charge fails, access continues through dunning. **Depends on Stripe portal configuration, which isn't in the repo** — severity is a guess.

**Fix:** portal on `always_invoice` + `pending_if_incomplete`; ignore `pending_update` and raise `plan` only once the upgrade invoice is paid; consider giving `past_due` the lower of old/new plans.

### LOGIC-07 — Concurrent queue / send-more calls can widen the audience or leave a campaign stuck (Low)

**Where:** `lib/sending.js:775-805`, `:833-859`, `:425-438`.

`insertCampaignMessages` runs **before** the conditional status update. Two racing calls (scheduled job + "send now" with a different segment/`audienceLimit`, or two `send_campaign_to_more`) both insert queued messages; the loser throws a conflict but its messages stay queued and may be sent (contacts outside the chosen segment get the email), or never be published, leaving the campaign `sending` forever and inflating `countEmailsInFlight`. Per-message checks still protect unsubscribed contacts and the plan limit holds.

**Fix:** claim the campaign first (`draft|scheduled → queueing` atomically), then insert; on failure delete that campaign's `queued` messages.

### LOGIC-08 — No sweeper for campaigns whose batch jobs died (Low)

**Where:** `app/api/jobs/send-batch/route.js:13`, `:40`; `lib/sending.js:327-335`. A batch returning 409 or repeated 503 ends in QStash's DLQ; its messages stay `queued`, the campaign never settles. Admins are alerted, nothing recovers automatically. **Fix:** cron step re-publishing `queued` messages of `sending` campaigns idle > X hours, or pausing them with a reason.

### LOGIC-09 — Reserved plan room can leak; refunds can go negative (Low)

**Where:** `lib/sending.js:984-985` (claim outside the `try` starting at `:990`), `:1122`; `lib/usage.js:28-48`. A throw before the `try` or a function timeout consumes the reservation without sending, and the retry claims again. If `rollUsagePeriod` resets `emailsSent` between claim and refund, the refund drives the new month negative. **Fix:** move the claim inside the `try`; make refunds conditional on the period (`periodEnd` filter).

### LOGIC-10 — Pending contacts can't unsubscribe (Low)

**Where:** `app/api/unsubscribe/route.js:11`, `:42-56`, `:97`. Confirmation emails carry unsubscribe links and `List-Unsubscribe`, but `unsubscribe()` only moves `active`/`dormant`. Someone signed up by a third party who clicks unsubscribe changes nothing (and can receive up to 3 confirmations), yet sees "done". **Fix:** include `pending`, or add a per-newsletter `manual` suppression.

### LOGIC-11 — Data surviving purge; unsubscribe links tied to customer domains (Low)

**Where:** `lib/account-purge.js:37`, `lib/archive-hosts.js:14-21`. `Segment` isn't in `CONTENT_MODELS`; segment conditions can hold personal values (e.g. `email equals …`) and survive the purge. Unsubscribe/confirm/tracking links are built on `newsletter.archiveHost`; if the customer removes or repoints the custom domain, unsubscribe links in already-sent emails break before the 30-day CAN-SPAM window ends. **Fix:** add `Segment` to the purge plus a test asserting every `accountScoped` model is covered; keep `List-Unsubscribe` on the app/shared domain.

### LOGIC-12 — No tracking opt-out; content check fails open (Info)

**Where:** `lib/tracking.js`, `lib/sending.js:210`, `lib/content-review.js:439-449`. Every campaign/welcome/test email gets the pixel and link rewriting, with no per-newsletter switch (relevant for ePrivacy-minded EU senders). When the classifier errors, the issue goes out unchecked (documented, admins alerted). **Fix:** per-newsletter tracking toggle; hold sends on check errors for accounts younger than N days.
