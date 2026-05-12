# 06 - Testing

## Table-Driven Tests

Use table-driven tests as the default pattern:

```go
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    Result
        wantErr bool
    }{
        {
            name:  "valid input",
            input: "hello",
            want:  Result{Value: "hello"},
        },
        {
            name:    "empty input",
            input:   "",
            wantErr: true,
        },
        {
            name:  "input with spaces",
            input: "hello world",
            want:  Result{Value: "hello world"},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

### Table Test Structure

| Field | Purpose |
|-------|---------|
| `name` | Human-readable scenario description. Required. |
| Input fields | One field per input parameter. |
| `want` / `wantErr` | Expected output. Use `wantErr bool` for simple error checks, `wantErrIs error` for sentinel checks. |
| Setup functions (optional) | `setup func(t *testing.T)` for per-case setup. |

## Avoid Unnecessary Complexity in Table Tests

Table-driven tests should be flat and declarative. Each row is a set of inputs and expected outputs -- nothing more. When table entries contain conditional logic, functions, or multi-branch assertions, the test is harder to read than the code under test.

**Red flags** in table tests:

- `if` statements inside the test loop (beyond `if tt.wantErr`)
- Functions or closures stored in test case fields
- Multiple assertion branches per case
- Different setup logic per case

```go
// Bad -- conditional logic makes the table unreadable
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Process(tt.input)
        if tt.shouldValidate {
            require.NoError(t, err)
            if tt.expectPartial {
                assert.Contains(t, got.Name, tt.partial)
            } else {
                assert.Equal(t, tt.want, got)
            }
        } else {
            require.Error(t, err)
            if tt.checkCode {
                assert.Equal(t, tt.wantCode, err.(*APIError).Code)
            }
        }
    })
}
```

Instead, split into separate test functions:

```go
// Good -- each function is flat and obvious
func TestProcess_ValidInput(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  Result
    }{
        {name: "simple", input: "hello", want: Result{Name: "hello"}},
        {name: "with spaces", input: "hello world", want: Result{Name: "hello world"}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Process(tt.input)
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}

func TestProcess_InvalidInput(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr string
    }{
        {name: "empty", input: "", wantErr: "input required"},
        {name: "too long", input: strings.Repeat("a", 1001), wantErr: "exceeds max length"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := Process(tt.input)
            require.ErrorContains(t, err, tt.wantErr)
        })
    }
}
```

A single `wantErr bool` branch is acceptable for short test bodies. Anything beyond that signals the table should be split.

## t.Helper()

Call `t.Helper()` in every test helper function -- ensures failure messages report the caller's line number.

```go
func assertValidResponse(t *testing.T, response *Response, expectedStatus int) {
    t.Helper()
    require.NotNil(t, response, "response must not be nil")
    assert.Equal(t, expectedStatus, response.StatusCode)
}
```

## Assertion Philosophy

Google's canonical guidance: **do not create assertion-style helpers that take `*testing.T` and fail the test directly.** They fragment the testing experience, swallow diagnostic information, and stop tests at the first failure when they should report every failure.

**Prefer:**
- Inline assertions written in the test function.
- Validation helpers that return `error` (or `(got, ok)`) — the test function decides whether to fail.
- Table-driven tests that keep logic inline.
- `cmp.Diff` / `cmp.Equal` with `cmpopts` for structural comparisons.

**Our exception:** we use `testify/require` and `testify/assert` as a pragmatic compromise. They are permitted because the codebase uses them consistently, and they provide predictable output when used correctly. Google's concerns still apply — minimize reach for `require` beyond preconditions, and write expressive failure messages:

```go
// Good -- testify used with clear intent
require.NoError(t, err, "creating client")
require.NotNil(t, client)

