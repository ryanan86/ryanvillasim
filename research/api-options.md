# Bali Villa Pricing API Options - Research Results

**Research date:** March 2026
**Use case:** Static HTML site fetching average nightly rates for villas/hotels in Seminyak, Canggu, Ubud, and Sanur (Bali, Indonesia)
**Key constraint:** Must be callable from browser JavaScript (CORS-enabled or JSONP)

---

## Executive Summary

| API | Free Tier | CORS / Browser | Price Data | Verdict |
|-----|-----------|----------------|------------|---------|
| Xotelo | Fully free, no key | Likely yes (no auth header needed) | TripAdvisor-sourced OTA rates | **Best option to test first** |
| Makcorps Free | 30 API calls, no CC | Unknown (JWT auth) | 200+ OTAs, real prices | Limited — good for PoC only |
| RapidAPI (Booking.com / Hotels4) | ~500 req/mo | Yes — RapidAPI does support browser fetch | Real-time hotel prices | Solid free option with API key |
| SerpAPI (Google Hotels) | 250 searches/mo | No — server-side only | Google Hotels prices | Needs proxy/backend |
| Apify (Airbnb scraper) | $5/mo credit (~1,600 results) | Discouraged from browser | Airbnb nightly rates | Needs backend or async run |
| Booking.com Demand API | Free (partner only) | No — requires approval | Booking.com inventory | Gated — not practical |
| HotelAPI.co | 100 calls on signup | Unknown | Multi-OTA prices | Worth testing |

---

## Option 1: Xotelo — Free Hotel Prices API

**URL:** https://xotelo.com/
**Data source:** TripAdvisor (rates from Hotels.com, Expedia, Booking.com, Agoda, etc.)

### Authentication
- None required. Fully open API, no API key, no signup.

### Endpoints

```
Base: https://data.xotelo.com/api/

GET /rates?hotel_key=<key>&chk_in=YYYY-MM-DD&chk_out=YYYY-MM-DD
GET /heatmap?hotel_key=<key>
GET /list?location_key=<key>&offset=0&limit=30&sort=best_value
GET /search?query=seminyak+bali&location_type=accommodation
```

### Example: Fetch hotel list for a Bali location

```javascript
// Step 1: Get location key for Seminyak
fetch('https://data.xotelo.com/api/search?query=seminyak+bali&location_type=accommodation')
  .then(r => r.json())
  .then(data => console.log(data));

// Step 2: Get rates for a specific hotel
// hotel_key format: g297930-d305178 (from TripAdvisor URL)
fetch('https://data.xotelo.com/api/rates?hotel_key=g297930-d305178&chk_in=2026-04-10&chk_out=2026-04-11')
  .then(r => r.json())
  .then(data => console.log(data));
```

### How to get hotel_key

From any TripAdvisor hotel URL, extract the segment between `Hotel_Review-` and `-Reviews`:
- URL: `tripadvisor.com/Hotel_Review-g297930-d305178-Reviews-...`
- Key: `g297930-d305178`

For Bali area location keys (for `/list` endpoint):
- Seminyak: find via `/search?query=seminyak`
- Canggu, Ubud, Sanur: same approach

### Example response (`/rates`)

```json
{
  "timestamp": 1710000000,
  "error": null,
  "result": {
    "chk_in": "2026-04-10",
    "chk_out": "2026-04-11",
    "rates": [
      { "name": "Hotels.com",  "rate": 85,  "currency": "USD" },
      { "name": "Expedia",     "rate": 87,  "currency": "USD" },
      { "name": "Booking.com", "rate": 82,  "currency": "USD" },
      { "name": "Agoda",       "rate": 80,  "currency": "USD" }
    ]
  }
}
```

### CORS status
- No API key means no custom auth headers required.
- The API appears designed for direct usage. No documentation warns against browser use.
- **Recommended first test:** open browser console and run:
  ```javascript
  fetch('https://data.xotelo.com/api/search?query=seminyak+bali')
    .then(r => r.json()).then(console.log)
  ```

### Rate limits
- Not published. Appears to be community-supported / donation-funded.
- No hard documented cap. Use conservatively (cache results, avoid hammering).

