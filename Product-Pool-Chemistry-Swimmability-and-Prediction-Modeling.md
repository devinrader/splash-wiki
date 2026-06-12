# Pool Chemistry, Swimmability, and Prediction Modeling Specification

[Back to Product](Product)

## Purpose

This page defines the long-term canonical modeling guidance for how Splash
should reason about pool chemistry, current swimmability, future swimmability,
maintenance readiness, maintenance recommendations, confidence, chemical
additions, and historical learning.

This page is intentionally broader than the current first-slice swimmability
algorithm documented in [Swimmability Scoring](Product-Swimmability-Scoring).
It is also more implementation-oriented than the higher-level
[Swimmability Design Guidance](Product-Swimmability-Design-Guidance).

This document exists so future implementation work can align on one modeling
architecture before introducing additional scoring engines, prediction models,
chemistry calculators, recommendation rules, or confidence systems.

## Core product questions

Splash should ultimately answer four questions:

1. Can I swim now?
2. Can I swim tomorrow?
3. What should I do next?
4. How confident is Splash in those answers?

All future scoring, prediction, recommendation, alerting, and automation
capabilities should align with these four questions.

## Core design principles

### Separate concerns

Splash should not treat all pool quality and maintenance concepts as one
monolithic score.

The platform should maintain separate but related models for:

- current swimmability
- predicted swimmability
- maintenance readiness
- data confidence

### Measured, observed, derived, predicted, estimated, and learned values must remain distinct

Splash should clearly distinguish among:

- measured values
- observed values
- user-entered values
- sensor-provided values
- derived values
- predicted values
- estimated values
- learned values

These categories must never be conflated in storage, explanation, or user
presentation. Future scoring, prediction, and recommendation systems should
preserve explicit provenance metadata such as value kind, source type,
freshness state, confidence band, and explainable reason strings.

### Recommendations must be explainable

Splash should never produce opaque pool advice.

Every recommendation should be traceable to:

- specific input values
- specific freshness and confidence states
- specific chemistry or equipment rules
- specific prediction assumptions when forward-looking reasoning is involved

Prediction and recommendation surfaces should follow a strict show-your-work
contract. They should visibly expose:

- the strongest drivers
- the key assumptions
- stale or missing input warnings
- confidence limits that reduce trust in the output

### Confidence is independent of pool quality

Confidence is not a synonym for score.

A pool can have:

- excellent current chemistry with low confidence because the data is stale,
  incomplete, or contradictory
- poor current chemistry with high confidence because multiple fresh inputs
  agree

### Chemical additions are predictive events

Chemical additions are not merely corrections to current chemistry state.
They are historical treatment events that alter the future trajectory of the
pool.

Splash should therefore treat chemical additions as first-class inputs for:

- predicted swimmability
- maintenance readiness
- maintenance recommendations
- pool-specific learning

### Provider independence

The architecture should support multiple providers and sources for the same
kind of information.

Examples:

- manual chemistry tests
- weather providers
- controller telemetry
- sensor telemetry
- direct device integrations
- user estimates
- derived calculations

No scoring engine should hard-depend on one single provider family.

## Long-term scoring architecture

### Current swimmability score

Question answered:

> Can someone safely and comfortably swim right now?

Characteristics:

- driven primarily by current measured or directly observed conditions
- sensitive to chemistry, visible water condition, and freshness
- should represent current conditions only
- should not attempt to predict future chemistry behavior

Expected outputs:

- numeric score such as `0-100`
- rating band such as `Excellent`, `Good`, `Fair`, `Poor`, or `Unsafe`
- confidence score or confidence band
- explainable driver list

### Predicted swimmability score

Question answered:

> How likely is the pool to remain swimmable in the future?

Characteristics:

- forward-looking
- depends on forecast, trends, and expected chemistry change
- should evaluate future degradation or improvement rather than only current
  conditions
- should support multiple prediction horizons

