# Splash Serial Service

[Back to README](Home)

## Purpose

This directory contains the service-specific design for `splash-serial`.

`splash-serial` is the transport-edge daemon that owns RS-485 port access on `splash-zero`. It is intentionally narrower than `splash-protocol` and should remain focused on serial lifecycle, raw transport publication, write execution, and transport observability.

## Scope

This service design complements, but does not replace:

- [System Architecture](Architecture-System-Architecture)
- [Deployment Architecture](Architecture-Deployment)
- [Resilience and Health](Architecture-Resilience)
- [Operations and Verification](Architecture-Operations)
- [NATS Messaging](Interfaces-NATS-Messaging)

## Service documents

- [Runtime Model](Services-Splash-Serial-Runtime)
  Process model, serial session lifecycle, reconnect behavior, and ownership boundaries.
- [Transport Contract](Services-Splash-Serial-Transport)
  Raw read semantics, outbound write handling, stream identity, and bus-idle enforcement.
- [Configuration](Services-Splash-Serial-Configuration)
  Environment variables, defaults, and validation expectations.
- [Observability](Services-Splash-Serial-Observability)
  Health, metrics, logging, and service-status expectations.
- [Testing](Services-Splash-Serial-Testing)
  Test strategy for unit, integration, and hardware-in-the-loop validation.
- [CI Runner Usage](Services-Splash-Serial-CI)
  Gitea Actions runner label, image, and workflow assumptions for no-hardware CI.

## Core responsibilities

- open and manage the serial adapter
- publish raw serial ingress as `serial.rx.raw`
- consume `serial.write.request` and perform actual writes
- publish write results as `serial.tx.raw`
- publish transport status through `serial.port.status`
- generate and persist a durable `serial_instance_id`
- enforce write serialization and bus-idle timing
- expose local `GET /healthz` and `GET /metrics`
- support a fully automated no-hardware test path for CI

## Dependencies

### Direct runtime dependencies

- configured serial adapter device such as `/dev/ttyUSB0`
- local runtime configuration file at `/etc/splash/splash-serial.env`
- NATS reachability to `splash-core`
- `systemd` service management on `splash-zero`
- local network connectivity between `splash-zero` and `splash-core`

### Consumed interfaces

- `serial.write.request` from `splash-protocol`

### Provided interfaces

- `serial.rx.raw`
- `serial.tx.raw`
- `serial.port.status`
- `GET /healthz`
- `GET /metrics`

The raw transport subjects must carry `serial_instance_id` because the serial
edge owns transport identity. `pool_id` remains a higher-layer domain concern
resolved on `splash-core`.

### Explicit non-dependencies

`splash-serial` should not require these services to start or continue basic transport operation:

- SQLite-backed API storage
- InfluxDB
- frontend availability
- REST API availability
- scheduler availability

## Explicit non-responsibilities

- frame reconstruction
- checksum validation
- payload decode
- normalized event generation
- command success determination beyond transport write result
- vendor-specific protocol logic

Those concerns belong to `splash-protocol`.
