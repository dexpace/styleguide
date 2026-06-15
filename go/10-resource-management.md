# 10 — Resource Management

Every acquired resource is a debt the program must repay. A file, a connection, a context, a goroutine — each one leaks if the cleanup is missed, and a leak compounds until the pool exhausts or the process is killed. This chapter makes release mechanical: pair each acquisition with a `defer` at the point of acquisition, drive every lifecycle through `context.Context`, bound every wait with a timeout, and configure pools explicitly rather than trusting defaults. Cleanup is not an afterthought handled at the end of the function; it is wired in the moment the resource is born.

## What good looks like

```go
const maxBodySize = 1 << 20 // 1 MiB

func (c *Client) FetchHotels(ctx context.Context, ids []string) (map[string]*Hotel, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel() // release context resources early, even on natural expiry

    db, err := sql.Open("postgres", c.dsn)
    if err != nil {
        return nil, fmt.Errorf("open db: %w", err)
    }
    defer db.Close()

    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return nil, fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() // safe no-op after Commit; defers unwind LIFO, releasing in reverse

    req, err := http.NewRequestWithContext(ctx, "GET", c.url, nil)
    if err != nil {
        return nil, fmt.Errorf("build request: %w", err)
    }
    resp, err := c.http.Do(req) // c.http has explicit timeouts and pool sizes
    if err != nil {
        return nil, fmt.Errorf("execute request: %w", err)
    }
    defer resp.Body.Close()

    limited := io.LimitReader(resp.Body, maxBodySize)
    data, err := io.ReadAll(limited)
    if err != nil {
        return nil, fmt.Errorf("read body: %w", err)
    }

    results := make(map[string]*Hotel, len(ids))
    if err := json.Unmarshal(data, &results); err != nil {
        return nil, fmt.Errorf("decode hotels: %w", err)
    }
    if err := tx.Commit(); err != nil {
        return nil, fmt.Errorf("commit: %w", err)
    }
    return results, nil
}
```

Each resource pairs acquisition with an immediate `defer` (10.1), so the `db`, `tx`, and response body release in reverse order without a manual cleanup epilogue (10.2, 10.3). The context carries the deadline and `cancel` is deferred (10.4, 10.5); the HTTP client is a configured value with timeouts and bounded pools, never `http.DefaultClient` (10.6). The body read is bounded by a named `maxBodySize` constant (10.8), and the result map is sized from `len(ids)` to avoid growth reallocation (10.14).

## Rules

### 10.1 — Pair every acquisition with an immediate `defer` close.

**Reasoning, step by step:**
1. Every resource acquisition must have a `defer` close immediately after the error check. The window between acquiring a resource and arming its cleanup is where leaks hide; close that window to zero lines.
2. Put no lines of logic between the error check and the `defer`. The order is fixed: acquire, check, defer — in that order, always. An early `return` inserted later can never slip past a `defer` that already sits on the line above it.

```go
f, err := os.Open(path)
if err != nil {
    return fmt.Errorf("open %s: %w", path, err)
}
defer f.Close()
```

**Enforcement:** review; `golangci-lint` (errcheck flags the unchecked `Close`).

### 10.2 — Acquire in dependency order; let LIFO release in reverse.

**Reasoning, step by step:**
1. Defers execute LIFO. Acquire resources in dependency order and the deferred releases automatically unwind in the reverse order, tearing down dependents before the things they depend on.
2. A deferred rollback after a deferred commit is safe: `tx.Rollback()` is a no-op once `Commit()` has succeeded, so the same `defer` covers both the success and failure paths without a branch.

```go
db, _ := sql.Open("postgres", dsn)
defer db.Close()

tx, _ := db.Begin()
defer tx.Rollback()  // safe: no-op after Commit()

// ... do work ...
return tx.Commit()
```

**Enforcement:** review.

### 10.3 — Always close HTTP response bodies.

**Reasoning, step by step:**
1. Always close response bodies. Always — even on error responses, where it is easy to forget because the happy path already returned.
2. Failing to close leaks connections and eventually exhausts the transport pool, at which point every new request blocks waiting for a connection that will never free.

```go
resp, err := http.DefaultClient.Do(req)
if err != nil {
    return fmt.Errorf("execute request: %w", err)
}
defer resp.Body.Close()
```

**Enforcement:** `golangci-lint` (bodyclose); review.

### 10.4 — Thread `context.Context` as the first parameter of long-running operations.

**Reasoning, step by step:**
1. Use `context.Context` for lifecycle management — cancellation, deadlines, and request-scoped propagation flow through it rather than through ad-hoc flags.
2. All long-running operations accept a context as the first parameter, and that context is wired into the calls they make (here, `http.NewRequestWithContext`) so cancellation reaches all the way down.

```go
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    // ...
}
```

**Enforcement:** review; `golangci-lint` (contextcheck).

### 10.5 — Bound every wait with a timeout and defer the cancel.

**Reasoning, step by step:**
1. Set timeouts on HTTP clients, database connections, and context deadlines. No unbounded waits — an operation with no deadline can hang forever and pin the resources it holds.
2. Always `defer cancel()`, even if the context expires naturally. Canceling releases the context's resources early instead of waiting for the deadline to elapse.

```go
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
```

**Enforcement:** `go vet` (lostcancel flags a missing `cancel`); review.

### 10.6 — Configure connection pools explicitly; never ship `http.DefaultClient`.

**Reasoning, step by step:**
1. Configure pool sizes explicitly and set idle timeouts. Leaving `MaxIdleConns`, `MaxIdleConnsPerHost`, and `IdleConnTimeout` at their defaults gives you behavior you did not choose and cannot reason about under load.
2. Never use `http.DefaultClient` in production code. It has no timeout, so a single unresponsive upstream can hang the calling goroutine indefinitely.

