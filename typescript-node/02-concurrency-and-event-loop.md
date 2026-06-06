# 02 — Concurrency and the Event Loop

The core guide's [chapter 09](../typescript/09-concurrency.md) keeps a single async program correct: every promise accounted for, every wait deadlined, every fan-out ceilinged. This chapter is about the runtime that program runs *inside*. A Node process is one event loop serving every concurrent request off one thread, so the rules here protect that shared thread — nothing synchronous stalls it, CPU work leaves it, backpressure bounds it, an unknown fault crashes it cleanly, SIGTERM drains it on a deadline. Where this chapter is stricter than [09](../typescript/09-concurrency.md), it wins for Node services.

## What good looks like

```ts
// server.ts — the process lifecycle: one loop, drained on a deadline, crash-only on the unknown.
const SHUTDOWN_DEADLINE_MS = 25_000; // illustrative; the real value comes from ch.05 config (SHUTDOWN_DEADLINE_MS), kept < the orchestrator's SIGKILL grace (2.6)
const controller = new AbortController(); // owns intervals, drains, in-flight tracking (2.9)

async function shutdown(signal: NodeJS.Signals): Promise<void> {
  log.info({ signal }, 'shutdown: draining');
  controller.abort(); // stop timers, intervals, background loops at once (2.9)
  const deadline = AbortSignal.timeout(SHUTDOWN_DEADLINE_MS); // bounds the drain; the catch forces exit on a hung close()/end() (2.6)
  try {
    await server.close();                          // 1. stop accepting new connections
    await drainInFlight({ signal: deadline });     // 2. let live requests finish, or time out (2.8)
    await pool.end();                              // 3. close DB pool after its last user (typescript/13.7)
  } catch (err: unknown) {
    log.error({ err }, 'shutdown: drain exceeded deadline; forcing exit');
    process.exit(1); // a stuck drain must not outlive the deadline
  }
  process.exit(0);
}

for (const sig of ['SIGTERM', 'SIGINT'] as const) {
  process.once(sig, () => void shutdown(sig)); // once: a second signal must not re-enter (2.6)
}

// Crash-only: an unknown fault leaves unknown state. Log, flush, exit — the supervisor restarts (2.5).
process.on('unhandledRejection', (reason) => { log.fatal({ reason }, 'unhandledRejection'); process.exit(1); });
process.on('uncaughtException', (err) => { log.fatal({ err }, 'uncaughtException'); process.exit(1); });
```

The sequence is ordered and deadline-bounded (2.6): stop accepting, drain what's live (2.8), then close pools last (mirroring [typescript/13.7](../typescript/13-resource-management.md)'s reverse-acquisition release); one `AbortController` is the shutdown switch every timer and loop listens to (2.9); and the two crash handlers refuse to limp — an unknown fault exits non-zero so the supervisor can replace the process from a clean state (2.5).

## Rules

### 2.1 — No `*Sync` API on a request path.

**Reasoning, step by step:**
1. `readFileSync`, `crypto.*Sync`, `zlib.*Sync`, `execSync`, `spawnSync` block the *event-loop thread*, not just the calling request. While that one call runs, every other in-flight request is frozen — the loop cannot resume a single pending `await` until it returns.
2. A 30ms synchronous read looks harmless alone. Under 500 req/s it is a 30ms tax on the tail of every concurrent request, and it surfaces as event-loop lag (2.2), not a slow handler — hard to attribute after the fact. The legitimate home for `*Sync` is startup — config, a TLS cert, a path resolved *before* the server listens, when there is no concurrent request to starve; once `server.listen()` has run, async is the only form.

```ts
import { readFile } from 'node:fs/promises';
const cert = readFileSync('tls.pem');          // fine at startup, before listen()
const body = await readFile(path, 'utf8');     // request path: async, the loop stays free
```

**Enforcement:** `no-restricted-syntax` banning `*Sync` identifiers in `src/**` outside a marked `bootstrap/` area, plus review.

### 2.2 — Event-loop lag is a first-class metric.

**Reasoning, step by step:**
1. Event-loop lag — the delay between when a timer or I/O callback *should* run and when the loop gets to it — is the truest health signal a Node service has (NODE-2). High lag means the thread is busy synchronously, and a busy thread serves no one; request latency follows lag.
2. Measure it with `monitorEventLoopDelay()` from `node:perf_hooks`, exported next to your request metrics. The default budget is **p99 lag < 50ms**; sustained breach is an alert, not a dashboard curiosity, because it predicts the timeouts users are about to see. A breach points at exactly this chapter's defects — a `*Sync` call (2.1), CPU work that belongs on a worker (2.3), a hot loop that never yields (2.7) — so the metric is how you find them in production instead of guessing.

```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';
const lag = monitorEventLoopDelay({ resolution: 20 });
lag.enable();
setInterval(() => metrics.gauge('eventloop.lag.p99.ms', lag.percentile(99) / 1e6), 5_000).unref();
```

