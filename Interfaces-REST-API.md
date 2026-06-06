# REST API

[Back to README](Home)

## API principles

- Base URL on LAN: `http://splash-core.local:8080`
- All request and response bodies are JSON
- Timestamps are ISO 8601 UTC
- All responses use an envelope shape

```json
{
  "data": {},
  "error": null
}
```

## REST resources

| Resource | Methods | Purpose |
| --- | --- | --- |
| `/pool` | `GET`, `PUT` | Pool profile |
| `/equipment` | `GET`, `POST` | Equipment inventory |
| `/equipment/:id` | `GET`, `PUT`, `DELETE` | Equipment record |
| `/equipment/:id/control` | `POST` | Equipment control command |
| `/chemistry/latest` | `GET` | Latest chemistry reading |
| `/chemistry/history` | `GET` | Chemistry history |
| `/chemistry` | `POST` | Manual chemistry reading |
| `/chemistry/additions` | `GET`, `POST` | Chemical-addition history and manual addition entry |
| `/rainfall` | `POST` | Standalone rainfall event |
| `/rainfall/history` | `GET` | Rainfall history |
| `/tasks` | `GET` | Filterable task list |
| `/tasks/:id/complete` | `POST` | Complete task |
| `/tasks/:id/dismiss` | `POST` | Dismiss task |
| `/tasks/:id/approve` | `POST` | Approve automation task |
| `/tasks/:id/snooze` | `POST` | Snooze task |
| `/schedules` | `GET`, `POST` | Maintenance schedules |
| `/schedules/:id` | `PUT`, `DELETE` | Schedule mutation |
| `/controller/schedules` | `GET` | Read validated controller-native schedules when available |
| `/controller/schedules/:schedule_id` | `PUT` | Directly update one validated EasyTouch controller schedule slot |
| `/controller/clock` | `GET`, `PUT` | Read and provisionally update EasyTouch controller clock configuration |
| `/controller/pumps/configuration` | `GET` | Read live EasyTouch installed-pump configuration for the EasyTouch8 page |
| `/controller/pumps/:pump_id/configuration` | `PUT` | Update one installed EasyTouch pump configuration from a fresh live baseline |
| `/controller/heater` | `GET` | Read EasyTouch-owned heater status, configuration, and editable field availability |
| `/controller/heater/configuration` | `PUT` | Update EasyTouch-owned heater configuration through controller action `162` |
| `/controller/heater/settings` | `PUT` | Update EasyTouch-owned heater settings through controller action `136` |
| `/telemetry/temperatures/latest` | `GET` | Latest EasyTouch temperature telemetry snapshot |
| `/telemetry/temperatures/history` | `GET` | Historical EasyTouch temperature telemetry series |
| `/telemetry/pumps/latest` | `GET` | Latest EasyTouch pump telemetry snapshot |
| `/telemetry/pumps/history` | `GET` | Historical EasyTouch pump telemetry series |
| `/weather/forecast` | `GET` | Latest normalized weather forecast snapshot for the active pool |
| `/weather/history` | `GET` | Historical normalized weather series for the active pool |
| `/weather/forecast/refresh` | `POST` | Trigger a manual weather forecast refresh |
| `/seasonal` | `GET` | Active seasonal checklist |
| `/seasonal/:id/start` | `POST` | Start checklist |
| `/seasonal/:id/steps/:step_id/complete` | `POST` | Complete checklist step |
| `/notifications` | `GET` | Notification inbox |
| `/notifications/:id/read` | `POST` | Mark read |
| `/notifications/read-all` | `POST` | Mark all read |
| `/pool/cover` | `GET`, `POST` | Current cover state and update |
| `/pool/cover/history` | `GET` | Cover event history |
| `/swimmability` | `GET` | Current swimmability assessment for the active pool |
| `/slam` | `GET` | SLAM sessions |
| `/slam/start` | `POST` | Start SLAM |
| `/slam/:id/criterion` | `POST` | Mark a SLAM criterion |
| `/slam/:id/complete` | `POST` | Complete SLAM |
| `/slam/:id/abandon` | `POST` | Abandon SLAM |
| `/protocol/frames` | `GET` as SSE | Live RS-485 frame stream |
| `/protocol/bundles` | `GET`, `POST` | Saved Protocol Explorer frame bundles |
| `/protocol/bundles/:id` | `GET` | One saved Protocol Explorer frame bundle |
| `/protocol/bundles/compare` | `POST` | Compare two saved Protocol Explorer frame bundles |
| `/protocol/watch-sessions` | `POST` | Start a live Protocol Explorer watch session, optionally filtered to selected events |
| `/protocol/watch-sessions/:id` | `GET` | Read one watch session and its captured frames |
| `/protocol/watch-sessions/:id/stop` | `POST` | Stop a live Protocol Explorer watch session |
| `/protocol/decode` | `POST` | Decode a raw frame |
| `/protocol/annotations` | `GET`, `POST` | Protocol annotations |
| `/protocol/prompts` | `GET`, `POST` | Operator-assisted decoding prompts |
| `/protocol/remote-layout/request` | `POST` | Manual Pentair Remote Layout page request |
| `/protocol/pump-info/request` | `POST` | Manual Pentair EasyTouch pump-slot information request |
| `/protocol/controller-schedule/request` | `POST` | Manual Pentair EasyTouch controller schedule request |
| `/protocol/pump-config/write` | `POST` | Manual Pentair EasyTouch pump configuration write |
| `/protocol/raw-frame/send` | `POST` | Manual raw protocol frame send for Explorer diagnostics |
| `/protocol/simulate` | `POST` | Dry-run or live-send protocol command |
| `/settings` | `GET`, `PUT` | Pool-scoped operator settings |
| `/api/settings/weather-location` | `GET`, `PUT` | Weather location settings |
| `/api/settings/pool-chemistry` | `GET`, `PUT` | Pool chemistry bounds and targets |
| `/events` | `GET` as SSE | Main frontend event stream |
| `/health` | `GET` | Service and dependency health |
| `/healthz` | `GET` | Process liveness |
| `/readyz` | `GET` | Service readiness |
| `/platform/status` | `GET` | Aggregated platform service health |
| `/setup/status` | `GET` | Onboarding status |
| `/setup/complete` | `POST` | Finalize onboarding |

## Health contract

Splash-owned services should expose:

- `GET /healthz`
- `GET /readyz`
- `GET /health`
- `GET /metrics`

`GET /healthz` returns `200` when the process is alive.

`GET /readyz` returns `200` only when the service can perform its primary role.

`GET /health` returns dependency-aware rich service state:

```json
{
  "status": "healthy",
  "message": "Service is ready",
  "checks": {
    "nats": {
      "status": "healthy"
    }
  },
  "last_checked": "2026-05-11T18:00:00.000Z"
}
```

Canonical health values are:

- `healthy`
- `degraded`
- `unhealthy`
- `down`
- `unknown`

`GET /platform/status` returns the browser-facing aggregated platform health
defined in [Service Health Architecture](Architecture-Service-Health).

### `GET /chemistry/observations`

Purpose:
- return recent operator-entered pool-condition observations

First-slice response fields per observation:
- `id`
- `pool_id`
- `clarity`
- `algae_presence`
- `debris_level`
- `bather_load_estimate`
- `notes`
- `source`
- `recorded_at`
- `created_at`

First-slice qualitative values:
- `clarity`
  - `clear`
  - `slightly_hazy`
  - `cloudy`
  - `opaque`
- `algae_presence`
  - `absent`
  - `suspected`
  - `visible`
- `debris_level`
  - `none`
  - `light`
  - `moderate`
  - `heavy`
- `bather_load_estimate`
  - `none`
  - `light`
  - `moderate`
  - `heavy`

### `POST /chemistry/observations`

Purpose:
- persist an operator-entered pool-condition observation

Rules:
- first slice is manual-only
- at least one observational field should be present
- observations must remain distinct from chemistry readings and chemical
  additions
- first slice should treat these inputs as qualitative operator assessments,
  not sensor-derived observations

### `GET /chemistry/additions`

Purpose:
- return durable chemical-addition history for the active pool

Query parameters:
- `start`
- `end`
- `limit`

Rules:
- return newest-first by default
- additions are treatment actions, not chemistry measurements
- support time-bounded review for the Chemistry workflow and later History
  overlays

Response fields per addition event:
- `id`
- `pool_id`
- `chemical_type`
- `amount`
- `unit`
- `notes`
- `source`
- `recorded_at`
- `created_at`

### `POST /chemistry/additions`

Purpose:
- record what the operator added to the pool

First-slice request fields:
- `chemical_type`
- `amount`
- `unit`
- `notes` optional

First-slice supported `chemical_type` values:
- `liquid_chlorine`
- `cal_hypo`
- `trichlor`
- `dichlor`
- `muriatic_acid`
- `soda_ash`
- `baking_soda`
- `calcium_chloride`
- `stabilizer`
- `salt`
- `algaecide`
- `other`

First-slice supported operator-facing `unit` values:
- `gal`
- `qt`
- `oz`
- `lb`
- `kg`
- `g`
- `L`

Rules:
- `amount` must be positive
- `chemical_type` must be known
- `unit` must be allowed
- the API assigns `recorded_at` and `created_at` in the first slice
- additions must not be stored in `chemistry_readings`
- the response returns the saved addition event in the standard envelope

### `GET /controller/schedules`

### `PUT /controller/schedules/:schedule_id`

Purpose:
- update one validated EasyTouch controller-native schedule slot directly on the
  controller using the compact seven-byte payload flow

Rules:
- this route is limited to EasyTouch controller-native scheduling
- accept only schedule ids `1-12`
- the first write slice accepts only validated compact payload fields:
  - `mode`: `repeat` or `egg_timer`
  - `circuit_id`: integer
  - `start_time_minutes`: integer for `repeat`
  - `end_time_minutes`: integer for `repeat`
  - `days_mask`: integer `1-127` for `repeat`
  - `runtime_minutes`: integer greater than `0` for `egg_timer`
- reject `run_once`, delete, disable, and heat-setting mutations until packet
  captures validate them
- reject unsupported or unvalidated fields with `400`
- return `409` when controller schedule writes are unsupported for the active
  controller selection or cached controller state
- return `503` when protocol command transport is unavailable
- after a successful write, Splash should refresh the same schedule slot and use
  the refreshed controller-native schedule record as the browser-facing source
  of truth

Example request body:

```json
{
  "mode": "repeat",
  "circuit_id": 12,
  "start_time_minutes": 510,
  "end_time_minutes": 1020,
  "days_mask": 62
}
```

Example accepted response:

```json
{
  "data": {
    "command_id": "command-123",
    "status": "accepted"
  },
  "error": null
}
```


Purpose:
- expose controller-native schedule visibility for the Automation `Schedules`
  tab without pretending that Splash already owns all scheduling

Rules:
- this route is read-only in the first slice
- it is specific to validated controller-native schedule visibility and must
  not replace the separate maintenance `/schedules` resource
- it should return only schedule fields whose meanings are protocol-validated
- if controller schedule data is unavailable, stale, or not yet sufficiently
  decoded, the route should return an explicit status and explanatory message
  rather than inventing values
- for EasyTouch, validated controller-native schedule visibility currently comes
  only from decoded schedule-detail frames with action `17` or `145`
- `action 30` must not be treated as an EasyTouch schedule source in this route

Example response:

```json
{
  "data": {
    "source": "controller_native",
    "controller_type": "easytouch",
    "status": "unavailable",
    "message": "EasyTouch schedule payload is not yet fully decoded",
    "last_checked": "2026-05-11T18:00:00.000Z",
    "schedules": []
  },
  "error": null
}
```

When validated records exist, each returned schedule record should include:

- stable schedule slot or controller identifier
- target circuit or schedule owner when validated
- mode when validated
- start time in minutes after midnight when validated
- stop time in minutes after midnight when validated
- day mask or explicit days when validated
- enabled or active state when validated

### `GET /controller/heater`

Purpose:
- expose the EasyTouch-owned heater read model used by the EasyTouch8 hardware page

Rules:
- this route is specific to controller-owned heater configuration and heat settings
- it must not imply direct UltraTemp ownership by Splash
- it should return only validated controller-derived fields plus explicitly labeled cached write-confirmation fields when a live controller read for that field is not yet available
- unsupported direct-heater fields must remain absent rather than guessed

The first slice should return:
- detected heater type when validated
- solar or heat-pump enablement
- heating enabled
- cooling enabled
- freeze protection enabled
- current pool and spa heat modes
- current pool and spa setpoints when available
- current cool setpoint when available
- field availability metadata for configuration vs heat-setting edits

### `PUT /controller/heater/configuration`

Purpose:
- update EasyTouch-owned heater configuration from the EasyTouch8 hardware page

Accepted request body:
- `heater_type`
- `cooling_enabled`
- `freeze_protection_enabled`

Rules:
- the first slice supports only documented EasyTouch action `162` variants:
  - `ultratempHeatPumpCom`
  - `ultratempEtiHybrid`
