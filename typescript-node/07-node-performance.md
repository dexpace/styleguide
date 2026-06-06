# 07 — Node Performance

The core guide's [chapter 15](../typescript/15-performance.md) is the V8 layer: hidden classes, allocation rate, batched awaits, the measure-first ethic. This chapter is the layer below *that* — the runtime's own pressure points. One event loop that a single slow span freezes for every client; a process whose heap V8 sizes from a default the OOM-killer never agreed to; an HTTP client whose global dispatcher quietly caps your throughput. The discipline is the one the JVM guide draws on JFR and async-profiler to keep — name the tool to the question, measure before you tune, record the number that earned the trick — ported to the tools Node ships. Read [chapter 15](../typescript/15-performance.md) first; this is where its rules meet the event loop, the heap limit, and the socket pool.

## What good looks like

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';
import { Agent } from 'undici';

// Alert fired: p99 event-loop lag 180ms on the export route (7.1). The loop was stalling,
// so every concurrent client stalled with it — the Node analog of a stop-the-world pause.
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable(); // sampled, exported as a metric, alerted at budget (2.2) — not eyeballed in dev

// Triage with the platform's own profiler, not a guess (7.2): node --cpu-prof, open in speedscope.
// The flame showed ~70% of wall time in TLS handshakes inside the report fan-out — the default
// global undici dispatcher opens a fresh connection per request: no keep-alive, no ceiling (7.3).

// before: one connection per call (the bare global dispatcher), handshakes dragging the loop into lag.
// after: an explicit pool — keep-alive amortizes the handshake, connections bound the fan-out,
// timeouts cap every wait so a slow upstream can't grow an unbounded queue (7.3, 9.6).
const pool = new Agent({
  connections: 64,            // bounded; a ceiling tuned to upstream headroom, not the input size
  keepAliveTimeout: 10_000,   // reuse the socket across requests — kill the per-call TLS cost
  headersTimeout: 5_000,
  bodyTimeout: 5_000,         // every external wait is deadlined (9.6)
});

