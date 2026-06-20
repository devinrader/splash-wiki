# Data Model

[Back to README](Home)

## Model overview

Splash uses a dual-database model:

- SQLite for relational, configuration, and user-log data
- InfluxDB for time-series measurements and telemetry

The design is single-pool in v1, but child records carry `pool_id` to keep future multi-pool support possible without a schema reset.

`#109` transition note:
- the current implementation may still contain PostgreSQL-backed code while the
  migration is in flight
- the canonical target design is SQLite as the embedded relational store on
  `splash-core`

## SQLite entities

| Entity | Purpose | Key fields |
| --- | --- | --- |
| `pools` | Root pool profile | `name`, `pool_type`, `water_type`, `surface_type`, `volume_gallons`, `surface_area_sqft`, `street_address`, `city`, `state`, `postal_code`, `zip_code`, `latitude`, `longitude`, `timezone`, `setup_complete` |
| `equipment` | Physical equipment inventory | `pool_id`, `equipment_type`, `brand`, `model`, `bus_address`, `capability_profile_id`, `capability_profile_source`, `install_date` |
| `maintenance_schedules` | Reminder schedules | `equipment_id`, `interval_type`, `interval_value`, `last_performed_at`, `next_due_at` |
| `checklist_definitions` | Seasonal checklist templates | `pool_id`, `season`, `title`, `sort_order` |
| `checklist_steps` | Checklist steps | `definition_id`, `title`, `notes`, `sort_order` |
| `checklist_completions` | Checklist runs | `definition_id`, `started_at`, `completed_at` |
| `chemistry_readings` | User and sensor chemistry log | `pool_id`, `ph`, `free_chlorine`, `total_chlorine`, `total_alkalinity`, `calcium_hardness`, `cyanuric_acid`, `source`, `recorded_at`, `created_at` |
| `chemical_additions` | Durable treatment-action history | `pool_id`, `chemical_type`, `amount`, `unit`, `notes`, `source`, `recorded_at`, `created_at` |
| `water_additions` | Durable refill and top-up history | `pool_id`, `water_source`, `amount`, `unit`, `reason`, `notes`, `source`, `recorded_at`, `created_at` |
| `pool_cover_events` | Append-only cover state history, including retroactive manual backfill events | `pool_id`, `state`, `cover_type`, `source`, `recorded_at`, `created_at` |
| `slam_sessions` | SLAM workflow state | `status`, `cya_at_start`, `slam_fc_target`, `criterion_cc`, `criterion_clear`, `criterion_oclt`, `oclt_fc_before`, `oclt_fc_after` |
| `tasks` | Actionable work items | `status`, `priority`, `source`, `automation_command`, `due_at`, `snooze_until` |
| `notifications` | Notification inbox | `pool_id`, `type`, `category`, `severity`, `title`, `body`, `read`, `source`, `related_entity_type`, `related_entity_id`, `created_at`, `read_at`, `acknowledged_at`, `resolved_at`, `resolution_source` |
| `protocol_annotations` | Saved protocol-discovery notes | `pool_id`, `bundle_id`, `frame_index`, `field_name`, `byte_start`, `byte_end`, `confidence`, `label`, `notes` |
| `pool_settings` | Pool-scoped settings and integration configuration | `pool_id`, `chemistry_prompt_interval_days`, `maintenance_reminder_lead_days`, `notification_preferences`, `weather_provider`, `weather_refresh_interval_hours`, `weather_config`, `water_testing_schedule`, `protocol_plugin`, `protocol_config`, `sensor_provider`, `sensor_config` |
| `pool_circuits` | Circuit label and display-name mapping | `pool_id`, `circuit_key`, `display_name`, `circuit_type`, `bus_address`, `action_code`, `sort_order`, `enabled` |
| `hardware_descriptions` | Pool-scoped configured hardware inventory and model limits | `pool_id`, `hardware_profile_id`, `equipment_id`, `configuration`, `source`, `confirmed_at` |

## Planned swimmability-input expansions

The current implementation can compute a first current-state swimmability
assessment from chemistry readings, cover state, weather context, water
temperature telemetry, and schedule-driven freshness. Future prediction and
maintenance guidance require additional durable input domains that are not yet
first-slice relational entities.

Planned expansion tracks:

- `#133` chemical-addition events:
  - chemical type
  - amount
  - unit
  - source
  - recorded time
