# 04 — Concurrency

Concurrency in Go is structured, not sprinkled. The core principle is Rob Pike's: don't communicate by sharing memory; share memory by communicating — channels are the primary synchronization mechanism, mutexes guard shared state when a channel would be contortion. Every goroutine has an owner and a shutdown path, every blocking operation respects `context.Context` cancellation, every channel declares its direction, and concurrency stays bounded. This chapter is the canonical home for channel-direction and channel-sizing guidance; correctness under the race detector outranks throughput.

## What good looks like

```go
// Pipeline: produce work, fan out to bounded workers, fan results back in,
// all under one cancellable context with a clear shutdown path.
type Event struct{ ID string }
type Result struct{ ID string }

// produce is send-only on out; it stops the moment ctx is cancelled.
func produce(ctx context.Context, events []Event, out chan<- Event) error {
    defer close(out) // sender closes
    for _, e := range events {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case out <- e:
        }
    }
    return nil
}

// Process is synchronous; the caller decides concurrency. Bounded by maxConcurrency.
func Process(ctx context.Context, events []Event, maxConcurrency int) ([]Result, error) {
    group, ctx := errgroup.WithContext(ctx)
    in := make(chan Event)
    results := make([]Result, 0, len(events))
    var mu sync.Mutex // guards results

    // Producer feeds the channel; closes it when done or cancelled.
    group.Go(func() error { return produce(ctx, events, in) })

    // Bounded fan-out: at most maxConcurrency handlers run at once.
    group.SetLimit(maxConcurrency + 1) // +1 for the producer above
    for ev := range in {
        group.Go(func() error {
            r, err := handle(ctx, ev)
            if err != nil {
                return fmt.Errorf("handle %s: %w", ev.ID, err)
            }
            mu.Lock()
            results = append(results, r)
            mu.Unlock()
            return nil
        })
    }
    if err := group.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

The exemplar shares memory by communicating (4.1): work flows over an unbuffered channel rather than a shared slice. `ctx` is the first parameter and threads every call, with cancellation respected in the producer's `select` (4.2, 4.7). The producer is send-only, the channel direction declared in the signature (4.11). `errgroup` owns the goroutines and propagates the first error (4.3); `SetLimit` bounds the fan-out so goroutines never go unbounded (4.6). The sender closes the channel, never the receiver (4.18). `Process` is synchronous so the caller — not the callee — decides concurrency (4.14), and the one piece of genuinely shared state is guarded by a small-critical-section mutex (4.4).

## Rules

### 4.1 — Share memory by communicating, not the reverse.

**Reasoning, step by step:**
1. Don't communicate by sharing memory; share memory by communicating. This is Go's founding concurrency principle.
2. Channels are Go's primary synchronization mechanism: prefer passing a value over a channel to a goroutine over exposing a shared variable and guarding it.
3. Mutexes remain available for the cases where a channel would be a contortion (see 4.4), but reach for communication first.

**Enforcement:** review.

### 4.2 — Make `context.Context` the first parameter and thread it through.

**Reasoning, step by step:**
1. `context.Context` is always the **first parameter** of every function that does I/O or may be cancelled: `func F(ctx context.Context, ...) (...)`.
2. Never store a context in a struct. Pass it through calls — context flows, it does not attach to state.
3. Never pass `nil` context — use `context.TODO()` if you don't have one yet.
4. Always respect `ctx.Done()` in loops.
5. Derive child contexts with `context.WithTimeout` / `context.WithCancel` for scoped deadlines.
6. There are bounded exceptions where context is not a literal first parameter, because the framework hands it to you another way:
   - HTTP handlers: obtain via `req.Context()`.
   - Streaming RPC methods: obtain via the stream's `Context()`.
   - Go 1.24+ tests: use `t.Context()`.
   - Entrypoints (`main`, `init`): use `context.Background()`.
7. **Never** create a base context (`context.Background()`) in the middle of a call chain. If you need one, you have designed something wrong.
8. **Never** define custom context types or use interfaces other than `context.Context` in function signatures. (Google canonical rule — no exceptions.)
9. Standard context semantics — cancellation interrupts the function, values flow through — do **not** need to be documented. Document only: if the function returns an error other than `ctx.Err()` on cancellation; if there is a non-context mechanism to stop the function (e.g., a `Stop` method); or if the function has special expectations (requires deadline, requires attached values, etc.). **Avoid designing APIs that make such demands.**

```go
// Good
func (c *Client) Fetch(ctx context.Context, url string) (*Response, error) { ... }
func (s *Store) Get(ctx context.Context, key string) ([]byte, error) { ... }

