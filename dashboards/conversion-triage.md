# Conversion Triage — PostHog Dashboard

**Live dashboard:** https://us.posthog.com/project/264212/dashboard/1534942

Pinned in the Slingshot Biosciences PostHog project. Each tile maps to one or more items in [`/CONVERSION_TRIAGE.md`](../CONVERSION_TRIAGE.md).

## Tiles

| # | Tile | What it shows | Triage item it monitors |
|---|---|---|---|
| 1 | [Pageviews & users by host (last 7d)](https://us.posthog.com/project/264212/insights/XxvelVUU) | Event volume grouped by domain. Surfaces domain redirects / DNS issues. | P1 #7 |
| 2 | [Daily checkout funnel — last 14 days](https://us.posthog.com/project/264212/insights/VUIEQA7n) | 6-step Shopify funnel: started → contact → address → shipping → payment → purchase. | P2 #8 |
| 3 | [Purchases per day (last 30d)](https://us.posthog.com/project/264212/insights/wAjBISEv) | Trip-wire trend with checkouts started overlay. | P1 #4 |
| 4 | [`$exception` count by day vs 7d baseline](https://us.posthog.com/project/264212/insights/S9iQlCZZ) | Spike detector. Apr 30 hit 405 (2.5× baseline). | P1 #4, P0 #3 |
| 5 | [`/shop` → store handoff conversion](https://us.posthog.com/project/264212/insights/SOHbiByb) | Funnel from `/shop` on the marketing site to any pageview on `checkout.slingshotbio.com`. | P0 #2 |
| 6 | [Top error pages (last 7d)](https://us.posthog.com/project/264212/insights/Q7A8qwke) | Error-throwing URLs broken down by browser and device. | P0 #3, P2 #10 |
| 7 | [Shopify alerts shown to users (last 14d)](https://us.posthog.com/project/264212/insights/2QOLBk2g) | `shopify_alert_displayed` count + DAU. Leading indicator of checkout friction. | P2 #9 |

## Active alerts

Each alert delivers via email to `josh.haydon@slingshotbio.com` **and** posts to Slack `#website-visitors` (channel id `C09P9UDAC15`, workspace `Slingshot Bio`, integration id `168970`) through a PostHog internal-destination CDP function.

| Alert | Insight | Trigger | Cadence | Slack destination |
|---|---|---|---|---|
| **No purchases in 24h** (id `019de5f3-bcc2-…`) | [Purchases per day](https://us.posthog.com/project/264212/insights/wAjBISEv) | series 0 (Purchases) drops below 1 | daily | CDP fn `019de5fa-ca70-…` |
| **Exception spike (z-score, 14d window)** (id `019de5f4-0290-…`) | [`$exception` count](https://us.posthog.com/project/264212/insights/S9iQlCZZ) | z-score detector, threshold 0.95, 14-day rolling window on series 0 (Exceptions) | daily | CDP fn `019de5fa-d48a-…` |

> **One-time setup needed in Slack:** invite the PostHog bot to `#website-visitors` with `/invite @PostHog`. Until that's done, the message-post will fail with `not_in_channel`. The PostHog UI surfaces these errors under Pipeline → Internal destinations.

## Refreshing

The dashboard refreshes on view. To force a recompute of every tile, click *Refresh* in the dashboard header (or call `dashboard-insights-run` with `id: 1534942`).
