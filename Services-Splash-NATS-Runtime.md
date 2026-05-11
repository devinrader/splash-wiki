# Runtime Model

[Back to Splash NATS Service](Services-Splash-API-Home)

## Process model

`splash-nats` runs on `splash-core` as a containerized infrastructure service.

It should:

- start under Docker Compose
- be deployed from a versioned artifact or image rather than a host-local ad hoc build
- bind to the LAN so `splash-zero` can connect
- provide both Core NATS and JetStream in the same service instance for v1

## Platform role

`splash-nats` is the integration boundary between Splash services.

Core NATS is used for:

- high-frequency transport events
- protocol decode outputs
- live state propagation

JetStream is used for:

- automation suggestions
- notifications
- tasks and other events where durability matters

## Runtime dependencies

The runtime model assumes:

- `splash-core` is reachable on the LAN
- local storage is available for JetStream state
- clients can reconnect after transient outages

## Service relationship summary

Primary clients:

- `splash-api`
- `splash-scheduler`
- `splash-protocol`
- `splash-serial`

Operational consumer:

- platform operators using logs, health, and metrics for troubleshooting

## Monitoring role

The existing NATS monitoring listener should remain the authoritative source
for broker-wide NATS traffic statistics.

Rules:

- Splash services may consume NATS monitoring data such as `/varz` to derive
  broker-level inbound and outbound message rates for operator-facing
  diagnostics
- broker-wide rates from NATS monitoring must remain distinct from
  service-local publish and subscribe counters exposed by individual Splash
  services
- the NATS monitoring listener should remain locally scoped or otherwise access
  controlled; it must not be treated as a public unauthenticated LAN API
