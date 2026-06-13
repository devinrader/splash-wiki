# Swimmability Design Guidance

[Back to Product](Product)

## Purpose

This page defines the canonical design guidance for how Splash should think
about swimmability-related scoring, prediction, recommendations, confidence,
staleness, and alerting.

This page should be read together with the deeper
[Pool Chemistry, Swimmability, and Prediction Modeling Specification](Product-Pool-Chemistry-Swimmability-and-Prediction-Modeling) when designing future scoring or recommendation systems.

This page is not the implementation contract for the current algorithm. It is
the higher-level architecture guidance that future design and implementation
work should follow.

For the current implemented first-slice score, see
[Swimmability Scoring](Product-Swimmability-Scoring).

For the detailed long-term modeling architecture covering chemistry,
prediction, maintenance readiness, chemical-event modeling, confidence, and
historical learning, see
[Pool Chemistry, Swimmability, and Prediction Modeling Specification](Product-Pool-Chemistry-Swimmability-and-Prediction-Modeling).

## Core purpose

Splash should ultimately provide three related capabilities:

1. Assess current swim readiness.
2. Predict future swim readiness.
3. Recommend maintenance actions that improve or preserve future swim
   readiness.

Splash should be able to explain:

- why a score was assigned
- which inputs contributed to it
- how trustworthy those inputs are
- which missing or stale inputs limit the result

## Guiding principle

Splash should ultimately answer four questions:

1. Can I swim now?
2. Can I swim tomorrow?
3. What should I do next?
4. How confident is Splash in those answers?

All future scoring, prediction, recommendation, alerting, and automation work
should align with these four questions.

## Recommended scoring architecture

Splash should not be modeled as one monolithic swimmability score.

Instead, the platform should maintain several independent but related scoring
or assessment models.

### 1. Current swimmability score

Question answered:

> Can someone safely and comfortably swim in the pool right now?

Characteristics:

- prioritizes current measured conditions
- is primarily driven by measured chemistry, visible condition, and freshness
- represents current conditions only
- should not try to predict future conditions

Example inputs:

- Free Chlorine
- pH
- Water Clarity
- Algae Presence
- Water Temperature
- Test Freshness

Outputs should include:

- numeric score
- qualitative rating such as `Excellent`, `Good`, `Fair`, `Poor`, or `Unsafe`
- confidence score
- explanation of the strongest drivers

### 2. Predicted swimmability score

Question answered:

> How likely is the pool to remain swimmable over the next several hours or
> days?

Characteristics:

- forward-looking
- incorporates forecast, trends, and expected chemistry change
- uses equipment and cover context where available
- supports multiple prediction horizons

Recommended horizons:

- 24 hours
- 48 hours
- 72 hours
- 7 days

Example inputs:

- Current Swimmability Score
- Chlorine Burn Rate
- UV Forecast
- Rain Forecast
- Cover State
- Pump Runtime
- SWG Output
- Water Temperature Trend
- Historical Consumption Patterns

Pump runtime guidance:
- use derived recent circulation summaries rather than assuming configured
  schedules equal actual runtime
- preserve explicit coverage and sufficiency metadata so prediction work can
  distinguish strong runtime evidence from sparse telemetry

Cover exposure guidance:
- use derived cover-exposure summaries from persisted cover events rather than
  assuming the current cover state describes the whole recent window
- preserve explicit `partial` or `insufficient_data` exposure states when
  history is sparse
- treat daylight-uncovered time as a first proxy for exposure before later UV
  weighting models exist

Outputs should include:

- future score
- predicted trend
- confidence score
- time horizon
- explanation of the strongest drivers
- explicit assumptions and missing-input constraints

First-slice prediction contract guidance:
- expose predicted swimmability through a dedicated read model rather than
  folding it into the current swimmability endpoint
- support first horizons of `24h`, `48h`, `72h`, and `7d`
- allow `unknown` or low-confidence prediction outputs when the available data
  is too thin to support a credible forecast
- require show-your-work driver and provenance fields from the first slice

First-slice prediction contract guidance:
- expose predicted swimmability through a dedicated read model rather than
  folding it into the current swimmability endpoint
- support first horizons of `24h`, `48h`, `72h`, and `7d`
- allow `unknown` or low-confidence prediction outputs when the available data
  is too thin to support a credible forecast
