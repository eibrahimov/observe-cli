# Roadmap

Phase 1 is the detailed plan (`docs/phases/phase-1.md`). Everything below is directional — Phase 1 will teach us what Phase 2 actually needs. These sections will expand into full phase docs as each phase becomes next.

## Phase 2 — Structure
Turn raw captured logs into queryable, migration-aware data. Add per-service parsing rules, stronger migration/request/trace ID extraction, multiline stack trace grouping, and better level detection. Index saved runs locally in `bun:sqlite` and add basic query and export helpers. Depends on: Phase 1 complete, normalized saved runs available as fixtures.

## Phase 3 — AI workflows
Use saved and structured data for faster investigation and summary generation. Add saved-run summaries, migration analysis across services, staging-vs-production comparison, and reusable prompt or report templates. Phase 3 operates on saved local data only — no live SSH and no fresh remote reads. Depends on: Phase 2 complete, stable schema, reliable local queries.

## Phase 4 — Hardening
Make the CLI safer and more durable for regular use. Expand integration and regression coverage, add schema-level config validation, define retention and cleanup policy, add redaction for saved output, and finish packaging and install guidance. Depends on: the core capture and analysis flows being stable enough to harden against real failure modes.