- `#149` water-addition and refill events:
  - water source
  - amount
  - unit
  - reason
  - source
  - recorded time
- `#134` observational pool-condition inputs:
  - water clarity
  - algae presence
  - debris level
  - bather-load estimate
  - first slice should store these as operator-entered qualitative
    assessments rather than attempting immediate sensor-based detection
- `#135` maintenance-activity history:
  - brushing
  - vacuuming
  - robot-cleaning
  - skimming
  - skimmer-basket cleaning
  - pump-basket cleaning
  - filter cleaning or backwashing
  - first slice should store these as immutable operator-entered
    maintenance events rather than inferred routine completion
- `#136` circulation summaries derived from pump telemetry:
  - runtime
  - circulation duration
  - runtime percent by recent window
  - sample coverage percent
  - last running timestamp
  - summary status such as `available`, `partial`, or `insufficient_data`
- `#137` chlorinator telemetry expansion:
  - SWG run state
  - SWG output percentage
  - related fault or availability context
  - first-slice latest-state fields:
    - `salt_ppm`
    - `output_percent`
    - `run_state`
    - `status`
    - `updated_at`
- `#138` filter and flow inputs:
  - filter pressure
  - flow rate
  - filter-condition status
  - first-slice latest-state fields:
    - `flow_gpm`
    - `filter_pressure_psi`
    - `filter_condition`
    - `updated_at`
- `#139` cover-derived exposure summaries:
  - covered duration
  - uncovered duration
  - daylight-uncovered duration
  - first-slice summary outputs such as:
    - `covered_minutes`
    - `uncovered_minutes`
    - `covered_percent`
    - `uncovered_percent`
    - `daylight_uncovered_minutes`
    - `last_cover_change_at`
    - `status` as `available`, `partial`, or `insufficient_data`
  - first slice should derive these on read from `pool_cover_events` rather
    than introducing a second persisted cover-state store
- `#140` normalized per-value provenance and confidence:
  - `value_kind` such as measured, observed, derived, predicted, or estimated
  - `source_type` and source detail
  - confidence band
  - freshness state such as fresh, aging, stale, missing, unavailable, or estimated
  - contradiction flags or reasons where inputs disagree
  - first slice should compute this metadata on read rather than introducing a giant universal provenance table immediately

ASSUMPTION: These tracks may first land as pool-scoped JSON-backed settings or
read models where that keeps the implementation smaller, but the input domains
themselves are part of the canonical future design.

## Entity relationships

- One `pool` owns all other major records
- `chemical_additions` are treatment actions and must remain separate from
  `chemistry_readings`, which represent measured water values
- `water_additions` are source-water events and must remain separate from both
  `chemistry_readings` and `chemical_additions`
- `equipment` optionally links into `maintenance_schedules`
- `checklist_definitions` own `checklist_steps` and `checklist_completions`
- `slam_sessions`, `tasks`, and `notifications` may reference related entities for UX and audit context
- `protocol_annotations` are pool-scoped so findings can differ by installation
- `protocol_annotations` may be attached to saved Protocol Explorer frame bundles so one controlled experiment can carry its own byte-level notes
- `pool_settings` is one-to-one with `pools`
- `pools.volume_gallons` is durable root-profile configuration, not transient
  telemetry; it should feed later chemistry, dosing, and SWG-support
  calculations when present
- `combined chlorine` is a derived interpretation value, not a first-slice
  primary stored chemistry setting; Splash derives it as `total_chlorine -
  free_chlorine` when both values exist
- `chemical_additions` should preserve what the operator added to the pool even
  when later chemistry readings make the treatment effect no longer visible
- `water_additions` should preserve what source water entered the pool, how
  much was added, and why the refill happened
- `water_testing_schedule` is pool-scoped configuration and stores freshness
  expectations for manually logged and sensor-derived water values
- `pool_circuits` is one-to-many from `pools`
- `hardware_descriptions` is pool-scoped and may reference one installed
  `equipment` row when the description applies to a specific controller or
  equipment instance
- future swimmability and recommendation engines should consume both durable
  user logs and derived telemetry summaries rather than relying only on the
  latest point-in-time values

## Pool profile settings

`#159` pool-volume guidance:

- `volume_gallons` belongs to the root `pools` profile rather than to the
  serialized `pool_settings` blob
- Splash may expose pool-profile editing through a `System` operator workflow,
  but the stored source of truth remains the durable pool profile record
