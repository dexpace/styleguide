# 11 — Documentation

Documentation is part of the API. A Go symbol's contract lives in its doc comment, and `go doc` surfaces it to every caller who never opens the source. This chapter makes that contract complete: a GoDoc comment on every exported symbol, written as full sentences starting with the name; package comments, testable examples, and explicit statements of the things a signature cannot show — concurrency safety, cleanup duty, and the errors a function returns. Comments explain why; the code shows what. Dead code and untracked TODOs do not survive review.

## What good looks like

```go
// Package hotel provides a client for the hotel search service.
//
// The primary entry point is Client, which executes search and booking
// operations against the remote API.
package hotel

// Client executes API operations against the hotel service.
//
// Client is safe for concurrent use by multiple goroutines.
type Client struct {
    timeout time.Duration // maximum wait for a single request
}

// NewClient creates a new Client. The caller must call Close when finished
// to release HTTP connections and stop background refresh goroutines.
func NewClient(opts ...Option) (*Client, error) {
    // ...
}

// Search returns hotels matching the given [SearchParams].
//
// Returns [ErrNotFound] if no hotels match.
// Returns [BadRequestError] if the params are malformed.
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
    // TODO(omar): Add pagination. Tracked in HOTEL-412.
    // Use %v (not %w) here: callers must not depend on the transport error type.
    resp, err := c.do(ctx, params)
    if err != nil {
        return nil, fmt.Errorf("search hotels: %v", err)
    }
    return resp.Hotels, nil
}

func ExampleClient_Search() {
    client, _ := NewClient(WithTimeout(30 * time.Second))
    hotels, err := client.Search(context.Background(), SearchParams{City: "Seattle"})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(hotels[0].Name)
    // Output: Grand Hotel
}
```

The package comment sits flush against the `package` clause (11.3); every exported symbol carries a name-led doc comment in full sentences (11.1, 11.2); `Client` states its concurrency guarantee (11.13) and `NewClient` its cleanup duty (11.14); `Search` documents its returned errors with `[Type]` links (11.12, 11.15) and is exercised by a testable `Example` (11.5); the struct-field comment is an allowed fragment (11.2); the TODO has an owner and a ticket (11.9); the `%v` deviation is signal-boosted (11.6) and explains why, not what (11.7); the wrapped error message is a lowercase fragment (11.16).

## Rules

### 11.1 — Put a GoDoc comment on every exported symbol, starting with its name.

**Reasoning, step by step:**
1. Every exported function, type, method, constant, and variable must have a GoDoc comment. It is the only contract a caller reading `go doc` ever sees.
2. Start the comment with the symbol name so the rendered documentation reads as a sentence about that symbol. An article (`A`, `An`, `The`) may precede the name.

```go
// Client executes API operations against the hotel service.
type Client struct { ... }

// A Request represents a search query sent to the hotel service.
type Request struct { ... }

// Search returns hotels matching the given search parameters.
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
```

3. Unexported symbols with non-obvious behavior should follow the same convention, starting with the symbol name. This eases future exporting.

**Enforcement:** golint; review.

### 11.2 — Write doc comments as complete, capitalized, punctuated sentences.

**Reasoning, step by step:**
1. Doc comments must be complete sentences — capitalized and ending with a period. They are public prose, so they read as prose.

```go
// ParseResponse reads the HTTP response body and decodes it into the target type.
```

2. End-of-line struct field comments may be sentence fragments, assuming the field name is the subject:

```go
type Config struct {
    Timeout time.Duration // maximum wait for a single request
    Retries int           // number of retry attempts
}
```

3. Sentence fragments elsewhere — short inline comments explaining a single line — have no punctuation requirements.
4. Exception: a doc comment may begin with an uncapitalized identifier name when the rest remains clear, e.g. `// io.Reader wraps...`.

**Enforcement:** golint; review.

### 11.3 — Give every package a package comment, flush against the clause.

**Reasoning, step by step:**
1. Every package must have a package comment. Place it in `doc.go` for packages with multiple files. Only one package comment per package — keep it in a single file.
2. The package comment must appear immediately above the `package` clause with no blank line between them.

```go
// Package transport provides HTTP transport implementations for the SDK.
//
// The primary interface is Transport, which abstracts over the actual HTTP
// client implementation.
package transport
```

3. For `main` packages, the comment describes the binary. Acceptable forms:

```go
// Binary seedgenerator creates a new seed for the random source.
// Command seedgenerator creates a new seed for the random source.
// The seedgenerator command creates a new seed for the random source.
```

4. Capitalize the binary name even when its command-line form is lowercase.
5. Multi-line `/* */` package comments are acceptable, especially when the comment contains content the reader is expected to copy — config snippets, URLs.
6. Comments intended only for package maintainers — not surfaced in `go doc` — go after the import block, not in the package comment.

**Enforcement:** golint; review.

### 11.4 — Name result parameters only when they make the signature clearer.

**Reasoning, step by step:**
1. Use named result parameters when they make the signature clearer: two or more results of the same type (disambiguating which is which); a result the caller must act on, like `cancel func()`; when named results materially improve the godoc; or when the result must be set inside a deferred closure.
2. Avoid named results when they only add redundant repetition (`func (n *Node) Parent1() (node *Node)` — just use `*Node`).
3. Avoid named results when they enable naked returns in non-trivial functions; naked returns are acceptable only in very short functions.
4. Avoid named results that exist only to save declaring a local variable.

**Enforcement:** golangci-lint; review.

### 11.5 — Write testable Example functions for public APIs.

