# 13 — Resource Management

Resources — sockets, file handles, database connections, timers, subscriptions, locks — outlive the call that opens them unless something explicitly closes them, and the explicit thing must run on the exceptional path too. TypeScript 5.2 made this a language feature: `using` and `await using` bind a resource's lifetime to its lexical scope, the way `use {}` does in Kotlin and `with` does in Python. This chapter is the rule that owned resources declare their own disposal, compose it deterministically, and never wait on the garbage collector.

## What good looks like

```ts
class MetricsReporter implements AsyncDisposable {
  readonly #stack = new AsyncDisposableStack();
  readonly #controller = new AbortController();

  static async start(endpoint: URL, source: EventTarget): Promise<MetricsReporter> {
    const self = new MetricsReporter();
    const signal = self.#controller.signal;

    await using bootstrap = new AsyncDisposableStack(); // unwinds if anything below throws
    const dial = AbortSignal.any([signal, AbortSignal.timeout(3_000)]); // 13.4 timeout + lifecycle, one handle
    const conn = await openConnection(endpoint, { signal: dial });
    bootstrap.use(conn); // implements Symbol.asyncDispose (13.2)

    const timer = setInterval(() => void conn.flush().catch(reportError), 5_000);
    bootstrap.defer(() => clearInterval(timer)); // 13.5 timer has an owner

    source.addEventListener('shutdown', () => self.#controller.abort(), { signal }); // 13.4

    self.#stack.use(bootstrap.move()); // commit: ownership transfers, reverse-order on dispose (13.3, 13.7)
    return self;
  }

  async [Symbol.asyncDispose](): Promise<void> {
    this.#controller.abort(); // cancel fetches, timers, listeners at once (13.4)
    await this.#stack.disposeAsync(); // releases in reverse acquisition order (13.7)
  }
}

// Call site: scope is lifetime. No manual close, no leak on throw.
await using reporter = await MetricsReporter.start(endpoint, process);
```

One `AbortController` is the lifecycle handle for the fetch, the interval, and the listener (13.4); the connection's disposal is part of its type (13.2); the `AsyncDisposableStack` composes teardown so release mirrors setup with no `try/finally` pyramid (13.3, 13.7); `bootstrap.move()` makes construction all-or-nothing — a throw before the move unwinds everything acquired so far; and the `await using` at the call site ties the reporter's life to its block (13.1).

## Rules

### 13.1 — `using` / `await using` for every disposable.

**Reasoning, step by step:**
1. `using conn = openConnection()` disposes `conn` when the block exits — by `return`, by `throw`, by a break. Disposal runs in reverse declaration order at end of scope. The scope *is* the lifetime; there is no path that skips cleanup.
2. This is the direct analog of Kotlin's `use {}` and Python's `with`. Manual `try { … } finally { conn.close() }` expresses the same intent in four more lines, and the day someone adds an early `return` above the `finally` the resource leaks. The keyword cannot be forgotten the way a `finally` can.
3. `await using` is the asynchronous form: it invokes `Symbol.asyncDispose` and awaits it. Use it for anything whose teardown is async — connections, async iterators, server handles. Use plain `using` only when disposal is synchronous.
4. Disposal still runs if the body throws; if disposal itself throws, the runtime wraps both in `SuppressedError` — the disposal failure becomes the primary error and the original is preserved as `.suppressed`, never silently lost. So `await using cursor = await db.openCursor(query)` disposes `cursor` even when the code that consumes it throws.

**Enforcement:** review; a locally-scoped resource with a `Symbol.dispose`/`Symbol.asyncDispose` member is bound with `using`/`await using`, never closed by hand. The disposable types need an explicit `"lib": ["es2023", "esnext.disposable"]` entry in `tsconfig` — a platform setting riding the same lane as ch01's `module: nodenext` override, not a seventh strictness flag (1.x).

### 13.2 — Owned resources implement `Symbol.dispose` / `Symbol.asyncDispose`.

**Reasoning, step by step:**
1. If a class owns something that must be released, disposal belongs in its *type*, not in a README sentence the caller is trusted to have read. A class that implements `Disposable` or `AsyncDisposable` advertises its contract in the signature, and the compiler enforces that `using` only binds things that are actually disposable.
2. Implement the method, do not expose a public `close()` as the primary interface. A bare `close()` is an instruction; `[Symbol.dispose]()` is a guarantee the language wires into `using`. If a legacy `close()` must remain for an existing caller, make `[Symbol.dispose]` delegate to it so there is one teardown path.
3. Make disposal idempotent. Guard with a `#disposed` flag and return early on the second call; double-dispose is legal and a `DisposableStack` plus an explicit `using` can both target the same object during a migration.
4. After disposal, other methods fail loudly with an `invariant` (05) — `invariant(!this.#disposed, 'use after dispose')` — rather than silently operating on a closed handle.

```ts
class TempDir implements AsyncDisposable {
  #disposed = false;
  private constructor(readonly path: string) {}
  static async create(): Promise<TempDir> {
    return new TempDir(await fs.mkdtemp(join(tmpdir(), 'app-')));
  }
  async [Symbol.asyncDispose](): Promise<void> {
    if (this.#disposed) return; // idempotent
    this.#disposed = true;
    await fs.rm(this.path, { recursive: true, force: true });
  }
}
```

