---
name: kotlin-styleguide
description: Use when writing, editing, or reviewing Kotlin (.kt) source in a dexpace project — enforces the dexpace Kotlin styleguide (non-null by default, sealed result types, scope functions, 60-line function cap). Also use before committing Kotlin, or when asked to review Kotlin against the styleguide.
---

# Kotlin styleguide

Extends the [Kotlin official coding conventions](https://kotlinlang.org/docs/coding-conventions.html) and the [Google Android Kotlin style guide](https://developer.android.com/kotlin/style-guide); where they conflict, the official conventions win, except the recorded deviations (120-column line limit; 60-line function cap).

## When this applies

Editing `*.kt`, or reviewing Kotlin. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data + functions: `data class` for state, top-level/extension functions for transformations, sealed hierarchies for closed polymorphism. No open inheritance for reuse.
2. Nullability is part of the type: resolve at the boundary with smart casts, `?:`, `?.`, or `requireNotNull`. `!!` is a missing model, never a shortcut.
3. `val` over `var`; immutable `List` over `MutableList`. Mutability is the choice you have to type.
4. Explicit over implicit: every dependency in the constructor or signature. No reflection-driven control flow, no global mutable state.
5. Errors are values: sealed `Result<T, E>` for expected failures; exceptions only for unrecoverable conditions; wrap at the boundary, never leak across layers.
6. Pick the concurrency primitive deliberately: coroutines for most async code, others only when forced. Name the seam.
7. Small functions, breathing room: **60-line hard cap, aim 15–30**; blank lines between logical sections.
8. Assert aggressively: `require` for caller contracts, `check` for invariants, `error(...)` for unreachable branches. ≥2 assertions per function on average; pair-assert when feasible.
9. Bound everything: every loop, retry, queue, timeout, scope, and unbounded-source `Flow` carries a fixed cap. No unbounded recursion in library code — use `tailrec` or iterate.
10. Exhaustive `when` over sealed hierarchies: no `else` for closed sets, so the compiler flags every new variant.
11. Compose via delegation (`by`), not inheritance: class delegation for decoration, property delegation for backing-field discipline.
12. Zero technical debt: what exists meets the design goals; harden public API fast with `internal` and `@RequiresOptIn`.

## Language hard rules

- `!!` is banned outside `main`/test scaffolding and justified bridges. Resolve nullability at the boundary; internals take non-null types. Use Elvis with a real default or `error(...)`, `requireNotNull`/`checkNotNull` for caller-fault contracts.
- No bare `as`; prefer `as?` plus handling, and any cast carries a why-comment.
- Model absence and results as sealed hierarchies; `when` over a sealed type stays exhaustive — no `else` catch-all that defeats narrowing.
- Errors: sealed `Result<T, E>` (or `kotlin.Result`) for expected failures; exceptions only for unrecoverable or programmer-error cases; never swallow.
- `by` delegation for reuse without "is-a"; `data class` for state, `value class` for IDs; no inheritance for code reuse; no `open` class without a documented contract.
- Structured concurrency: coroutines scoped (no `GlobalScope`), dispatchers explicit, `Flow` cold, `Channel`/buffers bounded, `withTimeout` on I/O.
- Assert ≥2 per function on average; **60-line hard cap** (ktlint/detekt-enforced).
- ktlint + detekt are law; 120-column lines; trailing commas; expression bodies where they read better.

## Before you finish — verify

```bash
ktlint --format     # autocorrect, then re-check
ktlint              # must pass clean
detekt              # must pass clean
./gradlew detekt test   # or the project's build/test task
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/kotlin/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/kotlin/01-formatting-and-tooling.md)
- [02 Naming Conventions](https://github.com/dexpace/styleguide/blob/main/kotlin/02-naming-conventions.md)
- [03 Nullability](https://github.com/dexpace/styleguide/blob/main/kotlin/03-nullability.md)
- [04 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/kotlin/04-variables-and-declarations.md)
- [05 Functions](https://github.com/dexpace/styleguide/blob/main/kotlin/05-functions.md)
- [06 Classes & Data Modeling](https://github.com/dexpace/styleguide/blob/main/kotlin/06-classes-and-data-modeling.md)
- [07 Kotlin Idioms](https://github.com/dexpace/styleguide/blob/main/kotlin/07-kotlin-idioms.md)
- [08 Error Handling](https://github.com/dexpace/styleguide/blob/main/kotlin/08-error-handling.md)
- [09 Concurrency & Async](https://github.com/dexpace/styleguide/blob/main/kotlin/09-concurrency.md)
- [10 API Design](https://github.com/dexpace/styleguide/blob/main/kotlin/10-api-design.md)
- [11 Testing](https://github.com/dexpace/styleguide/blob/main/kotlin/11-testing.md)
- [12 Module Organization](https://github.com/dexpace/styleguide/blob/main/kotlin/12-module-organization.md)
- [13 Resource Management](https://github.com/dexpace/styleguide/blob/main/kotlin/13-resource-management.md)
- [14 Documentation](https://github.com/dexpace/styleguide/blob/main/kotlin/14-documentation.md)
- [15 Performance](https://github.com/dexpace/styleguide/blob/main/kotlin/15-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
