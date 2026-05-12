# 01 — Java Interop

If a public Kotlin symbol can be called from Java, *its bytecode* is the API. The Kotlin source is just one of two views.

## Rules

### 1.1 — `@JvmStatic` on companion-object methods Java callers will use.

**Reasoning, step by step:**
1. Without `@JvmStatic`, a companion-object method `fun parse(s: String): Foo` is callable from Java as `Foo.Companion.parse(s)` — verbose and unidiomatic.
2. With `@JvmStatic`, Java sees `Foo.parse(s)` — exactly what Java callers expect of a static factory.
3. Apply to: factory methods (`fromJson`, `of`, `create`), constants needed as fields (paired with `@JvmField`).
4. **From the Expedia SDK:**
   ```kotlin
   companion object {
       @JvmStatic
       @Throws(IllegalArgumentException::class)
       fun fromCode(code: Int): Status = entries.find { it.code == code }
           ?: throw IllegalArgumentException("Invalid status code: $code")
   }
   ```
5. **Cost:** `@JvmStatic` generates an extra method on the enclosing class delegating to the companion. ABI-stable; cost is negligible.

### 1.2 — `@JvmOverloads` on default-argument functions exposed to Java callers or reflection.

**Reasoning, step by step:**
1. Kotlin default arguments do *not* generate Java overloads. `fun f(a: Int, b: Int = 0)` is callable from Java only as `f(a, b)`.
2. `@JvmOverloads` generates the overload set: `f(a)`, `f(a, b)`. Required for Java callers and for frameworks that pick constructors reflectively.
3. Apply to: constructors of Spring/JPA classes with default args, public functions in libraries.
4. **Anti-pattern:** sprinkling `@JvmOverloads` on every default-arg function. Apply only where Java/reflection callers exist.
5. Verify with `javap -p`. The generated overloads should be exactly what you expect.

### 1.3 — `@JvmField` is a limited tool, not a default. Reach for it only when Java demands a field.

**Reasoning, step by step:**
1. A Kotlin `val name: String` already does the right thing for Java callers: it compiles to a private field + a `getName()` getter, which Java calls as `obj.getName()`. **That's the idiomatic shape.** Don't reflexively `@JvmField` to make Java code look "nicer" — `obj.getName()` is exactly what Java code is supposed to look like.
2. `@JvmField val name: String` *replaces* the getter with a public field. Java sees `obj.name`. The cost: you've removed the property abstraction. No future custom getter. No `val` → `var` migration without breaking every Java caller. No `lateinit`. No `open`. No delegation. No `@Volatile` semantics. You traded a one-time syntactic win for permanent inflexibility.
3. **Legitimate uses** (all driven by something on the Java side requiring a real field):
   - A serializer / DI framework that reads public fields by reflection and doesn't follow getters.
   - A framework annotation that targets fields (`@JvmField @Volatile var ...` to satisfy a memory-model contract).
   - Performance-critical hot paths where the getter call is measurably the bottleneck (rare).
4. **Anti-pattern:** `@JvmField` on every property of a data class for "interop." Default Kotlin properties + generated getters work fine for Java.
5. Inside `companion object`: `@JvmField` exposes a constant as a static *field* (`MyClass.CONSTANT` from Java instead of `MyClass.Companion.getCONSTANT()`). `@JvmStatic` exposes a method as a static method. Pick by whether the Java side reads a field or calls a method.

### 1.4 — `@JvmName` to give Kotlin and Java distinct names for the same function.

**Reasoning, step by step:**
1. Kotlin allows function overloads that differ only by return type when generics erase the same way. Java does not — the bytecode names collide.
2. `@JvmName` gives the Java side a different name: `@JvmName("filterValid") fun List<X>.filter(...)`.
3. Also useful for extension functions that would otherwise have ugly mangled names.
4. **Pattern:** keep the Kotlin call site idiomatic; rename the Java call site for clarity.

### 1.5 — `@file:JvmName` to name the synthetic class for top-level declarations.

**Reasoning, step by step:**
1. Top-level functions in `Util.kt` compile to a `UtilKt` class — Java callers do `UtilKt.foo(...)`. Ugly.
2. `@file:JvmName("Util")` at the top of `Util.kt` renames the class to `Util`. Java does `Util.foo(...)`.
3. Apply to every file with public top-level declarations consumed by Java.
4. `@file:JvmMultifileClass` combines multiple Kotlin files into one Java-visible class — useful for splitting a large utility class across files.

### 1.6 — Platform types: pin them at the boundary.

**Reasoning, step by step:**
1. A Java method returning `String` is typed in Kotlin as `String!` — *platform type*. The Kotlin compiler will let you assign it to `String` or `String?` without complaint.
2. Platform types are silent NPEs waiting to happen. The compiler doesn't catch them.
3. **Rule:** at every Java-Kotlin boundary, give the value an *explicit* type. `val name: String = javaThing.getName()` throws NPE on null at the assignment line (loud, debuggable) instead of three calls later. `val name: String? = javaThing.getName()` says you'll handle null.
4. Prefer the latter when the Java side genuinely returns null in any flow. Prefer the former with a follow-up `?: error("...")` when null is a contract violation.
5. **Lint:** detekt's `PlatformType` rule on `internal`/`public` function return types.

### 1.7 — JSR-305 / Jakarta nullability annotations on Java boundary code.

