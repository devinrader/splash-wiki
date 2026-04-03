# Splash NATS Service

[Back to README](Home)

## Purpose

This directory contains the service-specific design for `splash-nats`.

`splash-nats` is the platform message backbone for Splash. It provides low-latency event delivery across services and limited durability through JetStream where message loss would create missed actions.

## Scope

This service design complements, but does not replace:

- [System Architecture](Architecture-System-Architecture)
- [Deployment Architecture](Architecture-Deployment)
- [Resilience and Health](Architecture-Resilience)
- [Operations and Verification](Architecture-Operations)
- [NATS Messaging](Interfaces-NATS-Messaging)

## Service documents

- [Runtime Model](Services-Splash-NATS-Runtime)
  Placement, role, startup expectations, and dependency boundaries.
- [Streams and Subjects](Services-Splash-NATS-Streams-and-Subjects)
  Core NATS vs JetStream usage and subject/durability expectations.
- [Operations](Services-Splash-NATS-Operations)
  Health, observability, recovery, and operational notes for the message backbone.

## Core responsibilities

- provide the event backbone between Splash services
- carry Core NATS transport and telemetry events
- provide JetStream durability where missed messages would create missed actions
- support reconnect behavior for clients across service restarts and transient outages

## Explicit non-responsibilities

- business logic
- protocol decode or encode
- persistent relational storage
- authoritative historical event archive

Those responsibilities remain with application services and data stores.

## Dependencies

### Direct runtime dependencies

- Docker Compose on `splash-core`
- local storage sufficient for JetStream state
- LAN reachability for local clients such as `splash-zero`

### Consumed interfaces

- none at the application-message level; `splash-nats` is infrastructure rather than an application consumer

### Provided interfaces

- Core NATS publish/subscribe transport
- JetStream streams and consumers where configured

### Explicit non-dependencies

`splash-nats` should not require these services to start:

- PostgreSQL
- InfluxDB
- frontend
- `splash-api`
- `splash-scheduler`
- `splash-protocol`
- `splash-serial`
