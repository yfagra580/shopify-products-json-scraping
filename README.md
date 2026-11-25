# Shopify scraper: how `/products.json` really works, where the 250-per-page crawl stops, and what you pay per 1,000 products

A Shopify store will hand you its whole catalog as JSON from `https://store.com/products.json?limit=250&page=1`. No API key, no app install, no store permission. One response carries up to 250 products with their variants, SKUs, prices and stock flags.

The hard part starts on page 2. Pagination has a ceiling, HTTP 429 is a routine answer, and plenty of stores serve the catalog route while returning an empty array on the collection route. Below is what actually breaks, the sitemap step most tutorials skip, and where ScraperAPI fits in the loop with the credit arithmetic done out.

> **If you would rather not build the fetch layer:** ScraperAPI's paid entry tier is listed at **$49/month for 100,000 credits and 20 concurrent threads**, and a standard Shopify catalog request bills as **1 credit**. Test your store list on the free 1,000 credits before you pay for anything.
> 👉 [Check ScraperAPI's current plans and credit allowances](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## ScraperAPI plans: per-request billing, not per-product

ScraperAPI charges one credit for a standard page request and adjusts the price for domains it has built custom handling for. That model suits Shopify unusually well, because a single request returns 250 products rather than one.

| Plan | Credits per month | Concurrent threads | Price | Start |
| --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0, no card | [Open the free plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | $49/month | [Start on Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup, Business, Scaling, Professional, Advanced, Enterprise | Larger allowances, listed per tier | Higher thread counts, listed per tier | Listed on the pricing page | [Compare the higher tiers](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two things about the upper tiers matter before you buy: opt-in auto-upgrade (move up a plan automatically when credits run out) is available on Hobby through Professional, and pay-as-you-go overflow only appears from Scaling upward. On every other plan, scraping simply pauses when credits run out until they renew.

Credit cost per request, as published:

| Request type | Credits |
| --- | --- |
| Standard page, including a Shopify `/products.json` call | 1 |
| Amazon | 5 |
| Google and Bing, including subdomains | 25 |
| LinkedIn | 30 |
| Any site where Cloudflare, DataDome or PerimeterX has to be bypassed | +10 on top of the base cost |

The dashboard's Domain Cost Estimator gives the exact figure for a specific URL, and the `max_cost` parameter caps what a single scrape is allowed to spend. Run your three or four hardest target stores through the estimator before you size a plan.

### What that works out to per product

At $49 for 100,000 credits, one credit costs about $0.00049. A catalog page carrying 250 products therefore costs roughly five hundredths of a cent, and a 5,000-product store costs about 20 credits to walk end to end. The same 5,000 products at a per-product price of $1.55 per 1,000 (the Apify actor further down) come to $7.75.

That gap closes fast if your job needs per-product requests. Sitemap reconciliation on a store that hit the pagination ceiling means one request per missed handle, so a 20,000-product store with a dead catalog route turns into 20,000 credits instead of 80. Per-request billing rewards you for using the bulk feed and punishes you for falling back to individual product calls.

## Four things that break a Shopify crawl, and one that only looks broken

### 250 products per page is the maximum the route accepts

`limit=250` is the largest page size the storefront `products.json` route will serve. There is no `limit=500`. Plan your loop around 250 and treat a short page as the tail of the catalog.

### The pagination ceiling is real, the number is not agreed on

Different tools report different walls. One Apify actor paginates `/products.json?limit=250&page=N` up to page 50 and calls that Shopify's hard limit of 12,500 products; when it hits the wall it switches to sitemap reconciliation. A widely used open-source Python scraper defaults to `--max-pages 100`, which is 25,000 products, and skips everything past that without complaint.

Both agree on the shape of the problem: at some depth the catalog feed stops being a complete inventory. Assume your crawl can be truncated, and log whether it stopped because a page came back short or empty, or because it hit your own configured maximum. Those are different outcomes and only one of them means you have the whole store.

### 429 is the normal failure, not a blocked IP

Shopify rate limits bursts. A working scraper paces its pagination about one second per page and retries a bounded number of times with exponential backoff; Scrapfly's implementation uses a 1.0-second delay, three retries, and a backoff of `delay × 2^(attempt+1)`. Sequential, paced pagination is also forced by the payload: it reports no page count, so the only stop signal is a short, empty or failed page.

ScraperAPI introduces a second, unrelated 429. That one means you exceeded your plan's concurrency, and the request is refused rather than queued, with no extra charge for the refusal. Two different 429s with two different fixes: one needs a slower loop, the other needs a higher thread limit.

### An empty collection feed does not mean an empty collection

`/collections.json` and `/collections/{handle}/products.json` are store-dependent. A store that serves `/products.json` perfectly can still return an empty list from the collection route. Treat an empty result as an unusable route, not as a category with no products, or you will silently drop an entire category from your dataset while the run reports success.

### Prices come back in cents

The raw endpoints return integers. The `/variants/{id}.js` response carries the price in cents, and so does the catalog feed. Tools that advertise "real decimal prices, not cents you have to divide by 100" are describing post-processing, not the source data. Pick one convention, apply it at ingest, and check it against a product you can see in a browser. A unit error here scales every comparison by 100.

### A dead route does not prove the store is not Shopify

If `/products.json` returns an HTML page, a login wall or a 403, that tells you the route did not answer this time. It does not tell you the store is off Shopify. For a quick yes/no, request `https://thestore.com/admin` and see whether it lands on a Shopify login screen.

## The sitemap step that makes big catalogs complete

When the catalog feed truncates, the product sitemap is the inventory of record. Four details decide whether your reconciliation works or silently returns nothing.

1. **Read `/sitemap.xml` first.** Shopify links its product sitemaps from the index with an id-range query. Requesting a numbered sitemap path directly answers with a 400.
2. **Parse it as XML with the namespace declared.** Sitemaps are namespaced XML, and current versions of parsel parse them as XML, where a bare `loc` XPath matches nothing. Use `http://www.sitemaps.org/schemas/sitemap/0.9`.
3. **Filter for product sitemaps.** Keep only the `loc` entries containing `sitemap_products`, and cap the number you follow so a store with dozens of sitemaps does not blow up the run.
4. **Fetch missed products by handle.** For each handle the pagination loop did not cover, request `/products/{handle}.json` and merge it into the dataset.

If the catalog JSON route is unusable altogether, the last resort is the product pages themselves: pull the JSON-LD blocks (`schema.org/Product`) out of the HTML. Watch the `offers` field there, which may be a single object or an array of variant offers depending on the theme.

## What ScraperAPI removes from this job

ScraperAPI fetches the URL you give it, rotates proxies, absorbs block pages and retries. Point it at a store's `/products.json` and you get the JSON body back, which for Shopify means you can skip the HTML-parsing layer entirely.

What it does not do is paginate for you. The loop, the stop condition, the 429 backoff on Shopify's side and the sitemap reconciliation are still your code. There is no Shopify-specific structured endpoint to call: the autoparse example in ScraperAPI's documentation is an Amazon product record, and a Shopify catalog feed does not need one because the response is already structured JSON with variants, prices and availability intact.

The parts it genuinely takes off your plate:

- **Per-store blocking.** Stores behind Cloudflare, DataDome or PerimeterX cost 11 credits per request instead of 1, and the bypass is handled. On a list of a few hundred stores that is the difference between a run that finishes and one that dies at store 40.
- **Concurrency.** 20 threads on Hobby against the roughly one-request-per-second pace Shopify tolerates means you can run several stores in parallel without each one crawling at a crawl.
- **Cost control.** `max_cost` per scrape, plus the Domain Cost Estimator, lets you price the exact list before committing.

The honest limit: none of this unlocks inventory counts or customer data. The public endpoints expose availability as a true/false flag per variant. Inventory depth and order data live behind the authenticated Admin API. If your project needs stock levels, no scraping API sells you that.

## Free script, Apify actors, or a scraping API

| Option | Model | Verified price | What to know |
| --- | --- | --- | --- |
| Open-source `shopify-store-scraper` (Python) | MIT, single file, standard library only | Free, no paid tiers | Defaults to 100 pages × 250 = 25,000 products per store and skips the rest; includes `/--check` to validate a URL list without downloading |
| Apify "Shopify Scraper" (automly) | Pay per product | $1.55 per 1,000 products, plus $0.05 per run at default settings; $5 monthly free usage covers about 3,200 products | Decimal prices, price and in-stock filters, schedules, exports to CSV, Excel, JSON, and pushes to Sheets, Make, Zapier, n8n |
| Apify "Shopify Product Scraper" (scrape_pad) | Actor, ISC licence | Not verified | Page-50 cap plus sitemap reconciliation, `curl_cffi` browser impersonation, writes a `SUMMARY` record with a `paginationCapHit` flag |
| ScraperAPI | Pay per request | $49/month for 100,000 credits, 1 credit per catalog page | You write the loop; the API handles proxies, blocks and concurrency |

The Apify run summary is worth copying regardless of which tool you use. A flag that tells you whether the pagination cap was hit is the cheapest safeguard against shipping a truncated catalog as if it were complete.

## Which one fits your job

**A one-off export of a small store.** The free Python script, or the free 1,000 ScraperAPI credits. Either finishes a 2,000-product catalog in minutes, and you should not be paying a subscription for it.

**Scheduled price monitoring across a few hundred stores.** Per-request billing wins here, because each request carries 250 products. 👉 [Size a ScraperAPI plan against your store list](https://www.scraperapi.com/pricing/?fp_ref=coupons) and count catalog pages, not products.

**A single store with a huge catalog.** Whatever you pick must reconcile sitemaps and must tell you when it truncated. A tool that caps at 12,500 or 25,000 products and reports success is worse than no tool.

**Live price and stock for one variant.** One request to `/variants/{id}.js`, price in cents, done. No crawler required.

## Before you pay

Three things to settle with the pricing page and your own list, in this order:

1. **Run your target URLs through the Domain Cost Estimator.** A list that is mostly protected stores bills at 11 credits per request, so a 100,000-credit plan covers roughly 9,000 catalog pages, not 100,000.
2. **Decide what should happen at zero.** Scraping pauses by default when credits run out. Auto-upgrade is opt-in on Hobby through Professional; pay-as-you-go overflow starts at Scaling. If you would rather be throttled than upgraded, leave both off.
3. **Check the renewal mechanics before an annual commitment.** Annual plans can renew early for a fresh batch of credits, and credits renew on your billing cycle. Confirm whether unused credits carry over and what the cancellation terms are, rather than assuming either way.

Start on the free credits with five or six of your hardest stores. If the JSON comes back and the page count matches what you expect from the storefront, 👉 [pick the ScraperAPI tier that matches your monthly page count](https://www.scraperapi.com/pricing/?fp_ref=coupons) and stop thinking about proxies.
