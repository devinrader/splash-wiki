# Pentair EasyTouch / IntelliTouch Reference

This page is the canonical Splash reference for the Pentair EasyTouch / IntelliTouch RS-485 protocol family.

Use this page for:
- frame structure
- checksum behavior
- address and action catalogs
- validated payload layouts
- stable value mappings

Use the companion pages for:
- installation-specific facts: [Pentair Observed Installation](Pentair-Observed-Installation)
- experiment history and unresolved hypotheses: [Pentair EasyTouch Reverse Engineering](../Research/Pentair-EasyTouch-Reverse-Engineering)

## Status

Status: `supported`

Pentair EasyTouch and IntelliTouch are the primary v1 protocol targets for Splash.

## Physical Layer

- medium: RS-485
- baud: `9600`
- data bits: `8`
- parity: none
- stop bits: `1`
- bus model: half-duplex, multi-device bus

## Frame Definition

Primary Pentair frame family:
- delimiter: `FF 00 FF A5`
- fields:
  - protocol/header byte
  - destination
  - source
  - action
  - payload length
  - payload
  - checksum
- checksum: 2-byte big-endian sum from `0xA5` through the end of payload

Checksum example:
- full request: `ff 00 ff a5 34 10 21 fd 00 02 07`
- checksum input bytes: `a5 34 10 21 fd 00`
- decimal sum: `165 + 52 + 16 + 33 + 253 + 0 = 519`
- hex checksum: `0x0207`

Current validation note:
- the `0xfd -> 0xfc` software-version request/reply flow is currently the strongest checksum validation on the observed EasyTouch installation
- Splash currently treats `include 0xa5 in the checksum basis` as the most strongly validated checksum rule for this controller-family frame set

Protocol-byte invariant:
- for the `FF 00 FF A5` frame family, Splash currently treats only `0x00` and `0x01` as valid protocol/header byte values
- bytes that do not fit a known frame family must remain visible as unknown data or unidentified frames

Observed multi-family nuance:
- controller and pump traffic use the `FF 00 FF A5` family
- Intellichlor traffic may also appear as a separate framed family using `10 02 ... 10 03`

## Addressing

| Value | Meaning |
| --- | --- |
| `0x10-0x1f` | controller address range |
| `0x20-0x2f` | remote / client address range |
| `0x60-0x6f` | IntelliFlo pump address range |
| `0x50` | Intellichlor destination in chlorinator-family traffic |
| `0x0f` | common controller-originated broadcast / panel destination |

## Known Message Areas

- `0x02`: controller status broadcast
- `0x07`: direct pump status / poll family under header `0x00`
- `0x19`: chlorinator information / status reply

## Value Mapping

