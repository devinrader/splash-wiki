# Overview

[Back to README](Home)

## Product summary

Splash is a self-hosted pool management platform for individual homeowners. It combines pool equipment connectivity, weather context, water chemistry tracking, and predictive guidance to reduce the time required to manage a residential swimming pool.

The stated SMART objective is to reduce average weekly pool-management time by 75%.

## Vision

Splash is intended to make proactive pool management practical for non-experts by:

- surfacing what needs attention before a problem becomes visible
- combining sensor, weather, and user-entered data into actionable guidance
- helping users replace reactive maintenance with structured routines and automation
- supporting both beginner and experienced pool owners without requiring paid pool service

## Target users

- First-time pool owners who need guided maintenance and confidence-building automation
- Experienced pool owners who want to reduce weekly effort
- Homeowners currently paying for pool service who want to self-manage
- Smart-home-oriented users who expect connected equipment and observability

## Scope

### Initial implementation target

See [Initial Milestone](Initial-Milestone.md) for the full description of the
end-to-end vertical slice.

Longer-term platform direction:

- In v1, the controller remains the source of truth for scheduling. Splash
  reads and surfaces controller schedules but does not override them by
  default. Operators can choose between controller-native and Splash-native
  scheduling modes; long-term direction is to make Splash-native scheduling the
  preferred mode in future releases. See
  [Requirements](Product-Requirements#equipment).
- pump-speed scheduling is the most important early scheduling replacement goal
- Splash should define machine-readable equipment-parameter data so the product
  can understand and explain model-specific limits such as circuit inventory,
  valid assigned names, supported functions, and other hardware constraints
- Splash should support syncing controller clocks such as the EasyTouch clock
  from the Splash system clock so controller-native schedules and timestamps
  stay aligned with the platform

### In scope

- Connectivity and control for pumps, heaters, lights, and related equipment
- Environmental integration for weather, temperature, UV, and rainfall context
- Water chemistry logging, interpretation, and alerts
- Future chemistry-adjacent workflow tracking for:
  - chemical additions
  - observational water-condition inputs
  - maintenance-activity history
- Equipment maintenance reminders and service history
- Seasonal opening and closing guidance
- Predictive task generation and suggest-and-approve automation
- Future swimmability improvements that depend on richer inputs, including:
  - circulation-duration context
  - cover-duration context
  - per-value provenance and confidence
  - contradiction-aware alerts
  - forecast-based prediction
  - explainable maintenance recommendations

### Out of scope

- No explicit product features are marked out of scope in the source document
- ASSUMPTION: The lack of out-of-scope items reflects an early design phase, not unlimited implementation commitment

## Business and product posture

See [Product Posture](Product-Posture.md) for the canonical product and
deployment stance.

## User experience direction

- Calm, clear, and trustworthy
- Light mode only in v1
- Responsive by default across desktop, tablet, and mobile
- Minimal friction for common actions like logging chemistry or approving automation
- Status communicated consistently with green, amber, and red indicators

## Primary screens

- Home
- Chemistry
- System
- History
- Routines
- Automation
- Diagnostics
- Settings

## Navigation model

The application uses a persistent left sidebar on desktop and a collapsed or
bottom-tab navigation model on smaller screens.

The target top-level navigation model is:

- Home
- Chemistry
- System
- History
- Routines
- Automation
- Diagnostics
- Settings

For the current frontend milestone shell, the implemented sidebar destinations
are still:

- Home
- System
- Routines
- History
- Automation
- Alerts
- Diagnostics
- Water Test Log
- Settings

Target information-architecture rules:

- `Chemistry` owns:
  - Water Test Log
  - chemistry status and manual readings
  - future chemical-additions workflow
  - future SLAM workflow entry
- `Routines` owns:
  - alerts inbox
  - reminders
  - maintenance and seasonal workflows
  - multi-step routines and guided processes
- `History` owns:
  - trends and overlays
  - not manual chemistry entry
- `Diagnostics` owns:
  - advanced tooling
  - event-log and low-level troubleshooting surfaces
  - not the primary user-facing trend history experience

Transitional mapping from the current milestone shell to the target model:

- `Water Test Log` becomes part of `Chemistry`
- `Alerts` becomes part of `Routines`
- `Diagnostics` should rename `Logs & History` style tooling labels so they do
  not compete with the primary `History` destination

Within the current milestone shell:

- `System` is the default operational dashboard view
- `Diagnostics` hosts the initial advanced tooling surface, including
  Protocol Explorer and related placeholder tabs
- some destinations may remain placeholders while their underlying workflows
  are still pending
- the current desktop and tablet shell should keep a fixed left navigation rail
  inside a centered, width-constrained application frame that matches the
  proportions shown in the existing UI mockup images

## Delivery roadmap

### V1 definition of done

See [Initial Milestone](Initial-Milestone.md) for the full definition of the
end-to-end vertical slice.

### Build phases

1. Initial implementation slice: see
   [Initial Milestone](Initial-Milestone.md)
2. Core platform expansion: chemistry screen, equipment screen, dashboard, notification delivery, and broader normalized equipment coverage
3. Remaining v1 features: maintenance reminders, seasonal checklists, task list, predictive rules, cover tracking, SLAM workflow, Protocol Explorer, and rainfall tracking
4. Future roadmap: scheduling-mode selection between controller-native and Splash-native scheduling, machine-readable equipment-parameter definitions and UI surfacing of equipment limits, controller clock sync from Splash system time, dosing recommendations, usage-based maintenance, autonomous automation, predictive fault detection, energy optimization, dark mode, and virtual or mock pool simulation for hardware-free testing and demos
5. Swimmability input expansion roadmap:
   - `#133` chemical additions logging
   - `#134` observational pool-condition inputs
   - `#135` maintenance activity history
   - `#136` pump runtime and circulation-duration derivation
   - `#137` chlorinator telemetry expansion
   - `#138` filter and flow telemetry inputs
   - `#139` cover-duration derivation
   - `#140` per-value provenance and confidence
   - `#141` contradiction and low-confidence alerts
   - `#142` forecast-based swimmability prediction
   - `#143` explainable maintenance recommendations

Developer-note:

- narrow Protocol Explorer tooling may still appear earlier than the broader
  product-facing Protocol Explorer screen when that tooling is needed to
  complete reverse-engineering work for the initial milestone

## Primary technical risks

- RS-485 decoding varies by vendor and firmware version
- Chemistry-sensor integrations and InfluxDB ingestion need early validation
- Predictive automation must start with conservative rule logic before any ML-oriented expansion

## Legacy source note

TODO: The source document references wireframes as future work. No wireframe artifacts were embedded in the Word file beyond the navigation diagram.
