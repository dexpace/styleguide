# 05 - API Design

## Accept Interfaces, Return Structs

- Accept the narrowest interface that satisfies the function's needs.
- Return concrete types. The caller decides when to abstract.

```go
// Good -- accepts interface, returns concrete type
func NewClient(transport http.RoundTripper) *Client { ... }

// Bad -- returns interface
func NewClient(transport http.RoundTripper) ClientInterface { ... }

// Bad -- accepts concrete type
func NewClient(transport *http.Transport) *Client { ... }
```

## Functional Options

Use functional options for configurable constructors:

```go
type clientConfig struct {
    timeout    time.Duration
    transport  http.RoundTripper
    retries    int
    baseURL    string
}

type Option func(*clientConfig)

func WithTimeout(duration time.Duration) Option {
    return func(config *clientConfig) {
        config.timeout = duration
    }
}

func WithTransport(transport http.RoundTripper) Option {
    return func(config *clientConfig) {
        config.transport = transport
    }
}

func WithRetries(count int) Option {
    return func(config *clientConfig) {
        config.retries = count
    }
}

func NewClient(baseURL string, opts ...Option) (*Client, error) {
    config := &clientConfig{
        timeout:   30 * time.Second,  // sensible default
        transport: http.DefaultTransport,
        retries:   3,
        baseURL:   baseURL,
    }

    for _, opt := range opts {
        opt(config)
    }

    // Validate after applying options
    if config.baseURL == "" {
        return nil, errors.New("base URL is required")
    }
    if config.timeout <= 0 {
        return nil, errors.New("timeout must be positive")
    }

    return &Client{config: config}, nil
}
```

Call site:

```go
client, err := NewClient("https://api.example.com",
    WithTimeout(10 * time.Second),
    WithRetries(5),
)
```

### Interface-Based Options (Public APIs)

For public APIs where you want to **prevent third-party callers from defining their own options**, implement options as a sealed interface with an unexported `apply` method rather than a function type. This restricts option construction to functions in your package.

```go
type Option interface {
    apply(*clientConfig)
}

type timeoutOption time.Duration

func (o timeoutOption) apply(c *clientConfig) { c.timeout = time.Duration(o) }

func WithTimeout(d time.Duration) Option {
    return timeoutOption(d)
}

func NewClient(baseURL string, opts ...Option) (*Client, error) {
    config := defaultConfig(baseURL)
    for _, opt := range opts {
        opt.apply(config)
    }
    // ...
}
```

Use this variant when:
- Options carry invariants the library must enforce — external callers should not be able to construct arbitrary options that violate them.
- You want to add or remove option types as unannounced internal changes (no closure-type dependency in caller code).

Use the simpler `type Option func(*config)` closure form when option extensibility is a non-goal — most internal or single-consumer APIs.

### Functional Options vs Config Struct

| Use | When |
|-----|------|
| Functional options | Public API with many optional parameters and sensible defaults |
| Config struct | Internal code, or when all fields are required, or when config must be serializable |

**Detailed trade-offs:**

Functional options excel when:
- You need backwards-compatible API evolution (add new `With*` functions without breaking callers).
- Most parameters have sensible defaults and callers only override a few.
- The constructor performs validation after applying all options.

