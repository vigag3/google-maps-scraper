# How to Scrape Google Maps with Python: A Practical Guide to Search Results, Business Data, Reviews, and Avoiding Blocks (Plus When to Use an API Instead)

If you've typed "scrape google maps with python" into a search bar, you're probably staring down one of two situations: you need a few hundred business listings for a lead list, or you're building something that needs this data on an ongoing basis. Either way, the first thing you'll discover is that Google Maps doesn't make this easy on purpose. There's no "export to CSV" button. The page is built on JavaScript that loads data as you scroll, the underlying HTML class names are scrambled and change without warning, and if you hit the site too aggressively you'll start seeing CAPTCHAs instead of business listings.

None of that means it's impossible. It just means you need to pick the right tool for the job — and that choice depends entirely on how much data you need and how often you need it.

## What's actually on a Google Maps listing (and why people scrape it)

Before getting into code, it's worth being clear about what you're extracting. A typical Google Maps business listing exposes:

- Business name and category (e.g., "Coffee Shop," "Hardware Store")

- Full address and GPS coordinates

- Phone number and website URL

- Star rating and review count

- Opening hours

- Photos (owner-uploaded and user-submitted)

- Written reviews and reviewer details

- Business posts (promotions, updates)

People scrape this for fairly predictable reasons: lead generation (pull every plumber in a metro area along with their phone number), competitor analysis, market research, and review sentiment tracking. It's a genuinely large dataset — there are reportedly over 200 million businesses listed on Google Maps, and most local consumers check the platform before they go anywhere.

## Three ways to get this data — and the real trade-offs

There isn't one "correct" way to scrape Google Maps. There are three approaches, and which one is right for you depends on volume, budget, and how much engineering time you want to spend on maintenance.

### Option 1: The official Google Places API

This is the sanctioned, ToS-compliant route. You set up a project in Google Cloud Console, enable the Places API, and get an API key. It returns clean, structured JSON for any listing if you already have its Place ID.

The catch is twofold. First, it's expensive at scale — pricing is per request and adds up fast once you're past a few thousand lookups. Second, it's limited in what it gives you. Most notably, the review limit through the official API is capped at five reviews per listing unless you actually own the business profile — there's no official way around that.

A minimal example using `requests`:

python

import requests

API_KEY = "YOUR_GOOGLE_API_KEY"

def search_places(query, location=None):

url = "https://places.googleapis.com/v1/places:searchText"

headers = {

"Content-Type": "application/json",

"X-Goog-Api-Key": API_KEY,

"X-Goog-FieldMask": "places.displayName,places.formattedAddress,"

"places.rating,places.userRatingCount,"

"places.internationalPhoneNumber,places.websiteUri"

}

payload = {"textQuery": query}

if location:

payload["locationBias"] = {

"circle": {

"center": {"latitude": location[0], "longitude": location[1]},

"radius": 5000.0

}

}

response = requests.post(url, json=payload, headers=headers)

return response.json()

results = search_places("italian restaurants in Brooklyn")

for place in results.get("places", []):

print(place.get("displayName", {}).get("text"), "-", place.get("formattedAddress"))



This is the right starting point if you need a small, defined dataset and want to stay fully within Google's terms.

### Option 2: Browser automation (Selenium / Playwright)

This is the "free but you'll pay for it in time" option. Since Google Maps renders results dynamically as you scroll, a script needs to drive an actual browser, wait for content to load, scroll repeatedly, and then parse the DOM.

Here's a simplified Playwright-based scraper that searches for a business type in a location and pulls names, ratings, and addresses from the results panel:

python

from playwright.sync_api import sync_playwright

import time

import csv

def scrape_google_maps(search_query, max_results=30):

results = []

with sync_playwright() as p:

browser = p.chromium.launch(headless=False)

page = browser.new_page()

page.goto(f"https://www.google.com/maps/search/{search_query}")

page.wait_for_timeout(3000)

# Handle cookie consent if it appears

try:

page.click("text=Accept all", timeout=3000)

except:

pass

# The results list scrolls independently of the page

feed = page.locator('div[role="feed"]')

seen = set()

while len(results) < max_results:

cards = feed.locator('div[role="article"]').all()

for card in cards:

