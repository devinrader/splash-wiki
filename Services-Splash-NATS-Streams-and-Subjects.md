# Streams and Subjects

[Back to Splash NATS Service](Services-Splash-API-Home)

## Purpose

This document defines how Splash uses Core NATS and JetStream at the service level.

The authoritative subject catalog lives in [NATS Messaging](Interfaces-NATS-Messaging). This document defines service-level durability expectations.

## Core NATS usage

Core NATS should be used where low latency matters more than durability.

Examples:

- `serial.rx.raw`
- `serial.tx.raw`
- `serial.port.status`
- `protocol.frame.raw`
- `protocol.frame.decoded`
- `equipment.state.*`
- `protocol.command.intent`
- `serial.write.request`
- `command.result.{command_id}`

Reason:

- these subjects represent live transport, protocol, and control state where occasional loss during reconnect windows is acceptable in v1

## JetStream usage

JetStream should be used where missed delivery would create missed work or user-visible loss of intent.

Examples:

- `automation.suggestion.*`
- `notification.send`
- `task.created`
- `task.updated`

Reason:

- these subjects represent actionable or user-relevant work that should survive transient service or connection failures

## Design rules

- do not use NATS as the primary historical archive for protocol traffic
- durability choice must follow business impact, not just implementation convenience
- subject ownership remains defined by the interface contracts
- service-specific docs should reference, not duplicate, the canonical payload schemas
