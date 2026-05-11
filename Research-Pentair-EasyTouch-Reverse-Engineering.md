# Pentair EasyTouch Reverse Engineering

This page holds experiment-driven Pentair notes that are not yet stable enough for the canonical reference page.

Use this page for:
- before / after frame comparisons
- unresolved bytes and competing hypotheses
- installation-specific evidence trails
- payload areas that are useful diagnostically but not yet stable enough for the canonical reference

Use the companion pages for:
- stable protocol behavior: [Pentair EasyTouch / IntelliTouch Reference](Protocols-Pentair-EasyTouch-Reference)
- the current panel-observed installation snapshot: [Pentair Observed Installation](Protocols-Pentair-Observed-Installation)

## Reverse-Engineering Rules

- equipment is the primary source of configuration truth
- any live config write must begin with a fresh read from the equipment
- cached or inferred configuration is diagnostic only
- unresolved bytes should remain explicit `ASSUMPTION:` or `QUESTION:` items

## `0x9b` Pump-Configuration Research Notes

Current working `0x9b` model:
- `0x9b` appears to be a pump-configuration command rather than part of the normal live control loop
- observed flow is currently believed to be:
  - remote `0x9b` request
  - controller `0x01` ACK
  - remote `0xd8` follow-up request
  - controller `0x18` broadcast / reply

Observed Pump `#1` `0x9b` payload example:

| Index | Value | Current interpretation |
| --- | --- | --- |
| `0` | `1` | pump slot / pump index = `1` |
| `1` | `128` | pump type = variable-speed pump |
| `2` | `0` | stale / older capture value; not authoritative after later live reads |
| `3` | `2` | likely assigned circuit / primary selector |
| `4` | `0` | unknown |
| `5` | `13` | earlier observed assignment byte |
| `6` | `4` | RPM high byte |
| `21` | `1` | older captured value; no longer trusted as current priming high byte |
| `22` | `126` | RPM low byte |
| `30` | `194` | older trailing byte from this captured write family |
| `31-45` | `0` | unknown / reserved |

Derived value from that observed payload:
- slot `1` RPM = `(payload[6] << 8) | payload[22] = 1150`

Important caution:
- older `0x9b` payload captures do not necessarily match the current live controller config
- a live write using a stale baseline was proven to overwrite unrelated Pump `#1` settings

## `0x18` Pump-Information Research Evidence

### Controlled Slot-2 Edit

Old Pump `#1` payload:
- `01 80 00 02 00 06 03 0b 04 0c 0d 0d 08 00 03 00 03 00 03 00 0b 01 84 4c 48 34 e8 e8 e8 b8 c2`

New Pump `#1` payload after changing slot `2` from `Pool low / 1100` to `Feature 6 / 1150`:
- `01 80 00 02 00 06 03 10 04 0c 0d 0d 08 00 03 00 03 00 03 00 0b 01 84 7e 48 34 e8 e8 e8 b8 c2`

Observed changes:
- byte `7`: `0x0b -> 0x10`
- byte `23`: `0x4c -> 0x7e`

What that proved:
- slot `2` assignment byte is `payload[7]`
- slot `2` RPM low byte is `payload[23]`
- `Feature 6` uses selector value `16`
- `1150 RPM` encodes as `0x04 0x7e`

### Priming Edit

Later controller reads after setting priming time `5` and priming speed `550` produced:
- byte `2` = `5`
- bytes `21` and `30` = `2` and `38`
- derived priming speed `(2 << 8) | 38 = 550`

What that proved:
- byte `2` is priming time on the observed VS Pump `#1` layout
- bytes `21` and `30` are priming speed high / low on the observed VS Pump `#1` layout

### Cleaner Slot Write Test

A live write changed the Cleaner slot from `2200` back to `2100`, but it also reverted unrelated values because the payload baseline was stale.

What that proved:
- slot `4` / Cleaner RPM bytes were correctly identified
- `0x9b` writes must be read-modify-write operations from a fresh live baseline
- partial or stale reconstruction is not safe enough

## Pump-Slot Selector Domain Hypothesis

Current evidence supports a broader selector domain than raw `circuitId` values.

Ordinary named selections observed live:
- `0` = `None`
- `6` = `Pool`
- `11` = `Pool low`
- `12` = `Pool high`
- `13` = `Cleaner`
- `16` = `Feature 6`

Special virtual trigger values observed or derived:
- `128` = `Solar`
- `129` = `Heater`
- `130` = `Pool Heater`
- `131` = `Spa Heater`
- `132` = `Freeze`

Research conclusion:
- pump-slot assignment values are not simply raw logical circuit ids
- the selector domain includes both ordinary named controller circuits and special virtual conditions

## `0x02` Controller-Status Research Notes

Trusted today:
- byte `0` = hour
- byte `1` = minute
- the observed EasyTouch 8 installation uses bytes `2-4` for circuit bitmasks

