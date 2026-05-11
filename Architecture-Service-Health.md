# Service Health Architecture

[Back to README](Home)

## Purpose

This document defines the canonical live health model for Splash platform
services and third-party operator infrastructure.

It exists to ensure:

- browser-visible platform health is sourced from one authoritative API
- service status uses a richer semantic model than binary up or down
- frontend health rendering is decoupled from direct service probing
- Prometheus and Grafana remain observability tools rather than the frontend's
  health authority

## Canonical statuses

Every browser-visible service health record must normalize to one of:

- `healthy`
- `degraded`
- `unhealthy`
- `down`
- `unknown`

Definitions:

- `healthy`: the service is reachable and can fully perform its intended role
- `degraded`: the service is reachable and can still perform its primary role,
  but non-critical dependencies, optional capabilities, or quality indicators
  are impaired
- `unhealthy`: the service is reachable but cannot meaningfully perform its
  primary role
- `down`: the service cannot be reached, timed out, refused connection, or is
  not running
- `unknown`: the service has not been evaluated yet, or the latest data is
  stale or insufficient to classify safely

## Evaluation rubric

Every service definition should document:

1. primary responsibility
2. required dependencies
3. optional dependencies
4. quality indicators
5. what makes the service reachable
6. what makes the service ready
7. what makes the service unable to do its job

Status evaluation rules:

1. if the service cannot be reached, mark it `down`
2. else if it is reachable but cannot perform its primary responsibility, mark
   it `unhealthy`
3. else if it can perform its primary responsibility but optional
   dependencies, quality indicators, or freshness are impaired, mark it
   `degraded`
4. else if all required dependencies and quality checks pass, mark it
   `healthy`
5. else mark it `unknown`

## Service criticality

Each service must declare one platform criticality:

- `critical`
- `important`
- `optional`

Criticality affects platform rollup:

- `critical` service `down` or `unhealthy` should make the platform
  `unhealthy`
- `important` service `down` or `unhealthy` should usually make the platform
  `degraded` unless the service is required for the active operator workflow
- `optional` service `down` or `unhealthy` should usually degrade diagnostics
  rather than core platform operation

## Overall platform rollup

The platform health rollup order is:

`down > unhealthy > degraded > unknown > healthy`

However, rollup must be criticality-aware rather than a blind max severity.

Rules:

- if any `critical` service is `down` or `unhealthy`, overall platform status
  is `unhealthy`
- else if any `critical` or `important` service is `degraded`, overall status
  is at least `degraded`
- else if only `optional` services are `down` or `unhealthy`, overall status
  should normally be `degraded`
- else if all evaluated services are `healthy`, overall status is `healthy`
- else if evaluation is incomplete or stale and no stronger condition applies,
  overall status is `unknown`

ASSUMPTION: the initial platform rollup should not emit top-level `down`; a
critical dependency that is unreachable should instead make the aggregated
platform state `unhealthy` while the per-service record remains `down`.

## Browser-visible authority

`splash-api` is the authoritative browser-visible health aggregator.

The frontend must not poll every platform service directly for live health.

The frontend should call:

- `GET /platform/status`

from `splash-api`.

`splash-api` should:

- poll Splash-owned health endpoints
- poll or probe third-party health targets
- normalize all results to the canonical health model
- retain recent health snapshots for stale-data fallback
- expose the aggregated result through REST and Prometheus metrics

## Required service-local endpoints

Splash-owned runtime services should standardize these endpoints where
applicable:

- `GET /healthz`
- `GET /readyz`
- `GET /health`
- `GET /metrics`

Endpoint intent:

- `/healthz`: liveness only; return `200` if the process is alive
- `/readyz`: readiness only; return `200` only when the service can perform its
  primary role
- `/health`: rich semantic health JSON for machine aggregation and operator
  diagnostics
- `/metrics`: Prometheus metrics

Rules:

- `/healthz` should not fail merely because an optional dependency is degraded
- `/readyz` should fail when the service cannot perform its primary role
- `/health` should include enough structured data for `splash-api` to classify
  the service without inferring hidden meaning from log text
- `/metrics` remains for time-series observability, not the browser health
  authority

## Aggregated platform status contract

`GET /platform/status` should return a canonical response shaped like:

```json
{
  "overall": "degraded",
  "generatedAt": "2026-05-11T18:00:00.000Z",
  "services": [
    {
      "name": "splash-serial",
      "type": "splash",
      "criticality": "critical",
      "status": "degraded",
      "message": "NATS unavailable; serial port is connected",
      "lastChecked": "2026-05-11T18:00:00.000Z",
      "responseTimeMs": 43,
      "checks": {
        "process": {
          "status": "healthy"
        },
        "serialPort": {
          "status": "healthy",
          "message": "Connected to /dev/ttyUSB0"
        },
        "nats": {
          "status": "unhealthy",
          "message": "Unable to connect to NATS"
        }
      }
    }
  ]
}
```

Each service record should include:

- `name`
- `type`
- `criticality`
- `status`
- `message`
- `lastChecked`
- `responseTimeMs`
- `checks`
- optional `raw` diagnostic content for debug mode

## Central service registry

`splash-api` should maintain a central registry describing platform services and
their health adapters.

Example:

```json
[
  {
    "name": "nats",
    "type": "third-party",
    "criticality": "critical",
    "healthUrl": "http://nats:8222/varz",
    "tcp": "nats:4222"
  },
  {
    "name": "splash-serial",
    "type": "splash",
    "criticality": "important",
    "healthUrl": "http://splash-serial:3000/health"
  }
]
```

