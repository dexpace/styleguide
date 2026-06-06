# 07 — Bun Performance

The core guide's [chapter 15](../typescript/15-performance.md) is the engine layer: allocation rate, batched awaits, the measure-first ethic. This chapter is the layer below *that* — the runtime's own pressure points. One event loop that a single slow span freezes for every client; a process whose resident memory grows past a container limit the OOM-killer never agreed to; an HTTP client whose global concurrency cap quietly bounds your throughput. The discipline is the one the JVM guide draws on JFR and async-profiler to keep — name the tool to the question, measure before you tune, record the number that earned the trick — ported to the tools Bun ships. Read [chapter 15](../typescript/15-performance.md) first; this is where its rules meet the event loop, the heap, and the socket pool.

**JSC is not V8 — carry only what is engine-independent.** Bun runs on JavaScriptCore, so the core 15 rules split. **Carries:** allocation hygiene (15.4 — every object on a hot path is collector work, on any engine), batching independent awaits (15.5), measure-first (15.6), and the optimization ledger (15.10). **Does not carry:** the V8-specific shape folklore — hidden-class transitions, the `delete`-is-a-tax rule, sparse-array deopts (15.2–15.3). JSC has its own object-shape and inline-cache machinery with different costs, so a V8 micro-trick is at best noise and at worst wrong here. The honest move is the one rule that *does* survive: measure on JSC instead of porting V8 intuition. Everything below is the JSC-and-Bun specifics that replace that folklore.

## What good looks like

```ts
import { heapStats } from 'bun:jsc';

// Alert fired: p99 event-loop lag 180ms on the export route (7.1). The loop was stalling,
// so every concurrent client stalled with it — the Bun analog of a stop-the-world pause.
const PERIOD_MS = 100;                  // interval-drift sampler (2.2): Bun has no verified monitorEventLoopDelay
let expected = performance.now() + PERIOD_MS;
setInterval(() => {                     // late tick == lag; sampled, exported, alerted at budget (2.2) — not eyeballed in dev
  const now = performance.now();
  metrics.observe('event_loop_lag_ms', Math.max(0, now - expected)); // p99 from the histogram
  expected = now + PERIOD_MS;
}, PERIOD_MS).unref?.();

// Triage with JSC's own tools, not a guess (7.2): bun:jsc startSamplingProfiler() over the
// hot route, generateHeapSnapshot() across time for what is growing. The sampling profile
// showed ~70% of wall time inside the report fan-out's outbound fetches — connections weren't
// being reused, so every call paid a fresh TCP+TLS handshake (7.3).

// before: handshakes dragging the loop into lag, fan-out unbounded against the 256 global cap.
// after: keep-alive reuse (Bun's default) kept, the fan-out ceilinged at the call site so it
// can't saturate the global slots, every wait deadlined so a slow upstream can't grow a queue (7.3, 9.6, 9.7).
const limit = pLimit(32);                         // bound concurrent outbound calls (9.7), not the input size
const reports = await Promise.all(ids.map((id) => limit(() =>
  fetch(`${UPSTREAM}/report/${id}`, {
    signal: AbortSignal.timeout(5_000),           // every external wait is deadlined (9.6)
    keepalive: true,                              // reuse the socket — Bun's default; amortize the handshake
  }),
)));

// oha -c 32 -z 30s http://localhost:3000/export   (production-shaped load, fast enough for Bun.serve, 7.6)
//   before: p50 41ms · p99 210ms · 1.2k req/s · RSS 690MB · loop-lag p99 180ms
//   after:  p50 12ms · p99  28ms · 9.8k req/s · RSS 240MB · loop-lag p99   6ms
```

The fix the profile actually rewarded was bounding the fan-out so keep-alive reuse could do its job, not a hand-tuned loop: the `bun:jsc` sampling profile named handshake-bound fetches as the cost, a call-site concurrency bound let Bun's connection reuse amortize them, and `oha` proved the delta with p50/p99 and RSS — the ledger entry justifying the deviation from the naive unbounded `Promise.all` (15.10). Each number traces to a rule: lag watched and budgeted (7.1), the right profiler for the question (7.2), bounded-and-deadlined outbound calls over the bare global cap (7.3), the load test that earns the claim (7.6), every wait deadlined as [chapter 09](../typescript/09-concurrency.md)'s §9.6 requires.

## Rules

### 7.1 — Event-loop lag is the GC-pause analog: measure it, budget it, alert on it.

