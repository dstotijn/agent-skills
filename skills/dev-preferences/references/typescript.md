# TypeScript Patterns

## Module Conventions

- Named exports only — no default exports.
- No barrel files (no `index.ts` re-exporting from a directory). Import directly from the source module.
- No parent-relative imports (`../`). Use the `@/` path alias for cross-directory imports. Sibling imports (`./`) within the same directory are fine.

```ts
// Good — path alias for cross-directory.
import { UserCard } from "@/users/user-card"

// Good — sibling import within the same directory.
import { formatName } from "./format-name"

// Bad — traversing up.
import { UserCard } from "../../users/user-card"
import { UserCard } from "../components"
```

### Path alias setup

Configure `@/` to point to the source root in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Enforce the parent-relative import ban via linting. After running `npx @biomejs/biome init` (never write `biome.json` from scratch), add the relevant rules to the generated config. If Biome doesn't yet support banning parent-relative imports, use ESLint's `no-restricted-imports` as a fallback:

```json
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": ["../*"]
    }]
  }
}
```

## tsconfig

Follow [Matt Pocock's TSConfig Cheat Sheet](https://www.totaltypescript.com/tsconfig-cheat-sheet). Base options for a frontend project using a bundler:

```json
{
  "compilerOptions": {
    "esModuleInterop": true,
    "skipLibCheck": true,
    "target": "es2022",
    "allowJs": true,
    "resolveJsonModule": true,
    "moduleDetection": "force",
    "isolatedModules": true,
    "verbatimModuleSyntax": true,

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    "module": "preserve",
    "noEmit": true,
    "lib": ["es2022", "dom", "dom.iterable"],

    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

Key settings:
- **`verbatimModuleSyntax`** — forces `import type` / `export type` for type-only imports.
- **`noUncheckedIndexedAccess`** — prevents accessing arrays/objects without checking they're defined.
- **`module: "preserve"`** — for bundler-based projects (no TS transpilation). Use `"NodeNext"` if TS handles transpilation directly.
- **`strict: true`** — non-negotiable.

## Types

Use `type` by default, not `interface`. `type` is more versatile (unions, mapped types, conditional types) and avoids the footgun of declaration merging where two `interface` declarations with the same name silently merge.

Use `interface` only when you need `extends` for object inheritance — it lets TypeScript cache the type by name, making it slightly faster than `&` intersections.

```ts
// Good.
type User = {
    id: string
    name: string
}

type Result = Success | Failure

// Only when inheriting.
interface AdminUser extends User {
    role: "admin"
}
```

Colocate types with the code that uses them. Don't create separate `types.ts` files — define types in the module where they're consumed or exported.

### No `as any` or `as unknown`

Never use `as any` or `as unknown` to bypass type checking. If the type system is fighting you, fix the types instead. Use type guards, discriminated unions, or `satisfies` to narrow types safely:

```ts
// Bad — silences the compiler.
const user = response.data as any

// Good — use a type guard to narrow safely.
function isUser(data: unknown): data is User {
    return typeof data === "object" && data !== null && "id" in data
}

// Good — use a discriminated union.
type Result =
    | { status: "ok"; data: User }
    | { status: "error"; message: string }

// Good — satisfies validates without widening.
const config = {
    port: 3000,
    host: "localhost",
} satisfies ServerConfig
```

## File Naming

Kebab-case for all TypeScript files — components, hooks, utilities, route modules, schemas:

```
user-card.tsx
use-user.ts
format-name.ts
user-schema.ts
```

## React Components

Function components only. One exported component per file, named after the component. Small private helper components can live in the same file.

Colocate related files — a component's tests, hooks, and types live next to it, not in separate directories:

```
users/
  user-card.tsx
  user-card.test.tsx
  use-user.ts
```

## Hooks

Prefix custom hooks with `use`. Keep them focused — one concern per hook.

```ts
export function useUser(id: string) {
    // ...
}
```

## Testing

Use Vitest. Colocate test files next to the code they test (`foo.test.ts` next to `foo.ts`).

```ts
import { describe, expect, it } from "vitest"

describe("createUser", () => {
    it("assigns default role when none provided", () => {
        // ...
    })
})
```

Test descriptions should be human-readable sentences, same philosophy as Go subtests.

## Data Fetching

Use TanStack Query for server state. Use TanStack Router's route loaders for data that should be available before a route renders.

```ts
// Route loader — data ready before render.
export const Route = createFileRoute("/users/$userId")({
    loader: ({ params }) => queryClient.ensureQueryData(userQueryOptions(params.userId)),
})

// TanStack Query — for reactive server state in components.
export function useUser(id: string) {
    return useQuery(userQueryOptions(id))
}

function userQueryOptions(id: string) {
    return queryOptions({
        queryKey: ["users", id],
        queryFn: () => fetchUser(id),
    })
}
```

Don't use `useEffect` for data fetching — use TanStack Query or route loaders instead.

## Validation

Use Zod to validate data at API boundaries. Don't trust `as` assertions on external data — parse it.

```ts
import { z } from "zod"

const UserSchema = z.object({
    id: z.string(),
    name: z.string(),
    email: z.string().email(),
})

type User = z.infer<typeof UserSchema>

// Parse at the boundary.
async function fetchUser(id: string): Promise<User> {
    const response = await fetch(`/api/users/${id}`)
    const data = await response.json()
    return UserSchema.parse(data)
}
```

## Logging

Use `console` methods (`console.log`, `console.error`, `console.warn`). Messages are short, capitalized labels ending with a period. All context goes in a structured object — never interpolate values into the message string.

```ts
// Good — structured.
console.error("Request failed.", { method, path, status })
console.info("User created.", { id: user.id, role: user.role })

// Bad — values baked into the message.
console.error(`Request to ${method} ${path} failed with ${status}`)
```

## Error Handling

Use `throw`/`catch` for errors. Don't return error tuples or result objects — that's a Go pattern, not idiomatic TypeScript.

```ts
// Good — throw and let the caller catch.
function findUser(id: string): User {
    const user = users.get(id)
    if (!user) {
        throw new Error(`User ${id} not found`)
    }
    return user
}

// Good — custom error class when callers need to distinguish.
class NotFoundError extends Error {
    constructor(public resource: string, public id: string) {
        super(`${resource} ${id} not found`)
        this.name = "NotFoundError"
    }
}
```

For React, use error boundaries to catch rendering errors — don't wrap every component in try/catch.

## Async

Use `async`/`await` — never `.then()` chains. Handle errors with try/catch at the appropriate boundary.

```ts
// Good — async/await.
async function fetchUser(id: string): Promise<User> {
    const response = await fetch(`/api/users/${id}`)
    if (!response.ok) {
        throw new Error(`Failed to fetch user: ${response.status}`)
    }
    return response.json()
}

// Bad — .then() chain.
function fetchUser(id: string): Promise<User> {
    return fetch(`/api/users/${id}`)
        .then((r) => r.json())
        .then((data) => data as User)
}
```
