# Pentair EasyTouch / IntelliTouch Reference

This page is the canonical Splash reference for the Pentair EasyTouch / IntelliTouch RS-485 protocol family.

Use this page for:
- frame structure
- checksum behavior
- address and action catalogs
- validated payload layouts
- stable value mappings

Use the companion pages for:
- installation-specific facts: [Pentair Observed Installation](Protocols-Pentair-Observed-Installation)
- controller menu hierarchy: [Pentair EasyTouch 8 Menu Structure](Protocols-Pentair-EasyTouch-Menu-Structure)
- experiment history and unresolved hypotheses: [Pentair EasyTouch Reverse Engineering](Research-Pentair-EasyTouch-Reverse-Engineering)

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

## Source/Destination Addresses

| Hex | Decimal | Equipment | Details |
| --- | --- | --- | --- |
| `0x10` | `16` | Equipment Controllers | SunTouch, IntelliTouch, EasyTouch, etc |
| `0x0f` | `15` | Broadcast | Panel / common controller-originated destination |
| `0x48` | `72` | Basic Remote Transceiver | QuickTouch |
| `0x60-0x6f` | `96-111` | Pumps | IntelliFlo / IntelliPro. In a multi-pump system, pumps are `0x60`, `0x61`, `0x62`, `0x64`, `0x65`, `0x66`, `0x67`, `0x68`, `0x69`, `0x6a`, `0x6b`, `0x6c`, `0x6d`, `0x6e`, `0x6f`. |
| `0x22` | `34` | Fancy Remote Transceiver | ScreenLogic |
| `0x50` | `80` | UltraTemp Heat Pump | Appears to possibly be in the low `0x50` range |
| `0xc8` | `200` | Broadcast | All interested recipients |

## Protocol/Header Values

| Hex | Decimal | Field | Meaning / Notes | Confidence |
| --- | --- | --- | --- | --- |
| `0x00` | `0` | protocol/header byte | Pump-family exchange header in `FF 00 FF A5` frames | strong |
| `0x01` | `1` | protocol/header byte | Alternate valid protocol/header byte in `FF 00 FF A5` frames; used by some controller-originated requests such as remote-status flows | inferred |
| `0x34` | `52` | controller-family header byte | Common controller request / reply family | strong |

## Action Catalogs

### Controller-Family Action Catalog

