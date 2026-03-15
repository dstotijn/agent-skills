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

## Skill maintenance

SKILL.md is always loaded when a skill triggers, but reference files are only read on demand. When updating a skill's `references/`, keep the pointers in SKILL.md accurate so the model knows what's in each reference and when to consult it. SKILL.md should summarize enough to guide decisions (e.g. tech stack choices, key constraints), while detailed patterns, examples, and setup instructions belong in reference files.
