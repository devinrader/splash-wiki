# Splash Frontend Runtime

[Back to Splash Frontend README](Services-Splash-API-Home)

## Runtime model

`splash-frontend` runs on `splash-core` as a browser-delivered web app built
with React, TypeScript, and Vite.

The initial milestone runtime is intentionally narrow:

1. fetch the current latest-state snapshot from `splash-api`
2. render the current controller, pump, and chlorinator readouts needed for the
   first milestone
3. open SSE for live updates
4. submit pump-speed control requests through the REST API
5. reflect command lifecycle state in the browser
6. render a left-sidebar operational shell whose active item controls the main
   content surface
7. treat `System` as the default view for the current milestone
8. expose a Diagnostics page with controlled tabs for:
   - `Protocol Explorer`
   - `Live Data Monitor`
   - `Device Inspector`
   - `Logs & History`
   - `Network`
9. expose the first Diagnostics `Protocol Explorer` tab that:
   - streams `GET /protocol/frames`
   - creates saved bundles
   - compares saved bundles
   - reads and creates annotations and prompts
   - requests one controller circuit-configuration record by circuit index
   - waits for only the matching decoded `circuit_configuration` reply instead
     of requiring the operator to inspect the full live frame stream manually
   - shows the matched `function_id`, `base_function_id`,
     `base_function_label`, `name_id`, `name_label`, `freeze_flag`, and
     `high_flag` in a compact result panel
   - submits manual Remote Layout page requests
   - submits manual raw frame sends
   - shows outbound diagnostic request events such as `protocol.command.encoded`
     and `serial.tx.raw` when present
10. allow the remaining milestone Diagnostics tabs and most non-System sidebar
    destinations to remain placeholder surfaces until their deeper product
    workflows are implemented
11. expose the first real `Settings` destination for:
    - weather-location configuration backed by `splash-api`
    - pool-chemistry bounds configuration backed by `splash-api`

## Data flow

1. browser loads `splash-frontend`
2. frontend calls `GET /equipment`
3. frontend hydrates local state from the REST response
4. frontend opens `EventSource` to `GET /events`
5. frontend merges incoming `equipment.state`, `pump.state`, and
   `command.result` events into client state
6. operator submits a pump-speed change
7. frontend sends `POST /equipment/:id/control`
8. frontend shows pending state until `command.result` resolves the action

Protocol Explorer flow inside Diagnostics:

1. browser opens `GET /protocol/frames`
2. frontend shows recent raw, decoded, and outbound diagnostic protocol events
3. operator saves one or more frame bundles through `POST /protocol/bundles`
4. frontend compares saved bundles through `POST /protocol/bundles/compare`
5. frontend reads or creates annotations and prompts through
   `/protocol/annotations` and `/protocol/prompts`
6. frontend may send a one-shot controller circuit-configuration request for a
   selected circuit index and then wait for the matching decoded reply
7. frontend may send manual Remote Layout or raw frame diagnostic requests

Cross-origin local development rule:

- when `splash-frontend` is served from a local Vite dev origin such as
  `http://127.0.0.1:3000` and `splash-api` is served from
  `http://127.0.0.1:8080`, the API must emit CORS headers so the browser can
  use both REST and SSE successfully

## Event handling rules

- initial state comes from REST, not SSE replay
- SSE is authoritative for live in-session updates
- on SSE disconnect or reconnect, the frontend should refetch `GET /equipment`
  to resynchronize with the latest server state
- `ready` SSE events are connection-level only and do not update equipment UI

## UI expectations

- render a responsive milestone-1 dashboard for desktop and mobile
- use the uploaded Splash design-system tokens as the styling baseline for the
  current milestone dashboard shell, including the documented light surfaces,
  dark navigation shell, semantic status colors, tokenized spacing, and card
  radii rather than ad hoc one-off values
