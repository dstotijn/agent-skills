---
name: go-dev
description: Go development conventions and idiomatic patterns for David Stotijn. Use this skill whenever writing, reviewing, or fixing Go code — including backend services, CLI tools, scripts, tests, and API handlers. Trigger on any Go-related task, even if the user doesn't mention conventions. Also consult when setting up a new Go project, choosing Go libraries, structuring packages, or debugging Go code.
---

# Go Dev

Idiomatic Go patterns and conventions. This skill builds on `dev-preferences` — consult that skill for cross-cutting concerns (code philosophy, API design, git, deployment, dependency management approach).

## Tech Stack

| Concern      | Choice                              |
|--------------|---------------------------------------|
| Database     | PostgreSQL or SQLite (modernc)        |
| SQL layer    | sqlc — never ORMs (no GORM, no ent) |
| Migrations   | goose                                 |
| Config       | koanf (YAML, JSON, env vars)          |
| IDs          | Short, sortable, no-ambiguity (custom, no lib) |
| Testing      | stdlib `testing` + `google/go-cmp/cmp` — no testify |
| Linting      | staticcheck                           |
| Formatting   | gofmt / goimports                     |

## Key Principles

- Follow Effective Go conventions closely — naming is critical.
- Don't stutter: `user.New` not `user.NewUser`, no `Get` prefix on getters.
- Short identifiers, single-letter receivers, `context.Context` always first parameter.
- Guard clauses and early returns — don't nest the success path.
- Error strings are lowercase with no trailing punctuation. Log messages are capitalized labels ending with a period.
- Prefer synchronous APIs — let callers add concurrency.
- Group by domain and intent, not by technical layer. No generic packages (`utils`, `helpers`, `common`).

## Reference

See **`references/patterns.md`** for the full guide covering: naming, logging, service structs, testing, composition, control flow, error handling, HTTP handlers, project layout, graceful shutdown, concurrency, slices, error strings, crypto rand, import grouping, goroutine lifetimes, receiver types, pass values, synchronous functions, in-band errors, and module hygiene.
