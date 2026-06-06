# 02 — Hooks

Hooks are how a component reaches outside its own render: state that survives a re-render, a subscription to a store, a fetch tied to a prop. React calls your component during render and trusts it to be pure — same inputs, same JSX, no side effects on the way through ([react.dev: Rules of React](https://react.dev/reference/rules)). Hooks are the sanctioned escape from that purity, and the React Compiler now checks the boundary mechanically. This chapter keeps hooks honest: effects synchronize with external systems and nothing else, every effect cleans up, dependencies never lie, and composition reads top-down the way functions do in the core guide. Cancellation and resource discipline are not re-derived here — they are the core guide's ([../typescript/09-concurrency.md](../typescript/09-concurrency.md) §9.5, [../typescript/13-resource-management.md](../typescript/13-resource-management.md) §13.4) rules, wearing a component's lifecycle.

## What good looks like

```tsx
/** Sync a component to a keyboard shortcut: bind the listener, follow the latest handler, drop it on unmount. */
function useKeyboardShortcut(combo: string, onMatch: () => void): { bound: boolean } {
  const handlerRef = useRef(onMatch); // 2.7 non-render mutable: latest handler without rebinding the listener

  useEffect(() => {
    handlerRef.current = onMatch; // 2.7 written in an effect, never during render
  }, [onMatch]);

  useEffect(() => {
    const controller = new AbortController(); // 2.4 unmount is a cancellation (core 13.4)
    const wanted = combo.toLowerCase().split('+'); // normalize once, off the listener's hot path

    const onKeyDown = (event: KeyboardEvent): void => {
      const pressed = [
        event.ctrlKey && 'ctrl',
        event.metaKey && 'meta',
        event.shiftKey && 'shift',
        event.key.toLowerCase(),
      ].filter(Boolean);
      if (wanted.every((k) => pressed.includes(k))) handlerRef.current(); // read the ref, not a stale closure
    };

    window.addEventListener('keydown', onKeyDown, { signal: controller.signal }); // core 13.4 listener via { signal }

    return () => controller.abort(); // 2.4 cleanup removes the listener — one abort, no manual removeEventListener
  }, [combo]); // 2.2 deps honest: the effect reads only combo (the handler is read through the ref)

  return { bound: true }; // 2.5 named object, not a bare boolean
}
```

This effect synchronizes the component with an external system — the document's keyboard — and does nothing derivable (2.3); it registers the `keydown` listener with `{ signal }` and returns a cleanup that aborts the controller, so one `controller.abort()` removes the listener on unmount or a changed `combo` with no manual `removeEventListener` (2.4, core 13.4); its dependency array names exactly what the effect reads, `[combo]`, while the freshest `onMatch` is reached through a ref so a new handler does not rebind the listener (2.2, 2.7); and it returns a named object so the call site reads by meaning (2.5). Note what this hook is *not*: it never fetches server data into `useState` — that is a query cache's job (REACT-3, [03 §3.3](./03-state-management.md)), not an effect's. A custom hook earns an effect only when it is syncing with a genuinely external, non-server system like this one.

## Rules

### 2.1 — `eslint-plugin-react-hooks` v6 `recommended`, on in the React overlay.

**Reasoning, step by step:**
1. The Rules of React are correctness rules, not style preferences. Calling hooks conditionally, mutating props, or running effects during render produces bugs that are intermittent, environment-dependent, and brutal to reproduce — exactly the class the core guide spends its assertion budget pushing to compile time. Lint catches them at author time instead.
2. Version 6 of the plugin bundles the React Compiler's static analyses as lint rules: `purity` (render computes without observable side effects), `immutability` (props and state are not mutated), `set-state-in-render` and `set-state-in-effect` (no render-phase or unconditional-effect state writes that loop), and `refs` (no reading or writing refs during render). These are the same checks the compiler runs to prove a component safe to memoize — running them as lint means the codebase stays compiler-ready whether or not the compiler is enabled.
3. Adopt `recommended` wholesale rather than cherry-picking. The set is curated to be true-positive-heavy; turning rules off individually is how a team accumulates the bugs the plugin exists to prevent (REACT-2).

**Enforcement:** `eslint-plugin-react-hooks` v6, `recommended` config extended in the React overlay; the bundled compiler-powered rules (`purity`, `immutability`, `set-state-in-render`, `set-state-in-effect`, `refs`) run on every file.

### 2.2 — `react-hooks/exhaustive-deps` is an error and is never disabled.

**Reasoning, step by step:**
1. The dependency array is a claim: "this effect's behavior depends on exactly these values." When the claim is false — a value read inside the effect is omitted — the effect holds a stale closure, running with the values from the render where it last fired. On a timer or a subscription that fires repeatedly, that is a value frozen in the past, and the bug surfaces as data that silently stops updating.
2. The lint rule computes the honest dependency set for you. A `// eslint-disable-next-line react-hooks/exhaustive-deps` is therefore a written admission that the effect is lying about what it reads — the precise stale-closure defect the rule exists to surface, suppressed at the one place it was caught.
3. When the array is "too big" or refires too often, that is signal, not noise: restructure. Move a function inside the effect so it is not a dependency, lift it out of the component if it is pure, wrap a genuinely stable callback (2.9), or split the effect so each piece depends on less. The fix is always to make the dependencies true, never to silence the check.

```tsx
// bad — count omitted; the interval closes over the first render's count forever
useEffect(() => {
  const id = setInterval(() => setTotal(count + 1), 1000);
  return () => clearInterval(id);
}, []); // eslint-disable-line react-hooks/exhaustive-deps  ← a lied-about dependency

// good — the updater reads no outer value, so the deps are honestly empty
useEffect(() => {
  const id = setInterval(() => setTotal((t) => t + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**Enforcement:** `react-hooks/exhaustive-deps` configured as `error`; suppression comments for it are rejected in review — the effect is restructured so the array is true.

### 2.3 — Effects synchronize with external systems only.

**Reasoning, step by step:**
1. An effect's job is to connect the component to something outside React's render: a network request, a DOM node React does not own, a browser API, a non-React store, a subscription (REACT-5). If an effect's body would still make sense as the answer to "what external system am I keeping in sync?", it belongs. If not, it is in the wrong place.
2. Data you can compute from props and state is computed during render, not stored in state and synced by an effect. The "derived state in an effect" pattern — `useEffect(() => setFullName(first + ' ' + last), [first, last])` — is the canonical anti-pattern ([react.dev: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)): it adds a render, risks an extra one, and creates a second source of truth that can disagree with the first. Just compute it inline.
3. Event-specific logic belongs in event handlers, not effects. "When the user clicks Buy, POST the order" is a handler; an effect that watches a flag and fires the request reintroduces the ordering and double-fire problems the handler avoids. Effects react to *being rendered with certain values*, not to *something having happened*.

```tsx
// bad — derivable data parked in state and synced by an effect
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(`${first} ${last}`); }, [first, last]);