- when `volume_gallons` is missing, ppm-normalized SWG support estimates and
  other volume-aware calculations should remain unavailable rather than being
  fabricated

## Equipment capability model

Capabilities describe what the platform believes a piece of equipment can do at the normalized command layer.

### Capability principles

- capabilities are platform-level, not protocol-packet-level
- capabilities may be configured, inferred from protocol observations, or both
- UI controls and automation suggestions must check capabilities before offering actions
- each equipment instance resolves to zero or one capability profile in v1
- capability profiles define baseline normalized capabilities; effective capabilities are derived from the assigned or inferred profile plus installation-specific evidence

### Capability examples

| Equipment type | Capability examples |
| --- | --- |
| pump | `set_speed`, `set_run_state`, `report_power`, `report_rpm` |
| heater | `set_heater_setpoint`, `set_heater_mode`, `set_run_state` |
| chlorinator | `set_chlorinator_output`, `report_salt_level` |
| light | `set_circuit_state` |
| generic circuit | `set_circuit_state` |

### Representation

ASSUMPTION: Capabilities should initially be represented as derived application state rather than a dedicated relational table.

### Capability profile model

A capability profile is a reusable normalized template that describes what a class of equipment is expected to report and control.

Examples:

- `pentair.intelliflo_vs`
- `pentair.easytouch_heater`
- `generic.circuit`
- `generic.pump`
- `pentair.intellichlor`

Profile characteristics:

- one equipment instance resolves to at most one active capability profile in v1
- profiles are defined by plugin and application logic, not authored as database rows
- profiles may exist at different specificity levels, such as exact model, vendor-family fallback, or generic equipment-type fallback
- effective capabilities are derived from the resolved profile plus pool configuration and observed runtime evidence

### Profile assignment and resolution

Persisted equipment fields:

- `capability_profile_id`: nullable stable profile identifier
- `capability_profile_source`: how the stored profile was assigned

Allowed `capability_profile_source` values:

- `user_confirmed`
- `setup_selected`
- `system_inferred`

Derived runtime fields:

- `resolved_profile_id`
- `resolved_profile_confidence`
- `effective_capabilities`

Allowed `resolved_profile_confidence` values:

- `confirmed`
- `probable`
- `unknown`

Resolution rule:

1. Use persisted `capability_profile_id` when it exists and is user-confirmed or setup-selected
2. Otherwise let the active protocol plugin infer the best-fit profile from inventory and protocol evidence
3. Otherwise fall back to a generic profile such as `generic.pump` or `generic.circuit`
4. Derive effective capabilities from the resolved profile plus circuit mappings and observed runtime evidence

Effective capabilities are a read model, not a primary persisted relational structure in v1.

Capability sources:

- resolved capability profile
- equipment type from `equipment`
- configured circuit mappings from `pool_circuits`
- protocol plugin knowledge
- observed decoded state where appropriate

### Hardware description model

Hardware descriptions persist the user-confirmed facts that the controller
cannot reliably broadcast.

Examples:

- installed controller family or model, such as `pentair.easytouch4` or
  `pentair.easytouch8`
- whether a spa body is physically attached
- which relay slots are physically installed
- whether `AUX EXTRA` exists on the installation
- model-level circuit limits and supported optional features

Hardware descriptions are configuration data, not runtime telemetry. They
should be merged with capability profiles and protocol observations to produce
effective capabilities and dashboard state.

Rules:

- model profiles define maximum supported hardware
- user-confirmed installed configuration defines actual physical availability
- protocol observations define latest state only for values the protocol can
  report
- when these sources disagree, user-confirmed installed hardware configuration
  wins for physical availability; protocol observations still win for mapped
  live state values

ASSUMPTION: The first implementation may store the description as JSON under a
pool-scoped or equipment-scoped record before normalizing it into dedicated
tables.

QUESTION: Should hardware descriptions be stored as a new first-class table, as
structured fields on `equipment`, or as a typed section of `pool_settings` for
the first implementation?

## Audit and key conventions

- UUID v4 primary keys on all relational entities that remain normalized rather
  than embedded in SQLite-friendly JSON structures
- `created_at` and `updated_at` on mutable entities
- Shared trigger updates `updated_at`
- Notifications are immutable after creation except for `read` state

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;
```

## Proposed relational definitions

Transition note for `#109`:
- the SQL examples in this section still use PostgreSQL-flavored DDL in places
  and should be translated to concrete SQLite migration files during
  implementation
