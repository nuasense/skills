---
name: nuasense
description: Build on NuaSense, East African farm and market data for software and agents. Use when a task needs live weather-station or soil-probe data with the agronomy computed from it (spray windows, disease and pest risk, irrigation need, a forecast corrected to the station, degree days), or market data for Kenya, Tanzania, Uganda and Rwanda (county fertilizer and crop prices, satellite demand, news and regulatory signals, food security, rainfall against normal). Covers getting a key, reading the catalog, calling datasets and analyses, handling units and errors, connecting a station of any brand, and the MCP server.
license: MIT
metadata:
  version: 0.1.0
  homepage: https://api.nuasense.com/index.md
---

# NuaSense

East African farm and market data behind one key and one base URL. Everything a dataset can answer is declared in a catalog, and the catalog is what you read first.

- Base URL: `https://api.nuasense.com/api/nuasense/v1`
- Analyses (ready-made answers): `https://api.nuasense.com/api/partner/v1/market`
- Docs as Markdown: `https://api.nuasense.com/index.md`; every field: `https://api.nuasense.com/llms-full.txt`
- MCP server: `https://mcp.nuasense.com/mcp` (signs in with the console account; no key to paste)

## Before the first call

1. **The key is the human's job, once.** A key is created in the console at https://platform.nuasense.com under **API Keys**. The console needs a confirmed email and a verified phone (a code by SMS or WhatsApp) before it hands one out, and shows it once. If there is no key in the environment, tell the user these four steps (sign up, confirm the email, verify the phone under API Keys, create a key) and wait for them to paste it. Never write it into the repository; read it from an environment variable such as `NUASENSE_API_KEY`.
2. **Send two headers on every call.**
   - `Authorization: Bearer <key>` (never the key in a URL)
   - `X-NuaSense-Client: skill/0.1.0; agent=<your name>` with your own name as the agent, lowercase, `[a-z0-9._-]`. It changes nothing in price or limits; it tells NuaSense the integration was built by an agent.
3. **Read the catalog for this key**: `GET /catalog`. It lists only the datasets this key reaches, with each dataset's path, filters and published fields. Build against that answer, not against a memory of the docs. `GET /` shows the key's scopes and budget, `GET /usage` the units left today.

```bash
curl https://api.nuasense.com/api/nuasense/v1/catalog \
  -H "Authorization: Bearer $NUASENSE_API_KEY" \
  -H "X-NuaSense-Client: skill/0.1.0; agent=claude-code"
```

## Reading a dataset

Every dataset is one `GET` with the same arguments:

| Parameter | Meaning |
|---|---|
| `start`, `end` | RFC3339 or relative (`-7d`, `-24h`). Default `-24h`; slow datasets carry their own default. |
| `every` + `agg` | Downsample, e.g. `every=1h&agg=mean`; `agg` in mean, min, max, sum, last, first, count. |
| `fields` | Comma-separated subset of the published fields. Naming an unpublished field answers `400`. |
| `limit` | Default 500. Ceiling 100 rows without credit, 1,000 with credit, 5,000 for the key's own farm data. Rows in total. |
| `order` | `asc` for oldest first; newest first is the default. |
| filters | Only the filters the catalog lists for that dataset. `gateway_id` picks one of the account's own stations. |

Response: `{ dataset, fields, filters, range, count, limit, rows: [{ time, ...tags, ...fields }], metering }`. `metering.unit` is `row` on market and location data and `call` on the account's own farm data.

**Units, not calls.** A row of market or location data is one unit; the account's own farm data is one unit per call; an analysis is one unit, priced separately. Without credit: 200 units a day, 1,000 a month, 100 rows per call, the last 30 days of market data. With credit or a station: 1,500 a day, 1,000 rows per call, full history. The response headers `X-Daily-Remaining`, `X-Monthly-Remaining` and `X-RateLimit-Remaining` say how much is left; start with a small `limit` and read them before looping.

### Which path for which question

| Need | Path | Notes |
|---|---|---|
| Live station readings | `/farm/weather` | about every 10 min; `/farm/weather/hourly`, `/farm/weather/daily` for roll-ups |
| Soil probes | `/farm/soil` and its roll-ups | `dev_eui` filter per node |
| Spray quality, ET0, VPD, delta T, leaf wetness, degree days | `/farm/derived` | hourly |
| Disease and pest risk | `/farm/disease-risk` | `threat` filter; risk %, level, conducive hours |
| Forecast for the station's position | `/farm/forecast` | 7 days hourly, refreshed every 6 h |
| Root-zone water balance | `/farm/water-balance` | daily |
| Satellite indices over the fields | `/farm/field-indices` | weekly; `field_id` filter |
| Climate archive for the position | `/farm/climate` | daily |
| Soil profile | `/farm/soil-baseline` | physical layers only; no pH or N/P/K on purpose |
| County fertilizer and crop prices | `/market/prices` | `commodity=fertilizer&variety=dap`; county slugs lowercase (`nakuru`, `uasin_gishu`) |
| Agro-dealer shelf prices | `/market/input-prices` | `country`, `category`, `product` |
| News and regulatory signals | `/market/news`, `/market/regulatory` | each row has a source URL and a verbatim quote |
| Satellite demand per county | `/market/demand-indices` | `country=ke|tz|ug|rw` |
| Food security, rainfall, production, FAO statistics | `/market/food-security`, `/market/rainfall-anomaly`, `/market/production`, `/market/fao-*` | continent-wide: a `country` (ISO3: `ken`, `tza`, `uga`, `rwa`) or another listed filter is required |