Still partly research-only:
- payload byte `9` mode / service / freeze interpretations
- payload byte `10` heater or body-routing interpretations
- payload bytes `11-22` mixed delay / temperature / firmware / heat-setting hypotheses

Current installation-specific bit evidence from controlled tests:
- byte `2`:
  - `0x01` = `spa`
  - `0x02` = `aux1`
  - `0x04` = `aux2`
  - `0x08` = `aux3`
  - `0x20` = `pool`
- byte `3`:
  - `0x04` = `pool_low`
  - `0x08` = `pool_high`
  - `0x10` = `cleaner`
  - `0x20` = `feature4`
  - `0x40` = `feature5`
  - `0x80` = `feature6`
- byte `4`:
  - `0x01` = `feature7`
  - `0x02` = `feature8`
  - `0x08` = `aux_extra`

Controlled feature-circuit sweep on `2026-04-22`:
- frame bundle: `feature-circuit-sweep-2026-04-22T21-05-07.685Z.json`
- toggled circuits: `pool_low`, `pool_high`, `cleaner`, `feature4`, `feature5`, `feature6`, `feature7`, `feature8`
- confirmed per-circuit deltas:
  - byte `3` `0x04` = `pool_low`
  - byte `3` `0x08` = `pool_high`
  - byte `3` `0x10` = `cleaner`
  - byte `3` `0x20` = `feature4`
  - byte `3` `0x40` = `feature5`
  - byte `3` `0x80` = `feature6`
  - byte `4` `0x01` = `feature7`
  - byte `4` `0x02` = `feature8`
- bytes `5-8` stayed `0x00` across that sweep, so no evidence from that run supports feature-circuit state living in bytes `5-8` on the observed controller
- multi-circuit expectation: circuits that share the same payload byte should combine with bitwise OR
  - example: `pool_low` `0x04` plus `cleaner` `0x10` predicts byte `3 = 0x14`
  - example: `feature7` `0x01` plus `feature8` `0x02` predicts byte `4 = 0x03`

Operator tooling note:
- the operator CLI/TUI sweep workflow should support both:
  - feature-circuit sweeps for `pool_low`, `pool_high`, `cleaner`, and `feature4-8`
  - non-feature system-circuit sweeps for the installed non-spa hardware set: `pool`, `aux1-7`, and `aux_extra`
- the system-circuit sweep is intended to validate the already-mapped EasyTouch `0x02` payload byte `2` and byte `4` masks and to probe unresolved AUX placement for `aux4-7` without changing the feature-circuit sweep workflow
- the operator CLI/TUI should also support two operator-assisted circuit-configuration mapping workflows:
  - `feature8-name-sweep`
    - target circuit: `Feature 8`
    - target circuit id / request index on the observed EasyTouch 8 installation: `17`
    - workflow:
      1. prompt the operator to change `Feature 8` assigned name on the controller panel to the next candidate label
      2. start a watch session before the protocol request is sent
      3. request controller circuit configuration for circuit index `17`
      4. capture the resulting `0x0b` `circuit_configuration` reply
      5. record the requested label together with the observed `nameId`, `name_label`, and full reply fields
  - `aux2-name-sweep`
    - target circuit: `AUX 2`
    - target circuit id / request index on the observed EasyTouch 8 installation: `4`
    - workflow:
      1. prompt the operator to change `AUX 2` assigned name on the controller panel to the next candidate label
      2. start a watch session before the protocol request is sent
      3. request controller circuit configuration for circuit index `4`
      4. capture the resulting `0x0b` `circuit_configuration` reply
      5. record the requested label together with the observed `nameId`, `name_label`, and full reply fields
  - `aux6-function-sweep`
    - target circuit: `AUX 6`
    - target circuit id / request index on the observed EasyTouch 8 installation: `8`
    - workflow:
      1. prompt the operator to change `AUX 6` circuit type on the controller panel to the next candidate label
      2. start a watch session before the protocol request is sent
      3. request controller circuit configuration for circuit index `8`
      4. capture the resulting `0x0b` `circuit_configuration` reply
      5. record the requested label together with the observed `functionId`, `base_function_id`, `base_function_label`, and full reply fields
  - `aux2-function-sweep`
    - target circuit: `AUX 2`
    - target circuit id / request index on the observed EasyTouch 8 installation: `4`
    - workflow:
      1. prompt the operator to change `AUX 2` circuit type on the controller panel to the next candidate label
      2. start a watch session before the protocol request is sent
      3. request controller circuit configuration for circuit index `4`
      4. capture the resulting `0x0b` `circuit_configuration` reply
      5. record the requested label together with the observed `functionId`, `base_function_id`, `base_function_label`, and full reply fields
- the saved report for these workflows should include the ordered requested labels plus the extracted observed configuration record for each step so the report itself can be used as the mapping table source without replaying raw frames