- the logical entities remain authoritative, but the exact SQL dialect is now
  expected to be SQLite-first

### `pool_settings`

Purpose: store pool-scoped operational settings and service-configuration choices that do not belong in the immutable pool profile.

```sql
CREATE TABLE pool_settings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  pool_id UUID NOT NULL UNIQUE REFERENCES pools(id),
  chemistry_prompt_interval_days INTEGER NOT NULL DEFAULT 3,
  maintenance_reminder_lead_days INTEGER NOT NULL DEFAULT 7,
  notification_preferences JSONB NOT NULL DEFAULT '{"in_app": true, "email": false, "push": false}'::jsonb,
  weather_provider TEXT NOT NULL DEFAULT 'tomorrowio',
  weather_refresh_interval_hours INTEGER NOT NULL DEFAULT 6,
  weather_location_mode TEXT NOT NULL DEFAULT 'address' CHECK (weather_location_mode IN ('address', 'coordinates')),
  weather_location_address_line1 TEXT,
  weather_location_address_line2 TEXT,
  weather_location_city TEXT,
  weather_location_state_region TEXT,
  weather_location_postal_code TEXT,
  weather_location_country TEXT,
  weather_location_latitude NUMERIC(9,6),
  weather_location_longitude NUMERIC(9,6),
  weather_location_timezone TEXT,
  weather_active_geocoding_provider TEXT,
  weather_geocoding_provider_configs JSONB NOT NULL DEFAULT '{}'::jsonb,
  weather_geocoded_latitude NUMERIC(9,6),
  weather_geocoded_longitude NUMERIC(9,6),
  weather_geocoded_formatted_address TEXT,
  weather_geocode_provider TEXT,
  weather_geocoded_at TIMESTAMPTZ,
  chemistry_bounds JSONB NOT NULL DEFAULT '{}'::jsonb,
  weather_config JSONB NOT NULL DEFAULT '{}'::jsonb,
  protocol_plugin TEXT NOT NULL DEFAULT 'pentair_easytouch',
  protocol_config JSONB NOT NULL DEFAULT '{}'::jsonb,
  sensor_provider TEXT NOT NULL DEFAULT 'manual',
  sensor_config JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (weather_location_latitude IS NULL OR weather_location_latitude BETWEEN -90 AND 90),
  CHECK (weather_location_longitude IS NULL OR weather_location_longitude BETWEEN -180 AND 180),
  CHECK (weather_geocoded_latitude IS NULL OR weather_geocoded_latitude BETWEEN -90 AND 90),
  CHECK (weather_geocoded_longitude IS NULL OR weather_geocoded_longitude BETWEEN -180 AND 180),
  CHECK (
    weather_location_mode <> 'coordinates'
    OR (weather_location_latitude IS NOT NULL AND weather_location_longitude IS NOT NULL)
  ),
  CHECK (
    weather_location_mode <> 'address'
    OR (
      weather_location_address_line1 IS NOT NULL
      AND weather_location_city IS NOT NULL
      AND weather_location_state_region IS NOT NULL
      AND weather_location_postal_code IS NOT NULL
      AND weather_location_country IS NOT NULL
    )
  )
);
```

Notes:

- `protocol_plugin` identifies the active `splash-protocol` plugin for the pool
- `protocol_config` stores plugin-specific values such as default controller type, decoder options, or family-specific quirks
- `sensor_config` stores provider-specific sensor options without forcing schema churn
- `weather_location_mode` defines whether Splash should resolve forecasts from a
  physical address or a manual coordinate override
- `weather_active_geocoding_provider` stores the selected provider id used for
  address resolution
- `weather_geocoding_provider_configs` stores persisted provider-specific
  configuration values keyed by provider id and config field key
- `weather_location_latitude` and `weather_location_longitude` store the
  operator-managed coordinate override values when coordinate mode is active
- `weather_geocoded_latitude` and `weather_geocoded_longitude` reserve durable
  storage for resolved address-geocoding results
- `weather_geocoded_formatted_address` stores the provider-normalized best
  formatted address when a street address is successfully resolved
- `weather_location_timezone` stores an operator-provided or later geocoded
  timezone hint when available
- `weather_config` remains the extension point for weather-provider-specific
  metadata that does not justify first-class columns
