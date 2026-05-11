# Pentair Observed Installation

This page records the current observed EasyTouch installation snapshot used during Splash protocol work.

Use this page for:
- equipment inventory
- panel-observed configuration
- current circuit table
- custom-name bank
- current pump-slot assignments
- current valve settings

Use the companion pages for:
- canonical protocol-family behavior: [Pentair EasyTouch / IntelliTouch Reference](Pentair-EasyTouch-Reference)
- experiment history and unresolved hypotheses: [Pentair EasyTouch Reverse Engineering](Research-Pentair-EasyTouch-Reverse-Engineering)

## Controller And Equipment

Current controller under test:
- Pentair EasyTouch 8
- single RS-485 IntelliFlo filter pump
- IntelliChlor chlorinator present
- UltraTemp heat pump present
- `Settings > Spa-side` = `Off`
- `Valves > A` = `None`
- `Valves > B` = `None`

Valve assignment options currently observed on the panel:
- `None`
- the standard 18 fixed, relay, and feature circuits
- `Heater`

## Current Pump Configuration

### Pump `#1`

Panel configuration:
- type: `VS`
- priming time: `5`
- priming speed: `550`
- slot `1`: `900 RPM`, circuit `Pool`
- slot `2`: `1100 RPM`, circuit `Pool low`
- slot `3`: `3400 RPM`, circuit `Pool high`
- slot `4`: `2100 RPM`, circuit `Cleaner`
- slot `5`: `1000 RPM`, circuit `None`
- slot `6`: `1000 RPM`, circuit `None`
- slot `7`: `1000 RPM`, circuit `None`
- slot `8`: `3000 RPM`, circuit `None`

Validated `0x18` request / reply pair:
- request: `ff00ffa5341021d8010101e4`
- reply: `ff00ffa5340f10181f018005020006030b040c0d0d08000300030003000b02844c4834e8e8e8b82606f5`

Validated decoded `0x18` payload values:
- pump slot: `1`
- pump type: `128` (`VS` on this installation)
- priming time: `5`
- primary selector: `2` (currently inferred `Pool`)
- slot `1`: selector `6`, RPM `900`
- slot `2`: selector `11`, RPM `1100`
- slot `3`: selector `12`, RPM `3400`
- slot `4`: selector `13`, RPM `2100`
- slot `5`: selector `0`, RPM `1000`
- slot `6`: selector `0`, RPM `1000`
- slot `7`: selector `0`, RPM `1000`
- slot `8`: selector `0`, RPM `3000`
- priming speed: `550`

### Pump `#2`

Current status:
- under active protocol testing
- has been observed in panel configuration as:
  - `None`
  - `VSF`
  - `VF`

Pump type options exposed by the panel:
- `VF`
- `None`
- `VS`
- `VSF`

Known Pump `#2` request / reply pair:
- request: `ff00ffa5341021d8010201e5`
- configured-pump reply example: `ff00ffa5340f10181f02400f0200001e001e001e001e001e001e001e001e001e00000000000000000290`

## Current Pump-Slot Selector Values

Validated ordinary selector values on this installation:

| Selector value | Panel selection |
| --- | --- |
| `0` | `None` |
| `6` | `Pool` |
| `11` | `Pool low` |
| `12` | `Pool high` |
| `13` | `Cleaner` |
| `16` | `Feature 6` |

Validated special virtual selector values:

| Selector value | Meaning |
| --- | --- |
| `128` | `Solar` |
| `129` | `Heater` |
| `130` | `Pool Heater` |
| `131` | `Spa Heater` |
| `132` | `Freeze` |

Interpretation note:
- these are pump-slot assignment selector values from the EasyTouch Pump menu
- they are not guaranteed to match the raw logical `circuitId` values returned by `0xcb -> 0x0b`

## Custom Name Bank

| Custom name index | Value |
| --- | --- |
| `0` | `Username-01` |
| `1` | `[space]Sername-02` |
| `2` | `Username-03` |
| `3` | `Username-04` |
| `4` | `Username-05` |
| `5` | `Username-06` |
| `6` | `Username-07` |
| `7` | `Username-08` |
| `8` | `Username-09` |
| `9` | `Username-10` |

## Logical Circuit Table

| Circuit ID | Circuit type | Circuit name | Function |
| --- | --- | --- | --- |
| `1` | `fixed/F` | `Spa - Err` | unresolved / special |
| `2` | `fixed/F` | `Pool` | `Master Pool` |
| `3` | `relay` | `Aux 1` | unresolved |
| `4` | `relay` | `Aux 2` | `Generic` |
| `5` | `relay` | `Lights` | `IntelliBrite` |
| `6` | `relay` | `Aux 3` | likely lights-related, still uncertain |
| `7` | `relay` | `Aux 5` | `Generic` |
| `8` | `relay` | `Aux 6` | `Generic` |
| `9` | `relay` | `Aux 7` | `Generic` |
| `10` | `feature` | `Pool low` | `Generic` |
| `11` | `feature` | `Pool high` | `Generic` |
| `12` | `feature` | `Cleaner` | `Generic` |
| `13` | `feature` | `Feature 4` | `Generic` |
| `14` | `feature` | `Feature 5` | `Generic` |
| `15` | `feature` | `Feature 6` | `Generic` |
| `16` | `feature` | `Feature 7` | `Generic` |
| `17` | `feature` | `Feature 8` | `Generic` |
| `18` | `solar` | `Aux Extra` | `Generic` |

## Confirmed Action `0x86` Feature Control Selectors

Live controller writes for action `0x86` have confirmed that the feature-circuit
control selector ids used for on/off writes are shifted relative to the older
provisional inventory assumptions above.

Confirmed so far:

- `Feature 1` -> selector id `11`
- `Feature 2` -> selector id `12`
- `Feature 3` -> selector id `13`

Current working write-selector mapping for action `0x86`:

- `Feature 1` -> `11`
- `Feature 2` -> `12`
- `Feature 3` -> `13`
- `Feature 4` -> `14`
- `Feature 5` -> `15`
- `Feature 6` -> `16`
- `Feature 7` -> `17`
- `Feature 8` -> `18`

`Aux Extra` remains unresolved for direct write-selector purposes and should not
be treated as confirmed writable until live validation exists.

## Current Installation Implications

For milestone-1 controller-managed pump-speed work on this installation:
- only Pump `#1` should be treated as the real RS-485 pump object
- controller-managed speed work should focus on Pump `#1`
- the currently relevant speed-bearing assignments are:
  - `Pool` -> `900 RPM`
  - `Pool low` -> `1100 RPM`
  - `Pool high` -> `3400 RPM`
  - `Cleaner` -> `2100 RPM`
- fixed body circuits such as `Pool` and `Spa` must not be treated as ordinary feature circuits even when they appear in the pump-slot selector domain
