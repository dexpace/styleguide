# 15 — Performance

Design-phase doctrine — the resource hierarchy, batching, caching, pooling, the rule that one eliminated round-trip beats any amount of CPU tuning — lives in [../performance.md](../performance.md) and is canonical for every language. This chapter is the layer below it: the V8 and TypeScript specifics. How the JIT rewards stable object shapes, why `delete` is a tax, where allocations hide in idiomatic pipeline code, and how the `interface` and tree-shaking discipline this guide already mandates doubles as a performance strategy. Read the cross-cutting guide first; reach here when the architecture is sound and the hot path still costs too much.

## What good looks like

```ts
interface PricePoint {readonly symbol: string; readonly bid: number; readonly ask: number}

// before: p99 quote latency 14ms over a 5k-symbol book; --cpu-prof blamed GC + the await chain.
async function quoteAll(symbols: readonly string[]): Promise<Map<string, PricePoint>> {
  const out = new Map<string, PricePoint>();
  for (const symbol of symbols) {
    const raw = await feed.fetch(symbol);   // serial round-trip per symbol — the killer (15.5)
    if (raw.stale) delete (raw as Partial<typeof raw>).ask; // poisons the hidden class (15.3)
    out.set(symbol, {...raw, symbol});      // fresh spread allocation every tick (15.4)
  }
  return out;
}

// after: one shape, no per-tick spread, awaits batched. p99 14ms -> 1.9ms, GC time down 8x.
async function quoteAllFast(symbols: readonly string[]): Promise<Map<string, PricePoint>> {
  const raws = await Promise.all(symbols.map(s => feed.fetch(s))); // batched fan-out (15.5)
  const out = new Map<string, PricePoint>();
  for (let i = 0; i < raws.length; i++) {
    const raw = raws[i];
    // each PricePoint built field-for-field in one order: a monomorphic set call (15.2).
    out.set(symbols[i], {symbol: symbols[i], bid: raw.bid, ask: raw.stale ? 0 : raw.ask});
  }
  return out;
}
```

The fast version keeps `PricePoint` shape-stable so V8 keeps `set` monomorphic (15.2), never `delete`s a field (15.3), drops the per-tick spread so the inner loop stops feeding the collector (15.4), and collapses N serial round-trips into one batched `Promise.all` (15.5) — the change the profile actually rewarded. The measurement note is not decoration: it is the ledger entry (15.10) that earns the deviation from the obvious serial code.

## Rules

### 15.1 — Optimize the slowest resource first; the design-time wins dwarf the rest.

**Reasoning, step by step:**
1. The 1000× wins are architectural and chosen once: the right request shape, the right cache, the right batch boundary. The V8-level work here buys factors, not orders of magnitude, so reaching for a V8 trick before the design is settled optimizes the wrong layer.
2. The resource order is fixed: **network > disk > memory > CPU**. One eliminated round-trip beats every call site you could hand-tune. Confirm with a profile which resource dominates before touching code.
3. [../performance.md](../performance.md) is canonical for the hierarchy, batching, caching, and pooling. This chapter assumes you have done that and the hot path still costs too much.

**Enforcement:** Design review against the four-resource sketch in [../performance.md](../performance.md); a CPU micro-fix proposed before a profile names CPU as the bottleneck is sent back.

### 15.2 — Keep object shapes stable so V8 stays monomorphic.

**Reasoning, step by step:**
1. V8 assigns every object a hidden class (a "shape") from its properties, their order, and their types. A property access or call that only ever sees one shape is *monomorphic* and compiles to a direct offset load. Two shapes make it polymorphic; many make it megamorphic and fall back to a hash lookup. Types count too: a field that holds an integer then a float forces a re-tag.
2. Construct objects with the same properties in the same order every time. Initialize every field in one place — the literal or the constructor — rather than bolting fields on later, which forks the shape into a transition chain.
3. This is why the `interface` discipline of chapter 06 is also a JIT strategy: one declared shape, fields assigned together, compiles to one hidden class. Plain typed objects are the fast path, not just the clean one.

```ts
interface Point {readonly x: number; readonly y: number}
const b = {y: 2, x: 1};   // bad — order differs from {x, y}, so a different hidden class
// good — one construction site, one order, one shape; the consumer's call stays monomorphic:
const make = (x: number, y: number): Point => ({x, y});
```

