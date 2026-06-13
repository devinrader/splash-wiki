# Requirements

[Back to README](Home)

## Functional requirements

### Initial implementation milestone

See [Initial Milestone](Initial-Milestone.md) for the full description of the
end-to-end vertical slice.

| Area | v1 requirements |
| --- | --- |
| Water chemistry | Log manual or sensor readings, trend chemistry over time, support chlorine and saltwater pools, prompt users to test water, record rainfall as chart context |
| Equipment management | Track equipment inventory, runtime, faults, and maintenance reminders; support equipment control through RS-485 where available |
| Seasonal workflows | Provide guided opening and closing checklists customized by pool type, region, and equipment |
| Predictive automation | Generate suggested pump and heater actions based on weather and pool state; require user approval before execution |
| Pool cover tracking | Record cover on/off events with cover type and use them to interpret chemistry and UV behavior |
| SLAM workflow | Guide the user through SLAM initiation, FC target calculation, periodic logging, OCLT, and completion criteria |
| Protocol Explorer | Provide live frame monitoring, decode tools, simulation, frame diffing, and annotation for RS-485 reverse engineering |
| Onboarding | Gate the app behind a setup wizard until pool profile and baseline setup are complete |
| Notifications | Persist and display maintenance, chemistry, seasonal, automation, and equipment-fault notifications |
| MagicMirror | Expose a read-only integration surface over REST and SSE for a future `MMM-Splash` module |

## Functional detail

### Chemistry

- Track manual chemistry values for pH, free chlorine, total chlorine, total alkalinity, calcium hardness, and cyanuric acid
- Track manual chemical-addition events separately from chemistry readings
- Allow partial chemistry entries
- Warn when both pH and free chlorine are omitted from a manual reading
- Track salt from chlorinator or controller telemetry and rainfall from weather history rather than manual chemistry entry
- Show ideal ranges, alerts, SLAM shading, cover events, and rainfall markers on trend charts
- Use CYA-adjusted free chlorine minimums instead of relying on a single FC target
- Future chemistry workflow coverage should include:
  - durable chemical-addition logging so dosing actions are not inferred only
    from later test values
  - operator-entered qualitative condition inputs such as water clarity, algae
    presence, debris level, and bather-load estimate
  - a dedicated observational history so current-condition scoring can reason
    about visible water quality separately from measured chemistry
  - durable maintenance-activity history for brushing, vacuuming, robot
    cleaning, skimming, and skimmer, pump-basket, or filter cleaning
- Future swimmability and recommendation work should consume:
  - per-value source and confidence metadata
  - contradiction or low-confidence status when inputs disagree
  - user-managed freshness policy from the water-testing schedule
  - recent chemical additions, observations, and maintenance history as
    explainable operator-context inputs

### Equipment

- Support inventory records for pump, filter, heater, lights, cleaner, and chlorinator
- Show live equipment state through SSE
- Expose command controls with loading state until `command.result`
- The first browser control surface must expose pump RPM readout and
  controller-managed pump-circuit RPM control before broader equipment-control
  coverage is considered complete
- v1 must support both controller-native and Splash-native scheduling modes.
  Users can choose their preferred mode. Controller-native remains the default
  in v1.
- Splash should support a user-selectable scheduling strategy so the operator
  can choose either controller-native schedule management or Splash-native
  schedule management depending on equipment capability and preference
- when controller-native scheduling is selected, Splash should provide a way to
  view, manage, or coordinate with controller schedules rather than assuming
  Splash is always the only scheduling authority
- when Splash-native scheduling is selected, Splash may become the preferred
  scheduling surface for supported workflows, but v1 should not assume that
  controller-native scheduling is absent or disabled
- Commands must wait for an explicit confirmation (`command.result`) and
  should not assume a value has changed until it is validated.
