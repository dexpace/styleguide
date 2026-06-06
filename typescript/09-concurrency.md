# 09 — Concurrency

JavaScript runs on a single thread. What we call concurrency here is interleaved I/O: many `await`s in flight, the runtime resuming each as its data arrives, never two lines of your code running at once. This chapter keeps that interleaving correct and bounded — every promise accounted for, every external wait deadlined, every fan-out ceilinged, every cancellation propagated to the leaves. Cross-thread parallelism is a separate concern and lives in the runtime guides ([typescript-node](../typescript-node/) workers).

## What good looks like

```ts
/** Fetch many users in parallel, bounded, cancellable, deadline-capped, partial-failure-aware. */
export async function loadUsers(
  ids: readonly string[],
  { signal }: { signal?: AbortSignal } = {},
): Promise<Map<string, User>> {
  invariant(ids.length <= 1000, `too many ids: ${ids.length}`); // 9.12 declared bound

  const results = await mapWithConcurrency(ids, 8, async (id) => {
    const deadline = AbortSignal.timeout(2000); // 9.6 no unbounded wait
    const merged = signal ? AbortSignal.any([signal, deadline]) : deadline; // 9.6 caller + deadline
    const res = await fetch(`/api/users/${id}`, { signal: merged }); // 9.11 propagated down
    if (!res.ok) throw new Error(`user ${id}: HTTP ${res.status}`);
    const user = parseUserOrThrow(await res.json()); // boundary parse (3.4, 10.7)
    return [id, user] as const;
  });

  const failures = results.flatMap((r) => (r.status === 'rejected' ? [toError(r.reason)] : [])); // 9.9 every rejection inspected
  if (failures.length > 0) {
    throw new AggregateError(failures, `${failures.length}/${ids.length} user loads failed`);
  }
  return new Map(results.flatMap((r) => (r.status === 'fulfilled' ? [r.value] : [])));
}
```

This bounds fan-out to eight in-flight at a time through `mapWithConcurrency` (9.7) rather than firing `ids.length` requests at once; it caps each external call with `AbortSignal.timeout` and folds in the caller's signal with `AbortSignal.any` (9.6); it threads `signal` down to `fetch` (9.11); it tolerates partial failure with `Promise.allSettled` semantics, inspecting every rejection and aggregating them into an `AggregateError` (9.9, cross-ref [08](./08-error-handling.md) §8.11); and it asserts the input bound up front (9.12). Nothing floats: the helper's promise is awaited, and so is every promise inside it (9.3).

## Rules

### 9.1 — Know the event-loop model before writing async code.

**Reasoning, step by step:**
1. JavaScript executes on one thread with one call stack; your code is never preempted mid-statement.
2. Every `await` is a suspension point: the function yields, the runtime advances other pending work, and execution resumes only when the awaited promise settles.
3. So "concurrency" here means *interleaved I/O* (overlapping waits, not simultaneous computation), and a CPU-bound loop with no `await` blocks the whole thread until it finishes.
4. Two consequences drive the chapter: a long synchronous span starves every other task (push CPU work off-thread), and state can change across an `await` even though no two lines run at once (9.10). True parallelism needs workers or processes — a runtime concern, not a language one.

**Enforcement:** review; CPU-bound spans between awaits are flagged and moved off the event loop.

### 9.2 — Use `async`/`await`; reserve `.then` for interop edges.

**Reasoning, step by step:**
1. `async`/`await` reads top to bottom, so the sequencing of an operation is visible in source order. A `.then`/`.catch` chain scatters that sequencing across callbacks and makes the error path easy to lose.
2. Mixing the two styles in one function hides the order of operations: a reader must hold both a linear and a callback mental model at once. Prefer `enrich(await fetchUser(id))` over `fetchUser(id).then(enrich)`.
3. The legitimate `.then` is at an edge you do not own — a framework callback that wants a promise, a fire-point adapting a non-async API. Convert to `await` as soon as you are back in your own function body.

**Enforcement:** review; `.then`/`.catch` only at interop boundaries, `await` everywhere else.

