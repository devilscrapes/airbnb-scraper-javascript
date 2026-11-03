<p align="center"><a href="https://apify.com/DevilScrapes/airbnb-scraper"><img src="assets/banner.svg" alt="Airbnb Listings &amp; Pricing Scraper — JavaScript example" width="100%"></a></p>

# Airbnb Scraper for Node.js: listing and price data for apps and backends

<p align="center">
<a href="https://apify.com/DevilScrapes/airbnb-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/airbnb-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/airbnb-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to run the **[Airbnb Listings & Pricing Scraper](https://apify.com/DevilScrapes/airbnb-scraper)** from JavaScript. It is a single ES module built around the official `apify-client` package. Your code sends a place name or a saved Airbnb search URL, plus optional dates and guest count, and gets back an array of home objects: stable listing id, `/rooms/` link, title, room type, map coordinates, nightly rate and stay total, rating, review count, superhost flag, room counts and up to twenty photo URLs. Apify handles the browsing through residential exits in the country you choose, so the Node service, cloud function or job queue calling it stays small.

> **Actor:** [Airbnb Listings & Pricing Scraper](https://apify.com/DevilScrapes/airbnb-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [airbnb-scraper-python](https://github.com/devilscrapes/airbnb-scraper-python)

## 🔥 What you get

- **Two ways to aim a search**: type a place such as `Lisbon, Portugal` and the Actor builds the `/s/<place>/homes` address for you, or paste a URL you filtered on airbnb.com and it is used exactly as given.
- **Stay totals that match real dates**: supply both `checkIn` and `checkOut` and `total_price` carries the amount Airbnb shows for that stay; leave the dates out and the column stays empty instead of holding a guess.
- **Room shape in numbers**: bedrooms, beds and bathrooms are parsed out of each card's summary line into integers and decimals, so half baths survive as `1.5`.
- **Every listing id once**: Airbnb's result pages overlap as you scroll; repeats are dropped before they reach the dataset or the bill. Up to 1,000 homes per run, 18 per results page.
- **Exit country you choose**: requests always leave through Apify's residential pool in the country given by `proxyCountry`, which matters because Airbnb localises prices and currency to the visitor.
- **Partial runs say so**: if Airbnb stops answering midway, the rows already collected are kept and the status message reads how many arrived out of how many were asked for. Throttled requests get a few spaced-out retries before that happens.

## 🎯 Use cases

### Feed a dynamic-pricing engine

A nightly worker can pull comparable homes for each market you manage and hand `price_per_night`, `rating` and `bedrooms` to your pricing model. Upsert by `listing_id` so the comp table stays one row per home across runs.

### Show nearby rentals on a map

Travel and relocation apps can turn `lat`, `lng` and the first photo into map pins. Run a search when a user picks a city, cache the result for a few hours and serve it from your own API.

### Quote a stay inside a chat or booking flow

Pass the user's dates as `checkIn` and `checkOut`, wait for the run in a serverless function and reply with the cheapest few `total_price` values and their links. The rows are plain JSON, easy to drop into a Slack or WhatsApp message.

### Alert when a market's prices shift

Store yesterday's median nightly rate per city, compare after each scheduled run and post to a webhook when the change crosses a threshold you choose. Revenue managers hear about a local event spike before their own bookings show it.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/airbnb-scraper-javascript.git
cd airbnb-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Airbnb Listings & Pricing Scraper](https://apify.com/DevilScrapes/airbnb-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

This is the integration with nothing extra: `call()` starts the run and resolves when it ends, then `listItems()` returns its dataset rows.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/airbnb-scraper').call({"location": "Austin, TX", "maxResults": 10, "proxyConfiguration": {"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"], "apifyProxyCountry": "US"}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` submits `input.json` verbatim. The example asks for ten homes in Austin, Texas through US residential exits, small enough for a first look at real data. Swap `location` for `searchUrl` when product people have already built the perfect filtered search in their browser.

```json
{
  "location": "Austin, TX",
  "maxResults": 10,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ],
    "apifyProxyCountry": "US"
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `location` | string | no | — | — |
| `searchUrl` | string | no | — | — |
| `checkIn` | string | no | — | — |
| `checkOut` | string | no | — | — |
| `guests` | integer | no | `1` | — |
| `maxResults` | integer | no | `50` | — |
| `enableDetailEnrichment` | boolean | no | `false` | — |
| `proxyCountry` | string | no | `"US"` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"], "apifyProxyCountry": "US"}` | — |

## 📤 Output

Each array element is a home object. Fields the search card lacks come back `null`, so guard with optional chaining before arithmetic. `total_price` is populated only when both dates were sent. `host_id`, `host_name`, `country`, `listing_type` and `max_guests` are present in the schema but empty in practice, so do not build features on them. `scraped_at` is a UTC timestamp string.

| Field | Type | Always present |
|---|---|---|
| `listing_id` | string | yes |
| `url` | string | yes |
| `title` | string or null | no |
| `listing_type` | string or null | no |
| `room_type` | string or null | no |
| `city` | string or null | no |
| `country` | string or null | no |
| `lat` | number or null | no |
| `lng` | number or null | no |
| `price_per_night` | number or null | no |
| `currency` | string or null | no |
| `total_price` | number or null | no |
| `rating` | number or null | no |
| `review_count` | integer or null | no |
| `host_id` | string or null | no |
| `host_name` | string or null | no |
| `is_superhost` | boolean or null | no |
| `bedrooms` | integer or null | no |
| `beds` | integer or null | no |
| `bathrooms` | number or null | no |
| `max_guests` | integer or null | no |
| `amenities` | array | no |
| `image_urls` | array | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Result scraped | $0.0029 |

Prices as configured on 2026-11-03. The [Actor page](https://apify.com/DevilScrapes/airbnb-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

The module below relies on Node.js built-ins only. It converts saved rows into a GeoJSON `FeatureCollection` that Mapbox, Leaflet or deck.gl load directly, then logs the median nightly rate of the mapped homes.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const listings = JSON.parse(await readFile('results.json', 'utf8'));
const features = listings
    .filter((row) => row.lat != null && row.lng != null)
    .map((row) => ({
        type: 'Feature',
        geometry: { type: 'Point', coordinates: [row.lng, row.lat] },
        properties: {
            id: row.listing_id,
            title: row.title,
            nightly: row.price_per_night,
            currency: row.currency,
            rating: row.rating,
            superhost: row.is_superhost,
            photo: row.image_urls?.[0] ?? null,
            url: row.url,
        },
    }));

await writeFile('listings.geojson', JSON.stringify({ type: 'FeatureCollection', features }));
const priced = features.map((f) => f.properties.nightly).filter((n) => n != null).sort((a, b) => a - b);
const median = priced.length ? priced[Math.floor(priced.length / 2)] : null;
console.log(`${features.length} mapped listings, median nightly rate ${median ?? 'n/a'}`);
```

## ❓ FAQ

### How long does a run take, and should my API wait for it?

Results pages are fetched one after another with a short pause between them, so a few hundred homes take minutes, not seconds. For user-facing requests, start the run, return a job id and finish the work from Apify's run-finished webhook.

### Are prices converted to one currency?

No. Prices arrive as Airbnb displayed them to a visitor from your `proxyCountry`, with `currency` set to USD, EUR or GBP when the symbol is recognisable. Convert in your own service if you mix markets.

### What happens if Airbnb blocks part of the way through?

Homes already read are kept and charged, the run still finishes, and the status message reports the shortfall against `maxResults`. If even the first results page cannot be read, the run fails so your job sees a real error.

### Can I call it from TypeScript or a Next.js route?

Yes. `apify-client` ships its own typings and works in any Node 20 runtime. Keep the token in a server-side environment variable; never expose it to the browser bundle.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/airbnb-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Airbnb Listings & Pricing Scraper on Apify Store](https://apify.com/DevilScrapes/airbnb-scraper)
- [Python version of this example](https://github.com/devilscrapes/airbnb-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Hostelworld Hostel Listings Scraper](https://apify.com/DevilScrapes/hostelworld-hostel-listings-scraper) — budget beds and hostel ratings for the same destinations.
- [Furnished Finder Corporate Housing Scraper](https://apify.com/DevilScrapes/furnished-finder-corporate-housing-scraper) — monthly furnished rentals, the mid-term side of the market.
- [Hipcamp Campsite Listings Scraper](https://apify.com/DevilScrapes/hipcamp-campsite-listings-scraper) — outdoor stays and glamping sites priced per night.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/airbnb-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Airbnb, Inc..