Fertilizer varieties: `dap`, `urea`, `npk`, `can`, `mop`, `tsp`, `sulphate_of_ammonia`, `ammonia_anhydrous`. Annual statistics are stamped 1 January of their year; query them with a multi-year window. Two datasets are declared but withheld on licence grounds and answer `451` to every key (`/market/trade`, `/market/field-structure`); do not build on them.

### Ready-made analyses

One call, one measured answer, under `https://api.nuasense.com/api/partner/v1/market`. Call `/meta` first for the valid names. Examples: `/fertilizer/price?variety=dap&county=nakuru`, `/fertilizer/trend?variety=dap`, `/fertilizer/county-brief?county=nakuru&country=ke`, `/analytics/spread?commodity=maize_dry&transport_cost_kes_per_kg=8`, `/analytics/season?county=nakuru`, `/analytics/yield-risk?county=uasin_gishu&product=maize`. The full list is in `https://api.nuasense.com/index.md`.

## Errors and what to do

| Status | Meaning | Do |
|---|---|---|
| `400` | unpublished field, bad time or `agg`, or a continent-wide dataset without a filter | fix the request; the body names the problem |
| `401` | no, bad, revoked or expired key | ask the user for a key |
| `402` | history beyond the free 30 days, credit spent, or a write on an account that never topped up | tell the user what needs credit; do not retry |
| `403` | scope missing, station not approved for API exposure, or a `gateway_id` the key does not own | a different key or request; retrying changes nothing |
| `404` | dataset not published to this key; the body names it | an answer, not a routing error |
| `429` | per-minute, daily or monthly budget; the body says which and how many units are left | wait `Retry-After`; ask with a smaller `limit` |
| `451` | withheld on licence grounds | nothing unlocks it |

## Honesty rules for what you show the user

- Never invent a price, a reading or a number the API did not return.
- Lead with rows marked `reliable=true` or medium or high trust where a dataset carries such a flag; say plainly when data is thin or stale.
- News rows: cite the source URL; `direction=unknown` is deliberate, a watch item rather than a fact.
- A farm dataset that answers with no rows means the account has no station or the station is not approved; say so, never estimate a reading from market data.
- The soil profile has no pH or N/P/K; current fertility comes only from a soil test.

## Connecting a station

Any station can feed the farm layer; the first three on an account connect free. The user adds it in the console under **External stations** (https://platform.nuasense.com/stations) and picks how it sends:

- A console with a custom-server screen (Ecowitt, Froggit, Ambient, Wunderground mode): server `ingest.nuasense.com`, path `/station/<token>`, port `80`, interval `60`.
- LoRaWAN: a webhook from The Things Stack or ChirpStack to `POST https://api.nuasense.com/lorawan/<token>`.
- Your own code: `POST /ingest/readings` with `{ station_id, readings: [{ time, temperature, humidity, pressure, light, wind_speed, wind_gusts, wind_direction, rain_mm }] }`, at most 500 readings per call, none older than 90 days. Needs the `ingest:write` scope and a funded account.
- Home Assistant, Node-RED, ioBroker and weewx packages: https://github.com/nuasense.

What comes back for an external station: `GET /ingest/first-look?station_id=` (tonight's low, the next calm dry daylight window, a plausibility check of the latest reading, the garden's watering need) and `GET /ingest/derived?station_id=` (ET0, VPD, delta T, leaf wetness, spray quality, degree days).

## The data

- When a station is connected, NuaSense stores its readings and computes the forecast, spray windows and risks from them.
- The readings are used aggregated and anonymised to improve forecasts and models for everyone; they are never passed on individually without the owner's opt-in, and positions on public maps are blurred.
- The owner can disconnect the station at any time and have all its data deleted.

A read buys use, not ownership: the data may be used inside the user's own product, analysis and reporting, not resold or republished as data. `GET /attributions` lists the credits that travel with some rows; it needs no key.

## MCP instead of REST

For a chat assistant rather than code, add `https://mcp.nuasense.com/mcp` as a connector (Claude.ai: Settings → Connectors; Claude Code: `claude mcp add --transport http nuasense https://mcp.nuasense.com/mcp`). It signs in with the console account and exposes the same data as tools with read-only and destructive hints.

Help: info@nuasense.com, a person reads it.
