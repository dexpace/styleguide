# 04 — Persistence

JPA, Hibernate, Exposed, and jOOQ each have a personality. Kotlin and JPA in particular don't get along by default. The rules here keep things tolerable.

## What good looks like

```kotlin
// Entity: regular class (not data class), identity equality, all associations LAZY.
@Entity
class Order(
    @Id @GeneratedValue var id: Long? = null,
    @ManyToOne(fetch = FetchType.LAZY) var customer: Customer? = null,
) {
    override fun equals(other: Any?): Boolean =
        this === other || (other is Order && id != null && id == other.id)
    override fun hashCode(): Int = javaClass.hashCode()
}

interface OrderRepository : JpaRepository<Order, Long> {
    fun findByCustomerId(customerId: Long): List<Order>  // derived name means what it says
}

// Use case owns the transaction; the lazy access happens inside it, so no LazyInitializationException.
@Service
class RefundService(private val orders: OrderRepository) {
    @Transactional
    fun refund(orderId: Long): RefundResponse {  // DTO out, never the entity
        val order = orders.findById(orderId).orElseThrow { OrderNotFoundException(orderId) }
        val customerName = order.customer?.name ?: "unknown"  // LAZY load inside the tx
        return RefundResponse(orderId, customerName)
    }
}
```

`Order` is a plain class, not a `data class`, with ID-based equality that rejects transient instances (4.3), and its `@ManyToOne` is `LAZY` (4.4). The repository follows the Spring Data naming convention (4.6). `@Transactional` sits on the use-case service method, not the repository, so the multi-step read and the lazy `customer` access share one transaction (4.7) — the lazy load resolves inside the session, sidestepping the trap in 4.4. The boundary returns a `RefundResponse` DTO, never the entity (4.5).

## Rules

### 4.1 — `kotlin-jpa` plugin: mandatory for JPA projects.

**Reasoning, step by step:**
1. JPA reflectively requires a no-arg constructor on `@Entity` classes. Kotlin's primary constructor enforces non-null fields, which is incompatible with the no-arg requirement.
2. `kotlin-jpa` (a `kotlin-noarg` preset) generates the no-arg constructor at compile time for `@Entity`, `@Embeddable`, and `@MappedSuperclass` classes. It's invisible from Kotlin code.
3. Configure in Gradle: `plugins { kotlin("plugin.jpa") }`.
4. Without it: write `constructor() : this(...)` by hand on every entity. Rots, gets forgotten, lies about invariants.

**Enforcement:** Gradle build scan / convention plugin asserts `kotlin("plugin.jpa")` is applied wherever `jakarta.persistence` is on the classpath.

### 4.2 — `kotlin-allopen` (via `kotlin-spring` or explicit config) for proxy-friendly entities.

**Reasoning, step by step:**
1. Hibernate proxies lazy associations, which requires `open` classes. `final` JPA entities can't be lazy-loaded — Hibernate falls back to eager loading or fails.
2. `kotlin-allopen` opens classes annotated with configured annotations. `kotlin-spring` opens Spring-related ones; for JPA: `allOpen { annotation("jakarta.persistence.Entity") }`.
3. Configure once. Don't write `open class` manually on entities.

**Enforcement:** convention plugin configures `allOpen` for the persistence annotations; review rejects manual `open class` on entities.

### 4.3 — Entities are *not* `data class`. Use regular classes with explicit equality.

**Reasoning, step by step:**
1. JPA entities have an *identity* (the database row), not value semantics. Two entity instances with the same fields aren't necessarily "equal" — they might be transient (no ID) or detached.
2. `data class` `equals`/`hashCode` use *all* primary-constructor properties. For an entity, that includes mutable fields that change after load. Putting an entity in a `HashSet` and then mutating it breaks the set.
3. **Equality by business key or by ID:**
   - If the entity has a stable natural key (`email` for users, `isbn` for books), use that.
   - Otherwise, compare by ID *after rejecting transient instances*: if `id == null` or the other side's `id == null`, return identity equality.
4. **Pattern:**
   ```kotlin
   @Entity
   class User(
       @Id @GeneratedValue var id: Long? = null,
       var email: String = "",
   ) {
       override fun equals(other: Any?): Boolean {
           if (this === other) return true
           if (other !is User) return false
           return id != null && id == other.id
       }
       override fun hashCode(): Int = javaClass.hashCode()  // stable, not based on mutable id
   }
   ```
5. Yes, the equality contract is weaker than usual. JPA's identity model and Kotlin's data classes are genuinely incompatible. Pick the right tool for the job.

**Enforcement:** Detekt/ArchUnit rule forbids `@Entity` on a `data class`; review checks `equals`/`hashCode` follow the ID-or-business-key pattern.

### 4.4 — `LAZY` everywhere. `EAGER` is opt-in, justified, and documented.

**Reasoning, step by step:**
1. `EAGER` means "every load of this entity also loads this association." Cascades through joins, surprises performance.
2. `LAZY` defers loading until access. Plays well with Hibernate's session model. The default for `@*ToMany`; should be the default for `@*ToOne` too (it isn't, by JPA spec — you have to specify).
3. **Rule:** annotate all associations `LAZY` unless a documented reason says otherwise.
4. **Trap:** `LAZY` access outside a Hibernate session throws `LazyInitializationException`. The fix is *not* to switch to `EAGER` — it's to ensure your data access happens inside the transaction.