| Decimal | Hex | Label | Category | Meaning / Notes | Confidence |
| --- | --- | --- | --- | --- | --- |
| `15` | `0x0f` | panel / broadcast destination | address | Common controller-originated destination | inferred |
| `16` | `0x10` | controller | address | EasyTouch controller address | strong |
| `33` | `0x21` | remote / client | address | Remote-emulation source used by Splash | strong |
| `52` | `0x34` | controller-family header byte | frame header | Common controller request / reply family | strong |
| `80` | `0x50` | Intellichlor | address | Chlorinator-family destination | inferred |
| `96` | `0x60` | IntelliFlo pump | address | Pump address family | strong |
| `1` | `0x01` | family-specific: pump exchange or controller ACK | action | Pump exchange under header `0x00`; controller ACK under header `0x34` | inferred |
| `2` | `0x02` | controller status | action | Repeating controller status broadcast | strongest current mapping |
| `4` | `0x04` | pump-state exchange | action | Pump-family only under header `0x00` | inferred |
| `5` | `0x05` | unknown controller reply | action | Observed during sweep; unresolved | unknown |
| `6` | `0x06` | pump drive-state exchange | action | Pump-family only under header `0x00` | inferred |
| `7` | `0x07` | pump status poll / reply family | action | Pump-family only under header `0x00` | inferred |
| `8` | `0x08` | heat / temperature information broadcast | action | Controller-family information reply | inferred |
| `10` | `0x0a` | custom-name value reply | action | Reply to `0xca`; fixed-width 12-byte payload | inferred from live request/reply |
| `17` | `0x11` | family-specific: schedule reply or chlorinator set-status | action | Pentair controller-family schedule reply; Aqualink-style chlorinator set-status | inferred |
| `18` | `0x12` | chlorinator action | action | Intellichlor-framed traffic; payload unresolved | inferred |
| `22` | `0x16` | spa-side information reply | action | Controller information reply | inferred |
| `24` | `0x18` | pump information reply | action | Controller pump information reply | inferred from live request/reply |
| `29` | `0x1d` | valve assignment reply | action | Controller valve-assignment reply | inferred |
| `30` | `0x1e` | high-speed circuits information reply | action | Controller information reply | inferred |
| `32` | `0x20` | IIS4 / IS10 information reply | action | Controller information reply | inferred |
| `35` | `0x23` | delays information broadcast | action | Controller information reply | inferred |
| `39` | `0x27` | IntelliBrite information reply | action | Controller information reply | inferred |
| `40` | `0x28` | settings information reply | action | Controller information reply | inferred |
| `134` | `0x86` | set circuit state | action | Request payload appears to be `[circuitNumber, state]` | inferred |
| `155` | `0x9b` | set pump circuit configuration | action | Controller pump-config write family | inferred |
| `200` | `0xc8` | heat / temperature information request | action | Controller information request | inferred |
| `202` | `0xca` | custom-name request | action | Request payload contains 0-based custom-name index | inferred from live request/reply |
| `203` | `0xcb` | circuit-configuration request | action | Request payload contains 1-based circuit index | inferred from live request/reply |
| `209` | `0xd1` | schedule-information request | action | Controller information request | inferred |
| `214` | `0xd6` | spa-side information request | action | Controller information request | inferred |
| `216` | `0xd8` | pump information request | action | Controller pump-information request | inferred from live request/reply |
| `217` | `0xd9` | chlorinator information request | action | Controller information request | inferred |
| `221` | `0xdd` | valves information request | action | Controller valve-assignment request | inferred |
| `222` | `0xde` | high-speed circuits information request | action | Controller information request | inferred |
| `224` | `0xe0` | IIS4 / IS10 information request | action | Controller information request | inferred |
| `225` | `0xe1` | remote-status request | action | Controller information request | inferred |
| `226` | `0xe2` | solar / heat-pump information request | action | Controller information request | inferred |
| `227` | `0xe3` | delays information request | action | Controller information request | inferred |
| `231` | `0xe7` | IntelliBrite information request | action | Controller information request | inferred |
| `232` | `0xe8` | settings information request | action | Controller information request | inferred |
| `252` | `0xfc` | software-version reply | action | Reply to `0xfd` | inferred from live request/reply |
| `253` | `0xfd` | software-version request | action | Zero-length payload | inferred from live request/reply |

## Action Catalog

