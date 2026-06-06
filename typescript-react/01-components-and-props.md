# 01 — Components and Props

This guide extends [`typescript/`](../typescript/05-functions.md) for code that renders React; where it is stricter, it wins for component code. A component is a function from props to UI, so the language's function rules already govern it: the 70-line cap, guard-clause early returns, explicit return types, and the options-object instinct all carry over with the props object standing in for the options bag. This chapter fixes what React adds on top — how a component is declared, how its props are typed and named, and how it composes with other components — so that a reader meets the same shape in every file.

## What good looks like

```tsx
import {ErrorBoundary} from 'react-error-boundary';
import type {ReactNode} from 'react';

export interface UserCardProps {
  userId: UserId;
  header?: ReactNode;
  density?: 'compact' | 'cozy';
  onSelect?: (id: UserId) => void;
}

export function UserCard({
  userId,
  header,
  density = 'cozy',
  onSelect,
}: UserCardProps): ReactNode {
  const user = useUser(userId);

  if (user.isPending) return <Spinner label="Loading user" />;
  if (user.isError) return <ErrorNotice>Could not load user.</ErrorNotice>;

  const fullName = `${user.data.first} ${user.data.last}`;

  return (
    <article data-density={density}>
      {header}
      <h2>{fullName}</h2>
      <p>{user.data.email}</p>
      <button type="button" onClick={() => onSelect?.(userId)}>
        Select
      </button>
    </article>
  );
}

export function UserCardBoundary(props: UserCardProps): ReactNode {
  return (
    <ErrorBoundary fallback={<ErrorNotice>User card crashed.</ErrorNotice>}>
      <UserCard {...props} />
    </ErrorBoundary>
  );
}
```

`UserCardProps` is an exported `interface` beside the component (1.2), so a caller imports the contract. The component is a plain function, not `React.FC` (1.1, 1.3), with its props destructured and `density` defaulted in the signature (1.4) and an explicit `ReactNode` return annotating the boundary (1.3, core [5.11](../typescript/05-functions.md)). The body reads in one fixed order — hook, then the two guard-clause early returns for loading and error, then the derived value, then the happy-path return (1.6, 1.7) — and stays well under the function cap (core [05](../typescript/05-functions.md)). `useUser` returns a discriminated union, so the guards narrow it: below them `user.data` is present and `fullName` and `user.data.email` need no `?.` or fallback, the union doing the work the guard clauses earn. `header` is a `ReactNode` slot rather than a `showHeader` boolean (1.5); selection is an explicit `<button>` control rather than a click handler on the `<article>`, because an interactive surface is a control, not a div (chapter [07](./07-accessibility.md)); and the lone class-shaped concern, the error boundary, is delegated to `react-error-boundary` (1.1).

## Rules

### 1.1 — Write function components; reach for a class only through `react-error-boundary`.

**Reasoning, step by step:**
1. A class component carries `this`, lifecycle methods, and a `render` indirection that a function component does without. Root rule 1 — data and functions, not objects — applies directly: a component is a function from props to elements, and hooks give it the state and effects classes once justified.
2. Modern React features are function-only. Hooks, `Suspense` data patterns, and Server Components have no class equivalent, so a class component is a dead end the day you need any of them.
3. The single concern that still requires a class is the error boundary, because catching a render error depends on the `componentDidCatch`/`getDerivedStateFromError` lifecycle. You do not write that class: `react-error-boundary` wraps it behind a function-component API, so even this case stays a function in your code.

Worked example: a `<Page>` that needs to contain render failures imports `ErrorBoundary` and writes `<ErrorBoundary fallback={…}><Page /></ErrorBoundary>` — no `class Page extends React.Component`. The one class-shaped need is satisfied by a library component, and the application code stays entirely functions.

**Enforcement:** ESLint `react/prefer-stateless-function`; review rejects any `extends React.Component`/`PureComponent` in application code, and error boundaries go through `react-error-boundary`.

### 1.2 — Declare props as an exported `interface XProps` beside the component.

**Reasoning, step by step:**
1. Core [06](../typescript/06-classes-and-data-modeling.md)'s rule stands: an object shape is an `interface`. A props bag is exactly an object shape — named fields a caller supplies — so it is an `interface`, named `XProps` after the component `X`.
2. Export it. A caller composing or wrapping the component needs to name its contract: to build a higher-order wrapper, forward props, or write `Pick<UserCardProps, 'userId'>`. An unexported props type forces every such caller to re-describe the shape by hand.
3. Put it directly above the component, in the same file. The contract and its implementation are one unit; a reader sees the inputs, then the function that consumes them, without jumping files.