// good — computed in render; one source of truth, no extra pass
const fullName = `${first} ${last}`;
```

**Enforcement:** review; an effect that only computes from props/state is replaced with a render-time expression, and event-triggered work moves to a handler. `set-state-in-effect` (2.1) catches a subset mechanically.

### 2.4 — Every effect returns cleanup; every fetch effect wires an `AbortSignal`.

**Reasoning, step by step:**
1. An effect that subscribes, opens, or starts something must return the function that reverses it. React runs that cleanup before the next effect run and on unmount, and in development runs setup→cleanup→setup once to flush out effects that forget. A subscription with no `removeEventListener`, a timer with no `clearInterval`, a connection with no close — each is the same leak the core guide names in [../typescript/13-resource-management.md](../typescript/13-resource-management.md) §13.5, now firing on every dependency change.
2. For a fetch, unmount *is* a cancellation. Create an `AbortController` inside the effect, pass `controller.signal` to `fetch`, and return `() => controller.abort()` — the exact lifecycle handle from core §9.5 and §13.4, scoped to one effect run. Without it, a response that arrives after the component navigated away calls `setState` on a gone component and, worse, an out-of-order earlier request can clobber a later one (the core §9.10 interleaving race, in a UI).
3. Guard the post-abort path: in the `catch`, return early when `signal.aborted`, because an aborted fetch rejects and that rejection is expected teardown, not an error to surface to the user. The controller is per-run, so each dependency change aborts the prior request before starting the next — no shared mutable flag, no manual `removeEventListener`.

```tsx
useEffect(() => {
  const onResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize); // cleanup mirrors setup
}, []);
```

**Enforcement:** review; every subscribing/opening effect returns a cleanup that reverses setup, and every fetch effect threads an `AbortController.signal` and aborts in cleanup (core 9.5/13.4).

### 2.5 — Custom hooks: `use` prefix, single responsibility, named-object return.

**Reasoning, step by step:**
1. The `use` prefix is not decoration — it is what tells React and the linter this function may call hooks, so the Rules of Hooks (2.8) apply to it. A function that calls `useState` but is named `getUser` is invisible to the tooling and a bug magnet; a function named `useUser` that calls no hooks is a lie. The prefix and the behavior must agree.
2. A custom hook does one thing. `useUser` fetches a user; it does not also manage a modal and a form. Single responsibility is the [../typescript/05-functions.md](../typescript/05-functions.md) §5.1 "one job" rule applied to hooks — the test is the same, if the honest summary needs an "and," it is two hooks. Small hooks compose (2.6); a kitchen-sink hook does not.
3. Return a named object so call sites read by meaning: `const { user, loading, error } = useUser(id)`. A positional tuple forces the caller to remember order and makes a three-value return unreadable. The one carve-out is a `useState`-like primitive returning a value and its updater, where a 2-tuple is idiomatic precisely because the caller renames both at the destructure — `const [open, setOpen] = useToggle()`, signature `readonly [boolean, () => void]`. Past two, or when the elements are not a value/updater pair, use an object.

**Enforcement:** review; custom hooks are `use`-prefixed iff they call hooks, hold a single responsibility, and return a named object (or a ≤2 tuple for `useState`-like value/updater pairs).

### 2.6 — Hooks compose top-down, the way functions do.

**Reasoning, step by step:**
1. A custom hook calls other hooks, so hooks form a tree with the same shape as a call tree — and it reads best with the same discipline as core [../typescript/05-functions.md](../typescript/05-functions.md) §5.4 step-down: the orchestrating hook sits above the ones it calls, each layer one level of abstraction below the last. A page-level hook orchestrates feature hooks, which orchestrate primitive hooks.
2. Concretely: `useCheckoutPage` calls `useCart` and `useShippingOptions`; `useCart` calls `useUser` and a `useLocalStorage` primitive. Each hook is small and testable, each layer names what it composes without reaching two levels down. This is how a screen's data dependencies stay legible — you read the top hook to learn what the page needs, and descend only as far as the question requires.
3. Push shared logic *down* into a reusable hook rather than *sideways* by duplicating effect code across components. Two components running the same subscription should both call one `useSubscription` hook; the deduplication that core §12 wants for functions applies identically to hooks.

**Enforcement:** review; a hook that composes others declares them top-down with the orchestrator above its callees (`useCheckoutPage` above `useCart` above `useUser`), and shared effect logic is extracted into a hook rather than copied across components.

### 2.7 — `useRef` for non-render state; never read or write a ref during render.

**Reasoning, step by step:**
1. `useRef` holds a mutable value that survives re-renders *without* triggering one. That is exactly right for things the UI does not display: a DOM node handle, a timer id, the previous value of a prop, a mutable instance that outlives a render. Reaching for `useState` here would re-render on every change for no visual reason; reaching for a module variable would leak state across component instances.
2. A ref is an escape hatch with a contract: it is read and written in effects and event handlers — *after* render — never during render itself. Reading `ref.current` while rendering ties the output to a value React does not track as a dependency, so the screen silently shows stale data; writing it during render is a side effect in a function that must be pure. The compiler's `refs` rule (2.1) flags both.
3. If a value affects what the user sees, it is state, not a ref. The decision is one question: does changing this need to re-render? Yes → `useState`/`useReducer`. No → `useRef`. So a `timerRef` holding a `setTimeout` id is read only in cleanup (`if (timerRef.current) clearTimeout(timerRef.current)`), while `return <span>{lastValueRef.current}</span>` is a bug — reading a ref during render couples the output to a value React never tracked, and the screen shows stale data. Using a ref to dodge a needed re-render is how the screen and the data drift apart.

**Enforcement:** `react-hooks` `refs` rule (2.1); review confirms refs hold only non-render state and are touched in effects/handlers, never read or written during render.

### 2.8 — No hook calls behind a condition, loop, or early return.

**Reasoning, step by step:**
1. React identifies a hook by its *call order*, not a name — the first `useState` this render is the same state as the first `useState` last render. Put a hook behind an `if`, a loop, or after an early `return`, and the order shifts between renders; React then pairs your state with the wrong slot, and the component corrupts in ways the stack trace cannot explain. This is the one rule whose violation has no graceful failure.
2. Hooks are called unconditionally, at the top level of the component or custom hook, every render, same order. The condition you wanted goes *inside* the hook — `useEffect(() => { if (!enabled) return; … }, [enabled])` — or *around the component* by splitting it, so the conditional path is a different component each with its own honest, unconditional hook list.
3. When a branch needs a whole different set of hooks, that is the signal to extract a component, not to branch the hooks. A parent renders `<Editor>` or `<Viewer>` by condition; each child calls its own hooks unconditionally. Early-return components are the idiom; conditional hooks are never the answer.

```tsx
// bad — hook order changes when isLoggedIn flips
if (isLoggedIn) { const [name, setName] = useState(''); /* … */ }