- undocumented gas or solar-only action `162` writes must be rejected with `400`
- direct UltraTemp action `114` / `115` behavior must remain out of scope
- Splash must use the protocol command path rather than browser-direct frame transmission
- transport write alone is not sufficient evidence of configuration truth
- the API should complete only after observing a compatible controller refresh or cached controller acknowledgement path for the configuration write

### `PUT /controller/heater/settings`

Purpose:
- update EasyTouch-owned heat settings from the EasyTouch8 hardware page

Accepted request body:
- `pool_setpoint`
- `spa_setpoint`
- `pool_heat_mode`
- `spa_heat_mode`
- `cool_setpoint`

Rules:
- the route must build EasyTouch action `136`
- setpoints must be validated as explicit integers within the configured safe range
- pool and spa heat modes must each be `0-3`
- unsupported or unvalidated fields must be rejected with `400`
- Splash must use the protocol command path rather than browser-direct frame transmission
- transport write alone is not sufficient evidence of live controller state
- the API should complete only after observing a compatible refreshed controller status or equivalent verified controller follow-up
- freshness metadata
- raw payload or debug data only when exposed through an explicit diagnostic
  field rather than mixed into operator-facing schedule values

### `GET /telemetry/temperatures/latest`

Purpose:
- return the latest persisted EasyTouch temperature telemetry values for the
  Home dashboard widget and future analytics surfaces

Rules:
- values come from InfluxDB-backed telemetry persistence when configured
- return `air`, `pool_water`, `spa_water`, and `solar` only when samples exist
- expose both original and normalized units
- include the latest packet timestamp and controller timestamp when available
- if no temperature telemetry has been captured yet, return an explicit empty
  state rather than inventing zeros

### `GET /telemetry/temperatures/history`

Purpose:
- return time-series temperature history for browser charts and later
  maintenance-model inputs

Query parameters:
- `sensorType`: one of `air`, `pool_water`, `spa_water`, `solar`
- `start`: ISO 8601 UTC inclusive range start
- `end`: ISO 8601 UTC inclusive range end
- `interval`: optional downsample window such as `10m`, `1h`, or `1d`

Rules:
- API should follow the existing JSON envelope pattern
- history points should expose `timestamp`, `value`, `normalizedF`, and
  `normalizedC`
- the first slice may use simple bucketed reads aligned with the requested
  interval rather than prediction-oriented analytics

### `GET /telemetry/pumps/latest`

Purpose:
- return the latest persisted EasyTouch pump telemetry values for configured
  pumps

Rules:
- values come from InfluxDB-backed pump telemetry persistence when configured
- return direct pump identity metadata such as `pump_id` and `bus_address`
- expose at least `running`, `rpm`, `watts`, and `timestamp`
- if no pump telemetry has been captured yet, return an explicit empty state
  rather than inventing zeros

### `GET /telemetry/pumps/history`

Purpose:
- return time-series pump RPM and watt history for browser charts and later
  maintenance-model inputs

Query parameters:
- `pumpId`: optional API-facing pump id such as `pump-main`
- `start`: ISO 8601 UTC inclusive range start
- `end`: ISO 8601 UTC inclusive range end
- `interval`: optional downsample window such as `1m`, `5m`, `1h`, or `1d`

Rules:
- API should follow the existing JSON envelope pattern
- history points should expose `timestamp`, `rpm`, `watts`, and `running`
- the first slice may use simple bucketed reads aligned with the requested
  interval rather than analytics-specific rollups

### `GET /weather/forecast`

Purpose:
- return the latest normalized weather forecast snapshot for the active pool

Rules:
- Splash should return provider-agnostic normalized daily and hourly forecast
  data
- the first Open-Meteo slice should request `forecast_days=10`
- the route should include provider metadata, `fetched_at`, and a `stale` flag
- if no forecast has been fetched yet, return an explicit empty or unavailable
  state rather than inventing forecast values

Expected normalized daily fields:
- `date`
- `weather_code`
- `high_temp_f`
- `high_temp_c`
- `low_temp_f`
- `low_temp_c`
- `precipitation_probability_max`
- `precipitation_amount`
- `uv_index_max`
- `sunrise`
- `sunset`

Expected normalized hourly fields:
- `timestamp`
- `temperature_f`
- `temperature_c`
- `relative_humidity`
- `dew_point_f`
- `dew_point_c`
- `precipitation_probability`
- `precipitation_amount`
- `cloud_cover`
- `wind_speed_mph`
- `wind_speed_kph`
- `wind_gusts_mph`
- `wind_gusts_kph`
- `uv_index` when available


### `GET /weather/history`

Purpose:
- return persistence-backed normalized weather history for browser charts and
  later analytics surfaces

Query parameters:
- `metric`: normalized metric selector such as `temperature_f`, `cloud_cover`,
  `uv_index`, `precipitation_probability`, or `precipitation_amount`
- `start`: ISO 8601 UTC inclusive range start
- `end`: ISO 8601 UTC inclusive range end
- `interval`: optional downsample window such as `1h`, `6h`, or `1d`

Rules:
- the response should remain provider-agnostic and must not expose
  Open-Meteo-native field names outside the normalized weather contract
- the first slice may serve weather-history data from persisted Influx-backed
  forecast snapshots when available
- each series point should include `timestamp` and `value`
- the response should include provider metadata, the effective metric name, and
  a `stale` flag when the latest cached forecast backing the history is stale
- if no weather history has been captured yet for the requested metric, return
  an explicit empty state rather than inventing values

### `POST /weather/forecast/refresh`

Purpose:
- trigger a best-effort manual refresh of the active pool forecast using the
  configured weather provider

Rules:
- manual refresh should reuse cached latitude/longitude when already known
- if latitude/longitude are absent and a pool address is present, Splash may
  geocode before requesting the forecast
- manual refresh failures should preserve the last known valid forecast and mark
  it stale

## API design notes

- The onboarding wizard is frontend-driven and only persists actual domain objects plus `POST /setup/complete`
- Automation approval executes prebuilt normalized command intent rather than reconstructing protocol frames in the API
- Error and degraded-state UX should derive from API responses and SSE
  connection state together; `GET /platform/status` is the authoritative
  browser-facing semantic health source while SSE remains the live event
  transport
- The initial implementation should expose enough latest-state and control
  surface to support a browser view of air temperature, water temperature, salt
  level, and pump RPM plus a controller-managed pump-circuit RPM control action
- the first Protocol Explorer implementation slice may expose `/protocol/frames`
  before broader protocol decode, annotation, and simulator endpoints are fully
  implemented
