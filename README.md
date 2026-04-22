# mpro-observe

Local-first CLI for collecting, filtering, normalizing, and saving remote Docker Compose service output for debugging and AI-assisted analysis.

## Status
Planning complete. Scaffold is next — **start with `BOOTSTRAP.md`**.

## Phase 1 focus
Phase 1 is **logs only**. Later phases may add other data sources.

## Planned commands
```bash
mpro-observe list-services --env staging
mpro-observe list-services --env production --remote

mpro-observe stream --env production --service api --since 5m --format text
mpro-observe stream --env staging --services api,migrator --migration-id MIG_123 --format jsonl --save
```

## Core rules
- Local-first: save evidence locally.
- Read-only in production.
- JSONL is the canonical normalized format.
- Save raw + normalized output.
- Does not mutate remote systems or store credentials.
- Every completed checkpoint updates `CHANGELOG.md`.

## Config
- Default path: `~/.config/mpro-observe/config.yml`
- Override: `MPRO_OBSERVE_CONFIG=/custom/path/config.yml`
- See `docs/mvp-spec.md` for the full Phase 1 contract.

## Current environment
- Staging compose: `/Users/boss/dev/migrationpro.io/mpro-setup/docker-compose.stage.yml`
- Production compose: `/Users/boss/dev/migrationpro.io/mpro-setup/docker-compose.yml`

## Document map
- `BOOTSTRAP.md` — first-time setup
- `docs/overview.md` — what this is and why it exists
- `docs/mvp-spec.md` — Phase 1 contract
- `docs/stack-reference.md` — tech decisions, build, gotchas
- Others: `docs/decisions.md`, `docs/roadmap.md`, `docs/future-improvements.md`, `docs/phases/phase-1.md`

## Roadmap
1. Phase 1 — capture
2. Phase 2 — structure
3. Phase 3 — AI workflows
4. Phase 4 — hardening
