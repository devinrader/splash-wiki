# Operations and Verification

[Back to README](Home)

## Purpose

This document captures testing, observability, and operational guidance that supports the architecture but should not overload the core architecture narrative.

## Testing strategy

- Unit tests cover rule evaluation, protocol decoder logic, chemistry calculations, and API handlers
- Integration tests cover API access to PostgreSQL and InfluxDB, scheduler jobs, and SSE broker behavior
- Integration tests should also cover raw-stream to decoded-event flow through `splash-protocol`
- Hardware-in-the-loop checks validate real raw-byte capture, command round-trip behavior, and adapter reconnect handling
- `splash-serial` tests should cover stream-id rollover, stale write rejection, native read-boundary publishing, and idle-timing enforcement

## Logging and metrics

- services should use structured JSON logging appropriate to their runtime
- ASSUMPTION: TypeScript services will use a structured logger such as `pino`, while `splash-serial` will use `zerolog`
- Docker log rotation is required to avoid unbounded growth on embedded storage
- Prometheus scrapes service metrics
- Grafana is the recommended operator-facing visualization layer for Prometheus metrics, but it should remain a visualization tool rather than a browser-facing application API dependency
- the most valuable first alert is RS-485 frame rate dropping to zero
- the first platform metrics slice should focus on RS-485 connectivity and message activity before adding broader broker or host dashboards
- `splash-serial` should expose Prometheus metrics for connection state, reconnect count, bytes read, bytes written, write failures, and current stream age
- `splash-serial` should also expose RS-485 message counters so Prometheus can derive receive and transmit rates from monotonic counters
- `splash-serial` should expose `GET /healthz` and `GET /metrics` over its local HTTP listener
- `splash-serial` should also expose `GET /readyz` and `GET /health` for
  semantic service aggregation
- `splash-protocol` should expose `GET /healthz` and `GET /metrics` over its local HTTP listener
- `splash-protocol` should also expose `GET /readyz` and `GET /health` for
  semantic service aggregation
- `splash-protocol` should expose metrics for frame decode success or failure, normalized event publication, command-result status, and command-correlation timeouts
- `splash-api` should remain the browser-visible aggregation boundary for dashboard-facing health and connectivity summaries even when Grafana is present for operator diagnostics
- `splash-api` should expose `GET /platform/status` as the authoritative
  browser-facing semantic health source
- Prometheus and Grafana should remain operator observability tools, not the
  frontend's direct health authority

## Backup and recovery

- PostgreSQL is backed up daily with `pg_dump`
- InfluxDB is backed up weekly
- backup retention target is 30 daily PostgreSQL backups and 4 weekly InfluxDB backups
- off-device copy is recommended but manual in v1
- Ansible-based restore is the intended disaster-recovery path

## Operational notes

- Prometheus metrics are included in the architecture, while Grafana is recommended but not part of the base v1 topology
- when Grafana is present, it should read from Prometheus for operator dashboards rather than becoming the frontend's direct metrics backend
- mDNS reliability should be validated on the target LAN, with reserved-IP fallback documented if needed
