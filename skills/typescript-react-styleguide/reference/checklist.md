# React styleguide checklist

Per-chapter audit for `typescript-react`. Extends `typescript-styleguide`; inherited TS rules still apply. Where this layer is stricter, it wins for React.

### 01 — Components and Props

- Function components only; the one class need (error boundary) goes through `react-error-boundary`. No `extends React.Component` in app code.
- Props are an exported `interface XProps` beside the component; export it so callers can `Pick`/forward.
- Type the props parameter directly and annotate the `ReactNode` return; never `React.FC` (phantom `children`, worse generics). Prefer `ReactNode` over `JSX.Element`.
- Destructure props in the signature; default optionals there. `ref` is a prop in `XProps`, not `forwardRef` — reach for the wrapper only as a migration note.
- Compose with `children` and named slots over boolean configuration; a second interacting boolean is a discriminated union (REACT-6).
- A component obeys the function rules: 70-line cap counting JSX, guard-clause early returns for loading/error, one exported component per file.
- Order the body: hooks → derived values → effects → handlers → return.
- Keep props serializable where you can; pass functions and `ReactNode`, never mutable class instances (breaks memo and the Server/Client boundary).

### 02 — Hooks

- `eslint-plugin-react-hooks` v6 `recommended` on; its compiler-powered rules (`purity`, `immutability`, `set-state-in-render`, `set-state-in-effect`, `refs`) run on every file.
- `exhaustive-deps` is an error and is never disabled; a suppression is an admission the effect lies — restructure so the array is true.
- Effects synchronize with external systems only; compute derivable data during render, put event-triggered work in handlers.
- Every subscribing/opening effect returns cleanup that reverses setup; every fetch effect threads an `AbortController.signal` and aborts in cleanup, guarding the post-abort path.
- Custom hooks: `use` prefix iff they call hooks, single responsibility, named-object return (a ≤2 tuple only for value/updater pairs).
- Hooks compose top-down like functions; push shared logic down into a hook, subscribe external stores via `useSyncExternalStore`.
- `useRef` holds non-render state only; never read or write a ref during render. Decide by "does changing it need a re-render?"
- No hook behind a condition, loop, or early return — split into separate components instead. `use` is the sole call allowed behind a guard.
- Don't hand-memoize first; `useMemo`/`useCallback` needs an attached Profiler trace (core 15.10), else the compiler owns it.

### 03 — State Management

- Keep state local first; lift to a common ancestor only when two components genuinely share it.
- Model view state as a discriminated union over a `status` discriminant, not parallel booleans; `switch` exhaustively.
- Server state is TanStack Query, full stop; never copy a response into `useState`; mutations invalidate affected keys.
- Context injects static dependencies (theme, clients, flags), not churning state; provide with `<Context value={…}>` (React 19 form).
- Zustand only for genuinely global and dynamic client state; one store per domain, state as a union mutated through named actions; no god-store.
- No Redux in new code; legacy slices stay with a ledger note. Modern split: Query for server, per-domain store for global client, component for local.
- Express multi-transition state as a pure `(state, action) => state` reducer (`useReducer`); `useActionState` owns the async submission-shaped machine.
- Treat the URL as state for filters, tabs, pagination, sort, selection; read from the router, write by navigating, never mirror into `useState`.

### 04 — Data Fetching and Forms

- Query keys come from a per-resource factory built with `as const`, never string literals; nest keys so invalidation can be coarse or fine.
- Parse every response with zod at the fetch boundary; derive the type with `z.infer`, call `.readonly()`; never `as`-cast a response.
- Destructure `{signal}` from the `queryFn` argument and thread it to `fetch` so navigation cancels in-flight work (core 9.5).
- Mutations declare exactly which queries they invalidate in `onSuccess` via the key factory; never bare `invalidateQueries()`, never invalidate nothing.
- Optimistic updates carry rollback: `useOptimistic` for view-layer, TanStack `onMutate`/`onError`/`onSettled` for cache-layer; if you can't write the rollback, show a pending state instead.
- Mutation-shaped forms use `<form action>` + `useActionState` (pending/error built in); complex validation UIs use react-hook-form with `zodResolver`. One zod schema validates and types both paths; no per-field `useState`.
- Wrap each route or independently-failing feature in an error boundary; pair query-driven boundaries with `QueryErrorResetBoundary` and `throwOnError`.
- Derive loading UX from query state (`isPending`/`isError`/`isSuccess`), never bespoke flags; render skeletons shaped like the content.