**Enforcement:** review; `monitorEventLoopDelay()` wired at startup, p99 exported, alert on the stated budget (NODE-2).

### 2.3 — CPU-bound work goes to `worker_threads` via a bounded pool.

**Reasoning, step by step:**
1. Hashing, image resizing, large JSON transforms, compression — CPU-bound spans hold the one thread for their whole duration. There is no `await` inside a tight computation for the loop to interleave on (restated from [typescript/9.1](../typescript/09-concurrency.md)), so every concurrent request waits for it to finish.
2. Move that work to `worker_threads` and manage them with **piscina**: a pool sized explicitly to the box's cores, with a bounded task queue. The loop serves I/O while workers serve math, and the pool's ceiling means a burst of CPU jobs queues against a known bound instead of spawning unbounded threads (root rule 9). Size it to `availableParallelism()`, not request volume — more workers than cores just thrashes the scheduler — and pin `maxThreads` explicitly, because a container's cgroup CPU quota can differ from what the default detects, so leaving it implicit risks a pool sized for the host's cores rather than the ones the orchestrator actually grants (root rule 2). Bound the queue so that at the limit you shed (2.8), not grow.

```ts
import Piscina from 'piscina';
import { availableParallelism } from 'node:os';
const pool = new Piscina({
  filename: new URL('./workers/resize.js', import.meta.url).href,
  maxThreads: availableParallelism(), // the loop serves I/O; workers serve CPU
  maxQueue: 1_000,                    // bounded queue; overflow rejects, not grows (2.8)
});
const out = await pool.run(job, { signal }); // signal propagates down (typescript/9.11)
```

**Enforcement:** review; CPU-bound spans run on a piscina pool sized from `availableParallelism()` with a bounded `maxQueue`, never inline on the loop.

### 2.4 — Backpressure is honored end to end.

**Reasoning, step by step:**
1. A producer that emits faster than the consumer can accept *is* an unbounded queue — the data piles up in memory between them until the process dies. Streams exist to apply backpressure: when the destination is full, the source pauses. Skipping that is the OOM with a delay from [typescript/9.12](../typescript/09-concurrency.md), wearing a stream's clothes.
2. Wire pipelines with `stream.pipeline` (or async iterators), never a bare `source.pipe(dest)`. `pipeline` propagates errors and backpressure across every stage and destroys the whole chain on failure; a lone `.pipe()` drops errors on the floor and leaks half-open handles when one stage fails. Async iteration (`for await (const chunk of source)`) gives the same backpressure for free — the loop body's `await` is the pause signal — and reads better when the transform is naturally a loop.

```ts
import { pipeline } from 'node:stream/promises';
await pipeline(req, createGunzip(), parseNdjson(), writeToSink(), { signal }); // backpressure + errors across all stages
req.pipe(sink); // bad: no error path, no teardown on failure — a leak and a swallowed error
```

**Enforcement:** review; `stream.pipeline`/async iterators for every multi-stage stream, never a bare `.pipe()` without error and teardown wiring.

### 2.5 — `unhandledRejection` and `uncaughtException`: log, flush, exit 1.

**Reasoning, step by step:**
1. These two events fire when a fault escaped every local handler — a promise no one caught, a throw outside any `try`. By definition the process is now in a state no code anticipated. This is the crash-only policy (NODE-1): the only safe assumption about unknown state is that it is corrupt.
2. A process that catches the unknown and keeps serving makes decisions from that corrupt state, and it does so *from a position of trust* — clients believe its responses. That is strictly worse than being down. Log the fault with full context, flush the logger, and `process.exit(1)`; the supervisor (Kubernetes, systemd, a process manager) restarts a clean instance (the exemplar shows the two-line shape). Do not abuse these as a catch-all for operational errors — those belong handled at their call site ([typescript/08](../typescript/08-error-handling.md)); the last-resort handler exists to *crash cleanly*, not to paper over routine failures.

**Enforcement:** review; both handlers present, each logs with context and exits non-zero — never a `return` that resumes serving (NODE-1).

### 2.6 — Graceful shutdown is ordered and deadline-bounded.

**Reasoning, step by step:**
1. SIGTERM is a contract with the orchestrator: *you have N seconds to finish, then I send SIGKILL.* Honoring it means draining in-flight work so live requests get answers instead of reset connections — and finishing inside the grace period, because anything still running when SIGKILL lands is killed mid-flight anyway.
2. The order is fixed (the exemplar): stop accepting (`server.close()`), drain in-flight on a deadline (2.8), then close pools and clients *after* their last user — reverse-acquisition release, mirroring [typescript/13.7](../typescript/13-resource-management.md). The drain carries an `AbortSignal.timeout`, and the forced exit on overrun is the backstop for a phase that ignores it — a hung `close()`/`end()`; an unbounded drain just trades a fast restart for a hung one, so set the deadline below the orchestrator's grace and let cleanup win the race.
3. Register with `process.once`, not `process.on`: a second SIGTERM during shutdown must not re-enter the sequence. If the deadline is exceeded, force `process.exit(1)` — a stuck drain is itself a fault, and limping past it defeats the contract.