Recommended horizons:

- `24 hours`
- `48 hours`
- `72 hours`
- `7 days`

Expected outputs:

- future score
- predicted trend
- confidence score
- horizon label
- explainable drivers

### Maintenance readiness score

Question answered:

> How much risk exists if no maintenance is performed?

Maintenance readiness is a forward-looking intervention-risk assessment. It is
intended to express how much operating margin remains before the pool should
reasonably receive maintenance, retesting, or treatment even if current
swimmability is still acceptable.

Characteristics:

- measures operational risk rather than current swim comfort alone
- remains independent from current swimmability
- identifies pools that are acceptable now but likely to deteriorate soon
- should be treated canonically as a band-first readiness or risk assessment
- should be presented primarily as a risk or urgency band
- may use an internal numeric index secondarily for sorting, thresholds, trend
  analysis, and later calibration, but the number should not be the primary
  user-facing concept in early generations
- should become one of the main inputs into future recommendation priority

Illustrative interpretation:

- Current Swimmability: `95`
- Predicted Swimmability (72 hr): `72`
- Maintenance Readiness: `40`

This means the pool is currently excellent but requires action soon.

Expected outputs:

- primary risk or urgency band
- explanation of risk drivers
- optional internal or secondary numeric index for ranking, sorting, and later
  calibration

### Data confidence score

Question answered:

> How trustworthy are the available conclusions?

Characteristics:

- evaluates data quality rather than pool quality
- should be computed independently from the pool-condition scores
- should always accompany scoring and recommendation results

Expected outputs:

- confidence band such as `High`, `Medium`, or `Low`
- explanation of uncertainty
- explanation of missing or stale inputs when relevant

## Swimmability input model

### Impact levels

Use these qualitative impact levels when designing or expanding models:

- `Major`
- `Moderate`
- `Minor`
- `None`

The table below defines initial design guidance rather than final weights.

### Input impact matrix

| Input | Current Impact | Predicted Impact |
| --- | --- | --- |
| Free Chlorine | Major | Major |
| Total Chlorine | Moderate | Minor |
| Combined Chlorine | Major | Moderate |
| pH | Major | Moderate |
| ORP | Moderate | Major |
| CYA | Moderate | Major |
| Total Alkalinity | Moderate | Moderate |
| Calcium Hardness | Minor | Minor |
| Water Temperature | Moderate | Moderate |
| Salt Level | Minor | Moderate |
| SWG State | Minor | Major |
| SWG Output | Minor | Major |
| Pump State | Minor | Moderate |
| Pump RPM | Minor | Moderate |
| Pump Runtime | Minor | Major |
| Flow Rate | Minor | Moderate |
| Filter Pressure | Minor | Moderate |
| Filter Condition | Minor | Moderate |
| Cover State | Moderate | Major |
| Cover Duration | Minor | Major |
| UV Index | Minor | Major |
| Air Temperature | Minor | Moderate |
| Water Temperature Trend | Minor | Moderate |
| Rainfall | Minor | Moderate |
| Rain Forecast | None | Major |
| Cloud Cover Forecast | None | Moderate |
| Wind | Minor | Minor |
| Sun Exposure Profile | Moderate | Major |
| Water Clarity | Major | Moderate |
| Algae Presence | Major | Major |
| Debris Level | Moderate | Minor |
| Bather Load | Minor | Major |
| Recent Chemical Additions | Moderate | Major |
| Phosphates | Minor | Moderate |
| Last Brushed | Minor | Moderate |
| Last Vacuumed | Minor | Moderate |
| Last Skimmer Cleaning | Minor | Minor |
| Test Freshness | Major | Major |
| Sensor Freshness | Major | Major |
| Data Source Reliability | Moderate | Major |
| Missing Data Count | Moderate | Major |

## Input tiers

### Tier 1

These are required for a meaningful swimmability assessment.

