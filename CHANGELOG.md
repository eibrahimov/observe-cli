# Changelog

All notable changes to `observe-cli` are recorded here.

This project follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This file starts from the current planning baseline, not from the draft-by-draft doc iteration that produced it.

## [Unreleased]
### Added
- `README.md`: project entrypoint and doc map.
- `BOOTSTRAP.md`: first-time setup runbook.
- `docs/overview.md`: product overview, boundary, principles, and risks.
- `docs/mvp-spec.md`: exact Phase 1 contract.
- `docs/stack-reference.md`: stack decisions, build commands, and coding gotchas.
- `docs/decisions.md`: ADR-lite decision log.
- `docs/roadmap.md`: short forward-look for Phases 2-4.
- `docs/future-improvements.md`: deferred ideas and promotion triggers.
- `docs/phases/phase-1.md`: detailed Phase 1 implementation plan.

### Changed
- Planning docs were restructured to match the scale of a solo pre-implementation CLI project.
- `docs/mvp-spec.md` was trimmed to contract-level scope; numeric implementation defaults stay in code.
- `README.md` was trimmed to a shorter start-here document map.
- File-path references were normalized across docs.

## Entry template
Use this shape for each completed checkpoint. Omit any section that has no entries.

```md
## [0.x.y] — YYYY-MM-DD — <phase/checkpoint>
### Added
- ...

### Changed
- ...

### Deprecated
- ...

### Removed
- ...

### Fixed
- ...

### Security
- ...

### Breaking
- ...

### Notes
- Optional context, risks, or follow-up items.
```

## Rules
- Update this file after every completed checkpoint.
- Bump the patch version (`0.0.x`) per checkpoint, minor (`0.x.0`) per phase completion.
- Record user-visible CLI changes.
- Record config schema changes.
- Record JSONL schema changes.
- Record storage layout changes.
- Record production safety changes.
- Record dependency changes.
- Record breaking changes prominently in a dedicated `### Breaking` section.
- Omit empty sections.