**Reasoning, step by step:**
1. Java code can express nullability via annotations: `@Nullable`, `@NonNull` (from `org.jetbrains.annotations`, JSR-305, JSpecify, etc.).
2. Kotlin reads these annotations and tightens the platform type accordingly. `@Nullable String` becomes `String?`; `@NotNull String` becomes `String`.
3. If you own the Java side too, annotate it. The Kotlin compiler then enforces nullability across the boundary.
4. Recommended annotation source for new code: **JSpecify** (`org.jspecify.annotations`). It's the emerging standard and Kotlin 2.x supports it well.
5. Configure the compiler with `-Xjsr305=strict` (or the JSpecify equivalent) to treat unmarked Java types as platform types but warn or error on misuse.

### 1.8 — `@Throws` for checked exceptions consumed by Java callers.

**Reasoning, step by step:**
1. Kotlin has no checked exceptions. Java does. A Kotlin function that throws `IOException` does *not* require Java callers to `try/catch` unless annotated.
2. `@Throws(IOException::class)` adds the exception to the bytecode signature. Java now requires handling.
3. Apply to: functions explicitly designed to throw checked exceptions that Java callers should handle.
4. Don't apply to: every `throw` site. Kotlin's design is that callers handle exceptions where they make sense, not where the language compels them to.
5. **From the Expedia SDK:**
   ```kotlin
   @JvmStatic
   @Throws(IllegalArgumentException::class)
   fun fromCode(code: Int): Status = /* ... */
   ```

### 1.9 — `internal` and Java: the name is mangled.

**Reasoning, step by step:**
1. `internal fun foo()` compiles to a public method with a mangled name like `foo$module_name`. The mangling is what makes "module visibility" work at the JVM level.
2. Java code in the same module *can* call `internal` Kotlin functions, but it must use the mangled name — unpleasant and brittle.
3. **Rule:** don't expose `internal` symbols to Java. If a Java caller needs it, make it `public` and document the contract; if it's truly module-private, keep Java out.
4. Cross-module Java callers cannot call `internal` Kotlin code. That's the whole point.

### 1.10 — Interface default methods: `-Xjvm-default=all` for new modules, `all-compatibility` for published libraries.

**Reasoning, step by step:**
1. Kotlin interfaces can have default method bodies. The compiler has three modes for emitting them:
   - **`disable`** (legacy default): no Java default methods; bodies live in a `DefaultImpls` inner class and are copied into every implementing class. Largest bytecode footprint; Java implementers must re-implement.
   - **`all`**: bodies compiled as proper Java 8+ default methods. Java implementers don't re-implement. ABI change vs `disable`.
   - **`all-compatibility`**: both — default methods *and* `DefaultImpls`. Largest of the three but ABI-compatible with code compiled under `disable`.
2. **Recommendation for new code:** `-Xjvm-default=all`. Smaller bytecode, idiomatic interop with Java 8+, the modern default for Kotlin 2.x projects.
3. **Recommendation for published libraries** consumed by code that may have been compiled against `disable`: `all-compatibility`. The ABI compatibility shim is worth the size.
4. Apply at the *module* level (in `compileOptions.freeCompilerArgs` or the `kotlin { compilerOptions { ... } }` DSL). Don't sprinkle `@JvmDefault` annotations — those were deprecated in favor of the compiler flag.
5. Verify with `javap` on a representative interface after a build to confirm default methods are present.

### 1.11 — Constructors with default args + Spring/JPA: pair with `@JvmOverloads` and the compiler plugin.

**Reasoning, step by step:**
1. Spring/JPA reflectively pick a constructor. They often want the no-arg one or a specific signature.
2. `data class User(val id: Long = 0, val email: String = "")` compiles to a constructor with two parameters. Spring's reflection won't find a no-arg one without `@JvmOverloads`.
3. **The right fix:** use the `kotlin-jpa` plugin (no-arg constructor for `@Entity`-annotated classes) or `kotlin-noarg` (configurable). The plugin generates the no-arg constructor at compile time without it being visible from Kotlin code.
4. **The wrong fix:** writing `constructor() : this(0, "")` by hand. Rots, gets forgotten on new classes, and lies about the data class's invariants.
5. See [chapter 04](./04-persistence.md).

### 1.12 — `lateinit var` for framework injection only, with KDoc explaining the lifecycle.

**Reasoning, step by step:**
1. Spring/JUnit reflectively inject after construction. The Kotlin language gives `lateinit var` for exactly this case.
2. Generic guide §3.6 forbids `lateinit` for "I'll set it later in my own code." That ban stands. Framework injection is the documented exception.
3. KDoc the lifecycle: `/** Injected by Spring. Available after bean initialization. */`.
4. Prefer constructor injection where the framework supports it (Spring does, since 4.3). `lateinit` is the fallback when the framework forces field injection (e.g., legacy `@Autowired` on a field that can't be in the constructor).

## Worked example

```kotlin
@file:JvmName("PaymentClients")

package com.acme.payments

// Java callers see: PaymentClients.create(http) and PaymentClients.create(http, clock)
@JvmOverloads
fun create(http: HttpClient, clock: Clock = Clock.systemUTC()): PaymentClient =
    PaymentClient(http, clock)

class PaymentClient @JvmOverloads constructor(
    private val http: HttpClient,
    private val clock: Clock = Clock.systemUTC(),
) {
    companion object {
        @JvmStatic
        @Throws(IllegalArgumentException::class)
        fun fromConfig(config: Config): PaymentClient = /* ... */

        @JvmField
        val DEFAULT_TIMEOUT: Duration = 10.seconds.toJavaDuration()
    }
}
```

## Cross-references

- Nullability and platform types: [generic guide ch. 03](../kotlin/03-nullability.md).
- Compiler plugins for Spring/JPA: [ch. 03](./03-jvm-frameworks.md), [ch. 04](./04-persistence.md).
- ABI stability checks: [ch. 08](./08-build-and-distribution.md).
