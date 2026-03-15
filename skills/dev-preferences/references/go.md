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