```go
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
    Timeout: 30 * time.Second,
}
```

**Enforcement:** review; `golangci-lint` (noctx, bodyclose).

### 10.7 — Shut down gracefully on `os.Signal`.

**Reasoning, step by step:**
1. Handle `os.Signal` for SIGTERM/SIGINT so an orchestrator's stop request drains in-flight work instead of dropping it on a hard kill.
2. Use `context.WithCancel` — here via `signal.NotifyContext` — to propagate shutdown, and bound the drain itself with its own timeout so a stuck handler cannot block termination forever.

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

server := &http.Server{Addr: ":8080"}
go func() {
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    server.Shutdown(shutdownCtx)
}()
```

**Enforcement:** review.

### 10.8 — Bound `io.ReadAll` on untrusted input with a named limit.

**Reasoning, step by step:**
1. Never use `io.ReadAll` on untrusted input without a size limit. An attacker (or a misbehaving upstream) can stream until the process runs out of memory.
2. Wrap the reader in `io.LimitReader` with an explicit cap before reading. Define `maxBodySize` as a named constant rather than a magic number so the limit is visible and tunable in one place.

```go
// Good -- bounded read
limited := io.LimitReader(resp.Body, maxBodySize)
data, err := io.ReadAll(limited)

// Bad -- unbounded, OOM risk
data, err := io.ReadAll(resp.Body)
```

**Enforcement:** review; `golangci-lint`.

### 10.9 — Reserve `sync.Pool` for hot-path allocations only.

**Reasoning, step by step:**
1. Use `sync.Pool` for frequently allocated and freed objects on hot paths — byte buffers, encoders — where churn dominates and reuse measurably cuts allocation pressure.
2. Do not use it for connection pooling or resource management. Objects in the pool may be collected at any time, so it offers no lifetime guarantee and is the wrong tool for anything that must be explicitly released.

**Enforcement:** review.

### 10.10 — Never rely on finalizers for cleanup.

**Reasoning, step by step:**
1. `runtime.SetFinalizer` is unreliable: execution timing is non-deterministic and not guaranteed to run at all before exit.
2. Use explicit cleanup with `defer` or `Close()` instead, so release is deterministic and happens at a point you control.

**Enforcement:** review; `golangci-lint` (forbidigo can ban `runtime.SetFinalizer`).

### 10.11 — Use `crypto/rand`, never `math/rand`, for unpredictable values.

**Reasoning, step by step:**
1. Use `crypto/rand`'s `Reader` for any value that must be unpredictable — keys, tokens, nonces, session identifiers. `math/rand` is predictable and must never be used for security-sensitive values.
2. For text output from `crypto/rand`, encode the random bytes as hex or base64 rather than treating raw bytes as a string.

```go
// Good
buf := make([]byte, 32)
if _, err := rand.Read(buf); err != nil {
    return "", fmt.Errorf("generate token: %w", err)
}
token := hex.EncodeToString(buf)
```

**Enforcement:** `golangci-lint` (gosec flags `math/rand` for security use); review.

### 10.12 — Treat slices as headers; copy explicitly when you need independence.

**Reasoning, step by step:**
1. Understand that slices are headers (pointer + length + capacity), not values. Reslicing shares the underlying memory, so two slices can alias the same backing array.
2. Appending may or may not allocate, depending on capacity: an `append` that fits within `cap` mutates the shared backing array, while one that grows past `cap` moves to a fresh array. The aliasing is therefore conditional and easy to get wrong.

```go
// Appending may or may not allocate -- depends on capacity
a := make([]int, 0, 5)
b := append(a, 1, 2, 3)  // b shares a's backing array (cap was sufficient)
c := append(b, 4, 5, 6)  // c may have a new backing array (grew past cap)
```

3. When you need an independent copy, make one explicitly — either via `copy` into a fresh slice or by appending to a `nil` slice. Never assume a slice returned from a function is safe to mutate unless documented.

```go
// Good -- explicit copy
dst := make([]string, len(src))
copy(dst, src)

// Also good -- append to nil slice
dst := append([]string(nil), src...)
```

**Enforcement:** review.

### 10.13 — Use the comma-ok idiom when the zero value is ambiguous.

**Reasoning, step by step:**
1. Always use the two-value form when the zero value is ambiguous. It distinguishes "key exists with the zero value" from "key missing", which the single-value form collapses.
2. The single-value form returns the zero value for both a missing key and a stored zero, making the two cases indistinguishable — a silent bug whenever zero is a legitimate stored value.

```go
// Good -- distinguishes "key exists with zero value" from "key missing"
count, ok := m["key"]
if !ok {
    // key not present
}

// Bad -- zero value and missing key are indistinguishable
count := m["key"]  // 0 if missing OR if stored value is 0
```

**Enforcement:** review.

### 10.14 — Give `make` a capacity hint whenever the size is known.

**Reasoning, step by step:**
1. Provide capacity hints to `make` whenever you know or can estimate the final size. Pre-sizing avoids the repeated allocations and copies a map or slice incurs as it grows.
2. A reasonable overestimate is better than no hint at all; the cost of slightly over-allocating once is far lower than the cost of growing repeatedly.

```go
// Good
results := make(map[string]*Hotel, len(ids))
for _, id := range ids {
    results[id] = fetch(id)
}

// Good
buf := make([]byte, 0, expectedSize)
```

**Enforcement:** review.

## Cross-references

- Error wrapping with `%w` in cleanup paths: [chapter 03](./03-error-handling.md).
- Context propagation and goroutine lifecycle: [chapter 04](./04-concurrency.md).
- Performance cost of allocations and pooling: [chapter 13](./13-performance-hints.md).
- Security-sensitive randomness and secret handling: [security guide](../security.md).
