# Platform Workflows

## Onboarding

The application gates normal navigation until `setup_complete = false` is resolved through the setup wizard.

### Steps

1. Welcome
2. Pool profile
3. Equipment inventory
4. Connect to pool
5. Completion summary

### Outputs

- pool profile created
- optional equipment records created
- default maintenance schedules seeded
- initial weather fetch triggered
- setup marked complete

## Initial Implementation Workflow

The first implementation target is a minimal end-to-end operational slice that proves Splash can read live equipment state, present it in the browser, and perform one real equipment write safely.

### Read Path

1. `splash-serial` reads live Pentair RS-485 traffic.
2. `splash-protocol` assembles frames and decodes trusted Pentair fields.
3. `splash-protocol` emits normalized state for:
   - controller air temperature
   - controller water temperature
   - chlorinator salt level
   - pump RPM
4. API persistence or latest-state projection records those values.
5. The browser shows the current values on the initial dashboard or equipment view.

### Write Path

1. The user changes pump-circuit RPM from the browser.
2. The API validates the normalized `set_speed` command against the target controller-managed pump circuit.
3. `splash-api` publishes `protocol.command.intent`.
4. `splash-protocol` encodes the Pentair controller-mediated circuit-speed command path and publishes `serial.write.request`.
5. `splash-serial` writes the bytes to the bus and reports `serial.tx.raw`.
6. `splash-protocol` correlates the write and resulting controller and pump-status frames.
7. The browser shows pending, transmitted, and completed or failed command state.

Milestone-1 rule:
- prefer controller-cooperative EasyTouch circuit-speed control rather than direct IntelliFlo RPM writes
- direct pump-only control and no-controller scenarios remain later follow-up work

### Initial Success Criteria

- the browser shows current air temperature
- the browser shows current water temperature
- the browser shows current salt level
- the browser shows current pump RPM
- the browser can change pump-circuit RPM through the documented command flow
- the system logs or persists enough state to verify the read path and resulting control action

## Equipment Scheduling Direction

Splash is intended to become the scheduling authority for normal pool-equipment operation rather than leaving day-to-day schedules in the EasyTouch controller.

Target direction:
- Splash should own pump-speed scheduling once the scheduling slice is implemented.
- Controller-native EasyTouch schedules should become unnecessary for normal operation.
- The initial implementation milestone does not require schedule replacement, but future scheduling work should be planned toward this outcome.

## Routine Chemistry Maintenance

### Daily

- Test free chlorine.
- Visually inspect the water.
- Empty skimmer and pump baskets as needed.

### Two To Three Times Per Week

- Test pH.
- Retest FC after heavy use, high UV, or rain.
- Brush pool walls and floor.
- Skim debris.

### Weekly

- Run a full chemistry test.
- Check combined chlorine.
- Record any meaningful chemical additions made during the week.
- Inspect or clean filter.
- Inspect the salt cell.

### Monthly Or As Needed

- Adjust total alkalinity.
- Adjust calcium hardness.
- Check CYA.
- Clean and inspect the salt cell.

### Treatment logging guidance

Testing and treatment are related but distinct actions.

Expected operator flow:

1. Test the water.
2. Decide what should be added.
3. Add the treatment.
4. Record what was added as a chemical-addition event.
5. Record any meaningful maintenance performed, such as brushing, vacuuming,
   skimming, or filter cleaning, as a maintenance-activity event.
6. Retest later as needed.

## Seasonal Workflows

### Opening

1. Remove and store the cover.
2. Reconnect equipment and fittings.
3. Fill to operating level.
4. Prime and start the pump.
5. Test full chemistry.
6. Correct chemistry in sequence.
7. SLAM if needed.
8. Re-enable automation and verify RS-485 connectivity.

### Closing

1. Balance chemistry first.
2. Raise FC appropriately before closing.
3. Add winterizing chemistry as appropriate.
4. Lower water and protect lines if freeze risk exists.
5. Drain vulnerable equipment.
6. Install cover.
7. Optionally disconnect RS-485 hardware for winter shutdown.