// autocannon -c 64 -d 30 http://localhost:3000/export  (production-shaped load, 7.6)
//   before: p50 41ms · p99 210ms · 1.2k req/s · RSS 690MB · loop-lag p99 180ms
//   after:  p50 12ms · p99  28ms · 9.8k req/s · RSS 240MB · loop-lag p99   6ms
```

The fix the profile actually rewarded was the connection pool, not a hand-tuned loop: the `--cpu-prof` flame named TLS setup as the cost, an explicit `undici` `Agent` amortized it, and `autocannon` proved the delta with p50/p99 and RSS — the ledger entry justifying the deviation from the one-line default (15.10). Each number traces to a rule: lag watched and budgeted (7.1), the right profiler for the question (7.2), a bounded keep-alive pool over the global dispatcher (7.3), the load test that earns the claim (7.6), every wait deadlined as [chapter 09](../typescript/09-concurrency.md)'s §9.6 requires.

## Rules

### 7.1 — Event-loop lag is the GC-pause analog: measure it, budget it, alert on it.

**Reasoning, step by step:**
1. A JVM team watches GC pauses because a stop-the-world collection freezes every thread; a Node team watches *event-loop lag* for the same reason, because one thread serves every request and a synchronous span freezes all of them at once. Lag is the single number that says "the loop is starving," and it is the runtime's defining health metric — the direct port of [chapter 15](../typescript/15-performance.md)'s GC-pressure focus to a runtime with one thread.
2. Measure it with the platform primitive, not a wall-clock guess: `monitorEventLoopDelay({ resolution })` from `node:perf_hooks` keeps a histogram you read as `mean`, `max`, and `percentile(99)` in nanoseconds. Sample it on an interval, not per request, so the measurement does not become the load.
3. Budget it and alert before clients feel it. A few milliseconds of p99 lag under load is healthy; tens of milliseconds means a synchronous stretch is blocking the loop and belongs off-thread (a `worker_threads` job, a stream, a smaller batch). The metric exists to fire an alert at a threshold, which is why this rule lives next to chapter 02's §2.2 event-loop budget — measuring lag is how you prove the budget is kept.

```ts
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => metrics.gauge('event_loop_lag_p99_ms', h.percentile(99) / 1e6), 5_000).unref();
```

**Enforcement:** Event-loop lag is sampled, exported as a metric, and alerted at a documented budget (chapter 02 §2.2); a service that ships without the gauge is sent back.

### 7.2 — Profile with the platform's own tools first; name the tool to the question.

**Reasoning, step by step:**
1. Node performance intuition is wrong about as often as the JVM's — the event loop, V8's optimizer, and GC interact non-locally, so "this is the slow part" is a hypothesis until a profile shows it. This is [chapter 15](../typescript/15-performance.md)'s §15.6 measure-first rule at the runtime layer, and the JVM guide's reflex applies unchanged: capture a profile that *shows the issue* before, another after, and let the delta be the proof.
2. Match the tool to the question instead of reaching for one by habit, exactly as a JVM team picks JFR versus async-profiler by what it is asking. For **CPU**: `node --cpu-prof` writes a `.cpuprofile` you open in speedscope or Chrome DevTools — the flame answers "where is wall time going." For **memory**: `--heap-prof` for an allocation profile, or a heap snapshot (`v8.writeHeapSnapshot()`, or DevTools) compared across time to find what never frees — the answer to "what is growing" (7.7). For **first-look triage**: `clinic` (`doctor` to classify, `flame` for CPU, `bubbleprof` for async) or `0x` for a one-command flamegraph, used to *locate* the problem before the precise tool measures it.
3. Profile production-shaped data. A flamegraph over toy input proves nothing about the hot path that exists under real payloads, and the cheapest way to chase a phantom is to optimize a profile of the wrong workload.

```bash
node --cpu-prof server.js    # CPU: open the .cpuprofile in speedscope — "where is wall time?"
node --heap-prof server.js   # allocations: "what is growing?" — or v8.writeHeapSnapshot() across time
clinic doctor -- node server.js   # triage first: classify, then reach for the precise tool (or 0x)
```

**Enforcement:** A perf change cites which tool was used and why; a CPU micro-fix proposed before a `--cpu-prof` capture names CPU as the bottleneck is sent back (15.6).

### 7.3 — Reach for `undici` with an explicit pool; the global dispatcher is not a strategy.

**Reasoning, step by step:**
1. Connection setup is expensive — TCP handshake, TLS negotiation, sometimes auth — and the core guide's pooling doctrine ([root rule on connection pooling](../performance.md)) says reuse it. `undici` is the runtime's high-performance HTTP client and the `fetch` engine; the lever that matters is the *dispatcher*, and its global default is tuned for "works," never for your workload.
2. Construct an `Agent` (or `Pool` per origin) with the knobs set deliberately: `connections` bounds in-flight sockets to the upstream's headroom so a fan-out cannot become a connection storm; `keepAliveTimeout` holds sockets open so the handshake amortizes across requests instead of repeating per call; `headersTimeout` and `bodyTimeout` deadline every wait so a slow dependency surfaces as an error, not an unbounded queue (cross-ref [chapter 09](../typescript/09-concurrency.md) §9.6). Set it once and route calls through it — the same singleton discipline [chapter 15](../typescript/15-performance.md) mandates for any pooled client.
3. The default global dispatcher leaves all of this unset: effectively unbounded concurrency, no tuned keep-alive, library-default timeouts. That is a strategy by omission, and under load it shows up as the handshake cost and queue growth the exemplar's profile caught.

```ts
import { Agent, setGlobalDispatcher, request } from 'undici';
const agent = new Agent({ connections: 64, keepAliveTimeout: 10_000, headersTimeout: 5_000, bodyTimeout: 5_000 });
setGlobalDispatcher(agent); // configured once; bounded, keep-alive, deadlined — not the bare default
const { body } = await request(`${UPSTREAM}/users/${id}`, { dispatcher: agent });
```

**Enforcement:** Reviewers reject HTTP calls on the unconfigured global dispatcher; an explicit `Agent`/`Pool` with bounded `connections`, keep-alive, and header/body timeouts is the default.

### 7.4 — Stream large payloads; buffering a 2GB export is an OOM with extra steps.

**Reasoning, step by step:**
1. Reading a large file, response, or query result fully into memory before handling it scales the process's RSS with the *payload*, not with a bound you chose. A 2GB export buffered into a `Buffer` or string is an out-of-memory crash waiting for the largest input — and the OOM-killer arrives without a stack trace.
2. Process it as a stream so memory stays bounded to one chunk plus the pipeline's buffers regardless of total size. Compose stages with `stream.pipeline` (or `pipeline` from `node:stream/promises`), which threads errors and cleanup through every stage so a failure mid-stream does not leak a half-open handle — the manual `.pipe()` chain drops both.
3. Honor backpressure: `pipeline` propagates it automatically, pausing the source when a downstream stage falls behind so a fast producer cannot outrun a slow consumer into unbounded memory. This is the same bounded-buffer discipline as chapter 02's §2.4 streams-with-backpressure rule, here as the defense against payload-sized allocation.

```ts
import { pipeline } from 'node:stream/promises'; // createReadStream from node:fs, createGzip from node:zlib
// memory bounded to a chunk + pipeline buffers, not the file size; errors and cleanup threaded through:
await pipeline(createReadStream(reportPath), createGzip(), res);
```

**Enforcement:** Reviewers reject reading large or unbounded payloads fully into memory; `stream.pipeline` with backpressure is required for files, large responses, and big query results (chapter 02 §2.4).

### 7.5 — Use schema-derived JSON serialization on hot routes — and only where measured.

**Reasoning, step by step:**
1. `JSON.stringify` walks an object reflectively on every call, and on a hot route that runs on the main thread serialization is often a bigger line in the profile than the business logic it wraps — the cost [chapter 15](../typescript/15-performance.md)'s §15.8 names. A schema-derived serializer specializes the work to a known shape and skips the reflection, which is where the win comes from.
2. On `fastify`, a response schema activates the `fast-json-stringify` path: declare the shape the route returns and the framework compiles a serializer for it. This is the response-schema discipline of chapter 03's §3.3 paying a second dividend — the schema you already write for contract and validation also buys the faster serializer, for free.
3. Take the win only where a profile earned it. A schema-derived serializer on a cold or low-traffic route is complexity with no payoff, and like every deliberate optimization it lands with the benchmark that justifies it (15.8) and a ledger note (15.10). Default to plain `JSON` until serialization shows up in a `--cpu-prof` capture naming it.

```ts
// the response schema doubles as the fast-json-stringify spec (3.3): contract + a compiled serializer.
app.get('/quotes', { schema: { response: { 200: QuoteListSchema } } }, async () => getQuotes());
```

**Enforcement:** A schema-derived serializer lands with the profile (15.6) and benchmark (15.8) that name serialization as the cost; default to `JSON`/standard route handling until then.

### 7.6 — Load-test before you ship a performance claim.

**Reasoning, step by step:**
1. "This is faster" is a result, not an adjective, and a micro-benchmark of a function in isolation does not prove a *service* got faster — it ignores the event loop, connection pools, GC under sustained pressure, and tail latency. The honest proof is a load test against the running service, the runtime analog of the JMH-grade rigor the JVM guide demands for its claims.
2. Use `autocannon` (or `k6`) with a *stated scenario*: concurrency, duration, route, payload shape. State it so the run is reproducible and the next person measures the same thing rather than a differently shaped load that happens to agree.
3. Record p50 and p99 — never the average, which hides the tail — alongside throughput and **RSS**, in the PR's test plan ([git-and-code-review.md](../git-and-code-review.md)). A latency win bought by doubling memory is a trade-off the reviewer must see, not a number to bury. Run against production-shaped data on production-shaped hardware; a load test over toy input on a laptop proves nothing about production.

```bash
# stated scenario: 64 connections, 30s, the export route — reproducible, not a vibe.
autocannon -c 64 -d 30 http://localhost:3000/export   # capture p50/p99/req-s/RSS for the PR test plan
```

**Enforcement:** A PR making a performance claim includes an `autocannon`/`k6` run with its scenario and p50/p99 + RSS in the test plan (git-and-code-review.md); a bare "it's faster" does not merge.

### 7.7 — Watch RSS against `heapUsed`, and set `--max-old-space-size` to the container limit minus headroom.

**Reasoning, step by step:**
1. Two memory numbers matter and they answer different questions. `process.memoryUsage().heapUsed` is V8's JS heap — what your objects occupy. **RSS** is the whole process's resident memory: the heap *plus* the off-heap world V8's heap stat never sees — `Buffer`s, native addons, the `undici` socket pools, stack, code. A gap between them that grows over time is an off-heap leak (often unclosed handles or buffers), which a heap snapshot alone will not show; track both (7.2).
2. V8 sizes its old-space heap from an internal default, and *by default it does not read the cgroup memory limit your container runs under*. The OOM-killer, however, reads exactly that limit. So V8 will happily let the heap grow toward a ceiling the kernel will not honor, and the process dies with `SIGKILL` and no stack trace instead of throwing a recoverable allocation error.
3. Close the gap explicitly: set `--max-old-space-size=<MB>` to the container's memory limit *minus headroom* for that off-heap world from point 1. A 1GB container is not a 1GB heap — leave room for `Buffer`s, native memory, and the socket pools, or the OOM-killer takes the difference. This is [chapter 15](../typescript/15-performance.md)'s allocation awareness made operational: the limit is a deliberate number tied to the deployment, not a default left to chance.

```bash
# 1GB container: cap V8's heap at 768MB, leaving ~256MB for off-heap (Buffers, native, undici pools).
NODE_OPTIONS="--max-old-space-size=768" node server.js   # the cgroup limit minus headroom, set on purpose
```

**Enforcement:** `--max-old-space-size` is set explicitly to the container limit minus headroom in the deploy manifest; both RSS and `heapUsed` are exported as metrics. A service relying on V8's default heap sizing under a cgroup limit is sent back.

### 7.8 — The optimization ledger applies: every Node-specific trick carries its measurement.

**Reasoning, step by step:**
1. A pinned `connections` count, an explicit `--max-old-space-size`, a stream where a buffer read more simply, a schema-derived serializer — each is a deviation from the obvious default, and [chapter 15](../typescript/15-performance.md)'s §15.10 ledger rule says the deviation is earned only by a number that travels with the code. Node tricks are not exempt; the runtime layer just adds new ones.
2. Write the justification at the site: what was slow, what the fix bought, how it was measured. `// connections: 64 — autocannon showed handshake-bound at the default; p99 210ms→28ms` is a ledger entry; `// tuned pool` is folklore. The next reader needs the evidence, not the adjective — and a pool size or heap cap with no recorded reason is exactly the kind of magic number this guide otherwise forbids.
3. This is the zero-technical-debt rule the [Node overlay](./README.md) reaffirms, applied to performance: an unexplained tuning constant is debt, because the next person cannot tell a load-bearing value from a guess, so they fear-freeze cruft or break a real win by "cleaning it up." The comment resolves it, and acts as a filter — if you cannot produce the measurement, that is the signal the cleverness should be simplified away.

