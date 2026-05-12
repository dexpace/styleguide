# 03 — Nullability

Nullability is the single most-improved part of Kotlin over Java. Use it. Disrespect it and you get NPEs the type system was specifically designed to prevent.

## Rules

### 3.1 — Non-null is the default. Null is something you opt into.

**Reasoning, step by step:**
1. `String` is non-null. `String?` is nullable. The `?` is a contract you owe the caller.
2. Adding `?` to a return type is a *breaking* change for callers who relied on the type system to mean "never null." Treat it like changing a method signature.
3. Removing `?` from a parameter type is similarly breaking — old callers may have been passing null.
4. **Default principle:** prefer the non-null type. If a value can be absent, ask whether "absent" deserves its own representation (sealed `Optional`-ish ADT, an empty collection, a sentinel) before reaching for `?`.

### 3.2 — `!!` is banned outside `main`/test scaffolding and well-justified bridges.

**Reasoning, step by step:**
1. `!!` is an unchecked assertion: "trust me, this isn't null." If it ever is, you get a `NullPointerException` with no useful context.
2. Every `!!` is either (a) a missing model — the value *should* be non-null in the type system, or (b) a latent production bug.
3. Acceptable uses:
   - Scripts and `main()` where a fast-fail with a meaningless stack trace is acceptable.
   - Test setup where a fixture is known to be configured before the test runs.
   - The boundary line between Java and Kotlin (see [JVM guide chapter 03](../kotlin-jvm/03-jvm-frameworks.md)) — and even there, prefer `requireNotNull(value) { "context" }`.
4. **Lint rule:** detekt's `UnsafeCallOnNullableType` set to error in CI.

### 3.3 — Resolve nullability at the boundary, not at the call site.

**Reasoning, step by step:**
1. A nullable value that crosses three function calls before being checked is three opportunities for the wrong default.
2. Resolve as close to the *origin* as possible: parse the input, validate, and convert `T?` to `T` (or to a sealed result type) at the adapter layer.
3. The body of the function should operate on non-null values — that's the whole point of the type system.
4. **Anti-pattern:** every internal function takes `User?` and re-checks with `?.let`. The check belongs at the input boundary; internal functions take `User`.

### 3.4 — Prefer Elvis (`?:`) with a meaningful default or `error(...)`.

**Reasoning, step by step:**
1. `val name = user?.name ?: "anonymous"` is clearer than `if (user != null) user.name else "anonymous"`.
2. For unrecoverable null: `val name = user?.name ?: error("user has no name")`. This throws `IllegalStateException` with a real message — better than `!!`'s naked NPE.
3. For "this is the caller's contract": `val name = requireNotNull(user.name) { "name required" }`. Throws `IllegalArgumentException`, which is the right exception for a contract violation.
4. Elvis chains read well: `cache[key] ?: load(key) ?: error("no $key")`. Use them.

### 3.5 — Smart casts: lean on them; don't fight them.

**Reasoning, step by step:**
1. After `if (x != null)`, `x` is smart-cast to non-null inside the branch. After `x is User`, `x` is smart-cast to `User`. Use this.
2. Smart casts fail on `var` properties of mutable classes and on `open`/`abstract` properties — because another thread or override could change them between the check and the use. The compiler is right; don't argue.
3. **Fix:** copy to a local `val` first. `val u = user ?: return; useUser(u)` — now `u` is smart-cast to `User` for the rest of the function.
4. Smart casts also work in `when` arms. Lean on this for sealed hierarchies.

### 3.6 — `lateinit` is for genuine framework injection only. Never for "I'll set it later in my own code."

**Reasoning, step by step:**
1. `lateinit` says: "this `var` is non-null, but it's not initialized at construction time, and I promise to set it before anyone reads it." Reading before set throws `UninitializedPropertyAccessException`.
2. Legitimate uses: dependency injection by reflection (Spring, Guice on JVM), test setup (`@BeforeEach`), and that's about it.
3. Illegitimate uses: working around constructor parameters being inconvenient. If you're doing this, the type should be `T?` and you should resolve to `T` explicitly, or the value should be `by lazy` (see 3.7).
4. `lateinit` cannot be `val`, cannot be a primitive (`Int`, `Long`), and cannot be nullable — all by design.

### 3.7 — `by lazy` for deferred non-null initialization driven by *your* code.

**Reasoning, step by step:**
1. `val config: Config by lazy { loadConfig() }` is the idiomatic way to defer expensive initialization.
2. Default mode is `LazyThreadSafetyMode.SYNCHRONIZED` — safe for shared state. Use `PUBLICATION` if the work is idempotent and benign to run twice. Use `NONE` only inside a single-threaded scope.
3. Prefer `by lazy` over `lateinit var` when you control when initialization happens. Reserve `lateinit var` for *external* initializers (frameworks, tests).
4. **Trap:** `by lazy` captures the enclosing `this`. If the holder leaks before the lazy runs, the captured references leak too. Be mindful in long-lived holders.

### 3.8 — `Optional<T>` does not exist in Kotlin. Don't import it.

**Reasoning, step by step:**
1. The Kotlin answer to `Optional<T>` is `T?`. The language gives you the same expressiveness without the box.
2. The only acceptable place to see `Optional` is at a Java boundary you don't control (see [JVM guide](../kotlin-jvm/01-java-interop.md)). Convert to `T?` at that boundary.
3. Returning `Optional<T>` from a Kotlin function is wrong twice: it allocates a box, and it contradicts the nullability type system.

### 3.9 — Public API: nullability annotations match intent and stay stable.

**Reasoning, step by step:**
1. A `public` function's nullability of return type and parameters is part of its semantic contract.
2. Don't loosen non-null returns to nullable to "fix" a bug — fix the cause, or change the function to return a `Result` / sealed ADT.
3. Don't tighten nullable parameters to non-null without a deprecation period — old callers may be passing null on purpose.
4. **Migration pattern:** add a new function with the better signature, deprecate the old one, deletion in a later release.

### 3.10 — `requireNotNull` and `checkNotNull` over manual null checks + throws.

**Reasoning, step by step:**
1. `val name = requireNotNull(user.name) { "user $userId has no name" }` is one line; the equivalent `if/throw` is three.
2. `requireNotNull` throws `IllegalArgumentException` — semantically: *the caller passed bad input*.
3. `checkNotNull` throws `IllegalStateException` — semantically: *our state is inconsistent*.
4. Choosing between them documents whose fault the null is. That's worth typing the extra letters for.

## Worked examples

```kotlin
// good — resolve at the boundary, operate on non-null internally
fun handleRequest(raw: String?): Response {
    val parsed = raw?.let(::parse) ?: return Response.badRequest("empty body")
    return process(parsed)  // process takes Parsed, not Parsed?
}

// good — fail fast with context
fun chargeCard(user: User, amount: Cents): Receipt {
    val card = requireNotNull(user.defaultCard) { "user ${user.id} has no default card" }
    return card.charge(amount)
}

// bad — !! everywhere
fun handleRequest(raw: String?): Response = process(parse(raw!!)!!)
```

## Cross-references

- `lateinit` + DI: [JVM guide chapter 03](../kotlin-jvm/03-jvm-frameworks.md).
- Platform types from Java: [JVM guide chapter 01](../kotlin-jvm/01-java-interop.md).
- Sealed `Result<T, E>` as an alternative to `T?`: chapter 08 (Error Handling).
