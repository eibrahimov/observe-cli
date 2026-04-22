# Overview

## Problem
Today, service inspection is manual and scattered:
- many services live behind Docker Compose
- staging and production require different commands and files
- logs are hard to save, compare, and reuse
- migration-level debugging is slow when context is split across services
- AI analysis is awkward when the input is noisy and inconsistent

## Goal
Design a personal CLI that can collect remote Docker Compose service output from staging and production, normalize it, filter it, and store it locally for debugging and AI-assisted analysis.

## Why a local CLI
A small local CLI creates a repeatable workflow:
- one entry point for staging and production
- one way to stream, filter, clean, and save service data
- one normalized format for AI and local analysis
- one local evidence trail for later review

## Primary use cases
- inspect one service such as `api` or `migrator`
- inspect multiple services together
- trace one migration across services
- save investigation data locally for later review
- convert noisy output into clean, reusable input for AI

## Product boundary
### This tool should do
- stream output from one service or many services
- save data locally
- output `raw`, `text`, and `jsonl`
- filter by environment, service, time, level, regex, and migration ID
- clean noise before showing or saving data

### This tool should not do
- deploy or change remote services
- mutate production systems
- replace a full observability platform
- automate remediation
- store remote credentials (relies on the user's SSH config)

## Design principles
- local-first
- read-only in production
- save raw + normalized output
- keep the CLI small and scriptable
- make migration tracing a first-class use case

## Phase boundary
- Phase 1 handles **logs only**.
- Later phases may add container exec, HTTP probes, database queries, and queue inspection.

## Main risks
- inconsistent log formats across services
- missing migration IDs in some logs
- multiline stack traces
- differences between local and remote Compose setup

## Document map
- `docs/mvp-spec.md` — exact Phase 1 scope and behavior
- `docs/phases/phase-1.md` — capture plan
- `docs/roadmap.md` — Phases 2–4 at a glance
- `docs/stack-reference.md` — tech decisions, build commands, gotchas
- `docs/decisions.md` — decision log
- `docs/future-improvements.md` — deferred ideas with promotion triggers
