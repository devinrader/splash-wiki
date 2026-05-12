# Runtime Model

[Back to Splash API Service](Services-Splash-API-Home)

## Process model

`splash-api` runs on `splash-core` as a TypeScript or Node.js service.

It should:

- expose LAN REST and SSE endpoints
- connect to NATS
- maintain an in-memory latest-state projection for the initial equipment slice
- publish normalized `protocol.command.intent`
- expose local dependency-aware health
- aggregate platform-wide semantic health for Splash-owned and selected
  third-party services
- expose a lightweight runtime diagnostics slice for live operator-visible rate
  summaries

## Initial runtime responsibilities

For the first browser milestone, `splash-api` should:

1. consume normalized equipment events:
   - `equipment.state.controller`
   - `equipment.state.pump`
   - `equipment.state.chlorinator`
2. maintain latest values for:
   - air temperature
   - water temperature
   - salt level
   - pump RPM
3. expose those values through REST
4. fan them out through SSE
5. accept pump-speed control requests and publish `protocol.command.intent`
6. consume `command.result.{command_id}` and relay command progress through SSE
7. expose a first Protocol Explorer frame-bundle slice by:
   - buffering recent `protocol.frame.raw`, `protocol.frame.unidentified`,
     `protocol.frame.decoded`, `protocol.command.encoded`, and `serial.tx.raw`
     events
   - allowing a client to save a bundle from that recent buffer
   - returning saved bundles through REST
8. expose a first Protocol Explorer frame-diff slice by:
   - comparing two saved bundles
   - highlighting changed byte offsets for known hex-bearing fields
   - returning the comparison through REST
9. expose a first confidence-aware annotation slice by:
   - accepting annotations tied to saved bundles and frame positions
   - preserving annotation confidence and byte ranges
   - returning saved annotations through REST
10. expose a first operator-prompt slice by:
    - accepting prompts tied to saved bundles and frame positions
    - preserving the prompt question, rationale, and expected input type
    - returning saved prompts through REST
11. expose a first manual Remote Layout request slice by:
    - accepting a single page index from Protocol Explorer
    - publishing a diagnostic `protocol.command.intent`
    - keeping the flow explicitly Explorer-only rather than normal automation
      or dashboard control
12. expose a first manual raw-frame send slice by:
    - accepting explicit lowercase hex bytes from Protocol Explorer
    - publishing a diagnostic `protocol.command.intent`
    - keeping the flow explicitly Explorer-only rather than normal automation
      or dashboard control
13. expose a first manual pump-info request slice by:
    - accepting an EasyTouch pump slot selector from Protocol Explorer
    - publishing a diagnostic `protocol.command.intent`
    - keeping the flow explicitly Explorer-only rather than normal automation
      or dashboard control
14. expose a first manual controller-schedule request slice by:
    - accepting an EasyTouch schedule selector from Protocol Explorer
    - validating that the selector is within the currently trusted range `1-12`
    - publishing a diagnostic `protocol.command.intent`
    - keeping the flow explicitly Explorer-only rather than normal automation
      or dashboard control
15. expose a first manual pump-config write slice by:
    - accepting a structured EasyTouch `0x9b` pump configuration payload from
      Protocol Explorer
    - publishing a diagnostic `protocol.command.intent`
    - keeping the flow explicitly Explorer-only rather than normal automation
      or dashboard control
16. expose a first watch-session slice by:
    - starting an explicit capture window
    - recording all live Explorer frame events into that watch session
    - allowing an optional per-session event filter so a watch can be limited
      to transport-only serial activity such as `serial.rx.raw` and
      `serial.tx.raw`
    - stopping and returning the captured frame set for later inspection
17. expose a first controller-circuit configuration discovery request by:
    - accepting a diagnostic API request from the dashboard or Protocol Explorer
    - publishing a `protocol.command.intent` that asks the active protocol
      plugin to request controller circuit-configuration records
    - keeping the request diagnostic and non-mutating
    - relying on the protocol command coordinator to pace the request range as
      one in-flight `0xcb` request at a time
    - recording decoded `0x0b` `circuit_configuration` replies into the
      controller latest-state diagnostic configuration map so dashboard rows can
      show returned function/name byte values and labels
    - continuing to show the raw `0xcb` request frames and `0x0b` controller
      replies in the Protocol Explorer frame stream
