# Configuration

[Back to Splash API Service](Services-Splash-API-Home)

## Static service configuration

The initial `splash-api` slice should support:

| Variable | Purpose |
| --- | --- |
| `API_POOL_ID` | Single-pool identifier used by the initial API slice |
| `NATS_URL` | NATS connection target |
| `API_NATS_MONITORING_URL` | Optional local NATS monitoring base URL, such as `http://127.0.0.1:8222`, used to derive broker-level NATS throughput from `/varz` |
| `API_SERIAL_HEALTH_URL` | Optional `splash-serial` health endpoint, such as `http://10.0.40.54:9108/healthz`, used for platform service aggregation |
| `API_PROTOCOL_HEALTH_URL` | Optional `splash-protocol` health endpoint, such as `http://127.0.0.1:9109/healthz`, used for platform service aggregation |
| `API_HTTP_BIND` | Local HTTP bind address |
| `DATABASE_URL` | PostgreSQL connection string used for durable relational settings and later multi-pool metadata |
| `PGHOST` | Optional PostgreSQL host when `DATABASE_URL` is not used |
| `PGPORT` | Optional PostgreSQL port when `DATABASE_URL` is not used |
| `PGDATABASE` | Optional PostgreSQL database name when `DATABASE_URL` is not used |
| `PGUSER` | Optional PostgreSQL username when `DATABASE_URL` is not used |
| `PGPASSWORD` | Optional PostgreSQL password when `DATABASE_URL` is not used |
| `DATABASE_MIGRATIONS_DIR` | Optional SQL migrations directory override |
| `LOG_LEVEL` | Structured log verbosity |
| `TZ` | Service timezone |

## Initial equipment bridge configuration

The first slice may include a minimal local equipment bridge configuration for
the direct pump target.

Required bridge fields:

- `equipment_id`
- `equipment_type`
- `protocol_name`
- `bus_address`

ASSUMPTION: the first implementation may hard-code or use a simple local config
for one direct pump target while the full repository-backed equipment catalog is
still pending.

ASSUMPTION: the first implementation may use a static `API_POOL_ID` because the
full repository-backed pool model is not yet required for the initial browser
milestone.

## Durable settings ownership

- PostgreSQL is the durable store for relational and operator-managed settings
  such as pool-scoped weather-location configuration
- InfluxDB remains reserved for time-series telemetry such as temperatures,
  pump read-state history, and weather forecast snapshots
- `splash-api` should treat durable settings as repository-backed state rather
  than transient process memory

## Migration expectations

- `splash-api` should apply SQL migrations in a deterministic sorted order from
  a migrations directory during an explicit migration run
- the migration runner should record applied migration names in PostgreSQL so
  repeated runs are idempotent
- a local operator should be able to run migrations before starting the API,
  for example through an npm script or dedicated startup helper
- migration failures should stop the run clearly rather than allowing the API
  to proceed against an unknown schema