- Free Chlorine
- pH
- CYA
- Water Temperature
- Water Clarity
- Algae Presence
- Test Freshness
- Cover State

In the canonical near-term design, observational condition inputs such as
water clarity, algae presence, debris level, and bather load should be modeled
first as operator-entered qualitative assessments. Future sensor or
computer-vision inputs may supplement them later, but they are not required to
introduce the domain into Splash

Tier 1 inputs should be prioritized for:

- collection
- alerting
- confidence evaluation
- prominent UI visibility

### Tier 2

These strongly improve prediction quality.

- UV Index
- Rain Forecast
- Pump Runtime
- SWG Output
- Salt Level
- Recent Chemical Additions
- Bather Load
- Water Temperature Trend
- Historical Chlorine Consumption

Tier 2 inputs should be treated as highly valuable for future predictive work.

For the first telemetry-backed SWG slice:
- `Salt Level` remains the direct chemistry-adjacent measurement
- `SWG Output` should come from chlorinator latest-state telemetry
- `SWG State` should remain distinct from output percentage so prediction work
  can distinguish configured intent from active production

For the first telemetry-backed filter and flow slice:
- `Pump Runtime` remains separate from instantaneous `Flow Rate`
- `Flow Rate` should come from equipment latest-state telemetry rather than
  being invented from RPM alone
- `Filter Pressure` and `Filter Condition` should remain operational context
  inputs rather than chemistry values

### Tier 3

These are optimization and refinement inputs.

- ORP
- Flow Rate
- Filter Pressure
- Phosphates
- Wind
- Cloud Cover
- Filter Maintenance History
- Skimmer Maintenance History

ORP, phosphates, and borates should remain optional advanced inputs unless a
future design explicitly elevates them. Splash should use them when available
but should not assume they are present for meaningful current-state assessment

Tier 3 inputs should refine prediction and recommendations, but should not
block initial model implementation.

## Chemical modeling architecture

### Philosophy

Chemical additions are not merely chemistry corrections.

They are predictive events that alter future pool behavior.

Splash should therefore model chemical additions as:

- first-class historical events
- context-aware events whose effect depends on pool, weather, circulation, and
  current chemistry
- future inputs to prediction, recommendations, and learning

### Chemical additions and measurements are different domains

Splash should keep these separate:

- chemistry readings describe what the water measured
- chemical additions describe what the operator or equipment added

A future prediction or recommendation engine should consume both types of
history, but they must remain stored and explained separately.

## Chemical impact matrix

The following entries describe common pool-chemistry products and the design
questions that Splash should model for them.

### Liquid Chlorine

- Primary effect: raises free chlorine quickly
- Secondary effects: may slightly affect pH depending on local conditions and
  subsequent chlorine consumption
- Response curve: immediate increase followed by consumption curve
- Persistence: short to moderate depending on UV, CYA, temperature, cover, and
  demand
- Prediction value: very high for short-horizon FC prediction

### Cal-Hypo

- Primary effect: raises free chlorine
- Secondary effects: raises calcium hardness over time
- Response curve: immediate or near-immediate increase
- Persistence: short to moderate for FC, long-term accumulation for calcium
- Prediction value: high because repeated use changes both sanitizer and CH
  trajectory

### Trichlor

- Primary effect: raises chlorine through slow release
- Secondary effects: increases CYA and tends to lower pH
- Response curve: continuous release
- Persistence: sustained while product remains in feeder or tablet form
- Prediction value: very high because ongoing release affects future FC and CYA

### Dichlor

- Primary effect: raises chlorine
- Secondary effects: raises CYA
- Response curve: near-immediate increase
- Persistence: short for FC, longer-term for CYA accumulation
- Prediction value: high due to sanitizer plus stabilizer coupling

### Muriatic Acid

- Primary effect: lowers pH
- Secondary effects: lowers total alkalinity gradually depending on dose and
  aeration behavior
