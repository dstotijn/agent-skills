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
