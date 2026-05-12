# 02 - Naming Conventions

## Name Length is Proportional to Scope

The length of a name should be **proportional to the size of its scope** and **inversely proportional to the number of times it is used**. (Google rule.)

| Scope | Convention |
|-------|-----------|
| 1-7 lines (tight loop, short function) | Single-word or single-letter names: `i`, `n`, `r`, `buf` |
| 8-15 lines (medium function) | Short but descriptive: `count`, `user`, `resp` |
| 15-25 lines | More descriptive: `userCount`, `retryDeadline` |
| 25+ lines or package-level | Multiple words as needed: `orderedPartnerAccounts` |

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

This rule **supersedes** any blanket "spell it out" preference. `resp` is appropriate in a 10-line HTTP handler; `response` is appropriate at package scope. Do not drop letters to save typing (`Sbx` is not an acceptable shorthand for `Sandbox`).

## Exported vs Unexported

- Uppercase first letter = exported (public API).
- Lowercase first letter = unexported (internal).
- No `public`, `private`, or `protected` keywords.

```go
type Client struct { ... }     // exported -- part of the package API
type clientConfig struct { ... } // unexported -- internal implementation detail
```

## Casing Rules

| Kind | Convention | Good | Bad |
|------|-----------|------|-----|
| Exported types | MixedCaps | `HTTPClient`, `TokenStore` | `Http_Client`, `tokenStore` |
| Unexported types | mixedCaps | `requestBuilder`, `authState` | `request_builder`, `AuthState` |
| Exported functions | MixedCaps | `NewClient`, `ParseToken` | `New_Client`, `parseToken` |
| Unexported functions | mixedCaps | `validateConfig`, `buildURL` | `validate_config`, `ValidateConfig` |
| Constants | MixedCaps (not SCREAMING) | `MaxRetries`, `DefaultTimeout` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Local variables | mixedCaps, short | `resp`, `ctx`, `buf` | `the_response`, `myContext` |

Never use underscores in Go names -- except in test function names (`TestParse_EmptyInput`).

## Package Names

- Short, lowercase letters and numbers only. No `camelCase`, no underscores.
- Multi-word package names must remain unbroken and all lowercase (e.g., `tabwriter`, not `tabWriter` or `tab_writer`).
- The package name is part of every call site — it must compose well.
- Avoid names likely to be shadowed by common local variable names (prefer `usercount` over `count`).
- Never stutter: if the package is `token`, the type is `Token`, not `TokenToken`.
- Avoid uninformative names: `util`, `utility`, `common`, `helper`, `model`, `testhelper`.

| Good | Bad | Why Bad |
|------|-----|---------|
| `transport` | `transportpkg` | Redundant suffix |
| `auth` | `authentication` | Too long |
| `pipeline` | `utils` | Describes nothing |
| `token` | `common` | Junk drawer |
| `store` | `shared` | Says where, not what |
| `config` | `base` | Implies inheritance Go doesn't have |

**Underscore exceptions for package names:**
- `_test` suffix for black-box test packages (`linkedlist_test`).
- Underscores within integration test package names (`linked_list_service_test`).
- `_test` suffix for package-level documentation example files.

## Receiver Names

Receiver names must be:
- **Short** — one or two letters.
- **An abbreviation of the type name** — derive from the type.
- **Consistent** — every method on a type uses the same receiver name.
- **Never `this`, `self`, or `me`**.
- **Never `_`** — omit the receiver if unused: `func (Validator) IsValid() bool`.

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

## Constant Names — Role, Not Value

Name constants by their **role**, not their value. A constant whose name describes only its value is unnecessary.

```go
// Good
const MaxPacketSize = 512
const DefaultTimeout = 30 * time.Second

// Bad
const Twelve = 12
const FiveHundred = 500
```

If you catch yourself writing `const X = "X"`, the constant serves no purpose — inline the value or find a role-based name.

## Avoid Name Repetition

The package name is already at the call site. Don't repeat it in type or function names:

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

This applies to fields, methods, and constants too:

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

### Don't repeat types, receivers, or parameters in function/method names

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

### Don't repeat concepts from the surrounding context

In package `sqldb`, use `Connection`, not `DBConnection`. On `type Project struct`, use `Name()`, not `ProjectName()`. In package `ads/targeting`, use `id` for a local variable — not `adsTargetingID`.

## Variable Shadowing

Never shadow package-level names, imported packages, or outer scope variables. Shadowing silently hides bugs:

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

## Test Helper Packages

Name test helper packages by appending `test` to the production package name: `auth` → `authtest`, `creditcard` → `creditcardtest`. Mark the Bazel `go_library` (or equivalent) as `testonly`.

