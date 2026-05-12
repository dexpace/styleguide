# 13 — Resource Management

Resources (file handles, sockets, threads, transactions, locks) outlive the call that opens them unless something explicitly closes them. The explicit thing must be in your code.

## Rules

### 13.1 — `use { }` on every `AutoCloseable`. Never `close()` by hand.

**Reasoning, step by step:**
1. `file.use { it.read() }` closes the file on exit — normal *or* exceptional. It's the safe shape.
2. Manual `try { ... } finally { file.close() }` is the same thing more verbosely, and you'll forget the `finally` once.
3. `use` handles suppressed exceptions correctly: if `close()` throws and the body also threw, the body's exception wins and `close()`'s is added as suppressed.
4. **Anti-pattern:** opening a resource and returning it from a function. The caller can't know to `use` it — design forces the resource to escape. Take a lambda parameter instead: `fun withDb(block: (Connection) -> R): R`.

### 13.2 — Coroutine scopes are resources. Cancel them.

**Reasoning, step by step:**
1. A `CoroutineScope` you hold is a live root for child coroutines.
2. When the holder shuts down, `scope.cancel()` cancels every child and propagates cancellation throughout the tree.
3. Tie scope lifetime to a *named* lifecycle: a service's `start`/`stop`, a request's begin/end, a session's open/close.
4. **Anti-pattern:** `CoroutineScope(Dispatchers.IO)` stored in a class without a `close()`/`dispose()` method. The scope leaks.
5. **Pattern:** `class Service : AutoCloseable { private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO); override fun close() { scope.cancel() } }`.

### 13.3 — `withTimeout` is the default bound for I/O. No exceptions.

**Reasoning, step by step:**
1. Every `suspend` I/O call without a timeout is a resource leak waiting to happen.
2. `withTimeout(5.seconds) { http.get(url) }` — the timeout is a hard contract.
3. Choose a timeout that matches user expectations or upstream SLAs. Don't pick "1 hour" because "it should be enough."
4. Tested by chaos / failure injection: a downstream that hangs should produce a `TimeoutCancellationException` within budget.
5. See chapter 09 for cancellation semantics.

### 13.4 — Connection pools and thread pools are resources. Bound them.

**Reasoning, step by step:**
1. An unbounded pool is just a resource leak with a delay.
2. Every pool (HTTP client, JDBC, executor service) has a configured maximum. Set it. Document the value.
3. Pool size is a *system* parameter — picked from downstream capacity, JVM RAM, and the connection-per-request ratio. Not a guess.
4. Monitor pool exhaustion: alerts on `getConnection` blocked time, queue depth, rejected tasks.

### 13.5 — `SecureRandom` for cryptographic randomness; `Random` for everything else.

**Reasoning, step by step:**
1. `Random` and `ThreadLocalRandom` are statistical-quality, not cryptographic.
2. For tokens, nonces, secrets, IDs that must be unpredictable: `java.security.SecureRandom` (or platform equivalent).
3. `kotlin.random.Random` is the cross-platform stdlib variant — fine for game logic, test data, jitter.
4. **Anti-pattern:** `UUID.randomUUID()` for short tokens. It's a 122-bit random value, formatted as a 36-char string with separators — wasteful when you want 16 bytes of entropy.

### 13.6 — Graceful shutdown: drain in-flight, refuse new, close pools.

**Reasoning, step by step:**
1. SIGTERM (or platform equivalent) means "you have N seconds; finish what's in flight and stop."
2. Shutdown sequence: stop accepting new work → wait for in-flight to drain (bounded) → close pools → exit.
3. A hung shutdown is worse than a forced one — the orchestrator will SIGKILL eventually, but you lose the chance to flush logs and metrics.
4. Implement a `shutdown()` function that times out: `withTimeoutOrNull(30.seconds) { scope.cancelAndJoin() }`. If we hit the timeout, log loudly and exit anyway.

### 13.7 — `Closeable`/`AutoCloseable` implement explicitly; document the lifecycle.

**Reasoning, step by step:**
1. If your class owns a resource, it implements `AutoCloseable` (or `Closeable` for IO-y things).
2. `close()` is idempotent — calling twice is fine.
3. After `close()`, calls to other methods either fail loudly (`IllegalStateException`) or are no-ops. Pick and document.
4. KDoc the lifecycle: "Holds an HTTP client. Call `close()` to release the connection pool."

### 13.8 — Locks are resources. Bound the critical section.

**Reasoning, step by step:**
1. A `Mutex`/`Lock` held during I/O is a sequential bottleneck — every other caller waits.
2. Critical sections should be short and CPU-only. If I/O happens, hold the lock only around the metadata, not around the I/O itself.
3. **Pattern:** read state under lock → release → do I/O → acquire lock → write result.
4. `Mutex.withLock { ... }` (coroutine-friendly) or `lock.withLock { ... }` (thread-blocking, JVM stdlib) handles release-on-exception.

### 13.9 — Cleanup on cancellation: `NonCancellable` for the *must complete* part.

**Reasoning, step by step:**
1. A cancelled coroutine should stop quickly. But sometimes cleanup itself is a suspend operation.
2. `withContext(NonCancellable) { resource.releaseAsync() }` — the cleanup runs to completion even though the parent is cancelled.
3. Use sparingly. Most cleanup is synchronous (`close()`) or should be skipped on cancellation (the resource will be GC'd anyway).
4. **Anti-pattern:** wrapping the whole function in `NonCancellable` to "make sure it finishes." That defeats cancellation entirely.

### 13.10 — Files and streams: prefer functional helpers over hand-rolled loops.

**Reasoning, step by step:**
1. `File.readText()`, `File.readLines()`, `File.useLines { lines -> ... }` are the right defaults.
2. `useLines` is a `Sequence<String>` that closes the file when the lambda returns — no manual `BufferedReader` work.
3. For large files: `useLines` (lazy) over `readLines` (eager, full-buffer).
4. For binary: `inputStream().buffered().use { ... }` or `readBytes()` for small files.

## Cross-references

- Coroutine cancellation and structured concurrency: chapter 09.
- JVM-specific resource concerns (Loom, native handles, classloaders): [JVM guide ch. 02](../kotlin-jvm/02-jvm-concurrency.md).
- Pool sizing on JVM frameworks: [JVM guide ch. 03](../kotlin-jvm/03-jvm-frameworks.md).