- the first saved-frame-bundle slice may remain in-memory and non-persistent as
  long as it captures recent `protocol.frame.raw` and `protocol.frame.decoded`
  traffic for one controlled experiment window
- the first annotation slice may also remain in-memory and API-local as long as
  annotations carry explicit confidence and target saved bundles or frame
  positions rather than pretending to be durable protocol truth
- the first operator-prompt slice may remain in-memory and API-local as long as
  prompts are explicitly tied to saved bundles and unresolved decoding work
- later Protocol Explorer API slices should support collaborative decoding by
  exposing annotation confidence, saved frame comparisons, and operator-needed
  prompts without bypassing the shared protocol engine
- the first manual Remote Layout request slice may expose one Explorer-only
  route that accepts a single page index and triggers a protocol-diagnostic
  request through `splash-protocol`
- the first manual raw-frame send slice may expose one Explorer-only route that
  accepts explicit lowercase hex bytes and forwards them through
  `splash-protocol` without rewriting checksums or other frame fields
- the first manual pump-info request slice may expose one Explorer-only route
  that accepts an EasyTouch pump slot selector and triggers the validated
  `0xd8` diagnostic request through `splash-protocol`
- the first manual pump-config write slice may expose one Explorer-only route
  that accepts a structured EasyTouch `0x9b` pump configuration payload and
  forwards it through `splash-protocol` without pretending the normalized
  controller-circuit `set_speed` contract is finished
- the first watch-session slice may expose explicit start and stop routes so a
  controlled “watch live frames” capture window can be preserved and later
  displayed without relying on the rolling recent-frame buffer
- watch sessions may accept an explicit event filter so Explorer can capture
  only transport-layer serial I/O such as `serial.rx.raw` and `serial.tx.raw`
  when the operator wants to inspect exactly what `splash-serial` received and
  wrote
- `/protocol/frames` may include outbound diagnostic traffic such as
  `protocol.command.encoded` and `serial.tx.raw` so Explorer can show the exact
  request bytes that were written, not only received bus frames
- `/protocol/frames` may also include derived receive-side diagnostic events
  such as `protocol.frame.unidentified` so Explorer can show bytes that were
  classified as unknown during frame assembly without mislabeling them as valid
  protocol frames
- `/protocol/frames` may also include `protocol.frame.buffered` so Explorer can
  show bytes still retained by the decoder buffer rather than implying that
  they were ignored
- browser-facing API routes should emit CORS headers compatible with the
  frontend deployment origin, including the local developer topology where the
  frontend and API run on different loopback ports

## Example payloads

### `GET /pool`

```json
{
  "data": {
    "id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
    "name": "Backyard Pool",
    "pool_type": "inground",
    "water_type": "saltwater",
    "surface_type": "plaster",
    "volume_gallons": 18000,
    "surface_area_sqft": 420,
    "zip_code": "28052",
    "latitude": 35.2621,
    "longitude": -81.1873,
    "timezone": "America/New_York",
    "setup_complete": true
  },
  "error": null
}
```

### `GET /chemistry/latest`

Purpose:
- return the latest known chemistry reading for the active pool

Response:

```json
{
  "data": {
    "id": "7b22a40f-f3e0-4ac6-8d6d-f3cb4b4d4f7d",
    "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
    "ph": 7.5,
    "free_chlorine": 5.8,
    "total_chlorine": 6.1,
    "total_alkalinity": 90,
    "calcium_hardness": 260,
    "cyanuric_acid": 70,
    "source": "manual",
    "recorded_at": "2026-03-26T19:30:00Z",
    "created_at": "2026-03-26T19:30:03Z"
  },
  "error": null
}
```

### `GET /chemistry/history`

Purpose:
- return chemistry-reading history for charting and recent-entry review

Query parameters:
- `start`
- `end`
- `interval`

Rules:
- the first slice supports `raw` and `1d`
- `raw` returns individual persisted readings in ascending time order
- `1d` returns daily average series for each supported chemistry metric
- the first slice includes:
  - `ph`
  - `free_chlorine`
  - `total_chlorine`
  - `total_alkalinity`
  - `calcium_hardness`
  - `cyanuric_acid`
- sparse values remain `null`; the API must not invent omitted measurements
- tracked rainfall and salt remain separate platform data sources and are not part of manual chemistry-log payloads

Response:

```json
{
  "data": {
    "start": "2026-03-01T00:00:00Z",
    "end": "2026-03-31T23:59:59Z",
    "interval": "raw",
    "readings": [
      {
        "id": "7b22a40f-f3e0-4ac6-8d6d-f3cb4b4d4f7d",
        "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
        "ph": 7.5,
        "free_chlorine": 5.8,
        "total_chlorine": 6.1,
        "total_alkalinity": 90,
        "calcium_hardness": 260,
        "cyanuric_acid": 70,
        "source": "manual",
        "recorded_at": "2026-03-26T19:30:03Z",
        "created_at": "2026-03-26T19:30:03Z"
      }
    ],
    "series": [
      {
        "metric": "ph",
        "points": [
          { "recorded_at": "2026-03-26T19:30:00Z", "value": 7.5 }
        ]
      },
      {
        "metric": "free_chlorine",
        "points": [
          { "recorded_at": "2026-03-26T19:30:00Z", "value": 5.8 }
        ]
      }
    ]
  },
  "error": null
}
```

### `POST /chemistry`

Request:

```json
{
  "ph": 7.5,
  "free_chlorine": 5.8,
  "total_chlorine": 6.1,
  "total_alkalinity": 90,
  "calcium_hardness": 260,
  "cyanuric_acid": 70,
  "source": "manual"
}
```

Rules:
- manual entry is the only writable browser source in the first slice
- partial chemistry entries are allowed
- `recorded_at` is assigned by the API at save time and is not client-editable in the first slice
- if both `ph` and `free_chlorine` are omitted, the API accepts the entry but returns a non-blocking warning
- the API must reject requests when every manual chemistry field is absent
- after persistence succeeds, the API emits a `chemistry.reading` event

Response:

```json
{
  "data": {
    "reading": {
      "id": "7b22a40f-f3e0-4ac6-8d6d-f3cb4b4d4f7d",
      "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
      "ph": 7.5,
      "free_chlorine": 5.8,
      "total_chlorine": 6.1,
      "source": "manual",
      "recorded_at": "2026-03-26T19:30:03Z"
    },
    "warnings": []
  },
  "error": null
}
```

### `GET /pool/cover`

Purpose:
- return the current known cover state for the active pool

Response:

```json
{
  "data": {
    "current": {
      "id": "2a26b4b9-6f6f-4b95-8dc6-8459f2a7c44d",
      "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
      "state": "on",
      "cover_type": "solar",
      "source": "manual",
      "recorded_at": "2026-06-04T18:30:00Z",
      "created_at": "2026-06-04T18:30:03Z"
    }
  },
  "error": null
}
```

Rules:
- when no cover event has ever been recorded, return `"current": null`
- the first slice supports `state` values:
  - `on`
  - `off`
- the first slice supports `cover_type` values:
  - `unknown`
  - `solar`
  - `winter`
  - `safety`
  - `automatic`

### `POST /pool/cover`

Purpose:
- record a new manual cover event and make it the current cover state

Request:

```json
{
  "state": "on",
  "cover_type": "solar"
}
```

Rules:
- the first slice is manual-entry only
- the API assigns `recorded_at` at save time
- `cover_type` is required when `state = "on"`
- `cover_type` may be omitted when `state = "off"` and should be stored as
  `unknown`
- after persistence succeeds, the API emits a `pool.cover.event` event

Response:
- return the saved event in the standard envelope

### `GET /pool/cover/history`

Purpose:
- return prior cover events for operator review and later chart overlays

Query parameters:
- `start`
- `end`
- `limit`

Rules:
- return newest-first events
- default limit: `100`
- the first slice does not aggregate cover history

### `GET /swimmability`

Purpose:
- return a first normalized swimmability assessment for the active pool

Response:

```json
{
  "data": {
    "status": "good",
    "score": 82,
    "summary": "Water is currently suitable for swimming.",
    "headline": "Safe for Swimming",
    "confidence": "high",
    "last_chemistry_age_label": "3 days ago",
    "highlights": [
      {
        "tone": "positive",
        "label": "Chemistry in range"
      },
      {
        "tone": "positive",
        "label": "Recent test available"
      },
      {
        "tone": "positive",
        "label": "No active swim advisories"
      }
    ],
    "updated_at": "2026-06-04T19:20:00Z",
    "drivers": [
      {
        "key": "free_chlorine",
        "severity": "good",
        "message": "Free chlorine is within the configured target range."
      },
      {
        "key": "chemistry_recency",
        "severity": "caution",
        "message": "Chemistry confidence is aging because the pool is uncovered and UV is elevated."
      },
      {
        "key": "water_temperature",
        "severity": "good",
        "message": "Pool water is within the preferred swim range."
      }
    ],
    "inputs": {
      "chemistry_latest_at": "2026-06-04T18:45:00Z",
      "cover_latest_at": "2026-06-04T17:30:00Z",
      "forecast_fetched_at": "2026-06-04T19:00:00Z",
      "telemetry_latest_at": "2026-06-04T19:18:00Z"
    }
  },
  "error": null
}
```

Rules:
- first-slice `status` values:
  - `good`
  - `caution`
  - `poor`
  - `unknown`
- first slice is read-only
- first slice should not block on all inputs being present
- when required inputs are missing, return `status = unknown` with explanatory
  drivers
- first-slice `confidence` values:
  - `high`
  - `medium`
  - `low`
  - `unknown`
- `headline` should be a short operator-facing interpretation derived from
  `status`
- `last_chemistry_age_label` should be a human-readable label derived from the
  latest chemistry timestamp and may be `null` when no chemistry reading exists
- `highlights` should be a curated summary list rather than a raw dump of all
  drivers
- the first slice should evaluate:
  - latest manual chemistry reading
  - configured chemistry bounds
  - latest pool-cover state
  - latest weather forecast snapshot
  - latest water-temperature telemetry when available
- chemistry-reading age should be a confidence input
- chemistry confidence should degrade faster when:
  - the latest cover state is `off`
  - forecast or current UV is elevated
  - forecast or current air temperature is high
  - water temperature is elevated when telemetry is available
- chemistry confidence should degrade more slowly when the latest cover state
  is `on`
- rainfall since the last chemistry reading should degrade confidence further
  because rain can disturb chemistry after the last test

### `GET /notifications`

Purpose:
- return the current pool notification inbox

Query parameters:
- `status`
  - `unread`
  - `all`
- `limit`
- `type`

Example response:

```json
{
  "data": {
    "status": "unread",
    "limit": 50,
    "notifications": [
      {
        "id": "notification-1",
        "pool_id": "pool-1",
        "type": "chemistry_test_due",
        "severity": "warning",
        "title": "Chemistry test is due",
        "body": "The latest chemistry reading is older than the configured testing interval.",
        "read": false,
        "source": "system",
        "related_entity_type": "chemistry_reading",
        "related_entity_id": null,
        "created_at": "2026-06-04T21:00:00Z",
        "read_at": null
      }
    ]
  },
  "error": null
}
```

First-slice `type` values:
- `chemistry_test_due`
- `swimmability_caution`
- `swimmability_poor`
- `rain_since_test`
- `chemistry_value_stale`
- `chemistry_value_unavailable`

First-slice `severity` values:
- `info`
- `warning`
- `critical`

Rules:
- return newest-first notifications
- default to `status=unread` when omitted
- first slice should not invent notifications when required source data is missing

### `POST /notifications/:id/read`

Purpose:
- mark one notification as read

Rules:
- idempotent
- returns the updated notification record

### `POST /notifications/read-all`

Purpose:
- mark all unread notifications as read for the active pool

Rules:
- first slice affects only the active pool inbox
- returns the number of updated notifications

### `GET /api/settings/water-testing-schedule`

Purpose:
- return the configured water-testing schedule together with current freshness
  status for each tracked value

Response per item:
- `chemicalKey`
- `displayName`
- `enabled`
- `expectedIntervalValue`
- `expectedIntervalUnit`
- `staleThresholdValue`
- `staleThresholdUnit`
- `unavailableThresholdValue`
- `unavailableThresholdUnit`
- `status`
- `lastObservedAt`
- `updatedAt`

Example response:

```json
{
  "data": {
    "items": [
      {
        "chemicalKey": "free_chlorine",
        "displayName": "Free Chlorine",
        "enabled": true,
        "expectedIntervalValue": 3,
        "expectedIntervalUnit": "days",
        "staleThresholdValue": 3,
        "staleThresholdUnit": "days",
        "unavailableThresholdValue": 7,
        "unavailableThresholdUnit": "days",
        "status": "current",
        "lastObservedAt": "2026-06-05T14:30:00Z",
        "updatedAt": "2026-06-05T15:00:00Z"
      },
      {
        "chemicalKey": "combined_chlorine",
        "displayName": "Combined Chlorine",
        "enabled": true,
        "expectedIntervalValue": 7,
        "expectedIntervalUnit": "days",
        "staleThresholdValue": 7,
        "staleThresholdUnit": "days",
        "unavailableThresholdValue": 7,
        "unavailableThresholdUnit": "days",
        "status": "unavailable",
        "lastObservedAt": null,
        "updatedAt": "2026-06-05T15:00:00Z"
      }
    ],
    "source": "sqlite"
  },
  "error": null
}
```

