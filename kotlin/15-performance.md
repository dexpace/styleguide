# 15 — Performance

Design-time is the best time for 1000× improvements. Code-time is the best time for 10× ones. Profile-time is the only honest time for everything below 10×.

## What good looks like

```kotlin
@JvmInline value class AccountId(val raw: String) // 15.3 no wrapping cost on the identifier

/** Sums the balances of active accounts, streaming the transform and avoiding boxing. */
fun activeBalanceCents(accounts: List<Account>): Long {
    // 15.4 one allocation at toList-equivalent; 15.6 no intermediate List per step
    return accounts.asSequence()
        .filter { it.active }
        .map { it.balanceCents }
        .sum()
}

/** Renders a CSV line once, with a size hint and no `+`-in-loop. */
fun toCsv(ids: List<AccountId>): String =
    buildString(ids.size * 12) { // 15.5 capacity hint; 15.7 buildString over `+`-in-loop
        ids.forEachIndexed { i, id ->
            if (i > 0) append(',')
            append(id.raw)
        }
    }

// 15.9 measured before trusting — the hot path is proven, not assumed
fun reportHotPath(accounts: List<Account>): Long {
    val (cents, nanos) = measureTimedValue { activeBalanceCents(accounts) }
    logger.debug { "activeBalanceCents took $nanos for ${accounts.size} accounts" }
    return cents
}
```

Network-then-CPU ordering keeps the work in memory rather than fanning out per account (15.1); the `value class` carries `AccountId` without a box (15.3); the `asSequence()` chain streams the filter-then-map without an intermediate `List` (15.4, 15.6); `buildString` takes a capacity hint and replaces `+`-in-loop concatenation (15.5, 15.7); and `measureTimedValue` makes the hot path measured rather than assumed (15.9).

## Rules

### 15.1 — Design for the slowest resource first: network > disk > memory > CPU.

**Reasoning, step by step:**
1. A request that does 10 sequential network calls is hopeless regardless of how tight the code is. Batch, parallelize, or cache the network *first*.
2. A loop that's slow because of allocation pressure rarely wins from instruction-level tuning; reduce allocations first.
3. Order of priority: eliminate the round-trip; batch the round-trips; cache the response; reduce allocations; tune the inner loop.
4. CPU optimizations are last because (a) they're the smallest absolute wins on most server workloads, (b) the JIT already handles many of them, (c) they're the easiest to do wrong.

**Enforcement:** review; design and PR discussion start from the resource hierarchy, not the inner loop.

### 15.2 — `inline` for higher-order functions in measured hot paths, and for `reified`.

**Reasoning, step by step:**
1. `inline` copies the function and its lambda arguments at every call site. It eliminates lambda allocation and call overhead.
2. Worth it when: (a) the function takes a lambda and runs in a hot loop, (b) you need `reified` generics, (c) the body is small (~10 lines) so call-site bloat is bounded.
3. Not worth it: pure-function inlines without lambda parameters (the JIT inlines them anyway), large bodies that bloat call sites, refactor-prone code that you'll want to evolve.
4. **Trap:** every `inline` is an ABI commitment. Changing the body is observable at every call site — including across modules.
5. Measure before and after. "It should be faster" is not a measurement.

**Enforcement:** review; `inline` carries a benchmark or `reified` justification, not "should be faster."

### 15.3 — `value class` to avoid wrapping cost on identifiers.

**Reasoning, step by step:**
1. `@JvmInline value class UserId(val raw: String)` compiles to `String` at most JVM call sites. No box, no extra allocation, with the type-safety win.
2. Use for: identifiers (`UserId`), units (`Cents`, `Millis`), validated wrappers (`Email`).
3. Caveat: value classes box when used as a nullable (`UserId?`), in generics (`List<UserId>`), or via an interface. Most server code is unaffected at hot-path scale, but profile if uncertain.
4. **Anti-pattern:** value class with an `init` block doing heavy work. The init runs every time the value class is constructed — which may be on every primitive conversion at the JVM level depending on context.

**Enforcement:** review; identifiers and units are `@JvmInline value class`, `init` blocks stay cheap.

### 15.4 — `Sequence` over `List` for chained transforms on large or unbounded data.

**Reasoning, step by step:**
1. `List.map { }.filter { }.map { }` allocates an intermediate list per step. For large inputs, this matters.
2. `asSequence().map { }.filter { }.map { }.toList()` lazily streams elements through all steps; one allocation at the end.
3. Crossover point: a few hundred elements, roughly, depending on transform cost. Below that, `List` is *faster* (sequences have per-element overhead).
4. **Anti-pattern:** `Sequence` on a small list "for performance." You added overhead.
5. **Anti-pattern:** terminal operations like `count()` or `first()` on `Sequence` when the source is genuinely small — no win.

**Enforcement:** review; multi-step transform chains on large inputs go through `asSequence()`, small ones stay `List`.

### 15.5 — Size hints on builders and collections.