assert.Equal(t, "hotel-123", result.ID)
```

**Do not** introduce other assertion libraries (`gocheck`, `gunit`, custom DSLs). Do not build project-specific assertion helpers that accept `*testing.T` and call `t.Fatal` / `t.Errorf` themselves — except for test double setup (`authtest`, `storetest`) where `t.Fatal` on setup failure is appropriate.

## Assertions: testify

Use `testify/assert` and `testify/require`. Do not mix assertion libraries.

| Package | Behavior | When to Use |
|---------|----------|-------------|
| `require` | Stops the test immediately on failure | Preconditions: nil checks, error checks, anything that makes subsequent assertions meaningless |
| `assert` | Records failure but continues | Value comparisons, non-critical checks |

```go
func TestNewClient(t *testing.T) {
    client, err := NewClient("https://api.example.com")

    require.NoError(t, err)           // stop here if creation failed
    require.NotNil(t, client)         // stop here if nil

    assert.Equal(t, "https://api.example.com", client.BaseURL())  // continue on failure
    assert.Equal(t, 30*time.Second, client.Timeout())
}
```

## httptest.NewServer

Use `httptest.NewServer` for HTTP-level integration tests:

```go
func TestClient_Fetch(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
        assert.Equal(t, "/api/v1/hotels", request.URL.Path)
        assert.Equal(t, "Bearer test-token", request.Header.Get("Authorization"))

        writer.Header().Set("Content-Type", "application/json")
        writer.WriteHeader(http.StatusOK)
        _, _ = writer.Write([]byte(`{"id": "123", "name": "Test Hotel"}`))
    }))
    defer server.Close()

    client, err := NewClient(server.URL, WithToken("test-token"))
    require.NoError(t, err)

    hotel, err := client.FetchHotel(context.Background(), "123")
    require.NoError(t, err)
    assert.Equal(t, "123", hotel.ID)
    assert.Equal(t, "Test Hotel", hotel.Name)
}
```

## Test Naming

Format: `TestFunctionName_Scenario` with underscores separating function from scenario.

```go
func TestParse_ValidJSON(t *testing.T) { ... }
func TestParse_EmptyInput(t *testing.T) { ... }
func TestParse_MalformedJSON(t *testing.T) { ... }
func TestNewClient_MissingBaseURL(t *testing.T) { ... }
func TestNewClient_WithAllOptions(t *testing.T) { ... }
```

For subtests within table-driven tests, the `name` field provides the scenario:

```go
t.Run("valid JSON", func(t *testing.T) { ... })
t.Run("empty input", func(t *testing.T) { ... })
```

## t.Parallel()

Mark independent tests as parallel:

```go
func TestParse_ValidJSON(t *testing.T) {
    t.Parallel()
    // ...
}
```

For parallel subtests in a table-driven test:

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        got, err := Parse(tt.input)
        // ...
    })
}
```

## No Global Test State

- Each test sets up its own state.
- No package-level variables shared between tests.
- Use `TestMain` only for genuine process-level initialization (database, external service).

```go
// Good -- self-contained
func TestTokenCache_Get(t *testing.T) {
    cache := NewTokenCache()
    cache.Set("token-123", time.Now().Add(1*time.Hour))

    got := cache.Get()
    assert.Equal(t, "token-123", got)
}

// Bad -- relies on global state
var globalCache *TokenCache

func TestMain(m *testing.M) {
    globalCache = NewTokenCache()
    os.Exit(m.Run())
}
```

## t.Cleanup()

Use `t.Cleanup()` instead of `defer` in tests -- runs even if test is skipped, runs in LIFO order after test completes.

```go
func TestWithServer(t *testing.T) {
    server := httptest.NewServer(handler)
    t.Cleanup(func() { server.Close() })

    // ... use server
}
```

## Golden Files

Store expected output in `testdata/` for complex output validation:

```go
func TestRender_ComplexTemplate(t *testing.T) {
    got := render(testInput)

    goldenPath := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        err := os.WriteFile(goldenPath, []byte(got), 0644)
        require.NoError(t, err)
    }

    expected, err := os.ReadFile(goldenPath)
    require.NoError(t, err)
    assert.Equal(t, string(expected), got)
}
```

Run with `-update` flag to regenerate golden files when output intentionally changes.

