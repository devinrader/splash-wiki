# Normalized Contracts

[Back to README](Home)

## Purpose

This document defines the normalized equipment events and command vocabulary that sit above protocol decoding and below product workflows.

Application services should depend on these contracts rather than vendor-specific packet details.

## Capability profile contract

Splash uses a profile-based capability model.

- a capability profile is a reusable normalized template for a class of equipment
- each equipment instance resolves to zero or one active capability profile in v1
- effective capabilities are derived from the resolved profile plus configuration and runtime evidence
- application services validate commands against effective capabilities, not against protocol packet details

### Initial profile catalog

The following profile identifiers form the initial platform vocabulary for capability mapping.

| Profile ID | Equipment type | Specificity | Intended use |
| --- | --- | --- | --- |
| `generic.controller` | controller | generic fallback | Minimal controller profile when only high-level control state is known |
| `generic.pump` | pump | generic fallback | Minimal fallback for pumps with uncertain model identity |
| `generic.heater` | heater | generic fallback | Minimal fallback for heaters with uncertain model identity |
| `generic.chlorinator` | chlorinator | generic fallback | Minimal fallback for chlorinators with uncertain model identity |
| `generic.circuit` | circuit | generic fallback | Minimal fallback for controller-managed circuits |
| `pentair.variable_speed_pump` | pump | vendor fallback | Pentair-family variable-speed pump behavior when exact model identity is not confirmed |
| `pentair.intelliflo_vs` | pump | exact model | Primary v1 Pentair variable-speed pump profile |
| `pentair.easytouch_heater` | heater | vendor/controller-specific | Heater control through Pentair EasyTouch or IntelliTouch controller messaging |
| `pentair.intellichlor` | chlorinator | exact family | Salt chlorinator behavior exposed through Pentair integrations |
| `pentair.controller_circuit` | circuit | vendor/controller-specific | Pentair controller-managed circuit with named circuit-state control |

ASSUMPTION: The exact profile catalog will expand as more captures and equipment-model validation are completed, especially for Hayward and Jandy integrations.

### Capability object shape

```json
{
  "name": "set_speed",
  "tier": "core",
  "support": "supported",
  "mode": "write",
  "source": "protocol_plugin",
  "arguments": {
    "rpm": {
      "type": "integer",
      "min": 450,
      "max": 3450,
      "step": 10
    }
  },
  "constraints": [
    "requires_live_protocol_connection",
    "requires_confirmed_target_address"
  ]
}
```

Allowed `tier` values:

- `core`
- `extended`
- `diagnostic`

Allowed `support` values:

- `supported`
- `configured`
- `observed`
- `inferred`
- `unsupported`
- `unknown`

Allowed `mode` values:

- `read`
- `write`
- `read_write`

Allowed `source` values:

- `protocol_plugin`
- `equipment_inventory`
- `pool_circuits`
- `protocol_observation`
- `manual_override`

### Equipment capability projection

Each equipment read model should expose a resolved profile and effective capabilities.

Example:

```json
{
  "equipment_id": "uuid",
  "equipment_type": "pump",
  "resolved_profile_id": "pentair.intelliflo_vs",
  "resolved_profile_confidence": "confirmed",
  "effective_capabilities": [
    {
      "name": "set_speed",
      "tier": "core",
      "support": "supported",
      "mode": "write",
      "source": "protocol_plugin",
      "arguments": {
        "rpm": {
          "type": "integer",
          "min": 450,
          "max": 3450,
          "step": 10
        }
      }
    },
    {
      "name": "report_power",
      "tier": "extended",
      "support": "observed",
      "mode": "read",
      "source": "protocol_observation"
    }
  ]
}
```

Profile resolution should prefer the most specific known profile in the catalog, then fall back to vendor-level profiles, then generic profiles.

### Hardware description configuration

Splash needs a user-editable hardware description layer in addition to runtime
protocol decoding.

Purpose:

- describe which controller model or hardware family is installed
- describe which optional hardware is physically present
- describe model-level limits that cannot be discovered from controller status
  broadcasts alone
- allow the API and dashboard to distinguish `off`, `unavailable`,
  `unsupported`, and `not installed`

The hardware description is not the same as latest runtime state.

- hardware description answers "what can exist here?"
- installed hardware configuration answers "what is present at this pool?"
- normalized state answers "what did the controller last report?"

Example:

```json
{
  "controller": {
    "hardware_profile_id": "pentair.easytouch8",
    "model": "EasyTouch 8",
    "installed_options": {
      "spa": false,
      "aux_extra": true
    },
    "circuits": [
      {
        "circuit_key": "pool",
        "display_name": "Pool",
        "circuit_type": "fixed",
        "installed": true,
        "configuration_circuit_index": 2,
        "write_circuit_id": 2,
        "state_source": "controller_status_bitmask",
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
        "state_source": "not_installed"
      }
    ]
  }
}
```

Circuit mapping rules:

- `configuration_circuit_index` is the index used when requesting or storing
  controller circuit-configuration records, such as EasyTouch `0xcb -> 0x0b`
  discovery
- `write_circuit_id` is the selector id used by control actions such as
  EasyTouch `0x86` circuit on/off writes
- these fields must not be assumed identical unless that has been validated for
  the specific protocol family and action
- hardware description should carry both fields when the protocol requires
  them, rather than forcing API or frontend code to hard-code the distinction

Allowed `state_source` values:

- `controller_status_bitmask`
- `protocol_observation`
- `manual_configuration`
- `not_installed`
- `unsupported`
- `unknown`

Rules:

- API and dashboard projections must not infer physical relay installation from
  a missing status-bit change alone
- controller models define maximum supported inventory, such as EasyTouch 4
  versus EasyTouch 8 circuit limits
- installed hardware configuration narrows that inventory to what is physically
  present at the pool
- unmapped or unobservable installed hardware should render as `Unavailable`,
  not `Off`
- hardware configured as not installed should render as `Not installed` or be
  hidden by default, depending on the UI context
- protocol plugins may provide default hardware profiles, but user-confirmed
  installation configuration is authoritative for physical availability

## Normalized event contracts

### `equipment.state.controller`

```json
{
  "pool_id": "uuid",
  "event_id": "uuid",
  "occurred_at": "2026-03-26T20:00:02Z",
  "source": {
    "service": "splash-protocol",
    "protocol_name": "pentair_easytouch",
    "frame_id": "uuid"
  },
  "water_temp_f": 63,
  "air_temp_f": 73,
  "solar_temp_f": 32,
  "controller_hour_24": 14,
  "controller_minute": 5,
  "controller_mode_byte": 9,
  "controller_mode_label": "run + freeze protection",
  "heater": {
    "enabled": false,
    "mode": "off",
    "setpoint_f": 84
  },
  "freeze_protection": false,
  "mode": "pool",
  "active_circuit_keys": ["pool"],
  "circuits": {
    "pool": true,
    "spa": false,
    "aux1": false,
    "aux2": false,
    "aux3": false,
    "pool_low": false,
    "pool_high": false,
    "cleaner": false,
    "feature4": false,
    "feature5": false,
    "feature6": false,
    "feature7": false,
    "feature8": false,
    "aux_extra": false
  }
}
```

Notes:
- `controller_hour_24` and `controller_minute` are the raw EasyTouch controller clock values from `0x02` payload bytes `0` and `1`
- dashboard or API formatting may present those fields as a human-readable system time, but the raw hour and minute remain authoritative
- `controller_mode_byte` is the raw inferred EasyTouch `0x02` payload byte `9`
- `controller_mode_label` is a diagnostic label derived from the currently documented byte-`9` bit meanings and must be treated as inferred rather than fully validated

### `telemetry.temperature.easytouch`

Purpose:
- provide a persistence-oriented normalized EasyTouch temperature event derived
  from controller-status broadcasts
- let downstream services store time-series history without reparsing raw
  protocol frames

Source rules:
- source action is EasyTouch controller broadcast `0x02`
- `air`, `pool_water`, and `solar` should come from already-validated
  controller-status temperature bytes
- `spa_water` may be included only when a separately validated EasyTouch source
  exists for the active controller and decoder slice; otherwise omit it
- raw payload and raw source bytes for each reported sensor should remain
  available for debugging and persistence

Example:

```json
{
  "pool_id": "uuid",
  "event_id": "uuid",
  "occurred_at": "2026-05-12T12:00:00.000Z",
  "source": {
    "service": "splash-protocol",
    "protocol_name": "pentair_easytouch",
    "frame_id": "uuid",
    "action": 2,
    "label": "easytouch.action2"
  },
  "controller": {
    "controller_id": "default",
    "controller_type": "easytouch",
    "timestamp": {
      "hour_24": 12,
      "minute": 0
    }
  },
  "temperatures": {
    "air": {
      "original_value": 78,
      "original_unit": "F",
      "normalized_f": 78,
      "normalized_c": 25.6,
      "raw_byte": 78
    },
    "pool_water": {
      "original_value": 82,
      "original_unit": "F",
      "normalized_f": 82,
      "normalized_c": 27.8,
      "raw_byte": 82
    },
    "solar": {
      "original_value": 90,
      "original_unit": "F",
      "normalized_f": 90,
      "normalized_c": 32.2,
      "raw_byte": 90
    }
  },
  "raw_payload": [12, 0, 32, 0, 0, 0, 0, 0, 0, 9, 0, 0, 0, 0, 82, 0, 2, 7, 78, 90]
}
```

Rules:
- normalized temperature events should preserve both original reported unit and
  normalized Fahrenheit/Celsius values
- packet receive time is `occurred_at`
- controller timestamp is diagnostic metadata, not a substitute for packet
  receive time
- this event is persistence-oriented and does not replace
  `equipment.state.controller` as the main latest-state projection

### `weather.forecast.updated`

Purpose:
- provide a provider-agnostic normalized weather forecast snapshot for one pool
- let downstream services and the frontend work from Splash-owned normalized
  data rather than provider-native response bodies

Rules:
- the event must not leak provider-native field names outside the weather
  provider boundary
- daily forecasts should include 10 days for the first Open-Meteo slice
- hourly forecasts should include the provider-backed hourly range returned for
  the configured 10-day forecast window
- stale forecasts should continue to publish the last known valid normalized
  data with `stale: true` instead of disappearing from the API

### `equipment.state.pump`

```json
{
  "pool_id": "uuid",
  "event_id": "uuid",
  "occurred_at": "2026-03-26T20:00:03Z",
  "source": {
    "service": "splash-protocol",
    "protocol_name": "pentair_easytouch",
    "frame_id": "uuid"
  },
  "equipment_id": "uuid",
  "equipment_type": "pump",
  "bus_address": "0x60",
  "running": true,
  "rpm": 2800,
  "watts": 1450
}
```

### `telemetry.pump.easytouch`

Purpose:
- provide a persistence-oriented normalized EasyTouch pump telemetry event
  derived from validated pump `0x07` read-state
- let downstream services store RPM and watt history without reparsing raw
  protocol frames or projecting from frontend state

Rules:
- source action is EasyTouch pump-status family `0x07`
- the first slice may reuse already-decoded normalized `equipment.state.pump`
  values instead of duplicating low-level byte parsing in downstream services
- telemetry payloads must preserve the direct pump identity needed for
  per-pump history reads
- omitted values should remain omitted rather than being synthesized

```json
{
  "occurred_at": "2026-05-18T01:53:29.016Z",
  "source": {
    "service": "splash-protocol",
    "protocol_name": "pentair_easytouch",
    "frame_id": "uuid",
    "action": 7,
    "label": "easytouch.action7"
  },
  "pump": {
    "pump_id": "pump-main",
    "controller_id": "default",
    "controller_type": "easytouch",
    "bus_address": "0x60"
  },
  "metrics": {
    "running": true,
    "rpm": 2800,
    "watts": 1450
  }
}
```

### `equipment.state.chlorinator`

```json
{
  "pool_id": "uuid",
  "event_id": "uuid",
  "occurred_at": "2026-03-26T20:00:04Z",
  "source": {
    "service": "splash-protocol",
    "protocol_name": "pentair_easytouch",
    "frame_id": "uuid"
  },
  "equipment_id": "uuid",
  "equipment_type": "chlorinator",
  "salt_ppm": 3100,
  "output_percent": 40,
  "target_output_percent": 40,
  "status": "ok"
}
```

Partial normalized event rule:

- normalized events may omit fields that are not yet confidently mapped for the
  active protocol family
- omitted fields should not be synthesized from uncertain byte interpretations
- protocol-level diagnostics remain the source for incomplete or still
  reverse-engineered payload areas

Expanded first-slice IntelliChlor-compatible fields may also include:

- `status_code`
- `water_temp_f`
- `model`
- `connected`
- `comms_lost`

Duty-cycle rule:

- `output_percent` and `target_output_percent` should be interpreted as
  configured or observed SWG duty-cycle support, not as proof of instantaneous
  active chlorine generation
- `current_output_percent` should not be normalized unless local captures
  validate that field for the active chlorinator installation
- `last_comm`
- `updated_at`

Rules:

- controller-observed chlorinator data and direct-control replies should both
  publish through this same normalized event family
