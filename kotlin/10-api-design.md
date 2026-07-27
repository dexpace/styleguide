# 10 — API Design

Designing the surface other code calls. The cost of a bad public API is paid by every caller, in every refactor, forever.

## What good looks like

```kotlin
/** Consumer-defined: the narrowest read surface a charge needs (10.2). */
fun interface AccountReader { // SAM-convertible, callable as reader(id) (10.3)
    operator fun invoke(id: AccountId): Account?
}

class BillingService internal constructor( // ctor not part of the contract (10.1)
    private val readAccount: AccountReader,
) {
    /** Explicit return type — the body can change, the contract can't (10.8). */
    fun charge(id: AccountId, amount: Cents): ChargeResult {
        val account = readAccount(id) ?: return ChargeResult.UnknownAccount
        return account.debit(amount)
    }

    /** Read-only view out; an internal MutableList never leaks (10.12). */
    fun history(id: AccountId): List<Charge> = ledger[id].orEmpty()
}
```

`AccountReader` is a one-method `fun interface` defined by what `charge` consumes, not by some 27-method repository (10.2, 10.3), and it is invoked as `readAccount(id)` through `operator fun invoke`. The primary constructor is `internal`, so collaborators inject but outsiders cannot bind to it (10.1). `charge` and `history` both declare explicit return types (10.8), and `history` hands back a read-only `List` rather than the backing `MutableList` (10.12).

## Rules

### 10.1 — `internal` is the default for things consumed within a module. `public` requires intent.

**Reasoning, step by step:**
1. Kotlin defaults declarations to `public`. The default is wrong for almost everything.
2. Visibility ladder (most-restrictive first): `private` (file/class), `internal` (module), `public` (world).
3. Apply the narrowest visibility that compiles. Widen only when a caller outside that scope genuinely needs the symbol.
4. `internal` is the freedom-to-refactor visibility. You can rename, retype, and delete `internal` symbols without breaking external callers.
5. Once `public`, removal/rename is a deprecation cycle. Get it right *before* the first release.

**Enforcement:** `explicitApi()` mode (warns on inferred-public declarations); review for the narrowest visibility that compiles.

### 10.2 — Small interfaces. Define them by the consumer's need, not the producer's surface.

**Reasoning, step by step:**
1. A `UserRepository` interface with 27 methods is impossible to fake in tests and brittle to extend.
2. Define interfaces by the *minimum a caller needs*. A function that just reads users takes `UserReader { fun find(id: UserId): User? }`, not a 27-method repo.
3. Consumer-defined interfaces let one concrete class implement many small interfaces — the concrete grows, the interfaces don't.
4. **Anti-pattern:** "anticipatory" methods on an interface because "we might need this someday." Add them when the second caller appears.
5. **Anti-pattern:** an interface with one implementation that exists only "for testing." If the implementation is the only one, fake it via constructor injection of dependencies, not via an interface layer.

**Enforcement:** review; interfaces sized to the consumer, no anticipatory or test-only methods.

### 10.3 — `fun interface` for single-method interfaces consumed as lambdas.

**Reasoning, step by step:**
1. Kotlin's `fun interface` enables SAM conversion: callers can pass a lambda where the interface is expected.
2. Use it for callbacks, hooks, single-method strategies. `fun interface RequestStep { operator fun invoke(req: Request): Request }` lets callers write `pipeline.add { it.copy(...) }`.
3. Don't use `fun interface` when the abstract method is part of a richer contract or when SAM conversion would hide intent.
4. For Java callers, a Kotlin `fun interface` is just a SAM interface — interop is seamless.

**Enforcement:** review; `fun interface` for lambda-consumed single-method contracts, plain `interface` when the method is part of a richer one.

### 10.4 — Generics: variance from the *use site* perspective. `out` for producers, `in` for consumers.

**Reasoning, step by step:**
1. `interface Source<out T> { fun get(): T }` — `T` is only produced (returned), so `Source<Cat>` is a `Source<Animal>`. Covariant.
2. `interface Sink<in T> { fun put(value: T) }` — `T` is only consumed (parameter), so `Sink<Animal>` is a `Sink<Cat>`. Contravariant.
3. Declare variance once at the type, not at every use site (Kotlin's *declaration-site variance* beats Java's `? extends T` wildcards).
4. Use-site variance (`List<out T>`) only when the type can't be declared variant (e.g., it's both producer and consumer at different methods).
5. **Don't reach for variance** prematurely. Many generic types are fine as invariant — the compiler will tell you when variance is needed.

**Enforcement:** the compiler rejects produced-`in`/consumed-`out` misuse; review for declaration-site variance over use-site wildcards.

### 10.5 — `reified` for generic dispatch on a runtime type.

**Reasoning, step by step:**
1. JVM erases generics — at runtime, `List<String>` and `List<Int>` are both `List`. Reified types break the erasure for inline functions.
2. `inline fun <reified T> Any.castOrNull(): T? = this as? T` — works because the type is known at the call site and inlined.
3. Use `reified` to (a) avoid passing a `Class<T>` parameter, (b) implement `instanceOf` checks generically, (c) deserialize without `KType` plumbing.
4. **Limit:** `reified` requires `inline`. Both come together.

**Enforcement:** the compiler requires `inline` for `reified`; review that it replaces a `Class<T>`/`KType` parameter rather than padding a hot inline path.

### 10.6 — Default arguments beat builders. Builders beat overload chains.