## Mocking Without Frameworks

Use interface + struct for simple mocks. No framework needed.

```go
type mockTransport struct {
    response *http.Response
    err      error
    requests []*http.Request
}

func (m *mockTransport) RoundTrip(request *http.Request) (*http.Response, error) {
    m.requests = append(m.requests, request)
    return m.response, m.err
}
```

Usage:

```go
func TestClient_AddsAuthHeader(t *testing.T) {
    mock := &mockTransport{
        response: &http.Response{StatusCode: http.StatusOK, Body: io.NopCloser(strings.NewReader(""))},
    }

    client := &Client{transport: mock, token: "bearer-xyz"}
    _, err := client.Do(context.Background(), &Request{URL: "/test"})
    require.NoError(t, err)

    require.Len(t, mock.requests, 1)
    assert.Equal(t, "Bearer bearer-xyz", mock.requests[0].Header.Get("Authorization"))
}
```

Use a mocking library (`gomock`, `mockery`) only when the interface is large or you need call-order verification.

## Test File Organization

- Test files live next to the code they test: `client.go` / `client_test.go`
- Use `package foo_test` (external test package) for black-box tests that verify the public API
- Use `package foo` (same package) only when you need to test unexported internals
- Prefer external test packages -- they catch API design problems early

## Keep Test Logic in the Test Function

Don't over-abstract test code into shared helpers. Each test function should be readable top-to-bottom without jumping to 5 helper functions. Duplication in tests is far less costly than indirection.

```go
// Good -- everything visible in the test
func TestClient_Search_ReturnsHotels(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        _, _ = w.Write([]byte(`[{"id":"1","name":"Hotel A"}]`))
    }))
    t.Cleanup(server.Close)

    client, err := NewClient(server.URL)
    require.NoError(t, err)

    hotels, err := client.Search(context.Background(), SearchParams{City: "Seattle"})
    require.NoError(t, err)
    assert.Len(t, hotels, 1)
    assert.Equal(t, "Hotel A", hotels[0].Name)
}

// Bad -- test logic scattered across helpers
func TestClient_Search_ReturnsHotels(t *testing.T) {
    client := setupTestClient(t, hotelFixture())
    assertHotelsReturned(t, client, "Seattle", 1, "Hotel A")
}
```

Extract a helper only when 3+ tests share identical setup AND the setup is non-trivial.

## Prefer Real Dependencies Over Mocks

Use real implementations whenever practical. `httptest.NewServer` over a mock transport. An in-memory store over a mock store. Real dependencies catch integration bugs that mocks hide.

Mock only when:
- The real dependency is slow, stateful, or non-deterministic (external APIs, databases).
- You need to simulate specific error conditions.
- The dependency has side effects (sending emails, charging cards).

## t.Fatal vs t.Error

Prefer `t.Error` / `assert.*` by default. Tests should generally report **all** failures in one run — aborting at the first problem hides information.

Use `t.Fatal` / `require.*` only when continuing would be meaningless:
- Setup failed (cannot create the subject under test).
- Nil pointer preconditions — subsequent assertions would panic.
- A required external resource is unavailable.

For **table-driven tests**:
- Use `t.Run` subtests, and `t.Fatal` inside the subtest when needed — it aborts only the current subtest, not the whole test function.
- Prefer subtests over a flat loop so one table entry's failure doesn't mask others.

## Never Call t.Fatal from a Goroutine

`t.FailNow`, `t.Fatal`, and `t.Fatalf` may only be called from the goroutine running the test function. Calling them from a spawned goroutine does not stop the test — it calls `runtime.Goexit` on the wrong goroutine, leaving the test in an undefined state.

From a goroutine, use `t.Errorf` and `return`:

```go
// Good
go func() {
    if err := background(); err != nil {
        t.Errorf("background: %v", err)
        return
    }
}()

// Bad -- t.Fatal from a goroutine does not stop the test
go func() {
    if err := background(); err != nil {
        t.Fatal(err)  // wrong goroutine
    }
}()
```

