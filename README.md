# API-Data-Mashup

**Country Explorer with Live Weather** — ITMGT 45.03 individual finals.

## Project Description

Country Explorer is a single-page web application that lets users search for any country and instantly view its key data alongside the current weather in its capital city. Each result is displayed as a card showing the country flag, name, capital, population, region, live temperature, wind speed, and a color-coded "Travel Suitability" badge that classifies current conditions as Excellent, Good, Fair, or Poor. The app is a client-side mashup of two public APIs — no server, build step, or API key required.

## APIs Used

### [REST Countries](https://restcountries.com/)
Free, no-authentication REST API that provides detailed information about countries including name, capital city, population, region, flag images, and geographic coordinates. The app queries the `/v3.1/name/{searchTerm}` endpoint to find countries matching the user's input.

### [Open-Meteo](https://open-meteo.com/)
Free, no-authentication weather API with an open CORS policy. It provides current weather data including temperature, weather code, and wind speed for any latitude/longitude coordinate. No rate limits apply for casual use. The app queries the forecast endpoint for each country's capital city coordinates.

## Setup Instructions

No installation or build step is required. The entire application is a single `index.html` file with inline CSS and JavaScript.

1. **Clone or download** the repository:
   ```bash
   git clone https://github.com/your-username/API-Data-Mashup.git
   cd API-Data-Mashup
   ```

2. **Open in a browser** — simply double-click `index.html` or open it from your browser's File menu. Alternatively, serve it locally:
   ```bash
   # Using Python (built-in on macOS/Linux):
   python3 -m http.server 8000
   # Then visit http://localhost:8000
   ```

   No web server is strictly necessary because both APIs use open CORS policies that accept requests from `file://` origins.

## Data Integration Explanation

The application integrates data from REST Countries and Open-Meteo using a **join key**: the capital city coordinates returned by REST Countries are used as the latitude/longitude parameters for the Open-Meteo forecast request.

**How the join works:**

1. The user enters a country name (e.g., "Philippines").
2. The app calls `https://restcountries.com/v3.1/name/philippines`, which returns an array of matching countries. For each country, it extracts `name.common`, `capital[0]`, `population`, `region`, `flags.svg`, and critically, `capitalInfo.latlng` — an array of `[latitude, longitude]` for the capital city.
3. Countries with missing or empty capital coordinates are filtered out to avoid wasted API calls.
4. For each remaining country, the app calls `https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lng}&current_weather=true` using the coordinates extracted in step 2.
5. Weather results are merged back with their corresponding country data by array index (both arrays are built from the same ordered list of countries).
6. Each merged record is enhanced with a computed "Travel Suitability" field derived from the weather temperature and WMO weather code.

**Example:** Searching "Japan" returns Japan's country data (name, Tokyo as capital, flag, etc.) with `capitalInfo.latlng` of `[35.68, 139.75]`. The app then fetches `https://api.open-meteo.com/v1/forecast?latitude=35.68&longitude=139.75&current_weather=true`, getting back temperature, wind speed, and a weather code. The code maps these to an emoji icon and a "Travel Suitability" classification — e.g., 22°C with clear skies produces an "Excellent" (green) badge.

Weather fetches use `Promise.allSettled()` so that if one country's weather fails to load (e.g., a capital has no coordinates or the API is temporarily unreachable), the remaining cards display normally with a "Weather unavailable" fallback.

## Known Limitations

- **Missing capital coordinates:** Some countries in the REST Countries API have `capital: []` or `capitalInfo: {}` with no `latlng`. These countries are silently excluded from weather results.
- **Partial name matching:** The REST Countries `/name` endpoint is a substring search — "land" returns England, Finland, Iceland, Thailand, etc. There is no exact-match mode, so results can be broader than expected.
- **No caching:** Each search triggers new API calls to both services. Frequently repeated searches for the same country will make redundant network requests.
- **Weather is capital-only:** The app only fetches weather for the capital city. Countries with extreme geographic diversity (e.g., Russia, Canada, USA) will not reflect conditions outside the capital.
- **Weather refresh:** Open-Meteo's `current_weather` is updated every 15 minutes, so repeated searches within that window return the same data.
- **No offline support:** The app requires an active internet connection — it fetches from both APIs on every search and has no service worker or cached fallback.
- **REST Countries undocumented rate limit:** While generous, repeated rapid searches could trigger a 429 response. The app handles this with a 10-second cooldown and user-facing warning.
- **No HTTPS enforcement:** If opened via `file://`, some browsers may block mixed content or restrict fetch for certain configurations.
