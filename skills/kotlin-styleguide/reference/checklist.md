# Kotlin styleguide — full checklist

One heading per chapter. Distilled from each chapter's `### N.M` rules. Read on demand for audits.

### 01 — Formatting & Tooling

- ktlint formats, detekt lints; CI enforces both. Tooling drift is a bug.
- 120-column hard line limit; trailing commas everywhere allowed.
- Expression bodies for single-expression functions.
- Function-size cap 60 lines, hard; aim 15–30.
- Explicit, sorted imports; no wildcards.
- One top-level declaration per logical unit, not per file.
- Blank lines as logical separators; no Yoda conditionals; braces on every `if`.

### 02 — Naming Conventions

- `camelCase` for non-type, non-constant; `PascalCase` types; `SCREAMING_SNAKE` constants.
- Names scope-proportional; longer names for wider scope.
- Properties replace getters/setters; name them as nouns.
- Backticks only in test names; packages all-lowercase, no underscores, by feature.
- No Hungarian notation or type prefixes.
- Class names say what they are, function names say what they do; booleans positive, actions verb-first.
- Parameter names are public (named-argument call sites); test doubles named for what they are.

### 03 — Nullability

- Non-null is the default; opt into null deliberately.
- `!!` banned outside `main`/test/justified bridges.
- Resolve nullability at the boundary, not the call site.
- Elvis (`?:`) with a meaningful default or `error(...)`; lean on smart casts.
- `lateinit` for genuine framework injection only; `by lazy` for your own deferred non-null init.
- No `Optional<T>`; public API nullability annotations match intent and stay stable.
- `requireNotNull`/`checkNotNull` over manual null-check-and-throw.

### 04 — Variables & Declarations

- `val` over `var`; mutate only when it provably matters.
- Immutable collection types over mutable; `List` over `MutableList`.
- Inference for locals, explicit types for public API.
- `const val` for compile-time constants, top-level `val` for runtime constants.
- `companion object` for class-scoped constants/factories only.
- Top-level functions/properties over utility classes; `by` for backing properties.
- Destructuring for `Pair`/data classes only; initializer blocks rare and explained; one declaration per line.

### 05 — Functions

- One purpose per function; 60 lines is the ceiling.
- Expression bodies for single expressions, block bodies otherwise.
- Default arguments over overloads; named arguments past two same-type params or any boolean.
- Extension functions to extend types you don't own or add coherent operations.
- `inline` for higher-order hot paths and `reified`.
- Scope functions picked by intent; `vararg` only when the call site varies, else take `List`/`Iterable`.
- Side-effects out of the body where possible; function references over wrapping lambdas; `tailrec` only when iteration is worse; never write `: Unit`.

### 06 — Classes & Data Modeling

- `data class` for value-shaped types; `value class` for IDs/wrappers (not `typealias`).
- Sealed hierarchies for closed polymorphism; `object` for singletons.
- No `open` class without a documented inheritance contract; composition over inheritance every time.
- Equality from `data class`; never hand-write `equals` for value types.
- `Pair`/`Triple` local use only, never in public API.
- `internal` aggressively; `public` is slowest to revoke.
- Secondary constructors only for framework/interop; custom accessors simple, side-effect-free, idempotent.

### 07 — Kotlin Idioms

- Class delegation (`by`) the default for decoration; property delegation `by lazy`/`observable`/custom.
- Scope functions picked by what should be returned.
- Type-safe builders / lambda-with-receiver for grouped construction.
- `when`/`if`/`try` as expressions; string templates over concatenation; single-expression overrides.
- Operator overloading only when the symbol is the domain operation; `infix` reads like English.
- Type aliases for intent, not substitution; destructuring is positional — choose carefully.
- Implement stdlib contracts over parallel ones; pipeline composition via list-of-steps + `fold`; trailing lambdas and SAM conversion.

### 08 — Error Handling

- Domain failures are sealed ADTs, not exceptions; use `kotlin.Result` or a project `Result<T, E>`.
- Exceptions for unrecoverable, programmer-error, external-fault only.
- `require`/`check`/`error` for three distinct fault modes.
- Wrap exceptions at module boundaries; don't leak between layers.
- Exhaustive `when` over sealed subjects, no `else`.
- `runCatching` only at adapter boundaries; error messages include context the caller can't see.
- `try`/`finally` cleanup is a smell — use `use {}` or scope; pick one `Result` consumption style per module.

