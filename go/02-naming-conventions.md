# 02 — Naming Conventions

Names in Go scale with scope, carry their case mechanically, and never stutter against the package that already qualifies them. This chapter restates Google's Go naming rules natively: size a name to its scope, use `MixedCaps` (never underscores or `SCREAMING_CASE`), name interfaces with `-er`, constants by role, and initialisms in one consistent case. Where Google's guidance governs casing, it wins; the taste rules — proportion, no repetition, behavior-named test doubles — are what make a package legible at the call site.

## What good looks like

```go
// Package tokenstore caches signed tokens with a bounded TTL.
package tokenstore

import (
	"context"
	"errors"
	"time"
)

// ErrExpired is returned when a token is past its deadline.
var ErrExpired = errors.New("token expired")

const defaultTTL = 5 * time.Minute

// Fetcher retrieves a raw token for a subject from an upstream source.
type Fetcher interface {
	Fetch(ctx context.Context, subject string) (string, error)
}

// Store caches tokens fetched from an upstream Fetcher.
type Store struct {
	fetcher  Fetcher
	ttl      time.Duration
	tokens   map[string]entry
}

type entry struct {
	value       string
	expiresAt   time.Time
}

// NewStore builds a Store; ttl <= 0 falls back to defaultTTL.
func NewStore(f Fetcher, ttl time.Duration) *Store {
	if ttl <= 0 {
		ttl = defaultTTL
	}
	return &Store{fetcher: f, ttl: ttl, tokens: map[string]entry{}}
}

// Get returns a cached token for subject, fetching on a miss.
func (s *Store) Get(ctx context.Context, subject string) (string, error) {
	if e, ok := s.tokens[subject]; ok && time.Now().Before(e.expiresAt) {
		return e.value, nil
	}
	value, err := s.fetcher.Fetch(ctx, subject)
	if err != nil {
		return "", err
	}
	s.tokens[subject] = entry{value: value, expiresAt: time.Now().Add(s.ttl)}
	return value, nil
}
```

This package practices the chapter throughout. Names are scope-proportional — `f`, `s`, `e`, `ok` in tight scope, `subject` and `expiresAt` where they live longer (2.1); `MixedCaps`/`mixedCaps` carry exportedness with no underscores or `SCREAMING_CASE` (2.3, 2.21); the package name `tokenstore` is short and lowercase and never stutters, so the type is `Store`, not `TokenStore` (2.4, 2.7). The receiver `s` is a one-letter abbreviation used consistently (2.5); `defaultTTL` names a role, not a value, with its initialism in one case (2.6, 2.15); the single-method `Fetcher` takes the `-er` suffix (2.12); `NewStore` is a `New`-prefixed constructor returning `*Store` (2.17); `ErrExpired` is a sentinel with the `Err` prefix (2.16); and `Get` reads as a noun-like accessor with no `Get` stutter beyond the resource verb (2.13, 2.14).

## Rules

### 2.1 — Size every name to its scope.

**Reasoning, step by step:**
1. The length of a name should be **proportional to the size of its scope** and **inversely proportional to the number of times it is used** (Google rule). A name living in a three-line loop and used twice earns one letter; a package-level identifier read across files earns full words.
2. Match the name to the scope band:

| Scope | Convention |
|-------|-----------|
| 1-7 lines (tight loop, short function) | Single-word or single-letter names: `i`, `n`, `r`, `buf` |
| 8-15 lines (medium function) | Short but descriptive: `count`, `user`, `resp` |
| 15-25 lines | More descriptive: `userCount`, `retryDeadline` |
| 25+ lines or package-level | Multiple words as needed: `orderedPartnerAccounts` |

3. Short scope permits short names; large scope demands clarity:

```go
// Good -- short name in small scope
for i, user := range users {
    if !user.Active { continue }
    process(i, user)
}

// Bad -- over-qualified in a tight loop
for indexInList, currentUser := range allUsersInSystem {
    // ...
}
```

4. This rule **supersedes** any blanket "spell it out" preference. `resp` is appropriate in a 10-line HTTP handler; `response` is appropriate at package scope.
5. Do not drop letters to save typing (`Sbx` is not an acceptable shorthand for `Sandbox`).

**Enforcement:** review; `golangci-lint` `varnamelen` flags names that are too short for their scope.

### 2.2 — Encode visibility with case, not keywords.

