# 07 — Kotlin Idioms: Sugar with Intent

Kotlin's syntactic surface is large. Used deliberately, it removes Java's busywork. Used carelessly, it produces code that looks clever and reads like a puzzle.

## What good looks like

```kotlin
/** One small step in the request pipeline; SAM-converted at the call site (7.14). */
fun interface RequestStep { operator fun invoke(req: Request): Request }

private val addAuth = RequestStep { it.with(header = "Authorization" to "Bearer $token") }
private val addTrace = RequestStep { it.with(header = "X-Trace" to newTraceId()) }

/** Composes steps by folding the request through the list — no inheritance, each step testable. */
class Pipeline(private val steps: List<RequestStep>) {
    fun run(initial: Request): Request = steps.fold(initial) { req, step -> step(req) }
}

fun buildRequest(url: String, token: String): Request =
    Request.Builder()
        .apply {                                 // configure-then-return: receiver as `this`
            setUrl(url)
            setTimeout(5.seconds)
        }
        .build()
        .also { log.debug("built ${it.method} ${it.url}") } // side-effect-then-pass-along

fun classify(status: Int): String = when {       // when as an expression, single val
    status < 400 -> "ok"
    status < 500 -> "client-error"
    else         -> "server-error"
}
```

`RequestStep` is a `fun interface` so the lambdas SAM-convert and read as plain blocks (7.14); its `operator fun invoke` lets `step(req)` and `addAuth(req)` call the step as if it were a function (7.8). `Pipeline` composes via list-of-steps + `fold` rather than a template-method hierarchy (7.13). `apply` returns the receiver while exposing it as `this`, and `also` passes the value along while logging (7.3). `classify` uses `when` as an expression so the result is one `val` (7.5), and the header pair leans on `to` as an infix builder (7.9).

## Rules

### 7.1 — Class delegation (`by`) is the default for decoration.

**Reasoning, step by step:**
1. `class LoggerDecorator(private val l: Logger) : Logger by l { override fun info(m: String) = l.info(decorate(m)) }` is the entire decorator pattern in one declaration.
2. Without `by`: write every method (`override fun debug(m: String) = l.debug(m)`, etc.) and remember to add new ones every time the interface grows. With `by`: the compiler does this for you and stays in sync.
3. Use class delegation to add cross-cutting behavior (logging, metrics, tracing, masking) to an existing interface implementation.
4. **Trap:** delegation captures the delegate *at construction*. If you swap out the implementation later (a field reassignment), the delegate continues calling the old one. For most cases this is fine — for swappable implementations, hand-write the override.
5. **Worked pattern (from the Expedia SDK):**
   ```kotlin
   class LoggerDecorator(private val logger: Logger) : Logger by logger {
       override fun info(msg: String) = logger.info(decorate(msg))
       override fun warn(msg: String) = logger.warn(decorate(msg))
       private fun decorate(msg: String): String = "[SDK] $msg"
   }
   ```
   `Logger by logger` covers every method we *don't* override; the explicit overrides intercept exactly what we care about.

**Enforcement:** review; decorators delegate with `by`, hand-written forwarding only where the delegate is swappable.

### 7.2 — Property delegation: `by lazy`, `by Delegates.observable`, then custom.

**Reasoning, step by step:**
1. `by lazy { compute() }` — deferred non-null initialization, thread-safe by default.
2. `by Delegates.observable(initial) { _, old, new -> ... }` — runs a callback on write.
3. `by Delegates.vetoable(initial) { _, _, new -> new.isValid }` — runs a predicate; rejects writes that return false.
4. `by map` — back a property by a `Map<String, *>` entry.
5. **Custom delegate:** implement `operator fun getValue(thisRef: T, prop: KProperty<*>): V` (and `setValue` for `var`). Worth it when the same backing pattern appears in 3+ properties. Not worth it for one-offs.
6. **Anti-pattern:** custom delegate where a regular `val` + a helper function would do the same job. Delegation is for *backing-field* discipline, not for hiding logic.

**Enforcement:** review; reach for stdlib delegates (`lazy`, `Delegates.*`) first, custom delegates only past the 3-property threshold.

### 7.3 — Scope functions: pick by what should be returned.

**Reasoning, step by step:**
1. Decision rule:

   | I want to return... | And I want the body to access the receiver as... | Use   |
   |---------------------|--------------------------------------------------|-------|
   | The receiver        | `this`                                           | `apply` |
   | The receiver        | `it`                                             | `also`  |
   | The lambda's result | `this`                                           | `run` / `with` |
   | The lambda's result | `it`                                             | `let`   |

2. Worked uses:
   - **Configure-then-return:** `Request.Builder().apply { setHeader("k","v"); setTimeout(5.s) }.build()`.
   - **Side-effect-then-pass-along:** `request.also { log(it); validate(it) }`.
   - **Resolve-null-then-transform:** `user?.let { renderProfile(it) }`.
   - **Group-and-derive:** `with(parsed) { Outcome(field1, field2, computeFromBoth()) }`.