### Free tier
- Completely free. No account needed. Accepts donations.

### Caveats
- Tied to TripAdvisor data; hotel keys derived from TripAdvisor URLs
- No guarantee of uptime (run by individual developer)
- Not all Bali villas may be listed on TripAdvisor

---

## Option 2: Makcorps Hotel Price API — Free Trial

**URL:** https://www.makcorps.com/
**Docs:** https://docs.makcorps.com/hotel-price-apis

### Authentication
JWT token required. You must first call the auth endpoint:

```
GET https://api.makcorps.com/auth
```
Returns a JWT token, which is then passed as `Authorization: JWT <token>` on subsequent calls.

### Free Tier
- **30 API calls total** (not per month — a one-time free trial)
- No credit card required
- Returns 30 hotels for any city
- Dates are random future dates (no check-in/out control on free endpoint)

### Free endpoint

```
GET https://api.makcorps.com/free/{city_name}
Authorization: JWT eyJ0eXAiOiJKV1Qi...
```

Example:
```bash
curl -X GET \
  -H "Authorization: JWT eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..." \
  https://api.makcorps.com/free/seminyak
```

### Example response

```json
[
  {
    "hotelName": "The Layar Seminyak",
    "hotelId": "654321",
    "prices": [
      { "vendor": "Booking.com", "price": 210, "tax": 30, "currency": "USD" },
      { "vendor": "Expedia",     "price": 215, "tax": 28, "currency": "USD" },
      { "vendor": "Hotels.com",  "price": 218, "tax": 30, "currency": "USD" }
    ]
  }
]
```

### CORS status
- Uses JWT in Authorization header — browsers can send custom headers cross-origin only if the server sends `Access-Control-Allow-Headers`.
- Documentation does not confirm CORS support.
- **Likely needs a proxy** for browser use. Test with a CORS probe first.

### Paid tiers (for reference)
| Plan | Price | Requests |
|------|-------|----------|
| Free Trial | $0 | 30 calls |
| Basic | $350/mo | 10,000/mo |
| Advance | $500/mo | 50,000/mo |

### Verdict for this use case
- 30 calls is enough to build a proof-of-concept only.
- If CORS works, useful for initial data; otherwise needs a thin serverless proxy.

---

## Option 3: RapidAPI — Booking.com / Hotels4 / Booking.com API V2

RapidAPI is a marketplace hosting several unofficial hotel scraper APIs. All use the same authentication pattern.

**Key APIs:**
- `tipsters/booking-com`: https://rapidapi.com/tipsters/api/booking-com
- `apidojo/hotels4` (Hotels.com data): https://rapidapi.com/apidojo/api/hotels4
- `georgekhananaev/booking-com-api-v2`: https://rapidapi.com/georgekhananaev/api/booking-com-api-v2

### Authentication
All RapidAPI calls require two headers:

```
X-RapidAPI-Key: YOUR_RAPIDAPI_KEY
X-RapidAPI-Host: booking-com.p.rapidapi.com
```

Sign up free at https://rapidapi.com — no credit card required for free tier.

### Free tier
- ~500 requests/month (varies by individual API; some set 500, some up to 1,000)
- RapidAPI platform-wide free tier: confirmed available

### CORS status
**RapidAPI DOES support browser-based JavaScript fetch.** The platform is designed to work with both server and client requests. The required headers (`X-RapidAPI-Key`, `X-RapidAPI-Host`) are standard HTTP headers that browsers can send after a preflight OPTIONS check. RapidAPI's infrastructure handles the CORS response headers.

**Important:** Your API key will be visible in browser source code. For a static site with low traffic / public data, this is an accepted tradeoff. For production, use a thin serverless proxy (Cloudflare Worker, Netlify Function) to hide the key.

### Example: Hotels4 / Hotels.com API

