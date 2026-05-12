# 03 - Error Handling

## The Rule

**Handle every error.** Never assign an error to `_` except in tests or deferred close calls where there is genuinely nothing to do. The `errcheck` linter enforces this.

```go
// Good
data, err := io.ReadAll(response.Body)
if err != nil {
    return fmt.Errorf("read response body: %w", err)
}

// Bad -- error silently discarded
data, _ := io.ReadAll(response.Body)
```

## Wrap with Context

- Never return a bare error. Always wrap with what you were doing when the error occurred.
- Use a lowercase verb phrase. No "failed to" prefix -- the error itself signals failure.

```go
// Good
if err != nil {
    return fmt.Errorf("authenticate with token endpoint: %w", err)
}

// Bad -- no context
if err != nil {
    return err
}

// Bad -- redundant "failed to"
if err != nil {
    return fmt.Errorf("failed to authenticate: %w", err)
}
```

| Good | Bad |
|------|-----|
| `"open config file: %w"` | `"failed to open config file: %w"` |
| `"parse response body: %w"` | `"error parsing response body: %w"` |
| `"authenticate: %w"` | `"authentication failed: %w"` |
| `"create HTTP request: %w"` | `"could not create HTTP request: %w"` |

## %w vs %v: Wrapping Is an API Decision

Use `%w` to preserve the error chain for `errors.Is()` and `errors.As()`. Use `%v` when you intentionally want to **break** the chain.

`%w` makes the wrapped error part of your API contract — callers can inspect it. This is desirable for sentinel errors and typed errors you document. It is **not** desirable for internal implementation errors you don't want callers depending on, and for errors crossing system boundaries (RPC, storage) where the underlying error should be translated into the boundary's error space.

**Use `%w` when:**
- Adding context while preserving the original for `errors.Is` / `errors.As`.
- The wrapped error is part of your package's documented contract.

**Use `%v` when:**
- You want to display the error for humans (logs, debug output) without exposing it to programmatic inspection.
- You are at a system boundary (RPC, IPC, storage) translating internal errors into canonical boundary errors.
- You are wrapping a transient internal error you explicitly don't want callers to depend on.

```go
// Good -- callers should be able to check for ErrNotFound
return fmt.Errorf("get hotel %s: %w", id, ErrNotFound)

// Good -- internal HTTP detail, callers should not depend on this
return fmt.Errorf("fetch config: %v", httpErr)
```

## Placement of `%w`

**Prefer `%w` at the end** of the error string. This produces a "newest-to-oldest" chain that mirrors the error's structure when printed:

```go
// Good
return fmt.Errorf("fetch hotel %s: %w", id, err)
```

**Exception — sentinel categorization:** Place `%w` at the start when the wrapped sentinel is the categorical label:

```go
// Good -- ErrParse surfaces first as the category
return fmt.Errorf("%w: unexpected token at position %d", ErrParse, pos)
```

**Never** place `%w` in the middle of the string — it produces output that is neither newest-to-oldest nor oldest-to-newest.

## Don't Match Error Text

Never distinguish errors by string comparison or regex against `err.Error()`. If callers need to branch on an error, give the error **structure** — a sentinel, a typed error, or a status code.

```go
// Bad
if strings.Contains(err.Error(), "not found") { ... }

// Good
if errors.Is(err, ErrNotFound) { ... }
```

## Don't Add Redundant Annotations

If an error already carries the relevant context (e.g., `*os.PathError` includes the path), don't repeat it when wrapping. If you have nothing to add, just return the error unmodified — don't wrap purely to signal "something failed here."

```go
// Bad -- adds no information over the original
if err != nil {
    return fmt.Errorf("os operation failed: %w", err)
}

// Good -- no annotation; the underlying error already describes the failure
if err != nil {
    return err
}

// Good -- adds meaningful context the caller cannot derive
if err != nil {
    return fmt.Errorf("load hotel %s from cache: %w", id, err)
}
```

## In-Band Errors: Never Sentinel Values

Never signal failure through special return values (`-1`, `""`, `nil`). Always use a second return value — either `error` or a `bool`.

