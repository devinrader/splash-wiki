# Command Flow

[Back to Splash Protocol Service](Services-Splash-API-Home)

## Purpose

This document defines how `splash-protocol` handles outbound command intent and
result correlation.

## Input contract

`splash-protocol` consumes `protocol.command.intent`.

It must:

- validate that an active plugin exists for the pool
- validate that plugin configuration supports the command
- encode the command into protocol bytes
- publish `protocol.command.encoded`
- publish `serial.write.request`
- track command state until transmit confirmation and completion or timeout

## Initial supported live command

The first live command supported by `splash-protocol` is:

- Pentair controller-managed pump-circuit `set_speed`

The first Explorer-only diagnostic command should be:

- Pentair manual Remote Layout page request
- Pentair manual raw frame send
- Pentair manual pump-info request
- Pentair manual pump-config write
- Pentair controller circuit-configuration discovery request
- Pentair controller date/time request
- Pentair controller date/time sync

Initial scope rules:

- target equipment type: `pump`
- protocol plugin: `pentair_easytouch`
- target path: controller-managed EasyTouch pump circuit
- command family: normalized `set_speed`

ASSUMPTION: the initial `set_speed` implementation targets the EasyTouch
controller-managed circuit that owns a stored pump-speed slot rather than a
direct IntelliFlo pump address.

## Initial Pentair controller-mediated write sequence

For the first supported live `set_speed` path, `splash-protocol` should:

1. resolve the controller-managed circuit that owns the target pump speed
2. request a fresh live controller pump-configuration read for the target pump
   slot before any write is encoded
3. use the fresh controller read plus controller system-status decoding to
   confirm the relevant circuit assignment and control clues before command
   encode
4. encode the Pentair controller `0x9b` pump-circuit configuration write needed
   to update that circuit slot speed
5. correlate the expected controller follow-up flow for that write, currently
   believed to be:
   - controller `0x01` ACK
   - remote `0xd8` follow-up request
   - controller `0x18` broadcast reply

Current EasyTouch read-before-write evidence:

- the controller pump-slot read path is now live-validated via `0xd8 -> 0x18`
- working raw follow-up requests captured so far are:
  - Pump `#1`: `ff00ffa5341021d8010101e4`
  - Pump `#2`: `ff00ffa5341021d8010201e5`
- these frames currently suggest the request layout still contains one
  unresolved byte beyond the simple "selector plus checksum" model, so the
  initial implementation should preserve the observed working bytes rather than
  force a cleaner schema before more captures exist
- the observed `0x18` replies strongly support:
  - payload byte `0` = pump slot / pump index
  - payload byte `1` = pump type
  - the reply behaves like a full pump-slot configuration read

Application-versus-protocol rule:

- read-before-write in Splash is an application safety rule, not a general
  Pentair protocol requirement
- some Pentair write-capable actions can be sent directly on the wire without a
  preceding read
- Splash should still prefer a fresh live baseline before controller
  configuration writes so the application does not overwrite unrelated state
  from stale assumptions

Rules:

- the sequence should remain part of one normalized command lifecycle
- the equipment is the primary source of configuration truth for config writes
- Splash must not trust a cached or previously observed pump configuration when
  encoding a live controller config write
- every Splash-managed live config write must begin from a fresh live read of
  the current equipment/controller configuration, even if Splash maintains a
  local cache
- controller-status circuit discovery is a prerequisite, not an optional hint,
  for the first controller-managed implementation
- all writes in the sequence should carry the same `command_id`
- `protocol.command.encoded` and `serial.write.request` may emit one message per
  encoded transport write while still representing one normalized command
- the plugin should attach `bus_requirements.requires_idle_ms = 50`
- unsupported Pentair direct-pump and non-controller write families remain out
  of scope for this first slice

## Controller Circuit Configuration Discovery

Protocol Explorer may trigger `request_circuit_config` to ask the EasyTouch
controller for circuit-configuration records.

Rules:

- encode Pentair `0xcb` get-circuit requests for the requested 1-based circuit
  index range
- publish the full encoded plan for observability
- transmit only one `0xcb` request at a time
- after the transport acknowledges a request write, wait for a decoded `0x0b`
  `circuit_configuration` reply before transmitting the next request
- if no reply is observed before the command timeout, time out the normalized
  command instead of continuing to send the remaining requests
- treat the decoded reply as diagnostic configuration data, not as proof of
  physical relay installation
- keep the same `command_id` across the whole range so API and dashboard clients
  can associate the sequence with one operator action

ASSUMPTION: until circuit index versus returned circuit id is fully validated,
the coordinator treats the next observed `circuit_configuration` reply during an
active discovery sequence as the response for the in-flight request.

## Controller Schedule Request

Protocol Explorer may trigger `request_controller_schedule` to ask the
EasyTouch controller for one schedule-detail record.

Rules:

- encode Pentair `0xd1` get-schedule requests using controller-family header byte `0x34` (`52`)
- validated request payload is a single one-byte 1-based schedule selector
- accept only selectors `1-12`
- publish the full encoded plan for observability
- keep the request diagnostic and non-mutating
- treat transport acknowledgement as sufficient command completion for this
  first request slice
