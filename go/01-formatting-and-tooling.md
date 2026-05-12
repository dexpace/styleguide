# 01 - Formatting & Tooling

## gofmt

- Run `gofmt` on save. Enforce in CI. Non-negotiable.
- Use only `gofmt` or `goimports` (which is `gofmt` + import management).
- Never override tab indentation.

```bash
# CI check -- fails if any file is not formatted
test -z "$(gofmt -l .)"
```

## golangci-lint

Use `golangci-lint` with the curated linter set below. One tool, one config.

### Curated Linters

| Linter | Why Enabled |
|--------|-------------|
| `errcheck` | Every error must be handled. Most important linter. |
| `govet` | Catches printf format mismatches, struct tag errors, unreachable code. |
| `staticcheck` | Gold standard of Go static analysis -- deprecated APIs, inefficient patterns, correctness bugs. |
| `unused` | Dead code is a liability. Remove it. |
| `gosec` | Security: hardcoded credentials, weak crypto, SQL injection. |
| `gocritic` | Unnecessary type conversions, nil checks, assignment simplification. |
| `exhaustive` | Switch on enum types must be exhaustive. Missing cases are bugs. |

### Linters to Disable

| Linter | Why Disabled |
|--------|-------------|
| `gocyclo` | Arbitrary thresholds. A function with 15 error checks is correct, not complex. |
| `wsl` | Whitespace opinions that conflict with Go conventions. |
| `funlen` | Line counts are a poor proxy for complexity. |
| `lll` | Hard line-length limits don't work with Go's verbose error handling. |

### `.golangci.yml`

```yaml
run:
  timeout: 5m

linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - gosec
    - gocritic
    - exhaustive
  disable:
    - gocyclo
    - wsl
    - funlen
    - lll

linters-settings:
  errcheck:
    check-type-assertions: true
    check-blank: true
  exhaustive:
    default-signifies-exhaustive: true
  gocritic:
    enabled-tags:
      - diagnostic
      - performance
      - style

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0
```

## go vet

- Run `go vet ./...` in CI alongside `golangci-lint`.
- Always run it separately to ensure standard toolchain checks pass independently.

## Import Ordering

Use `gci` for import grouping. Three groups, separated by blank lines:

```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/stretchr/testify/assert"
    "golang.org/x/sync/errgroup"

    "github.com/yourorg/yourproject/internal/auth"
    "github.com/yourorg/yourproject/transport"
)
```

1. Standard library
2. External dependencies
3. Internal packages

No exceptions.

## Function Size

- **Hard limit: 70 lines per function.** No exceptions.
- Aim for 20-40 lines. If a function exceeds 40 lines, it is probably doing too much.
- Every function does ONE thing. If you need the word "and" to describe it, split it.
- Functions over 20 lines must have blank lines separating logical sections.
- Long functions hide control flow and make assertions harder to place.

## Code Density & Whitespace

`gofmt` handles indentation and braces. Whitespace between logical blocks is a readability concern beyond formatting. Cramped code is unreadable code. Whitespace is free. Use it.

- Use blank lines to separate logical sections within functions.
- Blank line before `return` (unless the function is trivial).
- Blank line between `if err != nil` error handling blocks.
- Blank line between setup and assertions in table-driven tests.
- No more than 3-4 consecutive lines of code without a blank line for breathing room.
- Never cram multiple operations onto one line for brevity.

### Bad -- cramped, no breathing room

```go
func ProcessOrder(ctx context.Context, req OrderRequest) (*Order, error) {
	validated, err := validate(req)
	if err != nil {
		return nil, fmt.Errorf("validation: %w", err)
	}
	order, err := toOrder(validated)
	if err != nil {
		return nil, fmt.Errorf("mapping: %w", err)
	}
	saved, err := repo.Save(ctx, order)
	if err != nil {
		return nil, fmt.Errorf("save: %w", err)
	}
	metrics.RecordOrder(saved)
	notify(saved)
	return saved, nil
}
```

### Good -- logical sections breathe

```go
func ProcessOrder(ctx context.Context, req OrderRequest) (*Order, error) {
	validated, err := validate(req)
	if err != nil {
		return nil, fmt.Errorf("validation: %w", err)
	}

	order, err := toOrder(validated)
	if err != nil {
		return nil, fmt.Errorf("mapping: %w", err)
	}

	saved, err := repo.Save(ctx, order)
	if err != nil {
		return nil, fmt.Errorf("save: %w", err)
	}

	metrics.RecordOrder(saved)
	notify(saved)

	return saved, nil
}
```

## Line Length

- No fixed line length. (Google rule — overrides earlier 100-char guidance.)
- If a line feels too long, **prefer refactoring over splitting**. Extract helper variables or functions.
- Never split a line:
  - Before an indentation change (e.g., a function declaration or conditional).
  - To make a long string (e.g., a URL) fit into multiple shorter lines.
- If the line is already as short as it can reasonably be, let it stay long.
- Function signatures should remain on a single line where practical. Break only when the reader genuinely benefits:

```go
// Good -- keep on one line when practical
func NewClient(baseURL string, transport http.RoundTripper, opts ...Option) (*Client, error) {

// Acceptable -- break after opening parenthesis only when the single-line version is genuinely unreadable
func NewClient(
    baseURL string,
    transport http.RoundTripper,
    opts ...Option,
) (*Client, error) {
```

