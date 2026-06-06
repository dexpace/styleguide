# 08 — React Performance

React's performance model is the reconciler's: cost is *re-renders × the work each one does*, paid on the main thread between a user's action and the pixels that answer it. The language-level doctrine — the resource hierarchy, the measure-first discipline, the optimization ledger that earns every deviation — lives in [../typescript/15-performance.md](../typescript/15-performance.md) and is canonical here; this chapter is the layer above it, where the unit of work is a component subtree and the bottleneck is usually a render storm, not a hot loop. As of 2026 the React Compiler (stable v1) memoizes for you, so most of the old manual-memo lore is obsolete; what remains is structural. Read the core chapter first; reach here when the architecture is sound and a Profiler trace still shows the UI doing too much.

## What good looks like

```tsx
// before: React DevTools Profiler — typing one char in the filter box re-rendered the whole
// 800-row <Dashboard> subtree. Commit: 38ms, every <Row> flashed in "why did this render".
function Dashboard({rows}: {rows: readonly Row[]}) {
  const [query, setQuery] = useState('');
  const shown = rows.filter(r => r.name.includes(query)); // recomputed fine; not the problem
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ExpensiveChart rows={rows} />   {/* re-renders on every keystroke — owns none of `query` */}
      <Table rows={shown} />
    </>
  );
}

// after: state colocated in the only subtree that reads it (8.2); the keystroke can no longer
// reach <ExpensiveChart>, which now renders only when `rows` changes. Profiler commit: 38ms -> 4ms.
function Dashboard({rows}: {rows: readonly Row[]}) {
  return (
    <>
      <ExpensiveChart rows={rows} />   {/* sibling of the state, not a child — untouched by typing */}
      <FilterableTable rows={rows} />
    </>
  );
}
function FilterableTable({rows}: {rows: readonly Row[]}) {
  const [query, setQuery] = useState('');               // state pushed down to its consumer (8.2)
  const shown = rows.filter(r => r.name.includes(query));
  return <><input value={query} onChange={e => setQuery(e.target.value)} /><Table rows={shown} /></>;
}
```

The fix added no `memo` and no `useMemo`. It moved `query` to the only subtree that reads it (8.2), so a keystroke's re-render can no longer reach `ExpensiveChart` — structure did what a wall of memoization would have approximated, and the Compiler (8.1) handles the rest. The `38ms → 4ms` comment is not decoration: it is the ledger entry (core 15.10) and the Profiler trace (8.7) that together earn the refactor over the obvious single-component version.

## Rules

### 8.1 — Let the React Compiler memoize; hand-write `memo`/`useMemo`/`useCallback` only with a trace attached.

**Reasoning, step by step:**
1. The React Compiler (stable v1 as of 2026) memoizes components and values automatically, at a granularity finer than a human will maintain by hand. With it enabled, manual `memo`, `useMemo`, and `useCallback` are the exception, not the baseline — most are now redundant noise that the compiler already covers.
2. Hand-memoization has a real cost the compiler does not: every `useMemo` is a dependency array that can drift out of sync (the same stale-closure failure mode as any effect), and a wrong one caches a stale value silently. Adding memoization on a guess can make a component slower — the comparison runs every render and may never pay back.
3. So the rule inverts the old default: write the plain version, let the compiler optimize it, and reach for a manual hook only when a Profiler trace (8.7) shows a specific component the compiler did not cover. When you do, the measurement travels with it as a ledger entry (core 15.10) — the same evidence rule the language guide demands for any deviation, restated for hooks.

```tsx
const sorted = useMemo(() => rows.toSorted(cmp), [rows]); // bad on a guess — compiler likely covers this
const sorted = rows.toSorted(cmp);                        // good — plain; let the compiler memoize it
```

**Enforcement:** `eslint-plugin-react-hooks` v6 (the compiler-powered ruleset, README REACT-2) runs in CI and the compiler is on. A `useMemo`/`useCallback`/`memo` added in review without a profile comment is removed; this is rule 2.9 (stable identities only where the trace proves they matter) read as performance.

### 8.2 — Fix re-render storms by structure, not by memo.

