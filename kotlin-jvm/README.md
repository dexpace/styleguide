## Kotlin-JVM Code Style

Binding style rules for **server-side Kotlin on the JVM**, target **Kotlin 2.3** and **JDK 21+**. This guide *extends* the generic [Kotlin guide](../kotlin/). Where the JVM guide adds a stricter rule, it wins for JVM code; the generic guide remains the canonical Kotlin baseline.

This guide exists because the JVM has *its own* contract. Reflection, bytecode-level ABI, framework conventions (Spring/JPA/Ktor), the Java standard library, Loom, and the JVM's GC all impose rules that the generic Kotlin guide can't address.

### Style Priorities

Same as the generic guide:

1. **Clarity**
2. **Simplicity**
3. **Concision**
4. **Maintainability**
5. **Consistency**

Plus one JVM-specific tiebreaker: **Java callers don't read Kotlin documentation.** If a public Kotlin symbol is consumed from Java or reflectively (Spring, Jackson, JPA), the Java-side ergonomics drive the design.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Java Interop](./01-java-interop.md) | `@Jvm*` annotations, platform types, null annotations, file-level annotations, mangled internal names |
| 02 | [JVM Concurrency](./02-jvm-concurrency.md) | Virtual threads (Loom), coroutines vs Loom, Reactor at boundaries, `CompletableFuture` bridges, `ThreadLocal`/MDC + coroutines |
| 03 | [JVM Frameworks (Spring/Ktor)](./03-jvm-frameworks.md) | Constructor injection, no field injection, `@Configuration` over `@Component`, `@ConfigurationProperties`, transactional boundaries |
| 04 | [Persistence](./04-persistence.md) | JPA + `kotlin-jpa`, entity-vs-data-class, `LAZY` defaults, equality by business key, Exposed/jOOQ when SQL-first is right |
| 05 | [Serialization on JVM](./05-serialization.md) | kotlinx.serialization vs Jackson, `kotlin-module`, time types, never expose entities, null-vs-absent JSON semantics |
| 06 | [Logging on JVM](./06-logging.md) | SLF4J + kotlin-logging, lazy messages, MDC for correlation (with coroutines), PII masking, no `println` |
| 07 | [JVM Performance](./07-jvm-performance.md) | JFR, async-profiler, escape analysis, value-class boxing, GC awareness, GraalVM caveats |
| 08 | [Build & Distribution](./08-build-and-distribution.md) | Gradle Kotlin DSL, version catalogs, toolchains, `binary-compatibility-validator`, shadow jars, GraalVM native-image |

---

## Top-level JVM rules