// Bad -- no context
func (c *Client) Fetch(url string) (*Response, error) { ... }
```

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

**Enforcement:** `go vet`; `golangci-lint` (`contextcheck`); review.

### 4.3 — Manage goroutine groups with `errgroup`.

**Reasoning, step by step:**
1. Every goroutine must have a clear shutdown path. Never start a goroutine you cannot stop.
2. Use `errgroup` for managing goroutine groups with error propagation: `errgroup.WithContext` derives a context that is cancelled when the first goroutine returns an error, and `Wait` returns that first error.

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

**Enforcement:** `go test -race`; review.

### 4.4 — Protect shared state with `sync.Mutex`, kept small.

**Reasoning, step by step:**
1. Use `sync.Mutex` for protecting shared state. Channels are for communication; mutexes are for synchronization.

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

2. Observe the mutex rules:

| Rule | Rationale |
|------|-----------|
| Always `defer mu.Unlock()` | Prevents deadlocks on early returns or panics |
| Keep the critical section small | Hold the lock only while accessing shared state |
| Never hold a mutex while doing I/O | I/O can block indefinitely, deadlocking other goroutines |
| Use `sync.RWMutex` when reads vastly outnumber writes | Read-heavy workloads benefit from concurrent read access |
| Put the mutex above the fields it protects | Convention: `mu` guards the fields that follow it |

**Enforcement:** `go test -race`; `golangci-lint`; review.

### 4.5 — Use `sync.Once` for lazy initialization.

**Reasoning, step by step:**
1. Use `sync.Once` for lazy initialization — it guarantees exactly one execution under concurrent access.
2. Capture both the result and any initialization error so subsequent callers see a consistent outcome.

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

**Enforcement:** `go test -race`; review.

### 4.6 — Use `sync.WaitGroup` for fan-out without errors, and bound concurrency always.

**Reasoning, step by step:**
1. Use `sync.WaitGroup` for fan-out/fan-in when you don't need error propagation. Prefer `errgroup` (4.3) when you do.
2. Always call `Add()` before starting the goroutine, not inside it — calling it inside races the `Wait`.

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

3. Never create unbounded goroutines. Use a semaphore channel to cap the number in flight:

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

4. For more sophisticated limiting, use `golang.org/x/sync/semaphore.Weighted`.

**Enforcement:** `go test -race`; review.

### 4.7 — Give every long-running loop a `ctx.Done()` case.

**Reasoning, step by step:**
1. Every long-running loop must include a `ctx.Done()` case in its `select`. Without it, the loop cannot observe cancellation and the goroutine leaks.
2. Return `ctx.Err()` from the `Done` branch so the cancellation reason propagates.

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

**Enforcement:** `go vet`; `go test -race`; review.

### 4.8 — Never use `time.Sleep` in production code.

**Reasoning, step by step:**
1. `time.Sleep` is uninterruptible: a sleeping goroutine ignores cancellation and cannot be stopped early.
2. Use `time.Ticker` or `time.After` inside a `select` with `ctx.Done()` so the wait is cancellable.

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

**Enforcement:** `golangci-lint`; review.

### 4.9 — Use a channel of channels for request/response.

**Reasoning, step by step:**
1. Use a channel of channels when a goroutine needs to send a request and receive a response: the request carries the reply channel.
2. This is a clean alternative to shared state + mutex when fan-in is needed — the server owns the state, callers hand it work and a place to put the answer.

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

**Enforcement:** review.

### 4.10 — Parallelize across cores only for CPU-bound work.

**Reasoning, step by step:**
1. Use `runtime.GOMAXPROCS(0)` or `runtime.NumCPU()` only for CPU-bound work that benefits from true parallelism.
2. For I/O-bound work, goroutine count should be tuned to the target service's capacity, not the local CPU count — sizing to cores starves or floods the downstream.

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

**Enforcement:** review.

### 4.11 — Declare channel direction in every signature.

**Reasoning, step by step:**
1. Always declare channel direction in function signatures: send-only (`chan<- T`) for producers, receive-only (`<-chan T`) for consumers.
2. A directional type makes the contract explicit and lets the compiler reject a producer that accidentally receives or a consumer that accidentally sends — including the receiver-closes mistake (4.18).

```go
// Good
func produce(out chan<- Event) { ... }  // send-only
func consume(in <-chan Event) { ... }   // receive-only

