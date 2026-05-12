# styleguide

Binding rules for all dexpace projects. Follow them exactly. When in doubt, default to: **data + functions, immutable, explicit, errors handled**.

This repository holds:

- **Cross-cutting rules** (this README plus [security](./security.md), [performance](./performance.md), [git & code review](./git-and-code-review.md)) — apply to every language and runtime.
- **Per-language guides** — adapt the cross-cutting rules to each language's idioms and add language-specific conventions. Each guide names a canonical upstream authority and extends it.

## Priority

**Correctness > performance > developer experience.** When they conflict, this ordering decides.

Style concerns, in order:

1. **Clarity** — the code's purpose and rationale is clear to the reader.
2. **Simplicity** — the code accomplishes its goals in the simplest way possible.
3. **Concision** — the code has a high signal-to-noise ratio.
4. **Maintainability** — the code is written to be easily maintained.
5. **Consistency** — the code is consistent with the surrounding codebase.

When rules conflict, resolve by the order above. Consistency is the lowest-priority tiebreaker — it never overrides clarity, simplicity, concision, or maintainability.

## Rules

These rules apply to *every* dexpace project regardless of language. Each per-language guide restates them in language-native terms.

1. **Data and functions, not objects.** Group state into plain records (`struct`, `data class`, `@dataclass`, etc.). Group behavior into functions and small interfaces. No inheritance for code reuse, no deep class hierarchies, no method overriding for shared logic. Composition over inheritance, always.
2. **Explicit over implicit.** No hidden control flow. No framework magic. Every dependency visible in the signature. Every error returned or thrown explicitly. Library options follow their documented defaults — callers specify only what differs from the default.
3. **Immutable by default.** Return new values over mutating. Read-only collections in public APIs. Pipelines over in-place modification. Mutability is an explicit choice the author has to type — that's the right way around.
4. **Errors are values, handled explicitly.** Every error path covered. No silent swallowing. No discarding errors with the language's "ignore this" idiom (`_ = err`, bare `except:`, empty `catch`). Wrap errors with context (the cause, the inputs, the correlation ID) using the language's idiomatic mechanism.
5. **Composition over inheritance.** Pipelines over template methods. Functions over class hierarchies. Small interfaces composed together. Delegation (`by` in Kotlin, embedding in Go, mixins/protocols in Python) when code reuse is needed without an "is-a" relationship.
6. **Transform, don't mutate.** Each function takes input, returns new output. State changes are explicit, localized, and visible. Pure functions are easier to test, parallelize, and reason about.
7. **Always say why.** Comments explain reasoning, not mechanics. If you can't explain why a line exists, question whether it should.
8. **Assert aggressively.** Minimum two assertions per function on average. Preconditions at entry, postconditions at exit, invariants at boundaries. Fail fast with a clear message. Assert both positive space (what you expect) and negative space (what you don't). Split compound assertions. Use pair assertions: verify the same property from two different code paths.
9. **Limits on everything.** All loops, queues, retries, buffers, and timeouts must have a fixed upper bound. No recursion in library code — all execution must be provably bounded. Functions must not exceed the language's documented size cap (70 lines in Go, 60 in Kotlin, similar in Python).
10. **Small functions, breathing room.** Functions do one thing. Aim for 15–40 lines. Separate logical sections with blank lines. Cramped code is unreadable code. Whitespace is free — use it.
11. **Performance from the outset.** Design-time is the best time for 1000× improvements. Work with the grain of the runtime. Optimize for the slowest resource first: network > disk > memory > CPU. See [performance.md](./performance.md).
12. **Zero technical debt.** What exists meets the design goals. Do it right the first time. The second chance may never come.

## Per-language guides

Each guide adapts the cross-cutting rules to a specific language, names its canonical upstream authority, and adds language-specific conventions on top.

| Language | Guide | Upstream authority | Notes |
|---|---|---|---|
| Go | [`go/`](./go/) | [Google Go Style Guide](https://google.github.io/styleguide/go/) | Structs + functions, errors as values, goroutines + channels, `gofmt`/`golangci-lint`. |
| Kotlin (any target) | [`kotlin/`](./kotlin/) | [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html) | Platform-agnostic Kotlin: nullability, sealed hierarchies, scope functions, coroutines, structured concurrency. |
| Kotlin on JVM | [`kotlin-jvm/`](./kotlin-jvm/) | Extends `kotlin/`; defers to Kotlin coding conventions + Spring/Jackson/SLF4J ecosystem norms | Java interop annotations, Spring/Ktor wiring, JPA, Loom vs coroutines, Jackson at boundaries. |
| Python | [`python/`](./python/) | [PEP 8](https://peps.python.org/pep-0008/) + [PEP 20](https://peps.python.org/pep-0020/) + [PEP 484/604](https://peps.python.org/pep-0484/) | Python 3.12+. Type hints + mypy strict, dataclasses + Protocols, asyncio with `TaskGroup`, Ruff for lint + format. |

Each guide is split into topic files (formatting, naming, error handling, concurrency, API design, testing, etc.). Read the guide's `README.md` for the index.

## Cross-cutting guides

| Guide | Scope |
|---|---|
| [Security](./security.md) | Input validation, injection prevention, secrets, TLS, rate limiting, credential safety |
| [Performance](./performance.md) | Design-phase perf, batching, caching, pooling, profiling, benchmarking |
| [Git & Code Review](./git-and-code-review.md) | Commits, branches, PRs, review etiquette, merge strategy, releases |

## Influences

The rules above draw from many sources. Specific language guides defer to their own canonical authorities (linked in each guide), but the principles in *this* root README come from a wider set:

- **[Google Go Style Guide](https://google.github.io/styleguide/go/)** — clarity hierarchy (clarity → simplicity → concision → maintainability → consistency), naming, error messages.
- **[Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html)** and **Effective Kotlin** — nullability discipline, scope functions, composition idioms.
- **[PEP 8](https://peps.python.org/pep-0008/), [PEP 20 (The Zen of Python)](https://peps.python.org/pep-0020/), [Black](https://black.readthedocs.io/)** — "explicit is better than implicit," formatting as a non-discussion.
- **Elixir** — immutable data, pipeline composition, `{:ok, result}` / `{:error, reason}`, no objects.
- **Zig** — structs + free functions, no hidden control flow, errors as values, exhaustive switches.
- **TigerBeetle Tiger Style** — assertion density (2+ per function), pair assertions, positive and negative space, function size limits, limits on everything, no recursion, zero technical debt.
- **[Expedia Group SDKs](https://github.com/ExpediaGroup)** — concrete examples of pipeline composition, interface decoration via delegation, sealed error hierarchies with correlation context.

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **package / module level or larger** — never mix two styles within the same package. A half-migrated package is more confusing than either of the two end states.

Refactor the whole package in one commit, or split the migration across commits that each leave the package in a consistent state.

## Adding a new language guide

When a project lands in a new language:

1. Create a `<language>/` directory at the repo root.
2. Add a `README.md` that (a) names the language's canonical upstream authority, (b) lists topic files, (c) restates the 12 root rules in that language's vocabulary.
3. Split topic files by concern. Look at `go/` and `kotlin/` for the shape: formatting, naming, errors, concurrency, API design, testing, etc.
4. Add an entry to the "Per-language guides" table above.
5. Each rule in each topic file should include step-by-step reasoning — not just "do X," but *why* doing X serves correctness, performance, or developer experience.