// good — unconditional hook; the condition lives inside the effect
const [name, setName] = useState('');
useEffect(() => { if (!isLoggedIn) return; /* … */ }, [isLoggedIn]);
```

**Enforcement:** `react-hooks/rules-of-hooks` (in `recommended`, 2.1); a branch needing different hooks is split into separate components, each with an unconditional hook list.

### 2.9 — Don't hand-memoize first; reach for `useMemo`/`useCallback` only with a profile attached.

**Reasoning, step by step:**
1. The React Compiler memoizes components and hook results automatically, covering the default case better than hand-placed memos and without the dependency-array maintenance that `useMemo`/`useCallback` impose. With the compiler on, most manual memoization is dead weight: noise that can drift out of sync with its dependencies and mislead the next reader into thinking a real bottleneck was found here. Write the straightforward version first and let the compiler do its job.
2. Manual `useMemo`/`useCallback` is a deliberate optimization, and the core guide's [../typescript/15-performance.md](../typescript/15-performance.md) §15.10 rule governs every deliberate optimization: it carries the measurement that justifies it. A `useMemo` with a React Profiler trace at the call site — "this list re-rendered 38ms/keystroke, memo drops it to 4ms" — is a ledger entry; a bare `useMemo` wrapping a cheap computation is folklore, and folklore is removed in review.
3. The genuine cases are narrow and nameable: a referentially-stable value passed to a dependency array or a `memo`'d child to stop a re-render cascade, or a measurably expensive computation in a hot render. Each is justified by a profiler trace, not a hunch that "this might be slow" — see the rendering-performance chapter ([08](./08-react-performance.md)) for the measure-first workflow.

```tsx
// bad — wrapping a trivial computation; no measurement, the compiler already handles this
const label = useMemo(() => `${first} ${last}`, [first, last]);

