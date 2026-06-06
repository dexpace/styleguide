# 03 — State Management

React gives you many places to put state and almost no opinion about which to use. This chapter is that opinion. It says where each kind of state belongs — local to a component, the URL, a server cache, a global store, Context — and how to shape each so that an impossible UI cannot be rendered. The governing idea is that most "state management" problems are really state-*placement* problems: put each piece of state in the one place that owns it, model it so illegal combinations do not typecheck, and the bugs you would have managed never get created.

## What good looks like

```tsx
// View is a closed union: editing carries a draft, idle does not — illegal blends cannot be built (3.2).
type View =
  | {readonly status: 'idle'}
  | {readonly status: 'editing'; readonly draft: string};

function fetchUser(id: UserId): Promise<User> {/* ... */}
export function UserPanel({id}: {readonly id: UserId}) {
  const qc = useQueryClient();
  // Server slice lives in the cache, never copied into useState (3.3).
  const user = useQuery({queryKey: ['user', id], queryFn: () => fetchUser(id)});
  // Mutation invalidates the key so the cache refetches — no manual re-sync (3.3).
  const rename = useMutation({mutationFn: (n: string) => saveUserName(id, n), onSuccess: () => qc.invalidateQueries({queryKey: ['user', id]})});
  const [view, setView] = useState<View>({status: 'idle'});  // local, lifted no higher (3.1)

  if (user.isPending) return <Spinner />;
  if (user.isError) return <ErrorBox error={user.error} />;
  switch (view.status) {
    case 'idle':
      return <Profile user={user.data} onEdit={() => setView({status: 'editing', draft: user.data.name})} />;
    case 'editing':
      return <NameForm draft={view.draft} onSubmit={(name) => rename.mutate(name)} />;
  }
}
```

This exemplar keeps the server slice in TanStack Query and never copies it into `useState` (3.3); the local `view` lives in the only component that uses it and is lifted no higher (3.1); and `View` is a discriminated union, so `idle` carries no `draft` and the loading/error states come from the query rather than parallel booleans (3.2, applying core [6.1](../typescript/06-classes-and-data-modeling.md) inside React).

## Rules

### 3.1 — Keep state local first; lift only when two components genuinely share it.

**Reasoning, step by step:**
1. State has an owner: the component that reads it and writes it. Put it there with `useState`. Co-locating state with its single consumer keeps the data beside the logic that changes it, and unmounting the component disposes the state for free.
2. Lift to a common ancestor only once two siblings *both* read or write the value. Lifting earlier is speculative generality — re-render and prop-drilling cost for sharing that does not yet exist. The cost is asymmetric: lifting later is a local, mechanical refactor, while un-lifting means untangling consumers that already depend on the higher position. Prefer the change that is cheap to reverse.

```tsx
// bad — draft lives in the parent but only Editor reads or writes it, then is drilled back down
const [draft, setDraft] = useState('');  // ...later: <Editor draft={draft} setDraft={setDraft} />
// good — owned by its only consumer; lift the day a sibling needs it
function Editor() {
  const [draft, setDraft] = useState('');
  return <textarea value={draft} onChange={(e) => setDraft(e.target.value)} />;
}
```

**Enforcement:** Review heuristic — state passed straight through a component to a single child belongs in that child; a prop drilled more than two levels prompts the question "who actually owns this?"

### 3.2 — Model view state as a discriminated union, not a bag of booleans.

**Reasoning, step by step:**
1. Parallel booleans multiply the state space. `isLoading`, `isError`, and `isSuccess` admit eight combinations; four are legal and the rest — `isLoading && isError`, all-false — are renderable bugs. This is core [6.1](../typescript/06-classes-and-data-modeling.md) ("make illegal states unrepresentable") applied to a component.
2. Replace them with one `status` discriminant of `'loading' | 'success' | 'error'`, and hang each state's data off the member that owns it: `data` only on `success`, `error` only on `error`. The contradictory combinations stop being constructible, and rendering becomes a `switch` on the discriminant — the compiler narrows each branch, so `data` is in scope only under `success` and an added variant fails the exhaustiveness check rather than rendering blank.

```tsx
// bad — isLoading && isError is representable, and data may be null on success
interface State {isLoading: boolean; isError: boolean; data: User | null}
type State =  // good — one discriminant; each state owns exactly its fields
  | {status: 'loading'} | {status: 'success'; data: User} | {status: 'error'; error: Error};
```

**Enforcement:** `@typescript-eslint/switch-exhaustiveness-check` over the `status`; review rejects two booleans that encode one machine.