## SLAM Workflow

### Flow

1. Start SLAM session.
2. Calculate FC target from current CYA.
3. Raise FC to SLAM level.
4. Test and log FC every 2-4 hours.
5. Run OCLT.
6. Confirm all three pass criteria.
7. Complete or abandon session.

### Completion Criteria

- combined chlorine at or below `0.5 ppm`
- water visually clear
- overnight FC loss below `1.0 ppm`

## Automation Approval Workflow

1. Scheduler receives fresh weather data.
2. Rules are evaluated against pool, equipment, and weather state.
3. Matching rules publish `automation.suggestion.*`.
4. API persists a task and notification context.
5. User sees a suggestion in Equipment and Tasks.
6. User approves or dismisses.
7. Approved tasks publish normalized protocol command intent.
8. `splash-protocol` encodes the command into raw bytes.
9. `splash-serial` transmits the raw bytes to the bus.
10. `splash-protocol` correlates the resulting protocol activity and emits `command.result`.

### Command Lifecycle Details

- `splash-api` creates the `command_id`.
- The command remains pending until `command.result.{command_id}` reaches a terminal state.
- Transport success alone does not complete the command.
- Dry-run commands stop at the encode stage and return a synthetic successful result without bus transmission.
- Automation suggestions and approval cards should be expressed in normalized command vocabulary, not protocol-specific terminology.
- Automation suggestions must only target equipment whose effective capabilities support the proposed normalized command.

## Protocol Explorer Workflow

### Live Monitor

- consume decoded protocol frames from `splash-protocol`
- show decoded and raw fields
- filter, pause, and export
- optionally surface transport chunk boundaries for advanced debugging, but frame-level data remains the primary view
- surface buffered receive-side bytes separately when they may still become part of a valid frame
- surface unknown receive-side byte segments separately when the protocol layer can prove they were not part of any known valid frame
- clearly distinguish:
  - known fields
  - inferred fields
  - unknown bytes
  - operator-needed questions

Initial implementation note:
- the first Protocol Explorer slice may begin with a live frame stream only through `/protocol/frames`
- decode, annotation, diff, and simulator tooling may follow after the live stream is available for protocol discovery

### Watch Session

1. The operator or assistant starts a watch session explicitly.
2. Splash records all live Explorer frame events into that session from that point forward.
3. The operator performs one controller or equipment action.
4. The operator or assistant stops the watch session.
5. Splash returns the exact captured frame set so it can be displayed, summarized, or compared later.

Rules:
- watch sessions should capture receive-side and outbound Explorer events
- watch sessions may be filtered to selected event types when the operator wants a narrower transport-only view such as `serial.rx.raw` and `serial.tx.raw`
- watch sessions should not depend on the rolling recent-frame buffer after they start
- the first slice may remain in-memory and local to `splash-api`

### Manual Remote Layout Request

- the operator chooses a page index in Protocol Explorer
- `splash-api` publishes a diagnostic `protocol.command.intent`
- `splash-protocol` encodes a Pentair Remote Layout request:
  - protocol byte `0x01`
  - destination `0x10`
  - action `0xe1`
  - payload `[page_index]`
- `splash-serial` writes the request to the bus
- later controller `0x21` traffic is inspected through the live frame stream and saved bundles

### Manual Raw Frame Send

- the operator pastes an explicit lowercase hex frame into Protocol Explorer
- `splash-api` publishes a diagnostic `protocol.command.intent`
- `splash-protocol` treats the bytes as an Explorer-only raw write request
- `splash-serial` writes the exact bytes to the bus without checksum or field rewriting in the first slice
- Explorer inspects:
  - `protocol.command.encoded`
  - `serial.tx.raw`
  - any later receive-side response traffic

Guardrails:
- this path is diagnostic-only and not promoted into normal dashboard or task workflows
