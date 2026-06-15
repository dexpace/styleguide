# 08 — Error Handling

Errors are values. Exceptions are for the things you genuinely cannot handle. Two rules with sharp edges; everything else follows.

## What good looks like

```kotlin
sealed interface Result<out T, out E> {
    data class Ok<T>(val value: T) : Result<T, Nothing>
    data class Err<E>(val error: E) : Result<Nothing, E>
}

sealed interface ChargeError {
    data class Declined(val reason: DeclineReason) : ChargeError
    data class GatewayUnavailable(val cause: Throwable) : ChargeError
}

// Boundary: translate the gateway's exceptions into the domain's typed error channel.
suspend fun chargeCard(card: Card, amount: Cents): Result<Receipt, ChargeError> {
    require(amount.value > 0) { "amount must be positive, got $amount" } // 8.4 programmer error

    val response = try {
        gateway.submit(card.tokenize(), amount)
    } catch (e: GatewayException) { // 8.3 external fault, 8.5 wrapped at the boundary
        return Result.Err(ChargeError.GatewayUnavailable(e)) // 8.5 cause preserved
    }

    return when (response.code) { // 8.6 exhaustive, no else
        ResponseCode.OK        -> Result.Ok(response.toReceipt())
        ResponseCode.DECLINED  -> Result.Err(ChargeError.Declined(response.declineReason()))
        ResponseCode.TEMPORARY -> Result.Err(ChargeError.GatewayUnavailable(response.errorOrUnknown()))
    }
}
```

`ChargeError` is a sealed ADT, not an exception (8.1); the function returns the canonical `Result<out T, out E>` typed error channel (8.2); the gateway's external fault throws and is caught only at this boundary, with its cause preserved into the wrap (8.3, 8.5); the precondition uses `require` for the caller's bug (8.4); the `when` is exhaustive with no `else` (8.6). A throwing dependency becomes a `Result` exactly here, never deeper.

## Rules

### 8.1 — Domain failures are sealed ADTs. Not exceptions.

**Reasoning, step by step:**
1. A domain failure (card declined, user not found, validation failed, rate limit exceeded) is *expected*. The caller will route on it, log it, retry it, or surface it to the user.
2. Exceptions don't appear in signatures — `fun chargeCard(card: Card, amount: Cents): Receipt` could throw anything, and the type system won't tell you.
3. Sealed ADTs are visible in signatures: `fun chargeCard(card: Card, amount: Cents): ChargeResult` where `ChargeResult` enumerates the cases.
4. The caller pattern-matches with an exhaustive `when` — the compiler enforces that every case is handled.
5. **Worked shape:**
   ```kotlin
   sealed interface ChargeResult {
       data class Approved(val receipt: Receipt) : ChargeResult
       data class Declined(val reason: DeclineReason) : ChargeResult
       data class TemporaryFailure(val cause: Throwable) : ChargeResult
   }
   ```

**Enforcement:** review; expected domain failures are sealed ADTs in the return type, not thrown exceptions.

### 8.2 — Use `kotlin.Result` or a project `Result<T, E>` sealed class.

**Reasoning, step by step:**
1. Stdlib `kotlin.Result<T>` exists; its `failure` carries a `Throwable`. It's adequate for "succeeded or threw" call sites but has limitations: the failure type isn't generic, and it's intentionally restricted from being a return type by default (compiler warning) because it's easy to misuse.
2. For domain errors, prefer a project-defined sealed class:
   ```kotlin
   sealed interface Result<out T, out E> {
       data class Ok<T>(val value: T) : Result<T, Nothing>
       data class Err<E>(val error: E) : Result<Nothing, E>
   }
   ```
3. This gives you (a) a typed error channel — the `E` documents the *kinds* of failure, (b) compile-time exhaustiveness when unwrapping, (c) no entanglement with `Throwable`.
4. Avoid pulling in arrow-kt's `Either` unless you're already committed to its idioms. For a server-side codebase, a small in-repo sealed class is enough.

**Enforcement:** review; domain APIs return a project `Result<out T, out E>`, `kotlin.Result` confined to "succeeded or threw" call sites.