| Action | Direction / Family | Current purpose | Payload structure status | Notes |
| --- | --- | --- | --- | --- |
| `0x01` | controller `<->` pump, header `0x00` | pump speed / flow exchange | partial | Pump-family only |
| `0x01` | controller reply, header `0x34` | ACK / request accepted | inferred | Generic controller ACK in controller-family traffic |
| `0x02` | controller broadcast | controller status | partial but strongest current mapping | Primary live state source |
| `0x04` | controller `<->` pump, header `0x00` | pump-state exchange | partial | Pump-family only |
| `0x05` | controller-originated | unknown | unknown | Observed during brute-force sweep |
| `0x06` | controller `<->` pump, header `0x00` | pump drive-state exchange | partial | Pump-family only |
| `0x07` | controller `<->` pump, header `0x00` | pump status poll / reply family | partial | Pump-family only |
| `0x08` | controller-originated, header `0x34` | heat / temperature information broadcast | inferred | Field mapping partial |
| `0x0a` | controller broadcast / reply | custom-name value response | inferred from live request/reply | Reply to `0xca` |
| `0x11` | family-specific | schedule-information response or chlorinator set-status | inferred | Meaning depends on frame family |
| `0x12` | chlorinator | chlorinator action | inferred | Intellichlor-framed traffic |
| `0x16` | controller broadcast / reply | spa-side information reply | inferred | Reply to `0xd6` |
| `0x18` | controller broadcast / reply | pump information reply | inferred from live request/reply | Reply to `0xd8` |
| `0x19` | chlorinator / controller reply | chlorinator information / status reply | partial | Reply to `0xd9` |
| `0x1d` | controller broadcast / reply | valve assignment reply | inferred | Reply to `0xdd` |
| `0x1e` | controller broadcast / reply | high-speed circuits information reply | inferred | Reply to `0xde` |
| `0x20` | controller broadcast / reply | IIS4 / IS10 information reply | inferred | Reply to `0xe0` |
| `0x21` | controller broadcast / reply | remote-status reply | inferred | Reply to `0xe1` |
| `0x23` | controller broadcast / reply | delays information reply | inferred | Reply to `0xe3` |
| `0x27` | controller broadcast / reply | IntelliBrite information reply | inferred | Reply to `0xe7` |
| `0x28` | controller broadcast / reply | settings information reply | inferred | Reply to `0xe8` |
| `0x86` | remote `->` controller | set circuit state | inferred | Payload appears to be `[circuitNumber, state]` |
| `0x9b` | remote `->` controller | set pump configuration | inferred | Pump-config write family |
| `0xc8` | remote `->` controller | heat / temperature information request | inferred | Reply family `0x08` |
| `0xca` | remote `->` controller | custom-name request | inferred from live request/reply | 0-based custom-name index |
| `0xcb` | remote `->` controller | circuit-configuration request | inferred from live request/reply | 1-based circuit index |
| `0xd1` | remote `->` controller | schedule-information request | inferred | Reply family `0x11` |
| `0xd6` | remote `->` controller | spa-side information request | inferred | Reply family `0x16` |
| `0xd8` | remote `->` controller | pump information request | inferred from live request/reply | Reply family `0x18` |
| `0xd9` | remote `->` controller | chlorinator information request | inferred | Reply family `0x19` |
| `0xdd` | remote `->` controller | get valve assignments | inferred | Reply family `0x1d` |
| `0xde` | remote `->` controller | high-speed circuits information request | inferred | Reply family `0x1e` |
| `0xe0` | remote `->` controller | IIS4 / IS10 information request | inferred | Reply family `0x20` |
| `0xe1` | remote `->` controller | remote-status request | inferred | Reply family `0x21` |
| `0xe2` | remote `->` controller | solar / heat-pump information request | inferred | Reply family unresolved |
| `0xe3` | remote `->` controller | delays information request | inferred | Reply family `0x23` |
| `0xe7` | remote `->` controller | IntelliBrite information request | inferred | Reply family `0x27` |
| `0xe8` | remote `->` controller | settings information request | inferred | Reply family `0x28` |
| `0xfc` | controller reply | software-version response | inferred from live request/reply | Reply to `0xfd` |
| `0xfd` | remote `->` controller | software-version request | inferred from live request/reply | Validated with checksum including `0xa5` |

## Custom-Name Exchange

Current working model for `0xca -> 0x0a`:
- request payload contains the 0-based custom-name index
- observed EasyTouch exposes 10 custom-name slots
- reply payload is 12 bytes wide:
  - byte `0` = returned `nameId`
  - bytes `1-11` = either fixed-width 11-byte ASCII custom name text or an encoded default-name token followed by zero bytes
- not every circuit `nameId` resolves through the custom-name path
- higher `nameId` values observed in circuit-configuration replies appear to be default-name tokens rather than custom-name slot indices

Current EasyTouch built-in default circuit-name catalog includes:
- `POOL`
- `SPA`
- `AUX 1` through `AUX 10`
- `AUX EXTRA`
- `POOL LOW`
- `POOL HIGH`
- `CLEANER`
- `FEATURE 1` through `FEATURE 8`
- `HEATER`
- `SOLAR`
- and the broader built-in name dictionary documented during reverse engineering

