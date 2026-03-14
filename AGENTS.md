# AGENTS.md

Instructions for AI coding agents working with this repository.

## Commands

```bash
mise run install             # Install dev dependencies
mise run validate            # Validate all skills
mise run validate:skill mise # Validate a single skill
```

## Workflow

When creating or updating skills, use the `anthropics:skill-creator` skill for guided authoring and validation.
After creating or updating a skill, always run `mise run validate` to ensure the skill is valid.
