# ScraperAPI Python Complete Guide: How to Install the Python SDK? How to Use ScraperAPI with Requests & BeautifulSoup? How to Integrate It with Scrapy? Which Plan Fits Your Python Project? (Includes Full Pricing Comparison and Coupon Code)

If you've spent any real time in Python's web-scraping world, you've probably hit the same wall everyone hits eventually. Your `requests.get()` works fine on a quiet blog, then the moment you point it at Amazon, Google, or a Cloudflare-protected site, you're staring at a 403, a CAPTCHA page, or a stream of timeouts. You swap in proxies, you rotate user agents, you fiddle with headers — and somewhere around the third weekend of babysitting your scraper, you start wondering whether there's a simpler way.

That's the gap **ScraperAPI** was built to fill, and it's also why `scraperapi python` has become such a common search. Developers aren't really looking for another scraping library — `requests`, `BeautifulSoup`, `Scrapy`, and `aiohttp` already cover that. They're looking for infrastructure: proxy rotation, headless browser rendering, CAPTCHA handling, retries, and geotargeting, all wrapped behind a single API call they can drop into existing Python code without rewriting their scraper.

This guide walks through everything a Python developer actually needs: installing the official SDK, integrating it with `requests` and `BeautifulSoup`, scaling with concurrent threads, plugging it into Scrapy, understanding the parameter system, and — because this is the part that catches everyone off guard — understanding how the credit system really works so you can pick the right plan before you've burned a month of credits on the wrong target.

## What ScraperAPI Actually Is (The 30-Second Version)