**Enforcement:** Code review for fields assigned after construction on hot-path types; `interface` + factory (chapter 06) is the default. Confirm monomorphism in a `--cpu-prof` capture (15.6) when it matters; never assert it from reading the source.

### 15.3 — Never `delete` a property and never create a sparse array.

**Reasoning, step by step:**
1. `delete obj.field` does not just clear a value — it mutates the shape, transitions it to a new hidden class, and on repetition can drop the object into dictionary (slow, hashed) mode it never climbs out of. Every downstream access pays for it.
2. To clear a value while holding the shape, assign `undefined`: the field stays present, the hidden class is untouched, the slot is reused. If you genuinely need a smaller object, build a fresh one with exactly the fields it should have.
3. Sparse arrays are the element-level analogue. `arr[1000] = x` on a short array, or `delete arr[3]`, abandons V8's packed-elements representation for a dictionary store; indexing slows for the array's whole life. Keep arrays dense — `push`, or pre-size and fill every slot. The `Map`/`Set` reflex from chapter 07 sidesteps all of this: a keyed collection is built to add and remove entries, where an object is not a hash map and treating it as one with `delete` is what triggers the deopt.

```ts
delete session.token;                          // bad — transitions the hidden class
session = {...session, token: undefined};      // good — clear the value, keep the shape
tokens.delete(sessionId);                      // better — Map<string, Token>, built to mutate
```

