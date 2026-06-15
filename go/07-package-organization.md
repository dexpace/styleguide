# 07 — Package Organization

A package is the unit of compilation, naming, and dependency in Go, and its name appears at every call site, so the directory layout is an API the whole codebase reads. This chapter fixes that layout: one package per directory, short composable names, `internal/` to wall off implementation, a strict dependency direction from `cmd/` toward a dependency-free domain, and disciplined imports. Small focused packages that each describe one thing; no `utils`/`helpers`/`common` junk drawers; one file per major type.

## What good looks like

```text
sdk/
  go.mod
  go.sum
  client.go              # Client type, NewClient, public methods
  client_test.go         # Client tests
  options.go             # Option type, With* functions
  errors.go              # ErrNotFound, APIError, etc.
  doc.go                 # Package documentation
  internal/
    auth/
      oauth.go           # OAuth2 token management
      oauth_test.go
      basic.go           # Basic auth
      basic_test.go
    pipeline/
      step.go            # Pipeline step interface
      logging.go         # Logging middleware step
      retry.go           # Retry step
      retry_test.go
    transport/
      roundtripper.go    # Custom RoundTripper
      roundtripper_test.go
  cmd/
    sdkcli/
      main.go            # CLI tool entry point
  testdata/
    responses/           # Golden files for tests
      hotels.json
      error_400.json
```

Each directory is exactly one package (7.1) named for what it holds — `auth`, `pipeline`, `transport` — short nouns that compose at the call site (7.2). Implementation packages live under `internal/` so only `sdk/` can import them (7.3); the lone binary sits under `cmd/` and there is no `pkg/` because nothing here is for external consumption (7.4). No `utils`/`helpers`/`common` directory exists (7.6), each package is small and single-purpose (7.5), and files split one major type apiece — `client.go`, `options.go`, `errors.go`, `doc.go` (7.8, 7.9) — with the dependency arrows running inward toward a domain that imports nothing internal (7.7).

## Rules

### 7.1 — One package per directory, one directory per package.

**Reasoning, step by step:**

1. Go binds a package to a directory: every `.go` file in a directory must declare the same package, and a package cannot span directories. The mapping is one-to-one with no exceptions.
2. Honoring this mapping keeps the import path, the directory, and the package name in agreement, so a reader navigating the tree never has to guess which package a file belongs to.

**Enforcement:** `go vet`; the compiler rejects mixed package clauses in a directory.

### 7.2 — Name packages as short, lowercase, composable nouns.

**Reasoning, step by step:**

1. A package name must be a short noun, lowercase, with no underscores and no mixedCaps.
2. The package name appears at every call site, so it must compose well: `transport.New()`, `auth.NewOAuthProvider()`, `pipeline.Run()` read cleanly, while `transportpkg.NewTransport()`, `authentication.NewAuthenticationProvider()`, and `pipeline_runner.RunPipeline()` stutter and bloat.

```go
// Good
transport.New()
auth.NewOAuthProvider()
pipeline.Run()

// Bad -- stuttering
transportpkg.NewTransport()
authentication.NewAuthenticationProvider()
pipeline_runner.RunPipeline()
```

3. Choose a name that describes the content, not the role, location, or a non-existent inheritance relationship. The contrast:

| Good | Bad | Why Bad |
|------|-----|---------|
| `transport` | `transportpkg` | Redundant suffix |
| `auth` | `authentication` | Too long |
| `pipeline` | `utils` | Describes nothing |
| `token` | `common` | Junk drawer |
| `config` | `helpers` | Names the role, not the content |
| `store` | `shared` | Says where, not what |
| `metrics` | `base` | Implies inheritance Go doesn't have |

**Enforcement:** `golangci-lint` (`revive`/`stylecheck` package-name rules); review.

### 7.3 — Use `internal/` aggressively for implementation-detail packages.

**Reasoning, step by step:**

1. Put implementation-detail packages under `internal/`. The Go toolchain enforces that `internal/` packages can only be imported by code rooted at the parent of `internal/`, so the wall is mechanical, not a convention.
2. Use it aggressively: anything not part of the public surface belongs under `internal/`, where it can be refactored freely without breaking external callers.