**Reasoning, step by step:**
1. A JVM team watches GC pauses because a stop-the-world collection freezes every thread; a Bun team watches *event-loop lag* for the same reason, because one thread serves every request and a synchronous span freezes all of them at once. Lag is the single number that says "the loop is starving," and it is the runtime's defining health metric — the direct port of [chapter 15](../typescript/15-performance.md)'s GC-pressure focus to a runtime with one thread.
2. Measure it with the mechanism [chapter 02](./02-concurrency-and-event-loop.md)'s §2.2 settled on — Bun ships no verified `monitorEventLoopDelay`, so an interval-drift sampler stands in: schedule a tick on a fixed period, compare when it *actually* fired against `performance.now()`, and the drift is the lag the loop accumulated. It is sampled rather than measured per request so the measurement does not become the load — read it as a p99 in milliseconds next to your request metrics.
3. Budget it and alert before clients feel it. A few milliseconds of p99 lag under load is healthy; tens of milliseconds means a synchronous stretch is blocking the loop and belongs off-thread (a `Worker` job, a stream, a smaller batch). The metric exists to fire an alert at a threshold, which is why this rule lives next to §2.2's event-loop budget — measuring lag is how you prove the budget is kept.

```ts
// Bun has no verified monitorEventLoopDelay (perf_hooks 🟡 partial); sample the loop's own scheduling drift (2.2).
const PERIOD_MS = 100;
let expected = performance.now() + PERIOD_MS;
setInterval(() => {
  const now = performance.now();
  metrics.observe('event_loop_lag_ms', Math.max(0, now - expected)); // late tick == lag; p99 from the histogram
  expected = now + PERIOD_MS;
}, PERIOD_MS).unref?.();
```

**Enforcement:** Event-loop lag is sampled, exported as a metric, and alerted at the documented budget ([chapter 02](./02-concurrency-and-event-loop.md) §2.2); a service that ships without the gauge is sent back.

### 7.2 — Profile with JSC's own tools first; name the tool to the question.

**Reasoning, step by step:**
1. Performance intuition on JSC is wrong about as often as it is on V8 — the event loop, the JIT, and GC interact non-locally, so "this is the slow part" is a hypothesis until a profile shows it. This is [chapter 15](../typescript/15-performance.md)'s §15.6 measure-first rule at the runtime layer, and it bites doubly here because the JSC-vs-V8 split (see the scope note above) means you cannot fall back on V8 folklore: capture a profile that *shows the issue* before, another after, and let the delta be the proof.
2. Match the tool to the question instead of reaching for one by habit, exactly as a JVM team picks JFR versus async-profiler by what it is asking. For **CPU**: `bun:jsc`'s `startSamplingProfiler()` runs JavaScriptCore's sampling profiler over a window, and `profile(fn)` runs it for one function — the answer to "where is wall time going." For **memory**: `generateHeapSnapshot()` (imported from `bun`) writes a JSC-format JSON snapshot you open in Safari's Web Inspector (Timeline → JavaScript Allocations → Import), compared across time to find what never frees, while `heapStats()` and `memoryUsage()` from `bun:jsc` give the cheap running numbers (heap size, object counts) — the answer to "what is growing" (7.7). Reach for `Bun.gc(true)` before a snapshot so what you see is live, not yet-uncollected, garbage.
3. For **micro-comparisons**, use **mitata**, and state its caveats every time exactly as core §15.6 does for any micro-benchmark: it measures a function in isolation with a warm JIT, so it flatters code the optimizing compiler treats kindly and ignores the deopts a real call site triggers. Warm-up and variance make small deltas noise — trust order-of-magnitude gaps, distrust 10% ones. Time spans with `Bun.nanoseconds()` (nanoseconds since process start, high-precision) rather than `Date.now()`. And profile production-shaped data: a profile over toy input proves nothing about the hot path that exists under real payloads.

```ts
import { startSamplingProfiler, heapStats } from 'bun:jsc';
startSamplingProfiler();                                   // CPU: "where is wall time?" — JSC sampling profiler
await Bun.write('heap.json', JSON.stringify(Bun.generateHeapSnapshot())); // bun namespace; open in Safari Web Inspector
console.log(heapStats().heapSize, heapStats().objectCount);           // cheap running "what is growing?"
```

```ts
import { bench, run } from 'mitata'; // micro-comparison only — warm-JIT, in-isolation, NOT end-to-end
bench('parseFrame manual single-pass', () => parseFrameFast(SAMPLE));
bench('parseFrame pipeline baseline', () => parseFramePipeline(SAMPLE));
await run();
```