**Reasoning, step by step:**
1. Uppercase first letter = exported (public API). Lowercase first letter = unexported (internal).
2. Go has no `public`, `private`, or `protected` keywords; the case of the first letter is the entire access-control mechanism.

```go
type Client struct { ... }     // exported -- part of the package API
type clientConfig struct { ... } // unexported -- internal implementation detail
```

**Enforcement:** compiler (unexported names are not visible outside the package); `revive` `exported` checks exported identifiers carry doc comments.

### 2.3 — Case identifiers with MixedCaps, never underscores or SCREAMING_CASE.

**Reasoning, step by step:**
1. Apply the casing table by kind, exported vs unexported:

| Kind | Convention | Good | Bad |
|------|-----------|------|-----|
| Exported types | MixedCaps | `HTTPClient`, `TokenStore` | `Http_Client`, `tokenStore` |
| Unexported types | mixedCaps | `requestBuilder`, `authState` | `request_builder`, `AuthState` |
| Exported functions | MixedCaps | `NewClient`, `ParseToken` | `New_Client`, `parseToken` |
| Unexported functions | mixedCaps | `validateConfig`, `buildURL` | `validate_config`, `ValidateConfig` |
| Constants | MixedCaps (not SCREAMING) | `MaxRetries`, `DefaultTimeout` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Local variables | mixedCaps, short | `resp`, `ctx`, `buf` | `the_response`, `myContext` |

2. Constants are `MixedCaps` like everything else — Go has no `SCREAMING_SNAKE_CASE` convention for constants.
3. Never use underscores in Go names — except in test function names (`TestParse_EmptyInput`). The full underscore policy lives in 2.21.

**Enforcement:** `gofmt` and `revive` `var-naming`; `golangci-lint` `stylecheck` (ST1003) flags non-MixedCaps identifiers.

### 2.4 — Keep package names short, lowercase, and shadow-proof.

**Reasoning, step by step:**
1. Use short names of lowercase letters and numbers only. No `camelCase`, no underscores.
2. Multi-word package names must remain unbroken and all lowercase (e.g., `tabwriter`, not `tabWriter` or `tab_writer`).
3. The package name is part of every call site — it must compose well.
4. Avoid names likely to be shadowed by common local variable names (prefer `usercount` over `count`).
5. Never stutter: if the package is `token`, the type is `Token`, not `TokenToken`.
6. Avoid uninformative names: `util`, `utility`, `common`, `helper`, `model`, `testhelper`.

| Good | Bad | Why Bad |
|------|-----|---------|
| `transport` | `transportpkg` | Redundant suffix |
| `auth` | `authentication` | Too long |
| `pipeline` | `utils` | Describes nothing |
| `token` | `common` | Junk drawer |
| `store` | `shared` | Says where, not what |
| `config` | `base` | Implies inheritance Go doesn't have |

7. **Underscore exceptions for package names:**
   - `_test` suffix for black-box test packages (`linkedlist_test`).
   - Underscores within integration test package names (`linked_list_service_test`).
   - `_test` suffix for package-level documentation example files.

**Enforcement:** `revive` `package-comments` and `var-naming`; `golangci-lint` `stylecheck` (ST1003) flags underscores and mixed case in package names.

### 2.5 — Name receivers short, consistent, and derived from the type.

**Reasoning, step by step:**
1. Receiver names must be **short** — one or two letters.
2. They must be **an abbreviation of the type name** — derive from the type.
3. They must be **consistent** — every method on a type uses the same receiver name.
4. **Never `this`, `self`, or `me`**.
5. **Never `_`** — omit the receiver if unused: `func (Validator) IsValid() bool`.

```go
// Good
func (c *Client) Do(...) { ... }
func (c *Client) Close() error { ... }

func (ri *ResearchInfo) Summary() string { ... }

// Bad
func (this *Client) Do(...) { ... }
func (self *Scanner) Next() bool { ... }
func (client *Client) Do(...) { ... } // too long; use c
```

**Enforcement:** `revive` `receiver-naming` (consistency) and `golangci-lint` `stylecheck` (ST1006) reject `this`/`self`/`me`.

### 2.6 — Name constants by role, not value.

**Reasoning, step by step:**
1. Name constants by their **role**, not their value. A constant whose name describes only its value is unnecessary.

```go
// Good
const MaxPacketSize = 512
const DefaultTimeout = 30 * time.Second

// Bad
const Twelve = 12
const FiveHundred = 500
```

2. If you catch yourself writing `const X = "X"`, the constant serves no purpose — inline the value or find a role-based name.

