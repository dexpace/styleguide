# 09 — Concurrency & Async

Concurrency is where Kotlin's design earns its keep. Structured concurrency makes cancellation propagate, lifetimes explicit, and leaks unusual. Lean on it.

This chapter covers Kotlin-the-language primitives (coroutines, `Flow`, `Channel`, `Mutex`, atomics). JVM-specific concerns (virtual threads / Loom, Reactor, `CompletableFuture` interop, `ThreadLocal`/MDC) live in [JVM guide chapter 02](../kotlin-jvm/02-jvm-concurrency.md).

## Rules

### 9.1 — Every coroutine has an explicit `CoroutineScope`. No `GlobalScope`.

**Reasoning, step by step:**
1. `GlobalScope.launch { ... }` is a coroutine with no parent, no lifecycle, and no automatic cancellation. It will outlive the caller. It is, in practice, a leak.
2. Every coroutine you launch belongs to a *scope* — usually owned by an outer component (a request handler, a service, a UI controller, an actor).
3. The scope's lifecycle defines the coroutine's lifecycle. When the scope is cancelled, its coroutines are cancelled.
4. Forbidden: `GlobalScope`, top-level `runBlocking` outside `main()`/tests, fire-and-forget launches that escape the enclosing function.
5. **Acceptable scope sources:** `coroutineScope { ... }` inside a `suspend` function, `withContext(ctx) { ... }` for dispatcher switching, a `CoroutineScope` field on a service class (with explicit `cancel()` on shutdown).

### 9.2 — `suspend` functions are pure with respect to threading. Don't pick a dispatcher inside them.

**Reasoning, step by step:**
1. A `suspend` function should be runnable on any dispatcher. It doesn't choose; the *caller* chooses.
2. Don't wrap the body in `withContext(Dispatchers.IO) { ... }` unless the function genuinely *must* dispatch (it owns a thread-affine resource — uncommon).
3. The caller knows the threading context. Don't take it away from them.
4. **Exception:** if the function calls a blocking library and there's no async alternative, `withContext(Dispatchers.IO)` is the boundary. Even there, prefer wrapping at the *adapter* level — make the suspend function dispatcher-agnostic.

### 9.3 — Structured concurrency: launches inside a scope must complete before the scope returns.

**Reasoning, step by step:**
1. `coroutineScope { ... }` waits for all launched children. `supervisorScope { ... }` waits but doesn't cancel siblings on one child's failure.
2. This means: when a `suspend` function returns, *all its work is done*. There's no orphan coroutine running in the background.
3. **Anti-pattern:** `viewModelScope.launch { ... }` (or any external scope) from inside a function as a side effect. The work outlives the function; the caller can't see it.
4. If you need fire-and-forget semantics, that's a *different* coroutine on a *different* scope (typically a service-level one). Document it. Name the scope.

### 9.4 — Cancellation is cooperative. Honor it.

**Reasoning, step by step:**
1. A coroutine is cancelled by its scope. Cancellation is a `CancellationException` thrown at the next suspension point.
2. Long-running CPU loops without suspension points won't notice cancellation. Insert `ensureActive()` or `yield()` periodically.
3. `try/catch` in a coroutine must rethrow `CancellationException`. Catching `Throwable` and not rethrowing is the most common coroutine bug.
4. `runCatching { ... }` inside a coroutine is dangerous for the same reason. If you use it, handle `CancellationException` explicitly.
5. **Pattern:** `try { ... } catch (e: CancellationException) { throw e } catch (e: Exception) { /* handle */ }`.

### 9.5 — `withTimeout` is the bound on every `suspend` call that does I/O.

**Reasoning, step by step:**
1. I/O without a timeout is the most common production resource leak — slow servers hold our connections forever.
2. Wrap external calls in `withTimeout(...) { ... }` or `withTimeoutOrNull(...) { ... }`. Pick a number; document the choice.
3. The timeout should match the *user-perceived* SLA, not "infinity minus a bit." Aggressive timeouts surface real problems; lenient ones hide them.
4. `withTimeoutOrNull` returns `null` on timeout; `withTimeout` throws `TimeoutCancellationException` (a `CancellationException` — see 9.4). Pick by whether the caller wants to branch or fail.

### 9.6 — Dispatchers: `Dispatchers.Default` for CPU, `Dispatchers.IO` for blocking, custom for tightly-managed pools.