**Enforcement:** review; a class holding an unmanaged resource implements `Disposable`/`AsyncDisposable`, disposal is idempotent, and post-dispose use is asserted against.

### 13.3 — `DisposableStack` / `AsyncDisposableStack` for composite teardown.

**Reasoning, step by step:**
1. A unit that acquires three resources should not nest three `try/finally` blocks. That pyramid grows quadratically with resource count and gets the unwind order wrong under edits. A stack flattens it.
2. `stack.use(resource)` registers a disposable; `stack.defer(() => …)` registers a teardown callback; `stack.adopt(value, dispose)` registers a value with an external disposer. Disposing the stack runs every registration in reverse order (13.7), and one registration throwing does not skip the rest.
3. `stack.move()` transfers ownership to a fresh stack and disarms the original. This is the all-or-nothing constructor idiom: build inside a `using` stack, and on the last line `return stack.move()`. Any throw before the move unwinds everything acquired; success hands a fully-armed bundle to the caller.
4. The stack is itself a `Disposable`, so it composes — a field of type `AsyncDisposableStack` makes a whole object's teardown a single `await this.#stack.disposeAsync()`.

```ts
async function openPipeline(cfg: Config): Promise<AsyncDisposable> {
  await using stack = new AsyncDisposableStack();
  const source = stack.use(await openSource(cfg.in));
  const sink = stack.use(await openSink(cfg.out));
  stack.defer(() => log.info('pipeline closed'));
  wire(source, sink);
  return stack.move(); // caller owns the lot; a throw above here released it all
}
```

**Enforcement:** review; a function acquiring two or more disposables uses a `DisposableStack`/`AsyncDisposableStack` rather than nested `try/finally`, and constructors that acquire commit with `.move()`.

### 13.4 — `AbortController` is the universal lifecycle handle.

