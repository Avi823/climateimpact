# Trend Detective — NASA Space Apps web app

Trend Detective is a public-facing climate-trend explorer built from the supplied `trend-detective-v5.html`. The project keeps the Leaflet interface and in-browser trend statistics, while adding a FastAPI backend for NASA POWER requests, geocoding, runtime configuration, and Groq-powered impact assessments.

## What changed

- **FastAPI backend:** browser requests to NASA POWER, Nominatim, and Groq now go through `/api/...` routes.
- **Secrets:** the Groq API key is read only by the Python server. It is never embedded in the HTML, returned by `/api/config`, or saved in browser storage.
- **CARTO:** the current CARTO raster basemap URLs include `?key=...`. CARTO's key must appear in browser tile requests, so it is exposed as a public, referrer-restricted map key. Restrict it to your deployed hostname in the CARTO dashboard.
- **NASA POWER:** no NASA API key is needed for the monthly point-data endpoint used here.
- **Place lookup:** Nominatim search and reverse geocoding are proxied, cached, given an identifying User-Agent, and throttled to approximately one request per second. Place names and selected coordinates are sent to the app server and relevant providers to return results.
- **Human impact:** each report immediately shows variable-specific possible consequences for people, who may be most exposed, and practical mitigation and adaptation actions.
- **AI assessment:** Groq adds a qualitative, location-aware discussion of future impacts and recommended actions. It deliberately does not manufacture local SSP/CMIP6 numbers. A NASA POWER point record is not a downscaled climate simulation, nor does one variable quantify future deaths, displacement, or economic losses.
- **Basic abuse control:** the AI endpoint has a best-effort per-IP hourly cap (`AI_CALLS_PER_HOUR`, default 12). For a larger public launch, put a shared rate limiter/WAF in front of the service and monitor Groq usage.

## 1. Add your API keys

Copy `.env.example` to `.env`:

```bash
cp .env.example .env
```

Edit `.env`:

```dotenv
CARTO_BASEMAP_KEY=your_carto_basemap_key
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=openai/gpt-oss-120b
NOMINATIM_USER_AGENT=TrendDetective/1.0 (NASA Space Apps educational climate-data project)
AI_CALLS_PER_HOUR=12
```

### CARTO key

1. Request a basemap key at <https://carto.com/basemaps/apikey/>.
2. For local development, add a local host restriction in the CARTO dashboard (CARTO notes that `localhost` and `127.0.0.1` need their own key/restriction entry).
3. For production, restrict the key to your actual website hostname, such as `your-project.example.com`.
4. Keep the visible CARTO/OpenStreetMap attribution on the map.

**Important:** a CARTO basemap key is not a server secret. A public map needs to send it with each tile request. The correct protection is CARTO's website/referrer restriction and usage monitoring, not hiding it in Python.

### Groq key

1. Create a key in the Groq console: <https://console.groq.com/keys>.
2. Put it only in the server's `.env` file or your hosting provider's secret/environment-variable settings.
3. Do not paste it into `index.html`, commit `.env`, or send it to users.

The default model is `openai/gpt-oss-120b`. Override `GROQ_MODEL` only with a model enabled for your Groq account.

## 2. Run locally

Requires Python 3.10+.

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows PowerShell
.venv\Scripts\Activate.ps1