- Response curve: immediate to short-delay shift in pH
- Persistence: moderate, especially through TA reduction effects
- Prediction value: high for pH drift interpretation

### Dry Acid

- Primary effect: lowers pH
- Secondary effects: lowers TA; may introduce sulfate considerations depending
  on product
- Response curve: immediate to short-delay pH change
- Persistence: moderate
- Prediction value: moderate to high

### Soda Ash

- Primary effect: raises pH
- Secondary effects: raises TA
- Response curve: near-immediate
- Persistence: moderate
- Prediction value: high for pH trajectory and TA coupling

### Borax

- Primary effect: raises pH
- Secondary effects: may contribute to borate levels depending on usage
- Response curve: near-immediate
- Persistence: moderate
- Prediction value: moderate

### Baking Soda

- Primary effect: raises TA
- Secondary effects: smaller pH lift than soda ash
- Response curve: near-immediate with slower whole-pool equilibration
- Persistence: moderate to long
- Prediction value: moderate to high

### Stabilizer (CYA)

- Primary effect: raises cyanuric acid
- Secondary effects: changes chlorine protection and required FC targets
- Response curve: delayed ramp to final concentration
- Persistence: long
- Prediction value: very high because CYA strongly influences chlorine-loss and
  FC-target modeling

### Calcium Chloride

- Primary effect: raises calcium hardness
- Secondary effects: changes CSI or LSI behavior and scaling risk
- Response curve: delayed but generally monotonic increase
- Persistence: long
- Prediction value: moderate

### Salt

- Primary effect: raises salt concentration
- Secondary effects: enables or restores SWG effectiveness when low
- Response curve: delayed until fully dissolved and circulated
- Persistence: long
- Prediction value: high for SWG-enabled pools

For first-slice SWG telemetry modeling:
- `Salt Level` should remain a direct telemetry-backed measurement when the
  active chemistry source is chlorinator hardware
- `SWG Output` should be modeled as an operational telemetry input, not a
  chemistry value
- `SWG State` should be modeled separately from output percentage so prediction
  and readiness logic can distinguish active production from configured output
  level

### Borates

- Primary effect: raises borate concentration
- Secondary effects: may buffer pH drift and affect algae pressure depending on
  strategy
- Response curve: delayed to moderate
- Persistence: long
- Prediction value: moderate

### Phosphate Remover

- Primary effect: reduces phosphate concentration
- Secondary effects: may change filter loading and cloudiness temporarily
- Response curve: delayed with threshold-like operational effects
- Persistence: moderate to long depending on reintroduction sources
- Prediction value: moderate

### Clarifier

- Primary effect: improves particle aggregation and clarity support
- Secondary effects: may alter filter loading
- Response curve: delayed ramp
- Persistence: short to moderate
- Prediction value: moderate for condition prediction, low for chemistry-only
  prediction

### Flocculant

- Primary effect: causes suspended matter to settle for later removal
- Secondary effects: requires compatible operational workflow such as vacuum to
  waste
- Response curve: delayed then step-like effect after cleanup
- Persistence: short-lived treatment window
- Prediction value: moderate when coupled with maintenance events

### Algaecide

- Primary effect: inhibits or suppresses algae depending on type and context
- Secondary effects: can affect chlorine demand and water appearance depending
  on product
- Response curve: product-dependent, often delayed
- Persistence: moderate
- Prediction value: moderate to high when algae risk is part of the model

### Enzymes

- Primary effect: reduce certain non-living organic contaminants
- Secondary effects: may indirectly reduce sanitizer demand or scum pressure
- Response curve: gradual
- Persistence: moderate
- Prediction value: low to moderate until strong pool-specific evidence exists

## Chemical effect curves

Splash should support several effect-curve categories rather than assuming all
chemicals behave linearly.

Supported curve types:

- `Linear`
- `Mostly Linear`
- `Non-Linear`
- `Diminishing Return`
- `Threshold`
- `Continuous Release`
- `Delayed Ramp`
- `Step Function`
- `Step Function With Decay`

