# Redfin scraper API: is there an official endpoint, what does a scraping API return, and why ~350 homes per region is the ceiling (plans, credit costs, and the routes Redfin serves data from)

Redfin does not publish a public API for individual listings. Its Data Center publishes aggregate market statistics, and the Data Center API runs a waitlist aimed at enterprise users. If you need per-property rows, you are choosing between a scraper that already hands you structured fields and a general scraping API where you write the parser. Either way you run into the same ceiling: a single Redfin region returns roughly 350 homes, and page two repeats page one.

ScraperAPI sits in the second category, with one important exception: it ships a structured Redfin listing-search endpoint that takes a Redfin search-page URL and returns JSON, so the first layer of parsing is already done. Everything else on Redfin (detail pages, the CSV map endpoint, rentals) goes through its general proxy and anti-bot layer, and you own the parsing.

> **Quick take:** Hobby (100,000 credits/month, 20 concurrent threads, US & EU geotargeting) is enough to prototype a Redfin pipeline; Startup (1,000,000 credits, 50 threads) covers a daily refresh across a few dozen ZIPs. The 7-day trial gives you 5,000 credits with no card. 👉 [See ScraperAPI's current plans and prices](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Every ScraperAPI plan, side by side

Credits are the unit that matters, and they are not uniform. A standard page bills 1 credit. Bot-protection bypass (Cloudflare, DataDome, PerimeterX/HUMAN, Akamai) bills 10, JS rendering bills 10, and premium proxies plus rendering bills 25. Geotargeting, sessions, device type and auto-parsing cost nothing extra, which is good news for Redfin since you need a US exit but not a premium proxy surcharge on every call.

| Plan | Credits per month | Concurrent threads | Geotargeting | Price and billing | Start here |
| --- | --- | --- | --- | --- | --- |
| Free trial | 5,000 (7 days) | — | — | $0, no credit card | [Start the trial](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | Listed on the pricing page | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | Listed on the pricing page | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | Listed on the pricing page | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | $475/mo, or $427.50/mo billed annually | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | Listed on the pricing page | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | Listed on the pricing page | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Custom quote | [Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Annual billing takes 10% off every tier, and the Scaling figure shows the math: $427.50 is exactly 10% under $475. Three billing rules decide how a Redfin job behaves at month end. Credits reset at renewal and do not roll over on any plan. Auto-upgrade, which moves you up a tier when credits run out, is opt-in on Hobby through Professional. Pay-as-you-go, which keeps requests running at a fixed per-credit rate up to a cap you set, starts at Scaling. 👉 [Check the free trial terms and current per-plan prices](https://www.scraperapi.com/pricing?fp_ref=coupons)

Before you budget, price your actual Redfin URLs in the dashboard's Domain Cost Estimator, and set `max_cost` on a scrape so one hard page cannot quietly eat a day of credits. Whether Redfin bills at 1 credit or 10 depends on whether the request needs bot-protection bypass, and that can change.

## What you get for Redfin without writing a parser

The structured endpoint is the reason this keyword and this vendor overlap. You POST to `async.scraperapi.com/structured/redfin/search` with your API key, a Redfin search-page URL, and optional `country_code` and `tld`. It accepts a single URL or an array of URLs, and it takes a webhook callback, so the job posts its results to you instead of making you poll. The response to the initial call is a job record with an id, a status and a `statusUrl` you can check.

The constraint to design around: the input URL must be a Redfin search page. You cannot hand it a single home URL and get that property back. The docs' own sample uses a filtered city URL with `min-beds` and `min-baths` in the path, which is also a useful trick for shrinking an oversized region below the 350-home cap.

A second structured endpoint covers Redfin agents rather than listings. It returns agent name, license number, brokerage, contact number, languages, service areas, sales totals (deals, volume, highest price) and the agent's current listings with price, beds, baths, square footage and address. Useful if your comps analysis needs to know who listed what.

Anything outside those two endpoints, you fetch raw and parse yourself, through the same key and the same US geotargeting.

## The four routes Redfin serves data from

Redfin has no single feed. Depending on what you need, you are hitting one of these, and each returns a different shape.

**1. The `gis-csv` endpoint.** Redfin's map search is powered by a `gis-csv` endpoint that returns listing data as CSV. You resolve the region id from a Redfin URL, then call the endpoint with a browser-like fingerprint. Redfin's location-autocomplete endpoint is blocked by its own CDN, which is why every working tool asks you to paste a region URL instead of typing a city name.

**2. The search JSON API.** The response body arrives prefixed with `{}&&`. Strip that prefix, run it through a JSON parser, and the homes sit under `payload.homes`. Forget the prefix and your parser throws on a response that is perfectly valid data.

**3. Property detail pages, which are HTML only.** There is no JSON behind a listing page you can rely on, so the fields come from XPath and CSS. Price sits in `//div[@data-rf-test-id='abp-price']/div/text()`, the street address under a `street-address` class, the description inside `#marketing-remarks-scroll`, key details in `div .keyDetails-value`, and feature groups under `.amenity-group ul div.title`. This is the layer that breaks when Redfin redesigns, and it is why page-by-page scraping costs more maintenance than region search.

**4. Rentals, via a separate API.** Pull the 36-character `rental_id` out of the page's `og:image` meta tag, then request `https://www.redfin.com/stingray/api/v1/rentals/{rental_id}/floorPlans`. The ID length is the validation check; anything else means the property is not a rental.

One more gotcha lives in the sort parameter. Redfin's GIS rejects `newest` and `days-on-market-asc` outright with an input error. Supported orders are recommended, price ascending or descending, square footage descending, and lot size descending.

## The 350-home ceiling and the chain reaction it causes

Every serious Redfin tool documents the same number: roughly 350 homes per region per request. The open-source `redfin-data-scraper` defaults its `--limit` to 350 and says Redfin caps around there; an Apify actor states each region returns up to 350 distinct homes. Treat it as a hard wall rather than a default you can raise.

Pagination does not get you past it. The newer actor documentation is explicit that Redfin repeats the same homes on subsequent page numbers, so raising `maxPages` produces duplicates, not inventory. (A different actor exposes a 50-page pagination setting with a safety limit, but its own documentation concedes that actual coverage depends on Redfin. The repetition behaviour is the more reliable description of what the source does.)

What actually works is splitting the geography. Submit multiple region URLs in one run, one per ZIP or neighbourhood, and raise the total item cap. A metro with 40 ZIPs at 350 homes each gets you 14,000 rows before you hit any single-region limit. When one dense ZIP still blows past 350, narrow it with a Redfin filter URL (beds, baths, price) so each variant returns a different slice rather than the same first page.

Three data-quality rules follow from how Redfin works, and they matter more than field counts:

- **Sold listings run on a 90-day window.** Ask for sold data and you get recent sales, not history. Redfin also controls what it exposes and may omit the sale price on some records, so a comps table built from sold rows can have holes.
- **Local MLS rules hide some inventory.** Certain listings simply do not appear in downloadable feeds for licensing reasons, and no scraping layer fixes that.
- **Missing values stay missing.** A well-built scraper returns `null` for an absent feature rather than a zero or a guess, and Redfin sometimes truncates listing remarks, so the description you get may be shorter than the one on the page.

## Dedicated Redfin scraper or general scraping API?

This is the actual decision, and price is the least interesting part of it.

| Option | What you get | Verified cost | Where it hurts |
| --- | --- | --- | --- |
| Apify `s-r/redfin-scraper` | 38 original columns plus normalized aliases (`postal_code`, `lat`, `lng`, `living_area`), for-sale or sold, multiple regions per run | $0.002 per run start + $0.0003 per delivered home; free Apify plans cap runs at 10 homes; zero-result runs still pay the start fee | Region search results only; it does not open individual home pages |
| Apify `automation-lab/redfin-scraper` | 20+ fields, price/beds/baths/sqft filters, 50-page setting | $0.001 per run start, plus Apify platform usage | Coverage of pending and sold listings depends on Redfin's GIS |
| `redfin-data-scraper` (open source) | CLI and Python library, CSV/JSON/JSONL, no API key | Free, MIT licence, Python 3.10+, `requests` only | You handle blocking, retries and any change to Redfin's protection |
| ScraperAPI | Structured Redfin search endpoint plus a general proxy and anti-bot layer | Credits: 1 standard, 10 bypass, 10 JS rendering | You parse detail pages, `gis-csv` and rentals yourself |

Run the arithmetic on the per-home pricing: 350 homes at $0.0003 comes to about $0.105, plus $0.002 per run start. Cheap. But the field list is the real product. A scraper that returns 38 named columns has already solved the parsing, the null handling and the unit conventions (prices in USD, areas in square feet, `price_per_sqft` nulled out when either input is missing or zero). A general API gives you the response and nothing else.

So: if your target is a handful of ZIPs and you want comps tables this week, a dedicated actor or the open-source library is less work. If you already have a parsing pipeline, or you need detail pages, rental floor plans and agent data under one key with one US exit and one billing line, the general layer wins on control.

## Compliance, briefly

Tool vendors describe scraping publicly available listing data as generally lawful in the US, citing the Ninth Circuit's 2022 hiQ v. LinkedIn decision, and describe their own guardrails: public pages only, no logins, no agent-only areas, built-in delays. That is a vendor's position, not a legal opinion you can rely on. Redfin's terms of service and your local rules are the documents that decide your case, and none of the tools above changes that.

## How to pick a plan and what to check before you pay

Start from request volume, not from homes. Each Redfin search-page call is one request that bills 1 credit, or 10 if it needs bot-protection bypass. A daily refresh of 40 ZIPs is 40 requests a day, roughly 1,200 a month, which does not come close to Hobby's 100,000 credits even at the 10-credit rate. Detail-page scraping changes the math fast: 5,000 homes a day at 10 credits each is 1.5 million credits a month, which puts you on Business or above.

Four things to confirm before checkout:

1. **Region URLs, not city names.** Collect the actual Redfin URLs for each ZIP or neighbourhood you need. The autocomplete route is blocked.
2. **Your sold-data assumptions.** If your model needs more than 90 days of sales, plan a second data source, because the sold window will not stretch.
3. **Concurrency against your crawl rate.** Hobby's 20 threads is plenty for scheduled ZIP sweeps and thin for detail-page crawls at scale.
4. **The renewal mechanics.** Credits reset, nothing rolls over, and pay-as-you-go only exists from Scaling upward.

For most Redfin projects the sequence is trial, then Hobby for a month of real volume data, then a jump to whichever tier your measured credit burn supports. 👉 [Compare ScraperAPI plans and start with 5,000 free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)
