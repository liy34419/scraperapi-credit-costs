# ScraperAPI Python SDK: how do you install it, which parameters cost 10x credits, and when should you switch to async jobs?

`pip install scraperapi-sdk` is the whole installation, the current release is 1.5.4 (published 26 September 2025), and the shortest useful call looks like this:

python
import os
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient(os.environ["SCRAPERAPI_API_KEY"])
print(client.get("https://example.com/")[:300])


Two details separate working code from the tutorials still ranking on page one. The import is `scraperapi_sdk`; the `scraper_api` package used in 2021 write-ups is an older distribution, and code copied from those posts won't match the current release. And 1.5.4 ships `ScraperAPIAsyncClient`, so a 50-URL batch no longer forces you down to raw `requests`.

What follows is the part that decides your bill: the parameters that multiply a request by 10 to 75 credits, the arithmetic that turns those multipliers into a plan choice, the async path, and the Scrapy settings that quietly break things.

> **Quick pick:** Hobby at $49/month includes 100,000 credits, which is 100,000 plain-HTML pages or about 10,000 pages if every one needs JavaScript rendering. If more than a tenth of your targets are JS-heavy or behind bot protection, budget for Startup instead. 👉 [See the current plans and credit allowances](https://www.scraperapi.com/?fp_ref=coupons)

## What each plan costs and what it covers

| Plan | Price | Credits per month | Best for | Start |
| --- | --- | --- | --- | --- |
| Free | $0 | 1,000 (5,000 during your first 7 days) | Testing the SDK, 5 concurrent threads | [ Create a free account](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49/mo, or $44.10/mo billed yearly | 100,000 | Plain-HTML scraping at small scale | [ Start on Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | $149/mo | ~1,000,000 at 1 credit per page | The first tier with room for rendered pages | [ Compare Hobby and Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | $299/mo | — | Teams that outgrow a million credits | [ See the Business tier](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling, Professional, Advanced, Enterprise | Quoted individually | — | Pay-as-you-go with a monthly spend cap | [ Ask about volume pricing](https://www.scraperapi.com/?fp_ref=coupons) |

A dash means the tier is quoted through sales rather than sold as a fixed monthly allowance. ScraperAPI's billing documentation puts pay-as-you-go and monthly spending caps on Scaling and above; Hobby, Startup and Business instead get an opt-in auto-upgrade that bumps you to the next plan when credits run out. Without auto-upgrade, scraping pauses until the credits renew.

Three things worth knowing before you pay:

- Every new account starts with 5,000 credits for the first 7 days, then drops to the 1,000-credit free allowance. No card is required to start.
- Paid plans carry a 7-day, no-questions-asked refund policy.
- Annual plans hand you the entire year's credits upfront, so you can spend them on any schedule, and you can renew the same plan early if you run out.

Credits and dollars are separate ledgers. A $49 Hobby plan and a $149 Startup plan are not "three times the scraping" in every scenario, because the per-request multiplier decides how far the allowance stretches.

## Every parameter that changes the credit cost

| Parameter | What it does | Credits per request |
| --- | --- | --- |
| *(none)* | Standard request through the rotating proxy pool | 1 |
| `render=True` | Runs the page in a headless browser and returns rendered HTML | 10 |
| `premium=True` | Residential and mobile IPs for sites that block datacenter ranges | 10 (25 with `render`) |
| `ultra_premium=True` | Strongest anti-bot tier | 30 (75 with `render`) |
| `session_number=42` | Sticky IP reused across requests | No surcharge |
| `keep_headers=True` + `headers={...}` | Forwards your own headers to the target | No surcharge |
| `autoparse=True` | Returns parsed JSON for supported sites | Structured rates, typically 1–5 |
| `output_format="markdown"` | Returns Markdown, useful for LLM pipelines | Same as the underlying request |
| `wait_for_selector=".product-list"` | Waits for an element before capture; requires `render=True` | Included in the render cost |
| `max_cost=N` | Caps a single request at N credits; returns 403 if it would exceed that | — |

Three of these have gotchas that cost real money:

**`ultra_premium` ignores `keep_headers`.** The two are incompatible, so any custom header you pass is silently dropped. If a target needs both your `Accept-Language` and the strongest proxy tier, you can't have both in one request.

**`session_number` expires 15 minutes after the last use.** Any integer works as an ID. It's the right tool for pagination, multi-step flows and staying logged in, but a slow loop that idles longer than 15 minutes will come back on a different IP mid-session.

**`render=True` should be your second attempt, not your first.** ScraperAPI's own guidance is to send the request without rendering and add it only when the returned HTML is missing content, since rendering adds both credits and latency. `screenshot=True` turns rendering on automatically, which is easy to forget when you're debugging why a cheap request suddenly costs 10.

## The escalation ladder, and what a failed URL really costs

A common pattern from ScraperAPI's own SDK reference walks up the tiers until the response looks like real HTML:

python
def scrape(url):
    for params in [
        {},                                  # 1 credit
        {"render": True},                    # 10
        {"premium": True},                   # 10
        {"premium": True, "render": True},   # 25
        {"ultra_premium": True},             # 30
    ]:
        result = client.get(url, params=params)
        if result and "<html" in result.lower():
            return result
    return None


The loop is fine as a fallback, but do the arithmetic: a URL that fails every rung burns 76 credits before you give up. On Hobby, 1,300 such URLs would eat the entire monthly allowance. Add `max_cost` to the expensive rungs if you're running this at volume, and log which rung succeeded so you can start there next time instead of paying for the cheap attempts again.

Here's how the multipliers translate into actual page counts on the $49 Hobby plan:

- 10,000 plain pages: 10,000 credits, about a tenth of the allowance.
- The same 10,000 pages with `render=True`: 100,000 credits, the whole plan.
- 10,000 pages at `premium` plus `render`: 250,000 credits, which is Startup territory.
- Google or Bing results at 25 credits each: 4,000 searches on Hobby.
- Amazon pages at 5 credits each: 20,000 product pages on Hobby.

That last pair matters because scraping a search engine or a marketplace at 25 credits per request changes the plan math completely. A workload that looks like "a few thousand pages" can land on Business once the domain multiplier is applied.

## Structured endpoints cost less than rendering HTML yourself

For Amazon, Google, Walmart, eBay, Redfin and a few others, ScraperAPI returns parsed JSON instead of raw HTML. The verticals include `google/search`, `google/news`, `google/jobs`, `google/shopping`, `google/maps`, `amazon/product`, `amazon/search`, `amazon/offers`, `walmart/product`, `walmart/search`, `walmart/category`, `ebay/product`, `ebay/search`, `redfin/listing` and `redfin/search`.

Most of these cost 1–5 credits, against 10 for a rendered HTML fetch plus whatever you spend writing and maintaining the parser. An Amazon product through the structured endpoint comes back with title, price and review data already mapped to fields. The synchronous form takes the vertical in the path plus your `api_key` and the business parameters, with a 70-second timeout.

`autoparse=True` does something similar on the standard endpoint for supported sites, and `output_format="markdown"` is the option to reach for when the output feeds an LLM rather than a database.

## Sync or async? The rule is 20 URLs

ScraperAPI's own mapping table is the clearest guide: a single URL where you need the response now goes through `client.get()`. Twenty or more URLs, or a single URL that regularly times out, goes to the Async Jobs API.

Version 1.5.4 gives you that in the SDK:

python
import os
from scraperapi_sdk import ScraperAPIAsyncClient, ScraperAPIException

client = ScraperAPIAsyncClient(os.environ["SCRAPERAPI_API_KEY"])

job = client.create("https://example.com", webhook_url="https://your.app/hook")
result = client.wait(
    job["id"],
    cooldown=5,
    max_retries=10,
    raise_for_exceeding_max_retries=False,
)


`wait()` polls for you: `cooldown` is the seconds between retries (default 5), `max_retries` defaults to 10, and `raise_for_exceeding_max_retries=False` returns the last API response instead of raising when you hit the ceiling. If you'd rather not poll at all, pass a `webhook_url` and let ScraperAPI call you when the job finishes.

The async client also exposes platform-specific methods that submit structured jobs, which is the shortest route from "I have 40 ASINs" to parsed JSON:

python
job = client.amazon.products(
    ["B0B5PLT7FZ", "B00CL6353A"],
    api_params=dict(country_code="uk", tld="co.uk"),
)
result = client.get(job.get("id"))


The same pattern exists for `client.amazon.product(...)`, `client.amazon.search(...)`, and Google search, news, jobs and shopping.

For genuinely large runs, the REST endpoints are still there and still the right tool. `POST https://async.scraperapi.com/jobs` with `{"apiKey": ..., "url": ..., "apiParams": {...}}` returns an `id`, a `status` and a `statusUrl` you poll until it reads `finished`. Batch submissions go to `async.scraperapi.com/batchjobs` with a `urls` array, and the ceiling there is 50,000 URLs per batch.

## Scrapy: three settings, one of which silently breaks calls

The SDK wraps Scrapy requests for you:

python
yield scrapy.Request(client.scrapyGet(url, render=True), callback=self.parse)


Then fix `settings.py`:

python
CONCURRENT_REQUESTS = 5      # match your plan's concurrent thread count
ROBOTSTXT_OBEY = False       # otherwise Scrapy hits robots.txt first and breaks API calls
RETRY_TIMES = 3
# no DOWNLOAD_DELAY, no RANDOMIZE_DOWNLOAD_DELAY


`ROBOTSTXT_OBEY = False` is the one people miss. Scrapy's default behaviour sends a request to the target's robots.txt before your actual request, which interferes when every request is being routed through the API endpoint. `DOWNLOAD_DELAY` and `RANDOMIZE_DOWNLOAD_DELAY` should be removed too; the API handles proxy and header rotation, and those settings only throttle your own concurrency.

Scrapy already skips SSL verification by default, so there's nothing to disable there, unlike the plain `requests` proxy mode which needs `verify=False`.

On retries, ScraperAPI's documentation claims more than 97% of requests succeed on the first attempt for most sites, with near-100% after three retries. `RETRY_TIMES = 3` matches that.

## Status codes and what to do with each

| Code | Meaning | Action |
| --- | --- | --- |
| 200 | Success | Parse it |
| 401 | Invalid API key | Check `SCRAPERAPI_API_KEY` |
| 403 | Blocked, **or** credits exhausted | Escalate the proxy tier, or check your balance |
| 404 | Target page doesn't exist | Don't retry |
| 429 | Concurrent request limit hit | Back off, or move the batch to async |
| 500 / 503 | Transient failure | Retry with backoff; **500 responses aren't charged** |

The 403 is the one that wastes afternoons, because "blocked" and "out of credits" return the same code. Check the dashboard balance before you start tuning proxy tiers.

Two more constraints worth pinning to the wall: a synchronous SDK call blocks for up to 70 seconds while the API rotates proxies internally, so don't set a client-side timeout below that or you'll cut off requests that were about to succeed. And every request has a 2MB response ceiling, images and PDFs included.

The SDK raises `ScraperAPIException`, and you can read the underlying error via `.original_exception`:

python
from scraperapi_sdk.exceptions import ScraperAPIException

try:
    html = client.get(url, params={"render": True, "max_cost": 10})
except ScraperAPIException as e:
    print(type(e.original_exception).__name__, e)


## Questions that come up before signing up

**Does the SDK support async and batch jobs?** Yes, since the 1.5.4 release. `ScraperAPIAsyncClient` covers `create`, `get`, `wait`, webhook callbacks and structured Amazon/Google jobs. Raw `requests` against the async endpoints is only necessary for batch submissions beyond what the SDK's per-job methods handle, up to 50,000 URLs.

**Can I send my own headers?** Yes, with `keep_headers=True` and a `headers` dict. It won't work alongside `ultra_premium=True`.

**How do sticky sessions work?** Set `session_number` to any integer and repeat it across requests. The IP is held for 15 minutes after the last request that used it.

**Is there a hard spend limit per request?** Add `"max_cost": N` to any request. If the request would cost more than N credits, it returns 403 instead of running.

**Does a failed request cost credits?** A 500 doesn't. That's why retrying transient failures is safe.

## Which plan your workload actually needs

Work backwards from the multiplier, not from page count:

- Under roughly 100,000 plain pages a month, with no rendering: **Hobby at $49**. 👉 [Start with Hobby and 100,000 credits](https://www.scraperapi.com/?fp_ref=coupons)
- Most pages needing JavaScript, or a mix of `premium` and `render`: divide your monthly page count by 10 first. If the result is over 100,000 credits, **Startup at $149** is the cheaper mistake than blowing through Hobby mid-month.
- Google, Bing or LinkedIn as targets: those sit at 25–30 credits each, so a "small" SERP monitoring job can need more credits than a much larger HTML crawl.
- Twenty-plus URLs per run, or recurring timeouts: build on `ScraperAPIAsyncClient` from day one rather than wrapping `client.get()` in threads.
- Testing anything at all: the free tier gives you 1,000 credits a month and 5,000 for the first week, which is enough to run a real page through every rung of the escalation ladder.

Before you enter a card, confirm two numbers on the pricing page: the credit allowance at your target tier, and the concurrency limit that goes into `CONCURRENT_REQUESTS`. Getting those wrong shows up as either a paused account mid-crawl or a spider that's throttled for no reason. 👉 [Check current allowances and concurrency limits](https://www.scraperapi.com/?fp_ref=coupons)