```go
// Good
func Lookup(key string) (value string, ok bool)
func Lookup(key string) (*Entry, error)

// Bad -- "" could be a valid value or a sentinel for "not found"
func Lookup(key string) string
```

## Return Error, Not Concrete Error Types

Exported functions should return `error`, not a concrete error type (e.g., not `*APIError` directly). Returning a concrete pointer type leads to nil-pointer-in-interface bugs: a `nil *APIError` is a non-nil `error`.

```go
// Good
func (c *Client) Fetch(ctx context.Context, id string) (*Hotel, error) {
    return nil, &APIError{Code: 404}
}

// Bad -- callers writing `if err != nil` will incorrectly succeed on nil *APIError
func (c *Client) Fetch(ctx context.Context, id string) (*Hotel, *APIError) { ... }
```

## Error Strings: Lowercase, No Punctuation

Error strings returned from `errors.New` and `fmt.Errorf` must be lowercase and not end in punctuation. They are fragments composed by wrappers, not standalone sentences.

Exception: capitalize when starting with an exported name, a proper noun, or an acronym.

```go
// Good
return errors.New("base URL is required")
return fmt.Errorf("parse %s: %w", path, err)
return errors.New("HTTPS is required")  // acronym exception

// Bad
return errors.New("Base URL is required.")
return fmt.Errorf("Parse failed: %w", err)
```

Full messages that are displayed to users (log lines, HTTP response bodies, CLI output) may be capitalized and punctuated — they are not error strings; they are human-facing text.

## Sentinel Errors

Use package-level `var` for expected error conditions callers need to check for:

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrTimeout      = errors.New("timeout")
    ErrConflict     = errors.New("conflict")
)
```

Check with `errors.Is()`:

```go
if errors.Is(err, ErrNotFound) {
    // handle missing resource
}
```

Use sentinels when the caller's control flow depends on the error kind. If the caller only logs or propagates, a wrapped string error is sufficient.

## Custom Error Types

Use custom types when errors carry structured data beyond a message.

```go
type APIError struct {
    Code    int
    Message string
    cause   error
}

func (e *APIError) Error() string {
    return fmt.Sprintf("API error %d: %s", e.Code, e.Message)
}

func (e *APIError) Unwrap() error {
    return e.cause
}
```

- Always implement `Unwrap()` on custom error types that wrap another error.

### Structured Error Example

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %q: %s", e.Field, e.Message)
}
```

Inspect with `errors.As()`:

```go
var validationErr *ValidationError
if errors.As(err, &validationErr) {
    log.Printf("invalid field: %s", validationErr.Field)
}
```

## errors.Is() and errors.As()

- Always use `errors.Is()` and `errors.As()` for checking error types and values.
- Never use type assertions or direct equality on errors.

```go
// Good
if errors.Is(err, ErrNotFound) { ... }

var apiErr *APIError
if errors.As(err, &apiErr) { ... }

// Bad -- breaks when errors are wrapped
if err == ErrNotFound { ... }
if e, ok := err.(*APIError); ok { ... }
```

## Error Hierarchy via Wrapping

Build error hierarchies through wrapping, not inheritance:

```
APIError (Code: 503)
  wraps -> TransportError ("connection refused")
    wraps -> *net.OpError
      wraps -> *os.SyscallError
```

```go
func executeRequest(request *http.Request) (*http.Response, error) {
    response, err := client.Do(request)
    if err != nil {
        return nil, &TransportError{
            Message: "execute HTTP request",
            cause:   err,
        }
    }

    if response.StatusCode >= 400 {
        return nil, &APIError{
            Code:    response.StatusCode,
            Message: "unexpected status",
        }
    }

    return response, nil
}
```

## Don't Panic

- Panics are for programmer errors only — never for runtime conditions.
- Library code must never panic on bad input. Return an error.
- If a function in a library **can** panic, the caller is the one who pays. Never shift that cost to consumers.
- **Panics must not escape across package boundaries.** If a package uses panic internally (e.g., for deep parser recursion), it must recover at the public API boundary and convert to an error.
- For initialization errors, propagate upward to `main`, which should use `os.Exit` / `log.Exit` with an actionable message rather than letting a stack trace leak.