ScraperAPI is a web scraping API that takes over the unglamorous infrastructure work. You send it a URL plus your API key, and it returns clean HTML or JSON. Behind the scenes, it rotates requests through a pool of more than 40 million IPs across 50+ countries, handles proxy/header rotation, runs JavaScript rendering when needed, attempts CAPTCHA and anti-bot bypasses, and retries failed requests automatically. You only get billed for successful responses (anything outside a 200 or 404 doesn't burn credits).

For Python developers, the appeal is mostly that you don't have to change your scraping logic. ScraperAPI exposes three integration paths that all work from Python: the API endpoint (a normal HTTP GET), the official Python SDK, and a proxy port you can drop into any library that supports proxies.

👉 [Start with 5,000 free credits and test it against your real targets](https://www.scraperapi.com/?fp_ref=coupons)

## Installing the ScraperAPI Python SDK

The fastest path is the official SDK, published on PyPI as `scraperapi-sdk`.

bash
pip install scraperapi-sdk


Once installed, the basic usage is about as minimal as it gets:

python
from scraperapi_sdk import ScraperAPIClient

client = ScraperAPIClient('YOUR_API_KEY')
result = client.get(url='https://example.com/')
print(result)


That's the whole "hello world." The SDK has retry logic baked in (default is 3 retries, configurable via `retry=5` or whatever you want), so you don't have to write your own retry loops the way you do with the raw API endpoint. For most Python projects, this is the integration path I'd reach for first — it's the cleanest and removes the most boilerplate.

## Using ScraperAPI with `requests` and `BeautifulSoup`

If you'd rather not add a dependency, or you want tighter control over the request lifecycle, the API endpoint approach works with the standard `requests` library. The pattern is to send a GET to `http://api.scraperapi.com/` with your API key and target URL as query params:

python
import requests
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'
target_url = 'https://quotes.toscrape.com/page/1/'

params = {'api_key': API_KEY, 'url': target_url}
response = requests.get('http://api.scraperapi.com/', params=urlencode(params))
print(response.text)


A few things worth knowing before you ship this to production:

- **Timeouts**: ScraperAPI will retry internally for up to 60 seconds, switching proxy/header combinations if the first attempt gets banned or hits a CAPTCHA. Set your client timeout to at least 60 seconds, or leave it unset.
- **SSL verification**: When using the proxy port method (not the API endpoint above), you need `verify=False` in your `requests` call.
- **Request size limit**: 2 MB per request. Fine for HTML, worth knowing if you're scraping images or PDFs.
- **Failed requests return a 500** and are not billed. You should retry them yourself with a small loop — three retries covers ~99.9% of cases.

Here's a more complete pattern with retries and BeautifulSoup parsing:

python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'
NUM_RETRIES = 3
list_of_urls = [
    'https://quotes.toscrape.com/page/1/',
    'https://quotes.toscrape.com/page/2/',
]

scraped_quotes = []

for url in list_of_urls:
    params = {'api_key': API_KEY, 'url': url}
    for _ in range(NUM_RETRIES):
        try:
            response = requests.get('http://api.scraperapi.com/',
                                    params=urlencode(params))
            if response.status_code in [200, 404]:
                break
        except requests.exceptions.ConnectionError:
            response = ''

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        for quote_block in soup.find_all('div', class_='quote'):
            quote = quote_block.find('span', class_='text').text
            author = quote_block.find('small', class_='author').text
            scraped_quotes.append({'quote': quote, 'author': author})

print(scraped_quotes)


If you're using the SDK instead of the raw endpoint, you can skip the retry loop entirely — the SDK handles it for you.

## Scaling Up: Concurrent Threads in Python

This is where most Python scrapers fall over. A single-threaded loop works fine for a few hundred pages, but the moment you need millions of pages per day, you're bottlenecked on latency, not on the target site.

ScraperAPI's plans are differentiated partly by **concurrent threads** — the number of requests you can have in flight at the same time. The Hobby plan gives you 20 concurrent threads; Startup gives you 50; Business gives you 100; and it scales up to 500 at the Advanced tier. The more threads you have, the more requests you can run in parallel, and the faster your scraper completes.

Python's `concurrent.futures.ThreadPoolExecutor` is the simplest way to use those threads from a standard script:

python
import requests
from bs4 import BeautifulSoup
import concurrent.futures
from urllib.parse import urlencode

API_KEY = 'YOUR_API_KEY'
NUM_RETRIES = 3
NUM_THREADS = 20  # match this to your plan's concurrent thread limit

list_of_urls = ['https://quotes.toscrape.com/page/{}/'.format(i)
                for i in range(1, 101)]

scraped_quotes = []

def scrape_url(url):
    params = {'api_key': API_KEY, 'url': url}
    for _ in range(NUM_RETRIES):
        try:
            response = requests.get('http://api.scraperapi.com/',
                                    params=urlencode(params))
            if response.status_code in [200, 404]:
                break
        except requests.exceptions.ConnectionError:
            response = ''

    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        for quote_block in soup.find_all('div', class_='quote'):
            quote = quote_block.find('span', class_='text').text
            author = quote_block.find('small', class_='author').text
            scraped_quotes.append({'quote': quote, 'author': author})

with concurrent.futures.ThreadPoolExecutor(max_workers=NUM_THREADS) as executor:
    executor.map(scrape_url, list_of_urls)

print(scraped_quotes)


For higher concurrency (300+ parallel requests), most Python developers switch from `requests` to `aiohttp` with `asyncio`. The same principle applies — set your concurrency to match the thread limit on your plan, and let ScraperAPI handle the proxy rotation underneath.

👉 [Test concurrent scraping free for 7 days — no credit card needed](https://www.scraperapi.com/?fp_ref=coupons)

## Plugging ScraperAPI into Scrapy

If you're running Scrapy, you don't need to rewrite your spiders. ScraperAPI publishes an official middleware, `scrapy-scraperapi-proxy`, that routes every Scrapy request through ScraperAPI's proxy pool.

Install it from GitHub:

bash
git clone https://github.com/scraperapi/scrapy-scraperapi-proxy
cd scrapy-scraperapi-proxy
python setup.py install


Then enable it in `settings.py`:

python
DOWNLOADER_MIDDLEWARES = {
    'scrapy_scraperapi_proxy.ScrapyScraperAPIMiddleware': 350,
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
}

SCRAPERAPI_KEY = 'YOUR_API_KEY'


From the spider's perspective, nothing else changes. Scrapy still emits `Request` objects; the middleware transparently forwards them through ScraperAPI. This is the right approach when you already have a Scrapy codebase and you don't want to migrate your spiders to a new request pattern.

There's also a community-published `scrapy-scraperapi-middleware` package on PyPI for the same purpose — either works, the official one is maintained by ScraperAPI themselves.

## The Python SDK Parameter Set

The real power of ScraperAPI from Python isn't the basic GET — it's the parameters you can layer on. Here are the ones Python developers reach for most often:

| Parameter | What it does | Extra credit cost |
| --- | --- | --- |
| `render=true` | Enables JavaScript rendering (Puppeteer under the hood) | +10 credits |
| `premium=true` | Uses premium residential/mobile proxy pools | +10 credits |
| `country_code=us` | Geotargets the request to a specific country | 0 (free) |
| `ultra_premium=true` | Highest-quality proxies for the hardest targets | +30 credits |
| `screenshot=true` | Returns a screenshot of the rendered page | +10 credits |
| `keep_headers=true` | Preserves custom headers across the request | 0 |
| `session_number=N` | Pins a session so multiple requests share one IP | 0 |

With the Python SDK, you pass these as a `params` dict:

python
result = client.get(
    url='https://example.com/',
    params={'render': True, 'country_code': 'us'}
)


With the raw API endpoint, the same parameters become query string params on the GET request.

The combinations matter more than the individual parameters. `render=true` alone is 10 extra credits. `premium=true` alone is 10. But `render=true` + `premium=true` is **25 credits total**, not 20 — there's a combined multiplier. And `ultra_premium=true` + `render=true` jumps to **75 credits per request**. This is the single biggest source of "wait, where did my credits go?" surprises.

## The Credit Multiplier Reality Check

This is the part most `scraperapi python` tutorials gloss over, and it's the part that determines which plan you actually need.

ScraperAPI bills by **API credits**, not by requests. The base rate is 1 credit per request on a standard, unprotected page. But the target domain and the parameters you enable both multiply that cost:

- **Standard page (no protection)**: 1 credit
- **Amazon**: 5 credits
- **Google / Bing (and subdomains)**: 25 credits
- **LinkedIn**: 30 credits
- **Sites behind Cloudflare / Datadome / PerimeterX**: +10 credits on top of the base

Then the optional parameters stack on top of that. If you're scraping Amazon with JavaScript rendering enabled, that's `5 (Amazon) + 10 (render) = 15 credits` per request — meaning the Hobby plan's 100,000 credits get you roughly 6,600 successful Amazon scrapes, not 100,000.

Before committing to a plan, run a handful of test requests through the **Domain Credit Cost Estimator** in the ScraperAPI dashboard. It'll show you the exact per-request cost for any URL you're targeting, broken down by request type. It's the single most useful tool for picking the right plan, and it's the thing most new users skip.

The good news: failed requests (anything outside a 200 or 404) don't burn credits. You're paying for delivered data, not for the service's own failures.

## Full Plan Comparison — Every Tier on the Official Pricing Page

Every plan below includes the same core feature set: JS rendering, premium proxies, JSON auto-parsing, rotating proxy pools, custom headers, CAPTCHA/anti-bot bypass, custom sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. The differences are volume, concurrency, geotargeting scope, and a few perks at higher tiers.

| Plan | Monthly Price | Annual Price (10% off) | API Credits / Month | Concurrent Threads | Geotargeting | Get This Plan |
| --- | --- | --- | --- | --- | --- | --- |
| **Free Trial** | $0 (7 days) | — | 5,000 (one-time trial) | 5 | — |  [Start free trial](https://www.scraperapi.com/?fp_ref=coupons) |
| **Free Plan** | $0 | — | 1,000 (recurring monthly) | 5 | — |  [Sign up](https://dashboard.scraperapi.com/signup?fp_ref=coupons) |
| **Hobby** | $49/mo | $44.10/mo | 100,000 | 20 | US & EU only |  [Get Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Startup** | $149/mo | $134.10/mo | 1,000,000 | 50 | US & EU only |  [Get Startup](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Business** | $299/mo | $269.10/mo | 3,000,000 | 100 | Global |  [Get Business](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Scaling** (most popular) | $475/mo | $427.50/mo | 5,000,000 | 200 | Global |  [Get Scaling](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Professional** | $975/mo | $877.50/mo | 10,500,000 | 300 | Global |  [Get Professional](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Advanced** | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | Global |  [Get Advanced](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Enterprise** | Custom | Custom | 22,000,000+ | 500+ | Global |  [Contact sales](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

A few things that aren't obvious from the table:

- **Geotargeting is gated by tier.** Hobby and Startup are limited to US & EU proxies. If your Python project needs country-level targeting anywhere else, you need at least Business.
- **Pay-as-you-go overflow is only available from Scaling upward.** On Hobby, Startup, and Business, hitting your credit limit mid-cycle means upgrading or contacting support. From Scaling up, you can keep scraping at a fixed overage rate.
- **Credits don't roll over.** Whatever you don't use resets at renewal, so size the plan to your actual monthly volume, not to "just in case."
- **Unlimited analytics history** starts at Business. Hobby and Startup cap you at 30 days of dashboard history.

## How to Pick a Plan for Your Python Project

The "right" plan depends entirely on what you're scraping and how often, not on the sticker price.

**Hobby ($49/mo) fits** if you're running a personal project, a side hustle, or a prototype — checking competitor prices on a handful of products, monitoring a few dozen pages, building a proof of concept. 100,000 credits is plenty for plain unprotected pages. It evaporates fast once Amazon or Google enters the picture.

**Startup ($149/mo) fits** if you've outgrown casual scraping. A small SaaS product or an agency running scraping jobs for a handful of clients. 1,000,000 credits with 50 concurrent threads is a real step up, but you're still capped at US/EU geotargeting.

**Business ($299/mo) fits** if you need global geotargeting (not just US/EU), unlimited analytics history, or you're running production infrastructure that other parts of your business depend on. This is the first tier where the jump in concurrent threads (100) starts to matter for larger parallel jobs in Python.

**Scaling ($475/mo) and above fit** once you're past the "which plan" question and into "how do I keep this predictable at high volume." These tiers add pay-as-you-go overflow so you're never hard-capped mid-month, and priority support kicks in at Professional.

The cleanest way to decide is to run the trial: sign up for the 5,000-credit free trial, point your Python scraper at your real targets, and watch the credit consumption in the dashboard before committing to a paid plan.

👉 [Start the free trial and run your own numbers](https://www.scraperapi.com/?fp_ref=coupons)

## What Python Developers Actually Say About It

Independent review aggregators paint a fairly consistent picture. ScraperAPI sits around **4.5/5 on Trustpilot** (42+ reviews) and **4.4/5 on G2**. The recurring praise is the same on both platforms: clean documentation, genuinely simple integration (especially the Python SDK and the proxy port method, which let you drop it into existing code without rewriting your scraper), and responsive support. One long-time reviewer specifically called out that upgrading or downgrading plans was painless.

The criticism is consistent too, and it's worth hearing before you commit: the credit math is less intuitive than the headline numbers suggest. Independent benchmarking from third-party testers has noted that performance varies by target — ScraperAPI performs very well on mainstream sites like Amazon, GitHub, and standard e-commerce pages, but less consistently on sites with aggressive, frequently-changing anti-bot systems. The Domain Credit Cost Estimator exists specifically because so many users got caught off guard by multipliers.

For Python developers specifically, the most common positive theme in reviews is that the SDK "just works" — you install it, pass your API key, and you're scraping within minutes. The most common negative theme is that people wish they'd understood the credit multipliers before picking a plan.

## Active Discounts and Coupon Codes

A few savings paths are worth knowing about:

- **Automatic 10% off annual billing** — no code needed. Switch any plan to yearly billing and the discount applies at checkout. This is the simplest, most reliable saving.
- **7-day free trial with 5,000 credits** — no credit card required, available to every new account.
- **Recurring free plan** — 1,000 credits per month with 5 concurrent connections, permanently free, suitable for small ongoing projects.
- **Third-party coupon codes** — various promo codes circulate on coupon sites (e.g., `DATALOVER`, `ANWAR10`, `BOOM10`, `ARCHANA`, `rohit44`), most offering around 10% off monthly subscriptions. These are community-shared and not always verified, so test them at checkout before relying on them.

The most reliable route for new users is to sign up through the promotional link, take the free trial, and then switch to annual billing once you've confirmed the plan fits. That combination locks in both the trial credits and the 10% annual discount.

👉 [Claim the current sign-up offer and start your free trial](https://www.scraperapi.com/?fp_ref=coupons)

## Frequently Asked Questions

**Does one API request always cost one credit in Python?**
No. The base rate is 1 credit for a standard page, but the target domain (Amazon 5x, Google/Bing 25x, LinkedIn 30x) and any parameters you enable (rendering, premium proxies) multiply that cost. Use the dashboard's Domain Credit Cost Estimator before scraping at scale.

**Which Python integration should I use — SDK, API endpoint, or proxy port?**
Use the SDK (`pip install scraperapi-sdk`) for the cleanest code and built-in retries. Use the API endpoint with `requests` if you want minimal dependencies. Use the proxy port if you're integrating with a library that natively supports HTTP proxies (Scrapy, `aiohttp`, etc.).

**Can I use ScraperAPI with `asyncio` and `aiohttp`?**
Yes. The proxy port method works natively with `aiohttp`. For 300+ concurrent requests, `aiohttp` typically outperforms `httpx` by 1.5–5x in throughput.

**What happens if I run out of credits mid-month?**
On Hobby, Startup, or Business, you can upgrade to the next tier or contact support about a custom arrangement. Scaling, Professional, Advanced, and Enterprise customers can keep scraping via pay-as-you-go at a fixed rate.

**Can I cancel anytime?**
Yes — cancellation is available anytime from the dashboard or by contacting support, and you won't be charged again after cancelling. There's also a 7-day no-questions-asked refund policy.

**Do unused credits roll over?**
No. Your credit balance resets at each renewal, so match your plan size to your actual monthly usage rather than stockpiling.

**Is there a free plan?**
Yes — 1,000 free API credits per month with 5 concurrent connections, plus a one-time 7-day trial that bumps you to 5,000 credits. No credit card required for either.

## Bottom Line for Python Developers

If your scraping targets are mostly plain pages without heavy anti-bot protection, the Hobby plan at $49/month covers a surprising amount of ground for personal projects or early-stage Python products. The moment Amazon, Google, LinkedIn, or Cloudflare-protected sites enter the picture, run your numbers through the credit table first — the sticker price and the real cost per successful scrape are two different things, and the Python SDK won't warn you when a request burns 25 credits instead of 1.

The cleanest path forward is to test before you commit. Sign up for the free trial, drop the SDK into your existing Python scraper with `pip install scraperapi-sdk`, point it at your real targets, and watch the credit consumption in the dashboard. That's the only way to know which plan actually fits your workload — and it costs nothing to find out.

👉 [Start your free ScraperAPI trial — 5,000 credits, no credit card required](https://www.scraperapi.com/?fp_ref=coupons)