name = card.locator('div.fontHeadlineSmall').first

if name.count() == 0:

continue

name_text = name.inner_text()

if name_text in seen:

continue

seen.add(name_text)

results.append({"name": name_text})

if len(results) >= max_results:

break

feed.evaluate("el => el.scrollBy(0, 800)")

time.sleep(1.5)

browser.close()

return results

data = scrape_google_maps("coffee shops in Austin TX", max_results=20)

with open("results.csv", "w", newline="", encoding="utf-8") as f:

writer = csv.DictWriter(f, fieldnames=["name"])

writer.writeheader()

writer.writerows(data)



A few things worth knowing before you build on this:

- **CSS classes are obfuscated and change.** Google ships randomly generated class names, so selectors break without warning. Targeting stable attributes (like `role="article"` or `aria-label`) tends to hold up longer than class-name selectors.

- **Infinite scroll needs explicit handling.** You have to scroll the results panel itself (not the whole page) and detect when no new listings load.

- **CAPTCHAs and IP bans are a real risk at volume.** Run this from a single residential IP for a handful of searches and you're probably fine. Run it in a loop across hundreds of queries and Google will notice.

This route is genuinely useful to learn from, and fine for small one-off pulls. It gets painful once you need hundreds of searches a day, because you're now also responsible for proxy rotation, retry logic, and keeping selectors updated every time Google tweaks its frontend.

### Option 3: A scraping API that handles the proxy/CAPTCHA layer for you

This is the middle path most teams land on once browser automation starts eating real engineering hours. Instead of managing your own pool of proxies and headless browsers, you send your request to a third-party API that handles IP rotation, JavaScript rendering, and CAPTCHA solving, and hands back the rendered HTML or parsed JSON.