**Enforcement:** review.

### 2.7 — Don't repeat the package, type, receiver, or parameters in a name.

**Reasoning, step by step:**
1. The package name is already at the call site. Don't repeat it in type or function names:

```go
// Good -- package name provides context
package auth
type Provider struct { ... }       // auth.Provider
func NewProvider() *Provider       // auth.NewProvider()

// Bad -- stuttering
package auth
type AuthProvider struct { ... }   // auth.AuthProvider
func NewAuthProvider() *AuthProvider // auth.NewAuthProvider()
```

2. This applies to fields, methods, and constants too:

```go
// Good
package config
type Loader struct { ... }
const DefaultPath = "/etc/app.yaml"

// Bad
package config
type ConfigLoader struct { ... }
const ConfigDefaultPath = "/etc/app.yaml"
```

3. Don't repeat types, receivers, or parameters in function/method names:

```go
// Good
func (c *Config) WriteTo(w io.Writer) error { ... }
func Override(dest, source *Settings) { ... }
func Transform(input []byte) []byte { ... }
func yamlconfig.Parse(data []byte) (*Config, error) { ... }

// Bad
func (c *Config) WriteConfigTo(w io.Writer) error { ... }        // repeats receiver type
func OverrideFirstWithSecond(dest, source *Settings) { ... }      // repeats param names
func TransformToBytes(input []byte) []byte { ... }                // repeats return type
func yamlconfig.ParseYAMLConfig(data []byte) (*Config, error) { ... } // repeats package + return type
```

4. Don't repeat concepts from the surrounding context. In package `sqldb`, use `Connection`, not `DBConnection`. On `type Project struct`, use `Name()`, not `ProjectName()`. In package `ads/targeting`, use `id` for a local variable — not `adsTargetingID`.

**Enforcement:** `revive` `exported` (flags `pkg.PkgType` stutter); `golangci-lint` `stylecheck` (ST1016) and review.

### 2.8 — Never shadow package, import, or builtin names; distinguish stomping from shadowing.

**Reasoning, step by step:**
1. Never shadow package-level names, imported packages, or outer scope variables. Shadowing silently hides bugs:

```go
// Bad -- shadows the outer err
err := validate(input)
if err != nil {
    return err
}
result, err := process(input) // this err is fine -- same scope reassignment

if condition {
    err := otherCall()  // BAD: shadows outer err, outer err never updated
    _ = err
}

// Bad -- shadows imported package
var context string  // shadows "context" package

// Bad -- shadows builtin
len := computeLength()  // shadows builtin len()
```

2. **Stomping** — reusing an existing name with `:=` in the same scope where at least one variable on the left is new — is fine when the old value is no longer needed.
3. **Shadowing** — declaring a new variable in an inner scope with `:=` that shadows an outer variable — silently hides the outer value from subsequent code.
4. To avoid shadowing bugs, use `=` in inner scopes with pre-declared variables, or choose a new name for clarity.
5. **Never shadow** imported packages (`url`, `context`, `log`) or Go built-ins (`len`, `copy`, `error`).

**Enforcement:** `go vet -vettool` `shadow` analyzer; `golangci-lint` `govet` (shadow) and `predeclared`.

### 2.9 — Name test helper packages by appending `test`.

**Reasoning, step by step:**
1. Name test helper packages by appending `test` to the production package name: `auth` → `authtest`, `creditcard` → `creditcardtest`. Mark the Bazel `go_library` (or equivalent) as `testonly`.
2. **When one type is doubled**, use concise names:
   - `authtest.Stub` is strictly preferable to `authtest.StubService`.
   - `authtest.Fake` over `authtest.FakeService`.
3. **When multiple behaviors matter**, name by behavior: `authtest.AlwaysAllow`, `authtest.AlwaysDeny`, `creditcardtest.AlwaysCharges`, `creditcardtest.AlwaysDeclines`.
4. **When multiple types are doubled**, be explicit: `StubService`, `StubStoredValue`.

**Enforcement:** review.

### 2.10 — Prefix test-double variables to distinguish them from production types.

**Reasoning, step by step:**
1. When a test variable refers to a test double, **prefix the variable name** to distinguish it from a production type:

```go
// Good
spyCC := creditcardtest.NewSpy()
stubStore := storetest.NewStub()

// Bad
cc := creditcardtest.NewSpy()  // looks like a production credit card
```