Examples:

- Liquid chlorine:
  - immediate increase
  - followed by a consumption curve
- Stabilizer:
  - delayed increase
  - slowly reaches final concentration
- Trichlor:
  - continuous release over time
- SWG:
  - continuous production while active

The exact implementation mathematics may change over time, but the
architecture should assume that chemical-effect models are product-specific and
context-sensitive.

## Chemical interaction matrix

Future prediction models must consider interactions and not evaluate all
chemical effects independently.

Important interaction examples:

- Chlorine <-> CYA
- Chlorine <-> UV
- Chlorine <-> Water Temperature
- Chlorine <-> Bather Load
- Chlorine <-> Algae
- Acid <-> TA
- Soda Ash <-> TA
- Salt <-> SWG
- Cover State <-> UV
- Rain <-> Water Chemistry

These interactions should eventually shape:

- chlorine burn rate
- sanitizer demand models
- pH drift behavior
- maintenance recommendations
- confidence in future projections

## Required context for chemical modeling

### Pool context

Chemical modeling should have access to:

- pool volume
- pool type
- surface type
- shape
- features
- average depth
- waterline area

### Product context

For a logged chemical addition, Splash should be able to represent:

- chemical type
- active ingredient
- strength
- concentration
- physical form
- available chlorine when relevant
- purity
- secondary ingredients when known

Canonical taxonomy guidance:

- Splash should prefer a type-based product taxonomy rather than a parent/child
  product-profile hierarchy in the first generations of the chemistry model
- materially different strengths or formulations may be represented as distinct
  chemical types when that keeps prediction and dosing logic simpler
- for example, `10% liquid chlorine`, `12.5% liquid chlorine`, and `15% liquid
  chlorine` may be modeled as distinct types rather than one parent type with a
  required child-strength attribute

### Dosing context

Each chemical-addition event should eventually support:

- amount
- unit
- timestamp
- addition method
- addition duration
- addition location
- pump state
- pump runtime after addition
- cover state

### Chemistry context

Prediction and recommendation engines should be able to consider:

- FC
- TC
- CC
- pH
- TA
- CYA
- CH
- Salt
- Borates
- Phosphates
- ORP
- Water Temperature
- CSI or LSI

### Circulation context

Prediction and recommendation engines should be able to consider:

- pump state
- RPM
- flow
- return configuration
- cleaner state
- turnover estimate
- derived runtime summaries across recent windows
- sample coverage and data sufficiency for those runtime summaries

Circulation modeling guidance:
- first derive runtime summaries from real telemetry
- preserve uncertainty when telemetry coverage is sparse
- avoid pretending that configured schedules equal actual circulation
- defer turnover estimates until flow and pool-volume confidence are strong

### Environmental context

Prediction and recommendation engines should be able to consider:

- UV Index
- sun exposure
- air temperature
- water temperature
- rainfall
- rain forecast
- wind
- cloud cover
- cover state
- cover duration
- derived covered and daylight-uncovered summaries across recent windows
- explicit sufficiency metadata for those summaries when cover history is weak

Cover-exposure modeling guidance:
- derive cover-duration context from immutable cover-event history rather than
  inferring it from the latest cover state alone
- treat daylight-uncovered duration as the first conservative exposure proxy
  before later UV-weighted models exist
- do not present first-slice cover exposure summaries as exact UV dose

## Chemical event model

Chemical additions should be stored as immutable historical events.

Splash should preserve:

- what was added
- how much was added
- when it was added
- how it was added
- pool conditions at time of addition when available
- confidence level of the recorded event

Maintenance activities should also be stored as immutable historical events. Actions such as brushing, vacuuming, robot cleaning, skimming, and filter cleaning are not chemistry corrections themselves, but they are important contextual inputs for maintenance readiness, clarity interpretation, recommendation logic, and later pool-specific learning.