Config structs excel when:
- All or most fields are required (no defaults to manage).
- The config must be deserialized from JSON/YAML/env.
- The config is passed across package boundaries or stored for later use.
- You need to document all options in one place (the struct's fields).

Never mix both patterns in the same constructor. Pick one.

## Small Interfaces

- Use 1-2 method interfaces. `io.Reader` is the gold standard.
- Never define large interfaces upfront.
- Extract an interface only when you have 2+ implementations or need to mock for testing.
- Define interfaces where consumed, not where implemented.

```go
// Good -- small, composable
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Compose when needed
type ReadCloser interface {
    Reader
    Closer
}
```

```go
// Bad -- interface pollution
type ClientInterface interface {
    Get(ctx context.Context, url string) (*Response, error)
    Post(ctx context.Context, url string, body io.Reader) (*Response, error)
    Put(ctx context.Context, url string, body io.Reader) (*Response, error)
    Delete(ctx context.Context, url string) (*Response, error)
    Patch(ctx context.Context, url string, body io.Reader) (*Response, error)
    Head(ctx context.Context, url string) (*Response, error)
    Options(ctx context.Context, url string) (*Response, error)
    Close() error
}

// Good -- define where consumed
type Fetcher interface {
    Get(ctx context.Context, url string) (*Response, error)
}
```

## Constructor Functions

### Naming

| Name | When |
|------|------|
| `New` | Primary constructor. `transport.New()` when the package name provides context. |
| `NewX` | Named constructor. `NewClient()`, `NewTransport()`. |
| `Open` | Resources that need closing. `Open(path)` implies a `Close()`. |
| `Parse` | Deserialization. `ParseToken(s)`, `ParseConfig(data)`. |
| `From` | Conversion. `FromJSON(data)`, `FromProto(msg)`. |

### Validation in Constructors

Validate everything. Return `(*T, error)` when creation can fail.

```go
func NewClient(baseURL string, opts ...Option) (*Client, error) {
    if baseURL == "" {
        return nil, errors.New("base URL is required")
    }

    parsedURL, err := url.Parse(baseURL)
    if err != nil {
        return nil, fmt.Errorf("parse base URL: %w", err)
    }

    if parsedURL.Scheme != "https" {
        return nil, fmt.Errorf("base URL must use HTTPS, got %s", parsedURL.Scheme)
    }

    // ... apply options, build client
    return &Client{baseURL: parsedURL}, nil
}
```

## Method Receivers

### Pointer vs Value Receiver

**Use a pointer receiver when:**
- The method mutates the receiver.
- The struct contains fields that cannot be copied safely (`sync.Mutex`, `sync.Once`, `sync.WaitGroup`).
- The struct is large enough that copying is wasteful.
- Any field is a pointer to mutable state and the method needs to observe writes.
- Concurrent methods must see modifications.
- **When in doubt, use a pointer receiver.**

**Use a value receiver only when:**
- The type is a small, naturally-value type (primitive-like: `Point`, `Duration`, enum int).
- The type is a slice that is not resliced or reallocated.
- The type is a map, function, or channel.
- The type is a built-in (int, string).
- The method does not mutate the receiver.

Be consistent per type: **if any method uses a pointer receiver, all methods must use a pointer receiver.**

```go
// Good -- consistent pointer receivers
func (c *Client) Do(ctx context.Context, request *Request) (*Response, error) { ... }
func (c *Client) Close() error { ... }
func (c *Client) BaseURL() string { return c.baseURL }

// Bad -- mixed receivers
func (c Client) BaseURL() string { ... }   // value
func (c *Client) Do(...) (*Response, error) { ... }  // pointer
```

### Receiver Naming

Short, one or two letters derived from the type name. Never `this` or `self`.

```go
func (c *Client) Do(...) { ... }       // c for Client
func (s *Server) Start(...) { ... }    // s for Server
func (t *Transport) RoundTrip(...) { ... }  // t for Transport
```

## Middleware Pattern

Middleware is a function that wraps a handler:

```go
type Middleware func(http.Handler) http.Handler

func WithLogging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
            start := time.Now()
            next.ServeHTTP(writer, request)
            logger.Info("request",
                "method", request.Method,
                "path", request.URL.Path,
                "duration", time.Since(start),
            )
        })
    }
}

func WithAuth(validator TokenValidator) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
            token := request.Header.Get("Authorization")
            if err := validator.Validate(request.Context(), token); err != nil {
                http.Error(writer, "unauthorized", http.StatusUnauthorized)
                return
            }
            next.ServeHTTP(writer, request)
        })
    }
}
```

Compose by chaining:

```go
handler := WithLogging(logger)(WithAuth(validator)(router))
```

Or with a helper:

```go
func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

handler := Chain(router, WithLogging(logger), WithAuth(validator))
```

## Pass Values, Not Pointers (When Reading Only)

Don't pass pointers just to save bytes. If a function only reads `*x`, pass `x` directly. This rule applies especially to `*string`, `*io.Reader`, `*time.Time`, and other small values.

**Exception:** large structs, protobuf messages (always pointers — they implement `proto.Message`), and structs that may grow fields over time.

```go
// Good -- reads url, no mutation
func Download(url string) ([]byte, error) { ... }

// Bad -- gains nothing from *string
func Download(url *string) ([]byte, error) { ... }
```

## Use `any`, Not `interface{}`

In new code, prefer `any` over `interface{}` (Go 1.18+). They are equivalent aliases; `any` is shorter and standard.

## Type Aliases: Migration Only

Use type definitions (`type T1 T2`) to create new types. Use type aliases (`type T1 = T2`) **only** to aid migrating packages to new import paths. Do not alias when no migration is in progress.

```go
// Good -- new distinct type
type UserID string

// Good -- migration aid; old.Foo and new.Foo are interchangeable during transition
type Foo = new.Foo

// Bad -- alias where a definition is appropriate
type UserID = string
```

## Generics

- Use generics when they serve a clear business requirement.
- Do not use generics when only one type is ever instantiated in practice — a non-generic version is clearer.
- Do not use generics to invent DSLs or error-handling frameworks.
- If types share a useful interface, model with the interface first; reach for generics only when the interface approach is genuinely insufficient.
- Prefer a generic over an `any` parameter with extensive type switching.
- Document exported generic APIs thoroughly, including a runnable example.

## Channel Direction in Signatures

Always declare channel direction in function parameters where possible. This conveys ownership and prevents mis-use.

```go
// Good
func produce(out chan<- Event) { ... }   // send-only
func consume(in <-chan Event) { ... }    // receive-only

// Bad
func produce(out chan Event) { ... }     // could receive or send
```

## Interfaces: Design Principles

- **Don't create interfaces until a real need exists.** Wait for a second implementation or a testing need.
- **Don't define "interface for future mocking"** — consumer-defined interfaces at the test site do this better.
- **The consumer defines the interface**, not the producer (except when the interface is the product itself, like `http.Handler`).
- **Interfaces should be small** — prefer 1-2 methods. Compose larger interfaces from smaller ones.
- **Functions take interfaces; return concrete types.**
- **Don't export a test-double interface** from production code solely to enable testing — keep the interface at the test boundary.
- **Don't wrap RPC clients in interfaces** just for abstraction/testing. Use real transports (`httptest.NewServer`) or the service's own test helpers.

## Pointers to Interfaces

**Never pass a pointer to an interface.** An interface value already contains an internal pointer (to the concrete type). Wrapping it again in `*SomeInterface` produces a pointer-to-pointer that callers must dereference to use, for no benefit.

```go
// Good
func Do(r io.Reader) error { ... }

// Bad
func Do(r *io.Reader) error { ... }
```

Pass the interface by value. The only time to store a pointer to an interface is when you need to swap the concrete implementation through that pointer — rare, and almost always better expressed as a struct field holding the interface directly.

## Method Sets: Value vs Pointer Receivers

Method set rules determine which interfaces a value satisfies:

| Receiver type | Callable on | Interface satisfied by |
|--------------|-------------|-----------------------|
| `func (t T) M()` | `T` values and `*T` pointers | `T` and `*T` |
| `func (t *T) M()` | `*T` pointers and **addressable** `T` values | `*T` only |

Practical implications:
- A pointer-receiver method is **not** in the method set of a non-addressable `T` (e.g., a map value, a literal). So a `T` retrieved from `map[K]T` cannot call its own pointer methods directly — assign to a variable first.
- An interface assignment uses the method set. If `Storer` requires `Put()` and `Put` has a pointer receiver on `*Store`, then `var s Storer = Store{}` fails; `var s Storer = &Store{}` compiles.
- Keep receiver types consistent per type (all pointer or all value) so interface satisfaction is predictable.

## No Recursion in Library Code

All execution must be provably bounded. Use iteration with explicit limits instead of recursion. Recursion hides resource consumption in the call stack and makes worst-case behavior unpredictable.

## Options: Rely on Documented Defaults

Let sensible defaults apply at call sites. Specify options only when the caller needs a non-default value. (Google rule — supersedes earlier "always explicit" guidance.)

Reasons to rely on defaults:
- Reduces noise at call sites.
- Matches Go's zero-value conventions.
- Lets library authors evolve defaults as best practice changes.

```go
// Good -- default timeout/retries/transport are documented and appropriate
client, err := NewClient(baseURL)

// Good -- only override the option that matters
client, err := NewClient(baseURL, WithTimeout(5*time.Second))

// Overkill -- every option spelled out, hiding which one the caller actually cares about
client, err := NewClient(baseURL,
    WithTimeout(30 * time.Second),      // this IS the default
    WithRetries(3),                     // this IS the default
    WithTransport(http.DefaultTransport), // this IS the default
)
```

**Corollary for library authors:** defaults must be documented on the constructor. When a default changes, release notes must call it out.

## Option Structs vs. Variadic Options

| Use | When |
|-----|------|
| **Option struct** (last arg is a `*Options` or `Options` value) | Many options needed; options shared between multiple constructors; options serializable from config; option set is relatively closed. |
| **Variadic functional options** (`...Option`) | Public API where API evolution matters; most callers specify few options; options need to be composable or conditional; you need to restrict who can define options (unexported option-func parameter). |

**Never mix both patterns** on the same constructor.

Contexts are **never** included in option structs — they go as the first positional parameter.

## Option Naming: Take Parameters, Don't Encode State

Options should accept parameters rather than encoding state in their name:

```go
// Good
func FailFast(enable bool) Option { ... }
client, _ := NewClient(url, FailFast(true))

// Bad -- two functions where one with a bool suffices
func EnableFailFast() Option { ... }
func DisableFailFast() Option { ... }
```

Options are applied in order; last wins on conflict.

## Limits on Everything

Every operation that touches I/O or external systems must have bounds. Every queue, buffer, and retry loop must have a maximum size or count. Unbounded anything is a latent availability risk.

| What | How |
|------|-----|
| HTTP requests | `context.WithTimeout` or `http.Client.Timeout` |
| Retries | Max count + exponential backoff with jitter |
| Response bodies | `io.LimitReader` |
| Channel buffers | Explicit capacity: `make(chan Event, 100)` |
| Connection pools | `MaxIdleConns`, `MaxIdleConnsPerHost` |
| Request payloads | Validate `Content-Length` before reading |

```go
// Good -- bounded read
body, err := io.ReadAll(io.LimitReader(response.Body, maxResponseSizeBytes))

// Bad -- unbounded read, OOM risk
body, err := io.ReadAll(response.Body)
```

## Zero Values

Design types so that the zero value is useful or obviously invalid:

```go
// Good -- zero value is a usable mutex
var mu sync.Mutex

// Good -- zero value is a usable buffer
var buffer bytes.Buffer

// Good -- zero value is obviously invalid, forces use of constructor
type Client struct {
    baseURL   *url.URL    // nil zero value -- must use NewClient
    transport http.RoundTripper
}
```

## Type Assertions: Always Comma-Ok

Type assertions panic on failure when used in the single-value form. **Always use the two-value "comma-ok" form** for type assertions on values whose concrete type you don't fully control.

```go
// Good
if cfg, ok := v.(*Config); ok {
    use(cfg)
}

// Bad -- panics on any other concrete type
cfg := v.(*Config)
use(cfg)
```

The single-value form is permitted **only** when a panic is the intended behavior for a violated invariant — e.g., asserting a sealed-interface tag that must hold by construction. Document the reasoning at the call site.

For error inspection, use `errors.As`, never a direct type assertion (see `03-error-handling.md`).

## Type Switches

Use type switches for dispatching on interface types. Always include a `default` case:

```go
func describe(v any) string {
    switch val := v.(type) {
    case string:
        return fmt.Sprintf("string of length %d", len(val))
    case int:
        return fmt.Sprintf("integer %d", val)
    case error:
        return fmt.Sprintf("error: %s", val.Error())
    default:
        return fmt.Sprintf("unknown type %T", val)
    }
}
```

Prefer type switches over chains of `if val, ok := v.(Type); ok { ... }` when checking multiple types.

## Interface Embedding

Compose interfaces from smaller ones. This is the Go equivalent of interface extension:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composed from smaller interfaces
type ReadWriter interface {
    Reader
    Writer
}
```

Only embed interfaces that the composite needs in full. Don't embed a 5-method interface when you only need 2 of its methods -- define the 2-method interface instead.

## Generality

If a type exists only to implement an interface and has no exported methods beyond that interface, there is no need to export the type. Export only the interface; the constructor returns the interface value.

```go
// Good -- unexported type, exported interface
type handler struct { ... }

func NewHandler() http.Handler {
    return &handler{ ... }
}
```

This pattern hides the concrete type, ensuring callers depend only on the interface contract.

## new() vs make()

| Function | Purpose | Returns |
|----------|---------|---------|
| `new(T)` | Allocates zeroed memory for type T | `*T` (pointer to zero value) |
| `make(T, ...)` | Initializes slices, maps, and channels | `T` (initialized value, not a pointer) |

```go
// new -- rarely needed, prefer composite literals
p := new(Point)          // *Point with zero values
p := &Point{}            // equivalent, preferred

// make -- required for slices, maps, channels
s := make([]string, 0, 10)   // slice with len=0, cap=10
m := make(map[string]int)    // initialized map
ch := make(chan Event, 100)   // buffered channel
```

Prefer composite literals (`&Point{X: 1, Y: 2}`) over `new()` for structs.

## Interface Compliance Check

Verify interface implementation at compile time with a blank assignment. Place at the top of the file, after imports.

```go
var _ http.RoundTripper = (*Transport)(nil)
var _ io.ReadCloser = (*responseBody)(nil)
```

## Avoid Embedding Types in Public Structs

Embedded types leak implementation details into the public API. Every method and field of the embedded type becomes part of the embedding type's surface. This creates tight coupling, inhibits evolution, and obscures documentation.

```go
// Bad -- embeds AbstractList, exposing all its methods
type ConcreteList struct {
    AbstractList
}

// Bad -- embeds sync.Mutex, exposing Lock/Unlock as public API
type Cache struct {
    sync.Mutex
    data map[string]string
}

// Good -- mutex is a private implementation detail
type Cache struct {
    mu   sync.Mutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    v, ok := c.data[key]
    return v, ok
}
```

### Embedding Placement

When embedding is used (in unexported types, or when the full embedded interface is intentionally the public API), place embedded types **at the top of the struct's field list**, separated from regular fields by a blank line:

```go
// Good
type CountingReader struct {
    io.Reader

    count int64
}

// Bad -- embedded type mixed with regular fields
type CountingReader struct {
    count int64
    io.Reader
}
```

This signals at a glance which fields are embedded (contribute to the method set) versus regular data fields.

### Never Embed Mutexes (Even in Unexported Types)

`sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, and `sync.Once` must **never** be embedded — not in exported types, not in unexported types. Embedding leaks `Lock()` / `Unlock()` / `Wait()` to any caller with access to the struct, which is always the wrong API.

```go
// Bad -- even in an unexported type, Lock/Unlock are exported on the outer type
type cache struct {
    sync.Mutex
    data map[string]string
}

// Good -- named field; lock methods stay internal
type cache struct {
    mu   sync.Mutex
    data map[string]string
}
```

Embedding is acceptable in unexported types or when the full embedded interface is the desired API:

```go
// Acceptable -- unexported, no public API impact
type baseClient struct {
    http.Client
}

// Acceptable -- embedding io.Reader IS the intended API
type CountingReader struct {
    io.Reader
    count int64
}
```

Write forwarding methods explicitly when you need selective delegation. It's more code but it's visible, documented, and won't break when the embedded type adds methods.

## Start Enums at One

Start enum values at `iota + 1` so that the zero value indicates an unset or invalid state. Since Go variables default to zero, an uninitialized enum variable is distinguishable from a deliberately assigned value.

```go
// Good -- zero value is invalid, catches uninitialized usage
type Status int

const (
    StatusActive   Status = iota + 1 // 1
    StatusInactive                    // 2
    StatusPending                     // 3
)

// Bad -- zero value is a valid state, hides bugs
type Status int

const (
    StatusActive   Status = iota // 0 -- same as zero value
    StatusInactive               // 1
    StatusPending                // 2
)
```

**Exception**: when the zero value is the desired default behavior:

```go
type LogOutput int

const (
    LogToStdout LogOutput = iota // 0 -- sensible default
    LogToFile                    // 1
    LogToSyslog                  // 2
)
```

## Copy Slices and Maps at Boundaries

Slices and maps hold references to underlying data. When receiving or returning them across API boundaries, make defensive copies to prevent callers from mutating internal state or vice versa.

### Receiving

```go
// Good -- defensive copy, caller's slice cannot affect internal state
func NewProcessor(items []string) *Processor {
    return &Processor{
        items: append([]string(nil), items...),
    }
}

// Bad -- internal state aliases caller's slice
func NewProcessor(items []string) *Processor {
    return &Processor{items: items}
}
```

### Returning

```go
// Good -- returns a copy, caller cannot corrupt internal state
func (p *Processor) Items() []string {
    return append([]string(nil), p.items...)
}

// Good -- maps require explicit copy
func (c *Config) Headers() map[string]string {
    result := make(map[string]string, len(c.headers))
    for k, v := range c.headers {
        result[k] = v
    }
    return result
}

// Bad -- caller can mutate the internal map
func (c *Config) Headers() map[string]string {
    return c.headers
}
```

Copies are unnecessary for truly immutable data or performance-critical hot paths where the overhead is measured and unacceptable. In those cases, document the aliasing behavior.

## Avoid Mutable Globals

Mutable global variables make testing difficult, hide dependencies, and create implicit coupling. Use dependency injection instead.

```go
// Bad -- mutable global, difficult to test
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
}

func GetUser(id string) (*User, error) {
    return queryUser(db, id) // implicit dependency
}

// Good -- dependency injection, easy to test
type UserStore struct {
    db *sql.DB
}

func NewUserStore(db *sql.DB) *UserStore {
    return &UserStore{db: db}
}

func (s *UserStore) GetUser(ctx context.Context, id string) (*User, error) {
    return queryUser(ctx, s.db, id) // explicit dependency
}
```

This applies to function pointers too. Don't replace a global `var timeNow = time.Now` for testing -- pass a clock interface:

```go
// Bad
var timeNow = time.Now

// Good
type Clock interface {
    Now() time.Time
}

type realClock struct{}

func (realClock) Now() time.Time { return time.Now() }
```

## Exit in Main Only

Call `os.Exit` or `log.Fatal` only in `main()`. All other functions must return errors. This preserves testability, ensures `defer` statements execute, and keeps control flow visible.

```go
// Good -- main handles the exit, business logic is testable
func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}

func run() error {
    config, err := loadConfig()
    if err != nil {
        return fmt.Errorf("load config: %w", err)
    }

    server, err := newServer(config)
    if err != nil {
        return fmt.Errorf("create server: %w", err)
    }
    defer server.Close()

    return server.ListenAndServe()
}

// Bad -- log.Fatal deep in the call stack
func loadConfig() *Config {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        log.Fatalf("read config: %v", err) // skips defers, untestable
    }
    // ...
}
```

The `run()` pattern keeps `main()` short and makes the entire program testable. Use a single exit point in `main` whenever possible.