- normalized chlorinator events may include only the trusted subset of fields
  available from the active action type
- unknown fields should be omitted rather than guessed

## Normalized command vocabulary

| Command type | Purpose | Typical target |
| --- | --- | --- |
| `set_speed` | Set controller-managed pump-circuit RPM | controller circuit |
| `set_run_state` | Start or stop equipment | pump, heater, circuit |
| `set_circuit_state` | Turn a named circuit on or off | controller circuit |
| `set_heater_setpoint` | Change heater target temperature | heater |
| `set_heater_mode` | Change heater operating mode | heater |
| `set_chlorinator_output` | Set chlorinator output percent | chlorinator |

### Initial profile-to-command expectations

| Profile ID | Expected write capabilities | Expected read capabilities |
| --- | --- | --- |
| `generic.controller` | `set_circuit_state` where circuit mapping exists | controller status fields and named circuit states where available |
| `generic.pump` | `set_run_state` if commandable | `report_rpm` or basic running state where available |
| `generic.heater` | `set_run_state`, `set_heater_setpoint` if supported | heater mode and setpoint where available |
| `generic.chlorinator` | `set_chlorinator_output` if supported | `report_salt_level`, chlorinator status |
| `generic.circuit` | `set_circuit_state`, `set_speed` where a circuit owns controller-managed pump speed | on/off state |
| `pentair.variable_speed_pump` | `set_speed`, `set_run_state` | `report_rpm`, `report_power` |
| `pentair.intelliflo_vs` | `set_speed`, `set_run_state` | `report_rpm`, `report_power` |
| `pentair.easytouch_heater` | `set_heater_setpoint`, `set_heater_mode`, `set_run_state` | heater mode, enabled state, setpoint |
| `pentair.intellichlor` | `set_chlorinator_output` | `report_salt_level`, output percent, chlorinator status |
| `pentair.controller_circuit` | `set_circuit_state`, `set_speed` where the controller circuit owns a stored pump-speed slot | named circuit state |

Validation rules for `set_circuit_state`:

- control is allowed only when the target controller circuit has a known
  writable controller mapping
- clients must not assume circuit state changed when the command is accepted;
  the next controller-status or equivalent authoritative controller update is
  the source of truth for the resulting state

### Example `set_speed`

```json
{
  "pool_id": "uuid",
  "command_id": "uuid",
  "command_type": "set_speed",
  "target": {
    "equipment_id": "uuid",
    "equipment_type": "circuit",
    "circuit_key": "pool_high"
  },
  "arguments": {
    "rpm": 2800
  }
}
```

Validation rule:

- `set_speed` must be rejected before encode if the target equipment lacks an effective capability named `set_speed` with a support state suitable for write use
- in milestone 1 for Pentair EasyTouch, `set_speed` is a controller-circuit-
  targeted normalized command
- the write path is controller-managed and updates the stored pump-circuit slot
  configuration rather than issuing a direct standalone pump RPM write
- the protocol layer is responsible for resolving the target circuit to the
  owning EasyTouch pump slot and then performing the `0x9b` write plus
  `0xd8 -> 0x18` read/confirm sequence

### Example `set_circuit_state`

```json
{
  "pool_id": "uuid",
  "command_id": "uuid",
  "command_type": "set_circuit_state",
  "target": {
    "circuit_key": "pool"
  },
  "arguments": {
    "state": "on"
  }
}
```

### Example `set_heater_setpoint`

```json
{
  "pool_id": "uuid",
  "command_id": "uuid",
  "command_type": "set_heater_setpoint",
  "target": {
    "equipment_id": "uuid",
    "equipment_type": "heater"
  },
  "arguments": {
    "setpoint_f": 84
  }
}
```

Current EasyTouch rule:
- when the active heater capability resolves to `pentair.easytouch_heater`, Splash should prefer controller-owned heater writes through EasyTouch action `136`
- Splash must not use direct UltraTemp action `114` / `115` traffic unless a future direct-equipment-control mode is explicitly designed and enabled

## Mapping rule

- protocol plugins may emit protocol-specific decoded fields for diagnostics
- scheduler, API, task logic, and general UI state must rely on normalized events and normalized commands
- Protocol Explorer may use both normalized and protocol-specific data because diagnostics are part of its purpose
- richer vendor-specific functionality should be represented as extended normalized capabilities when it maps to a real user or automation intent, not suppressed to a lowest common denominator
- application services should depend on effective capability projections rather than assuming all equipment of a given type supports the same commands
