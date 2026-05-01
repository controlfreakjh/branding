# Conversion Triage Board

Last refreshed: **2026-05-01** (PostHog project: Slingshot Biosciences, id `264212`)

A live punch-list of issues blocking purchases on slingshotbio.com / checkout.slingshotbio.com. Each item has the supporting data point, the hypothesis, the fix, and who/what should verify it.

## Status legend
- **P0** – revenue is leaking right now, fix today
- **P1** – visibility/process gap, will cause future incidents to go undetected
- **P2** – conversion-rate optimization, work after P0/P1 are clear

---

## P0 — Revenue leaking right now

### 1. Paid ads still send traffic to the retired `shop.slingshotbio.com`
- **Signal:** Google Ads campaign `23502834626` ("SO – Display – Catalogue") drove a Safari/Mac user to `shop.slingshotbio.com/collections/viability-controls?gad_campaignid=23502834626…`. That single landing produced **193 `$exception` events from 1 user** on Apr 30 — the user kept retrying.
- **Root cause hypothesis:** the `shop.` → `checkout.` redirect either doesn't exist for that path, or returns a non-2xx that the browser surfaces as a JS exception loop.
- **Fix:**
  1. Audit every active Google Ads / Bing Ads / paid-social destination URL. Replace `shop.slingshotbio.com` with `checkout.slingshotbio.com` (or the new canonical store URL).
  2. Add a permanent 301 from `shop.slingshotbio.com/*` → `checkout.slingshotbio.com/*` so deep links from the wild (cached SERPs, old emails, indexed pages) resolve cleanly.
  3. Verify with `curl -I https://shop.slingshotbio.com/collections/viability-controls` — must return `301` and a `Location:` header pointing at the new domain.
- **Verify:** PostHog daily exception count for `properties.$host = 'shop.slingshotbio.com'` should drop to ~0 within 24h.

### 2. `/shop` page on the marketing site is leaking 84% of users
- **Signal:** `www.slingshotbio.com/shop` got **177 pageviews from 58 users** in the last 48h. Only **~9** of those reached `checkout.slingshotbio.com` in the same window.
- **Root cause hypothesis:** the "Shop / Buy now / Add to cart" CTAs on `/shop` may still point to `shop.slingshotbio.com`, or the redirect adds enough latency / breakage that users bail.
- **Fix:**
  1. Audit every link on `/shop` (and any product cards on `/`, `/products/*`, `/collections/*`). Repoint to `checkout.slingshotbio.com`.
  2. Click-test on Chrome desktop, Safari iOS, and Mobile Chrome — the top three browser/device combos in the data.
- **Verify:** in PostHog, the conversion rate `pageview on /shop` → `pageview on checkout.slingshotbio.com` within the same session should rise from ~15% back toward 50–70%.

### 3. Top product page `/products/spectracomp` is throwing JS errors
- **Signal:** `/products/spectracomp` is the most-viewed product page (104 pageviews / 47 users in 48h). It logged **TypeErrors on both Chrome/Mac and Chrome/Windows** in last 48h (12 events / 4 users captured; likely undercounted because `properties.exception_*` is empty).
- **Root cause hypothesis:** stale CDN assets (the project also shows `ChunkLoadError`s) from a recent deploy, or a third-party widget (Shopify Buy Button, reviews widget) failing to mount.
- **Fix:**
  1. Open the page in production with devtools — capture the actual stack trace.
  2. If `ChunkLoadError`: invalidate the Vercel/CDN cache and verify hashed asset filenames match the latest build.
  3. If a vendor script: gate it behind a try/catch and an error boundary so a broken vendor can't kill the page.
- **Verify:** PostHog Error Tracking → filter to `$current_url contains '/products/spectracomp'` → daily error count should fall to <5.

---

## P1 — Visibility & process gaps

### 4. We had ~48h with zero purchases before noticing
- **Signal:** Apr 30 + May 1: zero `purchase`, zero `shopify_checkout_started` on `checkout.` until investigated manually.
- **Fix:** add two PostHog alerts:
  - **No-revenue alert:** `count(purchase) = 0` over a rolling 24h window → notify Slack #web-ops.
  - **Anomaly alert:** `count($exception) > 2× rolling 7d avg` → notify Slack.
- **Owner suggestion:** web/eng on-call. Implement via PostHog → Insights → Alerts.

