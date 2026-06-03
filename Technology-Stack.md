# Technology Stack

[Back to README](Home)

## Summary

Splash uses a mixed stack chosen by service responsibility, deployment
constraints, and data shape.

## Application languages

- `splash-api`: TypeScript / Node.js
- `splash-scheduler`: TypeScript / Node.js
- `splash-protocol`: TypeScript / Node.js
- `splash-frontend`: TypeScript / React / Vite
- `splash-serial`: Go

## Data and storage

- SQLite for embedded relational storage on `splash-core`
- InfluxDB for time-series telemetry

## Messaging and streaming

- NATS as the event backbone
- JetStream for durable event workflows where message loss would create missed
  actions

## Observability

- Prometheus for service metrics scraping
- SSE for primary browser-side live updates

## Provisioning and operations

- Ansible for provisioning and disaster recovery
- containerized application services on `splash-core`
- `systemd`-managed native `splash-serial` on `splash-zero`

## Remote access direction

- Cloudflare Tunnel and Cloudflare Access are the planned v2 remote-access
  path, not part of the v1 baseline trust boundary
