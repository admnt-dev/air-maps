# Air Maps

A map-first app that shows air quality, chance of rain, temperature and UV on the roads themselves, and plans a trip by comparing the weather along each route. It runs as a desktop app (Electron) or in a browser. There is no backend and no API key: everything is fetched straight from free public services.

## What it does
- **Four layers:** Air quality (AQI), Rain, Temp, UV. Roads in view are coloured by the forecast for the area they sit in. A legend shows the scale with numbers.
- **48-hour time slider:** one slider drives every layer. "Now" resets it.
- **Click anywhere** for an info panel: the reading for the active layer, its category, and PM2.5/PM10 (air) or precipitation in mm (rain).
- **Search** places (Photon), or use the locate button.
- **Trip planner:** set From and To, choose walk, cycle or drive, and get routes with distance, duration and arrival time (ETA). Routes are ranked by your preferences: air quality importance, ideal temperature, and avoid or seek rain.
- **Light and dark mode**, and every panel is draggable and resizable. Layout and theme are remembered.

## Languages and tools
| Part | Choice |
|---|---|
| Language | TypeScript (strict) |
| UI | React 18 |
| Build | Vite 5 |
| Map | MapLibre GL 4 (used directly, no wrapper) |
| State | zustand 5 |
| Styling | Plain CSS with CSS variables (no UI kit, no Tailwind) |
| Icons | lucide-react |
| Desktop app | Electron 33 + electron-builder |
| Tests | Vitest |
| Package manager | pnpm |

## Data sources (all keyless)
| Service | Used for | Endpoint | Notes |
|---|---|---|---|
| OpenFreeMap | Map tiles and fonts | `tiles.openfreemap.org/planet` | OpenMapTiles schema. Custom flat style in `src/map/style.ts` |
| Open-Meteo Forecast | Temperature, rain chance, rain amount, UV | `api.open-meteo.com/v1/forecast` | Variables: `temperature_2m`, `precipitation_probability`, `precipitation`, `uv_index`. Free tier is non-commercial, CC BY 4.0 |
| Open-Meteo Air Quality | AQI, PM2.5, PM10 | `air-quality-api.open-meteo.com/v1/air-quality` | Variables: `us_aqi`, `pm2_5`, `pm10`. The US AQI scale is used throughout |
| Photon | Search suggestions | `photon.komoot.io/api/` | Debounced 350 ms, minimum 3 characters, stale requests cancelled |
| Nominatim | Place name for a clicked point | `nominatim.openstreetmap.org/reverse` | Limited to 1 request per second |
| OSRM (FOSSGIS) | Routes and alternatives | `routing.openstreetmap.de/routed-car`, `routed-bike`, `routed-foot` | Demo servers: light personal use only |
| ipwho.is | Approximate location if the OS can't give one | `ipwho.is` | Only used as a fallback after clicking locate |

The first six base URLs can be changed in `.env` (copy `.env.example`): `VITE_TILE_URL`, `VITE_GLYPHS_URL`, `VITE_WEATHER_URL`, `VITE_AQ_URL`, `VITE_PHOTON_URL`, `VITE_NOMINATIM_URL`, `VITE_ROUTING_URL`. The IP fallback address is fixed in `src/App.tsx`.

## How it works
**Layers and road colours.** For the active layer only, the visible map area is rounded outwards to a "nice" step (so small pans reuse cached data) and split into an 8×6 grid (48 cells). One multi-coordinate request fetches 48 hours of a single variable for all cell centres. The roads currently drawn are read from the map (up to 6,000 segments), and each is assigned to the cell containing its midpoint. Changing the hour only swaps a 48-colour list, so the slider is smooth and needs no new requests. Cells with no data stay uncoloured, so "no data" is never shown as zero. Colours are a viridis-style scale, and numbers appear in the legend and panels.

**Requests.** All fetches go through `fetchJson`: an in-memory cache (100 entries), with lifetimes of 12 minutes for weather and air quality, 5 minutes for routes and 1 hour for geocoding. Identical in-flight requests are shared, and stale ones are aborted. Each provider fails separately, with a retry button.