**Reasoning, step by step:**
1. One `AbortController` cancels everything downstream of it. Its `signal` threads into `fetch(url, { signal })`, into `addEventListener(type, fn, { signal })`, into any well-behaved async API. A single `controller.abort()` then tears down a fetch, a listener, and a timer in one call — the resource graph has one switch.
2. This is the canonical lifecycle primitive across the codebase (restated from 09's 9.11): pass `signal` down, never let a child create its own controller for work the parent owns. Cancellation flows the same direction as the call.
3. Compose signals rather than juggling several: `AbortSignal.any([userSignal, AbortSignal.timeout(5_000)])` aborts when *either* fires, so a per-call timeout (9.6) and a parent's shutdown signal coexist on one handle.
4. `addEventListener(type, fn, { signal })` is how you avoid the classic listener leak: the listener is removed automatically when the signal aborts — no paired `removeEventListener`, no stale closure pinning the target alive. A single `ac.abort()` then drops the listener and cancels an in-flight `fetch(url, { signal: ac.signal })` together.

**Enforcement:** review; one controller owns a unit of work, `signal` is threaded into every cancellable call, and listeners register with `{ signal }` rather than a manual `removeEventListener`.

### 13.5 — Every `setTimeout` / `setInterval` has an owner and a clear path to cleanup.

**Reasoning, step by step:**
1. A bare `setInterval(fn, 1000)` is a leak with an alarm clock: it keeps firing, keeps its closure alive, and on Node keeps the event loop from exiting, long after the thing it served is gone. A timer with no owner is a defect.
2. Every timer resolves to one of two shapes. Either tie it to a signal so it dies with its lifecycle — listen for `signal.addEventListener('abort', () => clearInterval(id))`, or register `stack.defer(() => clearTimeout(id))` in the owning stack (13.3). Or return a disposer the caller is obliged to run.
3. Prefer a managed wrapper to raw timers in long-lived code: a small `Disposable`-returning `interval(ms, fn, signal)` that clears on dispose keeps the call site honest and the cleanup local to the acquisition.
4. On Node, an unref'd timer (`timer.unref()`) still must be cleared if it holds a closure you want collected — unref affects loop liveness, not the leak. Clearing is the cleanup; unref is a scheduling hint. The managed shape is small: `setInterval`, then `signal.addEventListener('abort', () => clearInterval(id), { once: true })`, returning `{ [Symbol.dispose]: () => clearInterval(id) }`.

**Enforcement:** review; no `setInterval`/`setTimeout` whose handle is discarded — each is cleared via a signal, a `defer`, or a returned disposer.

### 13.6 — Pools, caches, and queues are bounded, with a TTL where applicable.

**Reasoning, step by step:**
1. An unbounded cache is a slow out-of-memory crash; an unbounded queue is the same crash with backpressure hidden. Root rule 9 — limits on everything — applies to every store that grows with input. The bound is a design parameter, named at the call site, not a guess.
2. Caches are LRU (or LFU) with a maximum entry count, sized to the expected working set (ported from root performance.md). Pair the size bound with a TTL: TTL-based expiry is a clock read, while event-based invalidation is a distributed-systems problem you do not want inside a cache. Never cache errors long-term; cache negative results briefly or not at all.
3. Connection and worker pools carry an explicit `max`, chosen from downstream capacity and memory — not "should be enough." Checkout itself is bounded by a timeout (9.6) so a saturated pool fails fast instead of queueing forever. Monitor checkout latency and exhaustion.
4. Queues that decouple producers from consumers have a `maxSize` and a defined policy at the bound: reject, drop-oldest, or apply backpressure. An unbounded in-memory queue moves the OOM from the producer to the queue.

```ts
const cache = new LRUCache<string, Session>({ max: 10_000, ttl: 60_000 }); // size + TTL, both explicit
const pool = createPool({ max: 20, acquireTimeoutMs: 2_000 }); // bound + bounded checkout
```

**Enforcement:** review; every cache, pool, and queue declares an explicit numeric bound, caches that admit staleness set a TTL, and the bound is a named constant.

### 13.7 — Release resources in reverse acquisition order.

**Reasoning, step by step:**
1. Teardown mirrors setup. If you open a transaction, then a cursor on it, you close the cursor first and the transaction second — the later acquisition depended on the earlier, so it must die first. Reverse order is the only order that never releases something still in use.
2. `DisposableStack` (13.3) gives this for free: it disposes registrations last-in-first-out, so as long as you `use`/`defer` in acquisition order, release order is correct by construction and stays correct as the list grows.
3. The hazard appears in hand-written cleanup: a single `finally` that closes A then B, where B was layered on A, releases A out from under B. If you must write the unwind by hand, write it bottom-up and add a comment naming the dependency.
4. Independent resources may release in any order, but defaulting to reverse costs nothing and removes the need to prove independence on every edit.

**Enforcement:** review; composite teardown is reverse-order, achieved through a `DisposableStack` by default; any hand-written unwind is bottom-up with the dependency noted.

### 13.8 — Never rely on the garbage collector for resources.

**Reasoning, step by step:**
1. The GC reclaims *memory*, on its own schedule, for objects it proves unreachable. It says nothing about file descriptors, sockets, or locks — those are OS handles the GC does not count and will not hurry to release. Waiting for it is how a process runs out of descriptors while memory looks fine.
2. `FinalizationRegistry` is not a cleanup strategy. Its callbacks may run late, may coalesce, and may never run at all — at process exit they are explicitly not guaranteed. It is a diagnostic aid (detecting that something *was* collected), not a release mechanism.
3. Close explicitly, every time, through `using` (13.1) or a `DisposableStack` (13.3). The deterministic path is the only path; the GC is a fallback that frees memory only, never your handles. `WeakRef`/`WeakMap` are legitimate for non-pinning caches and back-references, but collection of the referent runs no teardown you control — they are not cleanup either.

**Enforcement:** review; no `FinalizationRegistry` or `__del__`-style reliance for resource release; cleanup is always explicit and deterministic. A `FinalizationRegistry` used purely for leak diagnostics is commented as such.

### 13.9 — Tests assert cleanup.

**Reasoning, step by step:**
1. A leak is invisible until production. The test for resource management is the negative-space assertion (pairs with 11): not just that the resource worked, but that it was *released*. Spy on the disposer and assert it ran exactly once on both the happy path and the throwing path.
2. Drain fake timers. Under `vi.useFakeTimers()`, assert `vi.getTimerCount() === 0` after the unit under test disposes — a non-zero count is an orphaned `setInterval` caught at test time instead of in a memory graph at 3am.
3. Close handles in `afterEach`. Anything a test opens, the same test's teardown disposes, so a leak in the subject cannot mask itself by riding the suite's process exit. `await using` inside the test body gives this for free.
4. Assert the *order* of release for composite teardown (13.7): record disposals into an array and assert it equals the reverse of the acquisition sequence. Order regressions are silent otherwise.

```ts
it('disposes the connection even when the body throws', async () => {
  const dispose = vi.fn();
  const conn = { [Symbol.asyncDispose]: dispose };
  await expect(runWith(conn, () => { throw new Error('boom'); })).rejects.toThrow('boom');
  expect(dispose).toHaveBeenCalledOnce(); // released on the exceptional path
});
```

**Enforcement:** Vitest; cleanup is asserted with a spied disposer, `vi.getTimerCount()` is checked under fake timers, and `afterEach` closes what the test opened.

## Cross-references

- Cancellation, `AbortSignal.timeout()` (9.6), signal threading (9.11), bounded fan-out: [chapter 09](./09-concurrency.md).
- The `invariant` assertion helper for post-dispose guards: [chapter 05](./05-functions.md).
- Negative-space cleanup assertions, fake timers, `afterEach`: [chapter 11](./11-testing.md).
- Classes for lifecycle, making illegal states unrepresentable: [chapter 06](./06-classes-and-data-modeling.md).
- LRU + TTL caching, pool sizing, the slowest-resource-first ordering: [chapter 15](./15-performance.md) and root [performance.md](../performance.md).
- Bounded everything (root rule 9): [root README](../README.md).