Worked example: `export interface UserCardProps {…}` sits above `export function UserCard`. A parent that renders a list writes `const cards: UserCardProps[] = …` by importing the one exported type, rather than restating `{userId: UserId; …}` at the call site.

**Enforcement:** `@typescript-eslint/consistent-type-definitions` set to `interface`; review requires the props type be exported and named `XProps`.

### 1.3 — Type the props parameter directly; never `React.FC`. Annotate the `ReactNode` return on exported components.

**Reasoning, step by step:**
1. `React.FC` was the old way to type a component, and it carries defects. It declares an implicit `children` prop every component then claims to accept, even leaf components that render none, so the contract lies. It also infers generics worse: a generic `FC` cannot flow a type parameter from props to return the way a plain generic function does.
2. The honest form types the parameter and the return separately, exactly as core [5.11](../typescript/05-functions.md) requires of every exported function: `function UserCard(props: UserCardProps): ReactNode`. `children` appears only when `UserCardProps` declares it, so a component that takes no children says so.
3. Annotate the return as `ReactNode`. It is the boundary type, written not derived, so an accidental `return undefined` or a stray non-renderable value is a compile error at the component, not a runtime blank three screens away. Prefer `ReactNode` over the legacy `JSX.Element`, which excludes the `null`/`string`/array cases a component legitimately returns.

Worked example: replace `const UserCard: React.FC<UserCardProps> = (props) => …` with `export function UserCard(props: UserCardProps): ReactNode`. The component no longer advertises a phantom `children`, and the return annotation rejects a forgotten branch that falls through to `undefined`.

**Enforcement:** ESLint `react/function-component-definition` (function declarations) and a ban on `React.FC`; `@typescript-eslint/explicit-module-boundary-types` requires the return annotation.

### 1.4 — Destructure props in the signature and give defaults there.

**Reasoning, step by step:**
1. The signature is the documentation, the same instinct as core [5.5](../typescript/05-functions.md): the parameter list is what a reader scans before the body. Destructuring the props there lists every input the component reads, in one place: `function UserCard({userId, header, density, onSelect}: UserCardProps)`.
2. Put optional defaults in that same destructure — `density = 'cozy'` — so the default sits beside the name it fills. The reader learns both the field and its fallback in one token, and the body never repeats `density ?? 'cozy'` at each use.
3. This keeps the body honest about its inputs: a prop read but absent from the destructure is a visible omission, and a `defaultProps` block (deprecated for function components) is never needed.

Worked example: `function UserCard({userId, density = 'cozy', onSelect}: UserCardProps)` documents three inputs and one default in the signature. A body that instead reads `props.density ?? 'cozy'` in three places hides the default and forces the reader to reconstruct the input list from usage.

**Enforcement:** ESLint `react/destructuring-assignment` (always destructure) and `react/require-default-props` set to the signature-default form; review rejects `defaultProps` on function components.

### 1.5 — Compose with `children` and named slots, not boolean configuration (REACT-6).

**Reasoning, step by step:**
1. A component configured by booleans accretes them: `showHeader`, `showFooter`, `compact`, `bordered`. Each is an axis, and core [6.10](../typescript/06-classes-and-data-modeling.md) applies in JSX — two interacting booleans admit four combinations where the design has fewer, and the impossible ones are bugs the props invite.
2. Composition inverts the control. Instead of a `showHeader` boolean selecting hidden markup, accept a `header?: ReactNode` slot and let the parent pass the markup it wants, or nothing. The component stops owning a catalogue of variants and renders what it is handed.
3. When two boolean props genuinely interact to select a mode, that is a discriminated union begging to exist: replace `compact` plus `bordered` with `variant: 'plain' | 'card' | 'inset'`, a closed set the component switches over (core [6.10](../typescript/06-classes-and-data-modeling.md)). One labelled axis replaces a forest of independent flags.

Worked example: a `<Dialog showTitle showClose large>` with three booleans becomes `<Dialog header={<Title />} size="large"><CloseButton /></Dialog>` — the title is a slot, the close button is `children`, and the one remaining axis is a labelled union, not a pair of booleans whose `showTitle && !header` combination was never meant to render.

**Enforcement:** Review trigger (REACT-6) — a second boolean prop prompts "do these select a variant?"; if yes, model a union or a slot. ESLint flags `defaultProps`; the boolean-forest smell is caught in review.

### 1.6 — A component obeys the function shape rules.

