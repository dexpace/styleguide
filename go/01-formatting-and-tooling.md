# 01 — Formatting & Tooling

Formatting is mechanical, so delegate it to tools and spend judgment elsewhere. This chapter fixes the toolchain — `gofmt`, `goimports`/`gci`, `go vet`, `golangci-lint` — plus the layout, density, and CI gates every Go project inherits before a line of domain code is written. Everything here is enforced; later chapters cover the decisions a tool cannot make.

## What good looks like

```go
package order

import (
	"context"
	"fmt"

	"github.com/yourorg/yourproject/internal/repo"
)

// ProcessOrder validates, maps, persists, and notifies — one thing per stage,
// each stage given a blank line to breathe.
func ProcessOrder(ctx context.Context, req OrderRequest) (*Order, error) {
	if !req.IsValid() {
		return nil, fmt.Errorf("invalid order request: %s", req.ID)
	}

	order, err := toOrder(req)
	if err != nil {
		return nil, fmt.Errorf("mapping: %w", err)
	}

	saved, err := repo.Save(ctx, order)
	if err != nil {
		return nil, fmt.Errorf("save: %w", err)
	}

	return saved, nil
}
```

This exemplar is `gofmt`-clean with tab indentation untouched (1.1) and three import groups in `gci` order (1.4). It stays under the 70-line function ceiling and does one thing (1.5), separates its logical sections with blank lines and puts a blank line before `return` (1.6), keeps signatures on one line (1.7), leads with a guard clause so the happy path sits flush left (1.10), and avoids an unnecessary `else` by returning early (1.11).

## Rules

### 1.1 — Format with `gofmt`; never override its output.

**Reasoning, step by step:**
1. Formatting carries no correctness weight, so the only cost it has is the time spent arguing about it. `gofmt` ends the argument by deciding for everyone.
2. Run `gofmt` on save and enforce it in CI. It is non-negotiable — unformatted code does not merge.
3. Use only `gofmt` or `goimports` (which is `gofmt` plus import management). Do not introduce a second formatter that could disagree.
4. Never override tab indentation. Tabs are Go's house style and tools depend on them.

```bash
# CI check -- fails if any file is not formatted
test -z "$(gofmt -l .)"
```

**Enforcement:** `gofmt -l .` in CI (fails on any unformatted file).

### 1.2 — Lint with `golangci-lint` using the curated linter set.

**Reasoning, step by step:**
1. One tool, one config: `golangci-lint` runs every linter from a single `.golangci.yml`, so the linter set cannot drift between projects.
2. Enable exactly the linters below. Each earns its place by turning a class of latent bug into a build failure — `errcheck` is the most important, because every error must be handled.

| Linter | Why Enabled |
|--------|-------------|
| `errcheck` | Every error must be handled. Most important linter. |
| `govet` | Catches printf format mismatches, struct tag errors, unreachable code. |
| `staticcheck` | Gold standard of Go static analysis -- deprecated APIs, inefficient patterns, correctness bugs. |
| `unused` | Dead code is a liability. Remove it. |
| `gosec` | Security: hardcoded credentials, weak crypto, SQL injection. |
| `gocritic` | Unnecessary type conversions, nil checks, assignment simplification. |
| `exhaustive` | Switch on enum types must be exhaustive. Missing cases are bugs. |

3. Disable the linters below. Each measures the wrong thing — arbitrary thresholds or line counts that fight Go's verbose, correct error handling.

| Linter | Why Disabled |
|--------|-------------|
| `gocyclo` | Arbitrary thresholds. A function with 15 error checks is correct, not complex. |
| `wsl` | Whitespace opinions that conflict with Go conventions. |
| `funlen` | Line counts are a poor proxy for complexity. |
| `lll` | Hard line-length limits don't work with Go's verbose error handling. |

4. Commit the configuration below verbatim so every project runs the identical gate.

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

**Enforcement:** `golangci-lint run ./...` in CI against the committed `.golangci.yml`.

### 1.3 — Run `go vet` separately in CI.

**Reasoning, step by step:**
1. Run `go vet ./...` in CI alongside `golangci-lint`, not folded into it.
2. Always run it separately to ensure the standard toolchain checks pass independently — vet ships with Go itself and is the baseline that must hold even if the third-party linter is misconfigured.

**Enforcement:** `go vet ./...` in CI.

### 1.4 — Order imports in three groups with `gci`.

**Reasoning, step by step:**
1. Use `gci` for import grouping. Three groups, separated by blank lines, in this fixed order:
   1. Standard library
   2. External dependencies
   3. Internal packages
2. A consistent order means a reader always knows where to look for a given import, and diffs stay minimal. No exceptions.

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

**Enforcement:** `gci` (via `goimports`/`golangci-lint`) in CI.

### 1.5 — Cap functions at 70 lines; each does one thing.