These add to the 15 rules in the [generic guide root](../kotlin/README.md). When the rules below conflict with the generic ones (they shouldn't, but in edge cases like `lateinit` for Spring DI), the JVM rule wins for JVM code.

### JVM-1. Public Kotlin is a stable Java ABI.

**Step-by-step reasoning:**
1. Every `public` Kotlin symbol becomes a Java-callable, reflection-targetable surface in the compiled bytecode.
2. Renaming, reordering parameters, or changing nullability is a binary-incompatible change. Java callers, Spring's reflection, Jackson's `kotlin-module`, JPA's bytecode-weaving — all see the *bytecode*, not your source.
3. Reach for `internal` aggressively (generic guide §10.1). For modules that publish to other modules or external consumers, enforce stability with `binary-compatibility-validator` in CI.
4. New public symbols are easier to add than to remove. Don't ship symbols you wouldn't bet your refactor on.

### JVM-2. Annotate for the Java caller — when there is one.

**Step-by-step reasoning:**
1. Each interop annotation is a stylistic choice with real cost. Sprinkle them on every symbol and you bloat bytecode, lose abstractions (`@JvmField` strips the property), and signal that you didn't think about your callers. Omit them where they're needed and Java callers either can't call you or call you awkwardly.
2. Java callers can't pass named arguments. `fun f(a: Int = 1, b: Int = 2)` is callable from Java *only* as `f(a, b)` unless you add `@JvmOverloads`, which generates the overloads (`f()`, `f(a)`, `f(a, b)`).
3. Companion-object methods need `@JvmStatic` to be callable as `Type.method(...)` from Java instead of `Type.Companion.method(...)`.
4. Top-level functions live in `<FileName>Kt` from Java's view. Use `@file:JvmName("Util")` to name the class.
5. **The annotations are not decoration — they're how Java sees the contract. Missing them in Java-consumed code is a *bug*; adding them in pure-Kotlin code is noise.** First question: does this symbol have a Java caller (including framework reflection that emulates one — Spring, Jackson, JPA)? If yes, annotate deliberately. If no, leave them off.
6. See [chapter 01](./01-java-interop.md) for the per-annotation guidance.

### JVM-3. Platform types are unknowns — resolve them at the boundary.

**Step-by-step reasoning:**
1. A value coming from Java without nullability metadata has *platform type* `String!` — neither `String` nor `String?`. Kotlin lets you use it either way and *won't catch the NPE*.
2. At the Kotlin-Java boundary, every value gets either (a) an explicit type ascription (`val name: String = javaThing.getName()`, which throws on null), (b) a nullable-aware coercion (`val name = javaThing.getName() ?: error("...")` or `?.let { ... }`), or (c) an `@Nullable`/`@NotNull` annotation on the Java side that disambiguates.
3. Don't let platform types leak past the adapter layer. Internal code should see only proper Kotlin types.
4. Lint: detekt's `PlatformType` rule warns on functions whose return type is platform.

### JVM-4. Framework reflection needs compiler plugins, not workarounds.

**Step-by-step reasoning:**
1. Spring proxies require `open` classes/methods. JPA entities require a no-arg constructor. Hibernate requires `open` for lazy loading.
2. The wrong fix: hand-writing `open class ...` and `constructor()` constructors throughout. It rots, gets forgotten on new code, and lies about the inheritance contract.
3. The right fix: compiler plugins — `kotlin-spring` (opens `@Component`/`@Service`/`@Configuration`/etc.), `kotlin-jpa` (no-arg for `@Entity`/`@Embeddable`/`@MappedSuperclass`), `kotlin-noarg`/`kotlin-allopen` (configurable variants).
4. The plugins are the contract. They run at compile time and produce exactly what the framework expects. Configure them once; don't write hand-rolled workarounds.

### JVM-5. Pick the right concurrency primitive per JVM workload.

**Step-by-step reasoning:**
1. JDK 21+ has virtual threads (Project Loom). The Kotlin coroutine ecosystem still exists and isn't going away.
2. Decision:
   - **Coroutines** — cancellation-aware, structured concurrency, `Flow`, integrates with `kotlinx-coroutines-*`. Use when your code is `suspend`-shaped or you need cooperative cancellation.
   - **Virtual threads** — for *blocking* I/O without cancellation semantics. Cheap to spawn, you write code as if it were synchronous. Use via `Executors.newVirtualThreadPerTaskExecutor()` or as a coroutine dispatcher (`asCoroutineDispatcher()`).
   - **Reactor** — only at boundaries with frameworks that demand it (Spring WebFlux, certain reactive libraries). Bridge to coroutines with `kotlinx-coroutines-reactor`.
3. Never mix without naming the seam. A method that returns `Mono<T>` to a caller that wants `Flow<T>` is a boundary — convert at exactly one point.
4. See [chapter 02](./02-jvm-concurrency.md) for the full decision tree.

### JVM-6. JVM resources are not Kotlin resources.

**Step-by-step reasoning:**
1. Threads, classloaders, connection pools, native file handles, large off-heap buffers — these outlive `use { }` blocks. The GC doesn't help.
2. Bind their lifecycle to a *named* lifecycle: a Spring `DisposableBean`, a Ktor `monitor.subscribe(ApplicationStopping)`, a coroutine `SupervisorJob` with explicit `cancel()` on shutdown.
3. Test it: kill -SIGTERM the running process and verify clean shutdown logs.
4. Leaks here become OOMs and file-descriptor exhaustion in production, not GC pressure.

---

Zero technical debt holds here as everywhere: what ships meets the design goals. **Perfection over technical debt — debt never gets paid.** A JVM service runs unattended for months; the platform-type leak or hand-rolled `open` shortcut taken today is the NPE or framework break someone else debugs in production.

## Influences

- **[Kotlin documentation: Java interop](https://kotlinlang.org/docs/java-interop.html), [Calling Kotlin from Java](https://kotlinlang.org/docs/java-to-kotlin-interop.html)** — canonical interop reference.
- **[Spring Boot reference, Kotlin section](https://docs.spring.io/spring-boot/reference/features/kotlin.html)**.
- **[JEP 444 (Virtual Threads)](https://openjdk.org/jeps/444)** and `kotlinx-coroutines` docs on `Dispatchers.IO` vs virtual threads.
- **[Kotlin binary-compatibility-validator](https://github.com/Kotlin/binary-compatibility-validator)**.