**Reasoning, step by step:**
1. Write testable examples for all public functions. They run as tests and appear in GoDoc, so the documentation cannot drift from working code.

```go
func ExampleClient_Search() {
    client := NewClient(WithTimeout(30 * time.Second))
    hotels, err := client.Search(context.Background(), SearchParams{City: "Seattle"})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(hotels[0].Name)
    // Output: Grand Hotel
}
```

2. Examples must compile and pass. They are tests, not decorations.

**Enforcement:** go vet; review.

### 11.6 — Signal-boost code that subtly deviates from a common pattern.

**Reasoning, step by step:**
1. When code subtly deviates from a common pattern, add a comment to boost the signal so readers do not miss the distinction.

```go
// Good -- unusual pattern flagged for the reader
if err == nil { // if NO error
    useResult(result)
}

// Good -- explains why we intentionally break convention
// Use %v (not %w) here: callers must not depend on the internal gRPC error type.
return fmt.Errorf("fetch config: %v", grpcErr)
```

2. This matters most around: `err == nil` checks, when readers expect `err != nil`; assignments that look like comparisons (`=` vs `==`); negated conditions (`!`) that are easy to misread; and `%v` used where `%w` is expected.
3. If a single-character difference is structurally important, consider refactoring instead of relying on the comment.

**Enforcement:** review.

### 11.7 — Explain why in inline comments, not what.

**Reasoning, step by step:**
1. Inline comments explain why, not what. The code already shows what.
2. If you need a "what" comment, simplify the code instead.

```go
// Good -- explains why
// Retry because the token endpoint rate-limits aggressively.

// Bad -- restates the code
// Increment the counter by one.
```

**Enforcement:** review.

### 11.8 — Give every TODO an owner and a tracking reference.

**Reasoning, step by step:**
1. Every TODO must have an owner and a tracking reference. Untracked TODOs rot.

```go
// TODO(username): Description. Tracked in JIRA-123.
```

**Enforcement:** review.

### 11.9 — Put a license header on every source file, enforced in CI.

**Reasoning, step by step:**
1. Every source file must have the license header.
2. Automate enforcement in CI — do not rely on developers remembering.

**Enforcement:** golangci-lint; review.

### 11.10 — Delete dead code; never comment it out.

**Reasoning, step by step:**
1. Delete code instead of commenting it out. Version control remembers.
2. Commented-out code is dead weight that misleads readers.

**Enforcement:** golangci-lint; review.

### 11.11 — Link to other symbols with GoDoc `[Type]` syntax.

**Reasoning, step by step:**
1. Use `[Type]` syntax in GoDoc comments to link to other symbols. The Go documentation server renders these as clickable links.

```go
// Search returns hotels matching the given [SearchParams].
// It uses the underlying [http.Client] for HTTP calls.
// Returns [ErrNotFound] if no hotels match.
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
```

2. Use `[pkg.Type]` for cross-package references: `[context.Context]`, `[fmt.Stringer]`.

**Enforcement:** review.

### 11.12 — Document the concurrency safety of every exported type.

**Reasoning, step by step:**
1. Every exported type must document its concurrency guarantees.

```go
// Client is safe for concurrent use by multiple goroutines.
type Client struct { ... }

// TokenCache is NOT safe for concurrent use. Callers must synchronize access.
type TokenCache struct { ... }
```

2. If a type does not document this, callers must assume it is NOT safe. When in doubt, state it explicitly.

**Enforcement:** review.

### 11.13 — Document cleanup requirements on the constructor.

**Reasoning, step by step:**
1. If a type holds resources that must be released, document it on the constructor so the caller knows the obligation at the call site.

```go
// NewClient creates a new Client. The caller must call Close when finished
// to release HTTP connections and stop background refresh goroutines.
func NewClient(opts ...Option) (*Client, error) {
```

**Enforcement:** review.

### 11.14 — Document the errors a function may return.

**Reasoning, step by step:**
1. Document which errors a function may return, especially sentinel errors and typed errors. A returned error is part of the contract; the signature alone does not name it.

```go
// Translate converts an EGAID to an EG User ID.
//
// Returns [NoEGUserFoundError] if no mapping exists.
// Returns [BadRequestError] if the EGAID format is invalid.
// Returns [InvalidPrincipalJwtError] if the JWT is expired or malformed.
func (t *Translator) Translate(ctx context.Context, egaid string, jwt string) (string, error) {
```

**Enforcement:** review.

### 11.15 — Write error messages as lowercase, unpunctuated fragments.

**Reasoning, step by step:**
1. Error messages are lowercase, carry no terminal punctuation, and describe what failed.

| Good | Bad |
|------|-----|
| `"authenticate with token endpoint"` | `"Failed to authenticate with the token endpoint!"` |
| `"parse response body"` | `"Error: could not parse the response body."` |
| `"open config file"` | `"Unable to open configuration file"` |

2. Error messages are fragments that compose through wrapping. They are not user-facing sentences.

**Enforcement:** golangci-lint; review.

## Cross-references

- Error wrapping with `%w` and lowercase message style: [chapter 03](./03-error-handling.md).
- Concurrency guarantees and goroutine lifecycles: [chapter 04](./04-concurrency.md).
- Constructor and API surface conventions: [chapter 05](./05-api-design.md).
- `Example` functions as tests: [chapter 06](./06-testing.md).
- Package layout and `doc.go` placement: [chapter 07](./07-package-organization.md).
- `Close` and resource release obligations: [chapter 10](./10-resource-management.md).