Current operator-assisted candidate labels for `feature8-name-sweep`:
- `FEATURE 1`, `FEATURE 2`, `FEATURE 3`, `FEATURE 4`, `FEATURE 5`, `FEATURE 6`, `FEATURE 7`, `FEATURE 8`, `AERATOR`, `AIR BLOWER`, `AUX 1`, `AUX 2`, `AUX 3`, `AUX 4`, `AUX 5`, `AUX 6`, `AUX 7`, `AUX 8`, `AUX 9`, `AUX 10`, `BACKWASH`, `BACK LIGHT`, `BBQ LIGHT`, `BEACH LIGHT`, `BOOSTER PUMP`, `BUG LIGHT`, `CABANA LIGHTS`, `CHEM FEEDER`, `CHLORINATOR`, `CLEANER`, `COLOR WHEEL`, `DECK LIGHT`, `DRAIN LINE`, `DRIVE LIGHTS`, `EDGE PUMP`, `ENTRY LIGHT`, `FAN`, `FIBER OPTIC`, `FIBER WORKS`, `FILL LINE`, `FLOOR CLEANER`, `FOGGER`, `FOUNTAIN`, `FOUNTAIN 1`, `FOUNTAIN 2`, `FOUNTAIN 3`, `FOUNTAINS`, `FRONT LIGHT`, `GARDEN LIGHTS`, `GAZEBO LIGHTS`, `HIGH SPEED`, `HI TEMP`, `HOUSE LIGHTS`, `JETS`, `LIGHTS`, `LOW SPEED`, `LO TEMP`, `MALIBU LIGHTS`, `MIST`, `MUSIC`, `NOT USED`, `OZONATER`, `PATH LIGHTS`, `PATIO LIGHTS`, `PARAMETER LIGHTS`, `PG2000`, `POND LIGHT`, `POND PUMP`, `POOL`, `Pool high`, `Pool light`, `Pool low`, `Pool sam`, `Pool sam 1`, `Pool sam 2`, `Pool sam 3`, `Security light`, `Sliude`, `Solar`, `Spa`, `Spa high`, `Spa light`, `Spa low`, `Spa sal`, `Spa sam`, `Spa waterfall`, `Spillway`, `SPRINKLERS`, `STREAM`, `STATUE LT`, `SWIM JETS`, `WTR FEATURES`, `WTR FTR LT`, `WATERFALL`, `WATERFALL  1`, `WATERFALL 2`, `WATERFALL 3`, `WHIRLPOOL`, `WTRFL LIGHT`, `YARD LIGHT`, `AUX EXTRA`

Current operator-assisted candidate labels for `aux6-function-sweep`:
- `GENERIC`, `MAST CLEANER`, `LIGHT`, `SAM LIGHT`, `SAL LIGHT`, `PHOTON GEN`, `COLOR WHEEL`, `VALVE`, `SPILLWAY`, `FLOOR CLEANER`, `INTELLIBRIGHT`, `MAGIC STREAM`

Open questions after custom-name-bank validation:
- `0xca` appears to request a direct custom-name-bank slot `0-9`
- `0x0a` appears to carry direct custom-name-bank payloads and is the current
  working write family for updating one custom-name-bank entry
- the mapping from `0xcb -> 0x0b` circuit `nameId` values to custom-name-bank
  slot indexes remains unresolved and must be determined experimentally rather
  than inferred

Current research caution:
- these circuit-bit results are strongly supported on the observed EasyTouch 8 controller
- they should not be generalized to all Pentair-family controllers without more controller variants

## Pump `#2` Research Notes

Pump `#2` has been used to probe multi-pump and alternate pump-type behavior.

Observed panel-side modes during testing:
- `None`
- `VSF`
- `VF`

VF panel-side structure observed during testing:
- filter circuit selection from the normal fixed / relay / feature domain
- `Flows`: 8 flow configurations, default `30 GPM`, circuit `None`
- `Filtering`: size `15000`, turns `2`, manual filter `30`
- `Priming`: max flow `55`, time `14`, sys time `1`
- `Backwash`: 2 slots, flow `60`, duration `5`
- `Vacuum`: flow `50`, time `10`

Open question:
- the latest `0x18` Pump `#2` rereads did not clearly reflect all VF-side panel changes
- this suggests either pump-type-dependent payload layouts or controller caching that still needs controlled validation

## Open Questions

- `QUESTION:` what is byte `4` in the VS `0x18` Pump `#1` payload?
- `QUESTION:` does `0x9b` use the exact same field ordering as the validated `0x18` read model in all cases, or are there write-only / write-expanded fields?
- `QUESTION:` how should VF / VSF Pump `#2` payloads be modeled when their field shape diverges from the VS Pump `#1` layout?
- `QUESTION:` which remaining `0x02` bytes can be promoted from diagnostic hypotheses into stable normalized semantics?
- `QUESTION:` what is the true broader configuration-discovery family beyond `0xe1 -> 0x21`, if one exists?