- use the uploaded Splash expanded icon library as the default icon source for
  dashboard status, metric, equipment, navigation, and diagnostics affordances
  when a matching icon exists
- preserve the existing left sidebar layout and styling while allowing one
  active destination at a time
- use the active navigation destination to determine the page title and main
  content without reloading the app
- keep the left sidebar at a fixed `240px` width on the current desktop and
  tablet-only milestone shell
- place the entire application shell inside a centered desktop/tablet frame
  rather than letting either the sidebar or content stretch edge to edge on
  large screens
- on desktop widths starting at `1280px`, constrain the full application shell
  to a `max-width` of approximately `1280px`, inclusive of the fixed sidebar
  and main content area
- on larger desktop widths such as `1600px+`, keep that full application frame
  centered and do not expand past the desktop width cap
- on tablet widths from `1024px` through `1279px`, allow the application shell
  to use the available width while keeping horizontal padding and avoiding
  horizontal overflow
- use approximately `32px` horizontal padding for desktop and `24px`
  horizontal padding for tablet around the centered application shell
- follow the proportions shown in the current frontend mockup images:
  fixed-width dark sidebar, lighter content field, and a centered readable
  application frame
- on larger screens, keep the sensor and control surfaces in a narrower
  left-side dashboard rail, show a middle dashboard column for the default
  EasyTouch circuit inventory (`POOL`, `SPA`, `AUX 1-7`, `FEATURE 1-8`, and
  `AUX EXTRA`), and allow Protocol Explorer and frame-decoding workflows to
  occupy a broader section on the right
- within the `System` page only, render a secondary tab bar directly below the
  page header/title area for:
  - `Overview`
  - `Equipment`
  - `Sensors`
  - `Water Features`
  - `Connectivity`
- those `System` tabs should switch the main content area without changing the
  application shell, left navigation, top header layout, or page route
- the `System` tab content may remain placeholder-structured in this slice, but
  each tab should present a distinct layout aligned with the current mockup
  intent rather than duplicating the same content block across all tabs
- internal page grids should remain flexible and reflow cleanly within the
  constrained content container instead of relying on hard-coded fixed widths
- on the EasyTouch hardware detail surface, the `Circuit Configuration` table
  should render the columns:
  - `ID`
  - `Type`
  - `Function`
  - `Function Value`
  - `Name`
  - `Name Value`
  - `Freeze`
  - `State`
  - `Action`
- that table should be driven by a merge of:
  - API-provided controller hardware inventory for the circuit id, category,
    installation status, and writable capability
  - live controller circuit-configuration metadata for the configured name,
    function, and freeze state
  - live controller status for the current enabled or disabled state
- the `Function` column may render as a staged select box seeded from the known
  Pentair circuit-function list, with the current discovered circuit function
  selected
- the `Function Value` column should render the raw controller `function_id`
  numeric value when available
- the `Name` column may render as a staged select box seeded from the known
  Pentair assigned-name list, with the current discovered circuit name selected
- the `Name Value` column should render the raw controller `name_id` numeric
  value when available
- the `Freeze` column may render as a staged toggle seeded from the current
  discovered freeze flag
- the `State` column may render as a staged toggle only for circuits that have
  a known writable controller mapping and an authoritative boolean state;
  unsupported or unavailable state rows must remain non-interactive
- the `Action` column may render placeholder save and discard icon buttons in
  this slice; those buttons do not need to persist row edits yet
- the `Circuit Configuration` card should expose a `Refresh circuit
  configuration` action that triggers a fresh controller circuit-configuration
  discovery request using the existing diagnostic request path
- while that refresh is in flight, the card-level refresh action should disable
  itself and show pending status inline
- when a circuit is marked not installed by the configured hardware inventory,
  the table should render the staged `Name`, `Function`, `Freeze`, and `Action`
  cells as unavailable rather than interactive, and the `State` column should
  show `Not installed`