### 3.3 — Server state is TanStack Query, full stop.

**Reasoning, step by step:**
1. This rule is REACT-3 ("server state is not client state") made concrete. Data fetched from a server is not your state — it is a *cache* of state that lives elsewhere, with staleness, refetching, and invalidation semantics. TanStack Query models exactly that: keyed entries, background revalidation, deduplication, and `isPending`/`isError`/`data` already exposed as a discriminated lifecycle (status `'pending' | 'error' | 'success'`), the same union shape 3.2 prescribes.
2. Copying the response into `useState` forks the truth. Your snapshot drifts as the cache updates, so you write `useEffect` chains to sync them — each a fresh opportunity to disagree. That sync chain is the wrong tool: effects synchronize with external systems, they do not mirror one piece of in-memory state into another (REACT-5, 02-hooks 2.3). The bug class exists only because you made a second copy. Read server data straight from the query where you render; mutations go through `useMutation` and invalidate the affected keys, so the cache refetches and every consumer updates from one source.

```tsx
const {data, isPending, isError} =  // good — cache owns it; read at point of use, never mirrored
  useQuery({queryKey: ['user', id], queryFn: () => fetchUser(id)});
```

**Enforcement:** Review rejects `useEffect` + `useState` fetch-and-store for server data; server payloads are read from query hooks, never mirrored into component state.

### 3.4 — Use Context for static dependencies, not as a state manager.