## Circuit-Configuration Exchange

Current working model for `0xcb -> 0x0b`:
- request payload contains the 1-based circuit index
- observed EasyTouch controller exposes 20 circuits through this family
- reply payload is 5 bytes wide

Reply payload structure:

| Payload byte | Meaning | Notes |
| --- | --- | --- |
| `0` | `circuitId` | Circuit or feature identifier |
| `1` | `functionId` | Circuit function bitfield |
| `2` | `nameId` | Name reference value |
| `3` | reserved | Observed as `0` on the current installation |
| `4` | reserved | Observed as `0` on the current installation |

Current working `functionId` bit layout:
- bits `0-5`: base function (`0-63`)
- bit `6`: freeze flag (`0x40`)
- bit `7`: high / special-behavior flag (`0x80`)

Current working base-function mapping:
- `0`: Generic / Feature circuit
- `1`: Pool
- `2`: Spa
- `3`: Cleaner
- `4`: Pool Light
- `5`: Spa Light
- `6`: Aux (generic relay)
- `7`: Feature (generic)
- `8`: Heater
- `9`: Light group
- `10`: Light group (alternate)
- `11`: Valve actuator
- `12`: Spillway
- `13`: Floor cleaner
- `14`: Solar
- `15`: Heat boost
- `16`: Pump (feature-related)
- `17`: Pump (alternate / advanced)

Function meaning rule:
- `functionId` is behavior-critical controller configuration, not merely a display label
- write paths must preserve the existing `functionId` unless the explicit goal is to change controller behavior

## Remote Status And Layout Discovery

Current clarified mapping:
- `0xe1 -> 0x21` is currently treated as the remote-status request / reply flow, not the canonical Remote Layout exchange
- broader configuration discovery remains open research

## Controller-Family Payload Snapshots

These controller-family reply payloads are action-mapped, but most field meanings remain only partially decoded:

| Action | Payload length | Sample payload | Current status |
| --- | --- | --- | --- |
| `0x08` | `13` | `47474d01920000000000000000` | heat / temperature information family |
| `0x11` | `7` | `019b0000000000` | schedule-information reply |
| `0x16` | `16` | `01a004e2000000000000000000000000` | spa-side information reply |
| `0x18` | `31` | `01800002000d040000000000000000000000011a00000000000000c2` | pump information reply |
| `0x1d` | `24` | `030000008000ffffff010102030405060708090a01ab0000` | valve-assignment reply |
| `0x1e` | `16` | `01a80000959600001300000013000000` | high-speed circuits information reply |
| `0x20` | `11` | `01aa000000000000000000` | IIS4 / IS10 information reply |
| `0x21` | `4` | `01ab0000` | remote-status reply |
| `0x23` | `2` | `0100` | delays information reply |
| `0x27` | `32` | `01b1000000000000000000000000000000000000000000000000000000000000` | IntelliBrite information reply |
| `0x28` | `10` | `00000000000000000000` | settings information reply |

Current `0xdd -> 0x1d` valve-assignment interpretation:
- payload bytes `0-3` are header-like / unknown in this parser path
- payload bytes `4..N` form a one-byte-per-valve assignment table
- normalization rule:
  - `0` = unassigned
  - `255` = unassigned
  - any other value = assigned circuit number
- shared-system special handling:
  - valve id `3` = `Intake`
  - valve id `4` = `Return`

## Pump-Slot Model

EasyTouch model notes:
- EasyTouch behaves like a small fixed pump-slot model
- single installed IntelliFlo is treated as Pump `#1`
- EasyTouch multi-pump setups appear to use Pump `#1` and Pump `#2`
- IntelliTouch likely supports broader pump inventories

### Pump Information Request / Reply

Current working request examples:
- Pump `#1`: `ff00ffa5341021d8010101e4`
- Pump `#2`: `ff00ffa5341021d8010201e5`

