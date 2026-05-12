# 07 — JVM Performance

The JVM has thirty years of optimization in it. Trust it; verify with a profile. This chapter extends [generic guide chapter 15](../kotlin/15-performance.md) with JVM-specific rules.

## Rules

### 7.1 — Profile before you optimize. JFR + async-profiler + IntelliJ profiler.

**Reasoning, step by step:**
1. JVM performance intuition is wrong roughly half the time. Inlining, escape analysis, JIT warmup, and GC interact non-locally.
2. **Java Flight Recorder (JFR):** built-in, low-overhead. Capture with `-XX:StartFlightRecording=duration=60s,filename=app.jfr`. Analyze with JDK Mission Control.
3. **async-profiler:** sampling profiler with CPU, allocation, lock, and wall-clock modes. Better than JFR for allocation hot spots and lock contention.
4. **IntelliJ profiler:** built-in, integrates with the IDE. Good for local benchmarks.
5. **Rule:** before you "fix" a perf issue, capture a profile that *shows the issue*. After the fix, capture another. The delta is the proof.

### 7.2 — JIT warmup matters. Measure steady-state.

**Reasoning, step by step:**
1. The JIT compiles hot methods after they run a few thousand times. The first iterations of a benchmark are interpreter-speed.
2. JMH (`org.openjdk.jmh`) handles warmup and statistical noise correctly. Use it for microbenchmarks.
3. Don't trust `System.currentTimeMillis()` deltas around a single call. They include JIT compilation, GC, and noise.
4. For production: the *first* request after a deploy is slower than steady-state. Architect for it (warm-up endpoints, gradual traffic ramp).

### 7.3 — Escape analysis: small short-lived allocations are usually free.

**Reasoning, step by step:**
1. The HotSpot JIT performs escape analysis: if an allocation doesn't escape the method (no field assignment, no return), it can be stack-allocated or scalar-replaced.
2. This means lambda captures, intermediate `Pair`s, and many "allocation" sites cost nothing at runtime.
3. **Implication:** don't micro-optimize allocations without a profile showing they survive escape analysis. The JIT is smarter than you.
4. Profile allocations with async-profiler's `--event alloc` mode. The actual hot spots are usually not the ones you'd guess.

### 7.4 — Value classes erase to primitives — usually.

**Reasoning, step by step:**
1. `@JvmInline value class UserId(val raw: String)` compiles to `String` at most call sites. No box.
2. **Boxing happens** when:
   - The value class is used as `T?` (nullable).
   - The value class is used as a generic parameter (`List<UserId>` boxes; the JVM has erased generics).
   - The value class crosses an `interface` boundary where the interface type isn't the value class itself.
3. For server-side code, these boxing cases are usually fine — pool effects and JIT recoveries amortize the cost.
4. For genuinely hot paths (millions/sec), profile and consider plain primitives if boxing dominates.

### 7.5 — Garbage collection: G1 default; ZGC for low-pause-time SLAs.

**Reasoning, step by step:**
1. JDK 17+ defaults to G1 GC. It's a good default — pause times in tens of milliseconds, throughput is good.
2. For sub-millisecond pause SLAs: **ZGC** (`-XX:+UseZGC`) or **Shenandoah**. Both are concurrent collectors with sub-ms pauses.
3. Heap sizing: `-Xms == -Xmx` in containers (no resize work, predictable memory use). Cap container memory at heap + ~25% for non-heap (metaspace, code cache, direct buffers, native libs).
4. Monitor GC: enable GC logging (`-Xlog:gc*:file=gc.log:time,level,tags`). Watch for promotion failures, allocation pauses, and pause-time outliers.

### 7.6 — Bound the JIT code cache and metaspace.

**Reasoning, step by step:**
1. JVM defaults can let the code cache (`-XX:ReservedCodeCacheSize`) fill, after which the JIT stops compiling. Performance degrades silently.
2. For larger applications (Spring Boot, microservices): `-XX:ReservedCodeCacheSize=512m` (or higher).
3. Metaspace: dynamically sized by default. In leak scenarios (classloader leaks), it grows without bound. Set `-XX:MaxMetaspaceSize=256m` and monitor.
4. JIT compilation hot spots: `-XX:+PrintCompilation` shows what's compiling. `-XX:+PrintInlining` shows inlining decisions (verbose; for debugging).

