# 03 — JVM Frameworks (Spring / Ktor)

Frameworks impose contracts above the language. The rules that follow exist because Spring and Ktor expect particular shapes from your code.

## What good looks like

```kotlin
@ConfigurationProperties(prefix = "payments")
data class PaymentsProperties(
    val gatewayUrl: String,
    val timeout: Duration = Duration.ofSeconds(5),
)

@Configuration
@EnableConfigurationProperties(PaymentsProperties::class)
class PaymentsConfig {
    @Bean // third-party object, built explicitly, no Kotlin annotations on the result
    fun gateway(props: PaymentsProperties): Gateway =
        HttpGateway(props.gatewayUrl, props.timeout)
}

@Service
class PaymentService(private val gateway: Gateway, private val orders: OrderRepository) {
    @Transactional // unit of work, called from the controller boundary — never self-called
    fun charge(orderId: OrderId, amount: Cents): Receipt {
        val order = orders.find(orderId) ?: throw OrderNotFoundError(orderId)
        return gateway.submit(order, amount)
    }
}

@RestController
class PaymentController(private val payments: PaymentService) {
    @PostMapping("/orders/{id}/charge")
    suspend fun charge(@PathVariable id: OrderId, @RequestBody body: ChargeRequest): ReceiptResponse =
        payments.charge(id, body.amount).toResponse() // thin: parse, delegate, render
}
```

Every dependency arrives through the primary constructor (3.1), so the container — and a plain test — can build each class with no reflection; `kotlin-spring` opens these `@Service`/`@Configuration`/`@RestController` classes for proxying (3.2). Infrastructure wiring lives in `@Configuration` + `@Bean` (3.3), config binds to an immutable `data class` (3.4), `@Transactional` sits on the service unit of work and is reached across the proxy boundary (3.5), and the controller stays thin (3.6) with a `suspend fun` signature (3.7).

## Rules

### 3.1 — Constructor injection only. No field injection. Ever.

**Reasoning, step by step:**
1. `@Autowired` on a field requires reflection, makes the dependency invisible at the constructor, and prevents the class from being instantiated outside the container.
2. Constructor injection makes dependencies explicit: `class PaymentService(private val gateway: Gateway, private val auditor: Auditor)`. The compiler enforces that nothing is missing.
3. Spring 4.3+ does not require `@Autowired` on a single-constructor class — Kotlin's primary constructor is the injection target.
4. Field injection survives in legacy code. Migrate when you touch the class.
5. **Lint:** detekt's `SpringFieldInjection` rule to error.

**Enforcement:** detekt `SpringFieldInjection` set to error; review rejects `@Autowired` on fields.

### 3.2 — Compiler plugin: `kotlin-spring` is mandatory.

**Reasoning, step by step:**
1. Spring proxies need `open` classes. `kotlin-spring` automatically opens `@Component`, `@Service`, `@Repository`, `@Controller`, `@Configuration`, and `@Bean`-annotated classes.
2. Without it, you either (a) write `open class` everywhere — Java muscle-memory, error-prone, easy to forget, or (b) ship a working class today that breaks when proxying is added (transactions, caching, security advice).
3. Configure once in the Gradle build: `plugins { kotlin("plugin.spring") }`.
4. Don't write `open class` manually for Spring components. Let the plugin do it.

**Enforcement:** build fails without the `kotlin-spring` plugin block; review forbids hand-written `open` on Spring components.

### 3.3 — `@Configuration` over `@Component` for wiring; `@Bean` for third-party objects.

**Reasoning, step by step:**
1. `@Configuration` classes are pure wiring — they declare `@Bean` methods that create other beans. They don't contain business logic.
2. `@Component`-annotated business classes are the *targets* of wiring.
3. **The rule:** if a class is configuring infrastructure (`HttpClient`, `Clock`, `ObjectMapper`), it's `@Configuration` + `@Bean` methods. If it has business behavior, it's `@Service`/`@Component`.
4. `@Bean` methods are how you register third-party objects (no Kotlin annotations on them) and how you build objects with non-trivial construction logic.
5. Avoid `@Component` on configuration objects. The two annotations mean different things; don't blur them.