pip install -r requirements.txt
uvicorn app:app --reload
```

Open <http://127.0.0.1:8000>. Do not open `static/index.html` directly or use a static-only host: the page depends on the FastAPI `/api/...` routes.

Check configuration at <http://127.0.0.1:8000/api/health>. The endpoint reports whether keys are configured but never returns secret values.

## 3. Deploy publicly

This project includes a `Dockerfile` and can be deployed to a host that runs Docker (for example, Render, Railway, Fly.io, or a cloud container service).

1. Push this folder to a Git repository. Make sure `.env` is not committed; `.gitignore` already excludes it.
2. Create a web service using the repository's `Dockerfile`.
3. Add `CARTO_BASEMAP_KEY`, `GROQ_API_KEY`, `GROQ_MODEL`, and `AI_CALLS_PER_HOUR` as environment variables/secrets in the hosting dashboard.
4. Add the final website hostname to the CARTO key's referrer restrictions.
5. Open `/api/health` after deployment and confirm the service reports `carto_configured: true` and `groq_configured: true`.
6. Test a location search, a NASA investigation, the map-style switcher, and the AI assessment from the deployed HTTPS site.

The site and API are served from the same origin by FastAPI, so no separate frontend CORS configuration is required.

## API routes

| Route | Purpose |
|---|---|
| `GET /api/health` | Backend and key-configuration status; never returns key values |
| `GET /api/config` | Returns the public CARTO map key and whether Groq is configured |
| `GET /api/nasa/power` | Validates and proxies NASA POWER monthly point data; caches results |
| `GET /api/geocode` | Proxies and caches user-submitted place searches through Nominatim |
| `GET /api/reverse` | Proxies and caches user-triggered reverse geocoding through Nominatim |
| `POST /api/impact-assessment` | Validates local trend statistics and calls Groq server-side |

## Important limitations before a large public launch

- The built-in hourly AI cap and caches are in memory. They reset when the process restarts and are not shared between multiple worker instances. Use Redis or your hosting platform's rate-limiting service for a multi-instance launch.
- Every public AI call consumes your Groq account's quota. Keep usage limits enabled and monitor the provider dashboard. Consider CAPTCHA or user authentication if abuse becomes a concern.
- The public Nominatim endpoint is a limited community service. This app caches requests and serializes them, but a high-traffic public site should use a dedicated geocoding provider or a self-hosted Nominatim service.
- A historical linear trend is shown as a trend-continuation reference, not as a forecast. Real local future-impact estimates need climate-model ensembles plus local hazard, population, infrastructure, and vulnerability data.
- The qualitative AI narrative is educational synthesis, not an emergency warning, engineering specification, public-health forecast, or substitute for local climate-risk analysis.

## Sources used in the implementation

- CARTO Basemaps key setup and usage: <https://carto.com/basemaps/apikey/>
- NASA POWER monthly API: <https://power.larc.nasa.gov/docs/services/api/temporal/monthly/>
- Groq supported models and deprecations: <https://console.groq.com/docs/models> and <https://console.groq.com/docs/deprecations>
- Nominatim usage policy: <https://operations.osmfoundation.org/policies/nominatim/>


## Human-cost estimates: deaths, displacement, and economic loss

The report page has an **Estimate human cost** button (`POST /api/human-cost`). It returns ranges, per future decade (2030s, 2050s, 2080s), for extra deaths, people displaced, and economic loss per year if the measured trend continues.

How it works, and why the numbers are labelled estimates:

1. The server reverse-geocodes the point to a country (Nominatim) and pulls that country's population and GDP per person from the **World Bank API** (cached for 7 days).
2. Groq proposes the primary hazard, the number of people living within roughly 50 km, and low/high ranges, using a prompt that forbids invented citations and requires wider ranges when the trend is not significant.
3. The server **bounds** every figure: exposed people cannot exceed the country's population, annual deaths are capped at 0.5% of exposed people, displacement at 20%, and economic loss at 25% of the local annual output. Anything clamped is flagged to the user. Values are rounded to two significant figures.
4. If the trend is not statistically significant the page shows a "weak signal" warning and the server will not report high confidence.

These caps are sanity limits, not accuracy checks. A language model cannot produce validated local casualty or loss forecasts, and the page says so. For a defensible version, replace the model-estimated exposure with gridded population (WorldPop or GHSL) and replace the model's hazard numbers with CMIP6-driven impact functions, then keep the same bounding step. Points over open water or outside any country are rejected (HTTP 422). The endpoint shares the per-IP hourly quota logic with the impact assessment, in a separate bucket (`AI_CALLS_PER_HOUR` each).