```javascript
const options = {
  method: 'GET',
  headers: {
    'X-RapidAPI-Key': 'YOUR_KEY_HERE',
    'X-RapidAPI-Host': 'hotels4.p.rapidapi.com'
  }
};

// Step 1: Search for a location
fetch('https://hotels4.p.rapidapi.com/locations/v3/search?q=Seminyak+Bali&locale=en_US&langid=1033&siteid=300000001', options)
  .then(r => r.json())
  .then(data => {
    const destinationId = data.sr[0].gaiaId;
    // Step 2: Search hotels with that destination
    return fetch(`https://hotels4.p.rapidapi.com/properties/v2/list`, {
      method: 'POST',
      headers: { ...options.headers, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        currency: "USD",
        eapid: 1,
        locale: "en_US",
        siteId: 300000001,
        destination: { regionId: destinationId },
        checkInDate: { day: 10, month: 4, year: 2026 },
        checkOutDate: { day: 11, month: 4, year: 2026 },
        rooms: [{ adults: 2 }],
        resultsStartingIndex: 0,
        resultsSize: 20,
        sort: "PRICE_LOW_TO_HIGH"
      })
    });
  })
  .then(r => r.json())
  .then(data => console.log(data.data.propertySearch.properties));
```

### Example response (Hotels4)

```json
{
  "data": {
    "propertySearch": {
      "properties": [
        {
          "name": "Villa Seminyak Estate & Spa",
          "id": "12345678",
          "price": {
            "lead": { "amount": 120, "formatted": "$120" },
            "strikeThrough": { "amount": 150 }
          },
          "reviews": { "score": 8.6, "total": 342 },
          "destinationInfo": { "distanceFromDestination": { "value": 0.3, "unit": "MILE" } }
        }
      ]
    }
  }
}
```

### Example: Booking.com API (tipsters)

```javascript
const options = {
  method: 'GET',
  headers: {
    'X-RapidAPI-Key': 'YOUR_KEY_HERE',
    'X-RapidAPI-Host': 'booking-com.p.rapidapi.com'
  }
};

// Search for destination ID
fetch('https://booking-com.p.rapidapi.com/v1/hotels/locations?name=Seminyak&locale=en-gb', options)
  .then(r => r.json())
  .then(locations => {
    const destId = locations[0].dest_id;
    return fetch(`https://booking-com.p.rapidapi.com/v1/hotels/search?dest_id=${destId}&dest_type=region&checkin_date=2026-04-10&checkout_date=2026-04-11&adults_number=2&room_number=1&locale=en-gb&currency=USD&order_by=price&filter_by_currency=USD`, options);
  })
  .then(r => r.json())
  .then(data => console.log(data.result));
```

### Rate limits (free tier)
- tipsters/booking-com: ~500 requests/month
- apidojo/hotels4: ~500 requests/month
- Both reset monthly

---

## Option 4: SerpAPI — Google Hotels

**URL:** https://serpapi.com/google-hotels-api

### Authentication
API key via `api_key` query parameter.

### Free tier
- **250 searches/month** — confirmed on their pricing page
- No credit card required for free plan

### CORS status
**Not browser-safe.** SerpAPI is a server-side scraping service. The API key must be kept secret and the docs do not mention CORS headers. Calling from browser JavaScript would expose your key and likely fails CORS checks.

**Requires a backend proxy** (Cloudflare Worker, AWS Lambda, etc.).

### Example request (server-side / proxy)

```
GET https://serpapi.com/search
  ?engine=google_hotels
  &q=Seminyak+Bali+villas
  &check_in_date=2026-04-10
  &check_out_date=2026-04-11
  &adults=2
  &currency=USD
  &gl=id
  &hl=en
  &api_key=YOUR_KEY