### 7.7 — Object pools: rarely worth it. Profile first.

**Reasoning, step by step:**
1. GC for short-lived objects is faster than pool management for most workloads.
2. Object pools win when (a) the object is genuinely expensive to construct (HTTP clients, JDBC connections, parsers with caches), (b) the object is large enough to put pressure on the young generation.
3. For everything else: pooling adds complexity, locking, and the risk of state leakage between users.
4. **Don't** roll your own pool. Use libraries (Apache Commons Pool, HikariCP for DB) that have the lifecycle right.

### 7.8 — Native libraries (JNI, JNA): treat as foreign code.

**Reasoning, step by step:**
1. JNI bypasses JVM safety: native memory, native threads, native deadlocks. Bugs there crash the JVM, not throw exceptions.
2. Use only when there's no pure-Java/Kotlin alternative. Common cases: cryptography (Conscrypt), image/video codecs, hardware accelerators.
3. The wrapper around JNI should be small, well-tested, and run on a dedicated dispatcher (don't share with regular workload).
4. Memory: JNI allocations are off-heap. `Runtime.maxMemory()` doesn't see them. Monitor RSS separately.

### 7.9 — Direct buffers and off-heap: bounded, explicitly closed.

**Reasoning, step by step:**
1. `ByteBuffer.allocateDirect(n)` allocates off-heap memory. It's released when the buffer is GC'd — slow, unpredictable.
2. For bounded use: try-with-resources / `use {}` with explicit cleanup (`Cleaner` API on JDK 9+).
3. Limit via `-XX:MaxDirectMemorySize=...`. Default is roughly equal to `-Xmx`, which is usually too much.
4. Common heavy users: Netty (HTTP servers), high-throughput JDBC drivers. Configure them explicitly.

### 7.10 — GraalVM native-image: viable, with caveats.

**Reasoning, step by step:**
1. GraalVM compiles a JVM application to a native executable. Cold start drops from seconds to milliseconds. Throughput is comparable to JIT-compiled JVM after warmup, sometimes lower.
2. **Caveats:** reflection, dynamic class loading, and resource loading need explicit configuration (`reflect-config.json`, `resource-config.json`). Many frameworks emit these automatically; Spring Boot 3+ has native-image support.
3. **Where it wins:** CLI tools, short-lived workloads (Lambda, Cloud Functions), workloads where cold start dominates.
4. **Where it loses:** long-running heavy-compute services that benefit from JIT speculation.
5. Test thoroughly. Native-image bugs surface only at native-image build time or first run — not in JVM-mode tests.

### 7.11 — `String.intern()` and weak references: don't.

**Reasoning, step by step:**
1. `String.intern()` puts strings in a JVM-managed pool. The pool is a shared synchronized structure; high-concurrency intern() calls become a bottleneck.
2. For deduplication of strings: use your own `ConcurrentHashMap<String, String>` or rely on the JVM's `-XX:+UseStringDeduplication` (G1).
3. Weak references and `SoftReference`: powerful, easy to misuse. The GC's policy on when to clear them is opaque. Use libraries (Caffeine for caches) that handle this correctly.

### 7.12 — Microservices: budget cold-start, warm-up, and shutdown.

**Reasoning, step by step:**
1. Cold start: time from container start to "ready to serve." For Spring Boot, often 5-15 seconds; for Ktor, 1-3 seconds. Plan for it in deployment (rolling restarts, traffic ramp-up).
2. Warm-up: the first N requests are JIT-warming. Either accept higher latency or pre-warm with synthetic traffic.
3. Shutdown: SIGTERM grace period. Drain in-flight, flush logs/metrics, exit cleanly. See [generic guide ch. 13](../kotlin/13-resource-management.md), §13.6.
4. Trade-offs: GraalVM native-image cuts cold-start but loses JIT peak throughput; large pools amortize warm-up but slow startup.

## Cross-references

- Allocation patterns and `Sequence`: [generic guide ch. 15](../kotlin/15-performance.md).
- Value classes: [generic guide ch. 06](../kotlin/06-classes-and-data-modeling.md), §6.2.
- Concurrency and pool sizing: [ch. 02](./02-jvm-concurrency.md), [ch. 04](./04-persistence.md).
