# 06 — Logging on JVM

Logging is a public API for your future on-call self. Treat it like one.

## Rules

### 6.1 — SLF4J + `kotlin-logging`. No direct logger imports.

**Reasoning, step by step:**
1. SLF4J is the canonical JVM logging facade. Use it.
2. `io.github.oshai:kotlin-logging-jvm` (or the older `mu.KotlinLogging`) wraps SLF4J with a Kotlin-friendly API: lazy message blocks, no SLF4J-style placeholder strings unless you want them.
3. **Pattern:**
   ```kotlin
   private val logger = KotlinLogging.logger {}

   fun load(id: UserId): User {
       logger.debug { "loading user $id" }       // lazy: only formatted at DEBUG level
       // ...
   }
   ```
4. **Anti-patterns:**
   - `java.util.logging.Logger` — wrong facade, harder to route.
   - `org.apache.commons.logging.*` — wrong era.
   - Direct `org.slf4j.LoggerFactory.getLogger(...)` — works, but loses lazy blocks.

### 6.2 — Logger declared as a `private val` at file/companion scope.

**Reasoning, step by step:**
1. `private val logger = KotlinLogging.logger {}` at the top of the file is the canonical pattern. The empty `{}` lambda lets the macro infer the logger name from the file/class.
2. Don't instantiate a logger per call — it's wasteful and the logger name becomes wrong.
3. For classes that need a logger and a companion object: declare in the `companion object`, again with `KotlinLogging.logger {}`.
4. Don't pass loggers as parameters except in test scaffolding.

### 6.3 — Lazy message blocks. Never string-concat in the call.

**Reasoning, step by step:**
1. `logger.debug { "expensive: ${heavy.compute()}" }` evaluates `heavy.compute()` *only if DEBUG is enabled*.
2. `logger.debug("expensive: " + heavy.compute())` evaluates `heavy.compute()` always.
3. The lazy form is `kotlin-logging`'s main reason to exist. Use it.
4. Same for SLF4J native: `logger.debug("user {} loaded", id)` — placeholders are formatted only if the level is enabled.

### 6.4 — Levels mean what they say.

**Reasoning, step by step:**
1. **ERROR** — something failed that requires action. A page. Don't emit ERROR for a 404 the user typed wrong.
2. **WARN** — something might be wrong, or a fallback kicked in. Worth periodic review.
3. **INFO** — normal-operation events that are useful in a forensic trace. Bounded in volume.
4. **DEBUG** — detail useful for debugging, off in production.
5. **TRACE** — extremely verbose, on only when diagnosing a specific incident.
6. **Volume rule:** if INFO outputs more than ~10 lines per request, you're using INFO where DEBUG belongs.

### 6.5 — Structured logging: key/value pairs, not free-form strings.

**Reasoning, step by step:**
1. `logger.info { "user $id charged $amount" }` is unparseable by log aggregators.
2. SLF4J 2.x supports structured logging via `Marker` / `KeyValuePair`. Logback/Logstash encoders add JSON output.
3. **Pattern:**
   ```kotlin
   logger.atInfo()
       .addKeyValue("userId", id.raw)
       .addKeyValue("amount", amount.value)
       .log("charge completed")
   ```
4. Alternatively, MDC for cross-cutting context (correlation IDs, user IDs) and KV pairs for per-event detail.
5. Configure the appender to emit JSON in production, plain text in dev. Both should carry the same structured fields.

### 6.6 — MDC for correlation context. Use `MDCContext` in coroutines.

**Reasoning, step by step:**
1. MDC is SLF4J's thread-local store for context that should appear in every log line: request ID, user ID, tenant ID.
2. Set at the request boundary (filter, interceptor). Clear at the end. `MDC.putCloseable("correlationId", id).use { ... }` is the safe shape.
3. **Coroutines pitfall:** MDC is thread-local; coroutines don't pin to a thread. Bridge with `kotlinx-coroutines-slf4j`'s `MDCContext`:
   ```kotlin
   withContext(MDCContext()) {
       // MDC visible throughout this scope, even across suspensions
       handleRequest()
   }
   ```
4. Without `MDCContext`, your MDC silently disappears after the first `suspend` call. This is the most common Kotlin logging bug on JVM.

### 6.7 — Never log secrets or PII verbatim. Mask, redact, or omit.

**Reasoning, step by step:**
1. Logs end up in aggregators, ticket systems, screenshots, and someone's terminal scrollback. They're effectively public.
2. Forbidden in plain text: passwords, tokens, API keys, full credit-card numbers, full SSNs, full email addresses where regulations require, raw request/response bodies for sensitive endpoints.
3. Mask at the source: substitute `"****"` for sensitive substrings, hash where the hash is useful (user IDs in some compliance regimes).
4. **Pattern (from the Expedia SDK):** a `MaskingRegexFactory` builds regexes that match sensitive field names in JSON; a `MaskJson` step applies them before logging.
5. PII discovery is a recurring audit; assume the auditor will read your logs.

### 6.8 — No `println`, no `System.err`. Loggers only.

**Reasoning, step by step:**
1. `println` writes to stdout, bypassing the logging configuration. It can't be routed, leveled, or aggregated.
2. `System.err` is slightly better but still bypasses level filtering and structured fields.
3. **Exception:** CLI tools where the program's *output* is the point. Even there, use a logger for diagnostics and stdout only for the data result.
4. Lint: detekt's `ForbiddenMethodCall` configured to catch `println`/`System.err.println` in production code.

### 6.9 — Logback or Log4j 2 as the implementation. Configured in code or YAML, not properties.

**Reasoning, step by step:**
1. Pick one — Logback (Spring Boot's default) or Log4j 2 (better async performance, more features).
2. Don't mix. Two logging backends + SLF4J bridges is a debug nightmare.
3. Configuration: YAML or Groovy (Logback) / XML or YAML (Log4j 2). `*.properties` is older and less expressive.
4. Configure per-environment: JSON output in production (aggregator-friendly), pattern output in dev (human-friendly).

### 6.10 — Log decoration via `LoggerDecorator` or MDC, not by formatting into the message.

**Reasoning, step by step:**
1. Adding `"[SDK] message"` prefixes inline is fragile and resists changing.
2. The Expedia SDK's pattern: `class LoggerDecorator(private val logger: Logger) : Logger by logger { override fun info(m: String) = logger.info(decorate(m)) }`. Interface delegation gives you decoration in one line; you intercept exactly what you care about.
3. For correlation context, prefer MDC. For library identification, a decorator. Both work; pick by who owns the context.
4. Don't put environment info (region, hostname) in the message. Put it in MDC or in the appender's pattern. It's the same value for every log line.

### 6.11 — Log on exception only at the boundary that handles it.

**Reasoning, step by step:**
1. Logging an exception at every level it bubbles through produces duplicate stack traces in the aggregator.
2. **Rule:** log when you handle. The handler knows the context that makes the log useful.
3. Re-throwing? Don't log first. Wrapping? Log only if the wrap loses information.
4. `logger.error(e) { "operation failed: ${describe()}" }` (kotlin-logging) or `logger.error("operation failed", e)` (SLF4J) — the throwable is a parameter, not concatenated.

## Cross-references

- Coroutine context propagation: [ch. 02](./02-jvm-concurrency.md).
- PII and validation: [ch. 05](./05-serialization.md), [generic guide ch. 08](../kotlin/08-error-handling.md).
- Class delegation pattern (logger decorator): [generic guide ch. 07](../kotlin/07-kotlin-idioms.md), §7.1.
