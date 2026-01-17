---
name: mise
description: This skill should be used when the user asks to "start a new project", "init project", "create a project", "scaffold project", "install node", "add python", "set up a project", "install dependencies", or when working in a project that uses development tools like Node.js, Python, Go, Ruby, or Rust. Also use when encountering global tool usage (npm -g, pip install without venv, uv tool install) to redirect to mise-managed tools.
---

# Mise Tool Management

Mise is a polyglot version manager for development tools. All tool installations and version management should go through mise to ensure reproducibility and avoid global tool pollution.

## Starting New Projects

When creating a new project, always create a `mise.toml` as part of the initial setup. This ensures tooling is version-controlled from the start.

## Core Principle: Always Pin Versions

Always use fully pinned versions in mise.toml for reproducibility. Never use partial versions or "latest".

```toml
# Good - fully pinned
node = "22.11.0"
python = "3.12.7"
go = "1.23.4"

# Bad - unpinned or partial
node = "22"
python = "latest"
uv = "latest"
```

To find the current version to pin:
```bash
mise ls-remote node | tail -1     # Latest available
mise current node                  # Currently active version
```

## Core Principle: No Global Tools

Never install tools globally. This includes:
- `npm install -g` or `npm i -g`
- `pip install` outside a virtual environment
- `uv tool install`
- `gem install` globally
- `go install` to GOPATH

Instead, use mise to manage all tool versions per-project or globally via mise.

## Checking for mise.toml

When working in a project, first check if mise configuration exists:

```bash
ls -la mise.toml .mise.toml .tool-versions 2>/dev/null
```

If none exists and the project uses development tools, propose creating a `mise.toml`.

## Creating mise.toml

For a new project or one missing mise configuration:

```toml
[tools]
node = "22.11.0"
python = "3.12.7"

[env]
# Project-specific environment variables
_.path = ["./node_modules/.bin", "./scripts"]
```

### Common Tool Configurations

**Node.js project:**
```toml
[tools]
node = "22.11.0"

[env]
_.path = ["./node_modules/.bin"]
```

**Python project:**
```toml
[tools]
python = "3.12.7"
uv = "0.5.11"

[env]
_.path = ["./.venv/bin"]
```

**Go project:**
```toml
[tools]
go = "1.23.4"
```

**Multi-language project:**
```toml
[tools]
node = "22.11.0"
python = "3.12.7"
go = "1.23.4"

[env]
_.path = ["./node_modules/.bin", "./.venv/bin"]
```

## Installing Tools

After creating or modifying mise.toml:

```bash
mise install
```

To install a specific tool:

```bash
mise use node@22.11.0        # Add to mise.toml and install
mise install node@22.11.0    # Install without modifying mise.toml
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

3. **CLI tools**: Add to mise.toml with pinned version
   ```bash
   mise use ripgrep@14.1.1
   mise use jq@1.7.1
   ```

## References

- **`references/advanced.md`** - Migration from asdf, environment variables, tasks, and troubleshooting
- **`references/lefthook.md`** - Setting up lefthook git hooks manager with mise postinstall automation
- **`references/uv.md`** - Python package management with uv, pyproject.toml setup, and mise integration
