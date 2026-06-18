# Kotlin-on-JVM styleguide — deep-review checklist

Additive to `kotlin-styleguide`. Apply the core Kotlin checklist first; these chapters add JVM rules and, where stricter, win for JVM code. One heading per `kotlin-jvm` chapter.

### 01 — Java Interop

- `@JvmStatic` on companion factories Java calls; otherwise Java sees `Type.Companion.method(...)`.
- `@JvmOverloads` on default-argument functions/constructors reached by Java or framework reflection; Kotlin defaults generate no Java overloads.
- `@JvmField` only when the Java side needs a real field; default `val` + generated getter is the idiomatic shape. No `@JvmField` blanket on data-class properties.
- `@JvmName` for return-type-only overloads and mangled extensions; `@file:JvmName` so top-level functions don't land in `<File>Kt`.
- Pin platform types at every Java boundary to explicit `String`/`String?`; never let `String!` leak past the adapter. Lint with detekt `PlatformType`.
- Annotate owned Java boundary code with JSpecify nullability; compile `-Xjsr305=strict`.
- `@Throws` only on functions designed for Java to catch, not every throw site.
- Don't expose `internal` to Java — its name is mangled; make it `public` with a contract or keep Java out.
- Set `-Xjvm-default=all` (`all-compatibility` for published libraries) at module level; no scattered `@JvmDefault`.
- `@Entity`/`@Embeddable` with default args rely on `kotlin-jpa`/`kotlin-noarg`, never a hand-written no-arg constructor.
- `lateinit var` for framework injection only, with KDoc naming the injector and lifecycle; prefer constructor injection.

### 02 — JVM Concurrency

- Default to coroutines (`suspend` + structured scopes) for new async business logic.
- Virtual threads (Loom) only for cancellation-free blocking I/O; never spawn inside a coroutine scope.
- Wrap blocking JDK calls in `withContext(Dispatchers.IO)`; switch in for the call, out for downstream work; never run them on `Dispatchers.Default`.
- Reactor only at framework boundaries; convert `Mono`/`Flux` to `suspend`/`Flow` at the one boundary; never propagate them through the domain.
- `CompletableFuture` bridges (`scope.future { }`) carry an owning scope; no `GlobalScope.future`.
- `runBlocking` only in `main` or tests; request handlers are `suspend fun`. WebFlux + `runBlocking` is a hard fail.
- Propagate MDC across suspensions with `withContext(MDCContext())`; other thread-locals via `asContextElement()` or explicit parameters.
- Cancellation crosses coroutine boundaries, not thread boundaries; wrap blocking calls that must cancel in `runInterruptible`.
- In suspend functions use `Mutex.withLock`, not `synchronized`; bound every lock hold time.
- Bridge a managed executor with `asCoroutineDispatcher()` and `close()` it at end of lifecycle.

### 03 — JVM Frameworks (Spring / Ktor)

- Constructor injection only. No `@Autowired` fields, ever. Lint with detekt `SpringFieldInjection`.
- `kotlin-spring` plugin is mandatory; never hand-write `open` on Spring components.
- `@Configuration` + `@Bean` for infrastructure wiring and third-party objects; `@Component`/`@Service` for business behavior. Don't blur them.
- `@ConfigurationProperties` as an immutable `data class` with constructor binding; `@Validated` to fail fast at startup.
- `@Transactional` on the public service method that is the unit of work — not repositories, controllers, or self-calls (self-calls bypass the proxy).
- Controllers are thin: parse, delegate, render; no domain rules or persistence in the handler.
- WebFlux controllers are `suspend fun` returning `Flow`; `Mono`/`Flux` confined to the signature.
- Ktor: one `Application.<feature>Routes()` per feature; install shared plugins in the top-level `module()` first; wire dependencies by constructor, resolved once, never inside handlers.
- Bean lifecycle hooks via `@PostConstruct`/`@PreDestroy`; `InitializingBean`/`DisposableBean` only with a recorded reason.
- `@ControllerAdvice` maps each domain exception to a status; don't rely on default Spring error handling.
- Singleton beans hold only `val` immutable state, or guard mutable state with a concurrency primitive.

### 04 — Persistence

- `kotlin-jpa` plugin mandatory wherever `jakarta.persistence` is on the classpath; `kotlin-allopen` for proxy-friendly entities. No manual `open`/no-arg.
- Entities are regular classes, not `data class`; equality by stable business key or by ID after rejecting transient instances; `hashCode` stable, not based on a mutable ID.
- All associations `LAZY`; `EAGER` is opt-in, justified, documented. Fix `LazyInitializationException` by accessing inside the transaction, not by switching to `EAGER`.
- DTOs at the API boundary; never expose entities. Explicit mapper at the edge.
- Repositories are Spring Data interfaces following `find*`/`count*`/`exists*`/`delete*`; use `@Query` past two derived properties; collections return non-null `List<T>`.
- `@Transactional` spans the use case on the service method, never on a repository interface.
- Exposed or jOOQ when SQL is the model; one persistence engine per module, recorded in an ADR.
- Connection-pool size is sized from the bottleneck and documented with its rationale; wire HikariCP metrics.
- Migrations as code (Flyway/Liquibase), forward-only in production, tested on a fresh DB; `hbm2ddl.auto` never `update`/`create` in production.