```

### Example response

```json
{
  "search_parameters": { "q": "Seminyak Bali villas", "check_in_date": "2026-04-10" },
  "search_information": { "total_results": 15000 },
  "properties": [
    {
      "name": "The Layar - Private Pool Villas",
      "type": "vacation rental",
      "rate_per_night": {
        "lowest": "$210",
        "extracted_lowest": 210,
        "before_taxes_fees": "$180"
      },
      "overall_rating": 4.8,
      "reviews": 312,
      "location_rating": 4.7,
      "gps_coordinates": { "latitude": -8.691, "longitude": 115.162 }
    }
  ]
}
```

### Strengths
- Queries Google Hotels directly — excellent coverage of Bali villas
- Supports `vacation_rentals=true` filter — ideal for villa use case
- Supports `bedrooms`, `bathrooms`, `min_price`, `max_price` filters
- Returns real nightly rates with tax-separated figures

### Verdict
Best data quality for Bali villas, but requires a proxy. 250 free searches/month is enough for a small static site if results are cached.

---

## Option 5: Apify — Airbnb Scraper

**URL:** https://apify.com/tri_angle/airbnb-scraper
**API base:** https://api.apify.com/v2/

### Authentication
Apify API token — passed via `Authorization: Bearer <token>` header or `?token=` query param.

### Free tier
- **$5/month in platform credits** — no credit card required
- Cost: ~$1.25 per 1,000 results
- $5 credit = ~4,000 Airbnb listings per month
- Bali villa search (240 results) costs ~$0.30 — about 16 searches/month free

### CORS / browser status
**Not recommended from browser.** Apify explicitly warns: "Do not share the API token with untrusted parties, or use it directly from client-side code, unless you fully understand the consequences."

However, the API technically returns CORS headers (the `apify-client` npm package supports browser environments). If you use a read-only token scoped to a single dataset, the risk is lower.

### Async run pattern (for static site)

The recommended approach for a static site is a **pre-run / scheduled scrape** rather than real-time browser calls:

1. Run the actor on a schedule (daily/weekly via Apify scheduler)
2. Store results in an Apify Dataset
3. Expose the dataset as a public JSON endpoint (datasets can be made public)
4. Static site fetches the pre-built public dataset URL — no auth needed

```javascript
// Fetch from a public Apify dataset (no auth, CORS-friendly)
fetch('https://api.apify.com/v2/datasets/YOUR_DATASET_ID/items?format=json&clean=true')
  .then(r => r.json())
  .then(listings => {
    const avgPrice = listings
      .map(l => l.price)
      .reduce((a, b) => a + b, 0) / listings.length;
    console.log('Average nightly rate: $' + avgPrice);
  });
```

### Actor input for Bali villas

```json
{
  "locationQuery": "Seminyak, Bali, Indonesia",
  "maxItems": 50,
  "includeReviews": false,
  "currency": "USD",
  "checkIn": "2026-04-10",
  "checkOut": "2026-04-11",
  "adults": 2,
  "minBedrooms": 1
}
```

### Example result item

```json
{
  "name": "Stunning 3BR Villa with Private Pool",
  "url": "https://www.airbnb.com/rooms/12345678",
  "price": 185,
  "currency": "USD",
  "rating": 4.92,
  "reviewsCount": 127,
  "location": "Seminyak, Bali",
  "bedrooms": 3,
  "bathrooms": 3,
  "amenities": ["Pool", "Air conditioning", "WiFi"]
}
```

### Verdict
Best for Airbnb-specific villa pricing. The scheduled + public dataset pattern solves the CORS problem cleanly. Ideal if Airbnb data is acceptable as a proxy for market rates.

---

## Option 6: Booking.com Demand API (Official)

**URL:** https://developers.booking.com/demand/docs/open-api/demand-api

### Access model
- Free — no commission charged
- **Gated:** must apply and be approved as an Affiliate Partner
- Approval process: submit application at https://partnerships.booking.com

### Authentication
OAuth2 / API key with `X-Affiliate-Id` header. Credentials issued after partner approval.

### CORS status
Not designed for direct browser use. Enterprise-grade API for server integrations.

### Current status (March 2026)
Connectivity API applications are paused; Affiliate Partner applications remain open. Approval is not guaranteed, and the process can take weeks.

### Verdict
Not suitable for this use case. Too much friction for a static site project.

---

## Option 7: HotelAPI.co

**URL:** https://hotelapi.co/
**Docs:** https://docs.hotelapi.co/free-hotel-api

### Free tier
- **100 free API calls** on signup (no credit card required)
- Free endpoint: `GET https://api.makcorps.com/free/{city}` (appears to be powered by Makcorps infrastructure)

### Authentication
JWT token (same pattern as Makcorps free API)

### CORS status
Unknown — uses Authorization header, so CORS support needs verification.

### Verdict
Worth testing if Makcorps free tier proves useful. May share the same backend.

---

## Recommended Implementation Strategy

### Tier 1: Test first (no cost, minimal setup)

