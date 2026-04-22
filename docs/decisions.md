# Decisions

An append-only log of significant technical and scope decisions. ADR-lite: short, dated, with enough context to understand why past-us made a choice.

## Conventions
- Entries are numbered `ADR-NNN`, never renumbered
- Status is one of: `accepted`, `superseded`, `deprecated`
- Superseded entries stay in the file but link to the replacement
- New entries go at the bottom

## Open questions
- none

---

## ADR-001 — Runtime: Bun + TypeScript
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need a fast, typed, scriptable CLI with a small dependency surface and straightforward packaging.

**Decision:** Bun + TypeScript.

**Rationale:** Bun gives native TypeScript execution, built-in YAML/SQLite/process APIs, and `bun build --compile` for single-binary distribution. Startup is fast enough for a frequently-invoked CLI.

**Consequences:** Build and packaging assume Bun. Windows stays secondary. Core storage and process choices follow Bun built-ins.

---

## ADR-002 — CLI framework: citty
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need typed command definitions without a large framework or extra dependency tree.

**Decision:** Use `citty`.

**Rationale:** `defineCommand()` keeps argument handling direct, typed, and lazy-load friendly. It also fits setup and cleanup hooks needed for long-running stream commands.

**Consequences:** Command structure follows citty's model, not commander-style method chaining.

---

## ADR-003 — Terminal colors: ansis
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need color output with low dependency weight and good Bun compatibility.

**Decision:** Use `ansis`.

**Rationale:** It stays small, handles multi-style output well, supports Truecolor, and avoids the extra scrutiny attached to other color packages.

**Consequences:** Terminal styling conventions and helpers assume `ansis`.

---

## ADR-004 — Table formatter: custom with Bun built-ins
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need readable terminal tables without pulling in another formatting dependency.

**Decision:** Use a custom table formatter built on `Bun.stringWidth()` and `Bun.sliceAnsi()`.

**Rationale:** Bun already exposes the hard parts: display width and ANSI-safe slicing. A small custom formatter keeps the dependency surface down.

**Consequences:** Table behavior is owned locally instead of outsourced to a package.

---

## ADR-005 — Local storage and query layer: `bun:sqlite`, DuckDB deferred
**Date:** 2026-04-16
**Status:** accepted

**Context:** Phase 2 needs local querying over saved runs without complicating distribution.

**Decision:** Use `bun:sqlite` for the first local index. Keep DuckDB deferred.

**Rationale:** SQLite is built into Bun, works in compiled binaries, and covers the Phase 2 workload. DuckDB stays attractive for larger analytical workloads, but it is not the right first step.

**Consequences:** Query helpers and saved-run indexing target SQLite first. DuckDB remains tracked in `docs/future-improvements.md`.

---

## ADR-006 — Validation library: Zod v3
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need schema validation for config and structured data.

**Decision:** Use Zod v3, not v4.

**Rationale:** Zod v4 currently has a Bun install bug in `.d.cts` files. Zod v3 works and is enough for the current scope.

**Consequences:** Do not upgrade validation code to Zod v4 until the Bun compatibility issue is resolved.

---

## ADR-007 — Lint and format: Biome v2
**Date:** 2026-04-16
**Status:** accepted

**Context:** Need one tool for linting and formatting without stacking ESLint and Prettier.

**Decision:** Use Biome v2.

**Rationale:** Biome replaces both tools with one binary and keeps setup small.

**Consequences:** Config is generated via `biome init`, not hand-written from scratch.

---

## ADR-008 — Repo structure: single-package
**Date:** 2026-04-16
**Status:** accepted

**Context:** The project is a solo CLI with no second package yet.

**Decision:** Keep a single-package repo. Defer monorepo structure.

**Rationale:** A workspace adds release coordination and build questions before there is a real second package.

**Consequences:** Shared-core extraction waits until there is actual duplication or a second deliverable.

---

## ADR-009 — Phase 1 scope: logs only
**Date:** 2026-04-16
**Status:** accepted

**Context:** The product vision is broader than logs, but the first implementation needs a tight boundary.

**Decision:** Phase 1 handles Docker Compose logs only.

**Rationale:** Logs cover the primary debugging path without pulling in container exec, HTTP probes, database access, or queue inspection.

**Consequences:** Broader capture modes stay out of the MVP and move to later phases or `docs/future-improvements.md`.

---

## ADR-010 — Stream merge ordering: arrival time
**Date:** 2026-04-16
**Status:** accepted

**Context:** Multi-service streams need a merge rule.

**Decision:** Merge by local arrival time, not parsed log timestamp.

**Rationale:** Arrival-time ordering avoids buffering latency and clock-skew assumptions. It is simpler and good enough for the first streaming implementation.

**Consequences:** Cross-service ordering is approximate in Phase 1. Revisit if real debugging shows it is not good enough.

---

## ADR-011 — Interrupt behavior: Ctrl-C means `interrupted`
**Date:** 2026-04-16
**Status:** accepted

**Context:** Long-running streams rarely have a natural end. The run lifecycle needs a clear signal contract.

**Decision:** `SIGINT` / `SIGTERM` mark the run as `interrupted`.

**Rationale:** A user-stopped stream is not a normal completion. Marking it `interrupted` preserves what happened.

**Consequences:** Signal handlers must flush buffers, update `meta.json`, and kill child SSH processes.

---

## ADR-012 — Exit code conventions
**Date:** 2026-04-16
**Status:** accepted

**Context:** The CLI needs stable shell-facing status codes.

**Decision:** Use `0` for success, `1` for runtime error, `2` for usage/config error, and `130` for `SIGINT`.

**Rationale:** These meanings are direct, script-friendly, and consistent with common shell expectations.

**Consequences:** Command docs and tests must treat exit codes as part of the public contract.

---

## ADR-013 — Migration ID matching: case-insensitive by default
**Date:** 2026-04-16
**Status:** accepted

**Context:** Migration IDs appear in inconsistent formats across services.

**Decision:** Compile configured `migration_id_patterns` case-insensitively by default.

**Rationale:** This keeps the MVP filter practical without adding per-pattern flags or overcomplicating config.

**Consequences:** Phase 1 matching stays simple. Structured extraction still belongs to Phase 2.

---

## ADR-014 — AI workflows operate on saved runs only
**Date:** 2026-04-16
**Status:** accepted

**Context:** Analysis features should stay reproducible and should not open fresh remote connections behind the user's back.

**Decision:** Phase 3 operates on saved local runs and the local index, not live SSH streams.

**Rationale:** Saved data is reproducible, cheaper to inspect, and keeps remote systems read-only in the strictest sense.

**Consequences:** Fresh data must be captured first with Phase 1, then analyzed locally.

---

## ADR-015 — Credentials stay in SSH config, not in the tool
**Date:** 2026-04-16
**Status:** accepted

**Context:** The tool needs remote access, but credential handling would expand scope and risk.

**Decision:** Rely on the user's SSH config, keys, or agent forwarding. Do not store remote credentials.

**Rationale:** SSH already solves this well, and the product boundary explicitly excludes credential storage.

**Consequences:** Passwordless SSH is a prerequisite. Remote credential storage stays a non-goal.