**Enforcement:** review; `@Bean` factories live in `@Configuration` classes, `@Component` is reserved for business behavior.

### 3.4 — `@ConfigurationProperties` as `data class`, with `kotlin-spring`'s constructor-binding.

**Reasoning, step by step:**
1. Spring Boot supports `@ConfigurationProperties` with constructor binding on Kotlin `data class`. This is the right shape — immutable, type-safe, validated.
2. Annotation: `@ConfigurationProperties(prefix = "payments")` + `@ConstructorBinding` (or, in newer Boot, implicit via primary constructor).
3. Enable in a configuration class: `@EnableConfigurationProperties(PaymentsProperties::class)`.
4. **Pattern:**
   ```kotlin
   @ConfigurationProperties(prefix = "payments")
   data class PaymentsProperties(
       val gatewayUrl: String,
       val timeout: Duration = Duration.ofSeconds(5),
       val retries: Int = 3,
   )
   ```
5. Validate with `@Validated` and Jakarta validation annotations. Fail fast at startup, not at first use.

**Enforcement:** review; properties bind to an immutable `data class`, `@Validated` enforced at startup by an integration test.

### 3.5 — Transactional boundaries: `@Transactional` on service methods only.

**Reasoning, step by step:**
1. `@Transactional` opens a transaction at method entry and commits or rolls back at exit. Applied via Spring's proxy — which means it must be on a *bean method* called *from another bean*.
2. Self-calls (`this.txMethod()`) bypass the proxy. The transaction does not start. This is the most common Spring transactional bug.
3. **Rule:** `@Transactional` on the public service-layer method that is the unit of work. Not on repositories, not on controllers, not on private helpers.
4. Propagation: default is `REQUIRED` — joins an existing transaction or starts a new one. Override only with explicit reasoning (`REQUIRES_NEW` for audit logs, `SUPPORTS` for read-only views).
5. Test boundaries: integration tests must verify rollback behavior on exception.

**Enforcement:** review; `@Transactional` only on public service methods, integration test asserts rollback on exception.

### 3.6 — Controllers are thin. Business logic lives in services.

**Reasoning, step by step:**
1. A controller's job: parse the request, call a service, render the response. Nothing else.
2. Putting validation, business rules, or persistence logic in the controller couples HTTP plumbing to the domain. Both become hard to test.
3. The controller is a *boundary* — apply the rules of boundaries: validate input, translate domain results to HTTP responses, translate domain exceptions to HTTP status codes (via `@ControllerAdvice`).
4. Spring WebFlux + Kotlin: prefer `suspend fun` controllers. Spring Boot 3 + Kotlin 1.9+ supports this without ceremony.

**Enforcement:** review; controllers parse-delegate-render only, no domain rules or persistence calls in the handler.

### 3.7 — Spring WebFlux: `suspend` controllers, `Flow` returns; no `Mono`/`Flux` in business code.

**Reasoning, step by step:**
1. Spring WebFlux exposes Reactor types (`Mono`, `Flux`). Kotlin's `kotlinx-coroutines-reactor` lets controllers be `suspend fun` and return `Flow<T>`.
2. The framework bridges `Mono`/`Flux` ↔ `suspend`/`Flow` at the controller signature. Inside, your code is suspend/Flow.
3. **Boundary pattern:**
   ```kotlin
   @GetMapping("/orders/{id}")
   suspend fun getOrder(@PathVariable id: OrderId): OrderResponse =
       orderService.find(id).toResponse()
   ```
4. Don't propagate `Mono`/`Flux` past the controller. The conversion happens once, at the edge.

**Enforcement:** review; `Mono`/`Flux` appear only at controller signatures, business code is `suspend`/`Flow`.

### 3.8 — Ktor: route groups by feature; explicit `Application.module` per feature.

