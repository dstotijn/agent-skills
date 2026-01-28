# Advanced Mise Usage

## Migrating from .tool-versions

If a project has `.tool-versions` (asdf format), mise reads it automatically. To convert to mise.toml:

```bash
mise use --path mise.toml $(cat .tool-versions | awk '{print $1"@"$2}' | tr '\n' ' ')
```

Then remove `.tool-versions` after verifying.

## Environment Variables and Tasks

mise.toml supports environment variables and tasks:

```toml
[env]
DATABASE_URL = "postgres://localhost/dev"
NODE_ENV = "development"

[tasks.dev]
run = "npm run dev"
description = "Start development server"

[tasks.test]
run = "npm test"
description = "Run tests"
```

Run tasks with:
```bash
mise run dev
mise run test
```

## Parallel Tasks

Use `depends` to run multiple tasks in parallel. Tasks without dependencies on each other run concurrently:

```toml
[tasks.dev]
description = "Run backend and frontend dev servers"
depends = ["dev:server", "dev:web"]

[tasks."dev:server"]
description = "Run Go backend"
run = "go run ./cmd/server"

[tasks."dev:web"]
description = "Run frontend dev server"
dir = "web"
run = "pnpm dev"
```

Now `mise run dev` starts both servers in parallel.

## Forcing Color Output

When running tasks in parallel, output may lose color because stdout is not a TTY. Use `env` to force color output:

```toml
[tasks."dev:web"]
description = "Run frontend dev server"
dir = "web"
run = "pnpm dev"
env = { FORCE_COLOR = "1" }
```

`FORCE_COLOR=1` is respected by most Node.js tools (Vite, esbuild, etc.).

## Troubleshooting

**Tool not found after install:**
```bash
mise doctor       # Check for issues
mise reshim       # Regenerate shims
exec $SHELL       # Reload shell
```

**Version conflicts:**
```bash
mise ls --current   # See which config is active
mise where node     # Find tool location
```

**Check mise is activated:**
```bash
which node          # Should point to mise shim
mise doctor         # Verify shell integration
```
