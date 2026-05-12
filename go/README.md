## Go Code Style

Binding style rules for Go projects. Prioritize **correctness**, **explicitness**, **simplicity** — never cleverness or abstraction.

This guide extends and defers to [Google's Go Style Guide](https://google.github.io/styleguide/go/) (Style Guide, Style Decisions, and Best Practices). Where our guidance conflicts with Google's, **Google wins** — Google's rules represent the canonical standard for idiomatic Go. This guide adds project-specific conventions on top: assertion density (TigerBeetle-inspired), 70-line function limit, explicit limits on all loops/queues/retries, and EG platform testing patterns. Additional supplemental guidance is drawn from [Effective Go](https://go.dev/doc/effective_go) and the [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md).

### Style Priorities (from Google)

1. **Clarity** — code's purpose is clear to the reader.
2. **Simplicity** — the simplest approach that accomplishes the goal.
3. **Concision** — high signal-to-noise ratio.
4. **Maintainability** — easy to modify correctly over time.
5. **Consistency** — matches the surrounding codebase.

Resolve rule conflicts in this order. Consistency is the tiebreaker, never an override.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Formatting & Tooling](./01-formatting-and-tooling.md) | gofmt, golangci-lint, imports, line length, conditionals, Yoda conditions, function size |
| 02 | [Naming Conventions](./02-naming-conventions.md) | Scope-proportional names, initialisms, receiver names, getters/setters, repetition rules, test doubles |
| 03 | [Error Handling](./03-error-handling.md) | `%w` placement, in-band errors, error strings, sentinels, typed errors, `Must` functions, panic discipline |
| 04 | [Concurrency](./04-concurrency.md) | context rules, custom-context ban, synchronous preferred, goroutine lifetimes, channels, mutexes |
| 05 | [API Design](./05-api-design.md) | Interfaces (small, consumer-defined), options (struct vs variadic), generics, type aliases, receiver types |
| 06 | [Testing](./06-testing.md) | Table-driven tests, assertion philosophy, `t.Fatal` from goroutines, useful failures, `cmp.Diff`, acceptance tests |
| 07 | [Package Organization](./07-package-organization.md) | Directory layout, internal packages, import grouping (stdlib/external/pb/side-effect), dot/blank imports |
| 08 | [Logging](./08-logging.md) | log/slog, levels, expensive-log guarding, no flags in libraries, PII, sensitive data masking |
| 09 | [Serialization](./09-serialization.md) | encoding/json, time handling, `%q`, struct/map init, validation, composite literals |
| 10 | [Resource Management](./10-resource-management.md) | defer, context, timeouts, connection pooling, `crypto/rand` for secrets, graceful shutdown |
| 11 | [Documentation](./11-documentation.md) | GoDoc, package comments, main binary comments, named result parameters, signal boosting |
| 12 | [Variables & Declarations](./12-variables-and-declarations.md) | `:=` vs `var`, struct field names, composite literals, channel direction, nil slices, raw strings |
| 13 | [Performance Hints](./13-performance-hints.md) | strconv over fmt, string-to-byte reuse, stack vs heap, size hints, string building |

---

## Cross-Cutting Concerns

Security, performance, and git practices are covered in the [root-level code style guide](../README.md).

## Design Goals

**Correctness > performance > developer experience.** When they conflict, this ordering decides.

## Philosophy

Go is closest to the Zig/Elixir ideal: no inheritance, no classes, no exceptions -- just structs, functions, interfaces, and goroutines. Assertion discipline, bounded execution, and explicit options are drawn from TigerBeetle's Tiger Style -- Go is the natural home for these principles.

1. **Structs + functions, no objects.** Structs hold data. Functions operate on data. Interfaces define behavior contracts. No inheritance, no method overriding.
2. **Errors are values -- handle every one.** No discarding errors with `_`. Every error is wrapped with context and propagated.
3. **Explicit over implicit.** No `init()` side effects. No global state. No magic. Dependencies are parameters.
4. **Immutable by default.** Return new values over mutating. Use exported fields when the zero value is meaningful and direct access is fine — this is the Go default (`http.Request.URL`, `http.Request.Method`). Reach for an unexported field + accessor only when you need to enforce an invariant, hide a computation, or keep the public API stable across internal changes. Accessors don't take a `Get` prefix — see [naming conventions §Getters and Setters](./02-naming-conventions.md).
5. **Accept interfaces, return structs.** Narrowest interface in, concrete type out. Interfaces are discovered, not designed upfront.
6. **Transform, don't mutate.** Process data through function pipelines. Each function takes input, returns new output.
7. **Always say why.** Comments explain reasoning, not mechanics.
8. **Assert aggressively.** Validate at every public boundary. Return errors, never accept garbage silently.
9. **Think about performance from the outset.** Stack over heap. Buffer reuse. Understand escape analysis.
10. **Zero technical debt.** Do it right the first time.
