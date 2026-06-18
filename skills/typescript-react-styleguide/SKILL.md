---
name: typescript-react-styleguide
description: Use when writing, editing, or reviewing React components (.tsx) in a dexpace project — extends typescript-styleguide with React rules (function components, hooks discipline, server state in TanStack Query, accessibility). Use alongside typescript-styleguide, not instead of it.
---

# React styleguide

## When this applies

Editing `*.tsx`, importing `react`/`react-dom`, writing JSX, or calling hooks. Reviewing React UI code. Priority: **correctness > performance > developer experience.**

## Inherited

First apply `typescript-styleguide`; this layer adds, and where stricter overrides, for React.

## Language hard rules

- Function components only; reach for a class solely through `react-error-boundary`. One exported component per file.
- Type props as an exported `interface XProps` beside the component; pass it as the parameter and annotate the `ReactNode` return. Never `React.FC` — it lies about `children` and infers generics worse.
- `ref` is an ordinary prop, declared in `XProps` and destructured in the signature; `forwardRef` is the pre-19 legacy form, kept only as a migration note. Destructure props in the signature and default optionals there.
- Compose with `children` and named slots, not a forest of boolean props; closed variants are a discriminated union, not parallel flags (REACT-6).
- `eslint-plugin-react-hooks` v6 `recommended` is on; `exhaustive-deps` is an error, never disabled. Restructure the effect to make the deps true — never silence the rule.
- Effects synchronize with external systems only; derive during render, never park derivable data in state and sync it with an effect. Every subscribing effect returns cleanup; every fetch effect threads an `AbortSignal`.
- Custom hooks are the unit of reuse: `use` prefix, single responsibility, named-object return. No hook calls behind a condition, loop, or early return (`use` is the lone exception).
- `useRef` for non-render state; never read or write a ref during render. Choose `useRef` vs `useState` by one question — does changing it need a re-render?
- Server state lives in TanStack Query, not `useState`; client state in `useState`/`useReducer`; URL-shaped state in the router. Model view state as a discriminated union, never a bag of booleans.
- Context wires static dependencies (theme, clients, flags), not churning state; use `<Context value={…}>`, the React 19 provider form. Reach for per-domain Zustand only for genuinely global, dynamic client state — no god-store, no new Redux.
- Queries parse every response with zod at the fetch boundary and thread the supplied `signal`; mutations declare exactly which keys they invalidate. Keys come from a factory, never string literals.
- Optimistic updates carry their rollback: `useOptimistic` for view-layer optimism, TanStack `onMutate`/`onError`/`onSettled` for cache-layer. Mutation-shaped forms use `<form action>` + `useActionState` (pending/error built in); rich validation UIs use react-hook-form. One zod schema validates and types either way.
- Accessibility is correctness: semantic HTML first, ARIA only to fill gaps, every interactive element keyboard-reachable and visibly focused, every input programmatically labelled. `eslint-plugin-jsx-a11y` `recommended` runs as the static floor.
- The React Compiler does memoization — write the plain version. Hand-written `memo`/`useMemo`/`useCallback` needs an attached Profiler trace. Fix re-render storms by structure (colocate, push down, lift to `children`), not by memo.
- **Recorded test substitution:** React component tests run on **Vitest + MSW** (with Testing Library, `user-event`, role-based queries), not `bun test` — `bun test` runs no Service-Worker interception for MSW and its DOM story is immature. Playwright covers the critical end-to-end flows.
- Inherited TS rules stand whole: no `any`, no `enum`, erasable syntax, no bare `as` or non-null `!`, discriminated unions kept whole, zod v4 top-level forms (`z.email()`/`z.uuid()`/`z.url()`), the 70-line function cap counting JSX. This layer only adds; it never weakens them.

## Before you finish — verify

```bash
gts lint        # eslint with react-hooks v6 + jsx-a11y recommended; must pass clean
tsc --noEmit    # the typecheck gate — React never type-checks at runtime
vitest run      # component tests on Vitest + MSW
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/typescript-react/README.md)
- [01 Components and Props](https://github.com/dexpace/styleguide/blob/main/typescript-react/01-components-and-props.md)
- [02 Hooks](https://github.com/dexpace/styleguide/blob/main/typescript-react/02-hooks.md)
- [03 State Management](https://github.com/dexpace/styleguide/blob/main/typescript-react/03-state-management.md)
- [04 Data Fetching and Forms](https://github.com/dexpace/styleguide/blob/main/typescript-react/04-data-fetching-and-forms.md)
- [05 Structure and Routing](https://github.com/dexpace/styleguide/blob/main/typescript-react/05-structure-and-routing.md)
- [06 Testing React](https://github.com/dexpace/styleguide/blob/main/typescript-react/06-testing-react.md)
- [07 Accessibility](https://github.com/dexpace/styleguide/blob/main/typescript-react/07-accessibility.md)
- [08 React Performance](https://github.com/dexpace/styleguide/blob/main/typescript-react/08-react-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