| Panic is acceptable | Panic is not acceptable |
|--------------------|----------------------|
| Unrecoverable program state (nil function pointer in a table) | Invalid user input |
| Bug in the program's logic (unreachable code reached) | Network failures |
| Failed assertion indicating a programming error | Missing configuration |
| `init()` that cannot load required embedded data | File not found |
| Unreachable code after `log.Fatal` (satisfy the compiler) | Any runtime condition |

## `Must` Functions

Functions named `MustXYZ` (or `mustXYZ` unexported) indicate a setup helper that panics on failure. They are appropriate only:

- Early in program startup (e.g., compiling a regex at init).
- At package initialization.
- In test helpers, where `Must` is conventional shorthand for "fail the test on error."

```go
// Good -- regex compiled at init; failure is a compile-time bug
var urlRegex = regexp.MustCompile(`^https?://`)

// Good -- test helper that fails the test on error
func mustOpenTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := openTestDB()
    if err != nil {
        t.Fatal(err)
    }
    return db
}

// Bad -- ordinary error handling is possible
func MustFetchUser(ctx context.Context, id string) *User { ... }
```

```go
// Good
func NewClient(baseURL string) (*Client, error) {
    if baseURL == "" {
        return nil, errors.New("base URL is required")
    }
    // ...
}

// Bad
func NewClient(baseURL string) *Client {
    if baseURL == "" {
        panic("base URL is required")
    }
    // ...
}
```

## recover() -- When and How

Use `recover()` only at server/handler boundaries to prevent one bad request from crashing the process. Never use `recover()` to implement normal error flow.

```go
// Good -- server boundary protection
func safeHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if v := recover(); v != nil {
                log.Printf("panic in handler: %v\n%s", v, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Bad -- using recover for control flow
func safeParse(s string) (v int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("parse failed: %v", r)
        }
    }()
    return riskyParse(s), nil  // don't do this
}
```

Rules:
- `recover()` only works inside a deferred function.
- A recovered panic does not restore the goroutine's call stack -- the goroutine continues from the deferred function.
- Log the stack trace (`debug.Stack()`) on every recovery. Silent recoveries hide bugs.

## Errors at the Boundary

Validate inputs at public function entry. Return errors immediately.

```go
func (c *Client) Search(ctx context.Context, query SearchQuery) ([]Result, error) {
    if query.Term == "" {
        return nil, &ValidationError{Field: "term", Message: "must not be empty"}
    }
    if query.MaxResults <= 0 {
        return nil, &ValidationError{Field: "maxResults", Message: "must be positive"}
    }
    if query.MaxResults > 1000 {
        return nil, &ValidationError{Field: "maxResults", Message: "must not exceed 1000"}
    }

    return c.doSearch(ctx, query)
}
```

## Inline Error Handling

Check errors immediately after the call. Never defer checking or accumulate errors for later.

```go
request, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return fmt.Errorf("create request: %w", err)
}

response, err := client.Do(request)
if err != nil {
    return fmt.Errorf("execute request: %w", err)
}

defer response.Body.Close()

data, err := io.ReadAll(response.Body)
if err != nil {
    return fmt.Errorf("read response body: %w", err)
}
```

## Handle Errors Once

An error should be handled exactly once. Handling means one of: logging, returning to the caller, or degrading gracefully. Never log AND return -- the caller will likely log it again, producing duplicate noise.

```go
// Good -- wrap and return, let the caller decide how to handle
func (c *Client) fetchConfig(ctx context.Context) (*Config, error) {
    resp, err := c.httpClient.Get(ctx, c.configURL)
    if err != nil {
        return nil, fmt.Errorf("fetch config from %s: %w", c.configURL, err)
    }

    return parseConfig(resp)
}

