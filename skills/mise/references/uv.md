# Python with uv and Mise

uv is a fast Python package manager. When managed through mise, it provides reproducible Python environments.

## Setup

In `mise.toml`:

```toml
[tools]
python = "3.14.2"
uv = "0.9.26"

[settings]
python.uv_venv_auto = true
```

The `uv_venv_auto` setting automatically creates and activates a `.venv` when entering the project directory.

## Project Configuration

Create `pyproject.toml`:

```toml
[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.14"
dependencies = [
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "ruff>=0.4.0",
]
```

## Common Commands

```bash
uv sync              # Install dependencies from pyproject.toml/uv.lock
uv sync --dev        # Include dev dependencies
uv add requests      # Add a dependency
uv add --dev pytest  # Add a dev dependency
uv run python app.py # Run command in venv
uv run pytest        # Run pytest in venv
```

## Mise Tasks with uv

Define tasks in `mise.toml` that use `uv run`:

```toml
[tasks.install]
description = "Install dependencies"
run = "uv sync"

[tasks.test]
description = "Run tests"
run = "uv run pytest"

[tasks.lint]
description = "Run linter"
run = "uv run ruff check ."
```

## Git Dependencies

For packages from git repositories:

```toml
[project]
dependencies = [
    "package @ git+https://github.com/org/repo.git",
    "package @ git+https://github.com/org/repo.git#subdirectory=subdir",
]
```

## Lock File

`uv.lock` should be committed for reproducible builds. It's automatically created/updated when running `uv sync` or `uv add`.
