# langchain scraperapi: what the three tools return, how fast credits burn, and which plan you actually need (install steps + full plan table)

`pip install -U langchain-scraperapi` adds three tools to your LangChain agent: one that opens any URL as text, markdown or raw HTML, and two that return structured Google and Amazon results as JSON or CSV. Set `SCRAPERAPI_API_KEY` in your environment and your agent can browse, search and pull product data without you running proxies or a headless browser.

The part that decides whether you pay for it is the credit math. Requests cost between 1 and 75 credits depending on the target domain and the flags you pass, so a plan advertised as a million credits can mean 66,000 real agent calls or a million, depending on what you point it at. Below: the working setup, the parameter list that actually matters, the multiplier table, and the three places the plan lineup limits you in ways the price column doesn't show.

> **The short version**
>
> - Start on the free trial: 5,000 credits, seven days, no credit card. Run it against your real target URLs, not `example.com`.
> - Plain article and documentation pages cost 1 credit. The $49 tier covers 100,000 of them, and that's enough for most RAG and research agents.
> - Google SERP costs 25 credits a request and Amazon costs 5 (plus 10 if the page needs JavaScript). Those two are what push people up a tier.
> - Geotargeting outside the US and EU starts at Business ($299). Pay-as-you-go overflow starts at Scaling ($475).
>
> 👉 [Start the 5,000-credit trial and test your own URLs](https://www.scraperapi.com/?fp_ref=coupons)

## Full plan comparison

| Plan | Monthly | Credits/month | Concurrent threads | Geotargeting | Sign up |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | 1,000 | 5 | Limited | [Free plan](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 | 100,000 | 20 | US & EU | [Hobby](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | $149 | 1,000,000 | 50 | US & EU | [Startup](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | $299 | 3,000,000 | 100 | Global (50+ countries) | [Business](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | $475 | 5,000,000 | 200 | Global + pay-as-you-go | [Scaling](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | $975 | 10,500,000 | 300 | Global + pay-as-you-go | [Professional](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | $1,975 | 21,500,000 | 500 | Global + pay-as-you-go | [Advanced](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ | Global + custom | [Contact sales](https://www.scraperapi.com/?fp_ref=coupons) |

Annual billing takes roughly 10% off list price on every tier. The figures above are the published list prices at the time of writing; tier contents and promotional bonus credits change, so confirm the number on the plan page before checkout. Every tier includes JavaScript rendering, premium proxies, CAPTCHA handling, rotating sessions, automatic retries and JSON parsing. The differences are volume, concurrency and how far you can target geographically.

## Install and get a working agent in one file

Three packages plus two environment variables. The ScraperAPI key comes from your dashboard; the OpenAI key needs a funded account, and the free OpenAI trial credit does not cover API calls.

bash
pip install -U langchain-scraperapi langchain langchain_openai openai

export SCRAPERAPI_API_KEY="your-scraperapi-key"
export OPENAI_API_KEY="your-openai-key"
# Windows CMD: set SCRAPERAPI_API_KEY="your-scraperapi-key"


The agent uses the current tool-calling API rather than the older `initialize_agent` pattern:

python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_scraperapi.tools import ScraperAPITool

tools = [ScraperAPITool()]
llm = ChatOpenAI(model_name="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant that can browse websites. "
               "Use ScraperAPITool to access web content."),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

executor.invoke({"input": "Browse hackernews and summarize the top story"})


Import from `langchain_scraperapi.tools`, not from the package root. Some tutorials show the shorter path; the official repository and documentation both use `.tools`, and that's the one that matches the published package layout.

The system prompt matters more than it looks. Without an instruction naming the tool, models frequently answer from memory instead of calling it, and you get a confident summary of last year's Hacker News front page.

## The three tools and the parameters worth setting

| Tool | Returns | Required | Options that change behavior |
| --- | --- | --- | --- |
| `ScraperAPITool` | Page content as raw HTML, `text` or `markdown` | `url` | `output_format`, `render`, `premium`, `country_code`, `device_type`, `keep_headers` |
| `ScraperAPIGoogleSearchTool` | Structured SERP data, `json` (default) or `csv` | `query` | `num`, `gl`, `hl`, `tld`, `country_code` |
| `ScraperAPIAmazonSearchTool` | Structured product results, `json` (default) or `csv` | `query` | `tld`, `page`, `country_code` |

Two things to know before you write your first call. `ScraperAPITool` returns raw HTML unless you pass `output_format`, which is the wrong default for an LLM: HTML burns context window and adds credits when the model asks for a second pass to re-extract the content. Set `"markdown"` and move on. Second, you'll see `wait` in some tutorials as a way to pause before capture. It isn't in the documented parameter list, so treat it as undocumented and don't build a pipeline around it.

For anything with a product or search dimension, use the structured tools instead of scraping the page yourself. The Amazon tool costs a fixed 5 credits and comes back as parsed fields; scraping Amazon product pages with `ScraperAPITool` costs the same 5 credits and leaves you writing the parser.

python
from langchain_scraperapi.tools import ScraperAPIAmazonSearchTool

amazon = ScraperAPIAmazonSearchTool()
products = amazon.invoke({
    "query": "noise cancelling headphones",
    "tld": "co.uk",
    "page": 2
})


## What a credit actually buys, and where agents blow the budget

This is the number the plan table hides. One credit is one standard page request. Everything else is a multiplier:

| Target or flag | Credits per request |
| --- | --- |
| Standard HTML page (blog, docs, news) | 1 |
| Amazon, eBay, Walmart | 5 |
| Google, Bing | 25 |
| LinkedIn | 30 |
| `render=true` (JavaScript execution) | +10 |
| `premium=true` (premium proxy) | +10 |
| `ultra_premium=true` | +30 |

Stack them and the math gets uncomfortable. An Amazon search with JavaScript rendering costs 15 credits, so the $49 Hobby plan's 100,000 credits deliver roughly 6,600 requests, not 100,000. The same 100,000 credits cover 100,000 plain documentation pages. That gap of roughly 15x is the single most common reason people buy the wrong tier.

Agents multiply it again. One `executor.invoke()` is a task, not a request: the model plans, calls the tool on a search page, calls it again on two or three of the results, and possibly retries one. Three to four tool calls per task is normal. Budget per task, then multiply by your expected task volume, then check the total against a plan.

Ways to keep the burn down without changing plans:

- Ask for `output_format: "markdown"` or `"text"` on every `ScraperAPITool` call. Less context, fewer follow-up calls.
- Leave `render` off unless the content genuinely requires JavaScript. It adds 10 credits every time.
- Cache SERP results. A Google query at 25 credits is the most expensive routine call in the toolkit, and ranking data doesn't change minute to minute.
- Check the Domain Cost Estimator in the dashboard for your actual target domains before you commit. It gives you the per-URL cost rather than a general rule of thumb.

👉 [Run the free trial against your real targets before picking a tier](https://www.scraperapi.com/?fp_ref=coupons)

Failed requests aren't billed. ScraperAPI's own documentation puts it as paying only for data that comes back clean, which matters more than it sounds when you're scraping sites that block aggressively: a 403 you never see doesn't cost you credits.

## Three limits the price column doesn't show

**Geotargeting is gated by tier.** Hobby and Startup route through US and EU proxies only. If your agent needs to see prices, search results or listings as a user in Brazil, Japan or Australia, you need Business at $299 and up. Passing a `country_code` on a lower tier doesn't fail loudly; you just get the wrong regional content, which is worse.

**There's no overage billing below Scaling.** On Hobby, Startup and Business, running out of credits mid-month pauses scraping until renewal unless you've switched on auto-upgrade, which moves you to the next tier at the next tier's price. From Scaling ($475) upward you can enable pay-as-you-go at a fixed per-credit rate with a monthly spending cap you set. If your volume is spiky, a lower tier plus a hard pause is a real operational risk.

**Concurrency limits reject rather than queue.** Exceed your plan's thread count and requests come back as HTTP 429 asking you to back off. Nothing is queued for you, and there's no surcharge. If your agent fans out parallel tool calls, the plan's thread count is the ceiling on that fan-out, not a soft guideline.

## Errors you'll hit in the first hour

**Authentication failures.** `SCRAPERAPI_API_KEY` has to exist inside the process that runs the agent. Exporting it in your terminal doesn't help if the script runs in Docker, a different virtualenv, or a systemd unit. Same for notebooks started before you exported it.

**Timeouts on hard targets.** Rendering JavaScript on a protected page takes time. Set your tool timeout to 60–90 seconds rather than the default, and wrap `tool.invoke` in a try/except with a fallback so one dead URL doesn't kill the whole run.

**Credits disappearing faster than expected.** Nine times out of ten it's `render=true` left on, or a target domain whose base cost is 5 or 25 rather than 1. Check the domain, then check the flags.

**The agent never calls the tool.** Rewrite the system prompt to name the tool explicitly, and check `verbose=True` output for whether a tool call was attempted at all.

## When you don't need this

If every page your agent reads is static HTML on a site that doesn't care about bots, LangChain's built-in `WebBaseLoader` fetches it for free and you can skip the whole dependency. That covers a lot of internal documentation and simple public sites.

The split shows up on three kinds of targets: pages that render their content with JavaScript, sites running Cloudflare or similar bot detection, and anything where you need clean markdown rather than a DOM to parse. `WebBaseLoader` fetches the URL directly, so on those it either fails or hands back a block page and raw HTML. That's the case ScraperAPI exists for, and it's also the case where a failed fetch silently poisons a RAG pipeline, because the model summarizes whatever it received.

## Which plan to pick

- **Evaluating, or a side project under a few thousand pages a month:** stay on the free tier's 1,000 monthly credits and use the 5,000-credit trial for the real test.
- **RAG over blogs, docs and news at 50,000–100,000 pages a month:** Hobby at $49. At 1 credit each, this is the tier's sweet spot.
- **Agents doing SERP monitoring or Google research:** do the arithmetic first. At 25 credits a query, Startup's million credits is 40,000 searches. If you need more than that, Business.
- **Amazon or marketplace price tracking:** 5 credits per structured search, 15 with rendering. Startup covers about 66,000 rendered Amazon searches a month.
- **Non-US/EU geotargeting of any kind:** Business, no way around it.
- **Volume you can't forecast:** Scaling, because that's where pay-as-you-go overflow begins.

Two things to confirm before you enter card details. First, run your actual target URLs through the Domain Cost Estimator so you know your true cost per request instead of the 1-credit headline. Second, check whether your monthly total lands near a tier boundary, because crossing from Startup to Business doubles the price and there's no partial-overage option to soften it. Cancellation is free at any time from the dashboard, and there's a 7-day no-questions refund if the trial numbers turn out worse than you expected.

👉 [Open a ScraperAPI account and price out your own target URLs](https://www.scraperapi.com/?fp_ref=coupons)
