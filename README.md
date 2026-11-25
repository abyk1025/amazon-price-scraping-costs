# Scrape Amazon Prices: DIY Selenium or a Paid API, What 20,000 Product Pages Cost, and Which ScraperAPI Plan You Need (Full Plan Table + the 4 CSV Fields That Break Price History)

Scrape Amazon prices once, for a few hundred products, and a Selenium script plus a CSV file is genuinely enough. Do it every day for thousands of ASINs and the script becomes the expensive part, because you spend your time repairing selectors and retry logic instead of using the data. ScraperAPI prices Amazon requests at **5 credits each**, its trial hands you 5,000 credits (about 1,000 Amazon product pages) without a card, and the Hobby plan's 100,000 monthly credits work out to roughly **20,000 Amazon pages a month**.

Below: the full plan table with credit-to-request math, the three ways people actually pull Amazon prices and where each one breaks, why the standard four-column CSV quietly ruins your price history, and the settings that decide whether you get billed for a failed request.

> **Quick pick.** Testing an idea: the 7-day trial, 5,000 credits, no credit card, roughly 1,000 Amazon product pages. Running it in production: Hobby, 100,000 credits a month, about 20,000 Amazon pages, 20 concurrent threads. Past ~20,000 Amazon pages a month, you're into Startup.