### 5. `$exception` events have empty type / message / fingerprint properties
- **Signal:** A HogQL query against `properties.$exception_type` and `properties.exception_fingerprint` returned all-null in the last 48h, even though PostHog's Error Tracking UI shows real types (TypeError, ReferenceError, ChunkLoadError).
- **Impact:** we can't slice errors by message in HogQL or build alerts on specific error families.
- **Fix:** confirm the PostHog JS SDK config — likely `capture_exceptions` is on but the property mapping isn't being persisted. Check whether the SDK version is current; PostHog moved fingerprint into `$exception_fingerprint` (with `$` prefix) in recent releases.
- **Verify:** after deploy, run `SELECT properties.$exception_type, count() FROM events WHERE event = '$exception' AND timestamp >= now() - INTERVAL 1 HOUR GROUP BY 1` — should return populated rows.

### 6. QA/internal traffic is polluting funnel data
- **Signal:** Apr 26 had 62 `survey shown` events from 34 users on `checkout.slingshotbio.com`; Apr 27 had 107. That's not normal customer behavior — it's the team clicking around.
- **Fix:**
  1. Create an "Internal users" cohort in PostHog (by email domain `@slingshotbio.com`, by IP, or by a `localStorage.is_internal` flag set on staff devices).
  2. Default every funnel and conversion dashboard to `filterTestAccounts: true` and exclude that cohort.
- **Verify:** baseline conversion numbers should drop slightly but become stable week-over-week.

### 7. No dashboard tile for "pageviews / events by host"
- **Signal:** the domain change wasn't reflected in any visible chart, so the disappearance of `shop.` events read as "checkout broken" instead of "shop redirected."
- **Fix:** add a tile to the main analytics dashboard:
  ```sql
  SELECT domain(properties.$current_url) AS host, count() AS events, uniqExact(person_id) AS users
  FROM events WHERE timestamp >= now() - INTERVAL 7 DAY
  GROUP BY host ORDER BY events DESC
  ```

---

## P2 — Conversion-rate optimization (after P0/P1)

### 8. 75% drop at checkout_started → contact_info_submitted
- **Signal (7d baseline before incident):** funnel was 20 → 5 → 5 → 2 → 2 → 2 (10% end-to-end). The biggest single-step loss is step 1→2 (Started → Contact Info), at 75%.
- **Hypotheses:**
  - Required-field UX: too many required fields, or unclear validation messages.
  - Guest checkout disabled (forcing account creation kills conversion).
  - Shipping calculator or Shopify Payments not loading on first paint, so users see a half-rendered checkout and bail.
- **Fix experiments to run** (one at a time, A/B via PostHog Experiments):
  1. Enable Shopify guest checkout if it's not already.
  2. Reduce required fields to: email, name, address, payment.
  3. Add an "Express checkout" button (Apple Pay / Shop Pay / Google Pay) at the top of the checkout page.

### 9. Long tail on contact-info submission time
- **Signal:** average time to submit contact info is 1,395 seconds; median is 19 seconds. So most users are quick, but a long tail sits stuck for ~23 minutes.
- **Hypothesis:** these users hit a validation error they don't understand and walk away, or open a new tab to find a coupon.
- **Fix:**
  1. Add inline validation hints (especially for international postal codes / VAT IDs since this is a B2B life-sciences shop).
  2. Capture `shopify_alert_displayed` events into a dashboard tile — Apr 28 had 21 alerts from just 2 users (10 alerts/user is a clear sign of struggle).
- **Verify:** session recordings on those long-tail sessions; PostHog has session replay for sessions with `shopify_alert_displayed`.

### 10. Marketing site has 423 active error-tracking issues
- **Signal:** PostHog Error Tracking shows 423 active issues, mostly `TypeError` and `ChunkLoadError` going back to late Dec 2025.
- **Fix (workstream, not single ticket):**
  1. Triage the top 20 by event volume × users affected.
  2. Group `ChunkLoadError`s separately — almost always a CDN / cache-busting problem at deploy time. Add a build step that purges the CDN and a reload-on-chunk-error boundary in the SPA.
  3. Assign owners per issue family.

---

## Tracking

When an item is fixed:
1. Move it to a "Closed" section at the bottom with the PR / change link.
2. Note the PostHog metric that confirmed the fix and the date.

## Open data questions
- Are the spike days (Apr 26–27) inflated by QA traffic, or did they include real customer purchases? Need a cohort filter to know.
- What share of `/shop` visitors come from organic vs. paid? If paid, the ad-URL fix above will recover them; if organic, the `/shop` link audit is more urgent.
- Are there any campaigns paused that should be re-enabled once `shop.` → `checkout.` redirects are healthy?
