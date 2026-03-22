# Go Patterns

## Table of Contents

- [Naming](#naming) — MixedCaps, don't stutter, packages, getters, interfaces, named returns
- [Logging](#logging) — slog, structured key-value pairs
- [Service Structs](#service-structs) — constructor pattern, functional options
- [Testing](#testing) — table-driven, go-cmp, subtest naming
- [Composition](#composition) — consumer-side interfaces, avoid unnecessary abstractions
- [Control Flow](#control-flow) — guard clauses, early returns
- [Error Handling](#error-handling) — wrapping, sentinels, errors.Join, structured errors
- [HTTP Handlers](#http-handlers) — stdlib net/http, unexported handler struct
- [Project Layout](#project-layout) — domain-based, no generic packages
- [Graceful Shutdown](#graceful-shutdown) — signal.NotifyContext, force quit
- [Concurrency](#concurrency) — channels, select, defer
- [Slices](#slices) — nil slices, declaration style
- [Error Strings](#error-strings) — lowercase, no punctuation
- [Crypto Rand](#crypto-rand) — never math/rand for keys
- [Import Grouping](#import-grouping) — stdlib first, blank line, third-party
- [Goroutine Lifetimes](#goroutine-lifetimes) — document exits, prevent leaks
- [Receiver Type](#receiver-type) — pointer vs value guidelines
- [Pass Values](#pass-values) — don't use pointers to save bytes
- [Synchronous Functions](#synchronous-functions) — prefer sync, let callers add concurrency
- [In-Band Errors](#in-band-errors) — use (value, ok) or (value, error)
- [Module Hygiene](#module-hygiene) — go mod tidy, go.sum, repeatable builds

## Naming

Naming is one of the most important aspects of idiomatic Go. Good names are short, precise, and carry meaning through context rather than length. A name's length should be proportional to its scope — a loop index can be `i`, but a package-level export needs to be clear at a distance.

### General rules

- **MixedCaps** always, never underscores: `MarshalJSON` not `Marshal_JSON`
- **Acronyms stay capitalized**: `ID`, `HTTP`, `URL`, `API` (not `Id`, `Http`, `Url`, `Api`)
- **Unexported by default** — only export what other packages actually need
- **Short local variables**: `cfg`, `buf`, `req`, `res`, `srv`, `ctx`, `err`
- **Single-letter receivers**: `func (s *Service) Run()`, `func (h *Handler) ServeHTTP(...)`
- **Single-letter loop/short-scope vars**: `i`, `n`, `k`, `v`, `b`
- **`context.Context` is always the first parameter**: `func FindUser(ctx context.Context, id string) (User, error)`

### Don't stutter

The package name is part of every qualified reference. Names should read well with the package prefix — never repeat the package name in the identifier:

```go
// Good — reads as "user.New", "user.Repository", "http.Server".
package user
func New(...) *Service
type Repository interface { ... }

// Bad — reads as "user.NewUser", "user.UserRepository", "http.HTTPServer".
package user
func NewUser(...) *User
type UserRepository interface { ... }
```

More examples:
- `ring.New` not `ring.NewRing`
- `order.Service` not `order.OrderService`
- `bytes.Buffer` not `bytes.ByteBuffer`
- `context.WithTimeout` not `context.NewContextWithTimeout`

### Packages

- Lowercase, single-word: `bytes`, `http`, `user` — not `httpUtils` or `user_service`
- The package name is the base name of its source directory: `encoding/base64` has package name `base64`, not `encodingBase64`
- Name after what it provides, not what it contains
- The package name is context for its exports — `bufio.Reader` not `bufio.BufReader`, because users already see the `bufio.` prefix
- Use the package structure to choose good names: if `ring` exports only `Ring`, the constructor is `ring.New` not `ring.NewRing`
- Avoid generic names: no `util`, `common`, `helpers`, `base`, `misc`
- Don't worry about name collisions across packages — the import path disambiguates, and callers can alias if needed

### Getters and setters

No `Get` prefix on getters — the exported field name *is* the getter:

```go
func (u *User) Name() string       // Not GetName().
func (u *User) SetName(name string) // Setter keeps the Set prefix.
```

This applies broadly: `Owner()` not `GetOwner()`, `Count()` not `GetCount()`.

### Interfaces

Single-method interfaces use the method name plus `-er`:

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Stringer interface { String() string }
type Closer interface { Close() error }
```

Use canonical method signatures (`Read`, `Write`, `Close`, `String`, `Err`) — don't invent alternatives with different signatures. If your type implements `Read`, use the exact `io.Reader` signature so it satisfies the interface.

### Named return values

Use named returns to document what's being returned, especially when multiple values have the same type:

```go
func (f *File) Read(b []byte) (n int, err error)
func nextInt(b []byte, pos int) (value, nextPos int)
```

Don't use naked returns in long functions — they obscure what's being returned.

## Logging

Use `log/slog` for structured logging — never `log.Printf`, `logrus`, or `zap`. Pass `*slog.Logger` as a dependency, don't use the global logger.

```go
log := slog.New(slog.NewJSONHandler(os.Stdout, nil))
log.Info("User created.", "id", user.ID, "role", user.Role)
```

Log messages are short, capitalized labels. All context goes in key-value pairs — never interpolate values into the message string.

```go
// Good — structured.
log.Error("Request failed.", "method", r.Method, "path", r.URL.Path, "err", err)

// Bad — values baked into the message.
log.Error(fmt.Sprintf("Request to %s %s failed: %v", r.Method, r.URL.Path, err))
```

## Service Structs

Use a `Service` struct as the main entry point for a domain. Dependencies go in as fields, configured via constructor.

```go
type Service struct {
    repo Repository
    log  *slog.Logger
}

func New(repo Repository, log *slog.Logger) *Service {
    return &Service{repo: repo, log: log}
}
```

For libraries or complex setup, use the **functional options** pattern:

```go
type Option func(*Service)

func WithLogger(log *slog.Logger) Option {
    return func(s *Service) { s.log = log }
}

func New(repo Repository, opts ...Option) *Service {
    s := &Service{repo: repo, log: slog.Default()}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

For simpler internal services, a plain `Config` struct is fine — don't reach for functional options when a struct with a few fields does the job.

## Testing

Table-driven tests with human-readable subtest names and assertion messages. Use `google/go-cmp/cmp` for comparisons.

```go
tests := []struct {
    name string
    // Test-specific fields go here.
}{
    {name: "returns error when user not found"},
    {name: "creates user with default role"},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        // ...
        if diff := cmp.Diff(want, got); diff != "" {
            t.Errorf("unexpected result (-want +got):\n%s", diff)
        }
    })
}
```

- Subtest names describe the scenario in plain English, not identifiers: `"returns error when user not found"` not `"TestFindUserByID_NotFound"`.
- Use `t.Helper()` in test helpers so failures point to the caller.
- Don't use testify or `reflect.DeepEqual`.

## Composition

Prefer interfaces and embedding over inheritance-like patterns. Keep interfaces small — one or two methods.

Define interfaces where they're consumed, not where they're implemented:

```go
// In the consumer package.
type Repository interface {
    FindUserByID(ctx context.Context, id string) (User, error)
}
```

Don't introduce interface abstractions just for testability when the stdlib provides test infrastructure. For example, services that make HTTP calls don't need an HTTP client interface — use `httptest.NewServer` to test against a real `*http.Client`.

## Control Flow

Use guard clauses and early returns — don't nest the success path inside conditionals:

```go
// Good — guard clause, success path flows down.
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
// Work with f.

// Bad — unnecessary nesting.
f, err := os.Open(name)
if err == nil {
    defer f.Close()
    // Work with f.
} else {
    return err
}
```

Omit the `else` when the `if` body ends with a `return`, `break`, `continue`, or similar.

## Error Handling

Return errors explicitly, handle at the call site. Wrap with context using `%w`:

```go
if err != nil {
    return fmt.Errorf("fetch user %q: %w", id, err)
}
```

- Don't panic for recoverable errors.
- Use sentinel errors (`var ErrNotFound = errors.New(...)`) for errors callers need to check.
- Use `errors.Is` / `errors.As` for error inspection.
- Use `errors.Join` to aggregate multiple errors (Go 1.20+):

```go
var errs []error
for _, item := range items {
    if err := process(item); err != nil {
        errs = append(errs, err)
    }
}
if err := errors.Join(errs...); err != nil {
    return err
}
```

- For structured error types, implement the `error` interface:

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %q not found", e.Resource, e.ID)
}
```

## HTTP Handlers

Use stdlib `net/http` with Go 1.22+ routing. `NewHandler` returns `http.Handler` — the struct is unexported, keeping the API surface minimal.

```go
func NewHandler(svc *Service, log *slog.Logger) http.Handler {
    h := &handler{svc: svc, log: log}
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", h.findUser)
    mux.HandleFunc("POST /users", h.createUser)
    return mux
}

type handler struct {
    svc *Service
    log *slog.Logger
}

func (h *handler) findUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    user, err := h.svc.FindUserByID(r.Context(), id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

Compose in `main.go` by mounting each domain's handler:

```go
mux := http.NewServeMux()
mux.Handle("/users/", user.NewHandler(userSvc, log))
mux.Handle("/orders/", order.NewHandler(orderSvc, log))
```

Don't use third-party routers (chi, gorilla, httprouter) — the stdlib mux covers most needs since Go 1.22.

### Timeouts

Always set timeouts on `http.Server` — never use the zero-value defaults, which allow connections to hang indefinitely.

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
```

When reading request bodies, use `http.MaxBytesReader` to prevent clients from sending unbounded payloads:

```go
r.Body = http.MaxBytesReader(w, r.Body, 512<<10) // 512 KB limit.
```

For outgoing HTTP requests, always set a timeout on `http.Client`. Never use `http.DefaultClient` in production — it has no timeout.

```go
client := &http.Client{
    Timeout: 10 * time.Second,
}
```

## Project Layout

Group by domain and intent. A typical service:

```
myservice/
  user/
    service.go
    handler.go
    repository.go
  order/
    service.go
    handler.go
  cmd/
    myservice/
      main.go
```

No generic packages (`utils`, `helpers`, `common`). If something doesn't belong to a domain, it probably belongs inline where it's used.

## Graceful Shutdown

Use `signal.NotifyContext` to tie the application lifecycle to OS signals. Pass the context down so all components can clean up.

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Error("Server failed.", "err", err)
    }
}()

<-ctx.Done()
log.Info("Shutting down... (Press ctrl+c to force quit)")

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

// Second ctrl+c forces immediate exit.
go func() {
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, os.Interrupt, syscall.SIGTERM)
    <-sig
    os.Exit(1)
}()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Error("Shutdown failed.", "err", err)
}
```

## Concurrency

Prefer channels for coordination, mutexes only for protecting shared state. Follow the Go proverb: don't communicate by sharing memory — share memory by communicating.

### Goroutines and channels

```go
results := make(chan Result, len(items))
for _, item := range items {
    go func() {
        results <- process(item)
    }()
}

var all []Result
for range items {
    all = append(all, <-results)
}
```

### Select

Use `select` to multiplex channels. Include a `default` case for non-blocking operations:

```go
select {
case msg := <-ch:
    handle(msg)
case <-ctx.Done():
    return ctx.Err()
}
```

### Defer

Use `defer` for cleanup immediately after acquiring a resource:

```go
mu.Lock()
defer mu.Unlock()

f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
```

Deferred calls run LIFO when the function returns. Arguments are evaluated at the `defer` statement, not at execution.

## Slices

Prefer nil slice declarations over empty literals:

```go
// Good — nil slice is the idiomatic default.
var t []string

// Avoid — unnecessary allocation, same behavior for append/len/range.
t := []string{}
```

Exception: when encoding JSON, use `[]string{}` if the consumer expects `[]` instead of `null`.

## Error Strings

Error strings are lowercase and have no trailing punctuation — they're often wrapped or prefixed by callers. This is distinct from log messages, which are capitalized labels ending with a period (see [Logging](#logging)).

```go
// Good.
fmt.Errorf("fetch user %q: %w", id, err)

// Bad — capital letter and period break when wrapped.
fmt.Errorf("Fetch user %q: %w.", id, err)
```

Proper nouns and acronyms keep their casing: `"TLS handshake failed"` is fine.

## Crypto Rand

Never use `math/rand` or `math/rand/v2` to generate keys, tokens, or anything security-sensitive. Use `crypto/rand`:

```go
// Keys and random bytes.
key := make([]byte, 32)
if _, err := crypto_rand.Read(key); err != nil {
    return err
}

// Random text (Go 1.24+).
token := crypto_rand.Text()
```

## Import Grouping

Group imports with a blank line between stdlib and third-party packages. Don't rename imports unless there's a collision.

```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/google/go-cmp/cmp"
    "mycompany.com/internal/user"
)
```

Side-effect imports (`import _ "pkg"`) belong only in `main` or test files.

## Goroutine Lifetimes

Every goroutine you spawn should have a clear exit condition. If it's not obvious when or whether a goroutine exits, document it — goroutines that block on channels or I/O indefinitely are a common source of leaks.

```go
// Fine — exits when ch is closed. Use when the producer guarantees close.
go func() {
    for msg := range ch {
        handle(msg)
    }
}()

// Better — exits on ctx cancellation even if ch is never closed.
go func() {
    for {
        select {
        case msg := <-ch:
            handle(msg)
        case <-ctx.Done():
            return
        }
    }
}()
```

When spawning goroutines in libraries, give callers a way to stop them (context cancellation, a `Close` method, or a done channel). `for range ch` is idiomatic when the channel lifecycle is well-defined — the concern is goroutines with no guaranteed exit path.

## Receiver Type

Choose pointer or value receivers consistently for a type — don't mix.

**Use pointer receivers** when:
- The method mutates the receiver.
- The struct contains `sync.Mutex` or other synchronization primitives.
- The struct is large (treat it like passing all fields as arguments).

**Use value receivers** when:
- The type is small and immutable (e.g., `time.Time`, coordinates, small value objects).
- The type is a map, func, or chan (already reference types).

When in doubt, use a pointer receiver.

## Pass Values

Don't use pointers just to save a few bytes. If a function only reads through `*x`, the argument shouldn't be a pointer. This applies especially to strings and interface values — `*string` and `*io.Reader` are almost never what you want.

```go
// Good — string is small and immutable.
func Greet(name string) string

// Bad — pointer adds indirection for no benefit.
func Greet(name *string) string
```

Use pointers when the value is genuinely large, when you need to distinguish "absent" from "zero", or when the caller needs to observe mutations.

## Synchronous Functions

Prefer synchronous functions that return results directly. Let callers add concurrency if they need it — don't bake goroutines and channels into your API.

```go
// Good — synchronous, caller decides on concurrency.
func FetchUser(ctx context.Context, id string) (User, error) {
    // ...
}

// Bad — forces async on every caller.
func FetchUser(ctx context.Context, id string) <-chan Result {
    ch := make(chan Result, 1)
    go func() { /* ... */ }()
    return ch
}
```

Synchronous APIs are easier to test, compose, and reason about. If the caller wants to run three fetches concurrently, they can wrap each in a goroutine themselves.

## In-Band Errors

Don't use magic return values to signal errors. Use multiple return values — either `(value, error)` or `(value, ok)`:

```go
// Good — caller can distinguish "not found" from "empty string".
func Lookup(key string) (value string, ok bool)

// Bad — "" could be a valid value or an error signal.
func Lookup(key string) string
```

This is especially important for maps, where the zero value is a valid entry:

```go
v, ok := m[key]
if !ok {
    // key genuinely absent
}
```

## Module Hygiene

Practices that keep builds repeatable and dependencies clean.

### Before every release

```bash
go mod tidy       # Remove unused deps, add missing ones.
go mod verify     # Verify checksums match go.sum.
go vet ./...      # Catch common mistakes.
go test ./...     # Run all tests.
```

### Commit both `go.mod` and `go.sum`

`go.mod` declares dependencies; `go.sum` contains cryptographic checksums that verify downloaded modules haven't been tampered with. Always commit both. Never edit either by hand — use `go get` and `go mod tidy`.

### Pin versions explicitly

```bash
go get example.com/pkg@v1.2.3   # Pin to exact version.
go get -u=patch ./...            # Upgrade to latest patch only.
```

Avoid `go get -u ./...` (upgrades all transitive deps to latest minor) unless you intend to upgrade everything — it can pull in breaking changes from indirect dependencies.

### Replace directives

Use `replace` for local development only. Remove them before committing unless the replacement is intentional and permanent (e.g., a fork).

```go
// Local development — never commit this.
replace example.com/pkg => ../pkg
```

### Private modules

Configure `GOPRIVATE` for internal modules so they bypass the public proxy and checksum database:

```bash
go env -w GOPRIVATE="github.com/mycompany/*"
```
