# 05 — Functions

Functions are the unit of reasoning. Most rules here exist to keep that unit small, named, and side-effect-honest.

## Rules

### 5.1 — One function, one purpose. 60 lines is the ceiling.

**Reasoning, step by step:**
1. A function should do *one* thing — described by its name without "and."
2. Hard cap: 60 lines including blanks and the closing brace. Aim 15–30.
3. If a function exceeds the cap, extract a private helper, a `when` over a sealed type, or a `Sequence` pipeline. The name of the extracted helper documents the section you removed.
4. Top-level functions, member functions, and lambdas all count. A 60-line lambda is the same problem as a 60-line method.
5. KDoc lines don't count.

### 5.2 — Prefer expression bodies for single expressions; block bodies otherwise.

**Reasoning, step by step:**
1. `fun double(x: Int): Int = x * 2` is unambiguous. The body is the return value.
2. The moment a body needs a local `val`, two `when` arms with side effects, or sequential statements, use `{ ... return ... }`.
3. Public-API expression bodies still declare return types explicitly. Inference is allowed in private helpers if it's obvious.
4. **Anti-pattern:** stuffing a multi-step body into `run { ... }` to keep using `=`. See [01-formatting-and-tooling.md §1.4](./01-formatting-and-tooling.md).

### 5.3 — Default arguments over overloads.

**Reasoning, step by step:**
1. Kotlin has default arguments. Java's overload-explosion antidote is to use them.
2. `fun connect(host: String, port: Int = 443, timeout: Duration = 10.seconds)` replaces three overloads.
3. Callers use named arguments to skip middle defaults: `connect(host, timeout = 5.seconds)`.
4. **Caveat on JVM:** default args don't directly create Java overloads — add `@JvmOverloads` if Java callers exist. See [JVM guide](../kotlin-jvm/01-java-interop.md).
5. **Beware** of using *mutable* default values like `mutableListOf()` — each call evaluates the default fresh, but if the default is a captured singleton, every caller shares state. Prefer `List<T> = emptyList()` patterns.

### 5.4 — Named arguments at every call site with more than two parameters of the same type, or any boolean.

**Reasoning, step by step:**
1. `connect("localhost", 8080, 10000, true, false)` — readers cannot tell what those mean.
2. `connect(host = "localhost", port = 8080, timeoutMillis = 10000, useTls = true, followRedirects = false)` is self-documenting.
3. **Hard rule:** booleans at call sites must be named. `setVisible(true)` is fine because the function name implies the parameter; `withRetries(true)` is not.
4. Adjacent same-typed parameters (`crop(image, 10, 20, 30, 40)` — which two are width/height?) must be named.
5. Lint: detekt's `NamedArguments` rule, threshold = 3 (or 2 for booleans).

### 5.5 — Extension functions: extend types you don't own; add coherent operations to types you do.

**Reasoning, step by step:**
1. Extensions are *not* polymorphic. `String.greet()` resolves at compile time based on the static type. This is by design.
2. Use extensions to (a) add operations to types you don't own (`String.parseIso()`), (b) provide infix or operator forms (`Duration.times(...)`), (c) attach domain-specific helpers without inflating the host class.
3. Don't use extensions to fake inheritance — if the operation needs polymorphism, it belongs as a method.
4. **Scope rule:** the smallest scope that compiles is the right one. Local extension inside a function > private file-level extension > top-level extension > internal extension > public extension. Public extensions become part of your API forever.
5. Don't extend types you fully own when a regular member would do — methods participate in inheritance, extensions don't.

### 5.6 — `inline` for higher-order functions in hot paths and where you need `reified`.

**Reasoning, step by step:**
1. `inline` copies the function body (and its lambda arguments) at every call site. It eliminates lambda allocation and call overhead.
2. Use it when: (a) the function takes a lambda and is called in a hot path; (b) you need `reified` type parameters; (c) the function is small enough that inlining is cheap (~10 lines of body).
3. Don't inline (a) large functions — every call site bloats; (b) functions without lambda parameters where the gain is marginal; (c) functions you'll refactor heavily — inlining ABI-couples callers to the body.
4. `crossinline` if you pass the lambda to another function that captures it. `noinline` to opt a specific lambda out of inlining (e.g., to store it in a variable). The compiler errors will tell you exactly when you need these.

### 5.7 — Scope functions: pick by intent.

