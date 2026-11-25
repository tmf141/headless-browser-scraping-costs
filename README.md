# Headless browser scraping: should you run your own browser fleet or rent rendering? What 4GB of RAM really buys you, how credits multiply by target, and how to move existing Playwright code over (full plan comparison)

A rented rendering API costs less than most people assume once you price the alternative. Each headless Chrome instance holds 200–500MB of RAM, a single page takes 2–10 seconds to render, and a self-hosted browser tier tends to land somewhere between $200 and $800 a month before anyone gets paged at 2am. Self-hosting still wins when your volume is modest, your targets serve plain HTML, or you need a login flow you control end to end.

ScraperAPI sits on the rental side. You send one HTTP request with `render=true`, its service spawns a headless Chrome, executes the page's JavaScript, and returns the finished HTML. Rendering is available on every plan including the free one, which gives you 1,000 credits and 5 concurrent threads, and paid plans carry a 7-day no-questions-asked refund.

## Quick pick: where most readers should start

> Start on the free tier to measure what your actual target list costs in credits, then buy the smallest paid plan whose monthly allowance covers that number. You are buying volume and concurrency here, not access to rendering, since `render=true` works on the free plan too. 👉 [Check the current ScraperAPI plan prices and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Why JavaScript rendering is the whole problem

Modern sites ship a skeleton HTML file and let JavaScript fill in the content. A server-side parser like BeautifulSoup sees the skeleton, so your "empty container" bug is usually not a bug at all.

You can confirm this in two minutes in Chrome. Compare "View Page Source" (raw server response) with "Inspect Element" (the live DOM after scripts run). If they look nothing alike, you have a dynamic site. The faster version: disable JavaScript in browser settings, reload, and watch the content disappear. React/Vue/Angular single-page apps, infinite scroll, lazy-loaded images, and content fetched by API after first paint all behave the same way.

That is the reason headless browsers exist in scraping at all. Puppeteer and Playwright are the two mainstream control libraries, Selenium predates both and speaks the slower WebDriver protocol, and Pyppeteer is a less actively maintained Python port that talks directly to Chromium.

## What self-hosting actually costs

The numbers below are a third-party estimate from a scraping vendor's engineering blog (December 2025), not an official benchmark, but they are the most concrete public figures available and they match what most teams report.

- **200–500MB per headless Chrome instance**, with CPU spikes while JavaScript executes.
- **2–10 seconds per page**, depending on how much has to render.
- **A 4GB cloud server realistically runs 5–8 concurrent browser instances** before it starts thrashing memory.
- **$200–800 per month in infrastructure**, plus the maintenance time, which grows faster than the bill.

Run the arithmetic on throughput. Eight instances at five seconds per page is about 1.6 pages per second at absolute ceiling, and real duty cycles are far below that once retries, timeouts, and proxy rotation enter the picture. Memory, not CPU, is what stops you first, and it stops you at eight.

The open-source projects are attacking exactly that bottleneck. Lightpanda's Python package README includes an example that scrapes 100 quotes from a JavaScript-rendered page in about 3 seconds using 35MB, against roughly 6 seconds and 1.2GB for Selenium with headless Chrome, measured as the median of five runs. Same job, roughly 34× the memory on the Chrome side. That single number explains why a 4GB box feels cramped: it is a Chrome constraint, not a headless-browser constraint.

If your monthly page count is in the low tens of thousands and most targets are ordinary HTML, a $20–40 server plus a scheduler will genuinely be cheaper than any API. Past that, the maintenance line item starts competing with the infrastructure line item, and the calculus flips. 👉 [Price your own target list against ScraperAPI's credit system](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Where the open-source engines are now

Two projects are worth knowing about because they change the cost curve, though neither removes the operational work.

**Lightpanda** is an open-source headless browser built for scraping, automation, and AI agents. It installs with `pip install lightpanda` and the wheel bundles the browser binary, so there is no second install step. It ships a Chrome DevTools Protocol server, which means existing Puppeteer code can connect with `puppeteer.connect({ browserWSEndpoint })` and Playwright with `chromium.connectOverCDP(...)` without downloading Chromium at all. It also serves WebDriver, classic and BiDi. The catch worth knowing before you build on it: the package version follows browser releases with no backward-compatibility guarantee, so pin with `~=0.4.0` rather than `==0.4.0` if you want patch releases.

**Obscura** is a Rust headless engine that runs real JavaScript through V8 and speaks CDP, positioned as a drop-in replacement for headless Chrome. The project's own benchmark table puts static HTML around 51ms against roughly 500ms for Chrome, JS-plus-XHR pages at 84ms against about 800ms, and claims 30MB of memory against 200+MB for headless Chrome. Those are the maintainers' measurements, not independent ones, so treat them as directional.

### Two limits that will bite you and rarely get mentioned

**Robots.txt obedience applies to every request, not just the page you asked for.** Lightpanda's `--obey-robots` flag is off by default. Turn it on and the rule applies to each subresource as well. Plenty of sites disallow the directory their own CSS and JavaScript live in, so a page you are allowed to fetch can come back blank or unstyled because its assets were blocked. That is robots.txt working as intended, not a rendering bug. Obscura has the same flag with the same off-by-default setting. If a page renders strangely empty under robots obedience, read the site's robots.txt before you file an issue.

**An independent rendering engine is not Chromium.** Obscura implements block, inline, flex, grid, table, float, positioning, overflow, transform, text, image, SVG, canvas, background, border, and animation paths, and its own documentation says long-tail CSS, some Web APIs, media playback, compositor effects, and platform font rasterization may differ from Chromium. It also caps page script execution at 30 seconds by default, adjustable through `OBSCURA_SCRIPT_DEADLINE_MS`, which a heavy React SPA on a slow connection can exceed before it fires its data requests. Screenshot-diff your most important targets against real Chrome before you migrate a pipeline onto it.

## How ScraperAPI handles a rendered page

The integration is HTTP-level rather than browser-level. You pass `render=true` as a parameter, or set the proxy username to `scraperapi.render=true` if you are routing through the proxy port. The service spawns a headless browser, runs the page scripts, and returns the final response.

`wait_for_selector` lets you hold the response until a specific element appears, for pages where content arrives late. It has to be paired with `render=true` and does nothing on its own. Batch work goes through an asynchronous jobs endpoint that accepts `render` inside `apiParams`.

Around that core you get automatic retries, CAPTCHA and anti-bot handling, custom sessions, custom headers, desktop and mobile user agents, unlimited bandwidth, and a 99.9% uptime guarantee. Your parser stays where it is, which is the practical difference from swapping browser engines: your Cheerio or BeautifulSoup code keeps working because what comes back is still HTML.

## The plan lineup

ScraperAPI currently lists a free tier and seven paid tiers. Allowances step up across them, from 100,000 credits with 20 concurrent threads up to 1,000,000 credits with 50 threads on the larger published tiers, and Enterprise is quoted against your workload.

| Plan | What is confirmed on the page | When credits run out | Current pricing |
| --- | --- | --- | --- |
| Free | 1,000 credits, 5 concurrent threads | Scraping pauses until renewal | [Start on the free tier](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Entry paid tier | Auto-upgrade available (opt-in) | [See the Hobby plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Auto-upgrade available (opt-in) | Moves to the next plan automatically | [See the Startup plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Auto-upgrade available (opt-in) | Moves to the next plan automatically | [See the Business plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Auto-upgrade plus pay-as-you-go | Continue at a fixed per-credit rate up to your cap | [See the Scaling plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Auto-upgrade plus pay-as-you-go | Continue at a fixed per-credit rate up to your cap | [See the Professional plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | Pay-as-you-go | Continue at a fixed per-credit rate up to your cap | [See the Advanced plan](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom pricing across your usage | Pay-as-you-go, with a separate minimum rate for Ultra Premium requests | [Ask about Enterprise pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two things vary by tier and matter more than the headline price. Geotargeting on the tiers where it is documented is limited to US and EU regions. Analytics history runs to the last 30 days on those same tiers. If you need city-level targeting outside those regions, confirm it before you buy rather than after.

## Credits are not pages

This is the part that decides whether a plan is big enough, and it rarely gets explained up front. A standard page costs 1 credit. The same allowance buys dramatically fewer pages depending on what you point it at.

| Target | Credits per request |
| --- | --- |
| Standard HTML page | 1 |
| Amazon | 5 |
| Google, Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Sites behind Cloudflare, DataDome, or PerimeterX | +10 on top of the base cost |

A 100,000-credit allowance is 100,000 ordinary pages, 20,000 Amazon pages, 4,000 Google or Bing queries, or roughly 9,000 pages behind bot protection. That is a 25× spread across the same invoice. The dashboard's Domain Cost Estimator prices any specific URL, and `max_cost` caps what a single scrape is allowed to spend.

ScraperAPI does not queue overages. Exceed your concurrency and you get a 429 telling you to come back within your plan's thread limit. There is no extra charge for the rejected request; it simply does not run. 👉 [See which plan carries the concurrency your crawler needs](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Three routes, side by side

| Route | What you operate | Where it breaks down |
| --- | --- | --- |
| Self-hosted Playwright or Selenium | Browser fleet, proxy pool, queue, monitoring | 200–500MB per instance, 2–10s per page, on-call maintenance |
| Lightweight open-source engine | Same stack, much smaller footprint | Independent engines diverge from Chromium on long-tail CSS and some Web APIs; robots handling surprises |
| Managed rendering API | An HTTP request with `render=true` | Credit multipliers on hard targets; concurrency capped by plan tier |

## FAQ

**Will a headless browser stop me from getting blocked?**
Rendering JavaScript and avoiding detection are separate problems. Bot protection scores fingerprints and IP reputation, not whether a browser executed your scripts. Obscura's `--stealth` mode randomizes per-session fingerprints and blocks 3,520 tracker domains; ScraperAPI handles CAPTCHA and anti-bot checks and charges 10 extra credits per request when it has to bypass Cloudflare, DataDome, or PerimeterX.

**Should I turn on robots.txt obedience?**
It is off by default in both Lightpanda and Obscura. Turn it on if compliance matters for your use case, but expect some pages to render blank because the site disallows its own asset directory. Read the target's robots.txt first.

**Why am I getting 429s when my server has plenty of RAM?**
Concurrency on a managed API is a plan limit, not a hardware limit. Raising it means changing tiers. Your own RAM is irrelevant to that response code.

**What happens the moment I hit my credit ceiling?**
Scraping pauses by default. On Hobby, Startup, Business, Scaling, and Professional you can enable auto-upgrade, which moves you to the next plan when credits run out. On Scaling, Professional, Advanced, and Enterprise, pay-as-you-go keeps requests flowing at a fixed per-credit rate up to a monthly cap you set. Annual plans can also renew early for a fresh batch.

**Can I keep my Playwright scripts?**
With a CDP-compatible engine, yes: you change the connection endpoint and leave the automation code alone. With ScraperAPI you keep your parsing code and swap the fetch layer for an HTTP call with `render=true`, which is a smaller change than it sounds if your extraction logic already runs on returned HTML.

**Is there a refund if it does not fit?**
Seven days, no questions asked.

## How to decide

- **Under ~10,000 pages a month, mostly ordinary HTML:** run Playwright yourself. The server bill is smaller than any subscription and you keep full control of sessions.
- **Google, Bing, LinkedIn, or bot-protected targets in volume:** check the credit multiplier before you compare prices anywhere. A plan that looks generous against standard pages can be 4,000 queries' worth on search results.
- **Tens of thousands of pages and no appetite for browser infrastructure:** go managed, and start on the free 1,000 credits to measure your real per-page cost against your own URL list.
- **More than 50 concurrent threads:** that is Enterprise territory, and worth a conversation before you build around a lower tier.

Two things to confirm before you pay: run your actual URLs through the Domain Cost Estimator so the credit number on the pricing page matches your workload, and decide whether you want auto-upgrade on. It prevents a stalled pipeline, but it also means the bill can move without you touching anything. 👉 [Start on the free tier and test your own URLs before choosing a plan](https://www.scraperapi.com/?fp_ref=coupons)
