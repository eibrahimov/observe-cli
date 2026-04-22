# MVP Spec
## Objective and boundaries
Ship a usable Phase 1 CLI that can stream remote Docker Compose logs from staging or production, filter them, normalize them, and save them locally.
- This spec defines the exact Phase 1 behavior. It does not choose the implementation tech stack.
- Phase 1 is **logs only**. Broader data sources belong to later phases.
## Scope
### In scope
- environments: `staging`, `production`
- source: Docker Compose logs over SSH
- targets: one service, many services, or all services
- stdout formats: `raw`, `text`, `jsonl`
- local save: raw logs + normalized JSONL
- filters: `--since`, `--grep`, `--level`, `--migration-id`
- service discovery from local Compose files
- production-safe read-only defaults
### Out of scope
- container exec
- HTTP or DB adapters
- DuckDB or other analytical stores
- AI summaries
- Markdown report generation
- advanced per-service parsing
- alerting or automation
## Config
- Default config path: `~/.config/mpro-observe/config.yml`
- Optional override: `MPRO_OBSERVE_CONFIG=/custom/path/config.yml`
- `ssh_target` must resolve through the user's SSH config without prompting for a password. Use SSH keys or agent forwarding. The tool does not store or prompt for credentials.
### Config schema
```yaml
storage_root: ~/Documents/mpro-data

environments:
  staging:
    ssh_target: mpro@mp-staging
    local_compose_file: /Users/boss/dev/migrationpro.io/mpro-setup/docker-compose.stage.yml
    remote_compose_file: /home/mpro/mpro-setup/docker-compose.stage.yml
    remote_env_file: /home/mpro/mpro-setup/.env.stage

  production:
    ssh_target: <TODO: set-production-ssh-target>
    local_compose_file: /Users/boss/dev/migrationpro.io/mpro-setup/docker-compose.yml
    remote_compose_file: /home/mpro/mpro-setup/docker-compose.yml
    remote_env_file: /home/mpro/mpro-setup/.env

filters:
  migration_id_patterns:
    - 'migration[_ -]?id[=: ](?<migration_id>[A-Za-z0-9_-]+)'
    - 'migrationId[=: ](?<migration_id>[A-Za-z0-9_-]+)'
```
## CLI commands
```bash
mpro-observe list-services --env staging
mpro-observe list-services --env production
mpro-observe list-services --env staging --remote

mpro-observe stream --env staging --service api --format text
mpro-observe stream --env production --services api,migrator --format jsonl --save
mpro-observe stream --env staging --all-services --since 10m --format raw
mpro-observe stream --env production --service migrator --migration-id MIG_123 --format jsonl --save
```
## Command contract
- `list-services`: requires `--env`, reads the local Compose file by default, can use `--remote` to run `docker compose config --services` over SSH, prints service names, and fails if the environment or Compose file is invalid.
- `stream`: requires `--env`, requires exactly one target selector (`--service`, `--services`, `--all-services`), opens one log stream per selected service, merges locally, attaches `env` and `service` metadata, applies cleanup and filters, prints in the requested format, and saves locally only when `--save` is set.
## Flag behavior
| Flag | Required | Meaning | Default |
|---|---|---|---|
| `--env` | yes | target environment | none |
| `--service` | one target mode | single service | none |
| `--services` | one target mode | comma-separated service list | none |
| `--all-services` | one target mode | all services from Compose | false |
| `--format` | no | `raw`, `text`, or `jsonl` | `text` |
| `--since` | no | initial time window | `5m` |
| `--grep` | no | substring or regex filter | none |
| `--level` | no | comma-separated levels | none |
| `--migration-id` | no | migration filter | none |
| `--save` | no | persist run locally | false |
| `--remote` | no | `list-services` only: query remote Compose | false |
| `--yes` | no | bypass confirmation for broad production streams | false |
Target selector validation: passing zero or more than one of `--service`, `--services`, `--all-services` must fail fast with exit code `2`:
```text
error: choose exactly one of --service, --services, or --all-services
```
## Stream behavior
- Formats: **`raw`** is unmodified `docker compose logs` output; **`text`** is ANSI-stripped one-line-per-event output prefixed with `[env/service]`; **`jsonl`** is one normalized JSON object per line and is canonical for `--save` and AI handoff.
- `--since`: relative durations only in Phase 1 (`30s`, `5m`, `2h`, `1d`). Absolute ISO timestamps are deferred.
- `--level`: matches normalized values such as `debug`, `info`, `warn`, `error`, `fatal`; events with no detected level may be excluded.
- `--grep`: applies to the raw line content before rendering; accepts substring match or JS-flavored regex.
- `--migration-id`: matches by exact substring or configured `migration_id_patterns`; patterns are compiled case-insensitively by default in Phase 1; structured cross-service extraction belongs to Phase 2.
- Merge ordering: multiple services are merged by local arrival time, not by log timestamp.
- Multiline events: each `docker compose logs` line is an independent Phase 1 event; stack traces and continuation lines stay split.
## Save behavior
- `--save` creates a timestamped run directory, saves raw log output per service, saves normalized JSONL per service, and writes `meta.json` with run metadata.
- Run lifecycle: `completed` means no signal and all buffered data flushed; `interrupted` means `SIGINT`/`SIGTERM`, forced stop, or unwritable save path; `failed` means SSH or Compose returned non-zero before any data arrived.
- Signal handlers must flush buffers, set `meta.json` to `interrupted`, and kill child SSH processes before exit.
- Minimum `meta.json` fields: `cli_version`, `config_path`, `start_time`, `end_time`, `environment`, `services`, `filters`, `format`, `status`, `bytes_written`.
## Normalized JSONL schema
```json
{
  "ts": "2026-04-16T12:00:01Z",
  "env": "staging",
  "service": "api",
  "source": "compose.logs",
  "level": "error",
  "migration_id": "MIG_123",
  "message": "database timeout",
  "raw": "original log line"
}
```
## Storage layout
```text
<storage_root>/
  runs/
    <timestamp>/
      meta.json
      raw/
        staging-api.log
        production-migrator.log
      normalized/
        staging-api.jsonl
        production-migrator.jsonl
```
## Cleanup and guardrails
- Cleanup rules: strip ANSI color, add env and service metadata, keep timestamps when present, detect JSON lines when possible, keep the original line in `raw`.
- Saved runs must warn at a configurable threshold and abort new writes at a configurable hard max.
- `--all-services` must enforce a configurable concurrency cap and fail fast when exceeded.
- `--all-services` in production without filters must require confirmation unless `--yes` is passed.
- Warn/max MB values, concurrency caps, and similar tunables live in code with sensible initial values. Adjust via config when needed, but keep the contract above stable.
## Error behavior
- invalid config, environment, or service selection must fail fast
- remote SSH or Docker Compose failures must return a non-zero exit
- if all selected services fail, the command fails
- save-path failures must return a non-zero exit and update `meta.json` to `interrupted`
- error messages should clearly name the environment and service involved
- exit codes: `0` success, `1` runtime error, `2` usage or config error, `130` interrupted by `SIGINT`
## Production safety
- explicit `--env production`
- read-only commands only
- default short window: `--since 5m`
- local save only
- `--all-services` in production without filters requires confirmation
## Acceptance criteria
- can list services for staging and production (local and remote)
- can stream one service or many services
- can save raw logs and normalized JSONL locally
- can filter by time, regex, level, and basic migration ID
- can output `raw`, `text`, or `jsonl`
- works without mutating remote systems
- `meta.json` status reflects run lifecycle correctly, including `SIGINT`
- save guardrails prevent runaway disk usage
## Change rule
After every completed checkpoint during MVP work, update `CHANGELOG.md`.