| Hex | Decimal | Direction / Family | Current purpose | Payload structure status | Notes |
| --- | --- | --- | --- | --- | --- |
| `0x01` | `1` | controller reply, header `0x34` | ACK / request accepted | inferred | Generic controller ACK in controller-family traffic |
| `0x02` | `2` | controller broadcast | controller status | partial but strongest current mapping | Primary live state source |
| `0x05` | `5` | controller-originated | unknown | unknown | Observed during brute-force sweep |
| `0x08` | `8` | controller-originated, header `0x34` | heat / temperature information broadcast | inferred | Field mapping partial |
| `0x0a` | `10` | controller broadcast / reply | custom-name value response | inferred from live request/reply | Reply to `0xca` |
| `0x11` | `17` | controller-family | schedule-information response | inferred | Reply to `0xd1` in controller-family traffic |
| `0x16` | `22` | controller broadcast / reply | spa-side information reply | inferred | Reply to `0xd6` |
| `0x18` | `24` | controller broadcast / reply | pump information reply | inferred from live request/reply | Reply to `0xd8` |
| `0x1d` | `29` | controller broadcast / reply | valve assignment reply | inferred | Reply to `0xdd` |
| `0x1e` | `30` | controller broadcast / reply | high-speed circuits information reply | inferred | Reply to `0xde` |
| `0x20` | `32` | controller broadcast / reply | IIS4 / IS10 information reply | inferred | Reply to `0xe0` |
| `0x21` | `33` | controller broadcast / reply | remote-status reply | inferred | Reply to `0xe1` |
| `0x23` | `35` | controller broadcast / reply | delays information reply | inferred | Reply to `0xe3` |
| `0x27` | `39` | controller broadcast / reply | IntelliBrite information reply | inferred | Reply to `0xe7` |
| `0x28` | `40` | controller broadcast / reply | settings information reply | inferred | Reply to `0xe8` |
| `0x86` | `134` | remote `->` controller | set circuit state | inferred | Payload appears to be `[circuitNumber, state]` |
| `0x9b` | `155` | remote `->` controller | set pump configuration | inferred | Pump-config write family |
| `0xc8` | `200` | remote `->` controller | heat / temperature information request | inferred | Reply family `0x08` |
| `0xca` | `202` | remote `->` controller | custom-name request | inferred from live request/reply | 0-based custom-name index |
| `0xcb` | `203` | remote `->` controller | circuit-configuration request | inferred from live request/reply | 1-based circuit index |
| `0xd1` | `209` | remote `->` controller | schedule-information request | inferred | Reply family `0x11` |
| `0xd6` | `214` | remote `->` controller | spa-side information request | inferred | Reply family `0x16` |
| `0xd8` | `216` | remote `->` controller | pump information request | inferred from live request/reply | Reply family `0x18` |
| `0xd9` | `217` | remote `->` controller | chlorinator information request | inferred | Reply family `0x19` controller-family response |
| `0xdd` | `221` | remote `->` controller | get valve assignments | inferred | Reply family `0x1d` |
| `0xde` | `222` | remote `->` controller | high-speed circuits information request | inferred | Reply family `0x1e` |
| `0xe0` | `224` | remote `->` controller | IIS4 / IS10 information request | inferred | Reply family `0x20` |
| `0xe1` | `225` | remote `->` controller | remote-status request | inferred | Reply family `0x21` |
| `0xe2` | `226` | remote `->` controller | solar / heat-pump information request | inferred | Reply family unresolved |
| `0xe3` | `227` | remote `->` controller | delays information request | inferred | Reply family `0x23` |
| `0xe7` | `231` | remote `->` controller | IntelliBrite information request | inferred | Reply family `0x27` |
| `0xe8` | `232` | remote `->` controller | settings information request | inferred | Reply family `0x28` |
| `0xfc` | `252` | controller reply | software-version response | inferred from live request/reply | Reply to `0xfd` |
| `0xfd` | `253` | remote `->` controller | software-version request | inferred from live request/reply | Validated with checksum including `0xa5` |

### Pump-Family Action Catalog

| Hex | Decimal | Direction / Family | Current purpose | Payload structure status | Notes |
| --- | --- | --- | --- | --- | --- |
| `0x01` | `1` | controller `<->` pump, header `0x00` | pump speed / flow exchange | partial | Pump-family only |
| `0x04` | `4` | controller `<->` pump, header `0x00` | pump-state exchange | partial | Pump-family only |
| `0x06` | `6` | controller `<->` pump, header `0x00` | pump drive-state exchange | partial | Pump-family only |
| `0x07` | `7` | controller `<->` pump, header `0x00` | pump status poll / reply family | partial | Pump-family only |

Chlorinator-family traffic uses a different frame structure and is documented separately in [Pentair Intellichlor Reference](Protocols-Pentair-Intellichlor-Reference).

## Controller Status (`0x02` / `2`)

Current role:
- primary live controller-state broadcast
- strongest current action mapping on the observed EasyTouch installation
- main source for normalized controller state such as temperatures, active circuits, and controller mode clues

Current trusted fields:
- time-of-day bytes
- circuit-state bytes
- water, air, and solar temperature bytes
- controller mode and freeze-protection clues
- firmware major/minor bytes

Current mapping rule:
- Splash should normalize only the trusted subset of `0x02` fields
- unresolved bytes should remain available in decoded payload output rather than being guessed

### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | Controller hour in 24-hour time | strong |
| `1` | `1` | Controller minute | strong |
| `2` | `1` | System-circuit enabled-state bitmask. On the observed EasyTouch 8 installation this byte covers `SPA`, `AUX 1`, `AUX 2`, `AUX 3`, and `POOL`. A set bit means the corresponding system circuit is enabled/on. See [EasyTouch Circuit Id And Default Name Mapping](#easytouch-circuit-id-and-default-name-mapping). | strong |
| `3` | `1` | System-circuit enabled-state bitmask. On the observed EasyTouch 8 installation this byte covers `FEATURE 1`, `FEATURE 2`, `FEATURE 3`, `FEATURE 4`, `FEATURE 5`, and `FEATURE 6`. A set bit means the corresponding system circuit is enabled/on. See [EasyTouch Circuit Id And Default Name Mapping](#easytouch-circuit-id-and-default-name-mapping). | strong |
| `4` | `1` | System-circuit enabled-state bitmask. On the observed EasyTouch 8 installation this byte includes `FEATURE 7`, `FEATURE 8`, and `AUX EXTRA`. On more feature-rich controllers it may continue the system-circuit bitmask space. A set bit means the corresponding system circuit is enabled/on. See [EasyTouch Circuit Id And Default Name Mapping](#easytouch-circuit-id-and-default-name-mapping). | strong |
| `5` | `1` | Additional system-circuit enabled-state bitmask on more feature-rich controllers. A set bit means the corresponding system circuit is enabled/on. Current per-mask meanings are not yet validated on the observed EasyTouch installation. | unknown |
| `6` | `1` | Additional circuit-state bitmask on more feature-rich controllers. Current per-mask meanings are not yet validated on the observed EasyTouch installation. | unknown |
| `7` | `1` | Additional circuit-state bitmask on more feature-rich controllers. Current per-mask meanings are not yet validated on the observed EasyTouch installation. | unknown |
| `8` | `1` | Additional circuit-state bitmask on more feature-rich controllers. Current per-mask meanings are not yet validated on the observed EasyTouch installation. | unknown |
| `9` | `1` | Controller mode/status bitmask. See [Controller Mode/Status Bit Meanings](#controller-modestatus-bit-meanings). | inferred |
| `10` | `1` | Valve-state bitmask | guess |
| `11` | `1` | Unknown | unknown |
| `12` | `1` | Delay-status byte | inferred |
| `13` | `1` | Unknown | unknown |
| `14` | `1` | Water temperature. If byte `9` bit `0x04` is clear, interpret as Fahrenheit. If byte `9` bit `0x04` is set, interpret as Celsius and convert to Fahrenheit only in derived views that require Fahrenheit output. See [Controller Mode/Status Bit Meanings](#controller-modestatus-bit-meanings). | inferred |
| `15` | `1` | Heater-state byte | guess |
| `16` | `1` | Firmware major version | inferred |
| `17` | `1` | Firmware minor version | inferred |
| `18` | `1` | Air temperature. If byte `9` bit `0x04` is clear, interpret as Fahrenheit. If byte `9` bit `0x04` is set, interpret as Celsius and convert to Fahrenheit only in derived views that require Fahrenheit output. See [Controller Mode/Status Bit Meanings](#controller-modestatus-bit-meanings). | inferred |
| `19` | `1` | Solar temperature. If byte `9` bit `0x04` is clear, interpret as Fahrenheit. If byte `9` bit `0x04` is set, interpret as Celsius and convert to Fahrenheit only in derived views that require Fahrenheit output. See [Controller Mode/Status Bit Meanings](#controller-modestatus-bit-meanings). | inferred |
| `20` | `1` | Unknown | unknown |
| `21` | `1` | Unknown | unknown |
| `22` | `1` | Heat-setting / heat-mode bitmask. See [Heat-Mode Bit Meanings](#heat-mode-bit-meanings). | inferred |
| `23` | `1` | Firmware-dependent value. Observed as `0x00` on firmware `1.0` and `0x10` on firmware `2.070`. | inferred |
| `24` | `1` | Observed as zero in current captures. | inferred |
| `25` | `1` | Appears related to heater setpoint or derived heat-setting state, possibly in combination with byte `26`. Current interpretation is unresolved. | guess |
| `26` | `1` | Appears related to heater setpoint. Current interpretation is more stable than byte `25`, but the exact rule remains unresolved. | guess |
| `27` | `1` | Observed values include `0x19` and `0x38`. Purpose unresolved. | guess |
| `28` | `1` | Observed values include `0x0a` and `0x0b` on firmware `2.070`. Purpose unresolved. | guess |

Remaining status-field hypotheses such as controller mode bits, heater-state bytes, delay bytes, and mixed temperature fields should be treated as diagnostic until further validation.

Derived-value rule:
- when a field's interpretation depends on another field, keep the raw byte value as the protocol truth and derive normalized values from the controlling field
- for temperature bytes, byte `9` bit `0x04` determines whether the raw values should be interpreted as Fahrenheit or Celsius

## Get/Set Actions

### Custom Names

#### Get Custom Name (`0xca` / `202`)

Request notes:
- request payload contains the 0-based custom-name index
- observed EasyTouch exposes 10 custom-name slots

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | 0-based custom-name slot index | inferred from live request/reply |

#### Set Custom Name (`0x0a` / `10`)

Reply notes:
- not every circuit-name reference value (`nameId`) resolves through the custom-name path
- higher `nameId` values observed in circuit-configuration replies appear to be built-in circuit-name tokens rather than custom-name slot indices

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | Returned custom-name slot index | strong |
| `1-11` | `11` | Fixed-width ASCII custom-name text | strong |

### Circuits

#### Get Circuit  (`0xcb` / `203`)

Request notes:
- request payload contains the 1-based circuit index
- observed EasyTouch controller exposes 20 circuits through this family

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | 1-based circuit index | inferred from live request/reply |

#### Set Circuit (`0x0b` / `11`)

Reply notes:
- reply payload is 5 bytes wide

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | `circuitId` circuit identifier for the returned circuit record | inferred |
| `1` | `1` | `functionId` circuit-function bitmask. See [Circuit-Configuration Function Bit Meanings](#circuit-configuration-function-bit-meanings). | inferred |
| `2` | `1` | `nameId` assigned-name id for the circuit assigned name. May point to a built-in assigned-name token or a custom-name-related reference. | inferred |
| `3` | `1` | Reserved / unknown. Observed as `0` on the current installation. | inferred |
| `4` | `1` | Reserved / unknown. Observed as `0` on the current installation. | inferred |

#### Circuit-Configuration Function Bit Meanings

Current working `functionId` bit layout:
- bits `0-5`: base function (`0-63`)
- bit `6`: freeze flag (`0x40`)
- bit `7`: high / special-behavior flag (`0x80`)

#### Circuit-Configuration Base Function Values

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

### Heat / Temperature

#### Get Heat / Temperature (`0xc8` / `200`)

Request notes:
- request action is observed and paired with `0x08`
- request payload shape is not yet validated

#### Set Heat / Temperature  (`0x08` / `8`)

Reply notes:
- payload length: `13`
- sample payload: `47474d01920000000000000000`
- status: heat / temperature information family

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-12` | `13` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Schedule

#### Get Schedule (`0xd1` / `209`)

Request notes:
- request action is observed and paired with `0x11`
- request payload shape is not yet validated

#### Set Schedule  (`0x11` / `17`)

Reply notes:
- payload length: `7`
- sample payload: `019b0000000000`
- status: schedule-information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-6` | `7` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Spa-Side

#### Get Spa-Side (`0xd6` / `214`)

Request notes:
- request action is observed and paired with `0x16`
- request payload shape is not yet validated

#### Set Spa-Side  (`0x16` / `22`)

Reply notes:
- payload length: `16`
- sample payload: `01a004e2000000000000000000000000`
- status: spa-side information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-15` | `16` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Valves

#### Get Valve (`0xdd` / `221`)

Request notes:
- request action is observed and paired with `0x1d`
- request payload shape is not yet validated

#### Set Valve (`0x1d` / `29`)

Reply notes:
- payload length: `24`
- sample payload: `030000008000ffffff010102030405060708090a01ab0000`
- status: valve-assignment reply
- payload bytes `0-3` are header-like / unknown in this parser path
- payload bytes `4..N` form a one-byte-per-valve assignment table
- normalization rule:
  - `0` = unassigned
  - `255` = unassigned
  - any other value = assigned circuit number
- shared-system special handling:
  - valve id `3` = `Intake`
  - valve id `4` = `Return`

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-3` | `4` | Header-like or unresolved bytes in the current parser path | guess |
| `4-23` | `20` | One-byte-per-valve assignment table. `0` and `255` mean unassigned; other values indicate assigned circuit id/number. Valve ids `3` and `4` receive shared-system special handling as `Intake` and `Return`. | inferred |

### High-Speed Circuits

#### Get High-Speed Circuits (`0xde` / `222`)

Request notes:
- request action is observed and paired with `0x1e`
- request payload shape is not yet validated

#### Set High-Speed Circuits (`0x1e` / `30`)

Reply notes:
- payload length: `16`
- sample payload: `01a80000959600001300000013000000`
- status: high-speed circuits information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-15` | `16` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### IIS4 / IS10

#### Get IIS4 / IS10 (`0xe0` / `224`)

Request notes:
- request action is observed and paired with `0x20`
- request payload shape is not yet validated

#### Set IIS4 / IS10 (`0x20` / `32`)

Reply notes:
- payload length: `11`
- sample payload: `01aa000000000000000000`
- status: IIS4 / IS10 information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-10` | `11` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Remote Status

#### Get Remote Status (`0xe1` / `225`)

Request notes:
- `0xe1 -> 0x21` is currently treated as the remote-status request / reply flow, not the canonical Remote Layout exchange
- broader configuration discovery remains open research

#### Set Remote Status (`0x21` / `33`)

Reply notes:
- payload length: `4`
- sample payload: `01ab0000`
- status: remote-status reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-3` | `4` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Pumps

EasyTouch model notes:
- EasyTouch behaves like a small fixed pump-slot model
- single installed IntelliFlo is treated as Pump `#1`
- EasyTouch multi-pump setups appear to use Pump `#1` and Pump `#2`
- IntelliTouch likely supports broader pump inventories

#### Get Pump (`0xd8` / `216`)

Request notes:
- Pump `#1`: `ff00ffa5341021d8010101e4`
- Pump `#2`: `ff00ffa5341021d8010201e5`
- the working `0xd8` request bytes do not yet fit the assumed controller-family request skeleton cleanly
- until more variants are validated, Splash should preserve the exact working bytes observed on the wire rather than overfitting a cleaner schema

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | Pump slot selector. Observed supported values are `1` for Pump `#1` and `2` for Pump `#2`. This is a pump-slot selector, not a circuit index or circuit id. | inferred from live request/reply |
| `1` | `1` | Additional request byte present in validated live captures. Preserve the observed working value instead of assuming a cleaner request schema. | inferred from live request/reply |

#### Set Pump (`0x18` / `24`)

Reply notes:
- payload length: `31`
- sample payload: `01800002000d040000000000000000000000011a00000000000000c2`
- status: pump information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0` | `1` | Pump slot / pump index | strong |
| `1` | `1` | Pump type. See [Pump-Type Values](#pump-type-values). | inferred |
| `2` | `1` | Priming time | inferred |
| `3` | `1` | Primary pump circuit selector, likely `Pool` on the observed single-pump installation. See [Pump Circuit Selector Values](#pump-circuit-selector-values). This is a controller-specific circuit-selector domain, not a circuit index, circuit id, or circuit-name token. | inferred |
| `4` | `1` | Unknown | unknown |
| `5` | `1` | Slot `1` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `6` | `1` | Slot `1` RPM high byte. Combine with byte `22` to derive RPM. | inferred |
| `7` | `1` | Slot `2` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `8` | `1` | Slot `2` RPM high byte. Combine with byte `23` to derive RPM. | inferred |
| `9` | `1` | Slot `3` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `10` | `1` | Slot `3` RPM high byte. Combine with byte `24` to derive RPM. | inferred |
| `11` | `1` | Slot `4` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `12` | `1` | Slot `4` RPM high byte. Combine with byte `25` to derive RPM. | inferred |
| `13` | `1` | Slot `5` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `14` | `1` | Slot `5` RPM high byte. Combine with byte `26` to derive RPM. | inferred |
| `15` | `1` | Slot `6` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `16` | `1` | Slot `6` RPM high byte. Combine with byte `27` to derive RPM. | inferred |
| `17` | `1` | Slot `7` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `18` | `1` | Slot `7` RPM high byte. Combine with byte `28` to derive RPM. | inferred |
| `19` | `1` | Slot `8` circuit selector. See [Pump Circuit Selector Values](#pump-circuit-selector-values). | inferred |
| `20` | `1` | Slot `8` RPM high byte. Combine with byte `29` to derive RPM. | inferred |
| `21` | `1` | Priming speed high byte. Combine with byte `30` to derive RPM. | inferred |
| `22` | `1` | Slot `1` RPM low byte. Combine with byte `6` to derive RPM. | inferred |
| `23` | `1` | Slot `2` RPM low byte. Combine with byte `8` to derive RPM. | inferred |
| `24` | `1` | Slot `3` RPM low byte. Combine with byte `10` to derive RPM. | inferred |
| `25` | `1` | Slot `4` RPM low byte. Combine with byte `12` to derive RPM. | inferred |
| `26` | `1` | Slot `5` RPM low byte. Combine with byte `14` to derive RPM. | inferred |
| `27` | `1` | Slot `6` RPM low byte. Combine with byte `16` to derive RPM. | inferred |
| `28` | `1` | Slot `7` RPM low byte. Combine with byte `18` to derive RPM. | inferred |
| `29` | `1` | Slot `8` RPM low byte. Combine with byte `20` to derive RPM. | inferred |
| `30` | `1` | Priming speed low byte. Combine with byte `21` to derive RPM. | inferred |

Validated selector rules:
- ordinary named circuit selections use the pump-slot circuit-selector domain
- this selector domain is distinct from circuit index, circuit id, and circuit-name reference values
- special values such as `Heater`, `Pool Heater`, `Spa Heater`, and `Freeze` are virtual trigger values, not ordinary controller circuits

### Delays (`0xe3 -> 0x23`)

#### Get Delays (`0xe3` / `227`)

Request notes:
- request action is observed and paired with `0x23`
- request payload shape is not yet validated

#### Set Delays (`0x23` / `35`)

Reply notes:
- payload length: `2`
- sample payload: `0100`
- status: delays information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-1` | `2` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### IntelliBrite Information (`0xe7 -> 0x27`)

#### Get IntelliBrite Information (`0xe7` / `231`)

Request notes:
- request action is observed and paired with `0x27`
- request payload shape is not yet validated

#### Set IntelliBrite Information (`0x27` / `39`)

Reply notes:
- payload length: `32`
- sample payload: `01b1000000000000000000000000000000000000000000000000000000000000`
- status: IntelliBrite information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-31` | `32` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

### Settings (`0xe8 -> 0x28`)

#### Get Settings (`0xe8` / `232`)

Request notes:
- request action is observed and paired with `0x28`
- request payload shape is not yet validated

#### Set Settings (`0x28` / `40`)

Reply notes:
- payload length: `10`
- sample payload: `00000000000000000000`
- status: settings information reply

##### Frame Payload

| Field byte(s) | Length | Description | Confidence |
| --- | --- | --- | --- |
| `0-9` | `10` | Reply payload observed and action-mapped, but per-byte field meanings remain only partially decoded. Keep the raw payload as protocol truth until byte-level mapping is validated. | guess |

#### Pump Circuit Selector Values

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
- bytes `5,7,9,11,13,15,17,19`: slot `1-8` circuit selectors in the pump-slot selector domain
- bytes `6,8,10,12,14,16,18,20`: slot `1-8` speed high bytes
- byte `21`: priming speed high byte
- bytes `22-29`: slot `1-8` speed low bytes
- byte `30`: priming speed low byte

Design rule for live writes:
- the equipment/controller is the primary source of configuration truth
- Splash must request a fresh live config read before constructing a `0x9b` write
- cached or inferred config must not be treated as authoritative for live writes

#### Pump-Type Values

Current working pump-type values for `0x9b` byte `1` and related `0x18` reply interpretation:
- `0` = unknown / none
- `1` = single-speed pump
- `2` = two-speed pump
- `4` = solar / booster / special
- `8` = feature / aux pump
- `16` = VF / flow-based pump
- `32` = VS pump (older encoding)
- `128` = variable-speed pump

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

## EasyTouch 8 System Circuit Inventory

| Circuit type | Count | Circuit indexes | Default labels / notes |
| --- | --- | --- | --- |
| fixed | `2` | `1-2` | `POOL`, `SPA` |
| relay | `8` | `3-9`, `18` | `AUX 1-7`, `AUX EXTRA` |
| feature | `8` | `10-17` | `FEATURE 1-8` |
| total | `18` | `1-18` | Full EasyTouch 8 circuit inventory |

## EasyTouch Circuit Id And Default System Name Mapping

| Circuit id | Circuit type | Circuit type name | Notes |
| --- | --- | --- | --- |
| `1` | fixed | `SPA` | Fixed circuit type name |
| `2` | fixed | `POOL` | Fixed circuit type name |
| `3` | relay | `AUX 1` | Relay circuit type name |
| `4` | relay | `AUX 2` | Relay circuit type name |
| `5` | relay | `AUX 3` | Relay circuit type name |
| `6` | relay | `AUX 4` | Relay circuit type name |
| `7` | relay | `AUX 5` | Relay circuit type name |
| `8` | relay | `AUX 6` | Relay circuit type name |
| `9` | relay | `AUX 7` | Relay circuit type name |
| `10` | feature | `FEATURE 1` | Feature circuit type name |
| `11` | feature | `FEATURE 2` | Feature circuit type name |
| `12` | feature | `FEATURE 3` | Feature circuit type name |
| `13` | feature | `FEATURE 4` | Feature circuit type name |
| `14` | feature | `FEATURE 5` | Feature circuit type name |
| `15` | feature | `FEATURE 6` | Feature circuit type name |
| `16` | feature | `FEATURE 7` | Feature circuit type name |
| `17` | feature | `FEATURE 8` | Feature circuit type name |
| `18` | relay | `AUX EXTRA` | Relay circuit type name |

## Controller Mode/Status Bit Meanings

| Field byte | Bit / Mask | Meaning | Confidence |
| --- | --- | --- | --- |
| `9` | `0x01` | Run-mode indicator. Covers both `Normal` and `Service` controller run modes. | inferred |
| `9` | `0x04` | Temperature-unit indicator (`F` or `C`). | inferred |
| `9` | `0x08` | Freeze-protection indicator (`on` / `off`). | inferred |
| `9` | `0x80` | Timeout-mode indicator. | inferred |

## Circuit-State Bitmask Values

| Field byte | Mask value | Circuit name | Circuit id | Confidence |
| --- | --- | --- | --- | --- |
| `2` | `0x01` | `SPA` | `1` | strong |
| `2` | `0x02` | `AUX 1` | `3` | strong |
| `2` | `0x04` | `AUX 2` | `4` | strong |
| `2` | `0x08` | `AUX 3` | `5` | strong |
| `2` | `0x20` | `POOL` | `2` | strong |
| `3` | `0x04` | `POOL LOW` | unknown | strong |
| `3` | `0x08` | `POOL HIGH` | unknown | strong |
| `3` | `0x10` | `CLEANER` | unknown | strong |
| `3` | `0x20` | `FEATURE 4` | `13` | strong |
| `3` | `0x40` | `FEATURE 5` | `14` | strong |
| `3` | `0x80` | `FEATURE 6` | `15` | strong |
| `4` | `0x01` | `FEATURE 7` | `16` | strong |
| `4` | `0x02` | `FEATURE 8` | `17` | strong |
| `4` | `0x08` | `AUX EXTRA` | `18` | strong |

## Heat-Mode Bit Meanings

| Field byte | Bit / Mask | Meaning | Confidence |
| --- | --- | --- | --- |
| `22` | `0x03` | Pool heat mode low bits: `0` = off, `1` = heater, `2` = solar preferred, `3` = solar | inferred |
| `22` | `0x0c` | Spa heat mode high bits: `0x00` = off, `0x04` = heater, `0x08` = solar preferred, `0x0c` = solar | inferred |

## Available Circuit Assigned Names

EasyTouch circuit assigned names may be selected from:
- the built-in assigned-name values below
- `Custom Name 1-10`

The table below lists the built-in assigned-name values.

Where known from live validation, the assigned-name integer representation is shown in parentheses.

<table>
  <tr><td><code>AERATOR</code></td><td><code>DRIVE LIGHT</code></td><td><code>HI-TEMP</code></td><td><code>POOL (3)</code></td></tr>
  <tr><td><code>AIR BLOWER</code></td><td><code>EDGE PUMP</code></td><td><code>HIGH SPEED</code></td><td><code>SECURITY LT</code></td></tr>
  <tr><td><code>AUX 10</code></td><td><code>ENTRY LIGHT</code></td><td><code>HOUSE LIGHT</code></td><td><code>SLIDE</code></td></tr>
  <tr><td><code>AUX 1</code></td><td><code>FAN</code></td><td><code>JETS</code></td><td><code>SOLAR</code></td></tr>
  <tr><td><code>AUX 2</code></td><td><code>FEATURE 1</code></td><td><code>LIGHTS (5)</code></td><td><code>SPA HIGH</code></td></tr>
  <tr><td><code>AUX 3</code></td><td><code>FEATURE 2</code></td><td><code>LO-TEMP</code></td><td><code>SPA LIGHT</code></td></tr>
  <tr><td><code>AUX 4</code></td><td><code>FEATURE 3</code></td><td><code>LOW SPEED</code></td><td><code>SPA LOW</code></td></tr>
  <tr><td><code>AUX 5</code></td><td><code>FEATURE 4</code></td><td><code>MALIBU LTS</code></td><td><code>SPA SAL</code></td></tr>
  <tr><td><code>AUX 6</code></td><td><code>FEATURE 5</code></td><td><code>MIST</code></td><td><code>SPA SAM</code></td></tr>
  <tr><td><code>AUX 7</code></td><td><code>FEATURE 6</code></td><td><code>MUSIC</code></td><td><code>SPA WTRFLL</code></td></tr>
  <tr><td><code>AUX 8</code></td><td><code>FEATURE 7</code></td><td><code>NOT USED</code></td><td><code>SPA (148)</code></td></tr>
  <tr><td><code>AUX 9</code></td><td><code>FEATURE 8</code></td><td><code>OZONATOR</code></td><td><code>SPILLWAY</code></td></tr>
  <tr><td><code>AUX EXTRA</code></td><td><code>FIBER OPTIC</code></td><td><code>PATH LIGHTS</code></td><td><code>SPRINKLERS</code></td></tr>
  <tr><td><code>BACK LIGHT</code></td><td><code>FIBER WORKS</code></td><td><code>PATIO LTS</code></td><td><code>STATUE LT</code></td></tr>
  <tr><td><code>BACKWASH</code></td><td><code>FILL LINE</code></td><td><code>PERIMETER L</code></td><td><code>STREAM</code></td></tr>
  <tr><td><code>BBQ LIGHT</code></td><td><code>FLOOR CLNR</code></td><td><code>PG2000</code></td><td><code>SWIM JETS</code></td></tr>
  <tr><td><code>BEACH LIGHT</code></td><td><code>FOGGER</code></td><td><code>POND LIGHT</code></td><td><code>WATERFALL 1</code></td></tr>
  <tr><td><code>BOOSTER PUMP</code></td><td><code>FOUNTAIN 1</code></td><td><code>POOL HIGH (64)</code></td><td><code>WATERFALL 2</code></td></tr>
  <tr><td><code>BUG LIGHT</code></td><td><code>FOUNTAIN 2</code></td><td><code>POOL LIGHT</code></td><td><code>WATERFALL 3</code></td></tr>
  <tr><td><code>CABANA LTS</code></td><td><code>FOUNTAIN 3</code></td><td><code>POOL LOW (0)</code></td><td><code>WATERFALL</code></td></tr>
  <tr><td><code>CHEM. FEEDER</code></td><td><code>FOUNTAINS</code></td><td><code>POOL PUMP</code></td><td><code>WHIRLPOOL</code></td></tr>
  <tr><td><code>CHLORINATOR</code></td><td><code>FOUNTAIN</code></td><td><code>POOL SAM 1</code></td><td><code>WTR FEAT LT</code></td></tr>
  <tr><td><code>CLEANER (62)</code></td><td><code>FRONT LIGHT</code></td><td><code>POOL SAM 2</code></td><td><code>WTR FEATURE</code></td></tr>
  <tr><td><code>COLOR WHEEL</code></td><td><code>GARDEN LTS</code></td><td><code>POOL SAM 3</code></td><td><code>WTRFL LGHT</code></td></tr>
  <tr><td><code>DECK LIGHT</code></td><td><code>GAZEBO LTS</code></td><td><code>POOL SAM</code></td><td><code>YARD LIGHT</code></td></tr>
  <tr><td><code>DRAIN LINE</code></td><td></td><td></td><td></td></tr>
</table>

Source note:
- official Pentair EasyTouch Control System User's Guide, "EasyTouch Circuit Names" list