### 9.3 — Float no promises.

**Reasoning, step by step:**
1. A promise you neither `await` nor `return` is an error path that vanishes: if it rejects, the failure surfaces as an unhandled rejection far from the cause, or not at all, and its ordering relative to the rest of the function is undefined.
2. Every promise has exactly one of three honest fates: `await` it, `return` it to the caller who will, or (for a genuine fire-and-forget) explicitly discard it with `void` *and* a comment saying why losing its result and its rejection is acceptable.
3. The same trap hides in misused positions: a promise where a `boolean` is expected (an `if (asyncCheck())` is always truthy), or an `async` callback handed to something that ignores its return — both caught by the companion lint rule.

```ts
await audit.record(event);          // good — awaited
return cache.set(key, value);       // good — returned to the caller
void fireMetric(name);              // good — deliberately dropped; metric loss is tolerable, no rejection path needed
fireMetric(name);                   // bad — floats; a rejection becomes an unhandled rejection
```

**Enforcement:** `@typescript-eslint/no-floating-promises` (the guide's flagship async lint rule) and `@typescript-eslint/no-misused-promises`.

### 9.4 — Never wrap async work in the `Promise` constructor.

**Reasoning, step by step:**
1. `new Promise(async (resolve, reject) => { … })` is a trap: a `throw` inside that `async` executor rejects the inner promise that nothing awaits, not the outer one, so the error is swallowed and the outer promise hangs forever.
2. Async work is already a promise. Wrapping `await`-able code in a constructor adds a layer that can only lose errors and obscure control flow; write an `async` function and `return` the value.
3. The one sanctioned `new Promise` adapts a callback-style API that predates promises (an event emitter, a Node-style `(err, value)` callback) with a *synchronous* executor that resolves or rejects from inside the callback.

```ts
// bad — async executor; a throw here rejects a promise no one holds
new Promise(async (resolve) => resolve(await fetchUser(id))); // no-async-promise-executor
// good — the only reason to reach for new Promise: bridge a callback API with a sync executor
const loaded = new Promise<void>((resolve, reject) => emitter.once('done', resolve).once('error', reject));
```

**Enforcement:** `@typescript-eslint/no-async-promise-executor`; `new Promise` only to adapt callback APIs.

### 9.5 — Make cancellation part of the signature.

**Reasoning, step by step:**
1. Every long-running async API takes an options object with `{ signal }: { signal?: AbortSignal }`. Cancellation is a first-class part of the contract, declared in the type, not an afterthought bolted on when a timeout finally bites.
2. A function that cannot be cancelled forces its callers to leak: a navigated-away request, an abandoned batch, a shut-down service all keep running with nowhere to deliver. The caller owns the lifetime and must be able to end it — the same cooperative-cancellation discipline the [Kotlin guide](../kotlin/09-concurrency.md) builds on structured scopes.
3. Accepting the signal is half the contract; honoring it is the other half (9.11) — the signal flows down to the I/O primitive (`fetch`, a timer, a stream) and is checked between CPU-bound steps.

**Enforcement:** review; every long-running async export accepts `{ signal?: AbortSignal }` and passes it down.

### 9.6 — Put a timeout on every external I/O call.

**Reasoning, step by step:**
1. An external call with no deadline is an unbounded wait, and an unbounded wait is an unbounded queue: a slow dependency becomes exhausted connections, growing memory, a stalled service. This ports the Python guide's `asyncio.timeout` discipline ([root rule 9](../README.md)).
2. `AbortSignal.timeout(ms)` produces a signal that aborts after `ms`; pass it to the call. Pick a number matching the user-perceived SLA, not "infinity minus a bit" — an aggressive deadline surfaces a real problem, a lenient one hides it.
3. The call usually has *two* reasons to abort: combine the caller's signal and the deadline with `AbortSignal.any([signal, AbortSignal.timeout(ms)])` so whichever fires first wins, and the abort `reason` tells you which (the headline exemplar shows the shape).

**Enforcement:** review; every external I/O call carries an `AbortSignal.timeout`, combined with the caller's signal where present.

### 9.7 — Bound fan-out with a worker pool.

**Reasoning, step by step:**
1. `Promise.all(items.map(fn))` over unbounded `items` is a resource bomb: it launches *N* operations at once, and for large or attacker-influenced *N* that means *N* open sockets, *N* in-flight requests, a thundering load on the dependency.
2. The fix is a fixed concurrency limit: at most `limit` operations in flight, the next starting only as one finishes. Define the worker-pool helper once, route every fan-out through it, and tune `limit` to the dependency's headroom, not the input size.
3. It returns `PromiseSettledResult`s, not a bare array — a single failure does not abandon the rest, and the caller decides what partial failure means (9.9).

Define it in full once, project-wide:

```ts
export async function mapWithConcurrency<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T, index: number) => Promise<R>,
): Promise<PromiseSettledResult<R>[]> {
  invariant(Number.isInteger(limit) && limit > 0, `limit must be a positive integer, got ${limit}`);
  const results = new Array<PromiseSettledResult<R>>(items.length);
  let next = 0;
  const worker = async (): Promise<void> => {
    while (next < items.length) {
      const i = next++; // single-threaded: this read-then-increment cannot interleave
      // items[i] is in-bounds: i < items.length is guarded by the while condition (3.4)
      try { results[i] = { status: 'fulfilled', value: await fn(items[i] as T, i) }; }
      catch (e: unknown) { results[i] = { status: 'rejected', reason: e }; }
    }
  };
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```

**Enforcement:** review; naked `Promise.all(items.map(...))` over unbounded input is rejected in favour of `mapWithConcurrency`.

### 9.8 — Don't `await` in a loop for independent work.

**Reasoning, step by step:**
1. `for (const x of xs) { await f(x); }` runs the calls strictly one after another. When the iterations do not depend on each other, that serialization is pure latency: ten 100 ms calls take a second instead of the 100 ms they could.
2. For independent work, start the operations together and await them as a batch — through `mapWithConcurrency` (9.7) when *N* is large or unbounded, or `Promise.all`/`Promise.allSettled` when *N* is small and fixed.
3. The carve-out is *legitimate serial* work: each step depends on the previous result, or the calls must not overlap (ordered writes, rate-limited politeness, a paginated cursor). Then the loop is correct, and a comment should say which reason applies so a reviewer does not "optimize" it into a race.

```ts
// bad — independent fetches, serialized; latency is the sum
for (const id of ids) users.push(await fetchUser(id));
// good — independent, batched and bounded
const settled = await mapWithConcurrency(ids, 8, (id) => fetchUser(id));
```

**Enforcement:** review (`no-await-in-loop` as an advisory signal, with the legitimate-serial carve-out documented at the call site).

### 9.9 — Reach for `Promise.allSettled` when partial failure is acceptable.

**Reasoning, step by step:**
1. `Promise.all` rejects on the *first* failure and abandons the rest, right only when any single failure makes the whole operation meaningless. When the units are independent and a partial result is useful, it throws away both the successes and the other failures.
2. `Promise.allSettled` waits for all and returns a `fulfilled`/`rejected` result per item. Every rejection is then *inspected*, never discarded to keep only the `fulfilled` ones (that silently swallows errors); collect the rejections, then if any exist surface them in a single `AggregateError` (cross-ref [08](./08-error-handling.md), §8.11) stating how many of how many failed.

```ts
const settled = await Promise.allSettled(orders.map((o) => persist(o)));
const failures = settled.flatMap((r) => (r.status === 'rejected' ? [toError(r.reason)] : []));
if (failures.length > 0) {
  throw new AggregateError(failures, `${failures.length}/${orders.length} orders failed`);
}
```

**Enforcement:** review; `Promise.allSettled` for tolerable partial failure, every rejection inspected and aggregated, never silently dropped.

### 9.10 — Treat interleaving as a real race.

**Reasoning, step by step:**
1. A single thread does not save you from races. Any `await` is a yield point: between the check and the act, another task can run and change the state you just checked. Check-then-act across an `await` is a time-of-check-to-time-of-use bug with no second thread in sight.
2. The classic shape is `if (!cache.has(k)) { cache.set(k, await load(k)); }` — two callers both pass the `has`, both `load`, and the second `set` clobbers the first or wastes the work. The state read before the `await` is stale after it.
3. Re-validate the invariant *after* every `await` that could have let the world move, or restructure to single ownership so only one task can mutate the state — an in-flight map keyed by the request, a queue, a lock-like promise the second caller awaits instead of racing.

```ts
// bad — has() checked before the await is stale after it; two callers both load
if (!cache.has(key)) cache.set(key, await load(key));

// good — single ownership: the in-flight promise is the lock; the second caller awaits it
let pending = inFlight.get(key);
if (pending === undefined) { pending = load(key); inFlight.set(key, pending); }
return pending;
```

**Enforcement:** review; check-then-act across an `await` is re-validated after the await or replaced with single ownership.

### 9.11 — Propagate signals downward.

**Reasoning, step by step:**
1. A signal accepted at the top (9.5) is only useful if it reaches the actual wait. Pass `signal` through every layer (handler to service to client to the `fetch` or timer) so a cancellation at the top tears down the work at the bottom. A signal that stops at the first function is decoration.
2. Between `await`s, a CPU-bound stretch never observes the abort on its own; insert `signal.throwIfAborted()` at the top of each iteration or before each expensive step so a cancellation is honored promptly rather than after the loop finishes. When combining the caller's signal with a local deadline (9.6), pass the *combined* signal downward so lower layers honor whichever fires first without knowing which.

```ts
for (const doc of docs) {
  signal?.throwIfAborted();                   // CPU step between awaits sees the cancel
  await index.put(doc.id, embed(doc), { signal }); // signal continues down to the I/O
}
```

**Enforcement:** review; `signal` threaded through every layer to the I/O primitive, `throwIfAborted()` in CPU loops between awaits.

### 9.12 — Bound every queue, buffer, and in-flight map.

**Reasoning, step by step:**
1. An unbounded queue is a memory leak with a delay: producers outrun consumers, the backlog grows without limit, and the process dies under load instead of degrading. The same applies to a retry buffer, a batch accumulator, an in-flight request map.
2. Declare the bound explicitly as a named constant near the structure, chosen from the slowest consumer's catch-up time or the maximum acceptable memory; this is the [root rule 9](../README.md) limits-on-everything discipline applied to async state. Assert it with `invariant` ([chapter 05](./05-functions.md)) where things are added, so a breach crashes at the cause instead of silently growing into an out-of-memory hours later.
3. At the bound, the honest responses are backpressure (suspend the producer), shed load (reject), or drop with a counter — never grow.

```ts
const MAX_INFLIGHT = 256;
const inFlight = new Map<string, Promise<Response>>();

function enqueue(key: string, task: () => Promise<Response>): Promise<Response> {
  invariant(inFlight.size < MAX_INFLIGHT, `in-flight map full at ${MAX_INFLIGHT}`); // bound asserted at insert
  const p = task().finally(() => inFlight.delete(key));
  inFlight.set(key, p);
  return p;
}
```

**Enforcement:** review; every queue, buffer, and in-flight map declares a named bound and asserts it with `invariant`.

## Cross-references

- `invariant` and assertion density: [chapter 05](./05-functions.md).
- `assertNever` and exhaustive narrowing: [chapter 06](./06-classes-and-data-modeling.md).
- `?.`/`??` for optional signals and absence: [chapter 07](./07-typescript-idioms.md).
- `toError`, `AggregateError` fan-out (§8.11), programmer vs operational errors: [chapter 08](./08-error-handling.md).
- `AbortController`/`AbortSignal` as a lifecycle handle, bounded pools and queues, `using`/`await using`: [chapter 13](./13-resource-management.md).
- Eliminating serial `await`s, V8 and the event loop: [chapter 15](./15-performance.md).
- Worker threads and cross-thread parallelism: [typescript-node](../typescript-node/).
