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
- a working schedule table using seeded rows that match the mockup structure
- columns for name, action, days, time, season, status, and next run
- a scheduling mode side panel with:
  - current mode summary
  - controller-managed vs platform-managed toggle presentation
  - explanatory copy
  - a non-destructive migration call to action

Rules:
- the first slice may present the scheduling mode toggle as UI-only
- the first slice must not claim that platform-managed scheduling is already
  active unless backed by real platform state

Next slice: controller-backed schedules
- when controller-native scheduling is selected and validated controller
  schedule data is available, the `Schedules` table should replace seeded rows
  with real EasyTouch controller schedule records
- those rows should come from a documented `splash-api` read-only route rather
  than direct frontend protocol access
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
