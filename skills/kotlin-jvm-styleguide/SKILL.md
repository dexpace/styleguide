---
name: kotlin-jvm-styleguide
description: Use when writing, editing, or reviewing Kotlin that runs on the JVM (Spring, Ktor, JPA, Gradle) in a dexpace project — extends kotlin-styleguide with JVM-specific rules (Java interop, virtual threads vs coroutines, framework wiring, persistence). Use alongside kotlin-styleguide, not instead of it.
---

# Kotlin-on-JVM styleguide

## When this applies

Editing server-side Kotlin on the JVM, or reviewing it. Triggers: Spring/Ktor imports, `@Configuration`/`@Entity`/`@Transactional`/`@Service`, `build.gradle.kts` and version catalogs, SLF4J/`kotlin-logging`, JPA repositories, coroutine-vs-Loom decisions. Priority: **correctness > performance > developer experience.**

## Inherited

First apply `kotlin-styleguide`; this layer adds, and where stricter overrides, for Kotlin on the JVM. Target Kotlin 2.3, JDK 21+.

## Language hard rules

- Public Kotlin is a stable Java ABI: every `public` symbol is reflection-targetable bytecode. Reach for `internal` aggressively; guard published modules with `binary-compatibility-validator`.
- Interop annotations are contract, not decoration. Add `@JvmStatic`/`@JvmOverloads`/`@JvmName`/`@file:JvmName`/`@Throws` only where a Java or reflection caller (Spring, Jackson, JPA) needs them; omit them in pure-Kotlin code. `@JvmField` only when the Java side requires a real field.
- Platform types are unknowns: pin every Java-boundary value to an explicit `String` or `String?`; never let `String!` leak past the adapter. Annotate owned Java with JSpecify; compile with `-Xjsr305=strict`.
- Framework reflection needs compiler plugins, not hand-written `open`/no-arg constructors: `kotlin-spring`, `kotlin-jpa`, `kotlin-allopen`. `lateinit var` only for framework injection, with KDoc naming the injector.
- Concurrency: coroutines for new business logic; virtual threads (Loom) for cancellation-free blocking I/O; Reactor only at framework boundaries, bridged once. `Mono`/`Flux` never reach the domain. Blocking JDK calls go in `withContext(Dispatchers.IO)`. `runBlocking` only in `main` or tests. Propagate MDC with `MDCContext()`.
- Frameworks: constructor injection only, never field injection; `@Configuration` + `@Bean` for wiring, `@Component`/`@Service` for behavior; `@ConfigurationProperties` as an immutable validated `data class`; `@Transactional` on the public service unit of work, never self-called or on repositories; controllers parse-delegate-render only.
- Persistence: JPA entities are regular classes (not `data class`), ID-or-business-key equality, `LAZY` associations; transactions span the use case; map to DTOs at the edge and never expose entities; repositories follow `find*`/`count*`/`exists*`/`delete*`, drop to `@Query` past two derived properties.
- Serialization: `kotlinx.serialization` for internal-controlled JSON, Jackson with `jackson-module-kotlin` at Spring boundaries; `java.time.*` only, ISO-8601 not timestamps; pick `null`-vs-absent semantics per field; validate at the deserialization boundary with `@Valid`.
- Logging: SLF4J via `kotlin-logging`, one `private val logger` per file; lazy `{ }` message blocks; structured key/value events; correlation in MDC; mask PII; log a failure once at the handling boundary; no `println`/`System.err`.
- Build: Gradle Kotlin DSL, version catalogs, pinned toolchains, `allWarningsAsErrors`, strict interop flags; `binary-compatibility-validator` on published modules; test on JDK 21+ matching production.

## Before you finish — verify

```bash
./gradlew ktlintCheck detekt   # style + static analysis, must pass clean
./gradlew build                # compiles with allWarningsAsErrors
./gradlew test                 # JDK 21+ toolchain
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/README.md)
- [01 Java Interop](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/01-java-interop.md)
- [02 JVM Concurrency](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/02-jvm-concurrency.md)
- [03 JVM Frameworks](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/03-jvm-frameworks.md)
- [04 Persistence](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/04-persistence.md)
- [05 Serialization](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/05-serialization.md)
- [06 Logging](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/06-logging.md)
- [07 JVM Performance](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/07-jvm-performance.md)
- [08 Build & Distribution](https://github.com/dexpace/styleguide/blob/main/kotlin-jvm/08-build-and-distribution.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
