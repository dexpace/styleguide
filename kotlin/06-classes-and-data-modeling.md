# 06 — Classes & Data Modeling

Kotlin makes the right things easy: immutable records, closed hierarchies, single-instance types. Use them. Inheritance is the last resort, not the first.

## What good looks like

```kotlin
@JvmInline value class UserId(val raw: String) {
    init { require(raw.isNotBlank()) { "UserId must be non-blank" } }
}

sealed interface Result<out T, out E> {
    data class Ok<T>(val value: T) : Result<T, Nothing>
    data class Err<E>(val error: E) : Result<Nothing, E>
}

data class Charge(
    val id: ChargeId,
    val user: UserId,
    val amount: Cents,
    val createdAt: Instant,
)

class ChargeService(private val gateway: Gateway) {
    fun charge(c: Charge): Result<Receipt, DeclineReason> = when (val r = gateway.submit(c)) {
        is GatewayResponse.Approved -> Result.Ok(r.receipt)
        is GatewayResponse.Declined -> Result.Err(r.reason)
    }
}
```

`UserId` is a `value class` wrapping a single `val`, distinct from `String` and validated in `init` (6.2); `Result` is a closed `sealed interface` whose nested `Ok`/`Err` variants drive an exhaustive `when` (6.3); `Charge` is an all-`val` `data class` carrying value-shaped state and no behavior (6.1); `ChargeService` composes its `gateway` dependency rather than inheriting one (6.6) and stays `final` and `public`-by-omission only where it must be (6.5, 6.9).

## Rules

### 6.1 — `data class` for value-shaped types.

**Reasoning, step by step:**
1. A "value-shaped" type is one whose identity is its content: a `Point(x, y)`, a `UserId`, a `PaymentRequest`. Two with the same fields are equal.
2. `data class` gives you `equals`, `hashCode`, `toString`, `componentN`, and `copy` for free. Writing them by hand is wasted typing and a future bug.
3. **Make all properties `val`.** A mutable data class breaks `hashCode` stability — if you put one in a `Set` and mutate it, the set is broken.
4. Don't add behavior beyond simple derived properties. Data classes are records, not service objects. Behavior lives in top-level functions, extension functions, or non-data classes.
5. **Trap:** `equals`/`hashCode` use only properties declared in the *primary constructor*. Properties added in the body are not part of equality. Keep all equality-relevant state in the primary constructor.

**Enforcement:** detekt `DataClassShouldBeImmutable` flags `var` properties; review for behavior creeping into records.

### 6.2 — `value class` for ID and wrapper types. Not `typealias`.

**Reasoning, step by step:**
1. `typealias UserId = String` doesn't create a new type — every `String` is still a `UserId`, and every `UserId` is still a `String`. `findUser(orderId)` compiles.
2. `@JvmInline value class UserId(val raw: String)` creates a *distinct* type. `findUser(orderId)` is a compile error.
3. Value classes erase to their underlying type at runtime (no boxing in most cases), so the runtime cost is near zero.
4. Use them for: identifiers (`UserId`, `OrderId`), units (`Cents`, `Millis`), wrappers (`Email`, `PostalCode`).
5. **Limit:** value classes have one property in the primary constructor. They cannot have an `init` block with side effects, but they can have a validating `init { require(...) }`.

**Enforcement:** review; identifiers and units declared as `@JvmInline value class`, not `typealias` or raw primitives.

### 6.3 — Sealed hierarchies for closed polymorphism.

**Reasoning, step by step:**
1. A sealed class or interface defines a finite set of subtypes known to the compiler. Exhaustive `when` over a sealed subject fails to compile if you miss a case.
2. Use sealed hierarchies to model ADTs: results (`Result.Ok`/`Result.Err`), states (`Connection.Connecting`/`Connection.Open`/`Connection.Closed`), commands (`Command.Start`/`Command.Stop`).
3. Prefer `sealed interface` over `sealed class` when there's no shared state. Interfaces allow a type to belong to multiple hierarchies; classes don't.
4. Variants live as nested types under the parent — the lexical grouping documents the closure: `sealed interface Result<out T, out E> { data class Ok<T>(val value: T) : Result<T, Nothing>; data class Err<E>(val error: E) : Result<Nothing, E> }`.
5. Sealed hierarchies are the Kotlin substitute for Go's interface-based polymorphism *when the set is closed*. Open polymorphism still uses regular interfaces.

