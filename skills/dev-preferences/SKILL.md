---
name: dev-preferences
description: Personal development preferences and conventions for David Stotijn. Use this skill whenever writing code, starting a new project, making architectural decisions, choosing libraries or tools, setting up CI/CD, structuring a codebase, or picking between implementation approaches. This skill applies to any coding task — even if the user doesn't mention preferences, consult it to stay aligned with their established conventions. Trigger whenever you're about to scaffold, write new code, choose a framework, pick a database, design an API, or set up tooling.
---

# Dev Preferences

Optimize for boring, readable code that's easy to maintain. Consistency matters more than perfection — when in doubt, follow what's here, but use judgment and flag cases where deviating would genuinely be better.

## Tech Stack

### Languages

TypeScript and Go. Choose based on the task:

- **Go** for backend services, CLI tools, and scripting.
- **TypeScript** for frontend.

### Frontend

| Concern        | Choice                          |
|----------------|---------------------------------|
| Framework      | TanStack Start (TanStack Router) |
| Data fetching  | TanStack Query                  |
| Forms          | TanStack Form                   |
| Validation     | Zod                             |
| Styling        | Tailwind CSS                    |
| Date/time      | date-fns (+ date-fns-tz)        |
| Lint/format    | Biome                           |
| Testing        | Vitest                          |
| Runtime        | Node.js with pnpm               |

Key constraints: named exports only, no default exports, no barrel files, no parent-relative imports (`../`) — use `@/` alias or `./` for siblings, `type` over `interface`, no `as any` or `as unknown` type assertions, colocate types with usage (no `types.ts` files), kebab-case for all file names.

See **`references/typescript.md`** for module conventions, type patterns, error handling, and examples.

### Backend (Go)

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

Naming is critical — follow Effective Go conventions closely: don't stutter (`user.New` not `user.NewUser`), no `Get` prefix on getters, short identifiers, single-letter receivers.

See **`references/go.md`** for full naming guide, service structs, HTTP handlers, control flow, testing, error handling, composition, concurrency, and project layout.

### Tooling

All dev tools managed via **mise** (see the `mise` skill). Never install tools globally.

### Editor & Formatter Config

- Copy **`assets/.editorconfig`** into new projects. Tabs, LF line endings, UTF-8.
- For Biome: always run `npx @biomejs/biome init` to generate `biome.json` — never write it from scratch. Then customize the generated config as needed.

## Code Style

- Concise, minimal, boring. Prefer readable over clever.
- Be idiomatic per language — don't write Go like TypeScript or vice versa.
- Don't create tons of tiny functions — inline logic when it's only used once.
- Only add abstractions when they earn their keep (used in 3+ places, or significantly clarify intent).
- Comments start with a capital letter and end with a period.
- Comment exported functions and public APIs — focus on **why**, not what.
- Go: follow godoc style (comment starts with the function/type name).
- TypeScript: plain `//` comments, no JSDoc.
- Use `TODO:` and `FIXME:` markers. Commented-out code is acceptable temporarily when paired with a TODO.
- No section divider comments (`// --- Handlers ---`).
- Organize by **domain and intent**, not by technical layer. Keep flat, avoid deep nesting.

See reference files for language-specific patterns, error handling, and project layout.

## Codegen & Static Analysis

- Prefer generating code from schemas and specs over writing it by hand — applies to SQL (sqlc), APIs (OpenAPI, protobuf), types, and clients.
- Lean on static analysis and linters (staticcheck, Biome) as a first line of defense. Don't disable rules to make code compile — fix the underlying issue.
- Never manually edit dependency files (`go.mod`, `package.json`). Use `go get`/`go mod tidy` and `pnpm add`/`pnpm remove` to manage dependencies.

## API Design

- **Simple services**: REST with JSON. Always define an **OpenAPI spec** and generate code from it.
- **Complex services**: ConnectRPC with Protocol Buffers.

## Dependencies

Prefer the libraries listed in the tech stack tables above. Don't avoid dependencies on principle, but prefer stdlib and focused libs over large frameworks.

Client state: React `useState`/`useContext` by default. Reach for Zustand when context gets unwieldy.

When a library outside the curated set would save significant effort or improve correctness, propose it — but explain why it's a good fit.

## Git

- **Commit messages**: short, imperative. `Add user endpoint`, not `feat(users): added new user endpoint`.
- **Merges**: squash merge to keep history clean.
- Don't commit until changes have been reviewed.

## Deployment

Fly.io or self-hosted. Always ask before choosing a deployment target — don't assume.