- `weather_geocoding_provider_configs` should preserve provider-defined config
  metadata such as:
  - `key`
  - `type`
  - `value`
  - `updated_at`
  - secret markers when the provider field is sensitive
- `chemistry_bounds` stores pool-specific chemistry target ranges keyed by
  stable chemical identifiers such as `free_chlorine`, `ph`, `salt`, and
  `water_temperature`
- each `chemistry_bounds` entry should preserve:
  - `display_name`
  - `unit`
  - `minimum`
  - `target`
  - `maximum`
  - `enabled`
  - `source_mode`
  - `source_binding`
- `source_mode` values are:
  - `manual`
  - `hardware`
- `source_binding` is nullable and, in the first slice, should preserve:
  - `provider_type`
    - `controller`
    - `chlorinator`
  - `provider_id`
  - `measurement_key`
- `manual` means the operator supplies the value through manual workflows such
  as `Water Test Log`
- `hardware` means Splash derives the value from an available hardware source
  and that manual entry should not be treated as the primary source for that
  measurement
- first slice only needs obvious built-in hardware bindings such as:
  - `salt` from a chlorinator
  - `water_temperature` from the controller
  - `maximum`
  - `enabled`
  - `sort_order`
- defaults for a saltwater residential pool should be seeded once and then
  treated as operator-editable durable settings rather than hard-coded rules in
  later recommendation logic

ASSUMPTION: `protocol_plugin` should evolve toward protocol-family-oriented identifiers rather than vendor-only labels when one vendor exposes multiple materially different integration surfaces.

### `equipment`

Purpose: persist installed pool equipment and any confirmed profile assignment that should survive restarts and future inference changes.

Expected additions to the canonical `equipment` shape:

- `capability_profile_id TEXT NULL`
- `capability_profile_source TEXT NULL`

Notes:

- `capability_profile_id` stores a stable profile identifier such as `pentair.intelliflo_vs` when the profile has been explicitly selected or confirmed
- `capability_profile_source` distinguishes persisted user or setup intent from transient runtime inference
- the effective runtime profile may still be surfaced as `resolved_profile_id` when no persisted assignment exists
- initial valid examples include generic fallbacks and Pentair-family profiles defined by the platform capability catalog

### `pool_circuits`

Purpose: persist friendly labels and metadata for controller circuits exposed to the UI and automation engine.

```sql
CREATE TABLE pool_circuits (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  pool_id UUID NOT NULL REFERENCES pools(id),
  circuit_key TEXT NOT NULL,
  display_name TEXT NOT NULL,
  circuit_type TEXT NOT NULL DEFAULT 'generic',
  configuration_circuit_index INTEGER,
  write_circuit_id INTEGER,
  bus_address TEXT,
  action_code TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0,
  enabled BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (pool_id, circuit_key)
);
```

Notes:

- `circuit_key` is the stable machine identifier used in normalized events and command intents
- `display_name` is user-facing and editable
- `circuit_type` describes the functional circuit class, not the user-visible
  label
- `configuration_circuit_index` stores the controller-specific index used for
  configuration discovery requests and replies when that protocol exposes one
- `write_circuit_id` stores the controller-specific selector used for control
  actions when that differs from the configuration index
- on Pentair EasyTouch and IntelliTouch, this distinction matters:
  - fixed or special circuits such as `pool` and `spa` have controller-defined
    behavior and are not just user labels; renaming them does not change their
    internal `pool` or `spa` semantics
  - `aux` circuits are relay-backed controller outputs that may be renamed
  - `feature` circuits are virtual controller functions often used for pump
    speeds, valve-only actions, or logic functions
- the protocol-facing mapping fields must be optional because not every
  controller family exposes both concepts, but when they do they should be
  modeled explicitly rather than inferred in UI code
- user-facing names such as `POOL LOW` or `POOL HIGH` should therefore be
  treated as `display_name` values on a stable circuit type such as `feature`,
  not as evidence that the underlying circuit type is `pool`
- `bus_address` and `action_code` are optional protocol hints, not required for all implementations

## InfluxDB measurements