**Xotelo** — no key, no signup, potentially CORS-open

```javascript
// Quick viability test — paste in browser console right now:
fetch('https://data.xotelo.com/api/search?query=seminyak+bali+villa')
  .then(r => r.json())
  .then(console.log);
```

If this works, use the `/list` endpoint to pull hotel lists by Bali region, then batch `/rates` calls.

### Tier 2: Free tier with API key (best data quality)

**RapidAPI (Hotels4 or Booking.com tipsters)** — 500 req/mo free, browser-compatible

- Sign up at rapidapi.com (free, no CC)
- Subscribe to Hotels4 or booking-com API (free tier)
- API key is exposed in browser JS — acceptable for public rate data on a static site
- Cache results in localStorage to minimize API calls

```javascript
const CACHE_TTL = 6 * 60 * 60 * 1000; // 6 hours

async function getCachedRates(area) {
  const cacheKey = `rates_${area}`;
  const cached = localStorage.getItem(cacheKey);
  if (cached) {
    const { data, ts } = JSON.parse(cached);
    if (Date.now() - ts < CACHE_TTL) return data;
  }
  const data = await fetchRatesFromRapidAPI(area);
  localStorage.setItem(cacheKey, JSON.stringify({ data, ts: Date.now() }));
  return data;
}
```

### Tier 3: Best data, needs proxy

**SerpAPI (Google Hotels)** — 250 searches/mo free, returns villa-specific pricing

Deploy a free Cloudflare Worker as proxy:
```javascript
// cloudflare-worker.js
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const area = url.searchParams.get('area') || 'Seminyak Bali';
    const serpUrl = `https://serpapi.com/search?engine=google_hotels&q=${encodeURIComponent(area + ' villas')}&check_in_date=2026-04-10&check_out_date=2026-04-11&vacation_rentals=true&api_key=YOUR_KEY`;
    const resp = await fetch(serpUrl);
    const data = await resp.json();
    return new Response(JSON.stringify(data), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      }
    });
  }
};
```

---

## Data Coverage: Bali Areas

| Area | Xotelo | RapidAPI/Hotels4 | SerpAPI | Apify/Airbnb |
|------|--------|-------------------|---------|--------------|
| Seminyak | Good (TripAdvisor has many listings) | Good | Excellent | Excellent |
| Canggu | Good | Good | Excellent | Excellent |
| Ubud | Good | Good | Excellent | Good |
| Sanur | Moderate | Good | Good | Good |

**Note:** For villa-specific pricing (vs hotels), Airbnb via Apify and SerpAPI with `vacation_rentals=true` will give the most relevant data. TripAdvisor/Xotelo and Hotels.com/Hotels4 skew toward hotel properties.

---

## References

1. [Xotelo - Free Hotel Prices API](https://xotelo.com/)
2. [Xotelo - Hotel Key Guide](https://xotelo.com/how-to-get-hotel-key.html)
3. [Makcorps Hotel Price API](https://www.makcorps.com/)
4. [Makcorps API Documentation](https://docs.makcorps.com/hotel-price-apis)
5. [HotelAPI.co Free Hotel API](https://docs.hotelapi.co/free-hotel-api)
6. [RapidAPI - Booking.com (tipsters)](https://rapidapi.com/tipsters/api/booking-com)
7. [RapidAPI - Hotels4 (apidojo)](https://rapidapi.com/apidojo/api/hotels4)
8. [RapidAPI - Booking.com API V2](https://rapidapi.com/georgekhananaev/api/booking-com-api-v2)
9. [SerpAPI - Google Hotels API](https://serpapi.com/google-hotels-api)
10. [SerpAPI - Pricing](https://serpapi.com/pricing)
11. [Apify - Airbnb Scraper](https://apify.com/tri_angle/airbnb-scraper)
12. [Apify - Pricing](https://apify.com/pricing)
13. [Booking.com Demand API](https://developers.booking.com/demand/docs/open-api/demand-api)
14. [Free & Paid Hotel APIs 2026](https://phptravels.com/blog/what-is-a-hotel-api-and-why-does-it-matter)
15. [Makcorps on RapidAPI](https://rapidapi.com/manthankool/api/makcorps-hotel-price-comparison)