- the `Custom Circuit Names` table may also render an `Action` column with the
  same placeholder save and discard icon buttons per row; those buttons do not
  need to persist custom-name edits yet in this slice
- when the API exposes both `configuration_circuit_index` and
  `write_circuit_id`, the dashboard should treat:
  - `configuration_circuit_index` as the diagnostic/config-discovery reference
  - `write_circuit_id` as the control selector reference
  - and should not assume those values are identical
- when hardware-description data is available, render controller circuits from
  the API-provided configured hardware inventory rather than a hard-coded
  EasyTouch circuit list
- use the configured controller hardware profile to reflect model limits, such
  as EasyTouch 4 versus EasyTouch 8 circuit inventory
- use the user-confirmed installed hardware configuration to distinguish
  physically absent relays from installed circuits whose live state is currently
  unavailable
- when a circuit state is not currently mapped from controller status data, the
  dashboard should show that state as unavailable rather than implying `Off`
- when a configured relay or optional circuit is not installed, the dashboard
  should show `Not installed` or hide it in normal operation rather than showing
  `Off`
- the circuit state pill may act as the dashboard toggle surface only for
  controller circuits that project a known writable controller mapping
- if a circuit does not have a known writable controller mapping, the pill must
  remain non-interactive even when the dashboard can display its latest
  observed state
- pills for `Unavailable`, `Not installed`, or unsupported circuit states must
  remain non-interactive
- mapped controller-status circuit bitmasks are authoritative when present;
  `POOL` and `SPA` should use the normalized controller `mode` only as a
  fallback when raw bitmask-derived circuit state is unavailable
- provide a diagnostic action to request controller-reported circuit
  configuration using the API discovery path; this action should not directly
  mutate dashboard circuit inventory until hardware-description persistence is
  implemented
- when the dashboard session starts, determine whether installed controller
  circuits with known `configuration_circuit_index` values already have
  controller-acquired function and name metadata in the API snapshot
- if that metadata is missing, the dashboard may automatically trigger one
  controller circuit-configuration discovery request for the configured index
  range
- once the API snapshot includes acquired controller function and name metadata,
  the dashboard should reuse that persisted snapshot data on reload rather than
  automatically triggering discovery again
- a manual circuit-config sweep remains authoritative as the explicit refresh
  mechanism and may replace previously acquired metadata
- provide controller date/time actions in the dashboard:
  - a diagnostic action to request controller date/time information
  - a best-effort action to sync controller date/time from Splash system time
- label both controller date/time actions as provisional until the EasyTouch
  protocol family and payload layout are confirmed by live captures
- when the dashboard session starts and no `0x05` controller date/time reply is
  currently present in the API snapshot, the frontend may issue one automatic
  diagnostic controller date/time request so the separate `0x05` metric card
  has a chance to populate without a manual button press
- show the latest known air temperature, water temperature, heater state,
  salt level, controller system time, and pump RPM
- on the `Home` destination, show a `Weather Impact` summary card when
  `splash-api` exposes a normalized site forecast
- replace the current Home `Weather Forecast` and `Weather Detail` cards with a
  single `Weather Impact` card
- keep the Home weather slice focused on operator-facing pool-impact context
  rather than a generic long-form forecast table
- the first `Weather Impact` card should show:
  - one larger `today` tile with:
    - current or primary daytime weather symbol
    - today temperature headline
    - short sky-condition label
    - short pool-impact label such as chlorine-demand guidance
  - three compact upcoming-day tiles with:
    - day label
    - high / low range
    - weather symbol
    - short impact or UV-intensity label
- the card may derive its first-slice pool-impact labels heuristically from
  existing normalized weather forecast data
- the first-slice impact language should remain simple and explainable, for
  example:
  - `Low chlorine demand`
  - `Elevated chlorine demand`
  - `Rain may dilute chemistry`
  - `Stable conditions`
- keep provider and stale metadata secondary:
  - show it in subdued supporting text on the card rather than as a primary row