**Reasoning, step by step:**
1. Ktor's `Application.module()` is the wiring point. Don't pile every route under one giant module.
2. Group by feature: `Application.paymentsRoutes()`, `Application.ordersRoutes()`. The main `module()` calls each.
3. Each feature module installs its own status pages, content negotiation overrides, and authentication where appropriate.
4. Plugin installation order matters. Install `ContentNegotiation`, `Authentication`, `StatusPages` at the top-level module before route modules.

**Enforcement:** review; one `Application.<feature>Routes()` extension per feature, top-level `module()` installs shared plugins first.

### 3.9 — Dependency injection in Ktor: explicit, not magic.

**Reasoning, step by step:**
1. Ktor doesn't have built-in DI. Use Koin, Kodein, or — for simple cases — a top-level `application.attributes` registry or just constructor wiring at `module()` time.
2. **Recommended for new code:** plain constructor wiring at the `module()` function. Build dependencies once, pass them into route handlers explicitly. No magic, no reflection.
3. For larger apps, Koin is fine but introduces a service-locator pattern. Use sparingly; resolve at the top of the module, not inside handlers.
4. Avoid `application.attributes` for anything other than truly request-scoped values.

**Enforcement:** review; dependencies wired by constructor at `module()`, resolved once at the top, never inside handlers.

### 3.10 — Bean lifecycle: prefer `@PostConstruct`/`@PreDestroy` for hooks; `DisposableBean`/`InitializingBean` only when forced.

**Reasoning, step by step:**
1. `@PostConstruct` runs after dependency injection completes. `@PreDestroy` runs before bean destruction. These are framework-standard.
2. `InitializingBean.afterPropertiesSet()` and `DisposableBean.destroy()` couple your class to Spring interfaces. Avoid.
3. For coroutine scopes owned by a bean: `@PreDestroy fun shutdown() = scope.cancel()`. Test that SIGTERM produces clean shutdown logs.

**Enforcement:** review; `@PostConstruct`/`@PreDestroy` for hooks, `InitializingBean`/`DisposableBean` only with a recorded reason.

### 3.11 — Exception handling at the boundary: `@ControllerAdvice` translates domain failures to HTTP.

**Reasoning, step by step:**
1. Domain code throws or returns `Result.Err`. The controller boundary translates to HTTP.
2. For thrown exceptions: `@ControllerAdvice` with `@ExceptionHandler` per exception type. Maps `ValidationException` → 400, `NotFoundException` → 404, etc.
3. For `Result.Err` returns: translate in the controller itself (`when (result) { is Ok -> ResponseEntity.ok(...); is Err -> ResponseEntity.status(...) }`).
4. **Don't:** let exceptions leak with default Spring handling. The error response shape is part of your API; control it.
5. Document the error response shape in OpenAPI/equivalent. Stability matters.

**Enforcement:** review; `@ControllerAdvice` maps each domain exception to a status, no reliance on default Spring error handling.

### 3.12 — Beans are stateless or thread-safe. The container assumes so.

**Reasoning, step by step:**
1. Spring beans are singletons by default. Multiple threads call them concurrently.
2. Mutable state in a singleton bean is a thread-safety bug waiting to happen.
3. **Rule:** beans either (a) are stateless (no `var` fields), (b) have only immutable state (all `val`), or (c) protect mutable state with appropriate concurrency primitives.
4. `@Scope("request")` for per-request state — rare and usually a smell.
5. Caches inside beans must be thread-safe (`ConcurrentHashMap`, Caffeine).

**Enforcement:** review; singleton beans hold only `val` immutable state or guard mutable state with a concurrency primitive.

## Cross-references

- DI patterns and constructor injection: [generic guide ch. 10](../kotlin/10-api-design.md).
- Persistence + transaction boundaries: [ch. 04](./04-persistence.md).
- Coroutines in WebFlux: [ch. 02](./02-jvm-concurrency.md).
