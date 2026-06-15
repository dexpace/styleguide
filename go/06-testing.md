# 06 — Testing

Tests are the executable contract for a package, and Go's `testing` package keeps them plain: functions, not frameworks. Default to table-driven tests that stay flat and declarative, prefer real dependencies over mocks, and write failures that diagnose themselves. Assertions report every problem in one run; only preconditions abort early. Helpers mark themselves with `t.Helper()`, never leak goroutines, and never call `t.Fatal` off the test goroutine. The aim is tests a reader trusts at a glance and a failure message that needs no second look at the source.

## What good looks like

```go
func TestParse(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name    string
        input   string
        want    Result
        wantErr bool
    }{
        {name: "valid input", input: "hello", want: Result{Value: "hello"}},
        {name: "input with spaces", input: "hello world", want: Result{Value: "hello world"}},
        {name: "empty input", input: "", wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := Parse(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            if diff := cmp.Diff(tt.want, got); diff != "" {
                t.Errorf("Parse(%q) mismatch (-want +got):\n%s", tt.input, diff)
            }
        })
    }
}

func assertValidResponse(t *testing.T, response *Response, expectedStatus int) {
    t.Helper()
    require.NotNil(t, response, "response must not be nil")
    assert.Equal(t, expectedStatus, response.StatusCode)
}
```

This exemplar is the default shape (6.1): one flat table, a `name` per row, `wantErr bool` as the only branch, driven through `t.Run` subtests that run in parallel (6.9). It compares whole structures with `cmp.Diff` and a `-want +got` direction key (6.14, 6.15, 6.16), keeps the test logic inline rather than scattered across helpers (6.11), and marks `assertValidResponse` with `t.Helper()` so failures report the caller's line (6.3). Inputs are deterministic fixtures, no global state ties the cases together (6.6).

## Rules

### 6.1 — Default to table-driven tests.

**Reasoning, step by step:**
1. Use table-driven tests as the default pattern. One loop drives every scenario; adding a case is adding a row, not a function.

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

2. Give every table the same structure:

| Field | Purpose |
|-------|---------|
| `name` | Human-readable scenario description. Required. |
| Input fields | One field per input parameter. |
| `want` / `wantErr` | Expected output. Use `wantErr bool` for simple error checks, `wantErrIs error` for sentinel checks. |
| Setup functions (optional) | `setup func(t *testing.T)` for per-case setup. |

**Enforcement:** review.

### 6.2 — Keep table tests flat and declarative.

**Reasoning, step by step:**
1. Table-driven tests should be flat and declarative. Each row is a set of inputs and expected outputs -- nothing more. When table entries contain conditional logic, functions, or multi-branch assertions, the test is harder to read than the code under test.
2. Treat these as **red flags** in table tests:

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

3. Instead, split into separate test functions so each one is flat and obvious:

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

4. A single `wantErr bool` branch is acceptable for short test bodies. Anything beyond that signals the table should be split.

**Enforcement:** review.

### 6.3 — Call `t.Helper()` in every test helper.

**Reasoning, step by step:**
1. Call `t.Helper()` in every test helper function -- it ensures failure messages report the caller's line number rather than the line inside the helper.

```go
func assertValidResponse(t *testing.T, response *Response, expectedStatus int) {
    t.Helper()
    require.NotNil(t, response, "response must not be nil")
    assert.Equal(t, expectedStatus, response.StatusCode)
}
```

**Enforcement:** review.

### 6.4 — Keep assertions inline; don't build `*testing.T` helpers that fail directly.

**Reasoning, step by step:**
1. Follow Google's canonical guidance: **do not create assertion-style helpers that take `*testing.T` and fail the test directly.** They fragment the testing experience, swallow diagnostic information, and stop tests at the first failure when they should report every failure.
2. **Prefer:**

- Inline assertions written in the test function.
- Validation helpers that return `error` (or `(got, ok)`) — the test function decides whether to fail.
- Table-driven tests that keep logic inline.
- `cmp.Diff` / `cmp.Equal` with `cmpopts` for structural comparisons.

3. Take **our exception:** we use `testify/require` and `testify/assert` as a pragmatic compromise. They are permitted because the codebase uses them consistently, and they provide predictable output when used correctly. Google's concerns still apply — minimize reach for `require` beyond preconditions, and write expressive failure messages:

```go
// Good -- testify used with clear intent
require.NoError(t, err, "creating client")
require.NotNil(t, client)

assert.Equal(t, "hotel-123", result.ID)
```

4. **Do not** introduce other assertion libraries (`gocheck`, `gunit`, custom DSLs). Do not build project-specific assertion helpers that accept `*testing.T` and call `t.Fatal` / `t.Errorf` themselves — except for test double setup (`authtest`, `storetest`) where `t.Fatal` on setup failure is appropriate.

**Enforcement:** review.

