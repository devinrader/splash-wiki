# Swimmability Scoring

[Back to README](Home)

## Purpose

This page describes the current first-slice swimmability algorithm used by
Splash.

The current implementation is:

- read-only
- heuristic
- intentionally explainable
- meant to guide an operator rather than automate equipment actions

Source issue:
- `#114 Feature: Build swimmability index model and operator-facing UI`

## High-level approach

Splash determines swimmability by combining:

- the latest chemistry reading
- configured chemistry bounds
- current cover state
- latest weather forecast
- latest water-temperature telemetry when available
- rainfall since the last chemistry reading

The first slice treats:

- chemistry as the primary safety signal
- chemistry age and post-test rainfall as confidence modifiers
- cover, UV, hot weather, and warmer water as confidence or comfort modifiers
- water temperature as a comfort signal rather than a hard safety block

## Output

The first slice returns:

- `status`
  - `good`
  - `caution`
  - `poor`
  - `unknown`
- `score`
  - bounded integer from `0` to `100`
- `summary`
  - short human-readable interpretation
- `headline`
  - short operator-facing status text for the Home card
- `confidence`
  - `high`, `medium`, `low`, or `unknown` depending on how trustworthy the
    chemistry context still is
- `last_chemistry_age_label`
  - human-readable chemistry recency text such as `3 days ago`
- `highlights`
  - a small curated set of operator-facing badges for the Home card
- `drivers`
  - explainable contributing reasons

## Step-by-step algorithm

### 1. Start optimistic

The first slice starts from:

- `score = 100`
- `status = good`

### 2. Require at least one chemistry reading

If there is no chemistry reading yet:

- `status = unknown`
- `score = 0`
- reason: Splash does not have enough chemistry evidence to judge swim
  conditions

### 3. Check documented do-not-swim chemistry conditions

If any of these are true, the result becomes `poor`:

- `pH < 7.0`
- `pH > 8.2`
- `free chlorine < 0.5 ppm`
- `cyanuric acid > 100 ppm`

These conditions are treated as hard safety drivers in the first slice.

### 4. Check chemistry against configured bounds

If values fall outside configured minimum or maximum ranges, Splash adds caution
drivers for the affected chemistry keys.

The first slice checks:

- free chlorine
- pH
- total alkalinity
- calcium hardness
- cyanuric acid

Values inside the configured ranges contribute positive or neutral guidance.
Values outside the configured ranges lower the score and can degrade `good` to
`caution`.

### 5. Check combined chlorine

When both total chlorine and free chlorine are present:

- `combined chlorine = total chlorine - free chlorine`

If combined chlorine is above `0.5 ppm`, Splash adds a caution driver.

## Chemistry recency and confidence

The first slice does not treat chemistry age as a simple fixed timer.

Instead, Splash computes a confidence signal based on:

- elapsed time since the latest chemistry reading
- whether the pool is covered or uncovered
- UV level
- air temperature
- water temperature when available
- rainfall since the last chemistry reading

### Base age thresholds

Before applying context modifiers:

- around `7 days` old becomes `caution`
- around `14 days` old becomes `unknown`

### Context-aware staleness

Chemistry confidence degrades faster when:

- the latest cover state is `off`
- UV is elevated or high
- air temperature is warm or hot
- water temperature is elevated or warm
- meaningful rainfall has occurred since the last chemistry reading

Chemistry confidence degrades more slowly when:

- the latest cover state is `on`

### Why cover and weather matter

The first slice assumes that:

- uncovered pools lose chlorine confidence faster under UV exposure
- hotter air and warmer water increase chlorine demand
- rainfall after a test can disturb or dilute chemistry enough to reduce
  confidence in the older reading

These factors modify confidence in the chemistry reading. They do not rewrite
the chemistry measurement itself.

## Rainfall effect

Rainfall is treated as a post-test context input.

If rain has fallen since the last chemistry reading:

- confidence degrades
- heavier rainfall degrades confidence more than lighter rainfall

This is why a pool can shift from `good` chemistry to `caution` or `unknown`
even when the last logged chemistry values were acceptable.

## Cover effect

Cover state influences how quickly chemistry confidence ages.

- `cover on` helps preserve confidence
- `cover off` accelerates chemistry staleness

The first slice uses cover as a context modifier, not as an independent safety
blocker.

## Water temperature effect

Water temperature is treated as a comfort modifier.

- very cool water can push the result toward `caution`
- very warm water can also push the result toward `caution`

Water temperature alone does not make the result `poor` in the first slice.

## Status interpretation

### `good`

Used when:

- no hard chemistry safety conditions are present
- chemistry is within or near configured ranges
- the chemistry reading is still trustworthy enough in current conditions

### `caution`

Used when:

- chemistry has drifted outside configured ranges, or
- chemistry confidence is weakening because of age, rain, UV, uncovered state,
  or heat, or
- water temperature is less comfortable even if chemistry is otherwise safe

### `poor`

Used when:

- at least one hard do-not-swim chemistry condition is present

### `unknown`

Used when:

- no chemistry reading exists, or
- the available chemistry reading is too stale or too context-weakened to trust

## What this is not

The first slice is not:

- a statistically trained model
- a predictive automation engine
- a substitute for operator judgment
- a substitute for direct water testing when confidence is low

## Future extensions

Later versions may extend the model with:

- richer chemistry trend analysis
- rainfall markers and weather overlays from `History`
- UV and chlorine-destruction modeling
- stronger cover-history interpretation
- seasonal adjustments
- SLAM-aware logic
- notification and task generation

## Code reference

Current implementation:

- [swimmability.ts](</Users/devinrader/Projects/splash/splash.platform/services/splash-api/src/swimmability.ts>)