```text
sdk/
  client.go           # public API
  options.go          # public API
  internal/
    auth/             # only sdk/ can import this
      oauth.go
      basic.go
    pipeline/         # only sdk/ can import this
      step.go
      logging.go
```

**Enforcement:** `go vet`; the toolchain rejects out-of-tree imports of `internal/`.

### 7.4 — Reserve `cmd/` for binaries and `pkg/` for genuine external consumption.

**Reasoning, step by step:**

1. The two top-level conventions divide as follows:

| Directory | Purpose |
|-----------|---------|
| `cmd/` | Executable entry points. Each subdirectory is a `main` package. |
| `pkg/` | Packages genuinely intended for external consumption. |

2. Do not use `pkg/` by default. Only use it when you have both internal and external packages in the same module and need to distinguish them; otherwise it is noise.
3. A simple project keeps its files at the root; a multi-binary project gathers entry points under `cmd/` and shares code through `internal/`:

```text
// Good -- simple project
myservice/
  main.go
  server.go
  handler.go
  internal/
    store/
    auth/

// Good -- multi-binary project
myproject/
  cmd/
    server/
      main.go
    cli/
      main.go
  internal/
    domain/
    store/
```

**Enforcement:** review; `cmd/*` subdirectories are `main` packages, `pkg/` used only with a stated reason.

### 7.5 — Keep packages small and single-purpose; split by responsibility.

**Reasoning, step by step:**

1. Prefer small, focused packages over large ones. A package with 20+ files or 3000+ lines is a signal to split, and a package should be describable in one sentence without using "and."
2. Watch for the signs a package is too large:
   - You struggle to name it without a generic word (util, service, core).
   - New contributors can't understand its purpose from the directory listing.
   - Changes to one feature trigger recompilation of unrelated code.
   - Tests for one feature need setup from another.
3. When you split, split by responsibility, not by technical layer — group code that changes together, not code that shares a mechanism.

**Enforcement:** review; package-size and naming signals flagged in review.

### 7.6 — Never create `utils/`, `helpers/`, or `common/` packages.

**Reasoning, step by step:**

1. Never create these packages. They attract unrelated code, grow without bound, and describe nothing.
2. Instead, put each function in a package that describes its domain, so the import path itself tells the reader what the code does:

```go
// Bad
import "github.com/yourorg/project/pkg/utils"
utils.FormatDate(t)
utils.ParseURL(s)
utils.RetryWithBackoff(fn)

// Good -- each function in a package that describes its domain
import "github.com/yourorg/project/internal/timeformat"
timeformat.ISO8601(t)

import "github.com/yourorg/project/internal/retry"
retry.WithBackoff(fn)
```

3. The test is naming: if you cannot name the package, the code does not belong together.

**Enforcement:** `golangci-lint`; review rejects `utils`/`helpers`/`common` package names.

### 7.7 — Keep dependencies pointing inward toward a dependency-free domain.

**Reasoning, step by step:**

1. Dependencies flow in one direction, from entry points inward, with the domain at the center importing nothing internal:

```text
cmd/         -->  domain/     -->  (no internal imports)
                  service/    -->  domain/
                  handler/    -->  service/, domain/
                  internal/   -->  domain/ (not service/, not handler/)
```

2. Main depends on domain. Domain depends on nothing internal. Never the reverse. Domain types must be importable without pulling in infrastructure.
3. The domain package declares the types and interfaces and stays free of internal imports; infrastructure packages import the domain and implement its interfaces, not the other way around:

```go
// Good -- domain package has no internal imports
package domain

type Hotel struct {
    ID   string
    Name string
}

type HotelStore interface {
    Get(ctx context.Context, id string) (*Hotel, error)
}
```

```go
// Good -- infrastructure implements domain interface
package postgres

import "github.com/yourorg/project/domain"

type HotelStore struct {
    db *sql.DB
}

func (s *HotelStore) Get(ctx context.Context, id string) (*domain.Hotel, error) {
    // ...
}
```

**Enforcement:** `golangci-lint` (`depguard`/import-cycle checks); review of import direction.

### 7.8 — One file per major type.

**Reasoning, step by step:**