### 6.5 — Use `testify/require` for preconditions and `testify/assert` for values.

**Reasoning, step by step:**
1. Use `testify/assert` and `testify/require`. Do not mix assertion libraries.
2. Pick the package by what failure should do to the rest of the test:

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

**Enforcement:** review.

### 6.6 — Keep test state self-contained; no global test state.

**Reasoning, step by step:**
1. Have each test set up its own state.
2. Share no package-level variables between tests.
3. Use `TestMain` only for genuine process-level initialization (database, external service).

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

**Enforcement:** review.

### 6.7 — Name tests `TestFunctionName_Scenario`.

**Reasoning, step by step:**
1. Use the format `TestFunctionName_Scenario` with underscores separating function from scenario.

```go
func TestParse_ValidJSON(t *testing.T) { ... }
func TestParse_EmptyInput(t *testing.T) { ... }
func TestParse_MalformedJSON(t *testing.T) { ... }
func TestNewClient_MissingBaseURL(t *testing.T) { ... }
func TestNewClient_WithAllOptions(t *testing.T) { ... }
```

2. For subtests within table-driven tests, let the `name` field provide the scenario:

```go
t.Run("valid JSON", func(t *testing.T) { ... })
t.Run("empty input", func(t *testing.T) { ... })
```

**Enforcement:** review.

### 6.8 — Use `httptest.NewServer` for HTTP-level integration tests.

**Reasoning, step by step:**
1. Use `httptest.NewServer` for HTTP-level integration tests — a real server exercises the actual transport, headers, and status handling that a mock would skip.

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

**Enforcement:** review.

### 6.9 — Mark independent tests parallel.

**Reasoning, step by step:**
1. Mark independent tests as parallel with `t.Parallel()`:

```go
func TestParse_ValidJSON(t *testing.T) {
    t.Parallel()
    // ...
}
```

2. For parallel subtests in a table-driven test, call `t.Parallel()` inside the subtest:

```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        got, err := Parse(tt.input)
        // ...
    })
}
```

**Enforcement:** review.

### 6.10 — Use `t.Cleanup()` instead of `defer` in tests.

**Reasoning, step by step:**
1. Use `t.Cleanup()` instead of `defer` in tests -- it runs even if the test is skipped, and runs in LIFO order after the test completes.

```go
func TestWithServer(t *testing.T) {
    server := httptest.NewServer(handler)
    t.Cleanup(func() { server.Close() })

    // ... use server
}
```

**Enforcement:** review.

### 6.11 — Keep test logic in the test function.

**Reasoning, step by step:**
1. Don't over-abstract test code into shared helpers. Each test function should be readable top-to-bottom without jumping to 5 helper functions. Duplication in tests is far less costly than indirection.

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

2. Extract a helper only when 3+ tests share identical setup AND the setup is non-trivial.

**Enforcement:** review.

### 6.12 — Prefer real dependencies over mocks.

**Reasoning, step by step:**
1. Use real implementations whenever practical. `httptest.NewServer` over a mock transport. An in-memory store over a mock store. Real dependencies catch integration bugs that mocks hide.
2. Mock only when:

- The real dependency is slow, stateful, or non-deterministic (external APIs, databases).
- You need to simulate specific error conditions.
- The dependency has side effects (sending emails, charging cards).

**Enforcement:** review.

### 6.13 — Mock with interfaces and structs, not frameworks.

**Reasoning, step by step:**
1. Use interface + struct for simple mocks. No framework needed.

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

2. Wire the mock directly into the subject under test and assert on what it recorded:

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

3. Use a mocking library (`gomock`, `mockery`) only when the interface is large or you need call-order verification.

**Enforcement:** review.

### 6.14 — Prefer `t.Error` over `t.Fatal`; abort only when continuing is meaningless.

**Reasoning, step by step:**
1. Prefer `t.Error` / `assert.*` by default. Tests should generally report **all** failures in one run — aborting at the first problem hides information.
2. Use `t.Fatal` / `require.*` only when continuing would be meaningless:

- Setup failed (cannot create the subject under test).
- Nil pointer preconditions — subsequent assertions would panic.
- A required external resource is unavailable.

3. For **table-driven tests**:

- Use `t.Run` subtests, and `t.Fatal` inside the subtest when needed — it aborts only the current subtest, not the whole test function.
- Prefer subtests over a flat loop so one table entry's failure doesn't mask others.

**Enforcement:** review.

### 6.15 — Write failures that diagnose themselves.

**Reasoning, step by step:**
1. A test failure should be diagnosable without reading the test source. Include:

- **The function name and inputs** that triggered the failure.
- **The actual result** (the "got").
- **The expected result** (the "want").

2. Use the **standard format:** `YourFunc(%v) = %v, want %v`. Print **got before want**. Use "got" and "want" (not "actual"/"expected").