- Log maintenance history and surface upcoming service dates
- Resolve each controllable equipment instance to zero or one capability profile in v1
- Validate UI controls and normalized commands against effective per-equipment capabilities rather than equipment type alone
- Ship an initial capability-profile catalog that includes generic fallbacks and Pentair-family v1 profiles for controller circuits, pumps, heaters, and chlorinators
- Define a machine-readable equipment-parameter documentation format that can
  describe model-specific limits, options, and inventories for supported pool
  equipment
- Use that equipment-parameter format to surface equipment limits in the Splash
  UI so both Splash and the operator understand constraints such as available
  EasyTouch 8 system circuits, valid assigned names, or supported function
  choices
- Support controller clock synchronization so Splash can set supported
  controller clocks, such as the EasyTouch clock, to the Splash system time
  when the operator chooses to keep controller-native schedules aligned
- Future pool-health and swimmability inputs should include:
  - derived pump runtime or circulation duration
  - chlorinator state and output percentage
  - filter pressure and flow rate where supported
  - filter-condition or filter-cleaning status when direct telemetry is not
    available
  - cover-duration and uncovered-exposure intervals derived from cover history

### Automation

- Evaluate weather-driven rules after each weather update
- Publish automation suggestions to NATS
- Require approve or dismiss actions from the user
- Prevent duplicate suggestions with minimum re-suggestion intervals
- Include an explainable reason string with each suggestion
- Expose an `Automation` frontend destination with tabbed `Overview`, `Schedules`, `Rules`, `Scenes`, `Triggers`, and `Logs` pages
- Allow the first Automation frontend slice to use seeded UI data from the approved mockups while live automation APIs are still pending
- Keep automation approval behavior aligned with tasks and normalized command workflows until richer automation-management contracts exist
- Future prediction and recommendation work should remain explainable and use
  persisted input history rather than opaque heuristics alone
- Future maintenance-recommendation work should:
  - present recommendations as separate operator guidance, not hidden score
    side effects
  - prioritize `retest` guidance when chemistry trust is weak
  - separate corrective, preventive, and investigative recommendations

### Seasonal and SLAM

- Store checklist definitions, checklist steps, and checklist completion history
- Track active SLAM sessions with all three completion criteria
- Suppress normal FC alerts during an active SLAM
- Increase chemistry-prompt cadence while SLAM is active

### Diagnostics and advanced tooling

- Provide a Diagnostics page that can host multiple advanced-tooling tabs
- The initial Diagnostics tabs should be:
  - `Protocol Explorer`
  - `Live Data Monitor`
  - `Device Inspector`
  - `Logs & History`
  - `Network`
- Stream raw RS-485 frames
- Decode known frames and annotate unknown fields
- Support collaborative protocol discovery by clearly separating decoded fields
  into `known`, `inferred`, and `unknown` confidence levels
- support saving confidence-aware annotations against saved frame bundles and
  byte ranges during collaborative decoding
- Allow Protocol Explorer to surface operator-needed questions or prompts when
  a frame cannot be safely decoded from captured traffic alone
- support saving operator-needed prompts with bundle and frame context during
  collaborative protocol decoding
- Preserve frame bundles or experiment snapshots so one controlled controller
  change can be compared against a baseline during reverse engineering
- the first frame-bundle slice may use an in-memory recent-event buffer before
  persistent Explorer storage is implemented
- the first frame-diff slice may compare saved bundles positionally and only
  highlight direct byte-level changes before richer protocol-aware diffing
  exists
- Support dry-run command simulation by default
- Require explicit confirmation for any live command transmission
- Centralize protocol decode and encode in a dedicated protocol service rather than spreading packet logic across API and scheduler
- Load protocol implementations through configuration-driven plugins
- Reuse the same protocol decode and encode engine for Protocol Explorer and production command handling
- Define explicit raw-transport, protocol-frame, and normalized-command contracts between services
- Ensure command lifecycle tracking distinguishes encode, transmit, and protocol-level completion states
- Maintain authoritative reference documentation for all known pool-equipment communication protocols, including supported, partial, and planned integrations
- Support profile-based capability mapping so protocol plugins can expose richer vendor-specific capabilities without collapsing to a lowest common denominator
- Support machine-readable equipment-parameter definitions alongside protocol
  reference docs so UI, validation, and operator guidance can share one
  structured source of truth for equipment limits