### 09 — Concurrency & Async

- Every coroutine has an explicit `CoroutineScope`; no `GlobalScope`.
- `suspend` functions don't pick a dispatcher internally; structured concurrency — launches complete before the scope returns.
- Cancellation is cooperative; honor it. `withTimeout` bounds every I/O suspend call.
- Dispatchers: `Default` CPU, `IO` blocking, custom for managed pools.
- `Flow` cold, `SharedFlow`/`StateFlow` hot; `buffer`/`conflate`/`collectLatest` deliberate, never unbounded.
- `Mutex` for coroutine state, `AtomicXxx` for primitives; `Channel` only when `Flow` doesn't fit; `async` only for concurrent results.
- Bound producers (backpressure is theirs); `runBlocking` only at entry point or tests.

### 10 — API Design

- `internal` default for in-module consumers; `public` requires intent.
- Small interfaces defined by the consumer's need; `fun interface` for single-method lambdas.
- Variance from the use site: `out` producers, `in` consumers; `reified` for runtime-type dispatch.
- Default arguments beat builders beat overload chains; `@RequiresOptIn` for experimental/dangerous API.
- Public functions have explicit return types always; stable identifiers, don't churn names.
- Pipeline pattern over inheritance hierarchies; `suspend` in public API commits callers to coroutines; never expose mutable types.

### 11 — Testing

- Tests describe behavior, not implementation; names are sentences.
- Arrange/act/assert, one concern per test; parameterized tests for table-driven cases.
- Real implementations where possible, mock only at the genuine seam; doubles named for what they are.
- Property-based tests for naturally invariant properties; tests independent — no order, no shared state.
- Fixtures via builders/functions, not mutable test-class state.
- Time, randomness, IDs injected, not pulled from the wild; useful failure messages.
- No `Thread.sleep` — poll, virtualize time, or use coroutines; ≥2 assertions per test on average.

### 12 — Module Organization

- Group by feature, not technical layer; mirror package structure in the source tree.
- One module per deployable boundary plus narrow shared modules.
- `internal` the workhorse for cross-package same-module code; no cyclic dependencies ever.
- Public API surface is a deliberate, written-down list.
- Generated code in its own source set, never edited.
- Docs live with the code: README per module, KDoc per public symbol; test organization mirrors production.
- `expect`/`actual` for multiplatform only where needed.

### 13 — Resource Management

- `use {}` on every `AutoCloseable`; never hand-call `close()`.
- Coroutine scopes are resources — cancel them; `withTimeout` the default I/O bound.
- Connection and thread pools are resources — bound them.
- `SecureRandom` for cryptographic randomness, `Random` otherwise.
- Graceful shutdown: drain in-flight, refuse new, close pools.
- Implement `Closeable`/`AutoCloseable` explicitly and document the lifecycle; locks are resources — bound the critical section.
- `NonCancellable` for the must-complete cleanup part; functional helpers over hand-rolled file/stream loops.

### 14 — Documentation

- Every `public` symbol has KDoc: summary line, blank, body, blank, tags.
- Explain why, not what.
- `@param` for params the signature doesn't document; `@return` for non-trivial returns; `@throws` for every deliberately thrown/propagated exception.
- `@sample` over copy-pasted examples; one `package.md` per public package.
- Links over restatement (`[OtherClass]`); don't document obvious overrides.
- `@Deprecated` with `ReplaceWith` and a removal date; in-body comments only where the why isn't in the code.

### 15 — Performance

- Design for the slowest resource first: network > disk > memory > CPU.
- `inline` for higher-order hot paths and `reified`; `value class` to avoid identifier wrapping cost.
- `Sequence` over `List` for chained transforms on large/unbounded data; size hints on builders/collections.
- Know where allocations live, eliminate where they matter.
- String ops: templates and `StringBuilder`, never `+` in loops; `IntArray`/`LongArray` over `List<Int>` in numeric hot paths.
- Don't micro-optimize without a profile; memoize with bounded caches.
- Coroutine overhead is cheap, not free; both cold-start and steady-state matter.