**Enforcement:** review.

### 2.11 — Name test doubles by behavior, not by "mock".

**Reasoning, step by step:**
1. Name test doubles by their behavior, not by the word "mock":

| Pattern | Name | When |
|---------|------|------|
| Records calls for later inspection | `spyTransport` | Verifying interactions |
| Returns canned data | `stubStore` | Providing test data |
| Full working implementation (in-memory) | `fakeCache` | Integration-style tests with simulated dependency |

```go
type spyTransport struct {
    requests []*http.Request
    response *http.Response
}

type fakeStore struct {
    data map[string][]byte
}
```

2. Reserve "mock" for generated mocks from frameworks (`gomock`, `mockery`).

**Enforcement:** review.

### 2.12 — Name interfaces with `-er` for one method, a noun for many.

**Reasoning, step by step:**
1. **Single-method interfaces**: use the `-er` suffix.

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }
type Handler interface { ServeHTTP(ResponseWriter, *Request) }
```

2. **Multi-method interfaces**: use a descriptive noun. No `-er` suffix.

```go
type Transport interface {
    RoundTrip(req *http.Request) (*http.Response, error)
    Close() error
}

type Store interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Put(ctx context.Context, key string, value []byte) error
    Delete(ctx context.Context, key string) error
}
```

**Enforcement:** review.

### 2.13 — Drop the `Get` prefix on getters; use `Set`, `Compute`, or `Fetch` deliberately.

**Reasoning, step by step:**
1. Never use `Get`/`get` prefix on getters, **unless the underlying concept uses "get"** (e.g., HTTP `GET`).
2. Use `Set` prefix on setters.
3. For expensive work, use `Compute` or `Fetch` rather than `Get`.

```go
// Good
func (u *User) Name() string          { return u.name }
func (u *User) SetName(n string)      { u.name = n }
func (r *Report) Compute() Statistics { ... }  // expensive, not a simple getter
func (c *Client) FetchUser(ctx context.Context, id string) (*User, error) { ... } // remote call

// Bad
func (u *User) GetName() string { return u.name }
```

**Enforcement:** `revive` `get-return` flags `Get`-prefixed methods that take arguments or do not return; review.

### 2.14 — Use nouns for value functions, verbs for action functions.

**Reasoning, step by step:**
1. Functions that return something get **noun-like** names: `JobName()`, `Count()`.
2. Functions that do something get **verb-like** names: `WriteDetail()`, `Close()`.
3. When identical functions differ only by type, append the type: `ParseInt`, `ParseInt64`, `AppendInt`, `AppendInt64`. If one is the "primary" version, omit the type from it: `Marshal()` vs `MarshalText()`.

**Enforcement:** review.

### 2.15 — Keep initialisms in a single consistent case.

**Reasoning, step by step:**
1. Words that are initialisms or acronyms use the **same case throughout** — all caps or all lowercase. Never mixed.
2. **Standard uppercase initialisms** (`XML`, `API`, `ID`, `DB`, `URL`, `HTTP`, `UUID`, `JSON`):

| Exported | Unexported |
|----------|-----------|
| `XMLAPI`, `ID`, `HTTPClient` | `xmlAPI`, `id`, `httpClient` |
| `URLParser`, `UUIDGenerator` | `urlParser`, `uuidGenerator` |

3. For unexported names, the **first** initialism is lowercased; subsequent ones stay uppercase.
4. **Lowercase-prefixed initialisms** (`iOS`, `gRPC`, `DDoS`): appear as in standard prose unless exportedness forces a case change.

| Identifier | Exported | Unexported |
|-----------|----------|-----------|
| iOS | `IOS` | `iOS` |
| gRPC | `GRPC` | `gRPC` |
| DDoS | `DDoS` | `ddos` |

5. **Special case:** `Txn` exported is `Txn`, not `TXN`.

| Good | Bad |
|------|-----|
| `URL` | `Url` |
| `userID` | `userId`, `userId` |
| `parseURL` | `parseUrl` |
| `HTTPClient` (exported) | `HttpClient` |
| `httpClient` (unexported) | `HTTPClient` (unexported) |

**Enforcement:** `golangci-lint` `stylecheck` (ST1003) and `revive` `var-naming` flag mixed-case initialisms.

### 2.16 — Name sentinel errors `Err…` and error types `…Error`.

**Reasoning, step by step:**
1. **Sentinel errors**: `Err` prefix, package-level `var`.

```go
var ErrNotFound = errors.New("not found")
var ErrTimeout = errors.New("timeout")
var ErrUnauthorized = errors.New("unauthorized")
```

2. **Error types**: `Error` suffix.

```go
type ValidationError struct { Field string; Message string }
type TransportError struct { StatusCode int; cause error }
type ConfigError struct { Key string; Reason string }
```

**Enforcement:** `golangci-lint` `stylecheck` (ST1012) flags error variables not named with the `Err`/`err` prefix.

### 2.17 — Prefix constructors with `New` and signal fallibility in the return.

**Reasoning, step by step:**
1. Always use `New` prefix.

| Pattern | When |
|---------|------|
| `NewClient(...)` | Primary constructor |
| `NewClientFromConfig(cfg Config)` | Constructor from a specific source |
| `Open(path string)` | Resources that need closing (files, connections) |
| `Parse(s string)` | Deserialization / parsing |

2. Return `(*T, error)` when construction can fail.
3. Return `*T` only when construction is infallible.

```go
// Can fail
func NewClient(baseURL string, opts ...Option) (*Client, error) { ... }