**Reasoning, step by step:**
1. A re-render cascades down the subtree under the state that changed. The lever is therefore *where state lives*, not how aggressively you cache: move the work out of the cascade and the cascade can no longer reach it. Memoization only papers over a structure that puts unrelated components beneath the same state.
2. Three structural moves, in order. Colocate state with the component that reads it (chapter 03's 3.1) rather than parking it high. Then two moves 3.1 does not cover: push state *down* into a small child so a change re-renders that child alone, not its siblings; and lift slow content *up*, passing it as `children` — a child passed in as a prop is created by the parent's parent, so it keeps its identity when the wrapper's own state changes and does not re-render with it.
3. Internalize the `children` move above all, because it is the one that hides: nothing at the `<Wrapper><Slow /></Wrapper>` call site signals the optimization, yet `Slow` survives every re-render of the state-churning `Wrapper`. This is composition (README REACT-6) paying a performance dividend — the same slot pattern, now load-bearing for render cost.

```tsx
// good — `children` is created by the parent; <Slow> keeps identity across <Toggle>'s state churn:
function Toggle({children}: {children: ReactNode}) {
  const [open, setOpen] = useState(false);
  return <><button onClick={() => setOpen(o => !o)}>toggle</button>{open && children}</>;
}
// <Toggle><Slow /></Toggle> — toggling never re-renders <Slow>; no memo needed.
```

**Enforcement:** A Profiler trace (8.7) showing a re-render storm is triaged structurally first — colocate, push down, lift to `children` — before any `memo` is considered. Reviewers reject "wrap it in `memo`" as the opening move when the real fix is moving the state.

### 8.3 — Split at route boundaries with `lazy()` + Suspense; go finer only on bundle evidence.

**Reasoning, step by step:**
1. A route is a natural code-splitting seam: the user navigating to `/settings` is exactly the moment its code is needed and not before. `React.lazy()` plus a `<Suspense>` fallback makes the bundle for a route arrive when the route does, so the initial load ships only the entry route — the largest, cheapest win available to a client bundle.
2. The boundary is the route component, paired with the router (chapter 05's 5.4 owns the route-tree wiring): each lazily-imported page is one async chunk with one fallback, which keeps the loading states coarse and predictable instead of scattering spinners through the tree.
3. Finer-grained splitting below the route — lazy-loading a heavy modal, an editor, a charting library — is a real tool but a conditional one: it costs a request and a fallback at an interaction boundary, so it earns its place only when the bundle analyzer (8.5) shows a specific dependency heavy enough to defer. Split on evidence, not on the reflex to lazy-load everything.

```tsx
const Settings = lazy(() => import('./routes/Settings')); // chunked; arrives with the route (5.4)
<Suspense fallback={<RouteSkeleton />}><Settings /></Suspense>
```

**Enforcement:** Route definitions use `lazy()` + `Suspense` by default (chapter 05). A below-route `lazy()` boundary lands with the bundle-analyzer (8.5) line that justifies the extra fallback; without it, default to a static import.

### 8.4 — Virtualize long lists; hundreds of rows is a windowing problem.

**Reasoning, step by step:**
1. Rendering a list mounts one component subtree and one set of DOM nodes per row. At hundreds or thousands of rows the cost is not any single row — it is the count, paid on mount, on every re-render, and in the memory the browser holds for nodes nobody can see. No amount of per-row `memo` fixes a problem whose size *is* the row count.
2. The fix is windowing: render only the rows in (and just around) the viewport, recycling nodes as the user scrolls, so render cost tracks the window's fixed size instead of the dataset's. A 10,000-row table then renders the ~20 rows on screen.
3. Use TanStack Virtual — it is headless, so it computes the window and leaves the markup and accessibility to you (the accessible-table semantics of README REACT-4 still apply to the rendered window). This is the language guide's "the cost is the *rate*, not the instance" (core 15.4) at component scale: the rendered-node *count* is the rate, and windowing bounds it.

```tsx
const v = useVirtualizer({count: rows.length, getScrollElement: () => ref.current, estimateSize: () => 40});
// render only v.getVirtualItems() — ~20 nodes for 10k rows, not 10k
```

**Enforcement:** A list known to reach into the hundreds is virtualized by default; reviewers flag an unbounded `.map()` over a large or paginated dataset. Confirm the window size against a Profiler commit (8.7) rather than asserting it.

### 8.5 — Budget the bundle in CI; a dependency that doubles it is a regression.

**Reasoning, step by step:**
1. Bundle size is interactivity: every kilobyte the client ships is parse, compile, and execution time between navigation and a usable page. Unmeasured, it only grows — each PR adds a little, no single one looks guilty, and the page is slow a quarter later. The control is a number that fails CI.
2. Run a bundle analyzer on every PR and diff against the base branch, so the review sees the delta this change introduces: a new dependency that doubles a route's chunk is a performance regression in exactly the way a doubled p99 latency is, and is reviewed as one — sometimes accepted, never invisible.
3. The biggest wins are the language guide's tree-shaking discipline (core 15.9) made visible: named exports, `"sideEffects": false`, and no import-time work let the bundler drop what a route does not use. The analyzer diff is what proves a module actually shook out instead of riding along — the synergy is real only when CI measures it.

```jsonc
// CI: build, then fail the PR if a route chunk grows past its budget.
{ "budgets": [{ "type": "bundle", "name": "main", "maximumError": "180kb" }] }
```

**Enforcement:** Bundle-analyzer output is posted on every PR and a per-route size budget fails CI when exceeded. A dependency that moves the diff materially is justified in the PR description or replaced with a lighter one.

### 8.6 — Treat images and fonts as performance, not assets.

**Reasoning, step by step:**
1. On a real page the largest bytes and the worst layout shifts are usually media, not JavaScript — an unsized hero image is both the LCP element and a CLS spike when it loads and shoves the text down. Media is a first-class part of the performance budget, not a content detail handed off after the code is fast.
2. Four habits cover most of it. Set explicit `width`/`height` (or `aspect-ratio`) so the browser reserves the box and nothing shifts when the image arrives (this is the CLS in 8.7). Ship modern formats (AVIF/WebP) for the smaller transfer. Add `loading="lazy"` to below-the-fold images so they do not compete with the initial render. And serve fonts with `font-display: swap` so text paints immediately in a fallback instead of blocking on the web font.
3. These map straight to the Web Vitals of 8.7: dimensions and `swap` protect CLS, modern formats and lazy-loading protect LCP. Media discipline is not separate from the metrics — it is one of the largest levers on them, which is why it is a rule and not a footnote.

```tsx
<img src="/hero.avif" width={1200} height={600} loading="lazy" alt="…" /> {/* sized: no CLS; lazy: no LCP contention */}
```

**Enforcement:** Reviewers flag an `<img>` without dimensions (or `aspect-ratio`) and raw formats where AVIF/WebP is available; `font-display: swap` is the default in the font stack. Regressions show up in the CLS and LCP numbers tracked under 8.7.

### 8.7 — Measure with the React DevTools Profiler and Web Vitals; optimize against traces, not vibes.

**Reasoning, step by step:**
1. Intuition about React render cost is wrong about as often as intuition about V8 (core 15.6): a component re-renders for a reason that is not visible in its own source, and "this feels slow" is a hypothesis, not a result. The React DevTools Profiler is the trace — it shows which components rendered, how long each commit took, and *why* each one rendered. Capture it before and after; the delta is the proof, the same discipline as the language guide's `--cpu-prof`.
2. The Profiler measures your render cost; Web Vitals measure the user's truth. LCP (when the main content paints), INP (how fast the UI answers an interaction), and CLS (how much the layout jumps) are what the person at the edge (README REACT-4) actually experiences, and they are the metrics that decide whether a change helped. Lab traces find the cause; field Vitals confirm it mattered.
3. So every rule in this chapter resolves to a number from one of these two instruments. A re-render fix (8.2) is a Profiler commit-time delta; an image fix (8.6) is a CLS or LCP delta; a split (8.3) or a bundle cut (8.5) is an LCP delta. Optimize against the trace, record the before/after as a ledger entry (core 15.10), and never against a feeling that the app got faster.

```tsx
// LCP 3.1s -> 1.4s after route-splitting the entry chunk (8.3); INP 210ms -> 60ms after 8.2.
// Profiler commit on the dashboard: 38ms -> 4ms. Numbers from DevTools + web-vitals, not vibes.
```

**Enforcement:** A PR claiming a React performance win attaches a before/after Profiler trace or a Web Vitals delta in the description; a fix without one does not merge. Web Vitals are tracked in the field, and a regression in LCP/INP/CLS is treated as a bug — correctness as the whole contract (README REACT-4, Influences / Tiger Style).

## Cross-references

- The Compiler-powered hooks lint and stable identities only where a trace proves them (2.9), behind rules 8.1 and 8.2: [02-hooks.md](./02-hooks.md). State colocation and lifting up (3.1): [03-state-management.md](./03-state-management.md); pushing state down and `children` as the structural fix for re-render storms are 8.2's own.
- Route-level code splitting and the router wiring (5.4) behind rule 8.3: [05-structure-and-routing.md](./05-structure-and-routing.md). Accessible-table and reachability semantics (README REACT-4) that the virtualized window (8.4) and image alt text (8.6) still owe the user: [07-accessibility.md](./07-accessibility.md).
- The language layer this chapter extends — the measure-first discipline and `--cpu-prof` analogue (core 15.6), the per-instance-rate cost model behind virtualization (core 15.4), tree-shaking as the bundle-budget lever (core 15.9), and the optimization ledger every fix here records (core 15.10): [../typescript/15-performance.md](../typescript/15-performance.md). Design-phase doctrine — the resource hierarchy, batching, caching — canonical for all of it: [../performance.md](../performance.md).