**Reasoning, step by step:**
1. **Hard limit: 70 lines per function.** No exceptions. A function you cannot see at once costs context on every read.
2. Aim for 20-40 lines. If a function exceeds 40 lines, it is probably doing too much.
3. Every function does ONE thing. If you need the word "and" to describe it, split it.
4. Functions over 20 lines must have blank lines separating logical sections.
5. Long functions hide control flow and make assertions harder to place.

**Enforcement:** review; `golangci-lint` (function-length checks are deliberately not the `funlen` linter — enforce the 70-line ceiling in review).

### 1.6 — Use whitespace to separate logical blocks.

**Reasoning, step by step:**
1. `gofmt` handles indentation and braces. Whitespace between logical blocks is a readability concern beyond formatting. Cramped code is unreadable code. Whitespace is free. Use it.
2. Use blank lines to separate logical sections within functions.
3. Put a blank line before `return` (unless the function is trivial).
4. Put a blank line between `if err != nil` error handling blocks.
5. Put a blank line between setup and assertions in table-driven tests.
6. Keep no more than 3-4 consecutive lines of code without a blank line for breathing room.
7. Never cram multiple operations onto one line for brevity.

```go
// Bad -- cramped, no breathing room
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

```go
// Good -- logical sections breathe
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

**Enforcement:** review (whitespace structure is a readability judgment, not a `gofmt` concern).

### 1.7 — No fixed line length; refactor instead of splitting.

**Reasoning, step by step:**
1. No fixed line length. (Google rule — overrides earlier 100-char guidance.)
2. If a line feels too long, **prefer refactoring over splitting**. Extract helper variables or functions.
3. Never split a line:
   - Before an indentation change (e.g., a function declaration or conditional).
   - To make a long string (e.g., a URL) fit into multiple shorter lines.
4. If the line is already as short as it can reasonably be, let it stay long.
5. Function signatures should remain on a single line where practical. Break only when the reader genuinely benefits:

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

6. **Indentation confusion:** avoid line breaks that would align wrapped lines with an indented code block. If unavoidable, add a separating blank line between the wrapped signature and the body.

**Enforcement:** review (the `lll` linter is deliberately disabled per 1.2).

### 1.8 — Don't line-break conditionals and loops; extract operands instead.

**Reasoning, step by step:**
1. `if` statements should not be line-broken — it causes indentation confusion. Instead, extract boolean operands into local variables:

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

2. Do not insert artificial line breaks into `for` statements. Let the line run long or refactor.
3. `switch`/`case` should remain on single lines. If cases are excessively long, indent all cases uniformly and separate with blank lines.
4. **Yoda conditions:** place the variable on the left of equality operators:

```go
// Good
if result == "foo" { ... }

// Bad
if "foo" == result { ... }
```

**Enforcement:** review.

### 1.9 — Group related declarations; separate unrelated groups.

**Reasoning, step by step:**
1. Group related constants, variables, and type declarations together. Separate unrelated groups with blank lines. This makes the relationship between items explicit and helps readers scan quickly.

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

2. Inside functions, group adjacent variable declarations for readability even when unrelated:

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

3. Within a file, organize functions by logical proximity:
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

4. Don't alphabetize methods -- call order is more useful than alphabetical order. A reader following `Search` should find the private methods it calls nearby, not scattered by name.

**Enforcement:** review.

### 1.10 — Reduce nesting with guard clauses and early returns.

**Reasoning, step by step:**
1. Handle errors and edge cases first, then proceed with the main logic. Guard clauses and early returns keep the happy path at the lowest indentation level.

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

2. In loops, prefer `continue` over deep nesting:

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

**Enforcement:** review.

### 1.11 — Eliminate unnecessary `else` with defaults and early returns.

**Reasoning, step by step:**
1. When a variable is set in both branches of an `if/else`, use a default value with a single-branch override:

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

2. This pattern eliminates `else` blocks, keeps variables initialized, and is easier to scan. Apply the same principle with early returns:

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

**Enforcement:** `gocritic` (flags `ifElseChain` and related patterns); review.

### 1.12 — Run the full CI gate on every project, race detector included.

**Reasoning, step by step:**
1. Minimum CI checks for every Go project:

```bash
gofmt -l .                  # formatting
go vet ./...                # standard checks
golangci-lint run ./...     # extended checks
go test -race ./...         # tests with race detector
```

2. The race detector (`-race`) is mandatory. Data races are correctness bugs.

**Enforcement:** the four-command CI pipeline above, with `go test -race ./...` required.

## Cross-references

- Naming conventions the formatter leaves untouched: [02-naming-conventions.md](./02-naming-conventions.md).
- Error handling, wrapping with `%w`, and `errcheck`: [03-error-handling.md](./03-error-handling.md).
- The race detector and data-race correctness: [04-concurrency.md](./04-concurrency.md).
- Variable and declaration grouping: [12-variables-and-declarations.md](./12-variables-and-declarations.md).