**Reasoning, step by step:**
1. A component is a function, so core [05](../typescript/05-functions.md) governs it whole. The 70-line cap applies — counting JSX — and a component that overruns it is doing more than one job: it is rendering and orchestrating and branching, three functions wearing one name. Extract child components and hooks until each fits.
2. Loading and error states are guard clauses (core [5.3](../typescript/05-functions.md)). Return early for each — `if (user.isPending) return <Spinner />` — so the exceptional cases leave at the top and the happy-path JSX sits flush left, not buried in a ternary pyramid inside the return.
3. One component per file. The file is named for the component it exports, the way core [05](../typescript/05-functions.md) wants a unit of reasoning to stand alone; small private sub-components used only by it may share the file, but two exported components do not.

Worked example: a 140-line `UserDashboard` that fetches, branches on three states, and lays out four panels splits into a `UserDashboard` orchestrator returning early for loading and error, plus `UserPanel`, `ActivityFeed`, and `useDashboardData` — each under the cap, each one job.

**Enforcement:** ESLint `max-lines-per-function` (the core [05](../typescript/05-functions.md) config, JSX counted) and `max-depth`; review enforces one exported component per file and guard-clause state handling.

### 1.7 — Order a component body: hooks, then derived values, then effects, then handlers, then return.

**Reasoning, step by step:**
1. Every component read in the same order is one less thing to re-learn per file. Fix the order: all hook calls first, then values derived from their results, then `useEffect` blocks, then event handlers, then the returned JSX. The body becomes a known column.
2. The order also respects React's constraints. Hooks must run unconditionally at the top, before any early return, so grouping them first makes the rules-of-hooks boundary visible — nothing conditional precedes them. Derived values come next because they read hook results; handlers come after because they close over both.
3. The returned JSX sits last, the single exit a reader scrolls to. With derivation and handlers already named above it, the JSX reads as a layout of named pieces, not a tangle of inline logic.

Worked example: in the exemplar, `useUser` (hook) precedes the loading/error guards, which precede `fullName` (derived, reading the narrowed `user.data`) and the final `return` — handlers like `onSelect?.(userId)` are wired in the JSX because they are one-liners, but a multi-statement handler would be named between the effects and the return. Reordering — a `useEffect` above a hook, or a handler defined mid-JSX — is the smell.

**Enforcement:** ESLint `react-hooks/rules-of-hooks` (hooks before returns); the full ordering is a review convention applied to every component.

### 1.8 — Keep props serializable where you can; pass functions and elements, never mutable class instances.

**Reasoning, step by step:**
1. Props that are plain data — strings, numbers, plain objects, arrays — are the React analogue of root rule 1 and core [6.3](../typescript/06-classes-and-data-modeling.md): data flows down, and data is inspectable, comparable, and serializable across a Server/Client boundary. Functions (handlers) and `ReactNode` elements are the sanctioned exceptions, since composition needs them.
2. A mutable class instance passed as a prop hides coupling. The child can call its methods and mutate its state, so the data flow the props promised is a lie, and the parent can no longer reason about what the child did to the object it lent out.
3. It also breaks memoization. `React.memo` and `useMemo` compare props by reference or shallow value; a class instance mutated in place keeps the same reference while its contents change, so a memo skips a render that should have happened. Plain data swapped for a new value on change makes the comparison honest. Across a Server Component boundary the instance is worse than wrong — non-serializable props throw.

Worked example: instead of `<Chart model={chartModel}>` where `chartModel` is a `class ChartModel` with `.addPoint()`, pass `<Chart points={points} onAddPoint={addPoint}>` — the data is a plain array `React.memo` can compare, and the behaviour is an explicit callback, so a mutation upstream produces a new `points` value and the re-render fires.

**Enforcement:** Review — props are plain data plus callbacks and `ReactNode`; a class instance in a props type is rejected, with serializability required at any Server/Client boundary.

## Cross-references

- The 70-line cap, `max-depth`, guard clauses, and explicit boundary return types that govern every component: core [05-functions.md](../typescript/05-functions.md).
- `interface` for object shapes, the optional-field explosion behind 1.5, and data-plus-functions over class instances: core [06-classes-and-data-modeling.md](../typescript/06-classes-and-data-modeling.md).
- Hooks discipline, dependency arrays, and custom hooks the 1.7 ordering assumes: [02-hooks.md](./02-hooks.md).
- State modeling as discriminated unions (the union discriminant the exemplar guards on): [03-state-management.md](./03-state-management.md) and core [06-classes-and-data-modeling.md](../typescript/06-classes-and-data-modeling.md).
