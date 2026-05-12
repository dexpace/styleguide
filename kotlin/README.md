## Kotlin Code Style

Binding style rules for Kotlin projects, target **Kotlin 2.3**. Prioritize **correctness**, **explicitness**, **simplicity** — never cleverness, never abstraction for its own sake.

This guide is platform-agnostic. It covers the Kotlin language, the standard library, and `kotlinx` ecosystem. JVM-specific rules (Java interop, Spring/JPA, Loom, Jackson, SLF4J) live in the companion [Kotlin-JVM guide](../kotlin-jvm/).

This guide extends and defers to the [Kotlin official coding conventions](https://kotlinlang.org/docs/coding-conventions.html) and the [Google Android Kotlin style guide](https://developer.android.com/kotlin/style-guide). Where our guidance conflicts with theirs, **the official conventions win**. This guide adds project-specific conventions on top: assertion density (TigerBeetle-inspired), 60-line function limit, explicit bounds on every loop/queue/retry, and `Result<T, E>`-as-sealed-ADT discipline.

### Style Priorities

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
| 01 | [Formatting & Tooling](./01-formatting-and-tooling.md) | ktlint, detekt, EditorConfig, line length, trailing commas, expression bodies, function size cap |
| 02 | [Naming Conventions](./02-naming-conventions.md) | Scope-proportional names, camelCase/PascalCase, packages, backticks, properties vs getters/setters |
| 03 | [Nullability](./03-nullability.md) | Non-null default, `!!` banned, Elvis, smart casts, `requireNotNull`, contracts |
| 04 | [Variables & Declarations](./04-variables-and-declarations.md) | `val` vs `var`, immutable collections, `lateinit` vs `lazy`, top-level vs `object`, delegation |
| 05 | [Functions](./05-functions.md) | Expression vs block bodies, default + named args, `inline`/`crossinline`/`noinline`, extensions |
| 06 | [Classes & Data Modeling](./06-classes-and-data-modeling.md) | `data class`, `value class`, sealed hierarchies, `object`, composition over inheritance |
| 07 | [Kotlin Idioms: Sugar with Intent](./07-kotlin-idioms.md) | `by` delegation, scope functions, builders, expression-oriented forms, stdlib contracts, operators/infix, type aliases |
| 08 | [Error Handling](./08-error-handling.md) | Sealed `Result<T, E>`, exceptions for unrecoverable, exhaustive `when`, no swallowing |
| 09 | [Concurrency & Async](./09-concurrency.md) | Coroutines, structured concurrency, dispatchers, `Flow`, `Mutex`, `Channel` |
| 10 | [API Design](./10-api-design.md) | Interfaces, generics + variance, default args vs builder, visibility, `@RequiresOptIn`, pipeline pattern |
| 11 | [Testing](./11-testing.md) | Parameterized tests, no shared state, fixtures, property-based, assertion discipline |
| 12 | [Module Organization](./12-module-organization.md) | Source-set / package layout, `internal`, public surface, no cyclic deps |
| 13 | [Resource Management](./13-resource-management.md) | `use {}`, `AutoCloseable`, structured cancellation, `withTimeout`, secure random |
| 14 | [Documentation](./14-documentation.md) | KDoc on public API, `@param`/`@return`/`@throws`/`@sample`, package docs, why over what |
| 15 | [Performance](./15-performance.md) | `inline`, value classes, `Sequence` vs `List`, allocations, no premature opt |

---

## Cross-Cutting Concerns

Security, performance, and git practices are covered in the [root-level code style guide](../README.md). JVM-specific concerns are in the [Kotlin-JVM guide](../kotlin-jvm/).

## Design Goals

**Correctness > performance > developer experience.** When they conflict, this ordering decides.

## Philosophy

Kotlin gives you both worlds: data classes, sealed hierarchies, immutable collections, and structured concurrency on one side; classes, inheritance, and unchecked exceptions on the other. We pick the safe side every time and use the rest only where it pays.

1. **Data + functions, not objects.** `data class` for state. Top-level / extension functions for transformations. Sealed hierarchies for closed polymorphism. Composition + `by` delegation for code sharing — never open inheritance.
2. **Nullability is part of the type — never `!!`.** Resolve at the boundary using smart casts, `?:`, `?.`, or `requireNotNull`. `!!` is a missing model, not a shortcut.
3. **`val` over `var`. Immutable collections over mutable.** Mutability is an explicit choice you have to type — that's the right way around.
4. **Explicit over implicit.** No magic. Every dependency in the constructor or function signature. No reflection-driven control flow in new code. No global mutable state.
5. **Errors are values.** Sealed `Result<T, E>` for expected/domain failures. Exceptions only for unrecoverable conditions and programmer errors. Wrap at the boundary; do not let exceptions cross module boundaries silently.
6. **Pick the concurrency primitive deliberately.** Coroutines for most async/cancellation-aware code. Other primitives only when the system forces it. Name the seam.
7. **Small functions, breathing room.** Hard limit: 60 lines. Aim for 15–30. Separate logical sections with blank lines.
8. **Assert aggressively.** `require` for caller contracts, `check` for state invariants, `error(...)` for unreachable branches. Minimum two assertions per function on average. Pair-assert when feasible.
9. **Bound everything.** All loops, retries, queues, timeouts, coroutine scopes, and `Flow` operators with potentially-unbounded sources must have a fixed upper bound. No unbounded recursion in library code — use `tailrec` (which the compiler rewrites to a loop) when recursion is the natural shape of the problem; otherwise iterate.
10. **Exhaustive `when` over sealed hierarchies.** No `else` for closed sets — let the compiler tell you when a new variant breaks the world.
11. **Compose via delegation (`by`), not inheritance.** Class delegation for decoration. Property delegation (`by lazy`, `Delegates.observable`, custom) for backing-field discipline.
12. **Scope functions have intent — pick by purpose.** `apply` (configure, return receiver), `also` (side-effect, return receiver), `let` (transform / null-resolve, return result), `run`/`with` (group, return result). Never use one because it's familiar; choose by what should be returned.
13. **Embrace expression-oriented Kotlin.** `when`/`if`/`try` as expressions. Single-expression functions for one-liners. String templates over concatenation. Type-safe builders over builder classes. Let the language replace boilerplate.
14. **Performance from the outset, but pay for what you use.** `inline` for higher-order hot paths. `value class` for ID/wrapper types. `Sequence` for chained transforms on large collections. Don't `inline` everything; don't `Sequence`-ify short lists.
15. **Zero technical debt.** What exists meets the design goals. Public API hardens fast — `internal` aggressively, `@RequiresOptIn` for experimental.

## Influences

- **[Kotlin official coding conventions](https://kotlinlang.org/docs/coding-conventions.html)** — canonical authority. Where our guidance collides, the official conventions win.
- **[Google Android Kotlin style guide](https://developer.android.com/kotlin/style-guide)** — supplemental; useful for several rules even outside Android.
- **[Effective Kotlin (Marcin Moskała)](https://kt.academy/book/effectivekotlin)** — the closest thing Kotlin has to a community canon.
- **[Expedia Group Java SDK](https://github.com/ExpediaGroup/expediagroup-java-sdk)** — concrete examples of decoration via `by`, pipeline composition, sealed exceptions with correlation IDs.
- **TigerBeetle Tiger Style** — assertion density, 60-line function limit, limits on everything, no recursion, zero technical debt.

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **module / package level or larger** — never mix two styles within the same package. A half-migrated package is more confusing than either end state.
