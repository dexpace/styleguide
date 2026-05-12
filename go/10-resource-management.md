# 10 - Resource Management

## defer for Cleanup

Every resource acquisition must have a `defer` close immediately after the error check:

```go
f, err := os.Open(path)
if err != nil {
    return fmt.Errorf("open %s: %w", path, err)
}
defer f.Close()
```

No lines of logic between the error check and the `defer`. Acquire, check, defer -- in that order, always.

## defer Ordering

Defers execute LIFO. Acquire in dependency order; defers automatically release in reverse:

```go
db, _ := sql.Open("postgres", dsn)
defer db.Close()

tx, _ := db.Begin()
defer tx.Rollback()  // safe: no-op after Commit()

// ... do work ...
return tx.Commit()
```

## HTTP Response Bodies

Always close response bodies. Always. Even on error responses:

```go
resp, err := http.DefaultClient.Do(req)
if err != nil {
    return fmt.Errorf("execute request: %w", err)
}
defer resp.Body.Close()
```

Failing to close leaks connections and eventually exhausts the transport pool.

## Context for Cancellation

Use `context.Context` for lifecycle management. All long-running operations accept a context as the first parameter:

```go
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    // ...
}
```

## Timeout Everything

Set timeouts on HTTP clients, database connections, and context deadlines. No unbounded waits:

```go
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
```

Always `defer cancel()` -- even if the context expires naturally, canceling releases resources early.

## Connection Pooling

Configure pool sizes explicitly. Set idle timeouts:

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

Never use `http.DefaultClient` in production code. It has no timeout.

## Graceful Shutdown

Handle `os.Signal` for SIGTERM/SIGINT. Use `context.WithCancel` to propagate shutdown:

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

## io.ReadAll Limits

Never use `io.ReadAll` on untrusted input without a size limit:

```go
// Good -- bounded read
limited := io.LimitReader(resp.Body, maxBodySize)
data, err := io.ReadAll(limited)

// Bad -- unbounded, OOM risk
data, err := io.ReadAll(resp.Body)
```

Define `maxBodySize` as a named constant. Make the limit visible.

## sync.Pool for Hot Paths

Use `sync.Pool` for frequently allocated/freed objects on hot paths (byte buffers, encoders). Not for connection pooling or resource management. Objects in the pool may be collected at any time.

## Never Rely on Finalizers

`runtime.SetFinalizer` is unreliable. Execution timing is non-deterministic and not guaranteed. Use explicit cleanup with `defer` or `Close()`.

## `crypto/rand`, Not `math/rand`

Use `crypto/rand`'s `Reader` for any value that must be unpredictable — keys, tokens, nonces, session identifiers. `math/rand` is predictable and must never be used for security-sensitive values.

For text output from `crypto/rand`, encode as hex or base64.

```go
// Good
buf := make([]byte, 32)
if _, err := rand.Read(buf); err != nil {
    return "", fmt.Errorf("generate token: %w", err)
}
token := hex.EncodeToString(buf)
```

## Slices: Length, Capacity, and Copy

Understand that slices are headers (pointer + length + capacity), not values. Reslicing shares underlying memory.

```go
// Appending may or may not allocate -- depends on capacity
a := make([]int, 0, 5)
b := append(a, 1, 2, 3)  // b shares a's backing array (cap was sufficient)
c := append(b, 4, 5, 6)  // c may have a new backing array (grew past cap)
```

When you need an independent copy:

```go
// Good -- explicit copy
dst := make([]string, len(src))
copy(dst, src)

// Also good -- append to nil slice
dst := append([]string(nil), src...)
```

Never assume a slice returned from a function is safe to mutate unless documented.

## Maps: Comma-Ok Idiom

Always use the two-value form when the zero value is ambiguous:

```go
// Good -- distinguishes "key exists with zero value" from "key missing"
count, ok := m["key"]
if !ok {
    // key not present
}

// Bad -- zero value and missing key are indistinguishable
count := m["key"]  // 0 if missing OR if stored value is 0
```

## Size Hints

Provide capacity hints to `make` whenever you know (or can estimate) the final size. This avoids repeated allocations and copies during growth:

```go
// Good
results := make(map[string]*Hotel, len(ids))
for _, id := range ids {
    results[id] = fetch(id)
}

// Good
buf := make([]byte, 0, expectedSize)
```

A reasonable overestimate is better than no hint at all.
