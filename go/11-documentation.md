# 11 - Documentation

## GoDoc on All Exported Symbols

Every exported function, type, method, constant, and variable must have a GoDoc comment. Start with the symbol name. An article (`A`, `An`, `The`) may precede the name:

```go
// Client executes API operations against the hotel service.
type Client struct { ... }

// A Request represents a search query sent to the hotel service.
type Request struct { ... }

// Search returns hotels matching the given search parameters.
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
```

Unexported symbols with non-obvious behavior should follow the same convention (start with the symbol name). This eases future exporting.

## Comment Sentences

Doc comments must be **complete sentences** — capitalized, punctuated. End-of-line struct field comments may be sentence fragments assuming the field name is the subject:

```go
type Config struct {
    Timeout time.Duration // maximum wait for a single request
    Retries int           // number of retry attempts
}
```

Sentence fragments elsewhere (e.g., short inline comments explaining a line) have no punctuation requirements.

Exception: a doc comment may begin with an uncapitalized identifier name if the rest remains clear. E.g., `// io.Reader wraps...`.

## Package Comments

Every package must have a package comment. Place in `doc.go` for packages with multiple files. The package comment must appear immediately above the `package` clause with **no blank line** between them. Only one package comment per package — place it in a single file.

```go
// Package transport provides HTTP transport implementations for the SDK.
//
// The primary interface is Transport, which abstracts over the actual HTTP
// client implementation.
package transport
```

For `main` packages, the comment describes the binary. Acceptable forms:

```go
// Binary seedgenerator creates a new seed for the random source.
// Command seedgenerator creates a new seed for the random source.
// The seedgenerator command creates a new seed for the random source.
```

Capitalize the binary name even when its command-line form is lowercase.

Multi-line `/* */` package comments are acceptable, especially when the comment contains content the reader is expected to copy (config snippets, URLs).

Comments intended only for package maintainers — not surfaced in `go doc` — go **after** the import block, not in the package comment.

## Named Result Parameters

Use named result parameters when they make the signature clearer:
- Two or more parameters of the **same type** (disambiguates which is which).
- A result the caller must act on, like `cancel func()`.
- When named results materially improve the godoc.
- When the result must be set inside a deferred closure.

Avoid named results when they:
- Add redundant repetition (`func (n *Node) Parent1() (node *Node)` — just use `*Node`).
- Enable naked returns in non-trivial functions (acceptable only in very short functions).
- Exist only to save declaring a local variable.

## Example Functions

Write testable examples for all public functions. These run as tests and appear in GoDoc:

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

Examples must compile and pass. They are tests, not decorations.

## Signal Boosting

When code subtly deviates from a common pattern, add a comment to "boost" the signal so readers don't miss the distinction:

```go
// Good -- unusual pattern flagged for the reader
if err == nil { // if NO error
    useResult(result)
}

// Good -- explains why we intentionally break convention
// Use %v (not %w) here: callers must not depend on the internal gRPC error type.
return fmt.Errorf("fetch config: %v", grpcErr)
```

This is especially important around:
- `err == nil` checks (when readers expect `err != nil`).
- Assignments that look like comparisons (`=` vs `==`).
- Negated conditions (`!`) that are easy to misread.
- `%v` used where `%w` is expected.

If a single-character difference is structurally important, consider refactoring instead.

## Inline Comments

Explain WHY, not WHAT. The code shows what. If you need a "what" comment, simplify the code instead.

```go
// Good -- explains why
// Retry because the token endpoint rate-limits aggressively.

// Bad -- restates the code
// Increment the counter by one.
```

## Comments Are Sentences

Start with a capital letter, end with a period.

```go
// ParseResponse reads the HTTP response body and decodes it into the target type.
```

## TODO Format

```go
// TODO(username): Description. Tracked in JIRA-123.
```

Every TODO must have an owner and a tracking reference. Untracked TODOs rot.

## License Headers

Every source file must have the license header. Automate enforcement in CI -- don't rely on developers remembering.

## Don't Comment Out Code

Delete it. Version control remembers. Commented-out code is dead weight that misleads readers.

## GoDoc Link Syntax

Use `[Type]` syntax in GoDoc comments to link to other symbols. The Go documentation server renders these as clickable links:

```go
// Search returns hotels matching the given [SearchParams].
// It uses the underlying [http.Client] for HTTP calls.
// Returns [ErrNotFound] if no hotels match.
func (c *Client) Search(ctx context.Context, params SearchParams) ([]Hotel, error) {
```

Use `[pkg.Type]` for cross-package references: `[context.Context]`, `[fmt.Stringer]`.

## Document Concurrency Safety

Every exported type must document its concurrency guarantees:

```go
// Client is safe for concurrent use by multiple goroutines.
type Client struct { ... }

// TokenCache is NOT safe for concurrent use. Callers must synchronize access.
type TokenCache struct { ... }
```

If not documented, assume NOT safe. When in doubt, state it explicitly.

## Document Cleanup Requirements

If a type holds resources that must be released, document it on the constructor:

```go
// NewClient creates a new Client. The caller must call Close when finished
// to release HTTP connections and stop background refresh goroutines.
func NewClient(opts ...Option) (*Client, error) {
```

## Document Returned Errors

Document which errors a function may return, especially sentinel errors and typed errors:

```go
// Translate converts an EGAID to an EG User ID.
//
// Returns [NoEGUserFoundError] if no mapping exists.
// Returns [BadRequestError] if the EGAID format is invalid.
// Returns [InvalidPrincipalJwtError] if the JWT is expired or malformed.
func (t *Translator) Translate(ctx context.Context, egaid string, jwt string) (string, error) {
```

## Error Message Style

Lowercase, no punctuation, describe what failed:

| Good | Bad |
|------|-----|
| `"authenticate with token endpoint"` | `"Failed to authenticate with the token endpoint!"` |
| `"parse response body"` | `"Error: could not parse the response body."` |
| `"open config file"` | `"Unable to open configuration file"` |

Error messages are fragments that compose through wrapping. They are not user-facing sentences.