// Cannot fail
func NewBuffer(size int) *Buffer { ... }
```

**Enforcement:** review.

### 2.18 — Govern abbreviations by scope; keep only idiomatic shorthands.

**Reasoning, step by step:**
1. Abbreviations are governed by the [scope rule](#21--size-every-name-to-its-scope). Short scope permits short names; large scope demands clarity.
2. Universal idioms (acceptable at any scope):

| Abbreviation | Full Form |
|-------------|-----------|
| `ctx` | `context.Context` |
| `err` | `error` |
| `r` | `io.Reader` or `*http.Request` |
| `w` | `io.Writer` or `http.ResponseWriter` |
| `i`, `j`, `k` | Integer loop indices |
| `x`, `y`, `z` | Coordinates |

3. Domain abbreviations (acceptable in tight scope, spell out at package level):

| Abbreviation | Full Form |
|-------------|-----------|
| `req`, `resp` | request, response |
| `buf` | buffer |
| `cfg` | config |
| `n` | count / length in a short loop |

4. Do not drop letters to save typing in exported names (`Sbx` for `Sandbox`, `Usr` for `User`).
5. Omit type information from names — prefer `users` over `userSlice`, `count` over `userCount` unless disambiguation is needed.

**Enforcement:** review; `golangci-lint` `varnamelen` for scope-mismatched abbreviations.

### 2.19 — Add units to names, qualifier last.

**Reasoning, step by step:**
1. Include units when a value has one. Put units and qualifiers last, sorted by descending significance: `timeout_ms_max` not `max_timeout_ms`. This groups related variables together when sorted alphabetically.

```go
// Good -- qualifier last, groups by concept
const timeoutMsDefault = 30000
const timeoutMsMax     = 60000
const bufferSizeBytesMax = 10 * 1024 * 1024
const retriesMax = 3
const tokenExpirationBufferSeconds = 60

// Bad -- qualifier first, scatters related names
const defaultTimeoutMs = 30000
const maxTimeoutMs     = 60000
const maxBufferSizeBytes = 10 * 1024 * 1024
const maxRetries = 3
```

2. A name with no units at all is ambiguous about its scale:

```go
// Bad -- no units at all
const defaultTimeout = 30000    // milliseconds? seconds? minutes?
const maxBufferSize = 10000000  // bytes? KB? records?
```

3. For `time.Duration`, the type carries the unit -- no suffix needed:

```go
const defaultTimeout = 30 * time.Second  // clear from the type
```

**Enforcement:** review.

### 2.20 — Name tests `TestFunc_Scenario` or use subtests.

**Reasoning, step by step:**
1. Format: `TestFunctionName_Scenario`. Underscores between the function name and scenario are expected — they are one of Go's three permitted underscore use cases.

```go
func TestParse_ValidInput(t *testing.T) { ... }
func TestParse_EmptyString(t *testing.T) { ... }
func TestNewClient_MissingURL(t *testing.T) { ... }
```

2. For subtests, prefer `t.Run(description, ...)` over encoding scenarios in the top-level function name:

```go
func TestParse(t *testing.T) {
    t.Run("valid input", func(t *testing.T) { ... })
    t.Run("empty string", func(t *testing.T) { ... })
}
```

3. Either pattern is valid. Choose one per package for consistency.

**Enforcement:** review; `go test` requires the `Test` prefix to run a test.

### 2.21 — Forbid underscores in Go names outside the permitted cases.

**Reasoning, step by step:**
1. Names in Go must not contain underscores (Google canonical rule — overrides any earlier guidance that prescribed underscore prefixes).
2. Exceptions:
   - `Test`, `Benchmark`, `Example`, `Fuzz` function names in `*_test.go` files (e.g., `TestParse_EmptyInput`).
   - Package names imported only by generated code.
   - Low-level libraries interoperating with the OS or cgo (very rare).
3. Distinguish package-level values from locals by:
   1. Declaring them in a grouped `var ( ... )` block near the top of the file.
   2. Using descriptive names: `defaultTimeout`, `maxRetries` — no underscore prefix.
   3. Sentinel errors use the `err` / `Err` prefix (`errNotFound`, `ErrNotFound`).

```go
// Good
var (
    defaultTimeout = 30 * time.Second
    maxRetries     = 3
)

