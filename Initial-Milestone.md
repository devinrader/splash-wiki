# Initial Milestone

[Back to README](Home)

## Summary

The initial implementation milestone is a narrow end-to-end vertical slice that
proves Splash can read live RS-485 telemetry, surface it through the API and
frontend, persist enough state to validate the workflow, and send one verified
equipment-control command back to the controller.

## End-to-end vertical slice

The first implementation of Splash should prove one complete read and write
equipment-control slice before the broader v1 surface is considered in scope.

Initial milestone goals:

- read and display current air temperature
- read and display current water temperature
- read and display current salt level
- read and display current pump RPM
- change pump-circuit RPM from the browser
- persist or log enough state and command history to verify the read and write
  path end to end

This first slice exists to validate the minimum useful closed loop from live
RS-485 traffic through protocol decode, persistence, API delivery, browser
presentation, command encode, serial write, and command-result tracking.

The milestone is intentionally narrower than the full v1 requirements table. It
validates:

- RS-485 read transport
- protocol decode and normalized event publication
- API and frontend state presentation
- command encode, serial write, and command-result tracking
- historical logging of surfaced values and control actions

## Definition of done

- browser UI shows current air temperature, water temperature, salt level, and
  pump RPM
- browser UI can change pump-circuit RPM
- the end-to-end read and write path is proven on real equipment
- working RS-485 connection can read status from and send commands to at least
  one connected pool-equipment target
- chemistry logging plus historical trend charts are available in the web UI
