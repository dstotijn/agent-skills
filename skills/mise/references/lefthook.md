# Lefthook with Mise

Lefthook is a fast git hooks manager. Manage it through mise to ensure consistent versions across the team.

## Adding Lefthook to a Project

In `mise.toml`:

```toml
[tools]
lefthook = "1.10.10"

[hooks]
postinstall = "lefthook install"
```

The postinstall hook automatically runs `lefthook install` after `mise install`, so git hooks are set up without manual steps.

Install:

```bash
mise install
```

## Lefthook Configuration

Create `lefthook.yml` in project root:

```yaml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: "*.{js,ts,tsx}"
      run: npx eslint {staged_files}
    format:
      glob: "*.{js,ts,tsx,json,md}"
      run: npx prettier --check {staged_files}

pre-push:
  commands:
    test:
      run: npm test
```

## Common Commands

```bash
lefthook install    # Install git hooks (runs automatically via postinstall)
lefthook uninstall  # Remove git hooks
lefthook run pre-commit  # Run hooks manually
```

## Team Onboarding

After cloning a repo with lefthook configured in mise.toml, team members only need:

```bash
mise install
```

The postinstall hook handles `lefthook install` automatically.