### Indentation Confusion

Avoid line breaks that would align wrapped lines with an indented code block. If unavoidable, add a separating blank line between the wrapped signature and the body.

### Conditionals and Loops

`if` statements should not be line-broken — it causes indentation confusion. Instead, extract boolean operands into local variables:

```go
// Good
inTransaction := db.CurrentStatusIs(db.InTransaction)
keysMatch := db.ValuesEqual(db.TransactionKey(), row.Key())
if inTransaction && keysMatch {
    // ...
}

// Bad
if db.CurrentStatusIs(db.InTransaction) &&
    db.ValuesEqual(db.TransactionKey(), row.Key()) {
    // ...
}
```

Do not insert artificial line breaks into `for` statements. Let the line run long or refactor.

`switch`/`case` should remain on single lines. If cases are excessively long, indent all cases uniformly and separate with blank lines.

### Yoda Conditions

Place the variable on the left of equality operators:

```go
// Good
if result == "foo" { ... }

// Bad
if "foo" == result { ... }
```

## Group Similar Declarations

Group related constants, variables, and type declarations together. Separate unrelated groups with blank lines. This makes the relationship between items explicit and helps readers scan quickly.

```go
// Good -- related constants grouped
const (
    StatusActive   = "active"
    StatusInactive = "inactive"
    StatusPending  = "pending"
)

const (
    DefaultTimeout = 30 * time.Second
    DefaultRetries = 3
)

// Bad -- unrelated constants in one block
const (
    StatusActive   = "active"
    DefaultTimeout = 30 * time.Second
    MaxRetries     = 3
    StatusInactive = "inactive"
)
```

Inside functions, group adjacent variable declarations for readability even when unrelated:

```go
// Good -- declarations grouped at the top
var (
    timeout = config.Timeout
    retries = config.Retries
)

// Bad -- scattered throughout
timeout := config.Timeout
// ... 10 lines of other code ...
retries := config.Retries
```

## Function Grouping and Ordering

Within a file, organize functions by logical proximity:

1. **Type definition** first.
2. **Constructor** (`NewX`) immediately after the type.
3. **Exported methods**, sorted by rough call order.
4. **Unexported methods** used by the exported ones.
5. **Standalone helpers** at the end of the file.

```go
// Good -- logical order
type Client struct { ... }

func NewClient(baseURL string, opts ...Option) (*Client, error) { ... }

func (c *Client) Search(ctx context.Context, query Query) ([]Result, error) { ... }
func (c *Client) Get(ctx context.Context, id string) (*Result, error) { ... }
func (c *Client) Close() error { ... }

func (c *Client) doRequest(ctx context.Context, req *http.Request) (*http.Response, error) { ... }
func (c *Client) parseResponse(resp *http.Response) (*Result, error) { ... }
```

Don't alphabetize methods -- call order is more useful than alphabetical order. A reader following `Search` should find the private methods it calls nearby, not scattered by name.

## Reduce Nesting

Handle errors and edge cases first, then proceed with the main logic. Guard clauses and early returns keep the happy path at the lowest indentation level.

```go
// Good -- guard clauses, happy path at left margin
func (c *Client) Fetch(ctx context.Context, url string) (*Response, error) {
    if url == "" {
        return nil, errors.New("url is required")
    }

    resp, err := c.doRequest(ctx, url)
    if err != nil {
        return nil, fmt.Errorf("fetch %s: %w", url, err)
    }

    return resp, nil
}

// Bad -- unnecessary nesting
func (c *Client) Fetch(ctx context.Context, url string) (*Response, error) {
    if url != "" {
        resp, err := c.doRequest(ctx, url)
        if err == nil {
            return resp, nil
        } else {
            return nil, fmt.Errorf("fetch %s: %w", url, err)
        }
    } else {
        return nil, errors.New("url is required")
    }
}
```

In loops, prefer `continue` over deep nesting:

```go
// Good
for _, item := range items {
    if !item.IsValid() {
        continue
    }

    process(item)
}

// Bad
for _, item := range items {
    if item.IsValid() {
        process(item)
    }
}
```

## Unnecessary Else

When a variable is set in both branches of an `if/else`, use a default value with a single-branch override:

```go
// Good -- default value, override in one branch
level := slog.LevelInfo
if config.Debug {
    level = slog.LevelDebug
}

// Bad -- unnecessary else
var level slog.Level
if config.Debug {
    level = slog.LevelDebug
} else {
    level = slog.LevelInfo
}
```

This pattern eliminates `else` blocks, keeps variables initialized, and is easier to scan. Apply the same principle with early returns:

```go
// Good -- early return, no else
func resolveTimeout(custom time.Duration) time.Duration {
    if custom > 0 {
        return custom
    }

    return DefaultTimeout
}

// Bad -- unnecessary else
func resolveTimeout(custom time.Duration) time.Duration {
    if custom > 0 {
        return custom
    } else {
        return DefaultTimeout
    }
}
```

## CI Pipeline

Minimum CI checks for every Go project:

```bash
gofmt -l .                  # formatting
go vet ./...                # standard checks
golangci-lint run ./...     # extended checks
go test -race ./...         # tests with race detector
```

The race detector (`-race`) is mandatory. Data races are correctness bugs.