**Reasoning, step by step:**
1. `ArrayList<T>(expectedSize)` skips the doubling pattern. `StringBuilder(expectedLen)` skips reallocation.
2. When you know the size approximately, pass it. Free win.
3. `buildList(n) { ... }`, `buildString(n) { ... }` accept a capacity hint.
4. Don't guess wildly. A hint of 1,000,000 for a typical 10-element list wastes memory.

**Enforcement:** review; builders fed from a known-size source pass the capacity hint.

### 15.6 — Allocations: know where they live, eliminate them where they matter.

**Reasoning, step by step:**
1. Common allocation sources: lambdas that capture state, boxed primitives (`Int?`), iterators on collections, string concatenation in loops, `Pair`/`Triple`, intermediate collections in chains.
2. The JIT escape-analyses many short-lived allocations away. Don't pre-optimize against it.
3. Profile (JFR, async-profiler on JVM) to find real allocation hot spots. Optimize those.
4. Common fixes: lift captured state out of the lambda (or inline the lambda away), use primitive specializations (`IntArray` over `Array<Int>`), `StringBuilder` over `+`-in-loop, `Sequence` or stream APIs.

**Enforcement:** allocation-profiler (JFR/async-profiler) evidence before an allocation rewrite; review otherwise.

### 15.7 — `String` operations: templates, `StringBuilder`, and don't `+` in loops.

**Reasoning, step by step:**
1. `"Hello, $name"` compiles to a `StringBuilder` chain — efficient and readable.
2. `s = s + part` in a loop allocates `O(n²)` characters. Use `StringBuilder` (or `buildString { }`).
3. `joinToString(",")` is `O(n)` and avoids the loop entirely.
4. `String.format` is slow and not type-safe — prefer templates or `buildString`.

**Enforcement:** review; no `+=`-string in a loop, `joinToString`/`buildString` instead.

### 15.8 — `Array` (especially `IntArray`/`LongArray`) over `List<Int>` in numeric hot paths.

**Reasoning, step by step:**
1. `List<Int>` boxes every element. `IntArray` does not.
2. For genuinely numeric workloads (statistics, vector math, image processing), the boxing cost is significant.
3. Most server code does not have these workloads. Don't reach for `IntArray` because "arrays are fast."
4. When you do: prefer `IntArray`/`LongArray`/`DoubleArray` over `Array<Int>`/`Array<Long>`/`Array<Double>` for the primitive specialization.

**Enforcement:** review; numeric hot paths use primitive arrays, non-numeric code keeps `List`.

### 15.9 — Don't micro-optimize without a profile.

**Reasoning, step by step:**
1. Intuition about JVM performance is wrong roughly half the time. JIT, inlining, escape analysis, and GC all interact non-locally.
2. Before optimizing: profile. After optimizing: profile again. The delta is the truth.
3. Tools: JFR + JMC, async-profiler, IntelliJ profiler. Microbenchmarks: JMH (with the right disclaimers about JIT warmup and noise).
4. Anti-pattern: "I read that `for` is faster than `forEach`, so we should switch them all." Without a profile, you've just churned the diff.

**Enforcement:** review; a performance PR attaches before/after profile or benchmark numbers.

### 15.10 — Memoization with bounded caches.

**Reasoning, step by step:**
1. Caching results is the cheapest win for pure-function hot paths.
2. **Bound the cache.** Unbounded caches are leaks. Use `caffeine` or equivalent with a max-size, TTL, or weak references.
3. Pure-function memoization: easy and safe. Memoizing impure functions: a footgun (stale results, hidden state).
4. Wrap as an extension: `fun <K, V> ((K) -> V).memoized(maxSize: Int): (K) -> V = ...`.

**Enforcement:** review; every cache declares a bound (max-size/TTL/weak), memoized functions are pure.

### 15.11 — Coroutine overhead: cheap, not free.

**Reasoning, step by step:**
1. A coroutine launch is ~hundreds of bytes; a context switch is a few hundred ns.
2. Launching millions per second is fine. Launching one per element of a 10M-element list, each doing a microsecond of work, is wasteful.
3. **Pattern:** batch work into chunks (`chunked(1024)`), launch per chunk, not per element.
4. `Flow` is lighter than a per-element `async`. Use it for streaming pipelines.

**Enforcement:** review; fan-out over large collections launches per chunk, not per element.

### 15.12 — Cold-start matters. So does steady-state.

**Reasoning, step by step:**
1. Cold-start (class loading, JIT warm-up, first cache miss) is what users feel on the first request.
2. Steady-state is what they feel after.
3. For server workloads: optimize for steady-state first (the common case), then for cold start where it matters (cold containers, CLI tools).
4. Tools: GraalVM native-image, AOT compilation, careful classpath trimming — see [JVM guide ch. 08](../kotlin-jvm/08-build-and-distribution.md).

**Enforcement:** review; the optimization target (cold-start vs steady-state) is named against the workload.

## Cross-references

- `inline`/`reified` semantics: chapter 05.
- `value class` semantics: chapter 06.
- JVM-specific performance (JFR, escape analysis, native-image): [JVM guide ch. 07](../kotlin-jvm/07-jvm-performance.md).
