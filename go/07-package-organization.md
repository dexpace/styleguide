# 07 - Package Organization

## Package = Directory

One package per directory. One directory per package. No exceptions.

## Package Names

- Short noun, lowercase, no underscores, no mixedCaps.
- The package name appears at every call site -- it must compose well.

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

| Good | Bad | Why Bad |
|------|-----|---------|
| `transport` | `transportpkg` | Redundant suffix |
| `auth` | `authentication` | Too long |
| `pipeline` | `utils` | Describes nothing |
| `token` | `common` | Junk drawer |
| `config` | `helpers` | Names the role, not the content |
| `store` | `shared` | Says where, not what |
| `metrics` | `base` | Implies inheritance Go doesn't have |

## internal/

Use `internal/` for implementation-detail packages. The Go toolchain enforces that `internal/` packages can only be imported by code rooted at the parent of `internal/`. Use it aggressively.

```
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

## cmd/ and pkg/

| Directory | Purpose |
|-----------|---------|
| `cmd/` | Executable entry points. Each subdirectory is a `main` package. |
| `pkg/` | Packages genuinely intended for external consumption. |

Do not use `pkg/` by default. Only use it when you have both internal and external packages in the same module and need to distinguish them.

```
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

## Package Size

Prefer small, focused packages over large ones. A package with 20+ files or 3000+ lines is a signal to split. A package should be describable in one sentence without using "and."

Signs a package is too large:
- You struggle to name it without a generic word (util, service, core).
- New contributors can't understand its purpose from the directory listing.
- Changes to one feature trigger recompilation of unrelated code.
- Tests for one feature need setup from another.

Split by responsibility, not by technical layer.

## No utils/, helpers/, common/

Never create these packages. They attract unrelated code, grow without bound, and describe nothing.

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

If you cannot name the package, the code does not belong together.

## Dependency Direction

```
cmd/         -->  domain/     -->  (no internal imports)
                  service/    -->  domain/
                  handler/    -->  service/, domain/
                  internal/   -->  domain/ (not service/, not handler/)
```

- Main depends on domain. Domain depends on nothing internal. Never the reverse.
- Domain types must be importable without pulling in infrastructure.

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

## One File Per Major Type

| File | Contents |
|------|----------|
| `client.go` | `Client` type, its methods, and its constructor |
| `client_test.go` | Tests for `Client` |
| `options.go` | `Option` type and all `With*` functions |
| `transport.go` | `Transport` type and its methods |
| `errors.go` | Sentinel errors and custom error types |
| `doc.go` | Package-level documentation |

Never put everything in one file. Never split one type across multiple files.

## doc.go

Use `doc.go` for package-level documentation -- only a package comment and the package clause:

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

## Import Grouping

Groups separated by blank lines, in this order:

1. **Standard library packages**
2. **External (third-party and internal project) packages**
3. **Protocol Buffer imports** (use `pb` suffix for `go_proto_library`, `grpc` suffix for `go_grpc_library`)
4. **Side-effect imports** (`_ "path/to/package"`)

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

## Import Renaming

You **must rename** an import when:
- It conflicts with another import (prefer renaming the most local/project-specific one).
- It is a generated protobuf package with underscores in the name — remove underscores and add `pb`/`grpc` suffix (e.g., `foosvcpb "path/to/foo_service_go_proto"`).

You **may rename** a package with an uninformative name (`util`, `v1`) to clarify intent. Do this sparingly.

When an import would collide with a common local variable name (e.g., importing `net/url` in a function with many `url` variables), add a `pkg` suffix: `urlpkg "net/url"`. Only necessary if the shadowing variable is in scope at usage sites.

## No Dot Imports

`import . "path"` is banned. No exceptions. It hides the origin of identifiers.

## Blank Imports (`import _`)

- Permitted only in `main` packages or tests that require them.
- Permitted for `//go:embed` (`import _ "embed"`).
- **Not permitted in library packages**, even transitively.

## Example Layout

```
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