// good — stable identity for a memo'd child, with the ledger note that earns it
// React Profiler: <Chart> re-rendered every keystroke, 38ms → 4ms with this callback memoized.
const onSelect = useCallback((p: Point) => setSelected(p), []);
```

**Enforcement:** review; manual `useMemo`/`useCallback` requires an attached React Profiler measurement (core 15.10), is otherwise removed in favor of the compiler's automatic memoization; see [08](./08-react-performance.md).

## Cross-references

- Cancellation in the signature, `AbortSignal.timeout`/`any`, signal propagation, the interleaving race: [core §09](../typescript/09-concurrency.md) (9.5, 9.6, 9.10, 9.11).
- `AbortController` as the universal lifecycle handle, timer ownership, listeners via `{ signal }`: [core §13](../typescript/13-resource-management.md) (13.4, 13.5).
- One job per function, the step-down rule: [core §05](../typescript/05-functions.md) (5.1, 5.4); boundary parsing and `toError`: [core §03](../typescript/03-the-type-system.md), [core §08](../typescript/08-error-handling.md).
- The optimization ledger — every deliberate optimization carries its measurement: [core §15](../typescript/15-performance.md) (15.10).
- Rendering performance, the React Profiler, `memo`, when memoization earns its place: [08](./08-react-performance.md); component structure and where hooks sit relative to components: [01](./01-components-and-props.md).