- require show-your-work driver and provenance fields from the first slice

### 3. Maintenance readiness score

Question answered:

> How much risk exists if no maintenance is performed?

Characteristics:

- measures operational risk
- remains independent from current swimmability
- identifies pools that are acceptable now but likely to deteriorate soon

Example interpretation:

- Current Swimmability: `95`
- Predicted Swimmability (72 hr): `72`
- Maintenance Readiness: `40`

This means the pool is currently excellent but likely needs intervention soon.

Example inputs:

- Time since testing
- Chlorine consumption rate
- Equipment runtime
- Weather forecast
- Filter condition
- Cover usage
- Recent maintenance history

Outputs should include:

- primary risk or urgency band
- explanation of the major risk drivers
- optional internal or secondary numeric index when useful for ranking or
  later tuning

### 4. Data confidence score

Question answered:

> How trustworthy are the calculated results?

Characteristics:

- evaluates data quality rather than pool quality
- should be calculated independently from pool-condition scores
- must remain visible whenever Splash presents scoring or recommendations

Confidence should be influenced by:

- input freshness
- input completeness
- source reliability
- contradictory values
- sensor health
- forecast quality
- historical-data sufficiency when prediction depends on it

Outputs should include:

- confidence band such as `High`, `Medium`, or `Low`
- explanation of uncertainty

Splash should clearly distinguish between:

> The pool appears unsafe

and

> The pool may be fine, but the available data is not trustworthy enough

These are separate concepts.

## Initial development guidance

Splash should evolve in phases.

### Phase 1

Build:

- Current Swimmability Score
- Data Confidence Score
- Stale Data Detection

Goal:

Provide reliable current-state assessment without overreaching into prediction.

### Phase 2

Add:

- Chlorine Burn Rate model
- stronger weather integration
- Predicted Swimmability Score

Goal:

Answer:

> Will my pool still be swimmable tomorrow?

### Phase 3

Add:

- Maintenance Readiness Score
- Recommendation Engine
- historical trend analysis

Goal:

Answer:

> What should I do today to keep my pool swimmable for the next several days?

### Phase 4

Add:

- sensor-driven chemistry inputs
- direct RS-485 equipment integration
- adaptive scoring models
- seasonal tuning
- provider reliability weighting

Goal:

Transition from a monitoring platform into a predictive pool-management
platform.

## Maintenance recommendations

Question answered:

> What actions should the owner take to improve or preserve swimmability?

Recommendations should be explainable and tied to observable inputs.

Examples:

- add chlorine
- adjust pH
- run pump longer
- clean filter
- empty skimmer baskets
- brush pool
- vacuum pool
- enable chlorination
- retest stale chemistry values
- investigate sensor failures

Recommendation systems should:

- use the current, predicted, readiness, and confidence models together rather
  than collapsing them into one score
- remain a separate read model rather than being folded into current or
  predicted swimmability responses
- distinguish urgent safety actions from lower-priority optimization actions
- avoid hiding uncertainty when input quality is weak
- prioritize `retest` guidance ahead of chemistry-adjustment guidance when
  chemistry freshness or trust is weak
- distinguish:
  - corrective recommendations
  - preventive recommendations
  - investigative recommendations
- generate trust-focused alerts when:
  - critical inputs are missing or stale
  - confidence is low
  - important provenance conditions contradict a positive-looking conclusion

Contradiction and low-confidence alerts should:
- explain what input is missing, stale, unavailable, or contradictory
- explain why that limitation matters
- distinguish uncertain conclusions from poor pool conditions
- favor explicit, show-your-work warning semantics over silent score mutation

First-slice maintenance recommendations should focus on explainable rule-based
operator guidance such as:

- retest free chlorine and pH
- add chlorine
- adjust pH
- run pump longer or check circulation
- use or remove the cover based on current context
- brush or vacuum the pool
- clean or inspect the filter
- investigate missing telemetry or stale inputs
- wait and monitor

First-slice maintenance recommendations should not yet:

- calculate exact chemical doses
- send direct control commands
- hide missing-data limitations behind confident-sounding advice

## Input impact matrix

The following matrix is design guidance only. It is not a final weighting
table.

