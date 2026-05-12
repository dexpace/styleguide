# 04 — Variables & Declarations

How values come into being and how long they live. Most bugs you've ever shipped started as a `var` that should have been a `val` or a global that should have been a parameter.

## Rules

### 4.1 — `val` over `var`. Mutate only when it provably matters.

**Reasoning, step by step:**
1. A `val` is a fact: after the assignment line, its value never changes. The reader can rely on this across the rest of the function.
2. Shared mutable state is the source of most concurrency bugs and a large share of logic bugs.
3. Default to `val`. Reach for `var` only when (a) accumulation in a loop is genuinely clearer than `fold`/`reduce`, or (b) you're working with mutable framework state that has no fluent alternative.
4. Lint: detekt's `VarCouldBeVal` set to error.

### 4.2 — Immutable collection types over mutable. `List` over `MutableList`.

**Reasoning, step by step:**
1. Kotlin's `List<T>` (from `kotlin.collections`) is *read-only*: no `add`, no `remove`. `MutableList<T>` extends it with mutators.
2. Public APIs that accept `List<T>` allow callers to pass any list, mutable or not, without losing the read-only contract.
3. Public APIs that return `List<T>` (not `MutableList<T>`) signal "you can read; don't try to mutate."
4. **Note:** Kotlin's `List<T>` is a *view* — the underlying object may still be mutable, and another holder could change it. For genuinely-frozen lists, copy to `List<T>.toList()` at the boundary, or use `kotlinx.collections.immutable` for truly immutable structures when sharing across threads.
5. `listOf`/`setOf`/`mapOf` over `mutableListOf`/`mutableSetOf`/`mutableMapOf`. Mutability is what you have to type — that's the right way around.

### 4.3 — Type inference: yes for locals, explicit for public API.

**Reasoning, step by step:**
1. `val name = "Alice"` is clearly a `String`. No need to write `val name: String = "Alice"`.
2. Public/protected return types: write the type explicitly. The reader of a class file shouldn't have to read the function body to learn what it returns.
3. Public/protected property types: same.
4. Complex generic inferences (`val xs = listOf(1, 2L, 3.0)` infers `List<Number>` which surprises) — write the type.
5. Trade-off: in test code, inference can be more liberal because the function body is right there.

### 4.4 — `const val` for compile-time constants; top-level `val` for runtime constants.

**Reasoning, step by step:**
1. `const val` requires a primitive or `String` initializer that's resolvable at compile time. It inlines at every call site — fast, but binary-incompatible to change.
2. Top-level `val` (no `const`) is a singleton initialized at class-load time. It can be any type, including a `List`, a `Regex`, or a lazily-built map.
3. Convention: `const val SCREAMING_SNAKE_CASE` only. Lowercase `const val` is allowed by the compiler but looks like a regular local and reads as a bug.
4. Top-level non-const `val`: camelCase if it's a "namespaced singleton" (`defaultClock`), SCREAMING_SNAKE_CASE only if it's a *value constant* (`MAX_RETRIES`).

### 4.5 — `companion object` for class-scoped constants and factories. No more.

**Reasoning, step by step:**
1. `companion object` is the Kotlin home for what Java would put on static. Use it for class-scoped constants (`MAX_BUFFER`), factory functions (`fromJson`, `of`), and that's it.
2. **Anti-pattern:** stuffing utility functions into a `companion object` because Java muscle memory says "static methods." Make them top-level functions instead, or extension functions on the relevant receiver.
3. Each class has at most one `companion object`. If you find yourself wanting more, you wanted a separate file.
4. Named companions (`companion object Factory`) only when you need to reference the companion by name from Java callers — see [JVM guide](../kotlin-jvm/01-java-interop.md).

### 4.6 — Top-level functions and properties, not utility classes.

**Reasoning, step by step:**
1. Kotlin allows top-level declarations. Use them. `fun parseIsoDate(...)` at file scope beats `object DateUtils { fun parseIsoDate(...) }`.
2. Group related top-level functions by feature in one file (`Json.kt`, `TimeFormat.kt`).
3. Extension functions on the receiver type are usually even better — they're discoverable via IDE completion.
4. **Counter-example:** if a "utility" needs internal state (a cache, a parser, a config), it's not a utility — it's an object. Make it a `class` injected as a dependency.

### 4.7 — Delegation (`by`) for backing properties.

**Reasoning, step by step:**
1. `val config: Config by lazy { loadConfig() }` defers expensive initialization to first read.
2. `var count: Int by Delegates.observable(0) { _, old, new -> log("count: $old -> $new") }` runs a side effect on every write.
3. `val name: String by map` reads from a `Map<String, *>`. Useful for config-like classes.
4. Custom delegates: implement `getValue`/`setValue` operator functions. Build them when (a) the same backing pattern appears in three or more properties, and (b) the pattern can't be expressed as a regular function.
5. **Anti-pattern:** custom delegate where a `val` + private helper function would work. Delegation is for backing-field discipline, not for compressing one-liners.

### 4.8 — Destructuring for `Pair`/data classes — and only there.

**Reasoning, step by step:**
1. `val (host, port) = address` is clearer than `val host = address.host; val port = address.port` *when* the data class has positional meaning (a coordinate, an endpoint).
2. Destructuring uses *positional* `componentN()` accessors. Reordering data-class fields silently breaks every destructuring call site. This is a feature, not a bug — but it means destructuring is fragile for types you don't control.
3. **Rule:** destructure your own data classes only when positional access is the natural reading order. For random data classes (a `User` with 12 fields), use named property access.
4. Loops: `for ((key, value) in map) { ... }` is idiomatic; keep it.

### 4.9 — Initializer blocks: rare and explained.

**Reasoning, step by step:**
1. `init { ... }` blocks run during construction. Prefer property initializers (`val x = compute()`) over `init` blocks where possible — the value is visibly tied to its declaration.
2. Use `init` when (a) initialization requires multiple constructor parameters interacting, or (b) you need a precondition assertion (`init { require(port in 1..65535) }`).
3. Avoid multiple `init` blocks per class. Multiple blocks execute in source order, which is invisible to anyone reading the class.
4. Long `init` blocks are a sign that a factory function is the right pattern instead.

### 4.10 — One declaration per `val`/`var` line.

**Reasoning, step by step:**
1. Kotlin doesn't support `val a, b = 0, 0`-style multi-declarations, and that's good. Don't simulate it with destructuring of a fake `Pair`.
2. One line per declaration aids diffs and grep.
3. Exception: `for ((k, v) in map)` — that's a single destructuring declaration with a clear purpose.

## Worked example

```kotlin
// good
private const val MAX_RETRIES = 3
private val isoFormatter = DateTimeFormatter.ISO_INSTANT

class PaymentClient(
    private val http: HttpClient,
    private val clock: Clock = Clock.systemUTC(),
) {
    private val cache: Map<PaymentId, Receipt> by lazy { loadCache() }

    fun charge(card: Card, amount: Cents): Result<Receipt, ChargeError> { /* ... */ }
}

// bad
class PaymentClient {
    companion object {
        const val maxRetries = 3                                 // 4.4 case wrong
        val ISO_FORMATTER = DateTimeFormatter.ISO_INSTANT        // 4.5 — could be top-level
    }
    lateinit var http: HttpClient                                // 3.6 — should be constructor-injected
    var cache: MutableMap<PaymentId, Receipt> = mutableMapOf()   // 4.1, 4.2
}
```

## Cross-references

- `lateinit` and `by lazy`: chapter 03 (Nullability).
- Delegation patterns: chapter 07 (Kotlin Idioms).
- Visibility (`internal`, `private`): chapter 10 (API Design).