### 8.3 — Exceptions for unrecoverable, programmer-error, and external-fault.

**Reasoning, step by step:**
1. Some failures are not "domain": the network died, the disk is full, the parser saw something it can't handle, a precondition was violated.
2. These should throw. Throwing here is correct — the caller usually cannot recover at this level, only farther up.
3. Three flavors:
   - `IllegalArgumentException` — caller violated a precondition. Use `require(...)`.
   - `IllegalStateException` — state is inconsistent. Use `check(...)`.
   - `error(...)` — unreachable branch reached. Indicates a bug.
4. Never catch `Throwable` or bare `Exception` to "be safe." That's how `OutOfMemoryError`, `StackOverflowError`, and `CancellationException` get swallowed.

**Enforcement:** review; detekt `TooGenericExceptionCaught` flags bare `Throwable`/`Exception` catches.

### 8.4 — `require`, `check`, `error` — three different exceptions for three different fault modes.

**Reasoning, step by step:**
1. `require(predicate) { "msg" }` → throws `IllegalArgumentException`. Use at *function entry* for input validation. The fault is the caller's.
2. `check(predicate) { "msg" }` → throws `IllegalStateException`. Use at *function body* when the object's state is inconsistent. The fault is ours.
3. `error("msg")` → throws `IllegalStateException` (since this is "we got somewhere we shouldn't"). Use in unreachable branches.
4. Why three primitives: at the throw site, the exception type tells maintainers who's at fault. `catch (e: IllegalArgumentException)` is a different recovery story than `catch (e: IllegalStateException)`.
5. **Assertion density rule:** average two assertions per function. Preconditions at entry, invariants at exit, no compound assertions (split them).
6. **Pair-asserting:** when feasible, verify the same property two ways. Example: after a sort, assert both `xs.size == originalSize` (no loss) and `xs.zipWithNext().all { (a, b) -> a <= b }` (ordering). Both must hold.

**Enforcement:** review; `require` at entry, `check` for state, `error` for unreachable branches, with assertion density ≈ two per function.

### 8.5 — Wrap exceptions at module boundaries; don't let them leak between layers.

**Reasoning, step by step:**
1. A `SQLException` from your repository layer should not propagate to your domain logic untouched. The domain doesn't know about SQL.
2. Boundary functions catch and wrap: `try { repo.find(id) } catch (e: SQLException) { return Result.Err(StorageError.UnavailableDb(e)) }`.
3. The wrapping function preserves the cause (`Throwable.cause`) so debug context isn't lost.
4. **Anti-pattern:** wrapping every exception into a custom `*Exception` that carries no extra information. Either add information (correlation ID, request context, structured fields) or don't wrap.
5. From the Expedia SDK: `ExpediaGroupAuthException(requestId, message, cause)` — the wrap is worth it because it adds correlation context.

**Enforcement:** review; foreign exception types are caught and translated at adapter modules, never observed in domain code.

### 8.6 — Exhaustive `when` over sealed subjects. No `else`.

**Reasoning, step by step:**
1. When the subject of a `when` is a sealed class/interface, an enum, or `Boolean`, the compiler can verify exhaustiveness if there is no `else` branch.
2. Adding a new variant to the sealed hierarchy then *fails to compile* everywhere a `when` doesn't handle it. This is exactly the refactor safety you want.
3. Adding `else -> TODO()` defeats this. Don't.
4. Acceptable `else`: when the subject is *open* (`Throwable`, arbitrary `Any`) and you genuinely don't know all the variants. State that in a comment.

**Enforcement:** compiler exhaustiveness on `when` expressions over sealed/enum subjects; detekt `ElseCaseInsteadOfExhaustiveWhen` flags an unnecessary `else`.

### 8.7 — `runCatching` only at adapter boundaries.

**Reasoning, step by step:**
1. `runCatching { ... }` returns `kotlin.Result<T>` — convenient for converting "throws" to "returns" at a boundary.
2. Don't sprinkle it through the codebase. Every `runCatching` inside the domain hides where the actual exception originated.
3. Right place: the *adapter* function that calls into a Java library or framework. Translate the result into your domain's `Result<T, E>` immediately.
4. **Trap:** `runCatching` catches `Throwable`, including `CancellationException`. In coroutines, you must rethrow `CancellationException` — see [chapter 09](./09-concurrency.md).