**Reasoning, step by step:**
1. Kotlin's five scope functions look interchangeable; they aren't. Pick by *what should be returned* and *whether the body wants `this` or `it`*.
2. Decision table:

   | Function | Receiver in body | Returns       | Use for                                      |
   |----------|------------------|---------------|----------------------------------------------|
   | `let`    | `it`             | lambda result | nullable resolution, type transform           |
   | `run`    | `this`           | lambda result | grouped ops on receiver, return derived value |
   | `with`   | `this`           | lambda result | same as `run`, when receiver isn't a chain    |
   | `apply`  | `this`           | receiver      | configuring a freshly-created object          |
   | `also`   | `it`             | receiver      | side effect (logging, validation, mutation)   |

3. Common mistakes:
   - `apply` to compute a derived value (use `run`).
   - `let` to configure a builder (use `apply`).
   - `also` for a transformation (use `let`).
4. **Anti-pattern:** chaining `?.let { it.foo }?.let { it.bar }?.let { it.baz }`. Use `?.foo?.bar?.baz` directly.
5. **Anti-pattern:** scope functions to avoid declaring a local `val`. A named local is often clearer than a `with` block. Don't compress for compression's sake.

### 5.8 — `vararg` only when the call site genuinely varies. Otherwise take a `List`/`Iterable`.

**Reasoning, step by step:**
1. `fun greet(vararg names: String)` is convenient for ad-hoc calls (`greet("a", "b", "c")`).
2. It costs: each call allocates an array. Inside a hot loop, this is real.
3. Use `vararg` when (a) the call site varies between literals, and (b) the function is the call site's terminal point.
4. Otherwise take `List<T>` or `Iterable<T>`. Callers can `*names.toTypedArray()` or `listOf(...)` as they prefer.

### 5.9 — Side-effects out of the body where you can. Pure functions are easier to test.

**Reasoning, step by step:**
1. A pure function: same input → same output, no observable side effect. Pure functions are easier to test, parallelize, and reason about.
2. Push side effects (I/O, logging, state mutation) to the *edges* — typically a thin shell function that calls pure helpers.
3. **Sample shape:**
   ```kotlin
   // shell — does the I/O
   suspend fun loadAndProcess(id: UserId): Result<Report, LoadError> {
       val raw = http.fetch(id).getOrElse { return Result.Err(it) }
       val report = buildReport(raw)              // pure
       audit.log(id, report)                      // side effect, separated
       return Result.Ok(report)
   }

   // pure core — trivially testable
   private fun buildReport(raw: RawData): Report = /* ... */
   ```
4. This is not religion. A logger call inside a function is fine. A function that does I/O, parsing, validation, and persistence is not.

### 5.10 — Function references over wrapping lambdas.

**Reasoning, step by step:**
1. `xs.map(::parseInt)` is clearer than `xs.map { parseInt(it) }`.
2. Member references work: `xs.map(String::length)`. Bound references too: `xs.map(parser::parse)`.
3. Use lambdas when the body does *more* than the reference would: arg reordering, partial application, additional logic.

### 5.11 — Tail-recursive only with `tailrec`, and only when iteration is genuinely worse.

**Reasoning, step by step:**
1. The JVM has no native tail-call elimination. Naive recursion stack-overflows on deep inputs.
2. Kotlin's `tailrec` modifier rewrites tail-recursive functions to loops at compile time. Use it.
3. **Rule:** no recursion in library code without `tailrec`. With `tailrec`, prove the recursive call is genuinely in tail position (the compiler will tell you if it isn't).
4. Most "tail-recursive" candidates read better as a `while` loop or a `fold`. Reach for `tailrec` only when the recursive structure mirrors the problem (tree walks, parser combinators).

### 5.12 — `Unit` return is implicit; don't write `: Unit`.

**Reasoning, step by step:**
1. `fun log(msg: String) { println(msg) }` returns `Unit` by inference. Writing `: Unit` is noise.
2. **Exception:** suspend functions where the explicit type aids reading. Even then, omit it unless the reader benefits.
3. `Unit` is the value, not the absence of one — you can pass it around. You rarely should.

## Cross-references

- `inline` performance trade-offs: chapter 15 (Performance).
- Scope functions in detail (with the Expedia SDK pattern): chapter 07 (Kotlin Idioms).
- Extension functions on JVM and `@JvmName` mangling: [JVM guide chapter 01](../kotlin-jvm/01-java-interop.md).