Rules:
- the first slice tracks:
  - `free_chlorine`
  - `ph`
  - `total_alkalinity`
  - `combined_chlorine`
  - `calcium_hardness`
  - `cyanuric_acid`
  - `salt`
  - `water_temperature`
- supported `status` values are:
  - `current`
  - `stale`
  - `unavailable`
  - `disabled`
- `combined_chlorine` is a derived freshness item rather than a primary stored
  chemistry field
- `water_temperature` should prefer telemetry-derived observations

### `PUT /api/settings/water-testing-schedule`

Purpose:
- update the full water-testing schedule for the active pool

Validation rules:
- known chemistry key only
- interval values must be positive
- units are limited to supported values such as `hours` and `days`
- disabled items must not generate stale or unavailable alerts

### `PUT /api/settings/water-testing-schedule/:chemicalKey`

Purpose:
- update a single water-testing schedule item

Rules:
- the path chemistry key must match a known tracked value
- applies the same validation rules as the full-schedule update route

### `POST /api/settings/water-testing-schedule/reset`

Purpose:
- reset the water-testing schedule to Splash defaults

First-slice defaults:
- `free_chlorine`: every `3 days`
- `ph`: every `3 days`
- `total_alkalinity`: every `7 days`
- `combined_chlorine`: every `7 days`
- `calcium_hardness`: every `30 days`
- `cyanuric_acid`: every `30 days`
- `salt`: every `30 days`
- `water_temperature`: unavailable after `1 hour` without recent telemetry

### `GET /settings`

```json
{
  "data": {
    "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
    "chemistry_prompt_interval_days": 3,
    "maintenance_reminder_lead_days": 7,
    "notification_preferences": {
      "in_app": true,
      "email": false,
      "push": false
    },
    "weather_provider": "openmeteo",
    "weather_active_geocoding_provider": "geoapify",
    "weather_refresh_interval_hours": 6,
    "weather_location": {
      "location_mode": "address",
      "address_line1": "123 Main St",
      "address_line2": "",
      "city": "Gastonia",
      "state_region": "NC",
      "postal_code": "28054",
      "country": "US",
      "latitude": null,
      "longitude": null,
      "timezone": null,
      "geocoded_latitude": null,
      "geocoded_longitude": null,
      "geocoded_formatted_address": null,
      "geocode_provider": null,
      "geocoded_at": null
    },
    "weather_config": {
      "openmeteo": {
        "base_url": "https://api.open-meteo.com/v1"
      }
    },
    "pool_chemistry": {
      "free_chlorine": {
        "display_name": "Free Chlorine",
        "unit": "ppm",
        "minimum": 3,
        "target": 5,
        "maximum": 10,
        "enabled": true,
        "sort_order": 10
      },
      "ph": {
        "display_name": "pH",
        "unit": null,
        "minimum": 7.2,
        "target": 7.6,
        "maximum": 7.8,
        "enabled": true,
        "sort_order": 30
      }
    },
    "protocol_plugin": "pentair_easytouch",
    "protocol_config": {
      "controller_type": "easytouch",
      "controller_address": "0x10"
    },
    "sensor_provider": "manual",
    "sensor_config": {}
  },
  "error": null
}
```

### `GET /api/settings/weather-location`

Response:

```json
{
  "data": {
    "pool_id": "0d0d6c6e-7c38-4c0c-9e6d-d4c6c3f4d0f1",
    "locationMode": "coordinates",
    "addressLine1": null,
    "addressLine2": null,
    "city": null,
    "stateRegion": null,
    "postalCode": null,
    "country": null,
    "latitude": 35.2621,
    "longitude": -81.1873,
    "timezone": "America/New_York",
    "formattedAddress": "5056 Stone Ridge Drive, Gastonia, NC 28056, United States",
    "geocodedLatitude": null,
    "geocodedLongitude": null,
    "geocodeProvider": null,
    "geocodedAt": null,
    "locationStatus": "resolved"
  },
  "error": null
}
```

### `GET /api/settings/geocoding`

Response:

```json
{
  "data": {
    "activeProviderId": "geoapify",
    "providers": [
      {
        "id": "geoapify",
        "displayName": "Geoapify",
        "description": "Street-address geocoding via Geoapify.",
        "configurationRequirements": ["api_key"],
        "configFields": [
          {
            "key": "api_key",
            "label": "API Key",
            "description": "Geoapify API key used for geocoding requests.",
            "type": "password",
            "required": true,
            "secret": true,
            "placeholder": "Enter Geoapify API key",
            "configured": true,
            "value": null
          },
          {
            "key": "base_url",
            "label": "Base URL",
            "description": "Override the Geoapify geocoding API base URL when needed.",
            "type": "url",
            "required": true,
            "secret": false,
            "placeholder": "https://api.geoapify.com/v1",
            "configured": true,
            "value": "https://api.geoapify.com/v1"
          }
        ],
        "available": true,
        "unavailableReason": null
      },
      {
        "id": "openstreetmap",
        "displayName": "OpenStreetMap / Nominatim",
        "description": "Street-address geocoding via Nominatim. Public endpoints are low-volume only.",
        "configurationRequirements": ["user_agent"],
        "configFields": [
          {
            "key": "base_url",
            "label": "Base URL",
            "description": "Override the Nominatim base URL for self-hosted or alternate deployments.",
            "type": "url",
            "required": true,
            "secret": false,
            "placeholder": "https://nominatim.openstreetmap.org",
            "configured": true,
            "value": "https://nominatim.openstreetmap.org"
          },
          {
            "key": "user_agent",
            "label": "User-Agent",
            "description": "Required user-agent string for Nominatim requests.",
            "type": "text",
            "required": true,
            "secret": false,
            "placeholder": "Splash/1.0 (ops@example.test)",
            "configured": false,
            "value": ""
          },
          {
            "key": "email",
            "label": "Contact Email",
            "description": "Optional contact email appended to Nominatim requests.",
            "type": "email",
            "required": false,
            "secret": false,
            "placeholder": "ops@example.test",
            "configured": false,
            "value": ""
          }
        ],
        "available": false,
        "unavailableReason": "user_agent is required."
      }
    ]
  },
  "error": null
}
```

Rules:

- all implemented providers should be listed even when unavailable
- unavailable providers must include a human-readable reason
- when no active provider is configured, `activeProviderId` should be `null`
- secret config fields must report `configured: true|false` but must not return
  the underlying saved value
- provider config metadata should be sufficient for the UI to render
  provider-specific forms without hard-coding per-provider fields

### `PUT /api/settings/geocoding`

Request:

```json
{
  "activeProviderId": "geoapify"
}
```

Rules:

- only registered and available providers may be selected
- selecting an unknown provider must return `400`
- selecting an unavailable provider must return `400`

### `PUT /api/settings/geocoding/provider/:providerId`

Request:

```json
{
  "config": {
    "api_key": "geoapify-live-key",
    "base_url": "https://api.geoapify.com/v1"
  }
}
```

Rules:

- only registered providers may be updated
- config validation should be driven by the provider-defined `configFields`
- required fields must be enforced
- secret fields must accept new input but must not be echoed back in full
- sending an empty secret field should preserve the existing stored secret
  unless an explicit clear action is supported later
- provider availability should be recalculated after the save

### `GET /api/settings/pool-chemistry`

Purpose:
- return the active pool chemistry bounds together with source-selection
  metadata for each chemistry key

Per-setting fields:
- `source_mode`
- `source_binding`
- `available_sources`

Example per-setting shape:

```json
{
  "data": {
    "settings": [
      {
        "chemicalKey": "salt",
        "displayName": "Salt",
        "unit": "ppm",
        "minimum": 3000,
        "target": 3400,
        "maximum": 4000,
        "enabled": true,
        "sortOrder": 70,
        "source_mode": "hardware",
        "source_binding": {
          "provider_type": "chlorinator",
          "provider_id": "chlorinator-1",
          "measurement_key": "salt"
        },
        "available_sources": [
          {
            "provider_type": "chlorinator",
            "provider_id": "chlorinator-1",
            "measurement_key": "salt",
            "label": "EasyTouch Chlorinator Salt"
          }
        ]
      }
    ],
    "chemistry_prompt_interval_days": 3,
    "source": "sqlite"
  },
  "error": null
}
```

Rules:
- `manual` requires no source binding
- `hardware` requires a valid binding from the `available_sources` list for
  that chemistry key
- the first slice exposes `total_chlorine` as the configured chlorine-total
  setting and does not expose `combined_chlorine` as an independently editable
  chemistry key
- first slice may return an empty `available_sources` array when no compatible
  hardware source is known
- if a saved hardware binding is no longer available, it should still be
  returned as the saved binding and the response should mark it unavailable in
  the per-setting metadata

### `PUT /api/settings/pool-chemistry`

Additional update rules:
- `chemistry_prompt_interval_days` may be saved alongside chemistry bounds
- per-setting `source_mode` and `source_binding` may be updated with the same
  request
- `combined_chlorine` is not a supported first-slice update key; Splash derives
  combined chlorine from `total_chlorine - free_chlorine`
- `hardware` is allowed only when the selected binding is compatible with the
  chemistry key being updated

`locationStatus` rules:

- `resolved` when coordinate mode is active or when an address already has a
  stored geocoded result
- `requires_geocoding` when address mode is active and no geocoded coordinates
  are stored yet

### `PUT /api/settings/weather-location`

Coordinate request:

```json
{
  "locationMode": "coordinates",
  "latitude": 35.2621,
  "longitude": -81.1873,
  "timezone": "America/New_York"
}
```

Address request:

```json
{
  "locationMode": "address",
  "addressLine1": "123 Main St",
  "addressLine2": "",
  "city": "Gastonia",
  "stateRegion": "NC",
  "postalCode": "28054",
  "country": "US"
}
```

Address-save rules:

- when the submitted location appears to be a physical street address, Splash
  should geocode it immediately using the active geocoding provider
- when geocoding succeeds, the response should include:
  - `geocodedLatitude`
  - `geocodedLongitude`
  - `formattedAddress`
  - `geocodeProvider`
  - `geocodedAt`
- when no active geocoding provider is configured, return a validation error:
  - `No geocoding provider is configured. Select a provider in Settings.`
- when a street address cannot be geocoded, return a validation error:
  - `Unable to geocode this address. Please check the address or enter latitude/longitude.`
- when coordinates are submitted directly, Splash should not invoke a geocoding
  provider
- city-only names, ZIP-only inputs, and direct coordinates should continue to
  work without forced street-address geocoding in the first slice

Validation rules:

- `locationMode` is required
- `coordinates` mode requires valid `latitude` and `longitude`
- `address` mode requires:
  - `addressLine1`
  - `city`
  - `stateRegion`
  - `postalCode`
  - `country`
- invalid coordinate ranges should return `400`
- invalid address payloads should return `400`

Validation error example:

```json
{
  "data": null,
  "error": {
    "code": "validation_error",
    "message": "Weather location settings are invalid.",
    "details": {
      "latitude": "Latitude must be between -90 and 90.",
      "longitude": "Longitude must be between -180 and 180."
    }
  }
}
```

### `GET /api/settings/pool-chemistry`

Response:

```json
{
  "data": {
    "settings": [
      {
        "chemicalKey": "free_chlorine",
        "displayName": "Free Chlorine",
        "unit": "ppm",
        "minimum": 3,
        "target": 5,
        "maximum": 10,
        "enabled": true,
        "sortOrder": 10
      },
      {
        "chemicalKey": "total_chlorine",
        "displayName": "Total Chlorine",
        "unit": "ppm",
        "minimum": 3,
        "target": 5,
        "maximum": 10,
        "enabled": true,
        "sortOrder": 20
      }
    ],
    "source": "sqlite"
  },
  "error": null
}
```

Rules:

- the response returns the full known chemistry-bounds set in a stable order
- the first slice includes `total_chlorine` and does not include
  `combined_chlorine` as an independently configured chemistry setting
- unknown custom chemistry keys are not part of the first slice
- if SQLite is unavailable, the route may return a safe default-backed
  payload with an explicit non-`sqlite` source indicator

### `PUT /api/settings/pool-chemistry`

Request:

```json
{
  "settings": [
    {
      "chemicalKey": "free_chlorine",
      "minimum": 3,
      "target": 5,
      "maximum": 10,
      "enabled": true
    },
    {
      "chemicalKey": "total_chlorine",
      "minimum": 3,
      "target": 5,
      "maximum": 10,
      "enabled": true
    }
  ]
}
```

Validation rules:

- `settings` must be a non-empty array
- each item requires `chemicalKey`
- the first slice allows only the documented built-in chemistry keys
- `combined_chlorine` is not a supported first-slice key; combined chlorine is
  derived from `total_chlorine - free_chlorine`
- numeric values must be valid numbers when present
- `minimum` must not be greater than `target`
- `target` must not be greater than `maximum`
- invalid payloads should return `400` with field-specific error details

Validation error example:

```json
{
  "data": null,
  "error": {
    "code": "validation_error",
    "message": "Pool chemistry settings are invalid.",
    "details": {
      "ph": {
        "target": "Target must be greater than or equal to minimum and less than or equal to maximum."
      }
    }
  }
}
```

### `POST /protocol/bundles`

Request:

```json
{
  "label": "pool-high-before-change"
}
```

### `POST /protocol/watch-sessions`

Request:

```json
{
  "label": "serial-only-watch",
  "events": ["serial.rx.raw", "serial.tx.raw"]
}
```

Response:

```json
{
  "data": {
    "id": "c8b0e1a1-58cb-4c52-91c4-4f4d4d0b7190",
    "label": "serial-only-watch",
    "status": "active",
    "events": ["serial.rx.raw", "serial.tx.raw"],
    "frame_count": 0,
    "created_at": "2026-03-30T19:10:00Z",
    "stopped_at": null
  },
  "error": null
}
```

Response:

```json
{
  "data": {
    "id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
    "label": "pool-high-before-change",
    "frame_count": 12,
    "created_at": "2026-03-30T19:15:00Z"
  },
  "error": null
}
```

### `GET /protocol/bundles/:id`

Response:

```json
{
  "data": {
    "id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
    "label": "pool-high-before-change",
    "created_at": "2026-03-30T19:15:00Z",
    "frames": [
      {
        "event": "protocol.frame.raw",
        "captured_at": "2026-03-30T19:14:58Z",
        "payload": {
          "frame_id": "frame-1",
          "bytes_hex": "ff00ffa5"
        }
      },
      {
        "event": "protocol.frame.decoded",
        "captured_at": "2026-03-30T19:14:58Z",
        "payload": {
          "frame_id": "frame-1",
          "action_code": "0x02"
        }
      }
    ]
  },
  "error": null
}
```

### `POST /protocol/bundles/compare`

Request:

```json
{
  "baseline_bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
  "comparison_bundle_id": "ad7c11db-e6bb-4a8b-b986-1e7bb30bc0f3"
}
```

### `POST /protocol/watch-sessions`

Request:

```json
{
  "label": "watch remote layout response"
}
```

Response:

```json
{
  "data": {
    "id": "0fdd95b9-e0f3-47c6-89bd-542b9584ecf9",
    "label": "watch remote layout response",
    "status": "active",
    "frame_count": 0,
    "created_at": "2026-03-31T03:45:00Z",
    "stopped_at": null
  },
  "error": null
}
```

### `GET /protocol/watch-sessions/:id`

Response:

```json
{
  "data": {
    "id": "0fdd95b9-e0f3-47c6-89bd-542b9584ecf9",
    "label": "watch remote layout response",
    "status": "stopped",
    "frame_count": 3,
    "created_at": "2026-03-31T03:45:00Z",
    "stopped_at": "2026-03-31T03:45:08Z",
    "frames": [
      {
        "event": "protocol.command.encoded",
        "captured_at": "2026-03-31T03:45:02Z",
        "payload": {
          "bytes_hex": "ff00ffa5011022e1010001ba"
        }
      }
    ]
  },
  "error": null
}
```

### `POST /protocol/watch-sessions/:id/stop`

Response:

```json
{
  "data": {
    "id": "0fdd95b9-e0f3-47c6-89bd-542b9584ecf9",
    "label": "watch remote layout response",
    "status": "stopped",
    "frame_count": 3,
    "created_at": "2026-03-31T03:45:00Z",
    "stopped_at": "2026-03-31T03:45:08Z"
  },
  "error": null
}
```

### `POST /protocol/annotations`

Request:

```json
{
  "bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
  "frame_index": 0,
  "field_name": "payload_hex",
  "byte_start": 2,
  "byte_end": 3,
  "confidence": "inferred",
  "label": "likely circuit id",
  "notes": "Changes when Pool High is edited."
}
```

### `POST /protocol/prompts`

Request:

```json
{
  "bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
  "frame_index": 0,
  "field_name": "payload_hex",
  "prompt": "What circuit was active when this frame was captured?",
  "why": "This byte range changes with pump-circuit edits.",
  "input_type": "controller_menu_state",
  "operator_response": null
}
```

### `POST /protocol/raw-frame/send`

Request:

```json
{
  "protocol_name": "pentair_easytouch",
  "bytes_hex": "ff00ffa5011022e1010001ba"
}
```

Rules:

- Explorer-only diagnostic route
- accepts explicit lowercase hex bytes only
- does not normalize, recalculate, or rewrite protocol fields in the first
  slice
- returns transport-acceptance only; protocol-level meaning is determined later
  from observed receive-side traffic

Response:

```json
{
  "data": {
    "id": "4d889762-e77b-46b5-b9e7-f261f61a2110",
    "bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
    "frame_index": 0,
    "field_name": "payload_hex",
    "prompt": "What circuit was active when this frame was captured?",
    "why": "This byte range changes with pump-circuit edits.",
    "input_type": "controller_menu_state",
    "operator_response": null,
    "status": "open",
    "created_at": "2026-03-30T20:20:00Z",
    "resolved_at": null
  },
  "error": null
}
```

Response:

```json
{
  "data": {
    "id": "bc1ecfe3-80be-4875-a56b-4da0f246b15f",
    "bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
    "frame_index": 0,
    "field_name": "payload_hex",
    "byte_start": 2,
    "byte_end": 3,
    "confidence": "inferred",
    "label": "likely circuit id",
    "notes": "Changes when Pool High is edited.",
    "created_at": "2026-03-30T20:05:00Z"
  },
  "error": null
}
```

Response:

```json
{
  "data": {
    "baseline_bundle_id": "d4d9ec0e-8295-43be-9bcf-f6c1c6cc9b5f",
    "comparison_bundle_id": "ad7c11db-e6bb-4a8b-b986-1e7bb30bc0f3",
    "frame_pairs": [
      {
        "index": 0,
        "baseline_event": "protocol.frame.raw",
        "comparison_event": "protocol.frame.raw",
        "changed_fields": [
          {
            "field": "bytes_hex",
            "byte_changes": [
              {
                "byte_index": 9,
                "baseline": "01",
                "comparison": "02"
              }
            ]
          }
        ]
      }
    ]
  },
  "error": null
}
```
