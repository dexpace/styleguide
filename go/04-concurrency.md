# 04 - Concurrency

## Core Principle

**Don't communicate by sharing memory; share memory by communicating.** Channels are Go's primary synchronization mechanism.

## context.Context

- Always the **first parameter** of every function that does I/O or may be cancelled: `func F(ctx context.Context, ...) (...)`.
- Never store a context in a struct. Pass it through calls.
- Never pass `nil` context — use `context.TODO()` if you don't have one yet.
- Always respect `ctx.Done()` in loops.
- Derive child contexts with `context.WithTimeout` / `context.WithCancel` for scoped deadlines.

**Exceptions where context is not the first parameter:**
- HTTP handlers: obtain via `req.Context()`.
- Streaming RPC methods: obtain via the stream's `Context()`.
- Go 1.24+ tests: use `t.Context()`.
- Entrypoints (`main`, `init`): use `context.Background()`.

**Never** create a base context (`context.Background()`) in the middle of a call chain. If you need one, you have designed something wrong.

**Never** define custom context types or use interfaces other than `context.Context` in function signatures. (Google canonical rule — no exceptions.)

### Documenting Context Behavior

Standard context semantics — cancellation interrupts the function, values flow through — do **not** need to be documented. Document only:

- If the function returns an error other than `ctx.Err()` on cancellation.
- If there is a non-context mechanism to stop the function (e.g., a `Stop` method).
- If the function has special expectations (requires deadline, requires attached values, etc.). **Avoid designing APIs that make such demands.**

```go
// Good
func (c *Client) Fetch(ctx context.Context, url string) (*Response, error) { ... }
func (s *Store) Get(ctx context.Context, key string) ([]byte, error) { ... }

// Bad -- no context
func (c *Client) Fetch(url string) (*Response, error) { ... }
```

### Respecting cancellation in a loop

```go
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case item := <-items:
        if err := process(ctx, item); err != nil {
            return fmt.Errorf("process item: %w", err)
        }
    }
}
```

## Goroutine Lifecycle

Every goroutine must have a clear shutdown path. Never start a goroutine you cannot stop.

### The errgroup Pattern

Use `errgroup` for managing goroutine groups with error propagation:

```go
func (s *Server) Start(ctx context.Context) error {
    group, ctx := errgroup.WithContext(ctx)

    group.Go(func() error {
        return s.serve(ctx)
    })

    group.Go(func() error {
        return s.watchConfig(ctx)
    })

    group.Go(func() error {
        return s.collectMetrics(ctx)
    })

    return group.Wait()
}
```

### The Worker Pattern

For long-lived workers, combine context cancellation with a done channel:

```go
type Worker struct {
    done chan struct{}
}

func (w *Worker) Start(ctx context.Context) {
    go func() {
        defer close(w.done)
        for {
            select {
            case <-ctx.Done():
                return
            default:
                w.doWork(ctx)
            }
        }
    }()
}

func (w *Worker) Stop() {
    <-w.done
}
```

## sync.Mutex

Use `sync.Mutex` for protecting shared state. Channels are for communication; mutexes are for synchronization.

```go
type TokenCache struct {
    mu    sync.Mutex
    token string
    expiry time.Time
}

func (c *TokenCache) Get() string {
    c.mu.Lock()
    defer c.mu.Unlock()
    if time.Now().After(c.expiry) {
        return ""
    }
    return c.token
}

func (c *TokenCache) Set(token string, expiry time.Time) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.token = token
    c.expiry = expiry
}
```

### Mutex Rules

| Rule | Rationale |
|------|-----------|
| Always `defer mu.Unlock()` | Prevents deadlocks on early returns or panics |
| Keep the critical section small | Hold the lock only while accessing shared state |
| Never hold a mutex while doing I/O | I/O can block indefinitely, deadlocking other goroutines |
| Use `sync.RWMutex` when reads vastly outnumber writes | Read-heavy workloads benefit from concurrent read access |
| Put the mutex above the fields it protects | Convention: `mu` guards the fields that follow it |