The platform should preserve the distinction between:

- measured values
- predicted values
- user-entered values
- derived values

These should never be collapsed into one undifferentiated chemistry record.

## Derived metrics

### Estimated chlorine burn rate

Purpose:

Estimate expected chlorine loss over time.

Potential inputs:

- CYA
- UV
- Cover State
- Water Temperature
- Historical Consumption
- Bather Load

This metric should be treated as one of the most important future prediction
inputs.

### Historical chlorine demand

Purpose:

Represent a pool-specific estimate of sanitizer consumption over time.

This metric should learn from actual pool history rather than generic pool math
alone.

### pH drift rate

Purpose:

Estimate natural pH movement over time.

### Predicted FC

Purpose:

Estimate future free chlorine.

Predicted FC should never replace measured FC. It is a forecast-only value.

### Predicted pH

Purpose:

Estimate future pH.

Predicted pH should never replace measured pH. It is a forecast-only value.

## Confidence architecture

Confidence scoring should remain separate from water-quality scoring.

Confidence should consider:

- missing inputs
- stale inputs
- unknown pool volume
- unknown product strength
- unknown dosage
- sensor failures
- conflicting measurements
- missing weather data
- missing historical data

A pool can have:

- excellent water quality with low confidence
- poor water quality with high confidence

These are separate concepts and should be presented separately.

### Contradiction and low-confidence alerting

The platform should emit alerts when trust in a conclusion is reduced by
provenance problems or explicit contradictions.

Examples:
- a current swimmability result appears positive, but chemistry freshness is
  stale
- water temperature is shown as part of the current context, but telemetry is
  unavailable
- a derived metric depends on incomplete upstream values

These alerts should be treated as:
- trust and explainability outputs
- separate from raw water-quality scoring
- inputs to operator action, retesting, and future recommendation flows

## Historical learning architecture

Splash should eventually support pool-specific learning.

Examples of learnable behavior:

- daily chlorine loss
- covered chlorine loss
- uncovered chlorine loss
- seasonal demand
- heat-wave demand
- rainfall impacts
- pH drift
- SWG production effectiveness

The purpose of historical learning is to let Splash predict a specific pool's
behavior more accurately than a generic pool calculator.

Learned pool behavior should supplement, not erase, generic chemistry
principles.

## Environmental modifier architecture

Splash should support configurable modifier systems rather than burying all
environmental influence in hard-coded rules.

Examples:

- UV modifier
- temperature modifier
- cover modifier
- rain modifier
- bather-load modifier

Modifier systems should be configurable and explainable.

## Recommended development phases

### Phase 1

Build:

- Current Swimmability
- Confidence Score
- Stale Data Detection

### Phase 2

Build:

- Chemical Event History
- Chlorine Burn Rate
- Predicted Swimmability

### Phase 3

Build:

- Maintenance Readiness
- Recommendation Engine
- Historical Trend Analysis

### Phase 4

Build:

- Pool-specific learning
- sensor-driven prediction
- direct RS-485 integration
- adaptive prediction models

## Implementation guidance

Future implementation work should:

- preserve measured, derived, and predicted distinctions
- keep confidence separate from pool-quality scores
- keep current-state scoring separate from future-state prediction
- treat chemical additions as predictive historical events
- use explainable rules and model outputs
- support multiple providers for the same data type
- keep thresholds and modifiers configurable where practical
- avoid assuming that generic chemistry calculators are sufficient for a
  specific pool without historical calibration

## Areas requiring future refinement

Future design work should still define:

- concrete scoring formulas
- normalization rules for mixed-source values
- confidence algorithms
- prediction algorithms
- recommendation rules
- alert rules for contradiction and low confidence
- chlorine-consumption modeling details
- seasonal tuning behavior
- provider-reliability weighting
- pool-volume-aware dosing calculations
- product-strength normalization rules

This page establishes the canonical architecture and terminology for those
future documents.
