---
name: mise
description: Use this skill whenever setting up a new project, initializing a repo, scaffolding, or adding development tools. Trigger phrases include "start a new project", "init project", "create a project", "scaffold", "install node", "add python", "set up a project", "install dependencies". Also use when encountering ANY global tool installation — npm -g, pip install without venv, uv tool install, brew install <runtime>, cargo install, curl|sh installer scripts, or when a README instructs users to install a tool globally. Redirect all of these to mise-managed tools. Use this skill even if the user doesn't mention mise — if they're about to install a dev tool globally or start a project without version-pinned tooling, this skill applies.
---

# Mise Tool Management

Mise is a polyglot version manager for development tools. All tool installations and version management should go through mise to ensure reproducibility and avoid global tool pollution.

## Starting New Projects

When creating a new project, set up mise-managed tooling as part of the initial setup. This ensures tooling is version-controlled from the start.

Add tools using `mise use --pin`, which fetches the latest stable version and pins it exactly in `mise.toml`:

```bash
mise use node --pin       # Adds node with latest version pinned
mise use python --pin     # Adds python with latest version pinned
mise use go --pin         # Adds go with latest version pinned
```

After adding tools, configure environment if needed by editing `mise.toml` directly:

```toml
[env]
_.path = ["./node_modules/.bin"]
```

Then trust and install:

```bash
mise trust                # Trust the new mise.toml config
mise install              # Install all tools
```

## Core Principle: Always Use `mise use --pin`

Never manually write tool versions in `mise.toml`. Always use `mise use <tool> --pin` to add or update tools. This ensures you always get the latest stable version with an exact pin — no stale versions, no partial pins, no guessing.

## Core Principle: No Global Tools

Never install development tools globally. Every tool should be managed by mise so that versions are pinned, reproducible, and project-scoped. This applies to:

- `npm install -g` or `npm i -g`
- `pip install` outside a virtual environment
- `uv tool install`
- `gem install` globally
- `go install` to GOPATH
- `brew install node`, `brew install python`, `brew install go` (or any dev runtime via Homebrew)
- `cargo install` globally
- `curl | sh` or `curl | bash` installer scripts for dev tools

Instead, use `mise use <tool> --pin` to add the tool to the project's `mise.toml`.

If mise is not installed in the current environment, install it first:
```bash
curl https://mise.run | sh
```

## Checking for mise.toml

When working in a project, first check if mise configuration exists:

```bash
ls -la mise.toml .mise.toml .tool-versions 2>/dev/null
```

If none exists and the project uses development tools, propose creating a `mise.toml`.

## Setting Up Tools by Project Type

Use `mise use --pin` to add tools. Then edit `mise.toml` for environment config if needed.

**Node.js project:**
```bash
mise use node --pin
```
Then add to `mise.toml`:
```toml
[env]
_.path = ["./node_modules/.bin"]
```

**Python project:**
```bash
mise use python --pin
mise use uv --pin
```
Then add to `mise.toml`:
```toml
[env]
_.path = ["./.venv/bin"]
```

**Go project:**
```bash
mise use go --pin
```

**Multi-language project:**
```bash
mise use node --pin
mise use python --pin
mise use go --pin
```
Then add to `mise.toml`:
```toml
[env]
_.path = ["./node_modules/.bin", "./.venv/bin"]
```

## Installing and Updating Tools

```bash
mise install                  # Install all tools from mise.toml
mise use node --pin           # Add/update tool to latest pinned version
mise install node             # Install without modifying mise.toml
mise trust                    # Trust mise.toml in current directory (required on first use)
```

## Checking Installed Tools

```bash
mise ls           # List installed tools
mise current      # Show active versions in current directory
mise doctor       # Diagnose mise setup issues
```

## Handling Global Tool Requests

When the user asks to install something globally:

1. **npm packages**: Add as dev dependency or use npx
   ```bash
   npm install --save-dev prettier   # Instead of npm i -g prettier
   npx prettier                       # Run without installing
   ```

2. **Python packages**: Use uv with mise, or project venv
   ```bash
   # In mise.toml, ensure uv is available
   uv pip install package  # Installs to project venv
   ```

3. **CLI tools**: Add to mise with pinned version
   ```bash
   mise use ripgrep --pin
   mise use jq --pin
   ```

## References

- **`references/advanced.md`** - Migration from asdf, environment variables, tasks, and troubleshooting
- **`references/lefthook.md`** - Setting up lefthook git hooks manager with mise postinstall automation
- **`references/uv.md`** - Python package management with uv, pyproject.toml setup, and mise integration