17. expose a first custom-name-bank diagnostic request by:
    - accepting a diagnostic API request from the dashboard or Protocol Explorer
      for a custom-name-bank index `0-9`
    - publishing a `protocol.command.intent` that asks the active protocol
      plugin to issue a direct Pentair custom-name-bank read
    - treating this flow as direct name-bank discovery, separate from
      controller circuit-configuration discovery
    - decoding observed `0x0a` custom-name-bank payloads into the Protocol
      Explorer frame stream
    - not inventing or projecting circuit `nameId -> custom-name-bank slot`
      mappings until they are live-validated
    - projecting observed custom-name-bank values into controller latest-state
      so frontend views can render cached custom-name values without requiring a
      page-local protocol request
    - allowing the API to auto-trigger one startup discovery sweep for
      custom-name-bank indexes `0-9` when controller state becomes available and
      the cached bank is still empty
18. expose first controller date/time actions by:
    - accepting a diagnostic API request to ask the controller for date/time
      information
    - accepting a best-effort API request to sync controller date/time from the
      Splash system clock
    - publishing diagnostic `protocol.command.intent` messages for both paths
    - keeping both paths explicitly provisional until live EasyTouch command
      captures validate the exact protocol family and payload shape
    - when decoded `0x05` controller date/time replies are observed, projecting
      their provisional decoded fields into controller latest-state as a
      separate diagnostic record rather than overwriting the normalized
      controller-status clock fields
19. expose a first controller software-version diagnostic request by:
    - accepting a diagnostic API request from the dashboard or Protocol Explorer
    - publishing a `protocol.command.intent` that asks the active protocol
      plugin to issue a direct Pentair software-version request
    - decoding observed `0xfc` software-version replies into the Protocol
      Explorer frame stream with the currently validated firmware and
      bootloader byte meanings
    - projecting the currently validated `0xfc` reply fields into controller
      latest-state as a separate `controller_software_version_reply` diagnostic
      record so frontend views can show cached firmware and bootloader values
    - keeping unresolved payload bytes visible as diagnostic data rather than
      projecting guessed meanings into equipment state
20. expose first controller-native schedule visibility by:
    - exposing a read-only `GET /controller/schedules` route for the Automation
      `Schedules` tab
    - projecting only validated EasyTouch controller schedule fields from
      schedule-detail actions `17` and `145` into that route
    - when controller state first appears and no cached schedule records exist,
      issuing a one-time startup warmup request for EasyTouch schedule slots
      `1-12`
    - caching decoded schedule-detail replies in API latest-state so the
      Automation page can render from warmed controller data without requiring a
      manual request first
    - returning an explicit unavailable or stale status when schedule payloads
      are not yet sufficiently decoded
    - keeping controller-native schedule visibility separate from maintenance
      schedules and from future Splash-native scheduling ownership
21. expose first EasyTouch temperature telemetry persistence by:
    - subscribing to normalized EasyTouch temperature events derived from
      controller-status broadcasts
    - writing sampled temperature points to InfluxDB no more than once every 10
      minutes per controller and sensor type by default
    - storing both original reported unit/value and normalized Fahrenheit and
      Celsius values
    - exposing read-only latest and history routes for frontend widgets and
      future analytics
    - keeping the sampling interval as configuration-backed application logic so
      it can later become a user-configurable platform setting
    - reporting degraded local health when telemetry persistence is configured
      but InfluxDB is unavailable
    - treating InfluxDB as an optional platform dependency unless future pool
      settings mark telemetry persistence as required
22. expose authoritative platform service health by:
    - maintaining a central health registry for Splash and third-party services
    - probing or polling the registered service health adapters on a short
      interval
    - normalizing those results to the canonical
      `healthy/degraded/unhealthy/down/unknown` model
    - exposing `GET /platform/status` as the browser-facing aggregated health
      source
    - exposing local `/healthz`, `/readyz`, `/health`, and `/metrics`
    - emitting Prometheus metrics for health-check state, duration, failures,
      and freshness

## Platform health aggregation

`splash-api` is the authoritative browser-facing health aggregator.

Responsibilities:

- store a service registry that includes service name, type, criticality, and
  health adapter configuration
- check Splash-owned health endpoints rather than scraping frontend semantics
  from Prometheus or Grafana
- probe selected third-party targets such as NATS, Prometheus, and Grafana
- retain recent health snapshots for stale-data fallback
- compute overall platform rollup using service criticality
- provide the result through `GET /platform/status`

The initial registry should cover:

- `grafana`
- `prometheus`
- `nats`
- `influxdb` when telemetry persistence is configured
- `splash-api`
- `splash-serial`
- `splash-frontend`
- `splash-protocol`

The API should support adapter-specific checkers for:

- Splash-native `/health`
- simple HTTP health probes
- TCP probes
- NATS probes
- Prometheus probes
- Grafana probes
- InfluxDB health probes