**Enforcement:** review requires a justifying comment on every `FetchType.EAGER`; a Detekt rule flags `@*ToOne` declarations missing an explicit `fetch`.

### 4.5 — DTOs at the API boundary. Never expose entities.

**Reasoning, step by step:**
1. An entity is a persistence concern. The API is a public contract. They evolve at different rates.
2. Exposing entities ties HTTP/JSON shape to schema. Schema migrations now ripple to API clients.
3. **Pattern:** a `User` entity in `com.acme.users.persistence`, a `UserResponse` data class in `com.acme.users.api`, and a mapper function. The boundary is explicit.
4. Yes, this is more code. The decoupling is worth it. (And see chapter 05 for the serialization concerns it solves.)

**Enforcement:** ArchUnit rule forbids `@Entity` types in controller/API package signatures; review checks for a mapper at the boundary.

### 4.6 — Repositories: Spring Data interfaces; method names mean what they say.

**Reasoning, step by step:**
1. Spring Data generates implementations from interface method names. `fun findByEmail(email: String): User?` works; `fun lookup(email: String): User?` doesn't (the framework can't infer the query).
2. Follow the convention: `find*`, `count*`, `exists*`, `delete*`. Use `@Query` for anything beyond derived queries.
3. **Rule:** if a repository method has more than two derived properties, write the JPQL/HQL explicitly with `@Query`. Derived names with five fields are unreadable.
4. Return types: `T?` for "may not exist," `T` (with a thrown exception or controlled `Optional` translation) for "must exist," `List<T>` (never `null`) for collections.

**Enforcement:** review enforces the `find*`/`count*`/`exists*`/`delete*` convention and the `@Query` threshold; repository startup validates derived names, failing fast on an unparseable method.

### 4.7 — Transactions span the *use case*, not the *repository call*.

**Reasoning, step by step:**
1. A use case (`refundOrder`) often touches multiple repositories. The transaction wraps the use case.
2. `@Transactional` on the *service* method that implements the use case, not on each repository call.
3. Multiple read-then-write sequences in one transaction guarantee consistency. Splitting them into per-call transactions breaks atomicity.
4. **Anti-pattern:** `@Transactional` on a repository interface method. Most operations don't need their own transaction; the caller's transaction will join.

**Enforcement:** ArchUnit rule restricts `@Transactional` to service-layer methods; review rejects it on repository interfaces.

### 4.8 — Exposed or jOOQ when SQL-first is the right model.

**Reasoning, step by step:**
1. JPA hides SQL behind an object-relational mapping. For complex queries, reporting, or any workload that *is* SQL, this hurts.
2. Alternatives:
   - **Exposed** (JetBrains): Kotlin-native DSL over SQL. Type-safe, no annotations, plays nicely with Kotlin.
   - **jOOQ**: code-generated DSL from your schema. Compile-time-validated SQL.
   - **Plain JDBC + Kotlin extensions**: when the layer above adds nothing.
3. **Decision rule:** pick JPA when the domain is genuinely object-relational and joins are simple. Pick Exposed/jOOQ when SQL semantics matter or the schema is the source of truth.
4. Mixing JPA and Exposed in the same module is a code smell. Pick one per module, even if different modules pick differently.

**Enforcement:** ArchUnit rule forbids both `jakarta.persistence` and `org.jetbrains.exposed` imports in the same module; the engine choice is recorded in an architecture decision record.

### 4.9 — Connection-pool sizing is an SRE decision, not a default.

**Reasoning, step by step:**
1. HikariCP defaults to 10 connections. That's wrong for any production workload — too small for high traffic, too big for tiny services.
2. Size from the bottleneck: typical pool size ≈ `min(db_max_connections / num_instances, expected_concurrent_requests)`.
3. Monitor: HikariCP exposes metrics (`HikariMetricsTrackerFactory`). Track `pending` connection requests and `usage` percentiles.
4. Document the value and the reasoning in the configuration. Future-you will thank you.

**Enforcement:** review requires the pool size and its sizing rationale in the configuration; HikariCP `pending`/`usage` metrics are wired to monitoring.

### 4.10 — Migrations as code: Flyway or Liquibase, versioned in source.

**Reasoning, step by step:**
1. Schema changes are code. They go through code review.
2. **Flyway** (linear, file-named) for simple cases. **Liquibase** (XML/YAML, with rollback support) for complex schemas needing rollback.
3. Migrations are *forward-only* in production. Don't edit applied migrations; write a new one.
4. Test migrations against a representative dataset before merging. CI should run migrations on a fresh database.
5. JPA's `hibernate.hbm2ddl.auto` is for development only — never `update` or `create` in production.

**Enforcement:** CI runs migrations on a fresh database against a representative dataset; a startup check fails the production profile if `hbm2ddl.auto` is `update` or `create`.

## Cross-references

- DTOs and API boundaries: [ch. 03](./03-jvm-frameworks.md).
- Serialization of entity-shaped data: [ch. 05](./05-serialization.md).
- Connection pools as resources: [generic guide ch. 13](../kotlin/13-resource-management.md).
