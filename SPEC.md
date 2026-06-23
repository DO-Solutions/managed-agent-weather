# SPEC.md — Airport Weather App

## What the app does

A user types an airport code (IATA, e.g. `BOS`, `JFK`, `LAX`). The app shows:

1. Current conditions at that airport.
2. The last 5 days of daily weather.
3. The next 5 days of daily forecast.

That is the entire scope. No accounts, no database, no saved searches.

## Tech stack

- **Framework:** React, built with Vite.
- **Language:** JavaScript (JSX). TypeScript is allowed but not required.
- **Styling:** Plain CSS or a single lightweight approach. Do not pull in a heavy component library.
- **Data:** Open-Meteo. No API key, no signup, CORS-enabled, so the React app can call it directly from the browser. No backend server is required.

## Data sources and exact API usage

### Airport code to coordinates

Open-Meteo takes latitude/longitude, not airport codes, so the app must resolve the IATA code to coordinates locally.

- Ship a bundled JSON file at `src/data/airports.json` mapping IATA code to `{ name, city, country, lat, lon }`.
- Generate it from the public-domain OurAirports dataset: `https://davidmegginson.github.io/ourairports-data/airports.csv`. Download it during setup, keep only rows that have a non-empty `iata_code` and a real airport `type` (large_airport, medium_airport, small_airport), and write the trimmed JSON. Do not commit the raw CSV.
- **Fallback:** if outbound network access to fetch the CSV is blocked in the sandbox, hardcode a starter list of ~40 major world airports so the app still works. Note this fallback in the README if you use it.
- Code lookup is case-insensitive. Unknown codes show a friendly "airport not found" message.

### Weather data

Single call to the Open-Meteo forecast endpoint. `past_days` returns recent history alongside the forecast, so one request covers all three sections:

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude={lat}
  &longitude={lon}
  &current=temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m
  &daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_sum
  &past_days=5
  &forecast_days=6
  &temperature_unit=fahrenheit
  &wind_speed_unit=mph
  &timezone=auto
```

- `past_days=5` plus `forecast_days=6` yields 11 daily entries: 5 past days, today, then the next 5 days. Split them by date for the three UI sections.
- `weather_code` is a WMO code. Map codes to a short text label and an emoji/icon in `src/lib/weatherCodes.js` (e.g. 0 = Clear, 61 = Rain, 71 = Snow, etc.).
- Default units Fahrenheit and mph. A C/F toggle is optional and out of scope for round one.

## Suggested project structure

```
weather-app/
  .do/app.yaml            DigitalOcean App Platform spec (static site)
  index.html
  package.json
  vite.config.js
  README.md
  SPEC.md
  src/
    main.jsx
    App.jsx
    components/
      SearchBar.jsx
      CurrentWeather.jsx
      DayCard.jsx
    data/airports.json
    lib/
      airports.js         lookup by IATA code
      api.js              Open-Meteo fetch + response parsing
      weatherCodes.js     WMO code to {label, emoji}
```

## Running locally

```
npm install
npm run dev
```

The Vite dev server should serve the app (default `http://localhost:5173`). It must build cleanly with `npm run build`, producing static output in `dist/`.

## Deploying to DigitalOcean App Platform

This is a static site (Vite build output), so it deploys as a Static Site component.

Provide a `.do/app.yaml` like this, with the repo placeholder filled in:

```yaml
name: airport-weather-app
static_sites:
  - name: web
    github:
      repo: DO-Solutions/managed-agent-weather
      branch: main
      deploy_on_push: true
    build_command: npm run build
    output_dir: dist
    routes:
      - path: /
```

A DigitalOcean API token is available in the sandbox as the env var `DIGITALOCEAN_ACCESS_TOKEN`. Deploy with `doctl`: authenticate with `doctl auth init -t "$DIGITALOCEAN_ACCESS_TOKEN"`, then `doctl apps create --spec .do/app.yaml` for the first deploy and `doctl apps update <app-id> --spec .do/app.yaml` afterwards. Retrieve the live URL with `doctl apps get <app-id>`.

### Expected one-time GitHub authorization

The `github` source with `deploy_on_push` requires the DigitalOcean GitHub integration to be authorized for this repo. That is an OAuth/GitHub-App install that only a human can approve in a browser, so the API token alone cannot set it up. This is expected, not a failure. Handle it like this:

1. Attempt `doctl apps create --spec .do/app.yaml`.
2. If it fails because the repo is not connected/authorized, pause and tell me, in plain terms, the exact step to take: go to the DigitalOcean App Platform dashboard, connect the GitHub account, and authorize access to `DO-Solutions/managed-agent-weather`. Give me the link if you have it.
3. Wait for me to confirm I have done it. Do not try to work around it.
4. Once I confirm, retry `doctl apps create --spec .do/app.yaml`, then report the live URL from `doctl apps get`.

Note in the README which path was used and that the one-time authorization was required.

## Acceptance criteria

- [ ] Repo builds with `npm install && npm run build` and no errors.
- [ ] `npm run dev` serves a working app locally.
- [ ] Entering a valid IATA code shows current conditions, 5 past days, and 5 forecast days.
- [ ] Each day shows date, high/low temp, a condition label and icon, and precipitation.
- [ ] Invalid or unknown codes show a clear, non-crashing error message.
- [ ] No API keys or secrets are required to run the app.
- [ ] App is reachable at a live DigitalOcean URL, and a push to `main` updates that live URL.
- [ ] README documents how to run locally and how the deploy works.

## Out of scope (round one)

Maps, hourly forecasts, multiple saved airports, geolocation, unit toggles, theming, tests, auth, and any backend service. Keep it minimal.