**Enforcement:** A perf change cites which tool was used and why; a CPU micro-fix proposed before a `bun:jsc` sampling profile names CPU as the bottleneck is sent back (15.6). A `mitata` number is never accepted as proof a *service* got faster — that is 7.6's job.

### 7.3 — Bound and deadline every outbound `fetch`; the global default is not a strategy.

**Reasoning, step by step:**
1. Connection setup is expensive — TCP handshake, TLS negotiation, sometimes auth — and the core guide's pooling doctrine ([root rule on connection pooling](../performance.md)) says reuse it. Bun's `fetch` already does the right thing here by default: it **reuses connections to the same host automatically** (Bun's own term is connection pooling) and sends `keepalive: true` unless you turn it off, so the handshake amortizes across requests for free. Do not defeat that — passing `keepalive: false` per call throws the win away, and there is rarely a reason to.
2. The explicit-bounds surface is thin, and this guide says so honestly: Bun exposes a *global* ceiling of 256 simultaneous fetch requests (`BUN_CONFIG_MAX_HTTP_REQUESTS`, raisable to ~65k) and a per-request `keepalive` toggle, but **no per-origin pool object** with a tunable `connections` bound the way undici's `Agent` offers on Node. So the bound that matters is the one you impose *at the call site*: deadline every request with `signal: AbortSignal.timeout(ms)` so a slow dependency surfaces as an error instead of an unbounded wait ([chapter 09](../typescript/09-concurrency.md) §9.6), and ceiling concurrent fan-out with the §9.7 worker-pool helper so a burst cannot consume the global slots and starve every other caller in the process.
3. Leaving both unset is a strategy by omission: requests with no timeout pile up against a slow upstream, and an unbounded `Promise.all` of `fetch`es races the 256-request cap into queueing or `EMFILE`-shaped failure. The default keep-alive is the part Bun gets right for you; the deadline and the concurrency bound are the part only you can set, because only you know the upstream's headroom and the caller's budget.

```ts
import pLimit from 'p-limit';                       // or the chapter 09 §9.7 bounded-pool helper
const limit = pLimit(32);                           // bound concurrent outbound calls to upstream headroom (9.7)
const users = await Promise.all(ids.map((id) => limit(() =>
  fetch(`${UPSTREAM}/users/${id}`, { signal: AbortSignal.timeout(5_000) }), // keep-alive on by default; deadlined (9.6)
)));
```

**Enforcement:** Reviewers reject an outbound `fetch` with no `AbortSignal.timeout` deadline, and reject unbounded `Promise.all`/loops of `fetch` past a handful of calls; bounded concurrency (9.7) plus a per-call timeout (9.6) is the default. `keepalive: false` requires a recorded reason.

### 7.4 — Stream large payloads; buffering a 2GB export is an OOM with extra steps.

**Reasoning, step by step:**
1. Reading a large file, response, or query result fully into memory before handling it scales the process's RSS with the *payload*, not with a bound you chose. A 2GB export buffered into a `Buffer` or string is an out-of-memory crash waiting for the largest input — and the OOM-killer arrives without a stack trace.
2. Process it as a stream so memory stays bounded to one chunk plus the pipeline's buffers regardless of total size. Bun is Web-Streams-native: `Bun.file(path).stream()` yields a `ReadableStream`, `Response` bodies are `ReadableStream`s, and they compose with `TransformStream`s and `pipeThrough`/`pipeTo`, which thread errors and cancellation through every stage so a failure mid-stream tears the chain down instead of leaking a half-open handle. The `node:stream/promises` `pipeline` is also available where a `node:`-stream stage is in play.
3. Honor backpressure: piping a `ReadableStream` through `pipeThrough`/`pipeTo` propagates it automatically, pausing the source when a downstream stage falls behind so a fast producer cannot outrun a slow consumer into unbounded memory. This is the same bounded-buffer discipline as [chapter 02](./02-concurrency-and-event-loop.md)'s §2.4 streams-with-backpressure rule, here as the defense against payload-sized allocation.

```ts
// memory bounded to a chunk + pipeline buffers, not the file size; backpressure + cancellation threaded through:
return new Response(Bun.file(reportPath).stream().pipeThrough(new CompressionStream('gzip')));
```

**Enforcement:** Reviewers reject reading large or unbounded payloads fully into memory; a streamed body with backpressure is required for files, large responses, and big query results ([chapter 02](./02-concurrency-and-event-loop.md) §2.4).

### 7.5 — Use schema-derived JSON serialization on hot routes — and only where measured.

**Reasoning, step by step:**
1. `JSON.stringify` walks an object reflectively on every call, and on a hot route that runs on the loop thread serialization is often a bigger line in the profile than the business logic it wraps — the cost [chapter 15](../typescript/15-performance.md)'s §15.8 names, and it is engine-independent, so it carries to JSC unchanged. A schema-derived serializer specializes the work to a known shape and skips the reflection, which is where the win comes from.
2. On the family's edge — Hono on `Bun.serve` ([chapter 03](./03-http-services.md) §3.1) — there is no compiled-serializer path, so the win is the §15.8 discipline directly: serialize less, skip fields a consumer ignores, and shape the handler's output to the response schema [§3.3](./03-http-services.md) already mandates so the outbound `.parse()` is walking a known, minimal shape; reach for a hand-rolled specialized serializer only where a profile demands it. (The Node-era path — Fastify compiling a response schema into a `fast-json-stringify` serializer — buys this for free, but Fastify is a legacy Node estate here (§3.2) and new Bun services do not start on it.)
3. Take the win only where a profile earned it. A schema-derived serializer on a cold or low-traffic route is complexity with no payoff, and like every deliberate optimization it lands with the benchmark that justifies it (15.8) and a ledger note (15.10). Default to plain `JSON` until serialization shows up in a `bun:jsc` sampling profile naming it (7.2).

```ts
// Hono on Bun.serve (3.1): no compiled serializer — the schema-shaped, minimal output IS the win (15.8).
app.get('/quotes', async (c) => c.json(QuoteListSchema.parse(await getQuotes()))); // 3.3 — reply pinned to a known shape
```

**Enforcement:** A schema-derived serializer lands with the profile (7.2, 15.6) and benchmark (15.8) that name serialization as the cost; default to `JSON`/standard route handling until then.

### 7.6 — Load-test before you ship a performance claim.

**Reasoning, step by step:**
1. "This is faster" is a result, not an adjective, and a `mitata` micro-benchmark of a function in isolation does not prove a *service* got faster — it ignores the event loop, connection reuse, GC under sustained pressure, and tail latency (7.2). The honest proof is a load test against the running service, the runtime analog of the JMH-grade rigor the JVM guide demands for its claims.
2. Use a load tool fast enough to keep up with `Bun.serve` — **`oha`** is this guide's pick (`bombardier` is an acceptable substitute); a Node-based tool is too slow to drive a Bun server honestly and will report the *tool's* ceiling as yours. Run it with a *stated scenario*: concurrency, duration, route, payload shape. State it so the run is reproducible and the next person measures the same thing rather than a differently shaped load that happens to agree.
3. Record p50 and p99 — never the average, which hides the tail — alongside throughput and **RSS**, in the PR's test plan ([git-and-code-review.md](../git-and-code-review.md)). A latency win bought by doubling memory is a trade-off the reviewer must see, not a number to bury. Run against production-shaped data on production-shaped hardware; a load test over toy input on a laptop proves nothing about production.

```bash
# stated scenario: 64 connections, 30s, the export route — reproducible, not a vibe.
oha -c 64 -z 30s http://localhost:3000/export   # fast enough for Bun.serve; capture p50/p99/req-s/RSS for the PR
```

**Enforcement:** A PR making a performance claim includes an `oha` (or `bombardier`) run with its scenario and p50/p99 + RSS in the test plan (git-and-code-review.md); a bare "it's faster," or a number from a Node-based load tool, does not merge.

### 7.7 — Watch RSS against the JS heap; size the deployment with `--smol` and container headroom.

**Reasoning, step by step:**
1. Bun runs two heaps, and they answer different questions. The **JS heap** is JSC's — what your objects occupy, read with `heapStats().heapSize` or `memoryUsage()` from `bun:jsc` (7.2). The **native heap** is everything else — `Buffer`s, the mimalloc allocator's arenas, sockets, native code — reportable with `MIMALLOC_SHOW_STATS=1`. **RSS** (`process.memoryUsage().rss`) is the whole resident process: both heaps plus stack and code. A gap between RSS and the JS heap that grows over time is a native-side leak (often unclosed handles or buffers) that a JS heap snapshot alone will not show; track both.
2. There is no `--max-old-space-size` here — that is a V8 knob, and JSC ignores it. JSC sizes its heap from its own heuristics, and like V8 it does *not* read the cgroup memory limit your container runs under, while the OOM-killer reads exactly that limit. So the process can grow resident memory toward a ceiling the kernel will not honor and die with `SIGKILL` and no stack trace instead of a recoverable error.
3. The lever Bun gives you is **`--smol`**: it trades performance for a smaller memory footprint — a tighter heap and more frequent GC — which is the right default for a memory-constrained container or a serverless box, and the wrong one for a throughput-bound service with headroom (it costs CPU, so it carries a 15.10 ledger note like any deliberate trade). Either way, set the *container's* memory limit to leave headroom above peak RSS for that native-side world from point 1: a 1GB container is not a 1GB JS heap. This is [chapter 15](../typescript/15-performance.md)'s allocation awareness made operational — the deployment's memory budget is a deliberate number tied to measured RSS, not a default left to chance.

```bash
# memory-constrained container: trade throughput for a smaller footprint, and leave RSS headroom for the native heap.
bun --smol server.ts    # MIMALLOC_SHOW_STATS=1 to size the native side; cap the container above measured peak RSS
```

**Enforcement:** Both RSS and the JS heap (`heapStats`/`memoryUsage`) are exported as metrics; the container memory limit is set above measured peak RSS with documented headroom, and `--smol` is a recorded trade-off (15.10), not a reflex. A service whose memory budget is a guess is sent back.

### 7.8 — The optimization ledger applies: every Bun-specific trick carries its measurement.

**Reasoning, step by step:**
1. A pinned fan-out bound, a `--smol` flag, a stream where a buffer read more simply, a schema-derived serializer — each is a deviation from the obvious default, and [chapter 15](../typescript/15-performance.md)'s §15.10 ledger rule says the deviation is earned only by a number that travels with the code. Bun tricks are not exempt; the runtime layer just adds new ones — and because the JSC-vs-V8 split (scope note) retires the old V8 folklore, the *measurement*, not the lore, is the only thing that justifies a tuning constant at all.
2. Write the justification at the site: what was slow, what the fix bought, how it was measured. `// limit(32) — bun:jsc profile was handshake-bound at unbounded fan-out; p99 210ms→28ms (oha)` is a ledger entry; `// tuned` is folklore. The next reader needs the evidence, not the adjective — and a concurrency bound or `--smol` flag with no recorded reason is exactly the kind of magic number this guide otherwise forbids.
3. This is the zero-technical-debt rule the [Bun overlay](./README.md) reaffirms, applied to performance: an unexplained tuning constant is debt, because the next person cannot tell a load-bearing value from a guess, so they fear-freeze cruft or break a real win by "cleaning it up." The comment resolves it, and acts as a filter — if you cannot produce the measurement, that is the signal the cleverness should be simplified away.

```ts
// limit(32) — bun:jsc sampling profile was handshake-bound on unbounded fan-out; p99 210ms→28ms via oha.
const limit = pLimit(32); // ledger entry per 15.10 (7.3, 7.6)
```

**Enforcement:** Reviewers require a measurement comment on any Bun-specific tuning constant or non-obvious perf construct, and simplify away cleverness that cannot cite one; the committed load test or profile (7.6, 15.6) is the ledger's backing evidence.

## Cross-references

- Event-loop budget, no blocking the request path, Workers for CPU, streams with backpressure: [02-concurrency-and-event-loop.md](./02-concurrency-and-event-loop.md) (§2.2 lag budget behind 7.1; §2.4 backpressure behind 7.4). Framework-agnostic handlers and per-route response schemas: [03-http-services.md](./03-http-services.md) (§3.3 behind 7.5). Pooled, bounded persistence clients: [04-persistence.md](./04-persistence.md).
- Measure-first, the committed micro-benchmark and its warm-JIT caveats, JSON serialization as a hot-path cost, the optimization ledger, and the allocation/batching rules that carry to JSC: [../typescript/15-performance.md](../typescript/15-performance.md) (§15.4, §15.5, §15.6, §15.8, §15.10). Batched fan-out (§9.8) and the bounded-concurrency worker pool (§9.7); deadlines on every external wait (§9.6): [../typescript/09-concurrency.md](../typescript/09-concurrency.md).
- Connection pooling, the resource hierarchy, caching, and design-phase doctrine, canonical for all of the above: [../performance.md](../performance.md). Load tests with p50/p99 + RSS in the PR test plan: [../git-and-code-review.md](../git-and-code-review.md). The JVM mirror — JFR and async-profiler kept to the same measure-first discipline: [../kotlin-jvm/07-jvm-performance.md](../kotlin-jvm/07-jvm-performance.md).
