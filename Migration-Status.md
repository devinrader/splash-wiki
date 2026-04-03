# Migration Status

The Gitea wiki is now the primary design authority for Splash.

## Current State

Migrated into the wiki:
- Product pages
- Architecture pages
- Interfaces pages
- Service pages
- Workflows page
- Pentair protocol material split into:
  - `Protocols/Pentair EasyTouch / IntelliTouch Reference`
  - `Protocols/Pentair Observed Installation`
  - `Research/Pentair EasyTouch Reverse Engineering`

Reduced in the local repository to migration pointers:
- `docs/architecture/architecture.md`
- `docs/architecture/equipment-protocols.md`
- `docs/interfaces/api-rest.md`
- `docs/interfaces/normalized-contracts.md`
- `docs/services/splash-api/runtime.md`
- `docs/services/splash-protocol/commands.md`
- `docs/workflows/workflows.md`

Still local and authoritative for workspace behavior:
- `docs/CONTRACT.md`
- `docs/README.md`
- `docs/WIKI-MIGRATION.md`
- `AGENTS.md`

## Remaining Cleanup

- decide whether the remaining repo-local design copies should also be reduced to pointers
- clean any copied wiki-page links that still assume repo-local navigation
- commit and push the wiki repository when the migrated structure is accepted
