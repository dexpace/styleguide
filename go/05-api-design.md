# 05 — API Design

An API is a contract the caller reads once and depends on forever. Shape it so the narrowest interface flows in and a concrete type flows out, so every optional knob has a documented default, and so every operation that touches I/O, a queue, or a call stack carries an explicit bound. The zero value either works or is obviously invalid; constructors validate; receivers stay consistent; nothing unbounded ships. Design the surface for the caller, not the implementer.

## What good looks like

```go
// Client talks to a remote API. Construct it with NewClient; the zero value is invalid.
type Client struct {
    baseURL   *url.URL // nil zero value -- must use NewClient
    transport http.RoundTripper
    timeout   time.Duration
    retries   int
}

type clientConfig struct {
    timeout   time.Duration
    transport http.RoundTripper
    retries   int
}

// Option configures a Client. Defaults are documented on NewClient.
type Option func(*clientConfig)

func WithTimeout(d time.Duration) Option {
    return func(c *clientConfig) { c.timeout = d }
}

func WithRetries(count int) Option {
    return func(c *clientConfig) { c.retries = count }
}

// NewClient accepts an interface (RoundTripper via options), returns a concrete *Client,
// and validates everything. Defaults: 30s timeout, http.DefaultTransport, 3 retries.
func NewClient(baseURL string, opts ...Option) (*Client, error) {
    if baseURL == "" {
        return nil, errors.New("base URL is required")
    }
    parsed, err := url.Parse(baseURL)
    if err != nil {
        return nil, fmt.Errorf("parse base URL: %w", err)
    }
    if parsed.Scheme != "https" {
        return nil, fmt.Errorf("base URL must use HTTPS, got %s", parsed.Scheme)
    }

    config := &clientConfig{
        timeout:   30 * time.Second,
        transport: http.DefaultTransport,
        retries:   3,
    }
    for _, opt := range opts {
        opt(config)
    }
    if config.timeout <= 0 {
        return nil, errors.New("timeout must be positive")
    }

    return &Client{
        baseURL:   parsed,
        transport: config.transport,
        timeout:   config.timeout,
        retries:   config.retries,
    }, nil
}

// Fetch reads a bounded response. The body read is capped; nothing here is unbounded.
func (c *Client) Fetch(ctx context.Context, path string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, c.timeout)
    defer cancel()
    // ... issue request with c.transport ...
    response, err := c.do(ctx, path)
    if err != nil {
        return nil, fmt.Errorf("fetch %s: %w", path, err)
    }
    defer response.Body.Close()
    return io.ReadAll(io.LimitReader(response.Body, maxResponseSizeBytes))
}
```

This client accepts the `http.RoundTripper` interface and returns the concrete `*Client` (5.1); configures via functional options with documented defaults the caller need not restate (5.2, 5.16); validates everything in the constructor and returns `(*T, error)` (5.4); uses consistent pointer receivers (5.5); keeps the zero value obviously invalid so the constructor is mandatory (5.20); and bounds the response read with `io.LimitReader` (5.19).

## Rules

### 5.1 — Accept interfaces, return concrete types.

**Reasoning, step by step:**

1. Accept the narrowest interface that satisfies the function's needs — the caller can pass any implementation, including a test double.
2. Return concrete types. The caller decides when to abstract; returning an interface forecloses that choice and hides the concrete API.

```go
// Good -- accepts interface, returns concrete type
func NewClient(transport http.RoundTripper) *Client { ... }

// Bad -- returns interface
func NewClient(transport http.RoundTripper) ClientInterface { ... }

// Bad -- accepts concrete type
func NewClient(transport *http.Transport) *Client { ... }
```

**Enforcement:** review.

### 5.2 — Use functional options for configurable constructors.

**Reasoning, step by step:**

1. A constructor with many optional parameters and sensible defaults is best configured with functional options: each `With*` returns a closure that mutates an unexported config, defaults are set before options apply, and validation runs after.

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

2. The call site reads as named overrides over defaults:

```go
client, err := NewClient("https://api.example.com",
    WithTimeout(10 * time.Second),
    WithRetries(5),
)
```