`GET /platform/status` should be safe to return partial results when some
non-critical probes fail, as long as the response clearly marks those services
`unknown`, `down`, `degraded`, or `unhealthy` as appropriate.

## EasyTouch temperature telemetry persistence

Preferred ownership:
- the first telemetry writer should live inside `splash-api`
- this keeps InfluxDB persistence behind the existing API boundary rather than
  coupling time-series storage logic to the frontend or protocol decoder

Configuration:
- `INFLUX_URL`
- `INFLUX_TOKEN`
- `INFLUX_ORG`
- `INFLUX_BUCKET`

Sampling policy:
- maintain a per-controller, per-sensor last-write timestamp in API runtime
  memory
- if a sensor has no previously written value in the current process, write the
  first observed valid point immediately
- otherwise write at most once every `10m` per sensor type by default
- persist the latest observed value when the sampling interval boundary is
  reached rather than storing every `0x02` broadcast
- keep the interval logic isolated so it can later become a pool setting

Stored sensor set:
- `air`
- `pool_water`
- `spa_water` when a validated source exists
- `solar`

Measurement model:
- measurement name: `easy_touch_temperature`
- tags:
  - `controller_id`
  - `controller_type`
  - `sensor_type`
  - `body`
  - `source`
  - `service`
- fields:
  - `original_value`
  - `original_unit`
  - `normalized_f`
  - `normalized_c`
  - `raw_byte`
  - `raw_payload_json`
  - `packet_timestamp`
  - `controller_timestamp`

Read model rules:
- `GET /telemetry/temperatures/latest` should return the newest persisted point
  per sensor type
- `GET /telemetry/temperatures/history` should return range-based series for
  one or more requested sensor types
- if no telemetry exists yet, routes should return explicit empty data rather
  than a transport or parsing error

## Initial equipment catalog bridge

The first API slice may use a minimal local equipment catalog bridge rather than
the full PostgreSQL-backed inventory model.

Rules:

- the bridge should expose a stable API-facing `equipment_id`
- the bridge should map that id to:
  - `equipment_type`
  - `protocol_name`
  - pump read-state identity such as direct pump `bus_address` where useful
  - controller-circuit control metadata for milestone-1 `set_speed`
- the bridge should be limited to the initial milestone equipment targets
- the bridge should be replaceable by the later repository-backed equipment
  model without changing the public REST route shape
- when the API projects or reconciles controller circuit state from a circuit
  command, it must merge the target circuit bit into the existing `0x02`
  circuit-status byte rather than replacing that byte with the single-circuit
  mask value
- for enable actions, set the target bit with bitwise OR against the current
  byte value
- for disable actions, clear the target bit while preserving any other enabled
  circuit bits in that byte
- when the API accepts `set_circuit_state` for a controller circuit, it should
  do so only when the target circuit has a known writable controller mapping
- the API should reject `set_circuit_state` for controller circuits whose
  writable mapping is unknown, unsupported, or not installed
- accepting a controller-circuit state command must not be treated as an
  immediate state change; the next authoritative controller-state update
  remains the source of truth for resulting circuit state

ASSUMPTION: the initial bridge may still expose one pump entry for live RPM read-state, but milestone-1 `set_speed` should target a controller-managed EasyTouch circuit rather than a direct IntelliFlo RPM write.

ASSUMPTION: the first slice may also use a static configured `pool_id` for
command publication and latest-state projection until the repository-backed pool
model exists.

## Hardware description projection

The API should expose configured hardware inventory and live state as separate
but related concepts.

Responsibilities:

- load the pool's configured hardware description
- resolve the controller hardware profile, such as EasyTouch 4 or EasyTouch 8
- merge user-confirmed installed hardware with protocol-profile defaults
- merge the latest normalized controller state into that configured inventory
- expose enough metadata for clients to distinguish `on`, `off`,
  `unavailable`, `unsupported`, and `not installed`

For controller circuits, the API should project each known circuit with:

- stable `circuit_key`
- display label
- circuit type, such as `fixed`, `relay`, `feature`, or `aux_extra`
- configured installation state
- state source
- latest live state when available
- capability summary for read and write actions
- an explicit indicator of whether `set_circuit_state` has a known writable
  controller mapping
- optional protocol mapping metadata for diagnostics
- when the protocol distinguishes between configuration discovery and control
  selectors, expose both `configuration_circuit_index` and `write_circuit_id`
  rather than collapsing them into one ambiguous `circuit_id`

Example:

