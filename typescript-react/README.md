## TypeScript with React

Binding style rules for **React UIs in TypeScript**. This guide *extends* [typescript/](../typescript/) for building user interfaces with React. It is additive: where this guide is stricter, it wins for React code; the core guide remains the canonical language baseline.

The core guide is platform-agnostic. React has its own contract — a render model that demands purity, a hooks mechanism with non-negotiable call rules, a reconciler that reuses and reorders work under an assumption of purity, and a user on the other side of every component. Those impose rules the language guide cannot address.

### Authorities

This guide extends and defers to an ordered chain. Where they conflict, the higher authority wins:

1. **[react.dev](https://react.dev) — the Rules of React** — canonical. Render purity, the Rules of Hooks, the reconciler's assumptions. These are correctness, not taste: break them and React quietly produces a stale or wrong UI — the Compiler and the reconciler both assume purity to skip and reorder work. Where our guidance collides with React's documented rules, React wins.
2. **[react-typescript-style-guide.com](https://react-typescript-style-guide.com/)** — community conventions. It fills the gaps React's docs leave open: component file structure, prop and hook typing, import and member ordering. Defer to it only where React's own rules are silent.
3. **This guide's overlay** — the dexpace React layer: server-state in a query cache, effects that synchronize and never derive, accessibility as a correctness gate, composition over boolean prop forests.

The core [typescript/](../typescript/) authority chain (Google → ts.dev → gts → its overlay) still governs the *language* layer. This guide adds the UI layer on top; it does not replace it.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Components & Props](./01-components-and-props.md) | Function components, props typed as interfaces, no `React.FC`, children/slots, one component per file |
| 02 | [Hooks](./02-hooks.md) | Rules of Hooks, custom hooks as the unit of reuse, dependency correctness, `useRef` vs `useState`, stable identities |
| 03 | [State Management](./03-state-management.md) | Server state in TanStack Query, client state in `useState`/`useReducer`, context for wiring not data, per-domain Zustand for global dynamic state, no god-store, no Redux |
| 04 | [Data Fetching & Forms](./04-data-fetching-and-forms.md) | Queries and mutations, cache invalidation, optimistic updates, react-hook-form + zod resolvers, validation at the edge |
| 05 | [Structure & Routing](./05-structure-and-routing.md) | Feature folders, route-level code splitting, query over router loaders, colocated UI, no barrels below the package root |
| 06 | [Testing React](./06-testing-react.md) | Testing Library, role-based queries, user-event over fireEvent, MSW for the network, behaviour over implementation |
| 07 | [Accessibility](./07-accessibility.md) | Semantic HTML first, ARIA only to fill gaps, keyboard reachability, focus management, `eslint-plugin-jsx-a11y` |
| 08 | [React Performance](./08-react-performance.md) | The compiler does memoization, render-cost awareness, lazy boundaries, profile before hand-tuning |

---

## The REACT rules

These add to [the 12 rules in the core guide](../typescript/README.md#the-12-rules-in-typescript). Each extends one or more core rules to the UI's edges. When a REACT rule conflicts with a core one, the REACT rule wins for React code.

### REACT-1. Render purity is law.

**Step-by-step reasoning:**
1. A component is a function from props and state to UI. During render it must be pure: same inputs, same output, no observable side effects, no mutation of anything it did not itself create that render.
2. This is not advice. The React Compiler and the reconciler both *assume* purity to skip and reorder work. When you lie — mutating a prop, reading a ref during render, writing to a module variable — nothing throws. You get a stale or wrong UI that reproduces only sometimes.
3. So all effects move out of the render path: into event handlers (the user did something) or into effects (synchronize with an outside system). Render itself only computes.
4. Props and state are read-only in the body. Derive new values, never edit the inputs. This is [core 3](../typescript/README.md#the-12-rules-in-typescript), immutable by default, made structural — React's model breaks without it.

### REACT-2. Hooks rules are compiler-checked.

**Step-by-step reasoning:**
1. Hooks are called unconditionally, in the same order, at the top level of a component or another hook. The mechanism is positional: a conditional hook corrupts the state slots for every hook after it.
2. This is mechanical and therefore machine-checkable. The overlay runs `eslint-plugin-react-hooks` v6 with the `recommended` config — the compiler-powered ruleset that flags render-purity violations alongside the classic call rules.
3. `exhaustive-deps` is an **error**, not a warning. A missing dependency is a stale-closure bug waiting for the input it forgot to change. The fix is correcting the dependency, never silencing the rule.
4. When the rule and your instinct disagree, the rule is right. This makes [core 8](../typescript/README.md#the-12-rules-in-typescript), assert aggressively, the linter's job for the one invariant React cannot enforce at runtime.

### REACT-3. Server state is not client state.

**Step-by-step reasoning:**
1. Data owned by the server — fetched, shared across views, able to go stale — is a *cache*, not application state. Storing it in `useState` reinvents caching, deduplication, and revalidation by hand, badly.
2. Server state lives in TanStack Query's cache, keyed and governed by explicit staleness semantics. The cache owns freshness, retries, and background refetch; components read from it.
3. `useState` and `useReducer` hold only the client's own facts: form drafts, toggles, selection, the things the server never knew. If a value came from or returns to the server, it is query state.
4. Keeping these separate kills a whole class of bugs — duplicated fetches, inconsistent copies of one record, manual loading flags. This extends [core 13](../typescript/13-resource-management.md), resource management, to the data layer of the UI.

### REACT-4. Accessibility is correctness.

**Step-by-step reasoning:**
1. A feature a user cannot reach — by keyboard, by screen reader — is a broken feature, the same as one that throws. Correctness is the whole audience, not the subset using a mouse and sighted.
2. Semantic HTML is the default: a `<button>` is a button, a `<nav>` is navigation. ARIA fills the gaps native elements leave, and a wrong ARIA attribute is worse than none.
3. The keyboard path and focus management are part of the feature, designed in, not retrofitted. `eslint-plugin-jsx-a11y` catches the static mistakes in the overlay.
4. Tests enforce it: role-based queries (`getByRole('button', { name })`) only pass against an accessible tree, so the same test that proves behaviour proves reachability. This is the [correctness value](../typescript/README.md#values) of the core guide applied to the human at the edge.

### REACT-5. Effects synchronize, never derive.

**Step-by-step reasoning:**
1. Most values people reach for an effect to compute can be computed during render from props and state. A derived value held in state and synced by an effect is a second source of truth that drifts and renders twice.
2. So derive during render. Filtered lists, formatted strings, totals — compute them in the body. If the work is expensive, the compiler memoizes it; you do not.
3. An effect is for synchronizing with a system *outside* React: a subscription, a DOM node you must touch imperatively, a non-React widget, an analytics call. That is the entire job description.
4. Every effect that subscribes returns its cleanup, and its dependencies are honest (REACT-2). An effect that only reshapes React state into other React state is a bug. This extends [core 6](../typescript/README.md#the-12-rules-in-typescript), transform don't mutate, into the render lifecycle.

### REACT-6. Composition over configuration.

**Step-by-step reasoning:**
1. A component that grows a forest of boolean props — `isPrimary`, `withIcon`, `compact`, `noBorder` — is encoding a type hierarchy in flags. The combinations multiply, most are nonsense, and the body becomes a maze of conditionals.
2. Compose instead. Pass `children` and named slots; let the caller assemble the parts. A `<Card>` takes a `<CardHeader>` and `<CardBody>`, not a dozen switches describing every header it might render.
3. When variants are genuinely closed, model them as a discriminated union of props, so illegal combinations cannot be expressed — not as independent booleans that can.
4. This is [core 1 and core 5](../typescript/README.md#the-12-rules-in-typescript) in JSX terms: data-and-functions over configuration objects, composition over inheritance. The component tree *is* the composition.

---

## Deviations from Upstream

This guide takes React's Rules of React as canonical and the community style guide as the convention layer. These entries are recorded so they can be revisited surgically.

| Rule | Upstream position | Our position | Why |
|---|---|---|---|
| GraphQL conventions | The community guide ships a GraphQL chapter | Not adopted | There is no dexpace GraphQL surface; our data layer is TanStack Query over typed HTTP (REACT-3). Revisit if GraphQL is adopted. |
| `React.FC` | Tolerated in places upstream | Banned — type props as an explicit interface parameter | Worse inference for generic components, plus the legacy implicit-`children` it drags in. See [01-components-and-props.md](./01-components-and-props.md). |
| File naming | Core guide names files kebab-case (core [2.4](../typescript/02-naming-conventions.md)) | React PascalCase component files, camelCase others | A component file's name is its import name and React components are PascalCase symbols, so the file matches. See [05-structure-and-routing.md](./05-structure-and-routing.md). |

---

Zero technical debt holds here as everywhere: what ships meets the design goals. **Perfection over technical debt — debt never gets paid.** A UI shortcut is the inaccessible control, the stale cache, the re-render storm that someone else debugs in production.

## Influences

- **[react.dev — Rules of React](https://react.dev/reference/rules)** — canonical: render purity, the Rules of Hooks, the reconciler's assumptions.
- **[react-typescript-style-guide.com](https://react-typescript-style-guide.com/)** — community conventions for component structure, prop typing, and ordering.
- **[TanStack Query](https://tanstack.com/query/latest)** — server-state cache with explicit staleness semantics.
- **[Testing Library](https://testing-library.com/)** — role-based, behaviour-first component testing.
- **[eslint-plugin-jsx-a11y](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y)** — static accessibility enforcement.
- **TigerBeetle Tiger Style** — limits on everything, correctness as the whole contract, zero technical debt.
