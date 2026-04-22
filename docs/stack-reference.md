# Stack Reference

Decisions, build commands, and gotchas for `observe-cli`. Looked up while coding, not read top to bottom.

**Setting up for the first time?** See `BOOTSTRAP.md` at repo root.
**Reading about deferred ideas?** See `docs/future-improvements.md`.
**Need the short decision log?** See `docs/decisions.md`.

## Stack

| Category      | Choice        | Version  | Why                                                                       |
| ------------- | ------------- | -------- | ------------------------------------------------------------------------- |
| Runtime       | Bun           | 1.3.12+  | 6ms startup, built-in SQLite/YAML/glob, `bun build --compile`             |
| Language      | TypeScript    | 5.8+     | native in Bun, zero config                                                |
| CLI framework | citty         | 0.2.2    | zero deps, typed `defineCommand()`, lazy subcommands, setup/cleanup hooks |
| Validation    | zod           | 3.23     | works on Bun; v4 has a Bun install bug                                    |
| Config format | `Bun.YAML`    | built-in | no package needed                                                         |
| Colors        | ansis         | 3.17     | 5.7KB, fastest at multi-style, Truecolor, zero deps, explicit Bun support |
| Tables        | custom        | —        | `Bun.stringWidth()` + `Bun.sliceAnsi()`, SIMD, CJK/emoji-safe, zero deps  |
| Storage       | `bun:sqlite`  | built-in | 3–6x faster than better-sqlite3, works in compiled binaries               |
| Process exec  | `Bun.spawn()` | built-in | fast; manual child cleanup required                                       |
| Testing       | `bun test`    | built-in | jest-compatible, recursive discovery                                      |
| Lint/format   | Biome         | 2.4      | single Rust binary, replaces ESLint + Prettier                            |

**Total npm deps: 3.** Everything else is Bun built-in.

## Decision log

**Why citty over commander.** Commander has better TypeScript inference only in theory; its method-chaining API defeats end-to-end type inference. citty's `defineCommand({ args: {...} })` pattern gives typed args without manual assertions. Also: zero deps, lazy subcommand loading, explicit setup/cleanup hooks that pair well with signal handling. See `docs/decisions.md` ADR-002.

**Why ansis over chalk or picocolors.** chalk was compromised in the September 2025 npm supply chain attack; v5.6.2+ is clean but the incident raised scrutiny. picocolors wins at single-style but loses at multi-style (real CLI usage) and has no Truecolor support. ansis wins on bundle size, multi-style performance, Truecolor, null/undefined safety, and explicit Bun compatibility. See `docs/decisions.md` ADR-003.

**Why custom tables over cli-table3 or console-table-printer.** `Bun.stringWidth` is ~6,756× faster than the `string-width` package and correctly handles CJK, emoji (including ZWJ sequences), and ANSI. `Bun.sliceAnsi` handles the hardest part of table layout (slicing ANSI-styled text by visual width). A custom formatter is ~150 lines with zero dependencies and no supply chain surface. See `docs/decisions.md` ADR-004.

**Why `bun:sqlite` over DuckDB for Phase 2 indexing.** See `docs/future-improvements.md` for the full DuckDB status. Short version: DuckDB's Bun compatibility fix is merged upstream but not yet released to npm, and DuckDB cannot be bundled with `bun build --compile`. `bun:sqlite` handles the Phase 2 workload (filter, time range, JSON extraction) and ships in every compiled binary. See `docs/decisions.md` ADR-005.

## Build and distribute

**Single binary:**
```bash
bun run build
# produces dist/observe-cli (~55–60MB)
```

**Cross-compile:**
```bash
bun build --compile --target bun-linux-x64 --outfile dist/observe-cli-linux src/index.ts
bun build --compile --target bun-darwin-arm64 --outfile dist/observe-cli-macos src/index.ts
bun build --compile --target bun-linux-arm64 --outfile dist/observe-cli-linux-arm64 src/index.ts
```

Supported targets: `bun-linux-x64`, `bun-linux-arm64`, `bun-darwin-x64`, `bun-darwin-arm64`, `bun-windows-x64`. The `-musl` variants exist for Alpine-based distributions.

**Recommended flags:**
- `--minify-whitespace --minify-syntax` — safe, no stack trace impact
- `--sourcemap` — keep for debugging
- `--bytecode` — faster startup

**Flags to avoid:**
- `--minify-identifiers` — breaks stack traces, breaks OpenTelemetry if added later
- `--bytecode --format esm` together — produces broken binaries (Bun issue #27955)

**Binary size.** The base Bun runtime (bundler, test runner, package manager, SQLite engine) is embedded in every binary regardless of what the code uses. Expect 55–60 MB. UPX compression can shrink to ~20 MB at cost of ~250 ms startup.

## Coding gotchas

**`Bun.spawn` does not kill children on parent exit.** Register SIGINT and SIGTERM handlers, track spawned processes in a Set, kill them on signal. Orphaned child SSH processes are a real problem when streaming multiple services. The `uid`/`gid` spawn options are silently ignored — don't rely on them.

**`--inspect` debugger is unreliable.** Crashes frequently on breakpoints. Plan for `console.log` debugging on tricky issues until this stabilizes.

**Windows support is experimental.** No PTY (`Bun.Terminal`), worker thread crashes under concurrency, `process.exit()` in a child can kill the parent. Treat Linux and macOS as the primary targets; test Windows builds explicitly if needed.

**`bun build --compile` and native addons.** Simple `.node` files with static paths embed correctly. Anything using `node-pre-gyp`, `prebuild-install`, or runtime `__dirname`-based discovery fails. This includes Sharp, better-sqlite3, bcrypt, canvas, and — currently — DuckDB. `bun:sqlite` embeds cleanly.

**Zod v4 workaround if you absolutely must upgrade.** The `.d.cts` files have unterminated quotes that Bun's installer chokes on. If a future need forces v4, install via npm or pnpm instead of `bun install`, or wait for the upstream fix. Default stays on v3 — see `BOOTSTRAP.md` and `docs/decisions.md` ADR-006.

**Biome v2 — `organizeImports` moved.** If hand-editing `biome.json`, the `organizeImports` key is now under `assist.actions.source`, not at the top level. Most other keys carried over from v1 with minor schema changes.

## If this becomes a monorepo

Not in scope today. If the project grows to need multiple packages (CLI, dashboard, shared core), a few things to know before restructuring:
- Bun 1.3.2+ uses isolated installs by default for new workspaces (no phantom dependencies).
- `workspace:*` and `catalog:` protocols are supported.
- `bun build --compile` resolving workspace deps into a single binary is not explicitly documented — test early.

Full monorepo migration notes live in `docs/future-improvements.md` if and when this is promoted. See `docs/decisions.md` ADR-008.

## When to revisit this document

- Bun releases a version that changes compile behavior, subprocess behavior, or bundle size meaningfully
- A dependency (citty, ansis, zod) has a major version bump
- DuckDB ships the Bun fix on npm (check `docs/future-improvements.md` for promotion trigger)
- A new coding gotcha bites you — add it here so it doesn't bite twice
