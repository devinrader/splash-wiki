# Splash Frontend Service

[Back to Services Index](Home)

## Purpose

`splash-frontend` is the browser-facing React application on `splash-core`.

For the initial implementation milestone, it is responsible for the minimum
browser slice required by [Product Overview](Product-Overview) and
[Product Requirements](Product-Requirements):

- display current air temperature
- display current water temperature
- display current salt level
- display current pump RPM
- allow the operator to change pump RPM
- show a basic Home temperature-history widget sourced from `splash-api` when
  EasyTouch telemetry is available
- show concise site-level weather forecast data sourced from `splash-api`
- show persistence-backed temperature and weather history charts on `History`
  when those API surfaces are available

## Runtime role

`splash-frontend` is a read and control client of `splash-api`.

It should:

- fetch the latest equipment snapshot from `GET /equipment`
- open `GET /events` for live updates
- render the milestone-1 latest-state surface
- submit pump-speed commands through `POST /equipment/:id/control`
- surface command pending, completion, and failure state to the operator
- expose the milestone operational sidebar and view switching shell for:
  - `Home`
  - `System`
  - `Routines`
  - `History`
  - `Automation`
  - `Alerts`
  - `Diagnostics`
  - `Water Test Log`
  - `Settings`
- treat `System` as the default operational dashboard view in the current
  milestone slice
- expose `Diagnostics` as a dedicated page within that shell
- allow the Diagnostics page to host multiple diagnostics-oriented tabs while
  still consuming the shared `splash-api` protocol endpoints without bypassing
  the documented API
- allow developer-oriented diagnostic actions such as manual Remote Layout and
  raw frame sends through the documented Protocol Explorer API routes
- expose a working Automation destination with in-page tabs for `Overview`,
  `Schedules`, `Rules`, `Scenes`, `Triggers`, and `Logs` as defined in
  [Splash Frontend Automation Surface](Services-Splash-Frontend-Automation)
- show a concise weather block on `Home` with:
  - current or next-day summary
  - 10-day daily forecast strip or table
  - UV max
  - precipitation chance
  - provider freshness or stale indicator
- show a first persistence-backed `History` destination with:
  - temperature history charts for `air`, `pool_water`, `spa_water`, and
    `solar` when available
  - weather-history charts for normalized weather metrics such as forecast air
    temperature, cloud cover, UV index, and precipitation probability or
    precipitation amount

It should not:

- talk to NATS directly
- decode protocol frames
- own command correlation
- implement equipment scheduling in the milestone-1 slice
- call external weather providers directly

## Home weather widget rules

- the frontend should treat `splash-api` as the authoritative weather source
- it should render normalized data only and must not know Open-Meteo response
  field names
- if no forecast has been fetched yet, it should show an explicit empty state
- if the latest forecast is stale, it should continue rendering the cached data
  with a visible stale indicator

## Initial slice assumptions

- the initial slice may implement the milestone operational sidebar before the
  final long-term product navigation tree is complete
- the UI should still preserve the long-term visual direction described in the
  product docs: calm, clear, trustworthy, and responsive
- the initial slice may rely on the milestone-1 `splash-api` equipment bridge
  rather than the future repository-backed equipment inventory
- the first frontend diagnostics slice may keep most non-System destinations as
  placeholders while `Diagnostics` hosts the initial Protocol Explorer work
- the first frontend Protocol Explorer slice may live inside the Diagnostics
  page rather than as a standalone route
- the first Automation slice may use seeded frontend-local content to replace
  placeholder views before live automation APIs exist

## Primary references

- [Architecture](Architecture-System-Architecture)
- [REST API Contract](Interfaces-REST-API)
- [SSE Event Contract](Interfaces-SSE-Events)
- [Product Overview](Product-Overview)
- [Product Requirements](Product-Requirements)


## History destination rules

- the frontend should treat `splash-api` as the authoritative history source
- it should not query InfluxDB, Open-Meteo, or any other storage or provider
  endpoint directly from the browser
- the first `History` slice should render persistence-backed charts rather than
  a generic placeholder when the documented history APIs are available
- the first temperature-history chart set should expose separate operator-facing
  series for `air`, `pool_water`, `spa_water`, and `solar` when those series
  exist in API results
- when a specific temperature series has no captured points, the UI should show
  that series as unavailable rather than inventing zeros
- the first weather-history chart set should use normalized browser-facing data
  from `GET /weather/history`
- weather-history charts should remain provider-agnostic and must not leak
  Open-Meteo-native field names into the UI contract
- the first weather-history slice may focus on forecast-derived trend views for:
  - air temperature
  - cloud cover
  - UV index
  - precipitation probability
  - precipitation amount
- if no persistence-backed history has been captured yet, the History page
  should show explicit empty states rather than preserving a generic placeholder