3. **Common mistakes:**
   - `apply` to compute a derived value → use `run`.
   - `let` to configure a builder → use `apply`.
   - `also` for a transformation → use `let`.
4. **Don't chain scope functions for the sake of chaining.** `?.let { it.foo }?.let { it.bar }` should be `?.foo?.bar`.
5. **Don't reach for a scope function when a named local is clearer.** A `val parsed = parse(raw)` followed by three lines of work is often more readable than a `with(parse(raw)) { ... }` block.

**Enforcement:** review against the return/receiver decision table; flag chained scope functions that a `?.` path would replace.

### 7.4 — Type-safe builders / lambda-with-receiver for grouped construction.

**Reasoning, step by step:**
1. Kotlin's `buildString { append("a"); append("b") }` is a lambda-with-receiver: inside the lambda, `this` is a `StringBuilder`.
2. The same pattern produces DSLs: `html { body { p("hello") } }`. Each block has a typed receiver, so completion and type-checking just work.
3. Use type-safe builders when (a) the construction has nested structure, (b) the same shape is built repeatedly, (c) the builder methods would otherwise return `this` for chaining.
4. Stdlib utilities to lean on: `buildString`, `buildList`, `buildMap`, `buildSet`.
5. **Authoring your own DSL:** annotate the receiver type with `@DslMarker` to prevent accidental access to outer receivers from inner blocks. This is the difference between "I built a DSL" and "I built a footgun."

**Enforcement:** review; prefer `buildString`/`buildList`/`buildMap`/`buildSet`, and require `@DslMarker` on every hand-authored builder receiver.

### 7.5 — Expression-oriented forms: `when`, `if`, `try` as expressions.

**Reasoning, step by step:**
1. Kotlin's `when`, `if`, and `try` are expressions — they have a value. Use that value.
   ```kotlin
   val grade = when {
       score >= 90 -> "A"
       score >= 80 -> "B"
       else        -> "C"
   }
   ```
2. Compared to assigning the result inside each branch (`if (score >= 90) grade = "A"`), the expression form makes the assignment a single `val` and a single point of update.
3. `try` as an expression: `val parsed = try { Json.decode(raw) } catch (e: JsonException) { null }`.
4. Exhaustive `when` on a sealed subject: omit `else` so the compiler enforces total coverage. See chapter 08.

**Enforcement:** review; assign once from the expression form rather than mutating a `var` across branches.

### 7.6 — String templates over concatenation.

**Reasoning, step by step:**
1. `"Hello, $name! You have ${cart.size} items."` is shorter, reads in document order, and never mismatches a `+`.
2. `"foo" + null + "bar"` produces `"foonullbar"` — silent. `"foo${null}bar"` does the same, but is uglier on purpose because you're meant to handle nullability.
3. Multi-line string templates use raw strings (`"""..."""`). Use `.trimIndent()` to strip leading whitespace.
4. **Anti-pattern:** template with a complex expression inside. If `${complex.expression}` is hard to read, lift it to a `val` above.

**Enforcement:** review; string templates over `+` concatenation, complex `${...}` lifted to a named `val`.

### 7.7 — Single-expression overrides and one-liners.

**Reasoning, step by step:**
1. `override fun toString(): String = "$method $url"` is enough. No `{ return ... }`, no `: Unit`.
2. Use the expression form for accessor-like overrides; reserve block bodies for genuine logic.
3. Works for properties too: `override val isEmpty: Boolean get() = size == 0`.

**Enforcement:** review; single-expression `=` form for accessor-like overrides, block bodies reserved for genuine logic.

### 7.8 — Operator overloading: only when the symbol *is* the domain operation.

**Reasoning, step by step:**
1. `Vector + Vector` means vector addition. `Duration + Duration` means add durations. These read like math because they *are* math.
2. `User + Order` doesn't mean anything obvious. Don't overload `+` for it. Use `User.addOrder(o)` or `User.with(order = o)`.
3. Heuristic: operator overloading is right when the symbol's *only* meaning in the domain matches what the operation does.
4. The most useful overloads: `plus`/`minus`/`times`/`div` for numeric-shaped types; `get`/`set` for collection-shaped types; `invoke` for command-shaped types (`val parse = JsonParser(); parse(input)`); `contains` (`in`) for set-shaped types; `compareTo` (`<`/`>`) for ordered types.
5. **Avoid:** `unaryMinus` cleverness, `rangeTo`/`rangeUntil` outside numeric/temporal contexts.

**Enforcement:** review; an overloaded operator is rejected unless the symbol's sole domain meaning matches the operation.

### 7.9 — `infix` functions: read like English in the call site.

**Reasoning, step by step:**
1. `infix fun User.likes(other: User): Boolean = ...` enables `if (alice likes bob)`.
2. Use `infix` when the call site reads better as `subject verb object` than as `subject.verb(object)`. Examples: `1 to "a"` (in `mapOf`), `value shouldBe expected` (in test DSLs).
3. **Don't** make `infix` something that reads worse: `account credit amount` versus `account.credit(amount)`. The dot is clearer.
4. `infix` functions take exactly one parameter, are member or extension, and don't accept `vararg` or default-value parameters.