- on the `Home` destination, keep the first slice focused on operator-facing
  summary cards rather than duplicating the richer telemetry and history views
  already owned by the `History` destination
- on the `Home` destination, add a `Pool Cover` card that:
  - loads current cover state from `GET /pool/cover`
  - shows current state, cover type, and last updated timestamp when a cover
    event exists
  - shows an explicit empty state when no cover event has been recorded
  - allows the operator to record:
    - `Cover On`
    - `Cover Off`
  - requires cover-type selection before submitting `Cover On`
  - uses first-slice cover-type choices of:
    - `Solar`
    - `Winter`
    - `Safety`
    - `Automatic`
    - `Unknown`
  - disables actions while a save is pending
  - refreshes current state and recent history after save success
  - surfaces load and save errors inline on the card
  - may show a compact recent-events list on the same card using
    `GET /pool/cover/history`
  - keeps cover-event overlays out of the first slice; those belong to later
    `History` page work
- on the `Home` destination, add a read-only `Swimmability` card that:
  - loads `GET /swimmability`
  - shows:
    - a score-ring style score treatment
    - overall status
    - short operator-facing headline
    - short summary
    - `Last Chemistry`
    - `Confidence`
    - curated highlight badges
  - uses first-slice visual states of:
    - `Good`
    - `Caution`
    - `Poor`
    - `Unknown`
  - surfaces an explicit unknown state when the API cannot make a confident
    assessment
  - keeps raw driver details out of the primary card body when curated
    highlights are available
  - omits a `View details` link until a real swimmability detail destination is
    defined
  - remains read-only in the first slice
  - keeps detailed chemistry entry on `Water Test Log`
  - keeps telemetry and chart context on `History`
  - treats chemistry as the primary safety signal
  - treats chemistry age and rainfall since the last test as confidence
    modifiers
  - treats cover, UV, hot weather, and warmer water as modifiers that can make
    chemistry confidence age faster when the pool is uncovered
- on the `Settings` destination, render a `Pool Chemistry` section that:
  - loads the configured chemistry bounds set from `GET /api/settings/pool-chemistry`
  - lets the operator edit supported built-in chemistry keys
  - preserves units, enabled flags, and min/target/max inputs
  - saves changes through `PUT /api/settings/pool-chemistry`
  - surfaces validation and save status using the same page-level patterns as
    the weather-location settings section
- expose a real `Water Test Log` destination at `/water-test-log` that:
  - loads chemistry history from `GET /chemistry/history`
  - lets the operator submit manual readings through `POST /chemistry`
  - accepts first-slice manual-entry inputs for `pH`, `Free Chlorine`,
    `Total Chlorine`, `Total Alkalinity`, `Calcium Hardness`, and
    `Cyanuric Acid`
  - does not expose editable `Recorded At` in the first slice; the API
    assigns save time as the reading timestamp
  - does not expose manual `Salt` or `Rainfall` inputs because those are
    tracked from chlorinator telemetry and weather history
  - disables submit while a save is pending
  - refreshes chemistry history data after save success
  - shows a non-blocking warning when both `pH` and `Free Chlorine` are
    omitted from a saved reading
  - surfaces load and save errors inline inside the existing page status area
    rather than through separate error summary cards
  - focuses on manual chemistry entry and prior-log review
  - shows a prior-log table with sparse values rendered as `—`
  - supports time-range presets of `Last 7 days`, `Last 30 days`, and
    `Last 90 days`
  - keeps first-slice chemistry logging separate from later SLAM, cover, or
    swimmability overlays
- on the `Alerts` destination, replace the placeholder with a real notification inbox that:
  - loads `GET /notifications`
  - defaults to unread notifications
  - renders a chronological inbox list with:
    - severity label
    - title
    - body
    - created time
    - read/unread state
  - allows:
    - mark one notification as read
    - mark all notifications as read
  - supports first-slice filtering by:
    - unread vs all
    - notification type
  - shows explicit empty states for:
    - no unread alerts
    - no alerts at all
  - surfaces load and save errors inline on the page
  - remains notification-only in the first slice and does not yet expose full task workflow controls