**Reasoning, step by step:**
1. For ≤ ~7 parameters with sensible defaults, use default args + named arguments. Single source of truth, no builder boilerplate.
2. For genuinely large configuration objects (>8 fields, complex defaults, conditional construction), a builder DSL with `apply { }` reads better than a 20-argument constructor.
3. Avoid Java-style telescoping constructors entirely.
4. **DSL builder example:**
   ```kotlin
   val client = httpClient {
       baseUrl = "https://api.example.com"
       timeout = 5.seconds
       retries { count = 3; backoff = 100.millis }
   }
   ```
   Use `@DslMarker` on the receiver type.

**Enforcement:** review; default + named args under ~7 params, a `@DslMarker` builder above that, never telescoping constructors.

### 10.7 — `@RequiresOptIn` for experimental, unstable, or dangerous API.

**Reasoning, step by step:**
1. `@RequiresOptIn` is Kotlin's "you must explicitly accept this" marker. Use it for unstable APIs.
2. Three tiers:
   - **Experimental** — may change at any time. Callers opt in by annotation or compiler flag.
   - **Internal-but-public** — used across modules but not part of the public contract. `@RequiresOptIn(level = ERROR)`.
   - **Delicate** — works correctly but easy to misuse. Document the trap; require opt-in to acknowledge.
3. Annotate at the *source* of the unstable API; callers see the require-opt-in warning.
4. **Note:** `@RequiresOptIn` is not for permission management. It's for *forcing the caller to read the docs*.

**Enforcement:** the compiler emits the opt-in warning/error at every unannotated call site; review that unstable APIs carry the marker.

### 10.8 — Public functions have explicit return types. Always.

**Reasoning, step by step:**
1. Inference on a public function means changing the body changes the published return type. Silent ABI breakage.
2. Write the return type. The reader of the API doesn't need to read the body to learn what it returns.
3. Same rule for public properties.
4. `internal` and `private` may rely on inference where it's obvious. The trade-off is local.

**Enforcement:** `explicitApi()` mode requires explicit types on public declarations; binary-compatibility validator catches drift.

### 10.9 — Stable identifiers. Don't churn names.

**Reasoning, step by step:**
1. Renaming a `public` function is a binary-incompatible change for every external caller.
2. Get names right *before* the first release of a module.
3. To deprecate: `@Deprecated("Use newName(...) instead", ReplaceWith("newName(...)"))`. Keep the old name shimmed for a release cycle.
4. To rename internally: `internal` only, refactor freely. That's the value of `internal`.

**Enforcement:** binary-compatibility validator gates the public ABI in CI; renames of publics go through `@Deprecated(ReplaceWith(...))`.

### 10.10 — Pipeline pattern for composed transformations (vs. inheritance hierarchies).

**Reasoning, step by step:**
1. When a transformation is the *composition of several small steps*, model it as `List<Step>` and `fold`. Don't model it as `AbstractRequestProcessor` with `open` hooks.
2. Each step is one class (or `fun interface`) with one responsibility. Steps are composable, testable, swappable.
3. **Worked pattern:**
   ```kotlin
   fun interface RequestStep { operator fun invoke(req: Request): Request }
   fun interface ResponseStep { operator fun invoke(res: Response): Response }

   class ExecutionPipeline(
       val requestSteps: List<RequestStep>,
       val responseSteps: List<ResponseStep>,
   ) {
       fun process(req: Request): Response =
           respond(send(requestSteps.fold(req) { acc, s -> s(acc) }))
               .let { initial -> responseSteps.fold(initial) { acc, s -> s(acc) } }
   }
   ```
4. Inversion of control: callers compose pipelines from steps. Authors add steps without changing existing ones.

**Enforcement:** review; composed transformations are `List<Step>` + `fold`, not `open` hooks on an abstract base.

### 10.11 — Suspend functions in public APIs commit you to coroutine callers.

**Reasoning, step by step:**
1. `suspend fun load(): User` is callable only from coroutines. Non-coroutine Java callers need a bridge (`GlobalScope.future { load() }`, see [JVM guide ch. 02](../kotlin-jvm/02-jvm-concurrency.md)).
2. Decide *deliberately*: is this API for Kotlin-with-coroutines callers, or do you need to support synchronous and `CompletableFuture` callers?
3. For "both," provide two functions: `suspend fun load(): User` for Kotlin and `fun loadAsync(): CompletableFuture<User>` for Java/sync. The async version is a thin bridge.
4. Don't expose `Flow` to Java callers without a bridge — Flow is a coroutine concept.

**Enforcement:** review; `suspend`/`Flow` in a public API is a deliberate coroutine-caller commitment, with a `CompletableFuture` bridge where Java/sync callers must be served.

### 10.12 — Don't expose mutable types in public API.

**Reasoning, step by step:**
1. Returning `MutableList<T>` says "callers may mutate this; we may be observing." Either is bad.
2. Returning `List<T>` (read-only view) is the default. Return `List<T>.toList()` if you genuinely need an immutable snapshot.
3. Accepting `MutableList<T>` as a parameter forces callers to convert. Accept `List<T>` (or `Iterable<T>`) and copy inside if you need to mutate.
4. Same applies to `Map`, `Set`, `Collection`.

**Enforcement:** review; public signatures use read-only `List`/`Map`/`Set`/`Collection`, never their `Mutable` counterparts.

## Cross-references

- Visibility and module boundaries: chapter 12.
- Generic types and value classes: chapter 06.
- Java interop annotations on public API: [JVM guide chapter 01](../kotlin-jvm/01-java-interop.md).
- `@RequiresOptIn` for ABI stability: [JVM guide chapter 08](../kotlin-jvm/08-build-and-distribution.md).