👉 [Check ScraperAPI's current plan rates and start the free trial](https://www.scraperapi.com/?fp_ref=coupons)

## All ScraperAPI plans, with the Amazon request math done for you

ScraperAPI quotes its plan rates on the live pricing page rather than in the docs, so treat the credit allowances below as the fixed part and the dollar figure as something you confirm on the page itself. The right-hand column is what actually decides your plan, since an Amazon request is not one credit.

| Plan | API credits / month | ≈ Amazon product pages | Concurrent threads | Overage handling | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | ~200 | 5 | Scraping pauses until credits renew | [Free plan details](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | ~20,000 | 20 | Auto-upgrade (opt-in); no pay-as-you-go | [Hobby plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 | ~200,000 | 50 | Auto-upgrade (opt-in); no pay-as-you-go | [Startup plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 | ~600,000 | 100 | Auto-upgrade (opt-in); no pay-as-you-go | [Business plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | 5,000,000 | ~1,000,000 | 200 | Auto-upgrade + pay-as-you-go | [Scaling plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | 10,500,000 | ~2,100,000 | 300 | Auto-upgrade + pay-as-you-go | [Professional plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | 21,500,000 | ~4,300,000 | 500 | Pay-as-you-go only | [Advanced plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | ~4,400,000+ | 500+ | Pay-as-you-go; custom annual terms | [Enterprise terms](https://www.scraperapi.com/?fp_ref=coupons) |

Two features split the table in half. Geotargeting is US & EU only on Free, Hobby and Startup, and country-level from Business upward. Analytics history is capped at the last 30 days on Free, Hobby and Startup, and unlimited from Business upward. If you're building a price-history product rather than a weekly spot check, that 30-day ceiling matters more than the credit count.

Every paid tier takes a **10% annual discount**; Enterprise is quoted custom. Credits do not roll over on any plan, including annual ones, so an oversized plan is money you donate once a month.

## What a request actually costs, credit by credit

The base unit is one credit for a standard page. Amazon, Walmart and eBay are billed at **5 credits per request**. Google and Bing SERPs are 25, LinkedIn is 30. On top of that:

- Bot-protection bypass (Cloudflare, DataDome, PerimeterX, Akamai) adds 10 credits.
- `render=true` for JavaScript rendering adds 10.
- `premium=true` proxies add 10, `ultra_premium=true` adds 30.
- Geotargeting, sessions, device type, output format and auto-parsing are free.

Combinations aren't simple addition: the pricing page lists premium proxies plus JS rendering at 25 credits and ultra premium plus rendering at 75, so don't assume 10 + 10. Run your exact URL through the Domain Cost Estimator in the dashboard before you commit to a volume, and set `max_cost` on the request if you want a hard ceiling per scrape.

The failure behaviour is the part most people miss. Blow past your concurrency limit and you get a **429**, the request is refused rather than queued, and you're not charged for it. Your pipeline needs to catch that and back off. A timeout or an anti-bot wall on a request you already paid for is a different story, which is why your monthly credit budget should include a retry allowance on top of your raw ASIN count.

## Three ways to scrape Amazon prices, and where each one hurts

### 1. Your own Selenium script

The reference implementations are well documented, which makes them look easier than they are. A typical Oxylabs Selenium build targets product blocks with `//div[@data-component-type='s-search-result']`, pulls the title from the `span` inside the `h2`, the link from `a.a-link-normal.s-no-outline`, and the price from `span.a-price`. The price itself is assembled from two elements: the integer from `a-price-whole`, the decimals from `a-price-fraction`, joined with a literal `"."` to produce something like `19.99`. The currency symbol comes from a third element, `a-price-symbol`.

That works. It also hardcodes a User-Agent claiming Chrome 90, injects fixed headers through `seleniumwire`, and calls `time.sleep(3)` after each page load before parsing. When the price element is missing, the script logs "Price not found for product ... Likely out of stock" and moves on. Multiply those assumptions by Amazon's variant layouts, coupon badges and business pricing, and you have a parser that needs attention every few weeks. Fine for a one-off pull. Bad as infrastructure.

### 2. A managed scraping tool

Apify's e-commerce actor is the common middle path: you pass `detailsUrls` for product pages, `listingUrls` for category pages, or a `keyword` plus `marketplaces` list, and pick `scrapeMode` as `AUTO`, `BROWSER` or `HTTP`. Billing is pay-per-event, meaning you're charged per product or per review retrieved rather than per platform minute, and the free plan includes a **$5 monthly credit** for testing. It's a reasonable choice if you also want reviews and seller profiles from the same interface.

### 3. An API that returns structured Amazon data

If prices are the whole job, the API route removes the parsing layer. Oxylabs does this with a `source` parameter that switches behaviour: `amazon_bestsellers` needs a category node ID, `amazon_search` takes a keyword, and `amazon` takes a page URL directly. There is no dedicated deals source, so deals pages go through the generic `amazon` source.

Here is the bug you will hit within an hour. The three modes don't return the same shape. Bestsellers come back at `results[0]["content"]["results"]`, while search and deals come back at `results[0]["content"]["results"]["organic"]`. Write one parser for all three and it breaks on two of them.

ScraperAPI's documented Amazon route is ASIN-based instead of URL-based, and it's the one to read closely if you're building a price tracker. The synchronous endpoint is a GET with `api_key`, `asin`, `country_code` and `tld`, plus `output_format` for CSV or JSON (JSON is the default). The async version posts to `async.scraperapi.com/structured/amazon/product` with an `apiKey` and either a single `asin` or an `asins` array, and takes a webhook `callback`, so you can queue a few hundred ASINs and receive results rather than holding connections open.

The TLD list is long: com, co.uk, ca, de, es, fr, ie, it, co.jp, co.za, in, cn, com.sg, com.mx, ae, com.br, nl, com.au, com.tr, sa, se, pl. ASINs are marketplace-specific, and if you need amazon.com pricing as seen from another country, you must send both `tld` and `country_code`.

👉 [Compare the plans that include structured Amazon endpoints](https://www.scraperapi.com/?fp_ref=coupons)

## The four-column CSV problem nobody warns you about

The free scraper tools converge on the same output: `amazon_prices.csv` with exactly four columns, `title`, `url`, `price`, `currency`. That's a clean demo and a broken database.

Three things go wrong. First, out-of-stock items are skipped, not flagged, so a missing row is indistinguishable from a product you never asked about. Second, there's no ASIN and no timestamp, so you can't join today's file to last week's, and your "price history" is a folder of files with no key. Third, `price` arrives as a string built from the whole and fraction elements, with the currency symbol stored separately, so any arithmetic needs a decimal conversion first.

Fix it at write time: derive the ASIN from the product URL, store it as your primary key, stamp every row with a UTC fetch time, keep price as a decimal plus an ISO currency code, and record a status field so "out of stock" survives the trip. Whatever tool you use, decide the out-of-stock rule before you start collecting: carry the last known price with a stale flag, or write a null row. Both work. Silence doesn't.

The structured endpoints close part of that gap for you. ScraperAPI's Amazon product response returns the product name, a `product_information` block with brand, dimensions, weight and similar attributes, variant details for size and color, and the publicly available reviews for that ASIN. That's a different order of magnitude from four columns, and it's the argument for paying per request instead of maintaining a parser.

## Sizing your plan from your own numbers

Work backwards from ASINs. A daily refresh of 5,000 ASINs is 150,000 requests a month, which is 750,000 credits, so Startup (1,000,000) fits with room for retries. A weekly refresh of the same 5,000 is about 21,500 requests a month, roughly 107,000 credits, which puts you just past Hobby. A few hundred ASINs a day sits comfortably inside Hobby.

Then add your retry margin. You know your own timeout rate better than any vendor page does; budget the extra credits explicitly rather than discovering the shortfall on the 28th of the month, because nothing rolls over and Hobby, Startup and Business have no pay-as-you-go. Run dry on those three and scraping pauses until renewal unless auto-upgrade is switched on, at which point you're moved to the next tier automatically. On Scaling, Professional, Advanced and Enterprise you can stay on your plan and keep going at a fixed per-credit rate, capped by a monthly spend limit you set yourself.

## A note on legality

Web scraping legality varies by jurisdiction and depends on the data, the methods and the intended use, so treat any flat "it's legal" claim, including one published by a scraping vendor about its own product, as a vendor position rather than a legal conclusion. Respect Amazon's terms of service, keep personal data out of your pipeline, and get advice if you're reselling the data.

## FAQ

**Can I scrape only a category's best sellers?** With Oxylabs you'd use the `amazon_bestsellers` source with the category node ID from the browse URL, for example `2975359011` for dog food. ScraperAPI's documented Amazon endpoint takes ASINs, so a best-seller or category page goes through as a normal page request and you parse the listing yourself.

**What happens to out-of-stock products?** The DIY implementations detect a missing price element and skip the product. Plan for it: a skipped product should produce an explicit row, not a gap.

**Why do search results and product pages return different JSON shapes?** Because they're different sources behind the same API. Oxylabs returns bestsellers at `content.results` and search or deals at `content.results.organic`. Read the response structure per source before writing a shared parser.

**Can I submit many ASINs at once?** Yes, on the async structured endpoint via the `asins` array plus a webhook callback. It's the difference between a script that runs for an hour and a job that runs in the background.

**Which Amazon marketplaces are covered?** The TLD parameter covers the major ones listed above. If you're scraping a marketplace outside that list, verify before you build on it.

## Which one to pick

Once, under about a thousand ASINs: write the Selenium script, or burn a free credit allowance on a managed tool and skip the selector work entirely.

Daily, thousands of ASINs, prices only: an API with an ASIN-based endpoint. Start on the trial, push 1,000 real Amazon URLs through it, look at what actually came back and what your retry rate was, then choose Hobby or Startup from measured numbers rather than guesses.

Two things to confirm before you pay. Check how you want overages handled, because no pay-as-you-go on Hobby, Startup or Business means a hard stop at renewal unless auto-upgrade is on. And run two or three of your target Amazon URLs through the cost estimator to confirm your TLD and any rendering you need land at the credit rate you budgeted for.

👉 [Start with the 5,000-credit trial and size your plan from real numbers](https://www.scraperapi.com/?fp_ref=coupons)