**When one type is doubled**, use concise names:
- `authtest.Stub` is strictly preferable to `authtest.StubService`.
- `authtest.Fake` over `authtest.FakeService`.

**When multiple behaviors matter**, name by behavior: `authtest.AlwaysAllow`, `authtest.AlwaysDeny`, `creditcardtest.AlwaysCharges`, `creditcardtest.AlwaysDeclines`.

**When multiple types are doubled**, be explicit: `StubService`, `StubStoredValue`.

## Local Variables in Tests

When a test variable refers to a test double, **prefix the variable name** to distinguish it from a production type:

```go
// Good
spyCC := creditcardtest.NewSpy()
stubStore := storetest.NewStub()

// Bad
cc := creditcardtest.NewSpy()  // looks like a production credit card
```

## Shadowing: Stomping vs. True Shadowing

- **Stomping** — reusing an existing name with `:=` in the same scope where at least one variable on the left is new — is fine when the old value is no longer needed.
- **Shadowing** — declaring a new variable in an inner scope with `:=` that shadows an outer variable — silently hides the outer value from subsequent code.
- To avoid shadowing bugs, use `=` in inner scopes with pre-declared variables, or choose a new name for clarity.
- **Never shadow** imported packages (`url`, `context`, `log`) or Go built-ins (`len`, `copy`, `error`).

## Test Double Naming

Name test doubles by their behavior, not by the word "mock":

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

Reserve "mock" for generated mocks from frameworks (`gomock`, `mockery`).

## Interface Naming

- **Single-method interfaces**: use the `-er` suffix.

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }
type Handler interface { ServeHTTP(ResponseWriter, *Request) }
```

- **Multi-method interfaces**: use a descriptive noun. No `-er` suffix.

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

## Getters and Setters

- Never use `Get`/`get` prefix on getters, **unless the underlying concept uses "get"** (e.g., HTTP `GET`).
- Use `Set` prefix on setters.
- For expensive work, use `Compute` or `Fetch` rather than `Get`.

```go
// Good
func (u *User) Name() string          { return u.name }
func (u *User) SetName(n string)      { u.name = n }
func (r *Report) Compute() Statistics { ... }  // expensive, not a simple getter
func (c *Client) FetchUser(ctx context.Context, id string) (*User, error) { ... } // remote call

// Bad
func (u *User) GetName() string { return u.name }
```

## Functions: Nouns for Values, Verbs for Actions

- Functions that return something get **noun-like** names: `JobName()`, `Count()`.
- Functions that do something get **verb-like** names: `WriteDetail()`, `Close()`.
- When identical functions differ only by type, append the type: `ParseInt`, `ParseInt64`, `AppendInt`, `AppendInt64`. If one is the "primary" version, omit the type from it: `Marshal()` vs `MarshalText()`.

## Initialisms and Acronyms

Words that are initialisms or acronyms use the **same case throughout** — all caps or all lowercase. Never mixed.

**Standard uppercase initialisms** (`XML`, `API`, `ID`, `DB`, `URL`, `HTTP`, `UUID`, `JSON`):

| Exported | Unexported |
|----------|-----------|
| `XMLAPI`, `ID`, `HTTPClient` | `xmlAPI`, `id`, `httpClient` |
| `URLParser`, `UUIDGenerator` | `urlParser`, `uuidGenerator` |

For unexported names, the **first** initialism is lowercased; subsequent ones stay uppercase.

**Lowercase-prefixed initialisms** (`iOS`, `gRPC`, `DDoS`): appear as in standard prose unless exportedness forces a case change.

| Identifier | Exported | Unexported |
|-----------|----------|-----------|
| iOS | `IOS` | `iOS` |
| gRPC | `GRPC` | `gRPC` |
| DDoS | `DDoS` | `ddos` |

**Special case:** `Txn` exported is `Txn`, not `TXN`.

| Good | Bad |
|------|-----|
| `URL` | `Url` |
| `userID` | `userId`, `userId` |
| `parseURL` | `parseUrl` |
| `HTTPClient` (exported) | `HttpClient` |
| `httpClient` (unexported) | `HTTPClient` (unexported) |

## Error Variables and Types

- **Sentinel errors**: `Err` prefix, package-level `var`.

```go
var ErrNotFound = errors.New("not found")
var ErrTimeout = errors.New("timeout")
var ErrUnauthorized = errors.New("unauthorized")
```

- **Error types**: `Error` suffix.

```go
type ValidationError struct { Field string; Message string }
type TransportError struct { StatusCode int; cause error }
type ConfigError struct { Key string; Reason string }
```

## Constructor Naming

Always use `New` prefix.

| Pattern | When |
|---------|------|
| `NewClient(...)` | Primary constructor |
| `NewClientFromConfig(cfg Config)` | Constructor from a specific source |
| `Open(path string)` | Resources that need closing (files, connections) |
| `Parse(s string)` | Deserialization / parsing |

- Return `(*T, error)` when construction can fail.
- Return `*T` only when construction is infallible.

```go
// Can fail
func NewClient(baseURL string, opts ...Option) (*Client, error) { ... }