// Bad
func produce(out chan Event) { ... }
```

**Enforcement:** `golangci-lint`; review.

### 4.12 — Keep channel buffer size at one or none.

**Reasoning, step by step:**
1. Channels should generally be unbuffered (size zero) or buffered with size one. Any other buffer size requires careful justification and scrutiny.
2. Pick the size by intent:

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

3. Buffered channels with size > 1 are appropriate for bounded work queues and semaphore patterns, but the size must be a named constant with a documented reason.

**Enforcement:** review.

### 4.13 — Never start goroutines in `init()`.

**Reasoning, step by step:**
1. Packages must not start goroutines in `init()`. There is no way for the caller to control the goroutine's lifecycle — no context to cancel, no method to call `Stop()`.
2. Instead, expose a type with explicit lifecycle management: a constructor that takes a context and returns a handle with a `Stop` method.

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

**Enforcement:** `golangci-lint`; review.

### 4.14 — Prefer synchronous functions.

**Reasoning, step by step:**
1. Prefer synchronous APIs. If the caller needs concurrency, they can wrap the call in a goroutine. Removing unnecessary concurrency at the call site is much harder than adding it.
2. Synchronous APIs keep goroutines localized to the caller, are easier to test (no goroutine lifecycle to manage), and avoid subtle leaks and races at the callee.

```go
// Good -- synchronous; caller decides concurrency
func Process(ctx context.Context, items []Item) ([]Result, error) { ... }

// Bad -- hides concurrency inside the callee, harder to test and reason about
func ProcessAsync(ctx context.Context, items []Item) <-chan Result { ... }
```

**Enforcement:** review.

### 4.15 — Give every goroutine a clear, documented lifetime.

**Reasoning, step by step:**
1. Every goroutine must have a clear, documented shutdown path. When the pattern is not obvious from the code, document **when and why** the goroutine exits.
2. Use `context.Context` or `sync.WaitGroup` to ensure the goroutine cannot outlive its parent function.
3. Don't use "fire-and-forget" goroutines with indeterminate lifecycles.

**Enforcement:** `go test -race`; review.

### 4.16 — Test for goroutine leaks with `goleak`.

**Reasoning, step by step:**
1. For packages that start goroutines (workers, pollers, reconcilers, pub/sub consumers), add a leak check to the test suite using [`go.uber.org/goleak`](https://pkg.go.dev/go.uber.org/goleak).

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

2. Use the per-test form for finer control:

```go
func TestWorker_ShutsDownCleanly(t *testing.T) {
    defer goleak.VerifyNone(t)

    worker := NewWorker(...)
    worker.Start(t.Context())
    worker.Stop()
}
```

3. `goleak` asserts no goroutines from the package remain after the test completes. It catches shutdown bugs the race detector does not — stalled workers, un-closed channels, and missing `cancel()` calls. Use it in every package that owns a goroutine lifecycle.

**Enforcement:** `go test -race` with `goleak`; review.

### 4.17 — Manage worker lifecycle with context plus a done channel.

**Reasoning, step by step:**
1. For long-lived workers, combine context cancellation with a done channel: the context stops the work, the done channel lets a caller block until shutdown completes.
2. `close(w.done)` on exit and a `Stop` that receives from it gives the caller a clean join point.

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

**Enforcement:** `go test -race` with `goleak`; review.

### 4.18 — Avoid the concurrency anti-patterns.

**Reasoning, step by step:**
1. Each of the following carries a concrete failure mode; avoid every one:

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

2. Only the sender closes a channel — closing from the receiver side panics any other sender and corrupts the protocol.

**Enforcement:** `go vet -race`; `go test -race`; `golangci-lint`; review.

## Cross-references

- Error wrapping with `%w` in goroutine bodies: [chapter 03](./03-error-handling.md).
- Channel-direction and channel-of-channels guidance referenced from [chapter 05](./05-api-design.md) and [chapter 12](./12-variables-and-declarations.md) lives here (4.9, 4.11).
- `goleak` and `-race` in the test suite: [go README](./README.md).
