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
      "geocode_provider": null,
      "geocoded_at": null
    },
    "weather_config": {
      "openmeteo": {
        "base_url": "https://api.open-meteo.com/v1",
        "geocoding_url": "https://geocoding-api.open-meteo.com/v1"
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
    "geocodedLatitude": null,
    "geocodedLongitude": null,
    "geocodeProvider": null,
    "geocodedAt": null,
    "locationStatus": "resolved"
  },
  "error": null
}
```

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
        "chemicalKey": "ph",
        "displayName": "pH",
        "unit": null,
        "minimum": 7.2,
        "target": 7.6,
        "maximum": 7.8,
        "enabled": true,
        "sortOrder": 30
      }
    ],
    "source": "sqlite"
  },
  "error": null
}
```

Rules:

- the response returns the full known chemistry-bounds set in a stable order
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
      "chemicalKey": "ph",
      "minimum": 7.2,
      "target": 7.6,
      "maximum": 7.8,
      "enabled": true
    }
  ]
}
```

Validation rules:

- `settings` must be a non-empty array
- each item requires `chemicalKey`
- the first slice allows only the documented built-in chemistry keys
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