**Enforcement:** review; `runCatching` appears only in adapter modules, immediately mapped to a domain `Result`, with `CancellationException` rethrown.

### 8.8 — Error messages: include the context the caller can't see.

**Reasoning, step by step:**
1. `"validation failed"` is useless. `"order ${id}: line item ${i} has negative quantity ${qty}"` is debuggable.
2. Include the identifying inputs to the function, not just the symptom.
3. Don't include sensitive values in messages — keys, tokens, full PII. Mask them. (See chapter 12 logging / [JVM logging guide](../kotlin-jvm/06-logging.md).)
4. Messages travel into logs, stack traces, and sometimes user-facing surfaces. Treat them like a public API.

**Enforcement:** review; secret-scanning in CI; messages carry identifying inputs with sensitive values masked.

### 8.9 — `try`/`finally` for cleanup is a code smell — use `use { }` or a coroutine scope.

**Reasoning, step by step:**
1. `try { ... } finally { resource.close() }` is the right idea but the wrong syntax.
2. For `AutoCloseable` resources: `resource.use { ... }` — concise, exception-safe, idiomatic.
3. For coroutine-managed resources: structured concurrency closes scopes on cancellation. See [chapter 13](./13-resource-management.md).
4. Acceptable `finally`: when you need ordering across multiple cleanups that don't share an `AutoCloseable` contract. Even then, consider a custom delegate or extension function.

**Enforcement:** review; `AutoCloseable` resources released via `use { }`, manual `try`/`finally` cleanup justified in a comment.

### 8.10 — `Result.getOrThrow()` / `.getOrElse { ... }` / explicit `when` — pick one style per module.

**Reasoning, step by step:**
1. Once you have a `Result<T, E>`, you'll need to unwrap. The three styles each have a place:
   - `value.getOrThrow()` — at a *truly* terminal point where any failure is a bug or unrecoverable.
   - `value.getOrElse { return Result.Err(it) }` — when propagating up.
   - `when (value) { is Ok -> ...; is Err -> ... }` — when both cases have non-trivial handling.
2. Mixing all three in the same module hurts readability. Pick the dominant pattern for the module and stick with it.
3. Project-`Result<T, E>` should expose helper functions that match what the codebase uses: `getOrElse`, `map`, `mapError`, `flatMap`, `fold`.

**Enforcement:** review; one dominant unwrap style per module, helper functions exposed on the project `Result`.

## Worked example

```kotlin
sealed interface ChargeResult {
    data class Approved(val receipt: Receipt) : ChargeResult
    data class Declined(val reason: DeclineReason) : ChargeResult
    data class TemporaryFailure(val cause: Throwable) : ChargeResult
}

suspend fun chargeCard(card: Card, amount: Cents): ChargeResult {
    require(amount.value > 0) { "amount must be positive, got $amount" }

    val response = try {
        gateway.submit(card.tokenize(), amount)
    } catch (e: GatewayTimeoutException) {
        return ChargeResult.TemporaryFailure(e)
    } catch (e: GatewayException) {
        return ChargeResult.TemporaryFailure(e)
    }

    return when (response.code) {
        ResponseCode.OK            -> ChargeResult.Approved(response.toReceipt())
        ResponseCode.DECLINED      -> ChargeResult.Declined(response.declineReason())
        ResponseCode.TEMPORARY     -> ChargeResult.TemporaryFailure(response.errorOrUnknown())
    }
}

// caller — exhaustive when, no else
when (val outcome = chargeCard(card, cents)) {
    is ChargeResult.Approved         -> log.info("charged ${outcome.receipt}")
    is ChargeResult.Declined         -> log.warn("declined: ${outcome.reason}")
    is ChargeResult.TemporaryFailure -> retry(outcome.cause)
}
```

## Cross-references

- `CancellationException` and coroutines: chapter 09.
- Sealed hierarchies in detail: chapter 06.
- Exception-to-Result wrapping in framework adapters: [JVM guide chapter 03](../kotlin-jvm/03-jvm-frameworks.md).
