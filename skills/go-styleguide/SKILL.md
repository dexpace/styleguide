---
name: go-styleguide
description: Use when writing, editing, or reviewing Go (.go) source in a dexpace project — enforces the dexpace Go styleguide (errors as values, structs + functions, bounded concurrency, 70-line function cap). Also use before committing Go, or when asked to review Go against the styleguide.
---

# Go styleguide

Extends the [Google Go Style Guide](https://google.github.io/styleguide/go/); where they conflict, Google wins, except the deviations below.

## When this applies

Editing `*.go`, or reviewing Go. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data + functions: structs for state, functions and small interfaces for behavior. No inheritance, no embedding for "is-a".
2. Explicit over implicit: every dependency in the signature, every error returned. Library options follow documented defaults.
3. Immutable by default: return new values; no mutating shared state; read-only intent at API edges.
4. Errors are values: handle every path. No `_ = err`. Wrap with `%w` plus context (cause, inputs, correlation ID).
5. Composition over inheritance: small consumer-defined interfaces composed together.
6. Transform, don't mutate: input in, new output out; state changes localized and visible.
7. Always say why: comments explain reasoning, not mechanics.
8. Assert aggressively: ≥2 assertions per function on average; preconditions, postconditions, invariants; split compound checks.
9. Limits on everything: bound every loop, channel, retry, buffer, timeout. No recursion in library code.
10. Small functions: one thing each; **70-line hard cap, aim 20–40**; blank lines between logical sections.
11. Performance from the outset: work with the runtime's grain; optimize slowest resource first (network > disk > memory > CPU).
12. Zero technical debt: do it right the first time.

## Language hard rules

- `%w` for wrapping, one level; place it at the end of the format string.
- No in-band errors (no sentinel `-1`/`""` for "not found") — return an explicit error or a typed result.
- Error strings lowercase, no trailing punctuation.
- `context.Context` is the first parameter, named `ctx`; never store it in a struct; no custom context types.
- Prefer synchronous APIs; the caller adds concurrency. Every goroutine has a known lifetime and a way to stop.
- Accept interfaces, return structs; interfaces defined by the consumer, kept small.
- `gofmt` and `golangci-lint` are law. `t.Fatal` only from the test goroutine.
- `crypto/rand` for secrets; never `math/rand`. `defer` for cleanup; timeouts on all external I/O.

## Before you finish — verify

```bash
gofmt -l .          # must print nothing
golangci-lint run   # must pass clean
go vet ./...
go test ./...
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/go/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/go/01-formatting-and-tooling.md)
- [02 Naming](https://github.com/dexpace/styleguide/blob/main/go/02-naming-conventions.md)
- [03 Error Handling](https://github.com/dexpace/styleguide/blob/main/go/03-error-handling.md)
- [04 Concurrency](https://github.com/dexpace/styleguide/blob/main/go/04-concurrency.md)
- [05 API Design](https://github.com/dexpace/styleguide/blob/main/go/05-api-design.md)
- [06 Testing](https://github.com/dexpace/styleguide/blob/main/go/06-testing.md)
- [07 Package Organization](https://github.com/dexpace/styleguide/blob/main/go/07-package-organization.md)
- [08 Logging](https://github.com/dexpace/styleguide/blob/main/go/08-logging.md)
- [09 Serialization](https://github.com/dexpace/styleguide/blob/main/go/09-serialization.md)
- [10 Resource Management](https://github.com/dexpace/styleguide/blob/main/go/10-resource-management.md)
- [11 Documentation](https://github.com/dexpace/styleguide/blob/main/go/11-documentation.md)
- [12 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/go/12-variables-and-declarations.md)
- [13 Performance Hints](https://github.com/dexpace/styleguide/blob/main/go/13-performance-hints.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