// Bad -- Go names may not contain underscores
var _defaultTimeout = 30 * time.Second
```

**Enforcement:** `golangci-lint` `stylecheck` (ST1003) and `revive` `var-naming` flag underscores in identifiers.

### 2.22 — Never name identifiers after predeclared builtins.

**Reasoning, step by step:**
1. Never use Go's predeclared identifiers as variable, function, or type names. They compile but shadow the builtin, creating confusing, hard-to-grep code:

```go
// Bad -- shadows builtins
var error string                // shadows the error interface
var string int                  // shadows the string type
func copy(dst, src []byte) int  // shadows builtin copy

// Good -- use descriptive names
var errMsg string
var count int
func copyBytes(dst, src []byte) int
```

2. The full list of predeclared identifiers to avoid: `any`, `append`, `bool`, `byte`, `cap`, `clear`, `close`, `comparable`, `complex`, `complex64`, `complex128`, `copy`, `delete`, `error`, `false`, `float32`, `float64`, `imag`, `int`, `int8`, `int16`, `int32`, `int64`, `iota`, `len`, `make`, `max`, `min`, `new`, `nil`, `panic`, `print`, `println`, `real`, `recover`, `rune`, `string`, `true`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`.

**Enforcement:** `golangci-lint` `predeclared` flags identifiers that shadow predeclared names.

### 2.23 — Avoid import aliases unless they resolve a conflict or mismatch.

**Reasoning, step by step:**
1. Avoid import aliases unless the package name conflicts with another import or doesn't match the last element of the import path:

```go
// Good -- no alias needed
import (
    "net/http"
    "github.com/stretchr/testify/assert"
)

// Good -- alias resolves conflict
import (
    "crypto/rand"
    mathrand "math/rand"
)

// Good -- package name doesn't match import path
import (
    yamlv3 "gopkg.in/yaml.v3"
)

// Bad -- gratuitous alias
import (
    h "net/http"
)
```

2. When an alias is needed, use a name that follows Go package naming conventions: short, lowercase, no underscores.

**Enforcement:** `golangci-lint` `importas` (enforces alias consistency) and review for gratuitous aliases.

### 2.24 — End format-string functions with `f`.

**Reasoning, step by step:**
1. When defining functions that accept format strings (like `fmt.Sprintf`), end the name with `f`. This enables `go vet` to check format string correctness at compile time:

```go
// Good -- go vet can verify format strings
func Wrapf(format string, args ...any) error {
    return fmt.Errorf(format, args...)
}

func Debugf(format string, args ...any) {
    logger.Debug(fmt.Sprintf(format, args...))
}

// Bad -- go vet cannot check format strings
func Wrap(format string, args ...any) error {
    return fmt.Errorf(format, args...)
}
```

2. When storing format strings for later use, declare them as `const` so `go vet` can still analyze them:

```go
const msgFormat = "processing %d items in batch %q"

log.Printf(msgFormat, count, batchID)
```

**Enforcement:** `go vet` `printf` analyzer (checks format strings on `f`-suffixed and `const` formats).

## Cross-references

- [03-error-handling.md](./03-error-handling.md) — sentinel error and error-type naming (2.16) in the context of error wrapping and handling.
- [06-testing.md](./06-testing.md) — test function naming (2.20), test helper packages (2.9), and test-double naming (2.10, 2.11).
- [12-variables-and-declarations.md](./12-variables-and-declarations.md) — variable declaration, grouped `var` blocks, units (2.19), and shadowing (2.8).
