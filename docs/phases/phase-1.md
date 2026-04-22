# Phase 1 — Capture

## Goal
Build the first usable CLI for streaming and saving Docker Compose logs from staging and production.

## Dependencies
- `docs/mvp-spec.md` is approved
- `BOOTSTRAP.md` has been followed (scaffold exists, `bun run dev --help` works)
- environment access exists for staging and production
- remote Docker Compose log access works over SSH without password prompts (SSH keys or agent forwarding)
- local storage path is writable

## In scope
- config loader and env validation
- environment definitions for staging and production
- service discovery from local Compose files (and remote via `--remote`)
- SSH + Docker Compose log streaming
- one service, many services, or all services
- stdout formats: `raw`, `text`, `jsonl`
- local save: raw logs + normalized JSONL
- basic filters: `--since`, `--grep`, `--level`, `--migration-id`
- run lifecycle with `meta.json` status
- signal handling and child process cleanup
- save guardrails (warn/max MB, confirmation for broad production streams)
- production read-only safeguards

## Out of scope
- repo scaffold (handled by `BOOTSTRAP.md`)
- exec, HTTP, DB, or queue adapters
- DuckDB or other analytical stores
- AI summaries
- advanced per-service parsers
- multiline grouping
- automated correlation
- automated retention and cleanup

## Deliverables
- initial CLI command surface
- config file support
- service discovery command (local + `--remote`)
- stream command with save support
- normalized JSONL output
- basic filters and production safeguards
- signal handling, clean child process shutdown
- save size guardrails

## Testing expectation
Every checkpoint ships with its own tests. `bun test` is built-in and fast — there is no excuse to defer tests to Phase 4. Minimum per checkpoint:
- unit tests for pure logic (parsers, filters, schema)
- one smoke test per command against a fixture

## Checkpoint rule
Every completed checkpoint must update `CHANGELOG.md`.

## Checkpoints
1. **Config loader**
   - assumes `BOOTSTRAP.md` is done (scaffold exists)
   - add YAML config loader that reads `~/.config/mpro-observe/config.yml` or `MPRO_OBSERVE_CONFIG`
   - add basic config validation: env exists, paths exist, `ssh_target` is non-empty
   - tests: config load + validation
2. **Service discovery**
   - implement `list-services --env <env>` from local Compose files
   - implement `--remote` variant that SSH's and runs `docker compose config --services`
   - tests: local + remote parsing, error cases
3. **Single-service streaming**
   - stream one remote service over SSH Docker Compose logs
   - support `raw` and `text`
   - install SIGINT/SIGTERM handlers, ensure child SSH process is killed on exit
   - tests: signal handling, ANSI strip
4. **Multi-service streaming**
   - support `--services` and `--all-services`
   - merge streams locally by arrival time with env/service metadata
   - enforce `limits.max_concurrent_services` cap
   - tests: merge ordering, concurrency cap
5. **Normalization (stdout)**
   - add `jsonl` stdout format
   - parse timestamps and levels when present
   - tests: normalization schema, level detection
6. **Persistence**
   - save raw logs and normalized JSONL locally
   - write `meta.json` with `running` status at start, flip to `completed`/`interrupted`/`failed` on exit
   - implement save guardrails (warn at `save_warn_mb`, abort at `save_max_mb`)
   - tests: meta.json lifecycle, guardrail triggers
7. **Filters + safety**
   - add `--since`, `--grep`, `--level`, and MVP `--migration-id`
   - add production-in-scope confirmation for `--all-services` without filters
   - tests: each filter, production confirmation
8. **Manual verification**
   - test staging and production paths
   - verify save layout and error handling

## Definition of done
- MVP acceptance criteria are met
- staging and production both work through config
- at least one end-to-end saved run exists for validation
- README examples match the implemented command surface
- `meta.json` correctly reflects all four lifecycle states in test runs
- all completed checkpoints are recorded in `CHANGELOG.md`
- test suite passes and covers each checkpoint's behavior

## Manual verification checklist
- list services for staging (local and `--remote`)
- list services for production (local and `--remote`)
- stream one service in `text`
- stream multiple services in `jsonl`
- save one run and verify `meta.json`, `raw/`, and `normalized/`
- interrupt a saved run with Ctrl-C, verify `meta.json` shows `interrupted`
- trigger save warn threshold with a verbose run
- verify `--grep`, `--level`, and `--migration-id`
- verify production `--all-services` without filters requires confirmation
- verify production commands remain read-only
- verify conflicting target selector flags exit with code `2`
