# Splash Frontend Automation Surface

[Back to Splash Frontend README](Services-Splash-Frontend-Home)

## Purpose

The `Automation` destination in `splash-frontend` is the browser-facing surface
for monitoring and later configuring Splash automation behavior.

For the current milestone, it should convert the existing static automation
mockups into a working tabbed frontend surface inside the shared application
shell without introducing new backend automation APIs.

## Initial milestone scope

The first Automation slice is a frontend-only information architecture and
presentation milestone. It should:

- replace the generic Automation placeholder page with a working tabbed surface
- preserve the mockup tab model and labels:
  - `Overview`
  - `Schedules`
  - `Rules`
  - `Scenes`
  - `Triggers`
  - `Logs`
- render content that matches the intent and structure of the existing mockups
  under `splash.mockup/site/automation/`
- use seeded frontend-local data where live automation APIs do not yet exist
- remain visually aligned with the shared frontend shell and design tokens
- behave like a real in-app page rather than static HTML files

The first Automation slice should not:

- invent new backend contracts for automation CRUD
- bypass `splash-api` for future live automation data
- imply that rules, scenes, triggers, or schedules are already persisted by the
  platform when they are still mock or seeded UI data

## Tab contract

### `Overview`

Purpose:
- provide a high-level summary of the current automation surface
- give the operator one place to see counts, current mode, upcoming work, and
  recent activity

Initial content:
- summary cards for schedule, rule, scene, and trigger counts
- a primary explanatory panel matching the existing mockup intent
- seeded status and activity content derived from mockup examples

### `Schedules`

Purpose:
- show the operator the current scheduling surface and scheduling-mode posture

Initial content:
- a controller-managed scheduling summary surface above the main table
- a compact capacity strip that explains EasyTouch scheduling limits:
  - `12 total programs max`
  - `9 max per circuit`
  - currently used vs remaining program capacity
- a per-circuit capacity card or meter for the currently emphasized circuit when
  validated controller schedule ownership is available
- a working schedule table that may evolve from the earlier seeded mockup into a
  more controller-native layout
- for the controller-backed slice, the primary columns may include:
  - circuit
  - program slot
  - days
  - start
  - stop
  - heat
  - status
  - actions
- when a field such as heat mode is not yet validated by the controller payload,
  the page should render an explicit placeholder such as `—` rather than invent
  a decoded value
- the page may still include scheduling-mode guidance and explanatory copy, but
  the primary emphasis should shift to controller capacity and schedule-slot
  visibility rather than the earlier side-panel-first presentation

Rules:
- program-capacity accounting in the summary strip should use active schedules
  to determine used vs unused controller slots
- inactive schedules should still remain visible in the table because they are
  part of the operator's controller-backed schedule inventory, but they should
  be visually secondary
- the schedule editor must appear inside the schedule-page main content
  rather than in a modal dialog
- when the desktop or tablet layout has a schedule-detail side column, the
  schedule editor should appear in that right-hand column beside the table rather
  than pushing the schedule table downward
- the schedule editor should remain visible by default in this slice
- when validated controller schedule rows are available, the editor should
  default to the first stable controller program, such as program `1`
- the schedule-row `Review` action should load the selected row into the
  right-hand editor instead of opening a modal
- the schedule editor should become controller-backed for the validated write
  slice rather than remaining browser-local preview only
- the first direct-write slice should support saving only validated EasyTouch
  `repeat` and `egg_timer` schedule fields
- the UI must reject or disable `run_once`, delete, disable, and heat-setting
  mutations until those controller semantics are protocol-validated
- the schedule editor should reflect the controller constraints in the UI:
  - circuit selector
  - schedule type or program intent
  - day selection
  - start and stop times
  - optional heat placeholder or disabled control when controller-backed heat
    semantics are not yet implemented
  - contextual copy about active-slot capacity and the selected controller
    program

Rules:
- save actions should show pending, success, and failure state inline in the editor
- after save, the editor should reload from refreshed controller data rather than trusting the unsaved draft as truth
- the first slice must not claim that platform-managed scheduling is already
  active unless backed by real platform state

Next slice: controller-backed schedules
- when controller-native scheduling is selected and validated controller
  schedule data is available, the `Schedules` table should replace seeded rows
  with real EasyTouch controller schedule records
- those rows should come from a documented `splash-api` read-only route rather
  than direct frontend protocol access
- the controller-backed schedule view may summarize schedule-slot utilization in
  the browser as long as it is derived only from validated schedule ownership
  and record counts returned by the API
- Splash must not present guessed byte interpretations as real schedule fields
- if validated controller schedule data is unavailable, stale, or only partially
  decoded, the page should surface that limitation explicitly instead of
  pretending to know the operator schedule semantics

### `Rules`

Purpose:
- explain and preview condition-based automation behavior

Initial content:
- a non-placeholder page with explanatory content based on the mockup
- seeded examples or summary content are allowed

### `Scenes`

Purpose:
- explain and preview reusable grouped actions across multiple equipment targets

Initial content:
- a non-placeholder page with explanatory content based on the mockup
- seeded examples or summary content are allowed

### `Triggers`

Purpose:
- show the real-time or conceptual inputs that can activate automation behavior

Initial content:
- a non-placeholder page with explanatory content based on the mockup
- seeded examples or summary content are allowed

### `Logs`

Purpose:
- provide a browser destination for reviewing automation history and outcomes

Initial content:
- a non-placeholder page with explanatory content based on the mockup
- seeded recent-event rows or summary cards are allowed

## Data expectations

For this milestone:

- content may come from seeded frontend-local fixtures
- seeded data should be clearly implementation-local and easy to replace with
  future API data
- the page should not fabricate mutating behavior that suggests persistence
  exists when it does not

Future live data should come from documented `splash-api` routes once those
contracts exist.

For controller-native schedule visibility specifically:

- the next slice should use `GET /controller/schedules`
- that route should return validated controller schedule records plus freshness
  and source metadata
- for EasyTouch, the first validated live records come from decoded schedule
  detail actions `17` and `145`
- the platform should warm controller schedule cache on API startup so the
  `Schedules` table normally renders from cached controller data rather than
  waiting for a manual operator refresh
- until the Pentair schedule payload is sufficiently decoded, the frontend must
  keep the existing seeded rows or show an explicit unavailable state rather
  than fabricating real schedules

## Interaction expectations

- Automation page tab changes should happen client-side without reloading the
  shell
- the active Automation tab should be visually distinct
- the page should remain responsive on desktop and tablet widths already
  supported by the frontend shell
- mockup copy may be tightened for production readability, but the structural
  intent should remain recognizable

## Relationship to broader automation workflows

This page does not replace the existing automation approval workflow defined in
[Platform Workflows](Workflows-Platform-Workflows).

Until richer backend automation resources exist:

- approve or dismiss actions remain task- and workflow-driven
- the Automation destination primarily acts as an operator-facing monitoring and
  navigation surface
- the schedule-mode presentation is informative rather than authoritative

## Primary references

- [Splash Frontend Service](Services-Splash-Frontend-Home)
- [Product Requirements](Product-Requirements)
- [Platform Workflows](Workflows-Platform-Workflows)
- [REST API Contract](Interfaces-REST-API)
