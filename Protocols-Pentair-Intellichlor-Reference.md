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

Intellichlor traffic is observed and recognized as a distinct family, but its payload semantics remain only partially mapped.

## Frame Definition

Observed chlorinator-family nuance:
- Intellichlor traffic may appear as a separate framed family using `10 02 ... 10 03`
- this frame family is distinct from the Pentair controller/pump `FF 00 FF A5` frame family

## Source/Destination Addresses

| Hex | Decimal | Equipment | Details |
| --- | --- | --- | --- |
| `0x50` | `80` | Intellichlor / chlorinator-family destination | Common chlorinator-family destination currently referenced in Splash docs |

## Chlorinator-Family Action Catalog

| Hex | Decimal | Direction / Family | Current purpose | Payload structure status | Notes |
| --- | --- | --- | --- | --- | --- |
| `0x11` | `17` | chlorinator-family | set-status / status-family behavior | inferred | Meaning differs from controller-family `0x11` |
| `0x12` | `18` | chlorinator-family | chlorinator action | inferred | Intellichlor-framed traffic |
| `0x19` | `25` | chlorinator / controller reply | chlorinator information / status reply | partial | Also appears as the controller-family reply to `0xd9` |

## Notes

- chlorinator-family actions should not be mixed into the controller-family or pump-family action catalogs because the frame structure differs
- controller-family `0xd9 -> 0x19` behavior may still interact with chlorinator state, but that does not make the underlying Intellichlor frame family identical to the EasyTouch / IntelliTouch controller family