- when no weather forecast has been fetched yet, the weather widget should show
  an explicit empty state rather than inventing weather values
- when the cached weather forecast is stale, the widget should continue showing
  the last known valid forecast together with a stale label or timestamp
- metric cards and status indicators should pair their text labels with icons
  from the Splash icon library when a suitable icon exists
- status indicators should present icon, text, and semantic color together and
  must not rely on color alone
- format controller system time from the controller-status hour and minute
  fields rather than from browser-local wall clock time
- when a provisional EasyTouch `0x05` controller date/time reply has been
  observed, show it as a separate diagnostic metric card rather than replacing
  the primary controller-status system-time card
- show when latest values are unavailable rather than inventing defaults
- disable pump-speed submission while a prior control request is unresolved
- show degraded API state when `/health` reports degraded or when SSE is not
  connected

## System page tab rules

- `Overview` should remain a high-level summary surface rather than a detailed
  inventory table
- `Equipment` should focus on equipment list/detail placeholders, search, and
  filters
- `Sensors` should focus on sensor readings and health placeholders
- `Water Features` should focus on named feature inventory and state
  placeholders
- `Connectivity` should focus on network/controller/event-bus/RS485 summary
  placeholders without duplicating the deeper Diagnostics page
- `Platform` should act as the operator-facing health surface for Splash
  software and infrastructure services rather than a generic placeholder-only
  runtime card stack
- the `Connectivity` tab may render live summary metric cards sourced from
  `splash-api` health data when those values are available
- the first live `Connectivity` metric card set should focus on RS485 metrics
  sourced from `splash-api` health data
- the first live `Connectivity` metric card set should include:
  - `RS485 Messages In`
  - `RS485 Messages Out`
- the first live `Connectivity` metric card set may later include:
  - `NATS Subscriptions`
  - `NATS In Messages`
  - `NATS Out Messages`
- when the API exposes transport or broker rates as per-second values, the
  dashboard may convert them into per-minute display values for the
  Connectivity summary cards
- when a requested Connectivity metric is unavailable, the dashboard should
  show `Unavailable` rather than preserving stale placeholders
- the `Connectivity` tab may also render a live line chart for transport
  activity
- the first Connectivity chart may use a React-native charting library such as
  Nivo and should stay visually consistent with the existing content cards
- the first Connectivity chart should plot two series:
  - `RS485 In`
  - `RS485 Out`
- the first Connectivity chart may later add:
  - `NATS In`
  - `NATS Out`
- the frontend may sample `splash-api` health data every `10s` during the
  active browser session and retain a bounded in-memory history for the chart
- for RS485 traffic specifically, the frontend should expect `splash-api`
  health rates to already represent rolling `10s` average messages-per-second
  values rather than instantaneous `1s` spikes
- when the API exposes per-second rate values, the frontend may derive the
  charted `10s` bucket counts by multiplying the latest per-second rate by `10`
- when a series is unavailable, the chart should omit that point or show an
  explicit unavailable state rather than inventing values
- the frontend should not query Grafana directly for these cards or charts;
  `splash-api` remains the browser-facing source for the first connectivity
  slice
- the `Platform` tab may render a first live service-health summary sourced
  from `splash-api` health data
- the first `Platform` service-health view may include:
  - `splash-serial`
  - `NATS`
  - `splash-protocol`
  - `splash-frontend`
  - `splash-api`
  - `prometheus`
  - `grafana`
  - `influxdb` when telemetry persistence is configured
  - `weather-provider` when weather forecast integration is configured
- each service row or card may show:
  - service name
  - current status
  - a short summary
  - optional last-updated or detail text