## sync.Once

Use `sync.Once` for lazy initialization -- guarantees exactly one execution under concurrent access.

```go
type Client struct {
    initOnce sync.Once
    conn     *Connection
    initErr  error
}

func (c *Client) connection() (*Connection, error) {
    c.initOnce.Do(func() {
        c.conn, c.initErr = dial()
    })
    return c.conn, c.initErr
}
```

## sync.WaitGroup

Use `sync.WaitGroup` for fan-out/fan-in when you don't need error propagation. Prefer `errgroup` when you do.

```go
func processAll(ctx context.Context, items []Item) {
    var waitGroup sync.WaitGroup
    for _, item := range items {
        waitGroup.Add(1)
        go func() {
            defer waitGroup.Done()
            process(ctx, item)
        }()
    }
    waitGroup.Wait()
}
```

Always call `Add()` before starting the goroutine, not inside it.

## Bounded Concurrency

Never create unbounded goroutines. Use a semaphore channel:

```go
func processAll(ctx context.Context, items []Item, maxConcurrency int) error {
    semaphore := make(chan struct{}, maxConcurrency)
    group, ctx := errgroup.WithContext(ctx)

    for _, item := range items {
        semaphore <- struct{}{} // acquire
        group.Go(func() error {
            defer func() { <-semaphore }() // release
            return process(ctx, item)
        })
    }

    return group.Wait()
}
```

For more sophisticated limiting, use `golang.org/x/sync/semaphore.Weighted`.

## select with ctx.Done()

Every long-running loop must include a `ctx.Done()` case in its `select`.

```go
// Good
func (w *Watcher) watch(ctx context.Context) error {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if err := w.poll(ctx); err != nil {
                return fmt.Errorf("poll: %w", err)
            }
        }
    }
}

// Bad -- ignores cancellation
func (w *Watcher) watch() {
    for {
        time.Sleep(30 * time.Second)
        w.poll()
    }
}
```

## No time.Sleep in Production Code

`time.Sleep` is uninterruptible. Use `time.Ticker` or `time.After` with context cancellation.

```go
// Good
select {
case <-ctx.Done():
    return ctx.Err()
case <-time.After(5 * time.Second):
    // proceed
}

// Bad
time.Sleep(5 * time.Second)
```

## Channel Direction

Always declare channel direction in function signatures:

```go
// Good
func produce(out chan<- Event) { ... }  // send-only
func consume(in <-chan Event) { ... }   // receive-only

// Bad
func produce(out chan Event) { ... }
```

## Channels of Channels

Use a channel of channels when a goroutine needs to send a request and receive a response:

```go
type request struct {
    args    []int
    resultC chan<- int
}

func server(reqs <-chan request) {
    for req := range reqs {
        result := compute(req.args)
        req.resultC <- result
    }
}
```

This is a clean alternative to shared state + mutex when fan-in is needed.

## Parallelization

Use `runtime.GOMAXPROCS(0)` or `runtime.NumCPU()` only for CPU-bound work that benefits from true parallelism. For I/O-bound work, goroutine count should be tuned to the target service's capacity, not the local CPU count.

```go
// CPU-bound: parallelize across cores
func parallelHash(chunks [][]byte) [][]byte {
    results := make([][]byte, len(chunks))
    group, _ := errgroup.WithContext(context.Background())
    group.SetLimit(runtime.NumCPU())

    for i, chunk := range chunks {
        group.Go(func() error {
            results[i] = sha256Sum(chunk)
            return nil
        })
    }

    group.Wait()
    return results
}
```

## Channel Size is One or None

Channels should generally be unbuffered (size zero) or buffered with size one. Any other buffer size requires careful justification and scrutiny.

| Size | Use Case |
|------|----------|
| `0` (unbuffered) | Synchronization -- sender blocks until receiver is ready. Guarantees handoff. |
| `1` | Signaling -- allows sender to proceed without waiting. Prevents goroutine leak on fire-once signals. |
| `N > 1` | Requires proof that the buffer size is correct. Must not be used as a substitute for proper backpressure. |