Adding `t.Parallel()` does **not** change this rule.

## Useful Test Failures

A test failure should be diagnosable without reading the test source. Include:
- **The function name and inputs** that triggered the failure.
- **The actual result** (the "got").
- **The expected result** (the "want").

**Standard format:** `YourFunc(%v) = %v, want %v`. Print **got before want**. Use "got" and "want" (not "actual"/"expected").

```go
// Good
if got, want := Parse(input), expected; got != want {
    t.Errorf("Parse(%q) = %q, want %q", input, got, want)
}
```

For diffs produced by `cmp.Diff`, include a direction key:

```go
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("Parse(%q) mismatch (-want +got):\n%s", input, diff)
}
```

## Equality Comparison

- Use `==` for scalar values only.
- Use `cmp.Equal` for structs, slices, maps, and interfaces.
- Use `cmp.Diff` for human-readable diffs in failure messages.
- For protobuf messages, pass `protocmp.Transform()` as a `cmp.Option`.
- **Never** use `reflect.DeepEqual` in new code — it is overly sensitive to unexported fields.
- `cmp` is testing-only; do not use it in production code (it may panic on non-comparable types).

## Full Structure Comparisons

Prefer comparing whole structures over field-by-field comparison:

```go
// Good
want := Hotel{ID: "123", Name: "Grand", Rating: 4.5}
got, err := fetch(ctx, "123")
require.NoError(t, err)
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("fetch mismatch (-want +got):\n%s", diff)
}

// Bad -- field-by-field misses future-added fields
got, err := fetch(ctx, "123")
assert.Equal(t, "123", got.ID)
assert.Equal(t, "Grand", got.Name)
assert.InDelta(t, 4.5, got.Rating, 0.001)
```

## Compare Stable Results

Don't compare on output whose stability is not guaranteed by its owner — e.g., don't assume `json.Marshal` produces a specific field order. Compare semantically: decode the JSON and compare the decoded structure.

## Designing Acceptance Tests for Extensible APIs

When your package defines an interface that others will implement (a "contract" API), provide an acceptance test in a sibling `test` package so implementers can verify conformance:

```go
// storetest/conformance.go
package storetest

// Conformance verifies store satisfies the Store contract.
// Returns a non-nil error if it does not.
func Conformance(ctx context.Context, store store.Store) error {
    // ... run a series of operations, aggregate failures, return error
}
```

The acceptance function **returns an error** rather than calling `t.Fatal`. The caller's test decides how to report:

```go
func TestRedisStore_Conformance(t *testing.T) {
    s := newRedisStore(t)
    if err := storetest.Conformance(t.Context(), s); err != nil {
        t.Fatal(err)
    }
}
```

Two discipline choices for the acceptance function:
- **Fail fast** — return on the first invariant violation. Simpler.
- **Aggregate** — collect all failures via `errors.Join` and return at the end. Better for implementers.

Pick one per acceptance function and document it.

## No Goroutines in Test Helpers

Test helper functions must not start background goroutines. Goroutines that outlive a test cause data races, flaky failures, and panic-on-closed-channel errors.

If a helper must start a goroutine, it must also stop it via `t.Cleanup`:

```go
func startBackground(t *testing.T) <-chan Event {
    t.Helper()
    ch := make(chan Event)
    ctx, cancel := context.WithCancel(context.Background())
    t.Cleanup(cancel)

    go func() {
        defer close(ch)
        for {
            select {
            case <-ctx.Done():
                return
            case ch <- produce():
            }
        }
    }()

    return ch
}
```

## Test Double Naming

Name test doubles by their behavior:

| Kind | Prefix | Example |
|------|--------|---------|
| Returns canned data | `stub` | `stubStore`, `stubTokenProvider` |
| Records interactions | `spy` | `spyTransport`, `spyLogger` |
| Full working in-memory impl | `fake` | `fakeCache`, `fakeDB` |
| Generated (gomock/mockery) | `mock` | `mockService` |
