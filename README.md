# styleguide
Binding rules for all dexpace projects. Follow them exactly. When in doubt, default to: **data + functions, immutable, explicit, errors handled**.

## Priority

**Correctness > performance > developer experience.** When they conflict, this ordering decides.

Style concerns, in order (from [Google's Go Style Guide](https://google.github.io/styleguide/go/guide)):

1. **Clarity** — the code's purpose and rationale is clear to the reader.
2. **Simplicity** — the code accomplishes its goals in the simplest way possible.
3. **Concision** — the code has a high signal-to-noise ratio.
4. **Maintainability** — the code is written to be easily maintained.
5. **Consistency** — the code is consistent with the broader Expedia Group and Go codebases.

When rules conflict, resolve by the order above. Consistency is the lowest-priority tiebreaker — it never overrides clarity, simplicity, concision, or maintainability.

## Rules

1. **Data and functions, not objects.** Structs hold data. Functions transform data. Interfaces define behavior contracts. No inheritance, no method overriding.
2. **Explicit over implicit.** No hidden control flow. No framework magic. Every dependency visible in the signature. Every error returned explicitly. Library options follow their documented defaults — callers specify only what differs from the default.
3. **Immutable by default.** Return new values over mutating. Unexported fields with exported getters. Pipelines over in-place modification.
4. **Errors are values.** Handle every error explicitly. No silent swallowing. No `_ = err`. Wrap with context using `%w`.
5. **Composition over inheritance.** Pipelines over template methods. Functions over class hierarchies. Small interfaces composed together.
6. **Transform, don't mutate.** Each function takes input, returns new output. State changes are explicit, localized, and visible.
7. **Always say why.** Comments explain reasoning, not mechanics. If you can't explain why a line exists, question whether it should.
8. **Assert aggressively.** Minimum two assertions per function on average. Preconditions at entry, postconditions at exit, invariants at boundaries. Fail fast with a clear message. Assert both positive space (what you expect) and negative space (what you don't). Split compound assertions. Use pair assertions: verify the same property from two different code paths.
9. **Limits on everything.** All loops, queues, retries, and timeouts must have a fixed upper bound. No recursion in library code -- all execution must be provably bounded. Functions must not exceed 70 lines.
10. **Small functions, breathing room.** Functions do one thing. Hard limit: 70 lines. Aim for 20-40. Separate logical sections with blank lines. Cramped code is unreadable code. Whitespace is free -- use it.
11. **Performance from the outset.** Design-time is the best time for 1000x improvements. Work with the grain of the runtime. Optimize for the slowest resource first: network > disk > memory > CPU.
12. **Zero technical debt.** What exists meets the design goals. Do it right the first time. The second chance may never come.

## Influences

- **[Google Go Style Guide](https://google.github.io/styleguide/go/)**: Canonical authority on Go style. Where our guidance collides with Google's, Google wins. Our guide extends Google with project-specific conventions (assertion density, option explicitness exceptions, etc.).
- **[Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)**: Supplemental guidance for concurrency, performance, and common pitfalls.
- **Elixir**: Immutable data, pipeline composition, `{:ok, result}` / `{:error, reason}`, no objects.
- **Zig**: Structs + free functions, no hidden control flow, errors as values, exhaustive switches.
- **TigerBeetle Tiger Style**: Assertion density (2+ per function). Pair assertions. Positive and negative space. 70-line function limit. Limits on everything. No recursion. Zero technical debt.

## Cross-Cutting Guides

| Guide | Scope |
|-------|-------|
| [Security](./security.md) | Input validation, injection prevention, secrets, TLS, rate limiting |
| [Performance](./performance.md) | Design-phase perf, batching, caching, pooling, profiling, benchmarking |
| [Git and Code Review](./git-and-code-review.md) | Commits, branches, PRs, review etiquette, merge strategy, releases |

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **package level or larger** — never mix two styles within the same package. A half-migrated package is more confusing than either of the two end states.

Refactor the whole package in one commit, or split the migration across commits that each leave the package in a consistent state.

## Go Style Guide

See [Go Code Style](./go/) -- 11 topic files covering formatting, naming, errors, concurrency, API design, testing, package organization, logging, serialization, resource management, and documentation.

