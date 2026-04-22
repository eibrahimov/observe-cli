# Future Improvements

This document tracks deferred ideas and explicitly-descoped items. Things land here when they are genuinely useful but not worth the cost in the current phase plan, or when ecosystem readiness is the blocker.

Each entry explains **what**, **why deferred**, and **what would trigger promotion** into a phase.

---

## DuckDB analytical store
**What:** Replace or augment the Phase 2 SQLite index with DuckDB for columnar analytics, Parquet export, and larger local datasets.

**Why deferred:**
- As of April 2026, `@duckdb/node-api` on npm does not include the Bun compatibility fix (duckdb-node-neo PR #388 merged April 5 but not yet released). Using it today means installing from git, which is not a responsible dependency for a personal tool.
- DuckDB cannot be bundled with `bun build --compile` — its native `.node` addon defeats the static bundler. Shipping would require either process-level isolation (sidecar) or giving up single-binary distribution.
- Phase 2's workload (filter by migration_id/service/level, time range, JSON extraction) is well served by `bun:sqlite`, which is built into Bun and works in compiled binaries.

**Promotion trigger:**
- DuckDB's npm release includes PR #388
- workloads emerge that SQLite genuinely cannot handle (multi-gigabyte saved runs, Parquet export to hand to external analysis tools, complex OLAP queries)
- either DuckDB's compile story improves, or we accept a sidecar process architecture

---

## Absolute timestamps for `--since`
**What:** Accept ISO 8601 timestamps in `--since` in addition to relative durations.

**Why deferred:** Relative durations cover every real debugging session so far. Absolute timestamps add complexity (timezone handling, validation, clock skew with remotes) for a rare use case.

**Promotion trigger:** An actual incident where relative `--since` is insufficient.

---

## Log-timestamp-ordered stream merge
**What:** Merge events from multiple services by parsed log timestamp rather than local arrival time.

**Why deferred:** Arrival-time merge is simpler, has zero buffering latency, and is good enough for real-time debugging. Log-timestamp merge requires clock-synced remotes, a merge window (how long to wait for out-of-order events), and introduces latency.

**Promotion trigger:** Cross-service correlation breaks because of arrival-order interleaving in real use.

---

## Container exec
**What:** Run arbitrary commands inside a running container (e.g., `observe-cli exec --env staging --service api -- ps aux`).

**Why deferred:** Mutation-adjacent. Conflicts with the "read-only in production" principle unless carefully scoped. Would need a safety model (allowlist of commands? explicit `--allow-exec` flag? separate subcommand with audit logging?).

**Promotion trigger:** A repeated need during debugging that can't be solved by better capture + analysis.

---

## HTTP probe adapter
**What:** Health-check style probes against service HTTP endpoints, with saved response capture.

**Why deferred:** Phase 1 is logs-only by design. HTTP probing is a different capture model (pull vs stream).

**Promotion trigger:** Real debugging sessions repeatedly need "what did this endpoint return at time X."

---

## Database query adapter
**What:** Read-only queries against service databases with saved results.

**Why deferred:** Significant complexity (connection strings, credentials, query safety, result schemas). Read-only enforcement at the DB level is non-trivial. Conflicts with "tool should not store remote credentials."

**Promotion trigger:** Enough debugging sessions that require DB state to justify a separate credential handling model.

---

## Queue inspection adapter
**What:** Inspect Celery / Redis / message queue state.

**Why deferred:** Same reasoning as DB adapter. Different capture model per queue technology.

**Promotion trigger:** Migration debugging repeatedly blocked by queue state visibility.

---

## Alerting and automation
**What:** Trigger actions based on patterns in live streams.

**Why deferred:** Not this tool's job. `observe-cli` is for human investigation, not automation. Crosses into observability platform territory.

**Promotion trigger:** Never, probably. If this is needed, use a different tool.

---

## Hosted dashboard
**What:** Web UI for browsing saved runs.

**Why deferred:** Solo developer tool. Files on disk + grep + the `query` command cover the need. A dashboard adds ops burden.

**Promotion trigger:** Team usage, not solo usage.

---

## Monorepo migration
**What:** Split `observe-cli` into a multi-package workspace, for example CLI + shared core + optional future UI or analysis package.

**Why deferred:** A single-package repo is simpler while the product shape is still settling. A monorepo adds workspace management, release coordination, and compile-path questions before there is a real second package.

**Promotion trigger:** A second real package exists and the single-package layout starts causing duplication, release friction, or unclear ownership.

---

## Remote credential storage
**What:** Store SSH credentials inside the tool.

**Why deferred:** Explicit non-goal per `docs/overview.md`. SSH config is the right place for this.

**Promotion trigger:** None — this stays out.