```json
{
  "id": "controller-main",
  "equipment_type": "controller",
  "hardware_profile_id": "pentair.easytouch8",
  "latest_state": {
    "circuits": {
      "pool": true
    }
  },
  "hardware": {
    "circuits": [
      {
        "circuit_key": "pool",
        "display_name": "Pool",
        "circuit_type": "fixed",
        "installed": true,
        "state_source": "controller_status_bitmask",
        "latest_state": true,
        "configuration_circuit_index": 2,
        "write_circuit_id": 2,
        "status_mapping": {
          "payload_byte": 2,
          "mask": "0x20"
        }
      },
      {
        "circuit_key": "aux4",
        "display_name": "Aux 4",
        "circuit_type": "relay",
        "installed": false,
        "configuration_circuit_index": 6,
        "write_circuit_id": 6,
        "state_source": "not_installed",
        "latest_state": null
      }
    ]
  }
}
```

Rules:

- do not treat an unchanging controller-status bit as proof that a relay is off
  when the hardware description says the relay is not installed
- do not render unsupported model capacity as unavailable live state
- do not require the protocol decoder to infer physical hardware that the
  controller cannot report
- keep raw latest-state values available for diagnostics, but use the merged
  hardware projection for dashboard inventory and command availability
- controller-circuit configuration discovery may request known circuit indexes
  from the controller, but the first implementation must treat the replies as
  diagnostic evidence until the hardware-description persistence model is in
  place
- decoded controller-circuit configuration replies should update any existing
  latest-state circuit-configuration entry for that circuit id rather than
  replacing the full circuit-configuration map

## Runtime diagnostics rates

`splash-api` should expose UI-facing rate summaries through its local health or
diagnostic surface.

Rules:

- RS485 rates exposed by `splash-api` should be derived from the live transport
  subjects the API already observes, specifically `serial.rx.raw` and
  `serial.tx.raw`
- those RS485 rates are service-local observed rates, not direct serial-driver
  or broker-global truth
- the first browser-facing connectivity metrics slice should focus on RS485
  traffic and connectivity before adding broader broker-rate cards
- RS485 receive and transmit messages-per-second values exposed through
  `GET /health` should be calculated as rolling `10s` averages rather than
  instantaneous `1s` snapshots so the frontend can sample the health endpoint
  at `10s` intervals without repeatedly collapsing active but bursty traffic to
  zero
- `splash-api` should remain the browser-visible aggregation boundary even when
  Prometheus and Grafana are deployed for operator observability
- Grafana should not become a required direct query dependency for the frontend
- broker-wide NATS rates should be derived from the existing NATS monitoring
  endpoint rather than inferred from API-local publish and subscribe activity
- the API may consume NATS monitoring `/varz` from an optional
  `API_NATS_MONITORING_URL` local configuration value
- when NATS monitoring is configured, the API should derive broker message
  rates from cumulative `in_msgs` and `out_msgs` values sampled over time
- when NATS monitoring is not configured or is temporarily unavailable, the API
  should expose the NATS broker rate slice as unavailable rather than inventing
  defaults
- first-slice API health data should expose:
  - RS485 receive messages per second
  - RS485 transmit messages per second
- RS485 connectivity status
- first-slice API health data may later expose:
  - broker NATS subscription count
  - broker NATS inbound messages per second
  - broker NATS outbound messages per second
- these health-facing rates are intended for frontend and operator diagnostics;
  they do not replace Prometheus-style counters in `splash-protocol`
- when Prometheus is available, operator dashboards may use Prometheus-derived
  RS485 rates for historical visualization while the frontend continues to use
  the API-facing summary contract
- direct custom-name-bank discovery should accept only explicit custom-name-bank
  indexes `0-9` and should not implicitly infer a bank index from a circuit
  `nameId` until that mapping is validated
- automatic custom-name-bank startup discovery should run only when the cached
  custom-name bank is empty for the current API session; the API should then
  serve the acquired bank values from latest-state cache through `GET /equipment`
- controller status broadcasts should retain the validated identity bytes in
  latest-state as `controller_sub_model_byte`, `controller_model_byte`,
  `controller_model_family`, and `controller_model_label`
- controller software-version replies should project into
  `latest_state.controller_software_version_reply` with
  `controller_firmware_major`, `controller_firmware_minor`,
  `bootloader_major`, `bootloader_minor`, and `updated_at`
- circuit-configuration replies should key off the configured
  `configuration_circuit_index`; control writes should use `write_circuit_id`
  when publishing `set_circuit_state`
- until the fuller hardware-description persistence model exists, the API
  should retain acquired controller circuit function and name metadata in its
  runtime latest-state projection and expose that snapshot through
  `GET /equipment`
- that runtime snapshot is the persisted source the dashboard should consult on
  reload before deciding whether to auto-trigger circuit-config discovery