**Enforcement:** review; `infix` only where `subject verb object` reads better than the dotted call.

### 7.10 — Type aliases for *intent*, not for *substitution*.

**Reasoning, step by step:**
1. `typealias UserId = String` is wrong: it creates no new type, so it doesn't prevent passing the wrong `String`. Use `value class` for that.
2. `typealias FooHandler = (Foo) -> Unit` is right: it gives a structural type a name so call-site signatures read better.
3. `typealias UserMap = Map<UserId, User>` is right: it shortens a verbose generic.
4. **Rule:** type aliases are documentation. They don't change types, they don't change ABI, they don't change anything except how the source reads. Use them only when a *name* makes the source read measurably better.

**Enforcement:** review; a `typealias` used for type safety is rejected in favor of a `value class`.

### 7.11 — Destructuring is positional. Choose carefully.

**Reasoning, step by step:**
1. `val (host, port) = address` calls `address.component1()` and `address.component2()` by position.
2. Reordering data-class fields silently breaks every destructuring call site. This is fine for your *own* tightly-coupled types; dangerous for types you don't control.
3. Use destructuring when (a) the positional reading matches the data — coordinates, key/value pairs, success/error tuples — and (b) the receiver type is unlikely to gain new fields.
4. `_` to ignore unwanted components: `val (_, value) = entry`.

**Enforcement:** review; destructure only types you control whose positional reading matches the data.

### 7.12 — Implement stdlib contracts instead of inventing parallel ones.

**Reasoning, step by step:**
1. Need pagination over an unknown-length stream? Implement `Iterator<T>`. Need ordering? `Comparable<T>`. Need release on scope exit? `AutoCloseable`.
2. The standard library has years of design behind these interfaces. They compose with `for`, `use`, sort functions, and every other piece of the stdlib that expects them.
3. **Worked example (from the Expedia SDK):**
   ```kotlin
   abstract class Paginator<T : PaginatedResponse<*, *>> : Iterator<T> {
       protected var hasNext = true
       override fun hasNext(): Boolean = /* ... */
   }
   ```
   `Paginator : Iterator<T>` lets every caller use `while (paginator.hasNext())` and `for (page in paginator)`. No custom `nextPage()` contract.
4. **Anti-pattern:** rolling your own `Disposable` when `AutoCloseable` works. Rolling your own `Stream` when `Sequence`/`Flow` works. Rolling your own `Either` when sealed `Result<T, E>` works.

**Enforcement:** review; implement the stdlib contract (`Iterator`, `Comparable`, `AutoCloseable`, `Sequence`/`Flow`) rather than a parallel interface.

### 7.13 — Pipeline composition via list-of-steps + `fold`.

**Reasoning, step by step:**
1. When you have a transformation that's the composition of several small steps, model it as a `List<Step>` and fold the input through it.
2. Pattern (from the Expedia SDK):
   ```kotlin
   fun interface RequestStep { operator fun invoke(req: Request): Request }

   class Pipeline(private val steps: List<RequestStep>) {
       fun run(initial: Request): Request = steps.fold(initial) { req, step -> step(req) }
   }
   ```
3. Benefits: each step is independently testable, the pipeline can be reordered or extended without inheritance, and the composition is explicit.
4. Use this in place of: template-method patterns, "abstract base classes with overridable hooks," chains of decorators that each subclass the previous.
5. Bound the list. `List<RequestStep>` of size 200 means 200 function calls per request — measure if you suspect this is hot.

**Enforcement:** review; compose linear step sequences as list-of-steps + `fold`, not template-method or decorator chains.

### 7.14 — Trailing lambdas and SAM conversion.

**Reasoning, step by step:**
1. When the last parameter of a function is a lambda, the call site moves it outside the parentheses: `xs.map { it * 2 }` rather than `xs.map({ it * 2 })`.
2. When the only parameter is a lambda, drop the parentheses: `synchronized { ... }`.
3. Kotlin allows SAM conversion for *Java* functional interfaces only by default. For Kotlin-defined `fun interface`, SAM conversion is enabled — prefer `fun interface` over a regular interface when there's exactly one abstract method.
4. **Anti-pattern:** trailing lambda on a function whose last parameter *isn't* meant to be lambda-style. If the lambda would obscure the call, write `f({ ... })` or named arguments.

**Enforcement:** review; `fun interface` for single-method types so SAM conversion applies, trailing-lambda syntax only where the last parameter is lambda-shaped.

## Cross-references

- Decision rule for scope functions is repeated in chapter 05 §5.7 — both stay in sync.
- `inline` and how it interacts with `crossinline`/`noinline` lambdas: chapter 05 §5.6.
- `fun interface` and ABI: chapter 10 (API Design).
- Sealed-hierarchy + exhaustive `when`: chapter 08 (Error Handling).
