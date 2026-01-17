# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
mise run install          # Install dependencies
mise run validate         # Validate all skills
mise run validate:skill mise  # Validate a single skill
```

## Workflow

After creating or updating a skill, always run `mise run validate` to ensure the skill is valid.

For comprehensive validation against plugin best practices (description quality, progressive disclosure, writing style), use the plugin-dev validator agent:

```
Review this plugin against best practices
```

This requires the plugin-dev plugin (`claude plugins add claude-plugins-official/plugin-dev`).

## Structure

This is a Claude Code plugin:

```
.claude-plugin/
└── plugin.json           # Plugin manifest
skills/
└── skill-name/
    ├── SKILL.md          # Required: frontmatter (name, description) + instructions
    └── references/       # Optional: detailed documentation loaded as needed
```