- manual circuit-config discovery remains the explicit refresh path and may
  replace previously acquired runtime metadata

## Platform service health aggregation

`splash-api` may expose a first normalized platform-services health slice
through `GET /health` so the frontend can render a single operator-facing
platform status surface without calling infrastructure-local service endpoints
directly.

Rules:

- `splash-api` remains the aggregation boundary for browser-visible platform
  health
- the frontend should not call `splash-serial`, `splash-protocol`, or NATS
  monitoring endpoints directly
- the first aggregated `platform_services` slice may include:
  - `splash_serial`
  - `nats`
  - `splash_protocol`
  - `splash_frontend`
- each service entry may expose:
  - `status`
  - `summary`
  - `detail`
  - `updated_at`
- allowed `status` values for the first slice may be:
  - `ok`
  - `degraded`
  - `error`
  - `unavailable`
- `splash_serial` status may be derived from its local `GET /healthz` endpoint
  on the hardware host when that endpoint is configured and reachable
- `nats` status may be derived from the existing API NATS supervisor snapshot,
  with optional broker-monitoring detail layered in when available
- `splash_protocol` status may be derived from its local `GET /healthz`
  endpoint when configured and reachable
- `splash_frontend` status may initially be API-local and synthesized as `ok`
  while the current browser session is active and the frontend bundle is
  serving normally
- when a configured upstream health source cannot be reached, the API should
  report that service as `unavailable` or `degraded` rather than inventing a
  healthy result
- the first slice does not need full historical uptime accounting or alert
  rollups; it is intended as a current operator-facing platform summary

## Degraded behavior

The service should stay alive but degrade when:

- NATS is unavailable
- no latest equipment state has been observed yet
- command-result updates are temporarily unavailable

The service should fail fast when:

- static service configuration is invalid
- the HTTP bind fails

## Initial route expectations

The first slice should at least support:

- `GET /equipment`
- `POST /equipment/:id/control`
- `GET /events`
- `GET /healthz`
- `GET /readyz`
- `GET /health`
- `GET /platform/status`
- `GET /protocol/frames`
- `GET /protocol/bundles`
- `POST /protocol/bundles`
- `GET /protocol/bundles/:id`
- `POST /protocol/bundles/compare`
- `POST /protocol/watch-sessions`
- `GET /protocol/watch-sessions/:id`
- `POST /protocol/watch-sessions/:id/stop`
- `GET /protocol/annotations`
- `POST /protocol/annotations`
- `GET /protocol/prompts`
- `POST /protocol/prompts`
- `POST /protocol/remote-layout/request`
- `POST /protocol/pump-info/request`
- `POST /protocol/controller-schedule/request`
- `POST /protocol/pump-config/write`
- `POST /protocol/circuit-config/request`
- `POST /protocol/controller-datetime/request`
- `POST /protocol/controller-datetime/sync`
- `POST /protocol/raw-frame/send`

Rules:

- `GET /equipment` may return the minimal bridged equipment model plus latest
  live state needed for the milestone
- `POST /equipment/:id/control` should only accept the normalized pump
  `set_speed` action in the first slice
- command ids should be created by `splash-api`
- command progress should be exposed through SSE `command.result`
- the first saved-frame-bundle slice may remain in-memory and non-persistent
  while Protocol Explorer is still a local protocol-discovery tool
- the first frame-diff slice may stay purely API-local and compare saved
  bundles positionally without trying to infer protocol meaning
- the first annotation slice may stay API-local and in-memory before the
  repository-backed `protocol_annotations` model is implemented
- the first operator-prompt slice may stay API-local and in-memory before a
  broader task or notification workflow exists
- the first manual Remote Layout request slice may remain a thin API-to-NATS
  bridge as long as it is explicitly scoped to Protocol Explorer diagnostics
- the first manual raw-frame send slice may remain a thin API-to-NATS bridge as
  long as it is explicitly scoped to Protocol Explorer diagnostics and does not
  silently rewrite the operator-provided bytes
- the first manual pump-info request slice may remain a thin API-to-NATS bridge
  as long as it is explicitly scoped to Protocol Explorer diagnostics
- the first manual pump-config write slice may remain a thin API-to-NATS bridge
  as long as it is explicitly scoped to Protocol Explorer diagnostics
- the first watch-session slice may stay API-local and in-memory before a
  broader persistent capture model exists
- the first watch-session filter may remain a simple explicit allowlist of
  known Explorer event names rather than a general-purpose query language
- the API should allow browser-origin requests from the frontend deployment
  origin, including the local milestone topology where `splash-frontend` runs
  on `127.0.0.1:3000` and `splash-api` runs on `127.0.0.1:8080`
