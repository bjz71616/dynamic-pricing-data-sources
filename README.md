# Dynamic pricing data: the six inputs that actually move prices, where each one comes from, and how to collect competitor prices at hourly cadence (with the credit math for sizing a scraper)

Dynamic pricing data falls into six buckets: historical transactions and prices, live inventory, competitor prices, behavioural signals like traffic and conversion rate, cost and margin floors, and customer segments. Four of those already live in systems you run. Competitor prices and the market context around them have to be pulled off the open web on a schedule, and that is the row where pricing projects go to die.

ScraperAPI is the collection layer for that row. It returns structured product and pricing data from Amazon, Walmart and Google Shopping as JSON or CSV, handles the proxy and rendering work, and bills by credit rather than by page. The rest of this piece is about the data itself: what each input does to a price, what to guard against, and how to work out what your watchlist will actually cost before you commit to a plan.

> **Quick pick.** Size a plan by credit burn, not by tier name. An Amazon product request costs 5 credits, a Google or Bing request costs 25, a standard page costs 1, and pages behind Cloudflare, DataDome or PerimeterX add 10 credits when they're bypassed. The free tier gives you 1,000 credits and a maximum of 5 concurrent connections, which is 200 Amazon checks or 40 Google queries worth of testing before you pay anything.
> 👉 [Start on the free plan and see what 1,000 credits covers](https://www.scraperapi.com/?fp_ref=coupons)

## The six data sources, and which ones you have to go get

Stripe's breakdown of what a pricing engine consumes is the cleanest one I've found, and it maps neatly onto where each dataset physically lives.

| Data source | Where it lives | Refresh need | Own or fetch |
| --- | --- | --- | --- |
| Historical transactions and prices | Order DB, BI warehouse | Daily, for trend modelling | Own |
| Real-time inventory and supply chain | ERP, WMS | Minutes to hours | Own |
| Competitor prices and market index data | Competitor pages, marketplaces | Hourly or faster | **Fetch** |
| Behavioural signals (site traffic, search volume, conversion rate) | Analytics, Search Console, CDP | Hourly | Own |
| Cost of goods and margin requirements | Finance | Static | Own |
| CRM and segmentation data (only if you personalise) | CRM | Daily | Own |

Rows one, two, four, five and six are an integration problem. You already store the data; the work is unifying it well enough to trust. Stripe flags exactly this as the usual failure point: data quality issues are a common reason dynamic pricing projects stall, and if you can't trust your inputs, the model isn't worth much.

Row three is the one that needs a crawler, a schedule, and a budget that scales with how often you look.

## Four triggers, and the guardrail each one needs

Shopify's breakdown of pricing triggers is useful because it pairs every trigger with a guardrail, which is the part that keeps an automated repricing job from turning into a public relations problem. Here it is with the data signal each trigger reads.

| Trigger | Price moves when | Data you need | Guardrail to set |
| --- | --- | --- | --- |
| Demand | A product goes viral, or sales velocity spikes | Page views, add-to-cart rate, sales velocity, search volume | Ceiling and floor at a set percentage above or below list price |
| Inventory | Stock is either rotting or about to run out | Units remaining, sell-through rate, days of supply | Floor at unit cost plus a minimum margin, lower for clearance |
| Competition | A rival undercuts you | Competitor price for the same SKU | Down only, floored at unit cost |
| Timing | Saturday peak, or 48 hours before an empty slot expires | Day of week, season, time to expiry | Floor above variable cost |

One clarification worth repeating because it changes how you budget for data: dynamic pricing is not the same thing as surge pricing. Stripe makes the point that rising prices under peak demand are just one use case. Plenty of real deployments are markdowns designed to move excess stock, stimulate demand, or hold a position in a fast-moving category. Those use cases still need competitor and inventory data, they just push prices down instead of up, which is why a down-only guardrail is a legitimate configuration rather than an admission of weakness.

## Where each source actually comes from

The owned data is unglamorous work: event tracking on product pages for view and add-to-cart counts, a warehouse view that joins order history to price history, a nightly export from the ERP for stock and cost. If your conversion rate and your stock level are already in the same warehouse, you have most of what a rules-based engine needs.

External competitor data is different because you don't control the source, the format, or the markup. Three things follow from that.

**Frequency is a cost decision, not a technical one.** Stripe puts the ceiling at "by the hour, the minute, or even faster." Almost nobody needs the minute. A marketplace seller matching on price needs hourly at most. A brand tracking category-level positioning can work from a daily pull. What you're really choosing is how much stale price you can tolerate versus what the crawl costs.

**Page structure changes.** Marketplace listings add fields, change layouts, and swap in different anti-bot measures. A hand-rolled Scrapy job needs an engineer every time a template shifts. A managed endpoint moves that maintenance to the vendor, which is the actual reason people pay for one.

**Geography changes the answer.** Prices differ by country and currency. ScraperAPI's structured endpoints accept a `country_code` parameter and a `domain` parameter (for example `co.uk` or `com.au`), so you can pull prices as a local customer rather than as whatever your server's IP resolves to.

## What collecting competitor prices actually costs

This is the arithmetic that decides your plan, and it's the step most pricing write-ups skip. ScraperAPI publishes the multipliers, so you can model a watchlist before you buy.

| What you monitor | Requests per month | Credits per request | Credits per month |
| --- | --- | --- | --- |
| 200 Amazon SKUs, hourly | 144,000 | 5 | 720,000 |
| 200 Amazon SKUs, every 6 hours | 24,000 | 5 | 120,000 |
| 1,000 Google Shopping queries, daily | 30,000 | 25 | 750,000 |
| 500 standard pages, daily | 15,000 | 1 | 15,000 |

Two conclusions fall out of that table. Moving from hourly to six-hourly on 200 SKUs cuts your bill by 600,000 credits a month. And a Google Shopping or SERP-based assortment check costs five times what an Amazon product check costs, so if your plan leans on search results, frequency is the first dial to turn down.

A few behaviours worth knowing before you commit:

- **Unused credits don't roll over.** The balance resets when the subscription renews.
- **Exceeding concurrency throws a 429, not a charge.** The request is turned away rather than queued, and there's no extra fee for going over.
- **You can cap a single request.** The `max_cost` parameter sets a ceiling per scrape, and the Domain Cost Estimator in the dashboard shows the exact credit cost for any URL before you run a batch.
- **Batches and scheduling exist.** The Async scraper takes a list of URLs and returns one result object per URL; you can either poll or configure a cloud storage destination and a callback so results land in your bucket when each job finishes. DataPipeline's scheduler handles recurring jobs if you'd rather not build a cron.

If you're running a few hundred SKUs and want the burn to sit in the low hundreds of thousands of credits, that works. 👉 [Compare ScraperAPI plans by credit allowance before you commit](https://www.scraperapi.com/?fp_ref=coupons)

## The ScraperAPI plans, and what's confirmed about each

ScraperAPI scales allowance and rate together across tiers, so the useful question isn't which plan name sounds right, it's where your monthly credit burn lands and whether you want automatic overflow protection. Here's what the vendor's own documentation confirms about each tier's billing behaviour.

| Plan | Confirmed billing behaviour | Price and billing cycle | Get it |
| --- | --- | --- | --- |
| Free | 1,000 API credits, maximum 5 concurrent connections | Free | [Free plan details](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | Auto-upgrade to the next tier when credits run out (opt-in) | Monthly or annual rate shown on the pricing page | [Hobby plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | Auto-upgrade available | Monthly or annual rate | [Startup plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | Auto-upgrade available | Monthly or annual rate | [Business plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | Auto-upgrade and pay-as-you-go at a fixed per-credit rate with a monthly cap you set | Monthly or annual rate | [Scaling plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | Auto-upgrade and pay-as-you-go | Monthly or annual rate | [Professional plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | Pay-as-you-go only | Monthly or annual rate | [Advanced plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Pay-as-you-go only | Monthly or annual rate | [Enterprise plan](https://www.scraperapi.com/?fp_ref=coupons) |

Two structural differences matter more than the numbers. From Scaling upward you get pay-as-you-go, which lets a month of heavy competitor monitoring spill past your allowance at a fixed per-credit rate instead of stalling. Below that, auto-upgrade is the overflow mechanism, and it moves you to the next tier rather than charging overage. Annual plans can also renew early for a fresh batch of credits, which is worth knowing if your category has a seasonal peak.

Bring the credit-per-month figure from the table above to the pricing page. That's the number that tells you which row you belong in.

## Rules first, models later

Shopify draws the line clearly: rules-based models use if/then logic, are auditable, and need little compute, which makes them the right tool for competitor price-matching and clearing excess stock. Algorithmic or model-driven pricing suits categories where demand swings faster than a human can write rules for.

Whatever you build, Stripe's four guardrails are the minimum viable set:

1. **Price floors and ceilings** so nothing leaves a brand-safe range.
2. **Rate-of-change limits** so a bad data feed can't double a price overnight.
3. **Fallback logic** that defines defaults when a feed is interrupted or inconsistent.
4. **Manual override paths** so a human can pause automation during a launch or a sale.

The operating loop around them is four steps: ingest and interpret multiple streams in real time, run the pricing logic or model at scale, push updates to the site, POS, app and catalog, then log results so you can monitor, analyse and override.

As for how much feedback you need before a model is worth it, there's a relevant theory result and it's worth being precise about what it says. A November 2026 preprint by researchers at MIT, CUHK-Shenzhen and Microsoft Research shows that in the standard single-product pricing model, a two-option menu achieves the same regret rates as an infinite menu, up to log factors. In plain terms, a small amount of structured feedback about willingness to pay goes a long way. That's a regret bound, not a revenue benchmark, and it says nothing about your category. It does argue against the instinct that you need every signal before you start.

On the practitioner side, Salesforce's write-up includes one first-hand data point: the author helped an online power tool retailer launch a first dynamic pricing strategy keyed to demand, competitor activity and seasonality, and reports a 9% quarterly revenue improvement. That's one project, described by the person who ran it, not an industry average. The same article warns that going live without clean data and guardrails damages trust, which is the more transferable lesson.

## The legal boundary around the data you collect

Collecting public price pages is one thing. Using personal data to set an individual price is another, and the disclosure rules are now concrete.

New York's Algorithmic Pricing Disclosure Act took effect on **November 10, 2025**. Where personal data sets the price, businesses must display the string "THIS PRICE WAS SET BY AN ALGORITHM USING YOUR PERSONAL DATA." Shopify describes it as the first law of its kind in the US, and other states are drafting similar rules.

For EU customers, the Omnibus Directive doesn't ban frequent price changes. It requires clear disclosure when a price is personalised, and when a discount is shown, which matters if your dynamic pricing logic also generates the strikethrough price.

The third risk is older than any of this. Raising prices during emergencies, exploiting vulnerable consumers, or pricing in a way that eliminates competition can violate antitrust and price gouging statutes. Salesforce adds the commercial version of the same problem: competitive repricing can trigger a price war where everyone undercuts until nobody makes margin.

This is the part that should shape what you collect. Competitor prices and stock levels are public market data. Browsing history tied to an identified individual is regulated data. Keeping those two pipelines separate is cheaper than retrofitting the distinction later.

## FAQ

**How often should I scrape competitor prices?**
Hourly is enough for most price-matching. Daily is enough for category positioning. Minute-level pulls are technically possible and rarely worth the credits, since the cost scales linearly with frequency while the pricing decision usually doesn't.

**What if the competitor page sits behind Cloudflare or DataDome?**
ScraperAPI handles those domains and adds 10 credits per request when protection is bypassed. That surcharge is worth building into your cost model early, because a protected marketplace can triple the effective cost of a 5-credit Amazon request.

**Do unused credits carry over?**
No. The balance resets at renewal, so an oversized annual allowance that you only use in Q4 is money spent twice.

**Can I stop a single request from burning through credits?**
Yes, with `max_cost`. The Domain Cost Estimator in the dashboard shows the cost of any URL before you run it, which is the cheapest way to audit a new target list.

**What happens when I hit my concurrency limit?**
You get a 429 response and the request is refused rather than queued. There's no overage charge, but there's also no automatic retry, so your job needs to handle the backoff itself.

## Which setup fits you

Match the configuration to the watchlist, not to the tier chart.

- **A dozen SKUs and a hypothesis to test.** Free tier. 1,000 credits and 5 concurrent connections is roughly 200 Amazon product checks, enough to prove the data is usable before you spend anything.
- **100 to 500 SKUs checked hourly.** You're looking at 120,000 to 720,000 credits a month depending on cadence and domain. Pick the tier that covers that burn and turn on auto-upgrade so a viral day doesn't leave you with stale prices.
- **Google Shopping or SERP-based assortment tracking.** At 25 credits per query this dominates your budget. Cut frequency before you cut coverage, because a weekly full sweep beats a daily sweep of a third of your category.
- **Full catalogue with warehouse delivery.** Async plus a storage destination, pay-as-you-go on Scaling or above, and a monthly cap you set deliberately rather than discover.

Two things to confirm before you order anything: run your real target URLs through the Domain Cost Estimator so the multipliers in your model are the actual ones, and decide whether your burn pattern is flat enough for annual billing or spiky enough that pay-as-you-go earns its place.

👉 [Check the current ScraperAPI plans and start with your own watchlist](https://www.scraperapi.com/?fp_ref=coupons)