**Reasoning, step by step:**
1. `Dispatchers.Default` — bounded by CPU count, for CPU-bound work.
2. `Dispatchers.IO` — much larger pool (64+ threads by default), for *blocking* I/O.
3. Don't run blocking I/O on `Default` (starves CPU work). Don't run CPU work on `IO` (wastes threads on a parallelism bound that doesn't match cores).
4. For tightly-managed pools (DB connection-bound work, a third-party SDK with its own concurrency rules), construct a custom `CoroutineDispatcher` from an `Executor`. Document the pool's size and the reason.
5. Switching dispatchers costs (a context switch); don't switch needlessly inside tight loops.

### 9.7 — `Flow` for cold streams; `SharedFlow`/`StateFlow` for hot.

**Reasoning, step by step:**
1. `Flow<T>` is *cold*: nothing runs until a collector subscribes. Each collector restarts the flow.
2. `SharedFlow<T>` / `StateFlow<T>` are *hot*: they emit regardless of subscribers, and multiple subscribers share.
3. Default to `Flow`. Reach for `SharedFlow` when (a) the producer is expensive and you want fan-out, or (b) you need replay/buffering for late subscribers.
4. `StateFlow<T>` is a `SharedFlow<T>` with exactly one current value. Use it for observable state (config, current user, current selection).
5. **Anti-pattern:** `Flow` carrying mutable state. Each collector gets a fresh run, but if the flow captures a `MutableList`, you've created a shared-state hazard.

### 9.8 — `buffer()`, `conflate()`, `collectLatest` — pick deliberately, never unbounded.

**Reasoning, step by step:**
1. `flow.buffer()` adds an *unbounded* channel between producer and collector by default. Unbounded = production OOM waiting to happen.
2. Always supply a capacity: `buffer(64)`. The number should be defensible against the slowest expected downstream.
3. `conflate()` keeps only the latest emission, dropping older ones. Use when the consumer only cares about "current state," not history (UI updates, dashboards).
4. `collectLatest { ... }` cancels the in-progress collector when a new value arrives. Use when each value triggers async work and only the latest matters.
5. **Limits-on-everything rule (TigerBeetle):** every buffer has a fixed bound. State the bound near the call site, ideally as a named constant.

### 9.9 — Concurrent state: `Mutex` for coroutines, `AtomicXxx` for primitives.

**Reasoning, step by step:**
1. `kotlinx.coroutines.sync.Mutex` is suspension-aware. Locking inside a coroutine is non-blocking.
2. `synchronized { ... }` blocks the underlying thread — fine for short critical sections, but it prevents coroutine cooperation. Avoid in suspend code.
3. `kotlin.concurrent.atomics.*` (Kotlin 1.9+ stdlib, stable in 2.x) for atomic primitives: `AtomicReference`, `AtomicInt`, etc.
4. **Prefer the immutable-update pattern:** `state.update { it.copy(x = newX) }` over a lock around a mutable field.
5. Bound the lock's hold time. A `Mutex` held during an I/O call is a sequential bottleneck.

### 9.10 — `Channel` only when `Flow` doesn't fit.

**Reasoning, step by step:**
1. `Channel<T>` is for *one-shot* delivery between coroutines — like a typed queue. `Flow` is for *streams*.
2. Most "channel" use cases are actually flow use cases. If multiple consumers should see each value, use `SharedFlow`. If exactly one consumer, often `Flow.collect` is enough.
3. Acceptable `Channel` use: actor patterns (a single consumer with sequential processing), backpressure-aware producer-consumer pairs.
4. Always bound: `Channel(capacity = 16)`. Never `Channel(Channel.UNLIMITED)` in production.

### 9.11 — `async { }` only when you genuinely need concurrent results.

**Reasoning, step by step:**
1. `async` launches a coroutine that produces a `Deferred<T>`. Use it for *parallel* work whose results you'll combine.
2. `awaitAll(a, b, c)` waits for several deferreds. This is the idiomatic parallel-then-join pattern.
3. **Anti-pattern:** `async { ... }.await()` immediately on the next line. That's just a `suspend` call wrapped in noise.
4. **Anti-pattern:** `async` to "fire and forget." That's `launch` — and even `launch` needs an explicit scope.
5. Failures in `async` are reported when you `await`. Don't lose the deferred handle, or the exception is lost.

### 9.12 — Backpressure is the producer's problem. Bound your producers.

**Reasoning, step by step:**
1. A producer that emits faster than the consumer can drain will (a) buffer until OOM, (b) drop, or (c) suspend. You pick.
2. `Flow` is naturally suspension-based: emit suspends until collect catches up. This is fine for in-process streams.
3. For producers from non-coroutine sources (`callbackFlow`, `channelFlow`), explicitly configure the buffering strategy: `BufferOverflow.SUSPEND`, `DROP_OLDEST`, `DROP_LATEST`.
4. Production rule: pick a strategy at the *producer* side; document why.

### 9.13 — `runBlocking` only at the program entry point or in tests.

**Reasoning, step by step:**
1. `runBlocking` *blocks* the calling thread. Inside an event loop, it's a deadlock or a denial-of-service.
2. Legitimate uses: `main()` for CLI/scripts, JUnit tests, debugging. Anywhere else, the caller should be a `suspend` function.
3. If you "need" `runBlocking` in production code, you have an async-sync boundary. Move that boundary outward — make the entire call path async.

## Cross-references

- Resource management with structured concurrency: chapter 13.
- Virtual threads (Loom) vs coroutines: [JVM guide chapter 02](../kotlin-jvm/02-jvm-concurrency.md).
- Reactor / `CompletableFuture` interop: [JVM guide chapter 02](../kotlin-jvm/02-jvm-concurrency.md).
- MDC and thread-local context in coroutines: [JVM guide chapter 06](../kotlin-jvm/06-logging.md).