// Cannot fail
func NewBuffer(size int) *Buffer { ... }
```

## Abbreviations

Abbreviations are governed by the [scope rule](#name-length-is-proportional-to-scope). Short scope permits short names; large scope demands clarity.

Universal idioms (acceptable at any scope):

| Abbreviation | Full Form |
|-------------|-----------|
| `ctx` | `context.Context` |
| `err` | `error` |
| `r` | `io.Reader` or `*http.Request` |
| `w` | `io.Writer` or `http.ResponseWriter` |
| `i`, `j`, `k` | Integer loop indices |
| `x`, `y`, `z` | Coordinates |

Domain abbreviations (acceptable in tight scope, spell out at package level):

| Abbreviation | Full Form |
|-------------|-----------|
| `req`, `resp` | request, response |
| `buf` | buffer |
| `cfg` | config |
| `n` | count / length in a short loop |

Do not drop letters to save typing in exported names (`Sbx` for `Sandbox`, `Usr` for `User`). Omit type information from names — prefer `users` over `userSlice`, `count` over `userCount` unless disambiguation is needed.

## Add Units to Names

Include units when a value has one. Put units and qualifiers last, sorted by descending significance: `timeout_ms_max` not `max_timeout_ms`. This groups related variables together when sorted alphabetically.

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

```go
// Bad -- no units at all
const defaultTimeout = 30000    // milliseconds? seconds? minutes?
const maxBufferSize = 10000000  // bytes? KB? records?
```

For `time.Duration`, the type carries the unit -- no suffix needed:

```go
const defaultTimeout = 30 * time.Second  // clear from the type
```

## Test Naming

Format: `TestFunctionName_Scenario`. Underscores between the function name and scenario are expected — they are one of Go's three permitted underscore use cases.

```go
func TestParse_ValidInput(t *testing.T) { ... }
func TestParse_EmptyString(t *testing.T) { ... }
func TestNewClient_MissingURL(t *testing.T) { ... }
```

For subtests, prefer `t.Run(description, ...)` over encoding scenarios in the top-level function name:

```go
func TestParse(t *testing.T) {
    t.Run("valid input", func(t *testing.T) { ... })
    t.Run("empty string", func(t *testing.T) { ... })
}
```

Either pattern is valid. Choose one per package for consistency.

## No Underscores in Go Names

Names in Go must not contain underscores. (Google canonical rule — overrides any earlier guidance that prescribed underscore prefixes.)

Exceptions:
- `Test`, `Benchmark`, `Example`, `Fuzz` function names in `*_test.go` files (e.g., `TestParse_EmptyInput`).
- Package names imported only by generated code.
- Low-level libraries interoperating with the OS or cgo (very rare).

Distinguish package-level values from locals by:
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

## Avoid Using Built-In Names

Never use Go's predeclared identifiers as variable, function, or type names. They compile but shadow the builtin, creating confusing, hard-to-grep code:

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

The full list of predeclared identifiers to avoid: `any`, `append`, `bool`, `byte`, `cap`, `clear`, `close`, `comparable`, `complex`, `complex64`, `complex128`, `copy`, `delete`, `error`, `false`, `float32`, `float64`, `imag`, `int`, `int8`, `int16`, `int32`, `int64`, `iota`, `len`, `make`, `max`, `min`, `new`, `nil`, `panic`, `print`, `println`, `real`, `recover`, `rune`, `string`, `true`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`.

## Import Aliasing

Avoid import aliases unless the package name conflicts with another import or doesn't match the last element of the import path:

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

When an alias is needed, use a name that follows Go package naming conventions: short, lowercase, no underscores.

## Printf-style Function Naming

When defining functions that accept format strings (like `fmt.Sprintf`), end the name with `f`. This enables `go vet` to check format string correctness at compile time:

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

When storing format strings for later use, declare them as `const` so `go vet` can still analyze them:

```go
const msgFormat = "processing %d items in batch %q"

log.Printf(msgFormat, count, batchID)
```