- when a service status is unavailable, the dashboard should show
  `Unavailable` rather than preserving stale placeholder text
- the Platform view should use `GET /platform/status` rather than probing
  platform services directly from the browser
- the Platform view should show:
  - overall platform status
  - per-service status cards or rows
  - last-checked time
  - primary message or reason
  - expandable per-check details
  - stale-data warning when cached data is being shown
- the frontend should cache the most recent successful
  `GET /platform/status` response in browser storage
- if `splash-api` is healthy, the dashboard should show live platform status
- if `splash-api` is degraded, the dashboard should show a degraded banner and
  render the current service details
- if `splash-api` is unhealthy, the dashboard should warn that live platform
  status may be incomplete and render last-known cached status when available
- if `splash-api` is down, the dashboard should show `Splash API unavailable`,
  retry automatically, and render last-known cached status with an explicit
  stale timestamp when available
- the frontend should not poll Grafana, Prometheus, NATS, `splash-serial`, or
  `splash-protocol` directly for browser-visible health


## History page rules

- the `History` destination may transition from placeholder content to a first
  persistence-backed trend surface once `splash-api` exposes the documented
  history routes
- the first `History` slice should not duplicate the concise Home widget; it
  should focus on longer-range trend visibility and operator review
- when the `History` destination contains multiple chart families, it should
  group them under internal page tabs rather than rendering every chart in one
  long scroll by default
- the first `History` tab set should include:
  - `Temperature`
  - `Pump`
  - `Weather`
  - `Chemistry`
- the default `History` tab should be `Temperature`
- the `Temperature` tab should render the first temperature-history slice
- the `Pump` tab should render the first pump-history slice
- the `Weather` tab should render the first weather-history slice
- the `Chemistry` tab should render the first chemistry-history slice sourced
  from `GET /chemistry/history`
- the first `Chemistry` tab should chart:
  - `pH`
  - `Free Chlorine`
  - `Total Chlorine`
- the first richer-context chemistry-history slice should also load
  `GET /pool/cover/history` for the same active History time range
- the first cover-overlay slice should render read-only vertical event markers
  on the chemistry charts for:
  - `Cover On`
  - `Cover Off`
- the first cover-overlay slice should:
  - include compact marker labels derived from `cover_type`
  - include a small legend explaining cover-on and cover-off markers
  - remain read-only and not duplicate cover logging from `Home`
- defer temperature-tab cover overlays, interval shading, and richer combined
  chemistry-plus-weather-plus-cover annotations to later `#117` work
- the frontend should lazy-load History tab datasets on first activation rather
  than fetching all History families on initial page mount
- once a History tab dataset has been loaded successfully, the frontend may
  retain it in in-memory page state for fast tab switching during the active
  session
- the History page should expose a time-range selector that applies to
  Temperature, Pump, Weather, and Chemistry tabs
- the first selector slice should support:
  - `Last hour` with `5m` history interval
  - `Last 6 hours` with `15m` history interval
  - `Last 12 hours` with `15m` history interval
  - `Last 24 hours` with `15m` history interval
  - `Last 36 hours` with `15m` history interval
  - `Last 3 days` with `1h` history interval
  - `Last 7 days` with `4h` history interval
- the default selected range should be `Last 36 hours`
- when the selected range changes, the frontend should request the newly
  selected range for the active History tab and use the matching interval for
  documented history API calls
- inactive History tabs should not mount their chart grid until the tab is
  activated
- the first temperature-history slice should render charted series for:
  - `air`
  - `pool_water`
  - `spa_water`
  - `solar`
- the frontend may request those series through repeated
  `GET /telemetry/temperatures/history` calls or one future expanded history
  response, but it must stay within the documented API contract
- the first pump-history slice should render charted series from
  `GET /telemetry/pumps/history`
- the first pump-history chart set should support at least:
  - pump RPM
  - pump watts
- the first slice may render one card per metric for the default API-facing
  pump id such as `pump-main`
