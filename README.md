# agent-skills

A Claude Code plugin with assorted user-level skills by David Stotijn.

## Installation

```bash
claude plugins add /path/to/agent-skills
```

Or for development:

```bash
claude --plugin-dir /path/to/agent-skills
```

## Skills

### mise

Tool version management with [mise](https://mise.jdx.dev/). Ensures:

- All development tools are managed through mise (no global npm/pip/etc.)
- Projects have a `mise.toml` with pinned versions
- New projects start with proper tool configuration

Triggers on: "start a new project", "install node", "add python", "set up a project", or when encountering global tool usage.