**Enforcement:** Lint `delete` on object properties via `no-restricted-syntax` with the selector `UnaryExpression[operator='delete']` (the chapter's pattern elsewhere). Reviewers redirect add/remove-keyed-data patterns to `Map`/`Set` (chapter 07).

### 15.4 — Keep allocations out of hot paths.

**Reasoning, step by step:**
1. Every object, array, and closure created on a hot path is future work for the collector. One allocation is free; the same allocation a million times a second is a GC bill paid in pause time and throughput. The cost is the *rate*, not the instance.
2. Idiomatic pipeline code hides allocations: `arr.map(...).filter(...)` materializes an array per stage, a spread `{...x, ...y}` allocates an object, a closure capturing loop variables allocates per iteration. None are wrong — they are wrong *in the inner loop of a measured hot path*.
3. The fixes, in order: hoist the allocation out of the loop (build the closure or buffer once, reuse it); restructure the pipeline into one `for…of` pass so no intermediate array is built; reuse a pre-sized output buffer; prefer `Map`/`Set` and typed arrays where the shape allows. Do not pre-optimize cold paths, though: V8's generational GC makes short-lived garbage cheap, and clear pipeline code beats a micro-saved allocation everywhere the profile is flat. Apply this where a profile shows GC pressure, not on reflex.

```ts
// bad — three intermediate arrays plus a closure per call, on a hot path:
const totals = orders.map(o => o.lines).flat().filter(l => l.taxable).map(l => l.cents);
// good — one pass, one output array, no intermediates:
const out: number[] = [];
for (const o of orders) for (const l of o.lines) if (l.taxable) out.push(l.cents);
```

**Enforcement:** Found in a `--cpu-prof` flamegraph or allocation profile (15.6), not asserted. Every hot-loop restructuring carries the ledger note (15.10) that justifies trading the cleaner pipeline for the manual pass.

### 15.5 — Batch independent awaits; serial round-trips are self-inflicted latency.

**Reasoning, step by step:**
1. `await` inside a loop, when iterations do not depend on each other, serializes work that could overlap. N independent fetches at 20ms each become 20×N ms for no reason — the canonical self-inflicted latency, and usually the largest number in a slow request's profile.
2. When the awaits are independent, collect the promises and await together: `await Promise.all(items.map(fn))`. The round-trips overlap and the cost collapses toward the slowest single call. This is rule 9.8 restated as performance, because it is both.
3. Independence is the precondition. If each step needs the previous step's result, the serial loop is correct and the latency is real — do not "fix" a genuine dependency chain into a race. And `Promise.all` is unbounded fan-out: firing ten thousand requests at once is its own outage, so past a handful of concurrent calls bound it with the chapter 09 worker-pool helper. Batched, not unleashed.

```ts
// bad — serial: total latency is the sum of every round-trip
for (const id of ids) users.push(await db.getUser(id));
// good — batched: total latency approaches the slowest single round-trip
const users = await Promise.all(ids.map(id => db.getUser(id)));
```

**Enforcement:** `no-await-in-loop` flags the serial pattern; reviewers confirm the iterations are genuinely independent before batching. Bounded fan-out via the chapter 09 worker-pool helper for large `id` sets.

### 15.6 — Measure first and keep the benchmark; a perf fix without one is a rumor.

**Reasoning, step by step:**
1. Intuition about V8 is wrong roughly half the time — JIT, inlining, escape analysis, and GC interact non-locally, and "this should be faster" is a hypothesis, not a result. Capture a profile that shows the problem before, and another after. The delta is the only proof.
2. For real workloads, run `node --cpu-prof` (or `--heap-prof` for allocations) and open the result in speedscope or Chrome DevTools. Profile production-like data; a flamegraph over toy input proves nothing about the hot path that exists.
3. For micro-comparisons, use `vitest bench`, and state its caveats every time: it measures a function in isolation with a warm JIT, so it flatters code the optimizing compiler treats kindly and ignores the deopts a real call site triggers. Warm-up and variance make small deltas noise — trust order-of-magnitude gaps, distrust 10% ones. Keep the benchmark in the repo beside the code it guards: a committed `*.bench.ts` turns "we made this faster once" into a regression test the next refactor must beat.

```ts
import {bench} from 'vitest'; // committed beside the code as a regression guard
bench('parseFrame manual single-pass (warm-JIT, not end-to-end)', () => parseFrameFast(SAMPLE));
bench('parseFrame pipeline baseline (warm-JIT, not end-to-end)', () => parseFramePipeline(SAMPLE));
```

**Enforcement:** A PR claiming a performance win attaches before/after profile numbers in the description; a deliberate optimization without a committed benchmark or a captured profile does not merge.

### 15.7 — Build strings with `+=`; kill the join-always myth, and measure when it matters.

**Reasoning, step by step:**
1. V8 represents string concatenation with *ropes* (cons-strings): `a += b` does not copy both operands into a new buffer, it allocates a small node pointing at the two pieces and flattens lazily on demand. Building up a string with `+=` in a loop is not the O(n²) catastrophe it is in languages without this optimization.
2. The myth worth killing is "always use `Array.join` for performance." Pushing every fragment into an array and joining at the end allocates the array and its elements first; for ordinary string building it is often the same speed or slower than `+=`, and it is less readable. Reach for `join` when you genuinely have an array of parts, not as a concatenation reflex.
3. Template literals compile to ordinary concatenation and are the default for readability: `${a}${b}` and `a + b` are the same shape to V8, so choose by clarity. And "often" is not "always": if string assembly shows up in a profile, *measure* both approaches on real data with `vitest bench` (15.6) and let the number decide. This kills a cargo-cult default; it does not install the opposite one.

```ts
let csv = '';
for (const row of rows) csv += `${row.id},${row.name}\n`; // fine — V8 ropes, and it reads cleanly
const header = columns.join(','); // reach for join only when you already hold an array of parts
```

**Enforcement:** Reviewers do not demand `join` over `+=` on style grounds. A claim that either is faster on a hot path must cite a `vitest bench` result (15.6).

### 15.8 — Treat JSON parse and stringify as the hot-path cost they are.

**Reasoning, step by step:**
1. `JSON.parse` and `JSON.stringify` are O(payload size) and run on the main thread. On a request path that handles large or frequent payloads, serialization is often a bigger line in the profile than the business logic it wraps — it is real CPU, not a free boundary.
2. The first move is to do less of it: do not parse a body you will not read, do not stringify fields a consumer ignores, do not round-trip a value through JSON to clone it when `structuredClone` exists. The cheapest serialization is the one you skipped.
3. At scale, a generic `JSON.stringify` walks the object reflectively every call. A schema-derived serializer specializes the work to a known shape and can be markedly faster on a hot endpoint. That belongs to the server runtime — see the forward reference in [typescript-node](../typescript-node/) chapter 07 — and like any deliberate optimization it carries a benchmark (15.6) and a ledger note (15.10), justified only by a profile that names serialization.

```ts
const copy = JSON.parse(JSON.stringify(config)); // bad — round-trip through JSON to deep-clone
const copy = structuredClone(config);            // good — purpose-built, lossless, shape-preserving
```

**Enforcement:** Reviewers flag JSON used as a clone mechanism and bodies parsed but unread. A schema-derived serializer lands with the profile (15.6) that justifies it; default to `JSON` until then.

### 15.9 — Author every module for tree-shaking.

**Reasoning, step by step:**
1. Code that ships but never runs is negative performance: it inflates the bundle, lengthens parse and compile time, and on the client delays interactivity. The fastest code is the code the bundler proved it could drop — and that proof depends on how you author the module.
2. Three habits, all already mandated elsewhere in this guide, are what make a module shakeable. Export named bindings, never a default object that staples everything together (chapter 10). Set `"sideEffects": false` in `package.json` (or list the exact files that do have side effects) so the bundler may discard an unused import instead of keeping it for fear of an effect. And do no work at import time — rule 12.6 — because a top-level call is a side effect the bundler must preserve.
3. The mechanism: tree-shaking is static. The bundler can only remove what it can *prove* is unused, and a default export, an unflagged side effect, or import-time work each defeat that proof and pin the code in. This costs nothing extra — it is the chapter 10/12 discipline restated as performance, because shippable dead code is exactly that.

```ts
export default {parse, format, validate}; // bad — default bag the bundler must keep whole
console.log('utils loaded');              // bad — top-level side effect pins the module
export function parse(s: string): Frame {/* ... */} // good — named, no import-time work; shakes out
```

**Enforcement:** Named-exports-only and `"sideEffects": false` are checked in review (chapters 10 and 12); `no-restricted-syntax` against top-level calls backs rule 12.6. Bundle-analyzer output catches modules that refuse to shake.

### 15.10 — Every deliberate optimization carries the measurement that justifies it.

**Reasoning, step by step:**
1. A performance trick is a deviation from the obvious, clean version — a manual loop where a pipeline read better, a specialized serializer where `JSON` sufficed, a reused buffer where a fresh allocation was clearer. The deviation is only earned by a number, and the number has to travel with the code.
2. Write the justification as a comment at the site: what was slow, what the fix bought, and how it was measured. "p99 14ms → 1.9ms, --cpu-prof" is an entry in the optimization ledger; "// fast path" is folklore. The next reader needs the evidence, not the adjective.
3. This is the zero-technical-debt rule (root rule 12) applied to performance. Clever code without a recorded reason is debt: the next person cannot tell a load-bearing optimization from a superstition, so they preserve cruft out of fear or delete a real win by accident. The comment resolves it — and acts as a filter: if you cannot produce the measurement that justifies the cleverness, that is the signal it should be simplified away. Unexplained optimization does not survive review.

```ts
// monomorphic build, deliberate: pipeline showed 11% GC time in --cpu-prof; cut p99 14ms->1.9ms.
const point: PricePoint = {symbol, bid: raw.bid, ask: raw.stale ? 0 : raw.ask}; // (15.2, 15.4)
```

**Enforcement:** Reviewers require a measurement comment on any non-obvious perf construct and simplify away cleverness that cannot cite one. The committed benchmark or profile (15.6) is the ledger's backing evidence.

## Cross-references

- The `interface` discipline that compiles to one hidden class, branded primitives, and tested guards: [03-the-type-system.md](./03-the-type-system.md). Stable shapes for derived literals and the banned non-null `!`: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Small functions and bounded loops the JIT inlines: [05-functions.md](./05-functions.md). Plain typed objects + free functions as the monomorphic default, discriminated unions: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md). `Map`/`Set` over object-as-hashmap, pipeline allocation costs, `structuredClone`: [07-typescript-idioms.md](./07-typescript-idioms.md).
- Batched fan-out (9.8) and the bounded-concurrency worker pool behind rule 15.5: [09-concurrency.md](./09-concurrency.md). Named exports as a tree-shaking precondition: [10-api-design.md](./10-api-design.md). The testing discipline a committed `*.bench.ts` (15.6) inherits — determinism and colocation beside the code: [11-testing.md](./11-testing.md).
- `"sideEffects": false` and no import-time work (12.6): [12-module-organization.md](./12-module-organization.md). Bounded pools, caches, and buffer reuse on hot paths: [13-resource-management.md](./13-resource-management.md). Schema-derived serializers and runtime profiling: [typescript-node](../typescript-node/) chapter 07. Design-phase doctrine (the resource hierarchy, batching, caching, pooling), canonical for all of the above: [../performance.md](../performance.md).