```ts
// connections: 64 — --cpu-prof was TLS-handshake-bound on the global dispatcher; p99 210ms→28ms via autocannon.
const agent = new Agent({ connections: 64, keepAliveTimeout: 10_000 }); // ledger entry per 15.10 (7.3, 7.7)
```

**Enforcement:** Reviewers require a measurement comment on any Node-specific tuning constant or non-obvious perf construct, and simplify away cleverness that cannot cite one; the committed load test or profile (7.6, 15.6) is the ledger's backing evidence.

## Cross-references

- Event-loop budget, no blocking the request path, `worker_threads` for CPU, streams with backpressure: [02-concurrency-and-event-loop.md](./02-concurrency-and-event-loop.md) (§2.2 lag budget behind 7.1; §2.4 backpressure behind 7.4). Framework-agnostic handlers and per-route response schemas: [03-http-services.md](./03-http-services.md) (§3.3 behind 7.5). Pooled, bounded persistence clients: [04-persistence.md](./04-persistence.md).
- Measure-first and `node --cpu-prof` + speedscope, the committed benchmark, JSON serialization as a hot-path cost, the optimization ledger: [../typescript/15-performance.md](../typescript/15-performance.md) (§15.6, §15.8, §15.10). Batched fan-out (§9.8) and the bounded-concurrency worker pool (§9.7); deadlines on every external wait (§9.6): [../typescript/09-concurrency.md](../typescript/09-concurrency.md).
- Connection pooling, the resource hierarchy, caching, and design-phase doctrine, canonical for all of the above: [../performance.md](../performance.md). Load tests with p50/p99 + RSS in the PR test plan: [../git-and-code-review.md](../git-and-code-review.md). The JVM mirror — JFR and async-profiler kept to the same measure-first discipline: [../kotlin-jvm/07-jvm-performance.md](../kotlin-jvm/07-jvm-performance.md).
