# 02 — JVM Concurrency

Three concurrency models coexist on the modern JVM: coroutines, virtual threads (Loom), and reactive (Reactor, RxJava). They are not interchangeable. Pick deliberately; name every seam.

This chapter extends [generic guide chapter 09](../kotlin/09-concurrency.md), which covers coroutines, dispatchers, and `Flow` for language-only concerns.

## What good looks like

```kotlin
/** Boundary adapter: bridges a Reactor driver inward, fans out under structured concurrency. */
class OrderService(private val repo: OrderRepository) {

  /** Public seam is `suspend`; the Reactor type never escapes this method (2.4). */
  suspend fun loadCart(userId: UserId): Cart =
    repo.findCart(userId).awaitSingle() // Mono -> suspend at the exact boundary

  /** Fan-out: every child shares the parent scope, so one failure cancels the rest (2.1, 2.8). */
  suspend fun priceAll(items: List<Item>): List<Priced> = coroutineScope {
    items.map { item -> async { price(item) } }.awaitAll()
  }

  /** Blocking JDBC is confined to Dispatchers.IO and switched back out immediately (2.3). */
  private suspend fun price(item: Item): Priced {
    val rate = withContext(Dispatchers.IO) { repo.fetchRate(item.sku) } // blocking, scoped
    return Priced(item, rate)
  }
}
```

The public surface stays `suspend`/`Flow`-shaped and the `Mono` is awaited at the one boundary that knows about it (2.4); `coroutineScope` makes the fan-out structured so a single child failure cancels its siblings and propagates (2.1, 2.8); the blocking JDBC call is the only thing on `Dispatchers.IO`, switched in for the call and out for downstream work (2.3). No `runBlocking`, no `GlobalScope`, no virtual thread spawned inside the coroutine.

## Rules

### 2.1 — Default to coroutines for new business logic.

**Reasoning, step by step:**
1. Coroutines give structured concurrency, cooperative cancellation, `Flow` for streams, and uniform error handling. The ecosystem (`kotlinx-coroutines-*`) is mature.
2. The compile-time `suspend` marker makes async-ness visible. Java callers can't accidentally call a `suspend` function and block.
3. For new business code, `suspend` + structured scopes is the right default.
4. The exceptions are: deeply blocking workloads where cancellation isn't needed (use Loom — 2.2) and framework-mandated reactive types at the boundary (use Reactor — 2.4).

**Enforcement:** review; new async business functions are `suspend` with structured scopes, not raw threads or futures.

### 2.2 — Virtual threads (Loom) for blocking I/O without cancellation semantics.

**Reasoning, step by step:**
1. JDK 21+ provides virtual threads via `Thread.ofVirtual()` or `Executors.newVirtualThreadPerTaskExecutor()`. They're cheap to spawn (millions per JVM) and *block* the underlying carrier without consuming a platform thread.
2. Use when (a) you have a blocking library with no async alternative and don't need cancellation, (b) you're writing simple per-request fan-out where coroutines would add ceremony for no gain.
3. Coroutines on `Dispatchers.IO` have similar semantics for blocking work. Virtual threads can be the *underlying dispatcher* — `Dispatchers.IO` on JDK 21+ is virtual-thread-backed when configured.
4. **Don't mix carelessly:** spawning virtual threads inside a coroutine to do work the coroutine could do itself is a code smell. The coroutine's scope/cancellation doesn't follow into the virtual thread.
5. **Dispatcher pattern:** `val virtualDispatcher = Executors.newVirtualThreadPerTaskExecutor().asCoroutineDispatcher()` — gives you coroutine ergonomics with virtual-thread carriers.

**Enforcement:** review; virtual threads only for cancellation-free blocking work, never spawned inside a coroutine scope.

### 2.3 — `Dispatchers.IO` for blocking JDK calls when you're already in a coroutine.

**Reasoning, step by step:**
1. Don't run blocking JDBC/file/socket calls on `Dispatchers.Default` — you'll starve CPU work.
2. `withContext(Dispatchers.IO) { jdbcCall() }` switches to a thread pool sized for blocking work (~64 threads by default).
3. On JDK 21+, configure `Dispatchers.IO` to use virtual threads for unlimited blocking parallelism: see `kotlinx.coroutines` configuration or build a custom dispatcher from a virtual-thread executor.
4. Limit the scope: switch in for the blocking call, switch back out for downstream work. Long-held `Dispatchers.IO` contexts waste pool capacity.

**Enforcement:** review; blocking JDK calls wrapped in `withContext(Dispatchers.IO)`, never run on `Dispatchers.Default`.

### 2.4 — Reactor at framework boundaries only. Bridge with first-party adapters.

**Reasoning, step by step:**
1. Spring WebFlux, some MongoDB drivers, and a few other libraries return `Mono<T>` / `Flux<T>`. Inside your code, you want `suspend` / `Flow`.
2. Use `kotlinx-coroutines-reactor`: `mono.awaitSingle()`, `flux.asFlow()`, `flow.asFlux()`, `mono { ... }`, `flux { ... }`.
3. **Don't propagate Reactor types through your domain.** Convert at the exact boundary (controller method or adapter), then return `suspend`/`Flow` from the converted function downward.
4. Reactor operator semantics differ from `Flow` (hot vs cold, backpressure model). Mis-translating between them is a source of subtle bugs — read the docs at every conversion.

**Enforcement:** review; `Mono`/`Flux` confined to boundary signatures, converted to `suspend`/`Flow` before reaching the domain.

### 2.5 — `CompletableFuture` bridges for Java interop.