Current request caveat:
- the working `0xd8` request bytes do not yet fit the assumed controller-family request skeleton cleanly
- until more variants are validated, Splash should preserve the exact working bytes observed on the wire rather than overfitting a cleaner schema

### Validated `0x18` Pump `#1` Payload Model

| Byte | Meaning |
| --- | --- |
| `0` | pump slot / pump index |
| `1` | pump type |
| `2` | priming time |
| `3` | primary pump circuit selector, likely `Pool` on the observed single-pump installation |
| `4` | unknown |
| `5,7,9,11,13,15,17,19` | slot `1-8` assignment selectors |
| `6,8,10,12,14,16,18,20` | slot `1-8` RPM high bytes |
| `21` | priming speed high byte |
| `22-29` | slot `1-8` RPM low bytes |
| `30` | priming speed low byte |

Validated selector rules:
- ordinary named circuit selections use the pump-slot selector domain
- special values such as `Heater`, `Pool Heater`, `Spa Heater`, and `Freeze` are virtual trigger values, not ordinary controller circuits

Current validated special selector values:
- `128` = `Solar`
- `129` = `Heater`
- `130` = `Pool Heater`
- `131` = `Spa Heater`
- `132` = `Freeze`

Current validated ordinary selector values from the observed installation:
- `0` = `None`
- `6` = `Pool`
- `11` = `Pool low`
- `12` = `Pool high`
- `13` = `Cleaner`
- `16` = `Feature 6`

### `0x9b` Pump Configuration Write Model

Current working `0x9b` payload shape:
- byte `0`: pump id
- byte `1`: pump type
- byte `2`: priming time
- bytes `3-4`: unresolved
- bytes `5,7,9,11,13,15,17,19`: slot `1-8` assignment selectors
- bytes `6,8,10,12,14,16,18,20`: slot `1-8` speed high bytes
- byte `21`: priming speed high byte
- bytes `22-29`: slot `1-8` speed low bytes
- byte `30`: priming speed low byte

Design rule for live writes:
- the equipment/controller is the primary source of configuration truth
- Splash must request a fresh live config read before constructing a `0x9b` write
- cached or inferred config must not be treated as authoritative for live writes

Current working pump-type values for `0x9b` byte `1`:
- `0` = unknown / none
- `1` = single-speed pump
- `2` = two-speed pump
- `4` = solar / booster / special
- `8` = feature / aux pump
- `16` = VF / flow-based pump
- `32` = VS pump (older encoding)
- `128` = variable-speed pump

## Controller Status (`0x02`) Working Model

Trusted pieces:
- byte `0`: controller hour in 24-hour time
- byte `1`: controller minute

Current validated circuit-bit evidence on the observed EasyTouch 8 installation:
- payload byte `2`:
  - `0x01` = `spa`
  - `0x02` = `aux1`
  - `0x04` = `aux2`
  - `0x08` = `aux3`
  - `0x20` = `pool`
- payload byte `3`:
  - `0x04` = `pool_low`
  - `0x08` = `pool_high`
  - `0x10` = `cleaner`
  - `0x20` = `feature4`
  - `0x40` = `feature5`
  - `0x80` = `feature6`
- payload byte `4`:
  - `0x01` = `feature7`
  - `0x02` = `feature8`
  - `0x08` = `aux_extra`

Remaining status-field hypotheses such as controller mode bits, heater-state bytes, delay bytes, and mixed temperature fields should be treated as diagnostic until further validation.

## Command Notes

- commands are written when the bus is idle
- source notes reference a `50 ms` idle requirement before writing
- some pump-control flows require panel-control toggling before direct pump commands are accepted

## Normalized Mapping Targets

- `equipment.state.controller`
- `equipment.state.pump`
- `equipment.state.chlorinator`
- capability profiles such as `pentair.intelliflo_vs`, `pentair.easytouch_heater`, and generic fallbacks
- command types such as:
  - `set_speed`
  - `set_circuit_state`
  - `set_heater_setpoint`
  - `set_chlorinator_output`