- Preserve native serial read boundaries in the raw transport contract rather than introducing frame-aware buffering in `splash-serial`
- Define an explicit transport-facing outbound write contract from `splash-protocol` to `splash-serial`
- Future platform work should support virtual or mock pool equipment so users
  can create a simulated pool for testing, demos, and protocol experimentation
  without real hardware
- The initial `Network` diagnostics tab may use placeholder-only cards for:
  - topology overview
  - network statistics
  - RS485 bus status
  - event bus status
  - network interfaces

## Non-functional requirements

### Reliability

- Recover automatically from temporary RS-485 disconnects
- Recover automatically from NATS disconnects
- Degrade gracefully when weather data, InfluxDB, or SSE becomes stale
- Keep the app usable for read and manual-entry workflows during partial outages
- Reject stale transport write requests after serial reconnect rather than risking writes on a replaced port session

Services using Core NATS for high-frequency telemetry can accept loss of those
messages. All services must assume that NATS and RS-485 links can drop
messages. Command workflows should time-box execution and validate
configuration changes via command-result tracking before updating state.

### Performance and platform constraints

- Run on Raspberry Pi hardware
- Keep `splash-serial` lightweight enough for a Pi Zero 2W
- Use Docker Compose on `splash-core`
- Use `systemd` on `splash-zero` for the installed `splash-serial` service, with package-managed deployment preferred over raw binary copy
- Keep `splash-serial` observability lightweight through a minimal local HTTP health and metrics surface
- Prefer package-based release artifacts for deployable services, including services that ultimately run inside containers
- Publish prerelease artifacts on every merge to `main` and stable artifacts on semver tags

### Maintainability

- Use a polyglot monorepo that supports TypeScript/Node.js services alongside Go for `splash-serial`
- Keep protocols behind a pluggable `ProtocolDecoder`
- Keep weather behind a pluggable `WeatherProvider`
- Keep future chemistry sensors behind a pluggable `SensorProvider`
- Favor stable event contracts and repository patterns over tight coupling
- Prefer TypeScript for JSON-heavy application services and Go only where constrained-hardware transport needs justify it
- Keep raw transport concerns, protocol concerns, and product-workflow concerns separated by contract

### Security and safety

- Treat live RS-485 command transmission as a sensitive action
- Keep LAN-only operation in v1
- Use Cloudflare Access for v2 remote access rather than building application auth first
- Store secrets in Ansible Vault

### Language and runtime fit

- `splash-serial` must remain lightweight enough for Pi Zero 2W deployment and is expected to use Go
- `splash-api`, `splash-scheduler`, and `splash-protocol` should favor TypeScript/Node.js for faster development, shared types, and better alignment with frontend and Protocol Explorer work

## Dependencies

- SQLite
- InfluxDB
- NATS with JetStream
- USB-to-RS-485 adapter
- Weather provider API key
- Docker on `splash-core`
- Gitea package registry for internal artifact distribution
- Ansible for provisioning and disaster recovery
- Gitea Actions secrets for non-interactive package and image publishing

## Risks

- RS-485 protocol variability across vendors and controller generations
- CYA-adjusted chemistry rules being misunderstood if surfaced without context
- SSE stability through remote-access layers such as Cloudflare Tunnel
- Embedded-device storage and log growth on Raspberry Pi deployments
- Future multi-pool support not yet implemented despite `pool_id`-ready schemas
- mDNS reliability varying by router and LAN environment
- Chemistry-sensor compatibility and ingestion reliability
- InfluxDB sustained-write performance on Raspberry Pi hardware

## Constraints

- v1 is single-pool despite multi-pool-friendly schema design
- No application-layer authentication in v1 LAN mode
- Light mode only in v1
- Full dosing math is deferred to v2
- Hayward and Jandy support are planned but not fully implemented
