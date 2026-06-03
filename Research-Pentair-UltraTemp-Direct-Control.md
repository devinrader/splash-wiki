# Pentair UltraTemp Direct Control Research

[Back to README](Home)

## Scope

This document covers reverse-engineering notes for direct UltraTemp protocol control. It is research-only and does not authorize production command transmission.

## Known direct protocol details

Protocol family:
- `Heater`

Typical addressing:
- source: `16`
- destination: `112`

Known direct-control command:
- action: `114`
- expected response: `115`
- payload length: `10` bytes

Known outbound payload shape:

| Byte | Meaning |
| --- | --- |
| `0` | `144` |
| `1` | mode: `0 = off`, `1 = heat`, `2 = cool` |
| `2-9` | unknown / reserved |

Known inbound response shape:

| Byte | Meaning |
| --- | --- |
| `0` | `160` |
| `1` | observed value `1` |
| `2` | current mode: `0 = off`, `1 = heat`, `2 = cool` |
| `3` | believed temperature- or offset-related |
| `4-9` | unknown |

## Safety status

- protocol behavior is reverse-engineered, not vendor-documented
- field meanings are incomplete
- heartbeat and timing expectations are not yet understood
- compressor safety timing behavior is not yet understood
- EasyTouch coordination behavior is not yet understood
- packet captures are required before any production use

Direct heater control bypasses portions of Pentair EasyTouch automation and may affect compressor protection, heating/cooling arbitration, and freeze safety behavior. This functionality is experimental and should not be enabled on production pool systems until fully validated with packet captures and real-world testing.

## Research TODOs

- capture EasyTouch to UltraTemp startup negotiation
- capture periodic heartbeat traffic
- identify unknown payload bytes
- identify compressor protection timing behavior
- identify cooling-mode negotiation
- identify fault or error frames
- identify setpoint synchronization frames
- determine whether checksums or rolling state exist beyond the currently observed frame structure
- verify whether EasyTouch mirrors, proxies, or transforms direct heater state

## Future implementation notes

If Splash ever supports direct heater ownership, it will likely need:
- explicit opt-in
- a clear mode switch away from EasyTouch-owned control
- command arbitration
- anti-short-cycle protection
- compressor lockout timing
- pump and heater coordination
- freeze protection ownership
- state reconciliation
- heartbeat or status polling
- failure recovery behavior