**Enforcement:** compiler — exhaustive `when` over a sealed subject fails to compile on a missing branch (no `else` escape hatch).

### 6.4 — `object` for singletons. `companion object` for class-scoped statics.

**Reasoning, step by step:**
1. `object Logger { ... }` is a singleton: one instance for the JVM lifetime, thread-safe initialization, can implement interfaces.
2. `companion object` is the singleton inside another class — used for factory functions and constants. See chapter 04.
3. **Anti-pattern:** `object` as a namespace for unrelated utility functions. Use a file with top-level functions.
4. `object` is the right tool when (a) you need to implement an interface but only need one instance, (b) you have a true singleton (clock provider, config registry).

**Enforcement:** review; reject `object` used as a bag of unrelated utilities in place of top-level functions.

### 6.5 — No `open` class without a documented inheritance contract.

**Reasoning, step by step:**
1. Kotlin classes are `final` by default. To allow subclassing, mark `open`. This is a deliberate decision.
2. `open` says: *I have thought about how subclasses interact with my invariants, and I've documented which methods they may override.* If you haven't done that work, don't write `open`.
3. Inheritance issues compound: overriding methods must respect the parent's contract (Liskov), data classes can't be `open` in a useful way, and `open` properties break smart casts.
4. **Default alternatives:** composition + `by` delegation (see chapter 07), `sealed` hierarchies for closed polymorphism, interfaces for behavior contracts.
5. **JVM exception:** some frameworks (Spring, JPA) require `open` for proxying. Use the `kotlin-allopen` / `kotlin-spring` compiler plugins instead of hand-writing `open` everywhere — see [JVM guide](../kotlin-jvm/04-persistence.md).

**Enforcement:** review; every hand-written `open` carries a documented override contract, framework `open` comes from a compiler plugin.

### 6.6 — Composition over inheritance, every time.

**Reasoning, step by step:**
1. Inheritance couples the lifecycle of two types forever. A subclass is a parent — any change to the parent affects every subclass.
2. Composition: give the consumer the dependency, expose what you need on the public API.
3. Code reuse via inheritance is almost always achievable via (a) extension functions on a common type, (b) delegation, (c) helper top-level functions, (d) a sealed hierarchy with a shared interface.
4. **Pattern worth absorbing:** `class LoggerDecorator(private val logger: Logger) : Logger by logger { override fun info(m: String) = logger.info(decorate(m)) }`. Decoration in one line. No `open class AbstractLogger`.

**Enforcement:** review; reuse via delegation (`by`), extension functions, or sealed hierarchies, not an `abstract`/`open` base class.

### 6.7 — Equality: rely on `data class`; never hand-write `equals` for value types.

**Reasoning, step by step:**
1. Hand-written `equals`/`hashCode` are bug magnets. They're hard to get right and easy to silently break.
2. If you find yourself hand-writing them, the type should be a `data class`.
3. Exception: identity-based equality on reference types (a connection, a thread, a session). For these, *don't override* `equals` at all — `Any.equals` (identity) is correct.
4. For domain entities (a `User` with an ID), equality by ID can be modeled with a custom `equals`/`hashCode`, but consider whether the type should be split: a value-shaped `UserData` (data class) plus an identity-shaped `User` (regular class holding the data and the ID).

**Enforcement:** review; a hand-written `equals`/`hashCode` on a value type is a smell — convert it to a `data class`.

### 6.8 — `Pair` and `Triple`: local use only. Never in public API.