### 05 — Structure and Routing

- Everything a feature needs lives in its feature folder (components, hooks, queries, types, tests); group by feature, not by kind. No `index.ts` barrel inside a feature or `common/`.
- No cross-feature imports except through `common/`; gate with `import/no-restricted-paths` and `madge --circular`. Promote to `common/` only after a second consumer exists.
- PascalCase component files (import name = file name), camelCase folders and non-component files; hooks keep the `use` prefix. This overrides the core kebab-case rule, recorded in the README ledger.
- Keep route components thin: compose features, wire URL params, own Suspense/error boundaries, prefetch at the seam — data lives in TanStack Query, not router loaders.
- Colocate tests, stories, and styles beside their component (`X.tsx`, `X.test.tsx`, `X.stories.tsx`).
- One styling system per project, named in the README; Tailwind or CSS Modules (static at build), no runtime CSS-in-JS.
- GraphQL conventions: not adopted — ledger entry, reopen only on GraphQL adoption.

### 06 — Testing React

- Component tests run on **Vitest + MSW** (recorded substitution from `bun test`); globals off. Playwright covers critical end-to-end flows only.
- Query by accessible role first (`getByRole` with name); `data-testid` is the last resort and a hint the component may be missing a role.
- Drive interaction with `user-event` (`setup()` once per test, await every step), never `fireEvent`.
- Route every request through MSW with `onUnhandledRequest: 'error'`; never `vi.mock` an owned API module; query client retries off.
- Test behaviour the user observes; no assertions on class names, hook state, or whole-tree snapshots — only scoped inline snapshots of meaningful output.
- Resolve async UI with `findBy*`/`waitFor` on bounded timeouts; never sleep; a long timeout flags a missing fake timer.
- Assert accessibility in component tests — accessible names, roles, focus movement (`toHaveAccessibleName`, `toHaveFocus`).
- Assert negative space: the absent error, the forbidden state, the gone item, via `queryBy*` and state matchers.

### 07 — Accessibility

- Semantic HTML first; ARIA fills gaps only — no `role` on an element a native tag covers. No ARIA is better than bad ARIA.
- Every interactive element is keyboard-reachable and visibly focused; tab-through is a review gate; never `outline: none` without a `:focus-visible` replacement. Manage focus (trap in dialogs, return to trigger, move to errored field).
- Every input has a programmatic label (`<label htmlFor>` or `aria-labelledby`); placeholder is never a label. Wire errors with `aria-describedby` + `aria-invalid` + a live region.
- `eslint-plugin-jsx-a11y` `recommended` runs as the mechanical floor with violations as errors; it catches the static half only.
- Images, icons, and media carry text alternatives: informative `alt` describes, decorative `alt=""`, icon-only controls get `aria-label` with the glyph `aria-hidden`, media gets captions/transcripts.
- Color is never the only signal — pair it with text/icon/shape; contrast meets WCAG AA (4.5:1 text, 3:1 large/UI).
- Role-query testing is the enforcement loop: an element unreachable by `getByRole` is unreachable by assistive tech; assert focus and live-region behaviour.

### 08 — React Performance

- Let the React Compiler (stable v1) memoize; hand-written `memo`/`useMemo`/`useCallback` needs an attached Profiler trace, else it is removed.
- Fix re-render storms by structure, not memo: colocate state, push state down, lift slow content up as `children`.
- Split at route boundaries with `lazy()` + Suspense; go finer only on bundle-analyzer evidence.
- Virtualize long lists with TanStack Virtual; hundreds of rows is a windowing problem, not a per-row `memo` problem.
- Budget the bundle in CI; diff against base, fail per-route size budgets; a dependency that doubles a chunk is a regression.
- Treat images and fonts as performance: explicit dimensions or `aspect-ratio`, modern formats, `loading="lazy"`, `font-display: swap`.
- Measure with the React DevTools Profiler and Web Vitals (LCP/INP/CLS); optimize against traces, record before/after as a ledger entry, never against vibes.