| Input | Current Score Impact | Predicted Score Impact |
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
| SWG Output Percentage | Minor | Major |
| Pump Running State | Minor | Moderate |
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
| Bather Load Estimate | Minor | Major |
| Recent Chemical Additions | Moderate | Major |
| Phosphates | Minor | Moderate |
| Last Brushed | Minor | Moderate |
| Last Vacuumed / Robot Cleaned | Minor | Moderate |
| Last Skimmer Cleaning | Minor | Minor |
| Test Freshness | Major | Major |
| Sensor Freshness | Major | Major |
| Data Source Reliability | Moderate | Major |
| Confidence Score | Major | Major |
| Missing Data Count | Moderate | Major |

## Input tiers

### Tier 1: Core inputs

These are the highest-priority inputs for meaningful swimmability assessment.

#### Chemistry

- Free Chlorine
- pH
- CYA

#### Environmental

- Water Temperature

#### Condition assessment

- Water Clarity
- Algae Presence

#### Data quality

- Test Freshness

#### Pool state

- Cover State

These inputs should be prioritized for:

- data collection
- freshness tracking
- alerting
- user-interface visibility

### Tier 2: Prediction inputs

These inputs materially improve future-score quality.

- UV Index
- Rain Forecast
- Pump Runtime
- SWG Output
- Salt Level
- Recent Chemical Additions
- Bather Load
- Water Temperature Trend
- Historical Chlorine Consumption

These should be treated as required for advanced predictive modeling.

For the first telemetry-backed SWG slice:
- `Salt Level` remains the direct chemistry-adjacent measurement
- `SWG Output` should come from chlorinator latest-state telemetry
- `SWG State` should remain distinct from output percentage so prediction work
  can distinguish configured intent from active production

For the first telemetry-backed filter and flow slice:
- `Pump Runtime` remains separate from instantaneous `Flow Rate`
- `Flow Rate` should come from equipment latest-state telemetry, not be
  invented from RPM alone
- `Filter Pressure` and `Filter Condition` should remain operational context
  inputs rather than direct chemistry values

### Tier 3: Optimization inputs

These inputs refine the model but should not block early implementation.

- ORP
- Flow Rate
- Filter Pressure
- Phosphates
- Wind
- Cloud Cover
- Filter Maintenance History
- Skimmer Maintenance History

## Data quality requirements

Every tracked value should support:

- current value
- timestamp
- value kind
- data source
- freshness state
- reliability or confidence indicator
- short explainable reason strings when confidence is reduced

Recommended source types:

- Manual Test
- Sensor
- Weather Provider
- Controller
- Direct Device Integration
- Derived Calculation
- User Estimate

Recommended freshness states:

- Fresh
- Aging
- Stale
- Missing
- Unavailable
- Estimated

## Derived metrics

Splash should support calculated metrics in addition to raw measurements.

### Estimated chlorine burn rate

Purpose:

Estimate expected daily chlorine loss.

Potential inputs:

- CYA
- UV forecast
- Cover state
- Cover duration
- Water temperature
- historical chlorine consumption
- bather load

Example questions enabled:

- Will free chlorine remain acceptable tomorrow?
- How much chlorine should be added tonight?
- Is chlorine loss unusually high?

This derived metric is expected to become a major input into future prediction
models.

## Architectural guidance

The broader swimmability architecture should support:

- manual test entry
- weather providers
- sensor telemetry
- pool-controller integration
- direct RS-485 equipment control
- multiple providers for the same data type
- explainable recommendations
- confidence-aware recommendations
- user-adjustable thresholds where product-safe

The scoring engine should avoid hard dependencies on any single provider.

All recommendations should be traceable to specific inputs and rules.

The scoring architecture should prefer:

- normalized inputs over provider-specific payloads
- reusable freshness and confidence services
- separated current-state, predictive, and recommendation logic
- explainable driver lists over opaque combined heuristics

## Future documentation requirements

Follow-up design work should define:

- score calculation formulas
- input normalization rules
- confidence algorithms
- prediction algorithms
- maintenance recommendation rules
- alert generation rules
- chlorine-consumption modeling
- seasonal adjustments
- provider reliability weighting

This page is foundational design guidance for future scoring-model work rather
than a narrow feature request.
