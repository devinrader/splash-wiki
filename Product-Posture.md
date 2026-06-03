# Product Posture

[Back to README](Home)

## Canonical stance

Splash is a self-hosted, local-first pool management platform.

The canonical product and deployment posture for v1 is:

- self-hosted and local-first rather than SaaS-first
- free to use in v1 with no monetization feature set
- LAN-only by default in v1
- Cloudflare Tunnel and Cloudflare Access are deferred to v2 for remote access
- suggest-and-approve automation is the trust model in v1, not full autonomy
- no dosing calculator in v1
- responsive web UI, MagicMirror integration, and Protocol Explorer tooling are
  the primary delivery surfaces

## Deployment stance

- `splash-core` is the main host for application services and persistent
  storage
- `splash-zero` remains the transport-edge host for RS-485 serial I/O
- remote access is not a baseline deployment expectation in v1

## Product trust model

- commands that can affect live equipment should remain explicit user-approved
  actions in v1
- user trust should be built through observability, explainability, and
  suggested actions before any future autonomous control posture is considered