1. Give each major type its own file, paired with its test, and gather related declarations with the type they serve:

| File | Contents |
|------|----------|
| `client.go` | `Client` type, its methods, and its constructor |
| `client_test.go` | Tests for `Client` |
| `options.go` | `Option` type and all `With*` functions |
| `transport.go` | `Transport` type and its methods |
| `errors.go` | Sentinel errors and custom error types |
| `doc.go` | Package-level documentation |

2. Never put everything in one file, and never split one type across multiple files — the file boundary should track the type boundary.

**Enforcement:** review; file-per-type layout checked in review.

### 7.9 — Put package documentation in `doc.go`.

**Reasoning, step by step:**

1. Use `doc.go` for package-level documentation — only a package comment and the package clause, nothing else.
2. Write the comment as the package's front door: name the primary type, link related symbols, and point to the constructors:

```go
// Package transport provides HTTP transport implementations for the SDK.
//
// The primary type is [Client], which wraps an [http.RoundTripper] with
// retry logic, authentication, and request/response logging.
//
// Use [New] to create a Client with default settings, or [NewClient]
// with [Option] functions for customization.
package transport
```

**Enforcement:** review; `golangci-lint` (`godot`/doc-comment linters).

### 7.10 — Group imports in standard order, separated by blank lines.

**Reasoning, step by step:**

1. Separate import groups with blank lines, in this order:
   1. **Standard library packages**
   2. **External (third-party and internal project) packages**
   3. **Protocol Buffer imports** (use `pb` suffix for `go_proto_library`, `grpc` suffix for `go_grpc_library`)
   4. **Side-effect imports** (`_ "path/to/package"`)
2. The grouping makes provenance scannable — stdlib, then everything fetched, then generated protobufs, then side-effect-only imports last:

```go
import (
    "context"
    "fmt"
    "net/http"

    "github.com/stretchr/testify/assert"
    "golang.org/x/sync/errgroup"
    "github.com/yourorg/project/internal/auth"

    hotelpb "github.com/yourorg/project/gen/hotel"
    hotelgrpc "github.com/yourorg/project/gen/hotel"

    _ "net/http/pprof"
)
```

**Enforcement:** `golangci-lint` (`gci`/`goimports`).

### 7.11 — Rename imports only when required or genuinely clarifying.

**Reasoning, step by step:**

1. You **must rename** an import when:
   - It conflicts with another import (prefer renaming the most local/project-specific one).
   - It is a generated protobuf package with underscores in the name — remove underscores and add `pb`/`grpc` suffix (e.g., `foosvcpb "path/to/foo_service_go_proto"`).
2. You **may rename** a package with an uninformative name (`util`, `v1`) to clarify intent. Do this sparingly.
3. When an import would collide with a common local variable name (e.g., importing `net/url` in a function with many `url` variables), add a `pkg` suffix: `urlpkg "net/url"`. Only necessary if the shadowing variable is in scope at usage sites.

**Enforcement:** `golangci-lint` (`importas`/`stylecheck`); review.

### 7.12 — Never use dot imports.

**Reasoning, step by step:**

1. `import . "path"` is banned. No exceptions.
2. It hides the origin of identifiers: a reader can no longer tell which package an unqualified name came from, defeating the call-site readability that package names exist for (7.2).

**Enforcement:** `golangci-lint` (`stylecheck` ST1001); review.

### 7.13 — Restrict blank imports to `main`, tests, and `//go:embed`.

**Reasoning, step by step:**

1. Blank imports (`import _`) are permitted only in `main` packages or tests that require them.
2. They are also permitted for `//go:embed` (`import _ "embed"`).
3. They are **not permitted in library packages**, even transitively — a library must not impose a side-effecting import on everyone who depends on it.

**Enforcement:** `golangci-lint`; review of `_` imports outside `main`/tests/embed.

## Cross-references

- Package, file, and identifier naming: [02 — Naming Conventions](./02-naming-conventions.md).
- Sentinel errors and custom error types in `errors.go`: [03 — Error Handling](./03-error-handling.md).
- Domain interfaces and public surface design: [05 — API Design](./05-api-design.md).
- Package-level documentation and `doc.go`: [11 — Documentation](./11-documentation.md).