- if the API reports no persisted pump telemetry yet, the History page should
  render an explicit unavailable or empty state rather than a placeholder
- the first weather-history slice should render charted normalized weather
  series from `GET /weather/history`
- the first weather-history chart set should support at least:
  - air temperature
  - cloud cover
  - UV index
  - precipitation probability
  - precipitation amount
- if a requested series is unavailable, the History page should render an
  explicit unavailable or empty state rather than preserving placeholder text
- the History page should label stale weather data when the API reports that
  the most recent cached forecast snapshot is stale

## Settings page rules

- the `Settings` destination may transition from placeholder content to the
  first real operator-managed configuration surface once `splash-api` exposes
  the documented weather-location settings routes
- the first `Settings` slice should render a `Weather Location` section
- the section should allow the operator to choose exactly one location mode:
  - `Use physical address`
  - `Use latitude/longitude`
- when address mode is active, the form should collect:
  - `address line 1`
  - `address line 2` optional
  - `city`
  - `state/region`
  - `postal code`
  - `country`
- when coordinate mode is active, the form should collect:
  - `latitude`
  - `longitude`
  - `timezone` optional
- the frontend should load existing saved settings on page entry
- the frontend should save through `PUT /api/settings/weather-location`
- the frontend should render field validation errors returned by the API
- the frontend should render a clear save success or failure state
- the frontend should not require both address fields and coordinate fields in
  the same submission
- the first slice may stay focused on weather-location settings rather than
  opening the full broader Settings taxonomy

## Command UX rules

- the pump-speed control should target the API-facing pump id exposed by
  `GET /equipment`
- the operator enters RPM as an integer
- the UI should preserve the last requested RPM while the command is pending
- `command.result` should clear pending state and show success or failure
- if no matching completion arrives before the API reports timeout or failure,
  the UI should show the latest command state without guessing success
- when an operator clicks a writable controller-circuit pill, the dashboard
  should send a `set_circuit_state` request for the target circuit and desired
  on/off state
- the dashboard must not flip the pill optimistically after click; it should
  continue showing the pre-click state until the next authoritative controller
  status update confirms the change
- while a circuit toggle is unresolved, the dashboard may disable the pill or
  show pending state, but it must not imply success before controller state
  confirms the new value

## Diagnostics UX rules

- the first frontend Diagnostics page may be secondary to the milestone
  `System` dashboard rather than the primary default destination
- the Diagnostics page should provide a horizontal tab bar under the page title
  and switch tab content without full page reload
- the first four milestone non-Protocol-Explorer destinations outside
  `System` may render placeholders as long as the sidebar and active-state
  behavior are implemented consistently
- the Diagnostics `Network` tab should render exactly five reusable cards:
  - `Network Overview`
  - `Network Statistics`
  - `RS485 Bus`
  - `Event Bus`
  - `Network Interfaces`
- on desktop, the `Network` tab should use a responsive grid where:
  - the top row contains `Network Overview` and `Network Statistics`
  - those two top-row cards split the available row width evenly
  - the bottom row contains `RS485 Bus`, `Event Bus`, and
    `Network Interfaces`
  - `RS485 Bus` and `Event Bus` together should occupy the same total width as
    the top-row `Network Overview` card
  - `Network Interfaces` should occupy the same width as the top-row
    `Network Statistics` card
- the `Network` tab may remain placeholder-oriented overall, but the System
  `Connectivity` tab may render live metric cards and the first sampled
  message-activity chart sourced from `splash-api` health data

## Explorer UX rules

- the Explorer should distinguish:
  - live protocol events
  - saved bundles
  - bundle diffs
  - annotations
  - operator-needed prompts
- the live frame list should stay inside a bounded scroll region so ongoing
  frame traffic does not continuously increase overall page height
- the live frame panel may provide client-side action-code filters so the
  operator can either:
  - show only selected action codes
  - hide selected action codes