The initial registry should cover:

- `grafana`
- `prometheus`
- `nats`
- `splash-api`
- `splash-serial`
- `splash-frontend`
- `splash-protocol`

## Health adapters

`splash-api` should support adapter-specific checkers for:

- Splash-native `/health` JSON
- simple HTTP liveness or readiness probes
- TCP reachability probes
- Prometheus health and target checks
- Grafana health and datasource checks
- NATS TCP and monitoring checks

Rules:

- each check should use a short timeout such as `2s`
- a timed-out probe should normally classify the target as `down`
- stale cached data may downgrade the confidence of the classification to
  `unknown` when a fresh verdict cannot be made safely

## Staleness handling

Each service record should retain recent successful results.

Rules:

- if no check has completed yet, service status is `unknown`
- if the last successful result is too old and a fresh check cannot complete,
  the service should become `unknown` unless current evidence clearly indicates
  `down`
- the aggregated response should expose enough timestamp data for the frontend
  to warn when it is rendering stale cached status

## Platform service rubric

### NATS

- primary responsibility: provide the platform message bus
- criticality: `critical`
- healthy:
  - TCP client connections succeed
  - monitoring endpoint responds when configured
  - no obvious server error state is present
- degraded:
  - messaging works but monitoring suggests partial impairment
  - latency or server-state signals are abnormal but publish or subscribe still
    works
- unhealthy:
  - reachable but messaging cannot be used correctly
  - probe publish or subscribe path fails
- down:
  - TCP connection is refused or times out

### splash-api

- primary responsibility: serve frontend API traffic and aggregate platform
  health
- criticality: `critical`
- healthy:
  - `/healthz`, `/readyz`, `/health`, and `/platform/status` work
  - required dependencies are healthy
- degraded:
  - API can still serve core frontend paths
  - optional diagnostics or some downstream health collection is impaired
- unhealthy:
  - API is reachable but cannot serve primary browser-facing routes or
    aggregate platform status
- down:
  - API is unreachable

### splash-serial

- primary responsibility: connect to the configured RS-485 interface, read and
  write controller traffic, and publish serial events to NATS
- criticality: `critical` for live controller deployments
- healthy:
  - process alive
  - serial port open
  - NATS connected
  - read and write path functioning
- degraded:
  - serial connected but NATS unavailable
  - NATS connected but traffic stale beyond a warning threshold
  - error rate elevated but traffic still flows
- unhealthy:
  - serial port cannot open
  - required NATS publication path is unavailable
  - read loop or write path is broken
- down:
  - service unreachable

### splash-protocol

- primary responsibility: consume raw transport events, decode protocol
  messages, and publish normalized platform events
- criticality: `important`
- healthy:
  - reachable
  - NATS connected
  - decoders loaded
  - decode and publish path works
- degraded:
  - non-critical parser or plugin is impaired
  - unknown-frame or latency rate is elevated while decode still works
- unhealthy:
  - cannot connect to NATS
  - no decoders loaded
  - cannot publish normalized events
- down:
  - service unreachable

### splash-frontend

- primary responsibility: serve the browser UI
- criticality: `important`
- healthy:
  - frontend assets load
  - app shell initializes
  - `splash-api /platform/status` can be queried
- degraded:
  - frontend loads but API or diagnostics features are partially impaired
- unhealthy:
  - frontend responds but the app shell cannot initialize meaningfully
- down:
  - frontend unreachable

### Prometheus

- primary responsibility: collect and store operational metrics
- criticality: `optional`
- healthy:
  - UI or API reachable
  - `/-/healthy` reports healthy
  - config loads
  - storage writable
- degraded:
  - scrape targets are failing but Prometheus itself still works
  - latency or storage pressure is elevated
- unhealthy:
  - reachable but config or storage failure prevents useful metrics collection
- down:
  - unreachable

### Grafana

- primary responsibility: render operator dashboards
- criticality: `optional`
- healthy:
  - UI or API reachable
  - datasource to Prometheus works
  - provisioned dashboards load
- degraded:
  - reachable but datasource or some dashboards are impaired
- unhealthy:
  - reachable but core dashboard loading or persistence is broken
- down:
  - unreachable

## Frontend behavior

The frontend should render only the aggregated `splash-api` view.

Rules:

- if `splash-api` is healthy, show live platform status
- if `splash-api` is degraded, show the dashboard with a degraded banner and
  detailed service cards
- if `splash-api` is unhealthy, show a warning that current platform status may
  be incomplete and render last-known cached status when available
- if `splash-api` is down, show `Splash API unavailable`, retry automatically,
  and render last-known cached status with a stale timestamp when available
- cache the most recent successful `GET /platform/status` response in browser
  local storage or equivalent client storage

## Aggregator metrics

`splash-api` should expose Prometheus metrics for health aggregation, including:

- `splash_platform_service_health{service="<name>",status="<status>"}`
- health-check duration
- health-check failures
- last successful check timestamp

## Initial deployment notes

The initial aggregator configuration should support:

- Prometheus URL: `http://prometheus.rader.haus`
- Grafana URL: `http://grafafa.rader.haus`

ASSUMPTION: the provided Grafana hostname is intentional and should be treated
as the configured endpoint until explicitly corrected.