### 05 — Serialization on JVM

- `kotlinx.serialization` for internal-controlled JSON; Jackson with `jackson-module-kotlin` at Spring and JVM-ecosystem boundaries.
- `jackson-module-kotlin` mandatory if Jackson is on the classpath; round-trip-test missing optional fields so defaults fire.
- `java.time.*` only (`Instant`/`OffsetDateTime`/`LocalDate`); ban `java.util.Date`; emit ISO-8601 strings, not timestamps (`WRITE_DATES_AS_TIMESTAMPS = false`).
- Domain DTOs are dedicated `data class`es; never serialize entities or domain objects directly.
- Pick `null`-vs-absent semantics per field (required / optional-with-default / optional-nullable) and document it; set strict flags (`FAIL_ON_UNKNOWN_PROPERTIES = false`, `FAIL_ON_NULL_FOR_PRIMITIVES = true`).
- `@Serializable` on serialized data classes; a `Json` instance per use case with strict flags; no shared `Json.Default`.
- Polymorphic deserialization via library discriminators (`classDiscriminator` / `@JsonTypeInfo`) on sealed hierarchies; no hand-rolled type dispatch.
- Validate at the deserialization boundary with `@Valid` + Bean Validation; map failures to 400 via `@ControllerAdvice`.
- Binary formats (Protobuf/Avro/CBOR) at high-volume or schema-strict boundaries, schema checked into source.
- Stream large payloads (`JsonParser`/`JsonGenerator`/`Flow<T>`); no whole-stream `readValue` of unbounded input.

### 06 — Logging on JVM

- SLF4J via `kotlin-logging`; ban `java.util.logging`, `commons-logging`, direct `LoggerFactory`.
- One `private val logger = KotlinLogging.logger {}` per file or companion; never instantiate per call or pass as a parameter.
- Lazy `{ }` message blocks or `{}` placeholders; never string-concat inside a `logger.*(...)` call.
- Levels mean what they say; keep INFO bounded (≤~10 lines/request).
- Structured key/value events via `atInfo().addKeyValue(...)`/MDC, not interpolated strings.
- Correlation context in MDC via `MDC.putCloseable(...).use { }`; bridge across suspensions with `withContext(MDCContext())`.
- Never log secrets or PII verbatim; mask at the source.
- No `println`/`System.err` in production code.
- One logging backend (Logback or Log4j 2), configured in YAML/Groovy/XML, not `.properties`.
- Cross-cutting decoration via a `LoggerDecorator` (interface delegation), MDC, or appender pattern, never inlined.
- Log a throwable once at its handling boundary, passed as a parameter; never on a rethrow path.

### 07 — JVM Performance

- Profile before optimizing (JFR / async-profiler / IntelliJ); a perf PR attaches before/after profiles, not a narrative.
- Measure steady-state with JMH; never `currentTimeMillis()` deltas around a single call.
- Trust escape analysis — small short-lived allocations are usually free; allocation "fixes" cite an async-profiler `alloc` trace.
- `@JvmInline value class` erases to a primitive; know the boxing cases (nullable, generic, interface boundary); switch to primitives only behind a profile.
- G1 by default; ZGC/Shenandoah only behind a sub-ms pause SLA; `-Xms == -Xmx` in containers; GC logging on in production.
- Set `ReservedCodeCacheSize` and `MaxMetaspaceSize` explicitly; monitor fill.
- Object pools rarely worth it; justify with a profile and use a vetted library, never hand-rolled.
- Treat JNI/JNA as foreign code: small wrapper, dedicated dispatcher, RSS monitored separately.
- Direct buffers released via `use {}`/`Cleaner`; bound `-XX:MaxDirectMemorySize` below `-Xmx`.
- GraalVM native-image viable with reflection/resource config; CI tests the native executable, not only JVM mode.
- No `String.intern()`; dedup via `-XX:+UseStringDeduplication` or a map; caches via Caffeine, not raw weak/soft references.
- Budget cold-start, warm-up, and a draining SIGTERM shutdown.

### 08 — Build & Distribution

- Gradle Kotlin DSL (`build.gradle.kts`); no Groovy in new projects; don't mix `*.gradle` and `*.gradle.kts`.
- Version catalogs (`libs.versions.toml`); no hard-coded versions in modules.
- Toolchains pin the JDK; don't depend on the developer's `JAVA_HOME`.
- Let the Kotlin plugin manage `kotlin-stdlib`; no explicit override.
- `binary-compatibility-validator` (`apiCheck`) in CI for published modules; an out-of-date `*.api` fails the build.
- Compiler flags in the build, not CI: `allWarningsAsErrors`, `-Xjsr305=strict`, `-Xexplicit-api=strict` for libraries, `-Xjvm-default=all`.
- Reproducible builds: strip archive timestamps, fixed file order, embed `Build-SHA` not build time.
- Shadow jars only in application modules, with explicit `relocate` on bundled internal deps; don't shadow libraries.
- Track GraalVM `reachability-metadata` JSON in source; CI builds native and JVM both.
- Test on JDK 21+ matching production exactly; matrix-test each supported JDK for multi-version libraries.
- Publishing: `maven-publish` + `signing` + Dokka, run only in CI with the key from secrets.
- CI gate: compile, test, ktlint, detekt, `apiCheck`, license check, vulnerability scan — all green before merge.