- action-code filters should use an explicit multi-select list style UI rather
  than a free-text parser
- each selectable action should show:
  - a friendly protocol label when known
  - the decimal action value
  - the hex action code
- action-code filtering should apply only to what is rendered in the live frame
  list; it must not change the underlying `/protocol/frames` SSE subscription
- when a decoded frame exposes `action_code`, the filter should match against
  that field
- events without an `action_code` may remain visible unless later event-type
  filtering is added separately
- manual Remote Layout and raw-frame send actions should remain clearly labeled

## EasyTouch8 Hardware Page Controller Clock Surface

The EasyTouch8 hardware page should expose controller clock information in two
separate surfaces:

1. `Main Controller` summary
2. `Controller Clock Configuration` card

### Main Controller summary

Read-only fields:
- controller date
- controller time
- DST mode
- clock advance

Rules:
- this box is read-only
- it should show the latest controller-derived values when available
- it should not expose write buttons
- if values are only available from provisional `0x05` date/time reply data,
  the UI must label them as provisional

### Controller Clock Configuration card

Editable fields:
- date
- time
- DST mode
- clock advance

Actions:
- `Refresh controller clock`
- `Save controller clock configuration`

Rules:
- `clock advance` maps directly to the EasyTouch `Clock Advance` setting
- save behavior remains provisional until the EasyTouch clock-setting payload is
  fully live-validated
- the card must show pending, success, and failure states independently from the
  read-only summary
- after save, the page must refresh from controller-derived state rather than
  assuming the submitted values are authoritative

## EasyTouch8 Hardware Page Pump Configuration Card

The EasyTouch8 hardware page should expose a `Pump Configuration` card.

Rendering rules:
- render configuration sections only for pumps the live controller reports as
  installed
- do not render placeholder pump editors for pumps that are not present in live
  controller state
- the number and shape of pump sections should follow the EasyTouch controller
  configuration description/menu structure

Supported first-slice branches:
- Pump #1 when installed
- Pump #2 when installed
- `Pump Type`
- all documented configuration branches for the active pump type, including:
  - `VS`
  - `VSF`
  - `VF`

Interaction rules:
- load from live controller-acquired pump configuration, not from static local
  defaults
- writes must use read-modify-write from a fresh controller baseline
- save should remain per-pump, not global across all installed pumps
- each pump section must show independent pending, success, and failure state
- the page must refresh from controller-derived state after save confirmation

## EasyTouch8 Hardware Page Heater Panel

The EasyTouch8 hardware page should expose an EasyTouch-owned heater panel.

Panel sections:
- `Configuration`
- `Heat Settings`

Visible read-only status:
- detected heater type
- solar or heat-pump enabled
- heating enabled
- cooling enabled
- freeze protection enabled
- current pool heat mode
- current spa heat mode
- current pool setpoint when available
- current spa setpoint when available
- current cool setpoint when available

Editable configuration fields:
- heater type
- cooling enabled
- freeze protection enabled

Editable heat-setting fields:
- pool setpoint
- spa setpoint
- pool heat mode
- spa heat mode
- cool setpoint

Interaction rules:
- the page must expose separate `Save configuration` and `Save heat settings`
  actions
- each action must show independent pending, success, and failure state
- after a successful save, the page should reload from refreshed controller
  data
- unsupported direct UltraTemp controls must remain hidden or disabled with
  explanatory copy
- the UI must state clearly that EasyTouch remains the authoritative heater
  controller
  as diagnostic-only
- raw-frame send should preserve the operator-provided lowercase hex exactly and
  should not imply protocol-level success when the transport write succeeds
- the first slice may stay intentionally developer-oriented and does not need
  the full long-term product navigation treatment yet
- even within the developer-oriented milestone slice, diagnostics should remain
  visually subordinate to the primary operational dashboard and should reuse
  the same tokenized card and status language