```go
// Good
if got, want := Parse(input), expected; got != want {
    t.Errorf("Parse(%q) = %q, want %q", input, got, want)
}
```

3. For diffs produced by `cmp.Diff`, include a direction key:

```go
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("Parse(%q) mismatch (-want +got):\n%s", input, diff)
}
```

**Enforcement:** review.

### 6.16 — Compare with `cmp`, never `reflect.DeepEqual`.

**Reasoning, step by step:**
1. Choose the comparison by the value's shape:

- Use `==` for scalar values only.
- Use `cmp.Equal` for structs, slices, maps, and interfaces.
- Use `cmp.Diff` for human-readable diffs in failure messages.
- For protobuf messages, pass `protocmp.Transform()` as a `cmp.Option`.

2. **Never** use `reflect.DeepEqual` in new code — it is overly sensitive to unexported fields.
3. `cmp` is testing-only; do not use it in production code (it may panic on non-comparable types).

**Enforcement:** review.

### 6.17 — Compare whole structures, not field-by-field.

**Reasoning, step by step:**
1. Prefer comparing whole structures over field-by-field comparison — a field-by-field check silently passes when a newly added field is wrong:

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

**Enforcement:** review.

### 6.18 — Compare only stable results.

**Reasoning, step by step:**
1. Don't compare on output whose stability is not guaranteed by its owner — e.g., don't assume `json.Marshal` produces a specific field order. Compare semantically: decode the JSON and compare the decoded structure.

**Enforcement:** review.

### 6.19 — Store complex expected output in golden files.

**Reasoning, step by step:**
1. Store expected output in `testdata/` for complex output validation, gated on an `-update` flag so regeneration is explicit:

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

2. Run with the `-update` flag to regenerate golden files when output intentionally changes.

**Enforcement:** review.

### 6.20 — Organize test files next to code, prefer external test packages.

**Reasoning, step by step:**
1. Keep test files next to the code they test: `client.go` / `client_test.go`.
2. Use `package foo_test` (external test package) for black-box tests that verify the public API.
3. Use `package foo` (same package) only when you need to test unexported internals.
4. Prefer external test packages -- they catch API design problems early.

**Enforcement:** review.

### 6.21 — Provide acceptance tests for extensible (contract) APIs.

**Reasoning, step by step:**
1. When your package defines an interface that others will implement (a "contract" API), provide an acceptance test in a sibling `test` package so implementers can verify conformance:

```go
// storetest/conformance.go
package storetest

// Conformance verifies store satisfies the Store contract.
// Returns a non-nil error if it does not.
func Conformance(ctx context.Context, store store.Store) error {
    // ... run a series of operations, aggregate failures, return error
}
```

2. Have the acceptance function **return an error** rather than call `t.Fatal`. The caller's test decides how to report:

```go
func TestRedisStore_Conformance(t *testing.T) {
    s := newRedisStore(t)
    if err := storetest.Conformance(t.Context(), s); err != nil {
        t.Fatal(err)
    }
}
```

3. Make one of two discipline choices for the acceptance function, and document which:

- **Fail fast** — return on the first invariant violation. Simpler.
- **Aggregate** — collect all failures via `errors.Join` and return at the end. Better for implementers.

**Enforcement:** review.

### 6.22 — Never call `t.Fatal` from a goroutine.

**Reasoning, step by step:**
1. Know that `t.FailNow`, `t.Fatal`, and `t.Fatalf` may only be called from the goroutine running the test function. Calling them from a spawned goroutine does not stop the test — it calls `runtime.Goexit` on the wrong goroutine, leaving the test in an undefined state.
2. From a goroutine, use `t.Errorf` and `return`:

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

3. Note that adding `t.Parallel()` does **not** change this rule.

**Enforcement:** review.

### 6.23 — Never start background goroutines in test helpers without stopping them.

**Reasoning, step by step:**
1. Test helper functions must not start background goroutines. Goroutines that outlive a test cause data races, flaky failures, and panic-on-closed-channel errors.
2. If a helper must start a goroutine, it must also stop it via `t.Cleanup`:

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

**Enforcement:** review.

### 6.24 — Name test doubles by their behavior.

**Reasoning, step by step:**
1. Name test doubles by their behavior, using the prefix that matches what the double does:

| Kind | Prefix | Example |
|------|--------|---------|
| Returns canned data | `stub` | `stubStore`, `stubTokenProvider` |
| Records interactions | `spy` | `spyTransport`, `spyLogger` |
| Full working in-memory impl | `fake` | `fakeCache`, `fakeDB` |
| Generated (gomock/mockery) | `mock` | `mockService` |

**Enforcement:** review.

## Cross-references

- Sentinel errors, `errors.Join`, and error wrapping that tests assert against: [03 — Error Handling](./03-error-handling.md).
- Goroutine lifecycle, `context` cancellation, and the concurrency the goroutine rules guard against: [04 — Concurrency](./04-concurrency.md).