**Reasoning, step by step:**
1. Context is dependency injection: it threads a value — the theme, the auth client, feature flags, the query client — to a subtree without prop drilling. These are low-frequency values: they are set once and change rarely or never over a session.
2. Context is not a state container: every consumer re-renders whenever the provider's value changes, with no selector to subscribe to a slice. Put frequently-changing state behind it and an unrelated update repaints the whole subtree. Keep Context values stable and coarse; dynamic, high-frequency client state belongs in a store with selective subscription (3.5), while Context carries the dependencies that store and components both need.
3. Provide with `<Context value={…}>` directly — in React 19 the context object *is* the provider, so `<Context.Provider>` is the legacy form ([react.dev: Context as a provider](https://react.dev/blog/2024/12/05/react-19#context-as-a-provider)). Read at the top of a hook with `useContext`; reach for `use(Context)` only when the read must sit behind a condition or an early return (02-hooks 2.8), since `use` is the one reader allowed there.

```tsx
const AuthContext = createContext<AuthClient | null>(null);  // a stable client injected once
export function AuthProvider({client, children}: AuthProviderProps) {
  return <AuthContext value={client}>{children}</AuthContext>;  // React 19: the context is the provider
}
export function useAuth(): AuthClient {
  const client = useContext(AuthContext);
  if (client === null) throw new Error('useAuth needs <AuthProvider>');
  return client;
}
```

**Enforcement:** Review trigger — a Context whose value is a frequently-updated object (especially `useState` piped straight into a provider) should be a store; Context carries dependencies, not churn. New providers render `<Context value={…}>`; a new `<Context.Provider>` in fresh code is flagged for the React 19 form.

### 3.5 — Reach for Zustand only for genuinely global, dynamic client state.

**Reasoning, step by step:**
1. A store earns its place only when state is both *global* — read and written across unrelated parts of the tree — and *dynamic*, changing often during a session. Cross-cutting UI like a command palette, a toast queue, or a multi-step wizard shared across routes qualifies; a form's draft does not (3.1), and server data does not (3.3).
2. Model the store's state as a discriminated union with named transition actions, the discipline of 3.2 — actions are the only way the state moves, so legal transitions are written down and impossible ones absent. This is root rule "data and functions": the store is data plus the functions that transform it (3.7). Keep one store per domain; a single god-store is the global mutable bag the rest of this guide exists to prevent, while per-domain stores keep each state space small and its subscribers narrow.

```tsx
const usePalette = create<{  // state is a union; open/close are the only transitions (3.2)
  readonly state: {status: 'closed'} | {status: 'open'; query: string};
  open: () => void;
  close: () => void;
}>((set) => ({
  state: {status: 'closed'},
  open: () => set({state: {status: 'open', query: ''}}),
  close: () => set({state: {status: 'closed'}}),  // one store per domain; not a dumping ground
}));
```

**Enforcement:** Review gate — a new store must justify both *global* and *dynamic*; stores are per-domain, and state is a union mutated only through named actions.

### 3.6 — No Redux in new code.

**Reasoning, step by step:**
1. The real objection is architectural, not ergonomic. Redux Toolkit removed most of the classic boilerplate years ago — `createSlice` generates the action creators and reducers that were hand-written before — so "Redux means ceremony" is no longer the durable argument. What RTK does not change is the shape: even a lean RTK store pulls server cache, global client state, and incidental local state back into one global object, re-creating the god-store 3.5 exists to prevent. That concentration is the reason to avoid it.
2. The modern split keeps those concerns apart: server cache in Query (3.3), global client state in a per-domain store (3.5), local state in the component (3.1). Whatever boilerplate remains is the lesser, secondary cost on top of that. Existing Redux is not a rewrite mandate, though: legacy stores stay until there is reason to touch them, and when you do, record the decision in the migration ledger so the boundary between old and new is deliberate.

```tsx
// good — modern split: Zustand here, no useSelector/dispatch/connect; legacy slices stay with a ledger note
const value = useFeatureStore((s) => s.value);
```

**Enforcement:** Review rejects new Redux slices, actions, or `connect`; touching a legacy slice requires a ledger note recording why it was kept rather than migrated.

### 3.7 — Express state transitions as pure functions.

**Reasoning, step by step:**
1. Once a component's state is a small machine — more than a couple of independent transitions — scatter the `setState` calls across handlers and the legal transitions become impossible to see or test. Gather them into a `useReducer` whose reducer is a pure function `(state, action) => state`.
2. A pure transition function is testable without rendering: call `reducer(state, action)` in a unit test and assert on the returned state — no DOM, no act-warnings, no mounting. This is the root principle "data and functions, not objects": the machine is data, the transitions are functions beside it (see 3.5). Combined with 3.2, the reducer's state is a union and the function `switch`es on the action, so an unhandled action or illegal transition is a compile-time gap, not a runtime surprise.
3. When the machine *is* a form submission — pending, error, result threaded through one async transition — `useActionState` is the same `(state, action) => state` shape wired to a form Action, returning the next state plus an `isPending` flag for free ([react.dev: useActionState](https://react.dev/reference/react/useActionState), wired through form Actions in [04.6](./04-data-fetching-and-forms.md)). It complements this rule rather than replacing it: a pure `useReducer` still owns the synchronous client machine; `useActionState` owns the async, submission-shaped one.

```tsx
type Action = {type: 'edit'; draft: string} | {type: 'cancel'};
function reducer(state: View, action: Action): View {
  switch (action.type) {
    case 'edit': return {status: 'editing', draft: action.draft};
    case 'cancel': return {status: 'idle'};
    default: return assertNever(action);  // exhaustive — core 6.5
  }
}
```

**Enforcement:** Review trigger — three or more related `setState` calls become a `useReducer`; the reducer is pure and tested directly via `reducer(state, action)`, no render.

### 3.8 — Treat the URL as state.

**Reasoning, step by step:**
1. Filters, the active tab, pagination, sort order, and the selected entity are state — but state the user expects to share, bookmark, reload, and navigate with the back button. Putting them in the URL makes every one of those behaviors work for free, because the URL *is* the serialized, restorable form of that state.
2. Read this state from the router (search params or route segments) and write it by navigating. The browser's history stack becomes the undo log; reload restores the exact view because the state was never only in memory. Never duplicate router state into a store or `useState`: a copy reintroduces the sync problem of 3.3 — the URL and the store can disagree about the current tab — and breaks deep-linking, since the in-memory copy starts empty on a fresh load. The router is the single source for URL-shaped state.

```tsx
const [params, setParams] = useSearchParams();  // tab lives in the URL, not useState
const tab = params.get('tab') ?? 'profile';      // shareable, reload-safe, back/forward-correct
// a bare object would clobber sibling params; the updater preserves filters, pagination, sort
const setTab = (next: string) => setParams((prev) => { prev.set('tab', next); return prev; });
```

**Enforcement:** Review trigger — filters, tabs, pagination, or selection held in `useState`/a store that should survive reload and deep-linking belong in the URL; router state is never mirrored.

## Cross-references

- Making illegal states unrepresentable and exhaustive `switch`/`assertNever`: [../typescript/06-classes-and-data-modeling.md](../typescript/06-classes-and-data-modeling.md).
- Component body structure and where hooks are declared (1.7 ordering): [01-components-and-props.md](./01-components-and-props.md).
- The hooks themselves — `useState`/`useReducer`/`useContext`, dependency correctness, and custom-hook composition (3.7 builds on `useReducer`): [02-hooks.md](./02-hooks.md).
- Data fetching, mutations, and cache invalidation with TanStack Query: [04-data-fetching-and-forms.md](./04-data-fetching-and-forms.md).