**Reasoning, step by step:**
1. `Pair<String, Int>` tells the reader nothing about what `first` and `second` mean.
2. In a public signature, this is a documentation failure: `fun lookup(): Pair<User, Token>` — which one is which? What about `fun lookup(): Triple<User, Token, Expiry>`?
3. The fix is one line: `data class LookupResult(val user: User, val token: Token)`. Now the fields have names.
4. Acceptable use: local destructuring inside a function (`val (k, v) = map.entries.first()`), `Map.Entry`-shaped iteration.
5. Lint: detekt's `ReturnTypePair` rule.

**Enforcement:** detekt `ReturnTypePair` blocks `Pair`/`Triple` in return types; review confines them to local destructuring.

### 6.9 — Visibility: `internal` aggressively. `public` is the slowest to revoke.

**Reasoning, step by step:**
1. Kotlin's default is `public`. The default is wrong for most declarations.
2. Apply visibility in this order, narrowest first: `private` (file or class), `internal` (module), `public` (everywhere).
3. `internal` is the safety net: it confines the symbol to the current module, so you can refactor freely without breaking external callers.
4. Once `public`, every refactor is a deprecation cycle. Get it right before publishing.
5. See chapter 10 (API Design) for the full visibility playbook.

**Enforcement:** detekt `LibraryEntitiesShouldNotBePublic` / explicit-API mode; review demotes `public` to `internal` wherever a symbol need not cross the module boundary.

### 6.10 — Secondary constructors only when the framework or interop demands it.

**Reasoning, step by step:**
1. Kotlin's primary constructor + default arguments handles 95% of what Java needs secondary constructors for.
2. Secondary constructors are useful when (a) you need to delegate to a *different* primary constructor with substantial pre-work, (b) framework reflection requires a specific signature (see `kotlin-noarg` for JPA — [JVM guide](../kotlin-jvm/04-persistence.md)).
3. If you find yourself writing three secondary constructors, the right answer is a factory function (`fun User.fromCsv(row: String): User`) or a builder.

**Enforcement:** review; secondary constructors justified by framework/interop, parsing alternatives expressed as factory functions.

### 6.11 — Properties with custom accessors: simple, side-effect-free, idempotent.

**Reasoning, step by step:**
1. A property *looks* like field access. Callers expect cheap, deterministic reads.
2. Custom `get()`: must be cheap (no I/O, no expensive computation) and idempotent. If it isn't, make it a function — `loadConfig()` not `val config`.
3. Custom `set()`: side effects belong here only if they're necessary invariants of the property (caching a hash, notifying observers). Don't trigger remote work in a setter.
4. Backing field: use `field` inside accessors. Never expose `_name` private backing properties unless you genuinely need a writable internal view and a read-only external view.

**Enforcement:** review; custom `get()` stays cheap and idempotent — anything doing I/O or expensive work becomes a function.

## Worked example

```kotlin
// good
@JvmInline value class UserId(val raw: String) {
    init { require(raw.isNotBlank()) { "UserId must be non-blank" } }
}

sealed interface ChargeResult {
    data class Approved(val receipt: Receipt) : ChargeResult
    data class Declined(val reason: DeclineReason) : ChargeResult
    data class Error(val cause: Throwable) : ChargeResult
}

data class Payment(
    val id: PaymentId,
    val user: UserId,
    val amount: Cents,
    val createdAt: Instant,
)

class PaymentClient(
    private val http: HttpClient,
    private val auditor: Auditor,
) {
    suspend fun charge(payment: Payment): ChargeResult = /* ... */
}

// bad
open class AbstractPayment {                  // 6.5 — open without contract
    var id: String = ""                       // 6.1 — mutable + stringly-typed
    var amount: Int = 0
}

class Payment : AbstractPayment() {           // 6.6 — inheritance not needed
    fun toPair(): Pair<String, Int> = id to amount  // 6.8 — Pair in public API
}
```

## Cross-references

- Delegation (`by`) — chapter 07 (Kotlin Idioms).
- Sealed `Result<T, E>` — chapter 08 (Error Handling).
- Compiler plugins for `open` — [JVM guide chapter 04](../kotlin-jvm/04-persistence.md).