[ScraperAPI](https://www.scraperapi.com/?fp_ref=coupons) is one of the more established options here. The integration is a single HTTP call — you send the Google Maps URL you want, and it returns the rendered page (or structured data, for supported targets like Google search results):

python

import requests

API_KEY = "YOUR_SCRAPERAPI_KEY"

def scrape_with_api(target_url):

payload = {

"api_key": API_KEY,

"url": target_url,

"render": "true" # needed since Maps results load via JS

}

response = requests.get("https://api.scraperapi.com/", params=payload)

return response.text

html = scrape_with_api("https://www.google.com/maps/search/coffee+shops+austin+tx")

# Parse `html` with BeautifulSoup from here



The practical benefit is that you stop being responsible for the cat-and-mouse game with anti-bot defenses — that's the part of the stack the provider maintains. The trade-off is cost per request and the fact that you're now depending on a third party for uptime and accuracy.

## What it actually costs to scrape Google search/Maps results via API

This is where people get tripped up, because pricing isn't a flat per-request rate — it's credit-based, and different target sites cost different amounts of credits. For ScraperAPI specifically: a standard webpage costs 1 credit, while harder-to-scrape targets cost more — Google and Bing searches (including subdomains) cost 25 credits per request, and sites with extra bot protection like Cloudflare or Datadome add 10 credits on top of the base cost. That Google-specific premium matters directly for this use case, since Maps search traffic falls under that higher-cost bracket — worth factoring in before you estimate your monthly volume.

There's also a free tier if you just want to test the waters: 1,000 free API credits per month with up to 5 concurrent connections, plus a 7-day window after signup with access to 5,000 free requests for larger-scale testing.

Here's the current plan breakdown (verified against the live pricing page):

| Plan | Monthly Price | Annual Price | API Credits/mo | Concurrent Threads | Notable Limits | Get Started |

|---|---|---|---|---|---|---|

| **Free** | $0 | — | 1,000 | 5 | US & EU regions; testing only | 👉 [Start free](https://www.scraperapi.com/?fp_ref=coupons) |

| **Hobby** | $49 | $44/mo (billed yearly) | 100,000 | 20 | US & EU regions only | 👉 [View Hobby plan](https://www.scraperapi.com/?fp_ref=coupons) |

| **Startup** | $149 | $134/mo (billed yearly) | 1,000,000 | 50 | US & EU regions only | 👉 [View Startup plan](https://www.scraperapi.com/?fp_ref=coupons) |

| **Business** | $299 | $269/mo (billed yearly) | 3,000,000 | 100 | Country-level geotargeting unlocked | 👉 [View Business plan](https://www.scraperapi.com/?fp_ref=coupons) |

| **Scaling** | $475 | $427/mo (billed yearly) | 5,000,000 | 200 | Country-level geotargeting | 👉 [View Scaling plan](https://www.scraperapi.com/?fp_ref=coupons) |

| **Enterprise** | Custom | Custom | Custom (3M+) | Custom | Dedicated support, custom SLAs | 👉 [Contact sales](https://www.scraperapi.com/?fp_ref=coupons) |

*Note: ScraperAPI's pricing page doesn't expose distinct sub-page URLs for each plan tier — all plan selection happens on the single pricing/checkout page, so each link above routes through the same tracked entry point.*

A quick sanity check on what that means for Maps scraping specifically: if Google-domain requests cost 25 credits each, the 100,000-credit Hobby plan gets you roughly 4,000 Maps/Google search requests a month before you'd need to upgrade or top up — useful math to do before committing to a tier. You can also set a max_cost per scrape in the dashboard to cap what any single request is allowed to spend, which is a reasonable safety net if you're scraping a mix of target types.

If you outgrow a plan mid-cycle, you're not stuck — you can either auto-upgrade to the next tier for a typically better per-credit rate, or on Professional/Advanced/Enterprise plans, switch to pay-as-you-go at a fixed rate. And if you change your mind entirely, cancellation is straightforward: you can cancel anytime from the dashboard with no cancellation fee, and there's a 7-day no-questions-asked refund window if the service isn't a fit.

## Handling reviews specifically

Reviews are usually the trickiest part of any Google Maps scrape, for a structural reason, not a technical one: the official API artificially limits you to five reviews per listing unless you own the business. If review text and sentiment matter to your project — say, you're doing competitor analysis on customer complaints — you have two realistic paths: drive a headless browser into the reviews tab and scroll/parse manually, or route the request through a scraping API/service that's built to handle that specific endpoint.

Either way, plan for this part of the project to take longer than the basic listing scrape. Review panes load lazily, support multiple sort orders, and (if you're doing this for real analysis) you'll want pagination logic so you're not just capturing the first page Google shows you.

## Legal and ethical considerations — don't skip this part

This part isn't optional. A few ground rules that hold across jurisdictions and use cases:

> Google's Terms of Service restrict automated data collection, and large-scale scraping can violate those terms. Stick to publicly visible information, avoid aggressive request rates that could degrade the site for other users, and don't store personal data on EU residents in ways that would trigger GDPR obligations.

Practically, that means: scraping a few hundred business listings for your own lead research is a different risk profile than scraping millions of records and reselling the dataset. If you're doing anything at real commercial scale, it's worth having an actual conversation with someone who understands the legal exposure in your jurisdiction before you build the pipeline.

## So which approach should you actually use?

It comes down to volume and how much ongoing maintenance you're willing to take on:

- **Need under a few hundred records, once?** Write a quick Playwright script. It'll take an afternoon and you'll learn something.

- **Need a defined, smaller dataset and want to stay strictly within Google's terms?** Use the official Places API and accept its field and review limitations.

- **Need this running reliably, repeatedly, and don't want to be the person debugging broken CSS selectors at 11pm?** A scraping API is the pragmatic choice — you're paying for someone else to own the proxy rotation and anti-bot arms race so you can focus on what you do with the data.

If you go the third route, it's worth starting on [ScraperAPI's free tier](https://www.scraperapi.com/?fp_ref=coupons) to test request volume and credit costs against your actual use case before committing to a paid plan — the 1,000 free credits are enough to get a real feel for how many Maps/Google-domain requests you'll burn through per search, given the 25-credit-per-request rate on that target.

Whichever path you pick, start small. Scrape ten listings before you try to scrape ten thousand — it's a lot cheaper to find out your selector or credit math is wrong on a small test run than after you've kicked off an overnight job.

---

*Disclosure: This article contains an affiliate link to ScraperAPI. If you sign up through it, this site may earn a commission at no extra cost to you. Pricing and plan details were verified against ScraperAPI's live pricing page; always confirm current pricing and terms on their site before purchasing, as providers update plans periodically.*