**Reasoning, step by step:**
1. Java callers can't `await` a `Deferred<T>` or collect a `Flow<T>`. They can wait on `CompletableFuture<T>`.
2. Bridge with `kotlinx-coroutines-jdk8`: `scope.future { suspendingWork() }` returns `CompletableFuture<T>`.
3. The bridge launches a coroutine in the given scope; cancelling the `CompletableFuture` cancels the coroutine. Pick a scope that owns the work's lifecycle.
4. **Anti-pattern:** `GlobalScope.future { ... }` because it's convenient. The future has no owner; cancellation is meaningless.
5. **Worked example:** an `OAuthAsyncManager` uses `CompletableFuture` internally because the transport is `AsyncTransport` with futures. The wrapping is honest and explicit; the bridge is named.

**Enforcement:** review; `scope.future { }` carries an owning scope, no `GlobalScope.future`.

### 2.6 — `runBlocking` only at the program entry point or in tests.

**Reasoning, step by step:**
1. Restated from generic guide §9.13: `runBlocking` blocks a platform thread. Inside an event loop or a request handler, it's a deadlock or DOS.
2. On JVM specifically: in Spring MVC controllers (servlet-based), `runBlocking` *can* work because each request has its own thread — but it defeats the point of coroutines. Make the controller a `suspend fun` instead (Spring Boot 3+ supports it).
3. In Spring WebFlux: `runBlocking` is a hard-fail. The event loop will hang.
4. Legitimate JVM-specific uses: CLI `main()`, JUnit tests via `runTest` (preferred) or `runBlocking` (legacy).

**Enforcement:** detekt forbids `runBlocking` outside `main`/test source sets; request handlers are `suspend fun`.

### 2.7 — `ThreadLocal` and MDC: explicit propagation through coroutines.

**Reasoning, step by step:**
1. Coroutines don't pin to a thread. A `ThreadLocal` set at the start of a coroutine is *not* visible after a `withContext` or suspension.
2. SLF4J MDC uses `ThreadLocal`. Without help, MDC values silently disappear across suspension points.
3. Fix: `kotlinx-coroutines-slf4j`'s `MDCContext`. Launch with `withContext(MDCContext()) { ... }` to snapshot the current MDC and propagate it across suspensions.
4. **Pattern at the request boundary:** install MDC values (correlation ID, user ID), then wrap the request body in `withContext(MDCContext()) { ... }`. Coroutine logs now carry the context.
5. For other thread-locals: use `asContextElement()` on the `ThreadLocal`, or model the value as an explicit parameter / coroutine context element.

**Enforcement:** review; coroutines that log carry `MDCContext()`, thread-locals propagated via `asContextElement()` or passed explicitly.

### 2.8 — Cancellation crosses coroutine boundaries; it does not cross thread boundaries.

**Reasoning, step by step:**
1. Cancelling a coroutine cancels its children (structured concurrency). Cancelling a `CompletableFuture` cancels the bridged coroutine.
2. Cancelling a coroutine does *not* interrupt a blocking JDK call on a thread (e.g., `Thread.sleep`, `socket.read`). The coroutine is "cancelled," but the thread keeps running until the call returns.
3. For interruptible blocking calls: wrap with `runInterruptible { blockingCall() }`. The coroutine cancellation now translates to thread interruption.
4. Some libraries don't honor `InterruptedException` properly. Test cancellation behavior under load.

**Enforcement:** review; blocking calls that must be cancellable wrapped in `runInterruptible`, cancellation paths tested.

### 2.9 — `synchronized` and `Mutex`: don't mix in suspend functions.

**Reasoning, step by step:**
1. `synchronized { ... }` blocks the underlying thread. Inside a suspend function, this can deadlock or starve.
2. Use `kotlinx.coroutines.sync.Mutex` for coroutine-safe mutual exclusion — `mutex.withLock { ... }` is suspension-aware.
3. Acceptable `synchronized` in suspend code: very short critical sections that *cannot* suspend (no I/O, no other suspends). Even then, prefer `Mutex` for consistency.
4. Both kinds of locks need bounded hold time. A `Mutex` held during an I/O call is a single-threaded bottleneck.

**Enforcement:** review; suspend functions use `Mutex.withLock`, no `synchronized` block around a suspending body.

### 2.10 — Java executors → coroutine dispatchers, when you need to bridge.

**Reasoning, step by step:**
1. `executor.asCoroutineDispatcher()` converts a `java.util.concurrent.ExecutorService` into a dispatcher.
2. Use when you have a tightly-managed executor (sized pool, named threads, third-party SDK requirements) and want coroutines on top.
3. The executor's lifecycle is yours to manage: `close()` the dispatcher when done, which shuts down the executor.
4. Verify: thread dumps under load. The executor's named threads should appear, not generic `pool-1-thread-N`.

**Enforcement:** review; bridged dispatchers `close()`d at end of lifecycle, named threads confirmed in load-test thread dumps.

## Decision tree (quick reference)

```
Is this a brand-new business function that does async work?
├─ Yes → suspend fun with structured concurrency (kotlin/09).
│         Use Dispatchers.IO for blocking JDK calls inside it.
└─ No  → Is it a JVM framework boundary returning Mono/Flux/CompletableFuture?
          ├─ Yes → bridge once at the boundary; expose suspend/Flow inward.
          └─ No  → Is it a pure blocking workload with no cancellation needs?
                    ├─ Yes → virtual threads via Executors.newVirtualThreadPerTaskExecutor()
                    │         or Dispatchers.IO if you're already in a coroutine.
                    └─ No  → restate the question; you're probably overcomplicating it.
```

## Cross-references

- Coroutine fundamentals: [generic guide ch. 09](../kotlin/09-concurrency.md).
- Logging context with MDC: [ch. 06](./06-logging.md).
- Framework-specific concurrency (WebFlux vs MVC): [ch. 03](./03-jvm-frameworks.md).