**Enforcement:** review; SIGTERM/SIGINT handled with `once`, phases ordered, the drain bounded by `AbortSignal.timeout`, deadline below the orchestrator's grace, forced exit on overrun.

### 2.7 — Long-lived loops yield.

**Reasoning, step by step:**
1. A batch job iterating a million rows with no `await` is a `*Sync` call in disguise (2.1): it holds the thread for its entire run, and every request the process is also serving is frozen until it finishes. Starvation here is self-inflicted DoS — your own background work taking the server down.
2. Yield between chunks so the loop can serve I/O in the gaps. `await setImmediate()` returns control after pending I/O callbacks; `await scheduler.yield()` (from `node:timers/promises`) is the clearer spelling where available. Yield per *batch*, not per row — yielding too often is its own overhead. Pair the yield with `signal.throwIfAborted()` at the top of each chunk ([typescript/9.11](../typescript/09-concurrency.md)) so a shutdown (2.6) or client disconnect stops the loop promptly, not after the last row.

```ts
import { setImmediate as yieldToLoop } from 'node:timers/promises';
for (let i = 0; i < rows.length; i += BATCH) {
  signal.throwIfAborted();           // stop on shutdown/cancel (typescript/9.11)
  await processBatch(rows.slice(i, i + BATCH));
  await yieldToLoop();               // hand the loop back to in-flight requests
}
```

**Enforcement:** review; CPU-heavy loops `await setImmediate()` (or `scheduler.yield()`) per batch and check `throwIfAborted()`.

### 2.8 — In-flight request tracking is bounded; shed load over queueing.

**Reasoning, step by step:**
1. Under overload, the honest failure is to serve *some* requests well and reject the rest fast — not to queue everything and serve all of them slowly past their timeout. An unbounded accept queue is root rule 9 ignored: it grows until memory dies, and by then every queued request has already timed out at the client.
2. Track in-flight requests against a named ceiling. At the bound, respond `503 Service Unavailable` with a `Retry-After` header — load-shedding tells the caller (and its load balancer) to back off or try another instance, which is recoverable; silent queueing past the deadline is not. A bounded in-flight set is also what makes shutdown finite: it is a bounded drain (2.6), and you cannot deadline-bound a drain over a queue that had no limit going in.

```ts
const MAX_INFLIGHT = 512; // named ceiling; sized from memory + p99 latency (root rule 9)
let inFlight = 0;
app.addHook('onRequest', async (_req, reply) => {
  if (inFlight >= MAX_INFLIGHT) { reply.header('retry-after', '1').code(503); throw new ServiceUnavailable(); }
  inFlight++;
});
app.addHook('onResponse', async () => { inFlight--; });
```

**Enforcement:** review; in-flight work bounded by a named constant, overflow sheds with `503` + `Retry-After`, never an unbounded accept queue.

### 2.9 — Timers and intervals tie to lifecycle signals.

**Reasoning, step by step:**
1. A bare `setInterval` is a leak with an alarm clock (restated from [typescript/13.5](../typescript/13-resource-management.md)): it fires forever, pins its closure, and on Node keeps the loop from exiting — so it also breaks graceful shutdown by keeping the process alive past the drain.
2. Every long-lived timer is owned by the shutdown `AbortController` from the exemplar. Clear it on abort — `signal.addEventListener('abort', () => clearInterval(id), { once: true })` — so one `controller.abort()` in `shutdown()` (2.6) stops every background interval at once, and the timer dies with the lifecycle that created it. For metrics or health timers that should not by themselves keep the process alive, add `.unref()` (the 2.2 example does) — a liveness hint, not cleanup, so you still clear on abort to release the closure; the two concerns are independent ([typescript/13.5](../typescript/13-resource-management.md)).

```ts
const id = setInterval(() => void flushMetrics(), 5_000);
controller.signal.addEventListener('abort', () => clearInterval(id), { once: true }); // dies with the lifecycle (2.6)
```

**Enforcement:** review; no `setInterval`/`setTimeout` whose handle is discarded — each clears on the owning shutdown signal, `unref` used only as a liveness hint alongside the clear.

## Cross-references

- Promise discipline, `AbortSignal.timeout` (9.6), bounded fan-out (9.7), signal propagation (9.11), bounded queues (9.12): [typescript/09](../typescript/09-concurrency.md); `AbortController` as the lifecycle handle, timer ownership (13.5), reverse-order release (13.7): [typescript/13](../typescript/13-resource-management.md).
- Crash-only policy (NODE-1) and operational vs programmer errors: [typescript/08](../typescript/08-error-handling.md), [node README](./README.md). The Fastify request lifecycle these hooks attach to: [03 — HTTP services](./03-http-services.md); runtime and one-process-per-container: [01 — runtime and modules](./01-runtime-and-modules.md).