**Trip planner.** OSRM returns the fastest route plus alternatives. Five points are sampled evenly along each route, and one weather request and one air-quality request cover all of them. For each point the app uses the forecast hour you would actually be there (departure + share of the journey × duration). Departure is now, or the time slider's hour. Each route gets a cost (lower is better):
- Air quality: average AQI ÷ 150, times your importance slider (0–100%).
- Temperature: distance from your ideal temperature ÷ 15, times 0.7.
- Rain: the highest rain chance on the route. If you avoid rain, higher is worse. If you seek rain, lower is worse. Scaled by how strongly you set the slider.
- Duration: how much slower than the fastest route, times 0.8, so a long detour doesn't win on weather alone.

The lowest cost is marked "Best match". Sliders and the time slider re-rank instantly from cached data.

**Interface.** `Float.tsx` wraps each element: drag from any empty area, resize from edges and corners within set limits, positions saved in `localStorage` (`ui:<name>`). The theme (`theme`) and temperature unit (`unit`) are also saved.

**Desktop shell.** `electron/main.cjs` opens the built site (`dist/`) in a window and adds a Referer and User-Agent to Photon and Nominatim requests, which their usage policies expect.

## Run it
Needs Node.js 20+ and pnpm (`npm install -g pnpm`).

```
pnpm install
```
If pnpm says build scripts were ignored: `pnpm approve-builds`, select **electron** and **esbuild**, answer yes, then `pnpm install` again.

| Command | Does |
|---|---|
| `pnpm app` | Builds, then opens the desktop window |
| `pnpm dev` | Runs in a browser at http://localhost:5173 |
| `pnpm build` | Type-checks and builds the static site into `dist/` |
| `pnpm test` | Runs the unit tests |
| `pnpm dist` | Builds a macOS `.dmg` in `release/` |
| `pnpm exec vite build && pnpm exec electron-builder --win` | Builds a Windows installer in `release/` |

**macOS first launch:** the app is unsigned. Right-click it, choose Open, then Open again. If it says "damaged": `xattr -cr "/Applications/Air Maps.app"`.
**Windows:** if SmartScreen warns, choose More info, then Run anyway.

## Project layout
```
electron/main.cjs      desktop window
src/App.tsx            map, search, legend, scrubber, info panel
src/Trip.tsx           trip planner panel (lazy-loaded)
src/Float.tsx          drag and resize wrapper
src/layers.ts          the four layers: variable, colour scale, categories
src/providers.ts       Open-Meteo, Photon, Nominatim, OSRM
src/store.ts           map and UI state        src/tripStore.ts   trip state
src/lib/               fetchJson, grid maths, route sampling and scoring
src/map/style.ts       dark/light map style
src/*.test.ts          unit tests (grid, categories, route sampling and scoring)
```

## Limits to know about
- Values come from a coarse forecast grid, several km per cell. Road colours show the area's conditions, not each road's.
- Needs an internet connection for tiles, weather and routing.
- OSRM often returns only one route, and alternatives can be similar. Departure time usually matters more than route choice.
- No turn-by-turn directions.
- Open-Meteo's free tier is non-commercial. Commercial use needs their paid plan or a self-hosted instance. The OSRM, Nominatim and Photon public servers are for light use; swap in your own for anything heavier.
- Endpoints and variables were written from knowledge of each provider's docs, not re-checked against them live. Verify against the providers' current documentation if something fails.
- Not included: ESLint/Prettier setup, a "best time to leave" suggestion.

## Troubleshooting
- **Black window:** open developer tools (Cmd+Option+I, or Ctrl+Shift+I on Windows) and read the Console. Don't give two files names that differ only by letter case (macOS and Windows treat them as the same file).
- **Blank map:** check your connection, and that a firewall or ad blocker isn't blocking the providers above.
- **Panel lost off-screen:** run `localStorage.clear()` in the developer tools console, then reload.

## Attribution and sustainable development
Weather and air quality by [Open-Meteo](https://open-meteo.com) (CC BY 4.0). Map data © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors, tiles by [OpenFreeMap](https://openfreemap.org) and [OpenMapTiles](https://openmaptiles.org). Search by Photon and Nominatim, routing by OSRM (FOSSGIS). The app supports **SDG 11** (Sustainable Cities and Communities) by helping people plan journeys around air quality, weather and UV, and **SDG 13** (Climate Action) by making local environmental conditions easy to understand.
