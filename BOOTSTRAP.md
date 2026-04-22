# BOOTSTRAP

First-time setup runbook for `observe-cli`. Read once, top to bottom.

**Already set up?** Skip to `docs/phases/phase-1.md`.
**Looking up a pattern or decision?** See `docs/stack-reference.md`.

**If your repo already exists with planning docs** (`docs/overview.md`, `docs/mvp-spec.md`, `docs/phases/`, etc.), skip the "Create the repo" step and start at "Pin the Bun version." Everything else applies as written.

## Prerequisites
- **Bun v1.3.12 or newer.** Install: `curl -fsSL https://bun.sh/install | bash`
- **Git.**
- **SSH access** to staging (and eventually production) without password prompts. Use SSH keys or agent forwarding. This tool does not store credentials.

Verify Bun:
```bash
bun --version  # expect 1.3.12 or newer
```

## Create the repo
*(Skip if the repo already exists.)*

```bash
mkdir observe-cli && cd observe-cli
git init
```

## Pin the Bun version

```bash
echo "1.3.12" > .bun-version
```

## Create `package.json`

```json
{
  "name": "observe-cli",
  "version": "0.0.1",
  "type": "module",
  "private": true,
  "bin": {
    "observe-cli": "./src/index.ts"
  },
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "build": "bun build --compile --minify-whitespace --minify-syntax --sourcemap --bytecode --outfile dist/observe-cli src/index.ts",
    "test": "bun test",
    "check": "bunx --bun biome check --write ./src",
    "typecheck": "bunx --bun tsc --noEmit"
  },
  "dependencies": {
    "citty": "^0.2.2",
    "ansis": "^3.17.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "@biomejs/biome": "^2.4.0",
    "typescript": "^5.8.0"
  }
}
```

**Why Zod v3 and not v4:** Zod v4 currently has a Bun install bug where `.d.cts` files contain unterminated quotes. Stick with v3 until fixed.

## Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*", "test/**/*"]
}
```

## Create `.gitignore`

```
node_modules/
dist/
*.db
*.db-journal
.DS_Store
```

## Install dependencies

```bash
bun install
```

This must come before the next steps — `biome init` needs Biome in `node_modules`, and `src/index.ts` imports `citty`.

## Generate `biome.json`

```bash
bunx @biomejs/biome init
```

Then open `biome.json` and set `formatter.lineWidth` to `100`. Biome v2 changed the config schema — let `init` generate it rather than hand-writing.

## Create `src/index.ts`

```bash
mkdir src
```

Then create `src/index.ts` with this content:

```typescript
#!/usr/bin/env bun
import { defineCommand, runMain } from "citty"

const main = defineCommand({
  meta: {
    name: "observe-cli",
    version: "0.0.1",
    description: "MigrationPro observability CLI",
  },
  run() {
    console.log("observe-cli scaffold is running")
  },
})

runMain(main)
```

## First-run verification

```bash
bun run dev --help
```

Expected: citty prints auto-generated help output showing the command name, version, and description. If this works, the scaffold is done.

Optional sanity checks (should not error):
```bash
bun run typecheck
bun run check
```

`bun test` is skipped at this stage — no tests exist yet. Tests arrive with Phase 1 Checkpoint 1.

## First commit

```bash
git add .
git commit -m "chore: scaffold Bun + TS CLI"
```

Update `CHANGELOG.md` under `[Unreleased] > Added`:
> - Scaffolded Bun + TypeScript CLI with citty, ansis, zod.

## Next step

Phase 1 Checkpoint 1 picks up here. See `docs/phases/phase-1.md`. The config loader, env validation, and tests are Checkpoint 1's work — this document only covers the scaffold.

## Setup gotchas reference

- **Zod v3, not v4** — see note in the `package.json` section.
- **Biome v2 config via `init`** — hand-writing breaks; let the CLI generate it.
- **Pin Bun version** in `.bun-version`. Bun moves fast between minors.
- **SSH must be passwordless.** The tool spawns `ssh` as a child process and cannot handle prompts.

For everything beyond setup — stack rationale, build flags, coding gotchas — see `docs/stack-reference.md`.
