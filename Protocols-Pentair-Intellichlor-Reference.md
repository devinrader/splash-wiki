# Pentair Intellichlor Reference

This page holds the canonical Splash reference for Pentair Intellichlor chlorinator-family protocol behavior when it uses its distinct framed message family.

Use this page for:
- chlorinator-family frame structure
- chlorinator-family addressing
- chlorinator-family action catalogs
- chlorinator-family payload notes

Use the companion pages for:
- controller-family Pentair behavior: [Pentair EasyTouch / IntelliTouch Reference](Protocols-Pentair-EasyTouch-Reference)
- installation-specific facts: [Pentair Observed Installation](Protocols-Pentair-Observed-Installation)
- unstable experiment notes: [Pentair EasyTouch Reverse Engineering](Research-Pentair-EasyTouch-Reverse-Engineering)

## Status

Status: `partial`

Intellichlor traffic is observed and recognized as a distinct family.
Splash should support:

- observed-only IntelliChlor parsing by default
- optional direct control only when explicitly enabled

Payload semantics remain partially mapped, but the following action set is now
the canonical first implementation target.

## Frame Definition

Observed chlorinator-family nuance:
- Intellichlor traffic may appear as a separate framed family using `10 02 ... 10 03`
- this frame family is distinct from the Pentair controller/pump `FF 00 FF A5` frame family
- treat the chlorinator-family frame as:
  - start delimiter `10 02`
  - destination byte
  - action byte
  - payload bytes
  - checksum byte
  - end delimiter `10 03`
- Splash should validate checksum and preserve unknown frames in diagnostics
  rather than dropping them

## Source/Destination Addresses

| Hex | Decimal | Equipment | Details |
| --- | --- | --- | --- |
| `0x50` | `80` | Intellichlor / chlorinator-family destination | Common chlorinator-family destination currently referenced in Splash docs |

Addressing rule:

- Splash should treat `0x50` / `80` as the practical default destination
- Splash should not rely on unique multi-cell RS-485 addressing semantics
- direct-control mode should assume one IntelliChlor per RS-485 port

## Chlorinator-Family Action Catalog

| Hex | Decimal | Direction / Family | Current purpose | Payload structure status | Notes |
| --- | --- | --- | --- | --- | --- |
| `0x00` | `0` | controller `->` chlorinator | take-control command | supported | Payload `[0]`; expect action `1` |
| `0x01` | `1` | chlorinator `->` controller | take-control ACK | supported | ACK to action `0` |
| `0x03` | `3` | chlorinator `->` controller | model/name response | supported | Payload string such as `Intellichlor--40` |
| `0x11` | `17` | chlorinator-family | set-status / status-family behavior | inferred | Meaning differs from controller-family `0x11` |
| `0x12` | `18` | chlorinator-family | set-output reply | supported | Payload byte `0 * 50 = salt_ppm`; payload byte `1 & 0x7f = status` |
| `0x13` | `19` | chlorinator-family | keepalive / iChlor heartbeat | partial | No-payload / minimal-payload liveness evidence |
| `0x14` | `20` | controller `->` chlorinator | get-model command | supported | Payload `[0]`; expect action `3` when supported |
| `0x15` | `21` | chlorinator-family | fractional target-output command | partial | Parse inbound as `payload[0] / 10`; do not emit outbound yet |
| `0x16` | `22` | chlorinator-family | iChlor status / temp / current-output | partial | Payload byte `1 = current_output`; byte `2 = temp_f` when `>= 40` |
| `0x19` | `25` | chlorinator / controller reply | chlorinator information / status reply | partial | Also appears as the controller-family reply to `0xd9` |

## Canonical status-code mapping

Splash should use this first-slice status mapping when action `18` or other
Intellichlor-originating status bytes provide the chlorinator status code:

| Code | Meaning |
| --- | --- |
| `0` | ok |
| `1` | low_flow |
| `2` | low_salt |
| `3` | very_low_salt |
| `4` | high_current |
| `5` | clean_cell |
| `6` | low_voltage |
| `7` | low_water_temp |
| `8` | communication_lost |

## Duty-cycle generation model

Splash should treat IntelliChlor-family support as a duty-cycle generation
system, not as a guaranteed real-time production-state feed.

Documented behavior for IntelliChlor IC-family cells:

- one duty cycle is `300 seconds`
- `265 seconds` of that cycle are available for chlorine generation
- configured output percent represents the fraction of that available
  generation window used within each cycle

Modeling rules:

- `target_output_percent` may be treated as a configured duty-cycle target when
  confidently observed or commanded
- Splash should not claim an instantaneous `run_state = producing` or
  `current_output_percent` unless later captures independently validate those
  semantics for the active installation
- prediction and recommendation work should prefer estimated chlorine
  generation over time using:
  - production rate metadata
  - target output percent
  - circulation/runtime evidence
  - pool volume
  - weather and cover context

## Canonical model production metadata

Splash should treat these values as verified `lb/day at 100% output over 24h`
for IntelliChlor-family production metadata:

| Model | Production lb/day |
| --- | --- |
| `IC15` | `0.60` |
| `IC20` | `0.70` |
| `IC40` | `1.40` |
| `IC60` | `2.00` |
| `iChlor IC15` | `0.60` |
| `iChlor IC30` | `1.00` |
| `LT15` | `0.65` |
| `LT25` | `0.90` |
| `PLUS30` | `1.10` |
| `PLUS40` | `1.40` |
| `PLUS60` | `2.00` |

Rules:

- store both `lb/day` and derived `lb/sec`
- do not invent outputs for unknown future models
- if a model is unknown, runtime state may still be exposed without a
  production-rate estimate
- these production values are the canonical basis for estimated SWG chlorine
  generation modeling over time

## Notes

- chlorinator-family actions should not be mixed into the controller-family or pump-family action catalogs because the frame structure differs
- controller-family `0xd9 -> 0x19` behavior may still interact with chlorinator state, but that does not make the underlying Intellichlor frame family identical to the EasyTouch / IntelliTouch controller family
- action `21` should be parsed inbound but not emitted outbound unless later
  validation confirms its write semantics
- newer IntelliChlor-family cells may not answer action `20`; missing model
  replies should not be treated as fatal
- action `19` appears to behave like keepalive / liveness evidence, especially
  for iChlor-family traffic
- action `22` payload semantics should remain partial and should not be used as
  proof of real-time active chlorine generation until Splash captures validate
  that interpretation locally
- IntelliCenter v3 chlorinator config-slot offsets should remain marked partial
  until validated by Splash captures