// Good -- log and degrade (at the boundary, not returning the error)
func (s *Server) handleRequest(w http.ResponseWriter, r *http.Request) {
    result, err := s.process(r.Context(), r)
    if err != nil {
        s.logger.Error("process request", "error", err, "path", r.URL.Path)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    writeJSON(w, result)
}

// Bad -- log AND return (caller will log again)
func (c *Client) fetchConfig(ctx context.Context) (*Config, error) {
    resp, err := c.httpClient.Get(ctx, c.configURL)
    if err != nil {
        log.Printf("failed to fetch config: %v", err)  // logged here
        return nil, fmt.Errorf("fetch config: %w", err) // AND returned
    }

    return parseConfig(resp)
}
```

When a callee's contract defines specific errors, you may match and recover:

```go
result, err := cache.Get(ctx, key)
if errors.Is(err, ErrCacheMiss) {
    // specific error -- recover by fetching from source
    return c.fetchFromSource(ctx, key)
}
if err != nil {
    return nil, fmt.Errorf("cache lookup for %s: %w", key, err)
}
```

## Assertion Discipline

### Assertion Density

Target a minimum of 2 assertions per function on average. Error checks (`if err != nil`) count as assertions. High assertion density catches bugs close to their origin.

### Assert Preconditions

```go
func ProcessResponse(statusCode int, body []byte) (*Result, error) {
    if statusCode < 100 || statusCode > 599 {
        return nil, fmt.Errorf("invalid HTTP status code: %d", statusCode)
    }
    if body == nil {
        return nil, errors.New("response body must not be nil")
    }
    // ...
}
```

### Split Compound Assertions

Separate each validation into its own check -- never combine. On failure, you know exactly which condition was violated. Simpler checks are easier to reason about.

```go
// Good
if body == nil {
    return nil, errors.New("response body is nil")
}
if len(body) == 0 {
    return nil, errors.New("response body is empty")
}

// Bad
if body == nil || len(body) == 0 {
    return nil, errors.New("response body is invalid")
}
```

### Assert Positive AND Negative Space

Assert what you expect AND what you don't expect. Bugs live at boundaries between valid and invalid states.

```go
// Good -- positive and negative
if len(results) == 0 {
    return nil, errors.New("results must not be empty")
}
if len(results) > maxResults {
    return nil, fmt.Errorf("results count %d exceeds max %d", len(results), maxResults)
}
```

### Pair Assertions

Validate the same property from at least two different code paths. If one assertion is wrong, the other catches it.

```go
func (b *Buffer) Write(data []byte) error {
    // Assert before: capacity is sufficient
    if b.offset + len(data) > b.capacity {
        return fmt.Errorf("write of %d bytes exceeds remaining capacity %d", len(data), b.capacity - b.offset)
    }

    copy(b.data[b.offset:], data)
    b.offset += len(data)

    // Assert after: offset is still within bounds
    if b.offset > b.capacity {
        return fmt.Errorf("buffer offset %d exceeds capacity %d after write", b.offset, b.capacity)
    }

    return nil
}
```

### Handle Every Error

Never silently swallow errors. If you intentionally ignore one, explain why in a comment.

```go
// Bad
_ = file.Close()

// Good -- intentionally ignored with explanation
// Close error on read-only file is safe to ignore; data was already consumed.
_ = file.Close()

// Better
if err := file.Close(); err != nil {
    log.Printf("close config file: %v", err)
}
```

## Deferred Close Pattern

For read-only resources:

```go
func readConfig(path string) ([]byte, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("open config file %s: %w", path, err)
    }
    defer file.Close()

    data, err := io.ReadAll(file)
    if err != nil {
        return nil, fmt.Errorf("read config file %s: %w", path, err)
    }

    return data, nil
}
```

When the close error matters (writes), capture it:

```go
func writeConfig(path string, data []byte) (retErr error) {
    file, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create config file: %w", err)
    }
    defer func() {
        if closeErr := file.Close(); closeErr != nil && retErr == nil {
            retErr = fmt.Errorf("close config file: %w", closeErr)
        }
    }()

    if _, err := file.Write(data); err != nil {
        return fmt.Errorf("write config file: %w", err)
    }
    return nil
}
```