| Measurement | Source | Key fields |
| --- | --- | --- |
| `equipment_state` | controller 0x02 broadcast every 2s | controller time, water temp, air temp, solar temp, heater state, circuit states, freeze protection, mode |
| `pump_state` | pump 0x07 poll response | `running`, `watts`, `rpm`, `flow_gpm`, `filter_pressure_psi`, `filter_condition` |
| `chlorinator_state` | chlorinator broadcast or direct IntelliChlor reply | `salt_ppm`, `output_percent`, `current_output_percent`, `target_output_percent`, `run_state`, `status`, `status_code`, `water_temp_f`, `model`, `connected`, `comms_lost`, `last_comm`, `updated_at` |

IntelliChlor direct-control modeling rules:

- Splash should support one IntelliChlor per RS-485 port in direct-control mode
- direct control should remain opt-in and distinct from observed-only mode
- IntelliChlor production metadata should store:
  - `production_lb_per_day`
  - `production_lb_per_second`
- model metadata may be known even when controller-owned configuration remains
  partial

Configuration-model guidance:
- the root pool configuration should continue to own the pool-wide
  swimmability policy for swimmer-facing chemistry thresholds
- IntelliChlor should own a separate chlorinator operating profile for
  equipment-safe ideal and allowed chemistry thresholds
- overlapping chemistry keys should remain distinct records or documents so the
  UI and runtime can explain swimmer-safe versus equipment-safe differences
| `weather` | scheduler weather fetch | `temp_f`, `uv_index`, `humidity`, `condition`, `forecast_high_f`, `forecast_low_f`, `precip_chance_pct`, `actual_precip_in` |
| `weather_forecast_daily` | provider-normalized daily forecast snapshot | `weather_code`, `high_temp_f`, `high_temp_c`, `low_temp_f`, `low_temp_c`, `precipitation_probability_max`, `precipitation_amount`, `uv_index_max`, `sunrise`, `sunset`, `provider`, `fetched_at`, `forecast_generated_at` |
| `weather_forecast_hourly` | provider-normalized hourly forecast snapshot | `temperature_f`, `temperature_c`, `relative_humidity`, `dew_point_f`, `dew_point_c`, `precipitation_probability`, `precipitation_amount`, `cloud_cover`, `wind_speed`, `wind_gusts`, `uv_index`, `provider`, `fetched_at`, `forecast_generated_at` |
| `chemistry_sensor` | future automated chemistry hardware | `ph`, `free_chlorine`, `orp` |
| `rainfall` | manual or provider-derived rainfall events | `inches` |
| `easy_touch_pump` | EasyTouch `0x07`-derived pump telemetry sampled at most once every 1 minute per pump | `running`, `rpm`, `watts`, `packet_timestamp` |
| `easy_touch_temperature` | EasyTouch `0x02`-derived telemetry sampled at most once every 10 minutes per sensor type | `original_value`, `original_unit`, `normalized_f`, `normalized_c`, `raw_byte`, `raw_payload_json`, `packet_timestamp`, `controller_timestamp` |

## Retention policy

| Measurement | Retention |
| --- | --- |
| `equipment_state` | 30 days raw, then 5-minute downsample for 1 year |
| `pump_state` | 90 days raw, then hourly downsample for 2 years |
| `weather` | 1 year at 15-minute resolution |
| `weather_forecast_daily` | default bucket retention / indefinite until explicit retention policy is designed |
| `weather_forecast_hourly` | default bucket retention / indefinite until explicit retention policy is designed |
| `chemistry_sensor` | 2 years full resolution |
| `rainfall` | 2 years full resolution |
| `easy_touch_pump` | default bucket retention / indefinite until explicit retention policy is designed |
| `easy_touch_temperature` | default bucket retention / indefinite until explicit retention policy is designed |
| SQLite `chemistry_readings` | kept indefinitely as the permanent user log |

## Chemistry reference

### Parameter ranges and alert thresholds

| Parameter | Ideal target | Acceptable range | Alert low | Alert high | Critical low | Critical high |
| --- | --- | --- | --- | --- | --- | --- |
| pH | 7.4-7.6 | 7.2-7.8 | < 7.2 | > 7.8 | < 7.0 | > 8.2 |
| Free chlorine | CYA-adjusted | 1.0-5.0 ppm with CYA-adjusted minimum also met | < 1.0 ppm | > 5.0 ppm | < 0.5 ppm | > 8.0 ppm in SWG mode |
| Total alkalinity | 80-120 ppm, SWG: 80-100 | 60-180 ppm | < 60 ppm | > 180 ppm | TODO: not specified | TODO: not specified |
| Calcium hardness | 200-400 ppm, vinyl/fiberglass: 150-250 | 150-500 ppm | < 150 ppm | > 500 ppm | TODO: not specified | TODO: not specified |
| Cyanuric acid | 30-50 ppm, SWG: 60-80 | 20-80 ppm, SWG: 50-90 | < 20 ppm | > 80 ppm, SWG: > 90 ppm | TODO: not specified | > 100 ppm |
| Salt | 3000-3200 ppm | 2500-3500 ppm | < 2500 ppm | > 3500 ppm | < 2000 ppm | > 4500 ppm |

