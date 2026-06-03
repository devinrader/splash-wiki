# EasyTouch Heater Control Architecture

[Back to README](Home)

## Current recommended architecture

Splash currently treats EasyTouch as the authoritative heater controller.

```text
Splash -> EasyTouch -> UltraTemp
```

Responsibilities:

### Splash

- recommend heater actions
- schedule or request setpoint changes
- issue controller-level heat mode requests
- apply weather-aware optimization logic
- surface controller-derived heater status in the UI

### EasyTouch

- own heater state and heat/cool mode transitions
- coordinate body priority between pool and spa
- manage compressor protection timing
- arbitrate automation interactions with other controller features
- integrate freeze behavior with heater behavior
- remain the source of truth for heater configuration

### UltraTemp

- operate as controller-managed equipment behind EasyTouch ownership

Why this is the current recommended model:
- it preserves Pentair’s built-in automation and safety coordination
- it reduces the risk of conflicting EasyTouch and Splash heater commands
- it leaves compressor timing and heat/cool sequencing with the controller that already owns the broader pool automation state
- it keeps Splash aligned with controller-native body, valve, and schedule behavior

## Future research architecture

Direct Splash-owned heater control is intentionally out of scope for the current implementation slice.

```text
Splash -> UltraTemp directly
```

This architecture would bypass EasyTouch heater ownership and would make Splash the effective heater controller.

Known risks:
- conflicting EasyTouch and Splash commands
- compressor short-cycling risk if direct timing behavior is not fully understood
- inconsistent UI and controller status if EasyTouch still believes it owns the heater
- heat/cool oscillation if mode arbitration is not centralized
- automation race conditions across schedules, freeze protection, and body changes

Safety warning:

Direct heater control bypasses portions of Pentair EasyTouch automation and may affect compressor protection, heating/cooling arbitration, and freeze safety behavior. This functionality is experimental and should not be enabled on production pool systems until fully validated with packet captures and real-world testing.

## Implementation guidance

Current protocol work should:
- target EasyTouch actions `34`, `136`, and `162`
- implement payload builders, parsers, validation, and tests
- avoid direct UltraTemp `114` / `115` transmission
- avoid automatic controller configuration changes
- present operator controls, when exposed, as EasyTouch-owned configuration and
  heat-setting requests rather than direct heater ownership

Future Direct Equipment Control Mode would likely require:
- explicit operator opt-in
- disabling or bypassing EasyTouch heater ownership cleanly
- command arbitration between services and equipment
- anti-short-cycle protection
- compressor lockout timing ownership
- pump and heater coordination
- freeze protection ownership
- state reconciliation with controller-facing UI
- heartbeat and status polling
- deterministic failure recovery behavior

In that future mode, Splash would become responsible for:
- heater safety orchestration
- automation synchronization
- heater state-machine ownership