- expect decoded `0x11` or `0x91` EasyTouch schedule-detail replies to arrive
  asynchronously and be captured separately by the decode path

## EasyTouch Schedule Payload Construction

Splash may construct compact EasyTouch schedule update payloads for validation,
unit testing, and later command-path integration without transmitting them to
the RS-485 bus in this slice.

Rules:

- target only Pentair EasyTouch or IntelliTouch-style compact schedule payloads
- use action `0x91` (`145`) only as a frame-wrapping reference when the payload
  is later embedded into an outbound command
- keep payload construction separate from transport send-path behavior
- supported repeating schedule payload layout is `[scheduleId, circuitId,
  startHour, startMinute, endHour, endMinute, dayMask]`
- supported egg-timer payload layout is `[scheduleId, circuitId, 25, 0,
  runtimeHours, runtimeMinutes, 0]`
- validate all byte positions explicitly and fail closed on unsupported or
  unvalidated special cases
- do not guess run-once, delete, disable, or IntelliCenter schedule-table
  semantics in this slice
- warning: generated outbound schedule-write frames must be verified against
  packet captures before they are enabled on real equipment

## Startup Schedule Warmup

When `splash-api` first observes controller state and does not yet have cached
EasyTouch schedule records, it should issue a one-time warmup request for
schedule selectors `1-12`.

Rules:

- reuse `request_controller_schedule` for each slot
- issue the requests once per API process startup rather than continuously
- keep the warmup best-effort; failures should not block the rest of API
  startup behavior
- cache any decoded `0x11` or `0x91` replies that arrive so frontend schedule
  views can render from warmed data

## Controller Circuit State Actions

Dashboard or other API clients may trigger `set_circuit_state` for a known
writable EasyTouch controller circuit.

Rules:

- encode the write in the controller-family frame format with protocol byte
  `0x34`
- encode Pentair action `0x86`
- payload `[circuit_id, enabled]`
- after the transport acknowledges the write, do not complete the normalized
  command immediately
- wait for a decoded controller `0x01` ACK frame before completing the command
- a controller `0x01` ACK means the controller accepted the request at the
  protocol layer, not that the circuit state has already changed in the next
  status broadcast
- command lifecycle should therefore distinguish:
  - transport write observed
  - controller ACK observed
  - later controller status confirmation in `0x02`

ASSUMPTION: until a more specific ACK-correlation key is proven, the
coordinator treats the next observed controller `0x01` ACK during an active
single-write `set_circuit_state` command as the protocol-level acceptance
signal for that command.

## Controller Date / Time Actions

Dashboard or Protocol Explorer may trigger controller date/time actions.

Initial rules:

- `request_controller_datetime`
  - encode provisional Pentair action `0xc5`
  - payload `[0x00]`
  - initial completion rule: complete after transport acknowledgement
  - continue to surface any later controller-originated `0x05` reply in the
    frame stream for validation
- `sync_controller_datetime`
  - encode provisional Pentair action `0x85`
  - payload is derived from Splash system time in the API layer at request time
  - initial completion rule: complete after transport acknowledgement
  - dashboard should treat the command as a best-effort sync request until live
    reply confirmation is validated

ASSUMPTION: the exact `0xc5 -> 0x05` and `0x85` payload layouts remain
provisional and may need revision once live captures exist.

ASSUMPTION: the initial live implementation depends on captured EasyTouch
circuit-speed command and confirmation frames rather than the earlier direct
IntelliFlo Program 1 RPM inference.

ASSUMPTION: milestone 1 focuses only on controller-managed pump-circuit
configuration updates and does not yet expose direct pump `set_speed` fallback
for no-controller or no-circuit installations.

## Deferred direct-pump scenarios

The following scenarios are explicitly deferred beyond the first controller-
managed milestone slice:

1. pumps with no EasyTouch-managed circuits assigned
2. pumps not connected through an EasyTouch controller path
3. standalone or no-controller deployments where Splash must issue direct pump
   writes and keep the requested RPM in effect itself

Those scenarios require separate frame capture, protocol design, and operator
workflow decisions. They should not weaken the initial controller-managed
command model.

## Manual Remote Layout diagnostic request

Protocol Explorer may trigger a manual Pentair Remote Layout request without
pretending it is a normalized equipment-control action.

Initial rules:

- protocol plugin: `pentair_easytouch`
- command type: `request_remote_layout_page`
- destination: controller `0x10`
- source: Splash remote or client address
- frame:
  - protocol byte `0x01`
  - action `0xe1`
  - payload `[page_index]`
- initial completion rule:
  - complete once the transport write is observed successfully
  - later `0x21` response correlation is tracked separately as protocol mapping
    work, not required for this first diagnostic slice

## Manual raw-frame diagnostic request

Protocol Explorer may also trigger an explicit raw bus write without pretending
it is a normalized equipment-control action.

Initial rules:

- protocol plugin: `pentair_easytouch`
- command type: `send_raw_frame`
- arguments:
  - `bytes_hex`
- the first slice must:
  - validate lowercase hex shape
  - preserve the exact operator-provided bytes
  - avoid checksum or field rewriting
- initial completion rule:
  - complete once the transport write is observed successfully
- do not imply any protocol-level semantic success

## Manual pump-info diagnostic request

Protocol Explorer may trigger an explicit EasyTouch pump-slot information
request without pretending it is a normalized equipment-control action.

Initial rules:

- protocol plugin: `pentair_easytouch`
- command type: `request_pump_info`
- destination: controller `0x10`
- source: Splash remote/client address `0x21`
- initial supported selectors:
  - `1` for Pump `#1`
  - `2` for Pump `#2`
- working raw request frames currently validated:
  - Pump `#1`: `ff00ffa5341021d8010101e4`
  - Pump `#2`: `ff00ffa5341021d8010201e5`
- initial completion rule:
  - complete once the transport write is observed successfully
- later `0x18` response correlation remains part of protocol mapping and
  milestone-1 confirmation work

## Manual pump-config diagnostic write

Protocol Explorer may trigger an explicit EasyTouch pump configuration write
without pretending it is yet the finalized normalized milestone-1 `set_speed`
flow.

Initial rules:

- protocol plugin: `pentair_easytouch`
- command type: `write_pump_config`
- destination: controller `0x10`
- source: Splash remote/client address `0x21`
- action: `0x9b`
- request body supplies the full structured pump-config payload:
  - `pump_id`
  - `pump_type`
  - `priming_time`
  - `unknown_3`
  - `unknown_4`
  - `slots[1..8]` with `circuit_assignment` and `rpm`
  - `priming_speed`
- initial completion rule:
  - complete once the transport write is observed successfully
  - later `0xd8 -> 0x18` verification remains a separate operator-driven step

This route exists to support controlled protocol experiments and manual
controller configuration updates while the normalized controller-circuit
`set_speed` flow is still being formalized.

Safety rule:

- even for Explorer-driven controller configuration writes, operators and
  future automated flows should prefer read-modify-write from a fresh live
  controller baseline rather than constructing a full config block from cached
  or inferred local state
- this is a Splash application safety policy; it should not be mistaken for a
  claim that the underlying Pentair protocol always requires a preceding read

## Result stages

Command result progression should use `command.result.{command_id}`.

Expected states:

- `accepted`
- `encoded`
- `transmitted`
- `completed`
- `timed_out`
- `failed`

Expected stage meanings:

- `accepted`: intent was received and passed initial validation
- `encoded`: plugin successfully encoded the command
- `transmitted`: `serial.tx.raw.write_result = ok` was observed for the command
- `completed`: matching protocol evidence confirmed the intended command effect
- `timed_out`: no matching response or confirmation was observed before timeout
- `failed`: plugin validation failed, encode failed, stream went stale, or transport returned a non-`ok` terminal result

For the initial controller-managed pump-circuit `set_speed` path:

- `transmitted` means all required controller write frames for the command were
  acknowledged by `serial.tx.raw` with `write_result = ok`
- `completed` means later controller observations confirm that the requested
  circuit-slot speed update is reflected in the controller reply flow, currently
  expected to include `0x01`, `0xd8`, and `0x18`

## Correlation ownership

`splash-protocol` owns correlation between:

- `protocol.command.intent`
- `protocol.command.encoded`
- `serial.write.request`
- `serial.tx.raw`
- subsequent decoded response or status frames

Transport success does not imply protocol success.

Rules:

- `serial.tx.raw.write_result = ok` means bytes reached the transport layer
- only `splash-protocol` may determine whether the command actually completed
- stale-stream, timeout, rejected, or port-error transport results should become failed command states unless the command is retried through a new stream
- if any write in a multi-write command sequence fails, the whole normalized
  command should transition to `failed`

## Correlation window

Command state should be held in memory until:

- a terminal result is reached
- the active `stream_id` changes
- the command timeout elapses

ASSUMPTION: the default correlation timeout is `5000ms`, with plugin-specific
overrides allowed later.

## Stream changes

If the active `stream_id` changes while commands are pending:

- pending commands for the prior stream must not remain silently active
- command results should transition to `failed` with a machine-readable stale-stream or stream-reset reason
- a later retry must create a new command flow against the new stream

## Minimum bus requirements

The plugin may attach protocol-specific transport requirements to
`serial.write.request.bus_requirements`.

In v1 this primarily includes:

- `requires_idle_ms`

Rules:

- the plugin may recommend or require the minimum bus-idle wait
- `splash-serial` remains the owner of actual bus-idle enforcement

## Dry-run behavior

Dry-run command intent should:

- validate through the active plugin
- publish `protocol.command.encoded`
- avoid publishing `serial.write.request`
- publish a non-terminal or explicit dry-run command result as appropriate

ASSUMPTION: the first implementation may treat dry-run as `accepted` plus
`encoded` without a transmit stage, while a richer dry-run result shape can be
added later if needed.