```go
// Good -- unbuffered for synchronization
done := make(chan struct{})

// Good -- size 1 for fire-once signal
result := make(chan *Response, 1)

// Requires justification -- why exactly 100? What happens at 101?
work := make(chan Task, 100)
```

Buffered channels with size > 1 are appropriate for bounded work queues and semaphore patterns, but the size must be a named constant with a documented reason.

## No Goroutines in init()

Packages must not start goroutines in `init()`. There is no way for the caller to control the goroutine's lifecycle -- no context to cancel, no method to call `Stop()`.

Instead, expose a type with explicit lifecycle management:

```go
// Bad -- goroutine started in init, no way to stop it
func init() {
    go refreshLoop()
}

// Good -- caller controls lifecycle
type Refresher struct {
    cancel context.CancelFunc
    done   chan struct{}
}

func NewRefresher(ctx context.Context, interval time.Duration) *Refresher {
    ctx, cancel := context.WithCancel(ctx)
    r := &Refresher{cancel: cancel, done: make(chan struct{})}

    go func() {
        defer close(r.done)
        ticker := time.NewTicker(interval)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                refresh()
            }
        }
    }()

    return r
}

func (r *Refresher) Stop() {
    r.cancel()
    <-r.done
}
```

## Prefer Synchronous Functions

Prefer synchronous APIs. If the caller needs concurrency, they can wrap the call in a goroutine. Removing unnecessary concurrency at the call site is much harder than adding it.

Synchronous APIs:
- Keep goroutines localized to the caller.
- Are easier to test — no goroutine lifecycle to manage.
- Avoid subtle leaks and races at the callee.

```go
// Good -- synchronous; caller decides concurrency
func Process(ctx context.Context, items []Item) ([]Result, error) { ... }

// Bad -- hides concurrency inside the callee, harder to test and reason about
func ProcessAsync(ctx context.Context, items []Item) <-chan Result { ... }
```

## Goroutine Lifetimes Must Be Clear

Every goroutine must have a clear, documented shutdown path. When the pattern is not obvious from the code:
- Document **when and why** the goroutine exits.
- Use `context.Context` or `sync.WaitGroup` to ensure it cannot outlive its parent function.
- Don't use "fire-and-forget" goroutines with indeterminate lifecycles.

## Test for Goroutine Leaks

For packages that start goroutines (workers, pollers, reconcilers, pub/sub consumers), add a leak check to the test suite using [`go.uber.org/goleak`](https://pkg.go.dev/go.uber.org/goleak):

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

Per-test form for finer control:

```go
func TestWorker_ShutsDownCleanly(t *testing.T) {
    defer goleak.VerifyNone(t)

    worker := NewWorker(...)
    worker.Start(t.Context())
    worker.Stop()
}
```

`goleak` asserts no goroutines from the package remain after the test completes. It catches shutdown bugs the race detector does not — stalled workers, un-closed channels, and missing `cancel()` calls. Use it in every package that owns a goroutine lifecycle.

## Anti-Patterns

| Anti-Pattern | Why |
|-------------|-----|
| `go func() { ... }()` without shutdown path | Goroutine leak |
| Shared mutable state without synchronization | Data race (caught by `-race`) |
| Passing `context.Background()` everywhere | Defeats the purpose of context |
| Creating a base `context.Background()` mid-call-chain | Breaks cancellation propagation |
| Custom `Context` types or non-`context.Context` interfaces | Google-banned, no exceptions |
| Context stored in a struct field | Context must flow through calls, not attach to state |
| Closing a channel from the receiver side | Only the sender should close |
| Using `time.Sleep` for synchronization | Flaky, uninterruptible, untestable |
| Ignoring `ctx.Err()` after `ctx.Done()` fires | Swallows the cancellation reason |
| Exposing async APIs when sync would work | Hides complexity; hard to test |