Do-not-swim conditions called out in the source:

- pH outside 7.0-8.2
- free chlorine below 0.5 ppm
- cyanuric acid above 100 ppm

### CYA-adjusted free chlorine reference

| CYA ppm | Minimum FC | Target FC | SLAM level | Notes |
| --- | --- | --- | --- | --- |
| 0 | 0.5 | 1-2 | TODO: not specified | No stabilizer |
| 20 | 1.5 | 2-3 | 10 |  |
| 30 | 2.0 | 3-4 | 12 |  |
| 40 | 3.0 | 4-5 | 16 |  |
| 50 | 4.0 | 5-6 | 20 |  |
| 60 | 5.5 | 5-7 | 24 | Typical SWG minimum target |
| 70 | 5.0 | 6-8 | 28 | Typical SWG upper target |
| 80 | 6.0 | 7-9 | 32 |  |
| 90 | 7.0 | 8-10 | 36 | Consider partial drain |
| 100+ | impractical | TODO: not specified | TODO: not specified | Chlorine lock; partial drain required |

## Data flow summary

1. `splash-serial` reads the RS-485 bus and publishes raw transport events to NATS.
2. `splash-protocol` reconstructs and decodes frames, then publishes normalized equipment events.
3. `splash-scheduler` evaluates rules against normalized state and long-term weather updates; the first forecast-fetch slice may temporarily live in `splash-api`.
4. `splash-api` persists relational updates to SQLite, exposes REST and SSE, emits normalized command intents when actions are approved, writes sampled EasyTouch temperature telemetry to InfluxDB when that integration is configured, and stores normalized weather forecast snapshots for frontend and later analytics use.

## Weather forecast cache model

Purpose:
- cache the latest normalized site-level weather forecast per pool
- avoid re-geocoding on every refresh
- preserve last-known-valid forecast data when provider refresh fails

Preferred pool profile fields:
- `street_address`
- `city`
- `state`
- `postal_code`
- `latitude`
- `longitude`
- `timezone`

Rules:
- latitude and longitude may be user-specified overrides or geocoded provider
  results
- when latitude/longitude already exist, provider refreshes should use them
  directly without repeated geocoding
- `weather_provider` remains pool-scoped configuration
- `weather_active_geocoding_provider` remains separate from `weather_provider`
  because address resolution and forecast retrieval may use different services
- `weather_refresh_interval_hours` should default to `6`
- the canonical durable weather-location settings should live on
  `pool_settings` rather than a disconnected per-feature table while Splash is
  still single-pool oriented
- `weather_config` may store provider-specific or future override metadata such
  as customer-endpoint settings or later geocoding cache details
5. The frontend bootstraps from REST and stays current through SSE and normalized events.

## Protocol metadata persistence

The system design does not require durable storage of every raw transport chunk. Persistence should be selective and focused on user-facing or diagnostic value.

### Persisted by design

- `protocol_annotations` in SQLite
- `pool_settings` in SQLite
- `pool_circuits` in SQLite
- user-facing equipment state in InfluxDB through normalized measurements
- command/task history in SQLite

### Not required as a primary store

- every `serial.rx.raw` chunk
- every `serial.tx.raw` write event
- every reconstructed raw frame

ASSUMPTION: Raw transport and protocol-frame traffic is primarily ephemeral and observability-oriented unless a future diagnostic archive feature is explicitly added.

### Potential future persistence

- QUESTION: Should a bounded protocol-frame archive be added for debugging difficult vendor-specific issues?
- TODO: If protocol-frame archival becomes a requirement, add explicit SQLite
  archival constraints, a larger relational store, or object-storage design
  rather than treating NATS as a historical store.

## Remaining schema questions

- QUESTION: Should `pool_settings.protocol_config` and `pool_settings.sensor_config` remain free-form JSONB long term, or later split into typed columns once real provider requirements stabilize?