3. For public APIs where you want to **prevent third-party callers from defining their own options**, implement options as a sealed interface with an unexported `apply` method rather than a function type. This restricts option construction to functions in your package.

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

   Use the interface variant when: options carry invariants the library must enforce — external callers should not be able to construct arbitrary options that violate them; or you want to add or remove option types as unannounced internal changes (no closure-type dependency in caller code). Use the simpler `type Option func(*config)` closure form when option extensibility is a non-goal — most internal or single-consumer APIs.

4. Choose functional options versus a config struct by context:

| Use | When |
|-----|------|
| Functional options | Public API with many optional parameters and sensible defaults |
| Config struct | Internal code, or when all fields are required, or when config must be serializable |

   Functional options excel when: you need backwards-compatible API evolution (add new `With*` functions without breaking callers); most parameters have sensible defaults and callers only override a few; the constructor performs validation after applying all options. Config structs excel when: all or most fields are required (no defaults to manage); the config must be deserialized from JSON/YAML/env; the config is passed across package boundaries or stored for later use; you need to document all options in one place (the struct's fields). Never mix both patterns in the same constructor. Pick one.

**Enforcement:** review.

### 5.3 — Keep interfaces small.

**Reasoning, step by step:**

1. Use 1-2 method interfaces. `io.Reader` is the gold standard. Never define large interfaces upfront.
2. Extract an interface only when you have 2+ implementations or need to mock for testing. Define interfaces where consumed, not where implemented.

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

3. Large interfaces are interface pollution; the consumer defines the narrow slice it actually needs:

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

**Enforcement:** review.

### 5.4 — Name and validate constructors.

**Reasoning, step by step:**

1. Name the constructor for the shape of creation it performs:

| Name | When |
|------|------|
| `New` | Primary constructor. `transport.New()` when the package name provides context. |
| `NewX` | Named constructor. `NewClient()`, `NewTransport()`. |
| `Open` | Resources that need closing. `Open(path)` implies a `Close()`. |
| `Parse` | Deserialization. `ParseToken(s)`, `ParseConfig(data)`. |
| `From` | Conversion. `FromJSON(data)`, `FromProto(msg)`. |

2. Validate everything. Return `(*T, error)` when creation can fail.

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

**Enforcement:** review.

### 5.5 — Keep method receivers consistent.

**Reasoning, step by step:**

1. Use a pointer receiver when: the method mutates the receiver; the struct contains fields that cannot be copied safely (`sync.Mutex`, `sync.Once`, `sync.WaitGroup`); the struct is large enough that copying is wasteful; any field is a pointer to mutable state and the method needs to observe writes; concurrent methods must see modifications. **When in doubt, use a pointer receiver.**
2. Use a value receiver only when: the type is a small, naturally-value type (primitive-like: `Point`, `Duration`, enum int); the type is a slice that is not resliced or reallocated; the type is a map, function, or channel; the type is a built-in (int, string); the method does not mutate the receiver.
3. Be consistent per type: **if any method uses a pointer receiver, all methods must use a pointer receiver.**

```go
// Good -- consistent pointer receivers
func (c *Client) Do(ctx context.Context, request *Request) (*Response, error) { ... }
func (c *Client) Close() error { ... }
func (c *Client) BaseURL() string { return c.baseURL }

// Bad -- mixed receivers
func (c Client) BaseURL() string { ... }   // value
func (c *Client) Do(...) (*Response, error) { ... }  // pointer
```

4. Name receivers short, one or two letters derived from the type name. Never `this` or `self`.

```go
func (c *Client) Do(...) { ... }       // c for Client
func (s *Server) Start(...) { ... }    // s for Server
func (t *Transport) RoundTrip(...) { ... }  // t for Transport
```

**Enforcement:** review.

### 5.6 — Express middleware as a handler-wrapping function.

**Reasoning, step by step:**

1. Middleware is a function that wraps a handler: it takes the next `http.Handler` and returns a new one that runs before/after delegating.

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

2. Compose by chaining directly:

```go
handler := WithLogging(logger)(WithAuth(validator)(router))
```

3. Or with a helper that applies middlewares in reverse so the first listed runs outermost:

```go
func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

handler := Chain(router, WithLogging(logger), WithAuth(validator))
```

**Enforcement:** review.

### 5.7 — Pass values, not pointers, when only reading.

**Reasoning, step by step:**

1. Don't pass pointers just to save bytes. If a function only reads `*x`, pass `x` directly. This rule applies especially to `*string`, `*io.Reader`, `*time.Time`, and other small values.
2. The exception: large structs, protobuf messages (always pointers — they implement `proto.Message`), and structs that may grow fields over time.

```go
// Good -- reads url, no mutation
func Download(url string) ([]byte, error) { ... }

// Bad -- gains nothing from *string
func Download(url *string) ([]byte, error) { ... }
```

**Enforcement:** review.

### 5.8 — Use `any`, not `interface{}`.

**Reasoning, step by step:**

1. In new code, prefer `any` over `interface{}` (Go 1.18+). They are equivalent aliases; `any` is shorter and standard.

**Enforcement:** `golangci-lint`.

### 5.9 — Reserve type aliases for migration only.

**Reasoning, step by step:**

1. Use type definitions (`type T1 T2`) to create new types. Use type aliases (`type T1 = T2`) **only** to aid migrating packages to new import paths. Do not alias when no migration is in progress.

```go
// Good -- new distinct type
type UserID string

// Good -- migration aid; old.Foo and new.Foo are interchangeable during transition
type Foo = new.Foo

// Bad -- alias where a definition is appropriate
type UserID = string
```

**Enforcement:** review.

### 5.10 — Reach for generics only on a clear need.

**Reasoning, step by step:**

1. Use generics when they serve a clear business requirement. Do not use generics when only one type is ever instantiated in practice — a non-generic version is clearer.
2. Do not use generics to invent DSLs or error-handling frameworks. If types share a useful interface, model with the interface first; reach for generics only when the interface approach is genuinely insufficient.
3. Prefer a generic over an `any` parameter with extensive type switching. Document exported generic APIs thoroughly, including a runnable example.

**Enforcement:** review.

### 5.11 — Declare channel direction in signatures.

**Reasoning, step by step:**

1. Declare channel direction (`chan<-`, `<-chan`) in function parameters wherever possible: it conveys ownership and prevents mis-use. See `04-concurrency.md` for the full treatment.

**Enforcement:** `go vet`.

### 5.12 — Apply consumer-driven interface design principles.

**Reasoning, step by step:**

1. **Don't create interfaces until a real need exists.** Wait for a second implementation or a testing need. **Don't define "interface for future mocking"** — consumer-defined interfaces at the test site do this better.
2. **The consumer defines the interface**, not the producer (except when the interface is the product itself, like `http.Handler`). **Interfaces should be small** — prefer 1-2 methods. Compose larger interfaces from smaller ones. **Functions take interfaces; return concrete types.**
3. **Don't export a test-double interface** from production code solely to enable testing — keep the interface at the test boundary. **Don't wrap RPC clients in interfaces** just for abstraction/testing. Use real transports (`httptest.NewServer`) or the service's own test helpers.

**Enforcement:** review.

### 5.13 — Never pass a pointer to an interface.

**Reasoning, step by step:**

1. **Never pass a pointer to an interface.** An interface value already contains an internal pointer (to the concrete type). Wrapping it again in `*SomeInterface` produces a pointer-to-pointer that callers must dereference to use, for no benefit.

```go
// Good
func Do(r io.Reader) error { ... }

// Bad
func Do(r *io.Reader) error { ... }
```

2. Pass the interface by value. The only time to store a pointer to an interface is when you need to swap the concrete implementation through that pointer — rare, and almost always better expressed as a struct field holding the interface directly.

**Enforcement:** `golangci-lint`.

### 5.14 — Know the method-set rules for value vs pointer receivers.

**Reasoning, step by step:**

1. Method set rules determine which interfaces a value satisfies:

| Receiver type | Callable on | Interface satisfied by |
|--------------|-------------|-----------------------|
| `func (t T) M()` | `T` values and `*T` pointers | `T` and `*T` |
| `func (t *T) M()` | `*T` pointers and **addressable** `T` values | `*T` only |

2. A pointer-receiver method is **not** in the method set of a non-addressable `T` (e.g., a map value, a literal). So a `T` retrieved from `map[K]T` cannot call its own pointer methods directly — assign to a variable first.
3. An interface assignment uses the method set. If `Storer` requires `Put()` and `Put` has a pointer receiver on `*Store`, then `var s Storer = Store{}` fails; `var s Storer = &Store{}` compiles. Keep receiver types consistent per type (all pointer or all value) so interface satisfaction is predictable.

**Enforcement:** review.

### 5.15 — Bound recursion in library code.

**Reasoning, step by step:**

1. All execution must be provably bounded. Prefer iteration with explicit limits; it keeps resource consumption visible rather than hidden in the call stack.
2. Unbounded recursion is banned in library code — a hostile or deeply nested input must never be able to blow the stack.
3. Recursion is permitted only when its depth is provably bounded by an explicit, asserted limit, so worst-case stack usage is known. A recursive parser, for example, must check an incrementing depth counter against a hard cap and fail once it is exceeded (and recover at the public boundary — see `03-error-handling.md`).

**Enforcement:** review.

### 5.16 — Rely on documented defaults for options.

**Reasoning, step by step:**

1. Let sensible defaults apply at call sites. Specify options only when the caller needs a non-default value. (Google rule — supersedes earlier "always explicit" guidance.)
2. Relying on defaults reduces noise at call sites, matches Go's zero-value conventions, and lets library authors evolve defaults as best practice changes.

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

3. **Corollary for library authors:** defaults must be documented on the constructor. When a default changes, release notes must call it out.

**Enforcement:** review.

### 5.17 — Choose option structs vs variadic options deliberately.

**Reasoning, step by step:**

1. Pick the option-passing style by the shape of the option set:

| Use | When |
|-----|------|
| **Option struct** (last arg is a `*Options` or `Options` value) | Many options needed; options shared between multiple constructors; options serializable from config; option set is relatively closed. |
| **Variadic functional options** (`...Option`) | Public API where API evolution matters; most callers specify few options; options need to be composable or conditional; you need to restrict who can define options (unexported option-func parameter). |

2. **Never mix both patterns** on the same constructor.
3. Contexts are **never** included in option structs — they go as the first positional parameter.

**Enforcement:** review.

### 5.18 — Make options take parameters, not encode state in their name.

**Reasoning, step by step:**

1. Options should accept parameters rather than encoding state in their name:

```go
// Good
func FailFast(enable bool) Option { ... }
client, _ := NewClient(url, FailFast(true))

// Bad -- two functions where one with a bool suffices
func EnableFailFast() Option { ... }
func DisableFailFast() Option { ... }
```

2. Options are applied in order; last wins on conflict.

**Enforcement:** review.

### 5.19 — Put limits on everything.

**Reasoning, step by step:**

1. Every operation that touches I/O or external systems must have bounds. Every queue, buffer, and retry loop must have a maximum size or count. Unbounded anything is a latent availability risk.

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

**Enforcement:** review.

### 5.20 — Design types so the zero value is useful or obviously invalid.

**Reasoning, step by step:**

1. Design types so that the zero value is useful or obviously invalid — never a silently-wrong half-state.

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

**Enforcement:** review.

### 5.21 — Always use the comma-ok form for type assertions.

**Reasoning, step by step:**

1. Type assertions panic on failure when used in the single-value form. **Always use the two-value "comma-ok" form** for type assertions on values whose concrete type you don't fully control.

```go
// Good
if cfg, ok := v.(*Config); ok {
    use(cfg)
}

// Bad -- panics on any other concrete type
cfg := v.(*Config)
use(cfg)
```

2. The single-value form is permitted **only** when a panic is the intended behavior for a violated invariant — e.g., asserting a sealed-interface tag that must hold by construction. Document the reasoning at the call site.
3. For error inspection, use `errors.As`, never a direct type assertion (see `03-error-handling.md`).

**Enforcement:** `go vet`.

### 5.22 — Use type switches with a default case for interface dispatch.

**Reasoning, step by step:**

1. Use type switches for dispatching on interface types. Always include a `default` case:

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

2. Prefer type switches over chains of `if val, ok := v.(Type); ok { ... }` when checking multiple types.

**Enforcement:** review.

### 5.23 — Compose interfaces by embedding smaller ones.

**Reasoning, step by step:**

1. Compose interfaces from smaller ones. This is the Go equivalent of interface extension:

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

2. Only embed interfaces that the composite needs in full. Don't embed a 5-method interface when you only need 2 of its methods -- define the 2-method interface instead.

**Enforcement:** review.

### 5.24 — Export the interface, not a type that only implements it.

**Reasoning, step by step:**

1. If a type exists only to implement an interface and has no exported methods beyond that interface, there is no need to export the type. Export only the interface; the constructor returns the interface value.

```go
// Good -- unexported type, exported interface
type handler struct { ... }

func NewHandler() http.Handler {
    return &handler{ ... }
}
```

2. This pattern hides the concrete type, ensuring callers depend only on the interface contract.

**Enforcement:** review.

### 5.25 — Use `new()` and `make()` for their distinct purposes.

**Reasoning, step by step:**

1. `new` and `make` are not interchangeable:

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

2. Prefer composite literals (`&Point{X: 1, Y: 2}`) over `new()` for structs.

**Enforcement:** review.

### 5.26 — Assert interface compliance at compile time.

**Reasoning, step by step:**

1. Verify interface implementation at compile time with a blank assignment. Place at the top of the file, after imports.

```go
var _ http.RoundTripper = (*Transport)(nil)
var _ io.ReadCloser = (*responseBody)(nil)
```

**Enforcement:** `golangci-lint`.

### 5.27 — Avoid embedding types in public structs.

**Reasoning, step by step:**

1. Embedded types leak implementation details into the public API. Every method and field of the embedded type becomes part of the embedding type's surface. This creates tight coupling, inhibits evolution, and obscures documentation.

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

2. When embedding is used (in unexported types, or when the full embedded interface is intentionally the public API), place embedded types **at the top of the struct's field list**, separated from regular fields by a blank line:

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

3. `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, and `sync.Once` must **never** be embedded — not in exported types, not in unexported types. Embedding leaks `Lock()` / `Unlock()` / `Wait()` to any caller with access to the struct, which is always the wrong API.

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

4. Embedding is acceptable in unexported types or when the full embedded interface is the desired API:

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

**Enforcement:** review.

### 5.28 — Start enums at one.

**Reasoning, step by step:**

1. Start enum values at `iota + 1` so that the zero value indicates an unset or invalid state. Since Go variables default to zero, an uninitialized enum variable is distinguishable from a deliberately assigned value.

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

2. The exception: when the zero value is the desired default behavior, start at `iota`.

```go
type LogOutput int

const (
    LogToStdout LogOutput = iota // 0 -- sensible default
    LogToFile                    // 1
    LogToSyslog                  // 2
)
```

**Enforcement:** review.

### 5.29 — Copy slices and maps at API boundaries.

**Reasoning, step by step:**

1. Slices and maps hold references to underlying data. When receiving or returning them across API boundaries, make defensive copies to prevent callers from mutating internal state or vice versa.
2. When receiving, copy so the caller's slice cannot affect internal state:

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

3. When returning, copy so the caller cannot corrupt internal state; maps require an explicit element-by-element copy:

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

4. Copies are unnecessary for truly immutable data or performance-critical hot paths where the overhead is measured and unacceptable. In those cases, document the aliasing behavior.

**Enforcement:** review.

### 5.30 — Avoid mutable globals; inject dependencies.

**Reasoning, step by step:**

1. Mutable global variables make testing difficult, hide dependencies, and create implicit coupling. Use dependency injection instead.

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

2. This applies to function pointers too. Don't replace a global `var timeNow = time.Now` for testing -- pass a clock interface:

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

**Enforcement:** review.

### 5.31 — Exit only in main.

**Reasoning, step by step:**

1. Call `os.Exit` or `log.Fatal` only in `main()`. All other functions must return errors. This preserves testability, ensures `defer` statements execute, and keeps control flow visible.

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

2. The `run()` pattern keeps `main()` short and makes the entire program testable. Use a single exit point in `main` whenever possible.

**Enforcement:** `golangci-lint`.

## Cross-references

- Channel direction, ownership, and bounded concurrency: [`04-concurrency.md`](./04-concurrency.md).
- `errors.As`, error wrapping, and recovery at public boundaries: [`03-error-handling.md`](./03-error-handling.md).
