# Splash Frontend Testing

[Back to Splash Frontend README](Services-Splash-API-Home)

## Testing goals

For the initial milestone, `splash-frontend` should have enough automated
coverage to validate the browser read and control slice without a live backend.

## Required automated coverage

- render latest-state values from a mocked `GET /equipment` response
- update the UI from mocked SSE `equipment.state`, `pump.state`, and
  `command.result` events
- submit pump-speed control through `POST /equipment/:id/control`
- disable repeat submission while a command is pending
- surface API or command failures in the UI
- render a narrow Protocol Explorer panel from mocked protocol frame and bundle
  APIs
- show outbound Explorer diagnostic events such as `protocol.command.encoded`
  and `serial.tx.raw` when present
- create bundle comparison, annotation, and prompt requests through the
  documented API endpoints
- keep the live frame list inside a bounded panel rather than letting new frame
  traffic continually grow page height
- switch the main content area when a sidebar destination is selected
- apply active styling to exactly one sidebar destination at a time
- switch Diagnostics tab content without reloading the page
- switch Automation tab content without reloading the page
- render the Automation destination as a working tabbed page rather than a generic placeholder
- render the mockup-derived schedule table and scheduling-mode side panel
- render the placeholder `Network` diagnostics tab with five named cards
- keep the sidebar fixed while the main content remains centered and width
  constrained on desktop screens
- avoid horizontal overflow at the documented tablet and desktop breakpoints
- keep internal dashboard and diagnostics grids readable inside the constrained
  content container
- switch `System` page tabs without changing the surrounding application shell
- render distinct placeholder content for `Overview`, `Equipment`, `Sensors`,
  `Water Features`, and `Connectivity`

## Preferred tooling

- Vitest for unit and component tests
- React Testing Library for UI behavior
- lightweight fetch and `EventSource` mocks rather than a full browser backend

## Milestone-1 test boundaries

- no real NATS dependency
- no real browser automation requirement yet
- no real protocol or serial hardware dependency

## Follow-up work

- broader route coverage after the full long-term navigation model is
  implemented
- end-to-end browser tests after containerized frontend delivery exists
