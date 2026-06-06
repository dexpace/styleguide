# 04 — Data Fetching and Forms

The two places a React app meets the outside world: reading server state, and writing user input back. Both are external-system boundaries, and the discipline is the core guide's at every wire edge — parse the `unknown`, type from the schema, thread the signal, name the failure. This chapter routes all server reads and writes through TanStack Query (REACT-3), drives mutation-shaped forms with React 19's form Actions and `useActionState`, and reaches for react-hook-form on the rich multi-field validation surfaces — so caching, cancellation, invalidation, pending state, and validation are framework and platform concerns, not per-component improvisation — REACT-3, server state is not client state, made operational. The zod boundary ([core 10.7](../typescript/10-api-design.md)), signal-threaded cancellation ([core 9.5](../typescript/09-concurrency.md)), and the view-union from state management ([03.2](./03-state-management.md)) are the raw material; here they compose into the data layer.

## What good looks like

```tsx
// booking-api.ts — keys, schema, and typed fetchers for one resource.
export const bookingKeys = {
  all: ['bookings'] as const,                                  // 4.1 the cache's schema, in one place
  detail: (id: BookingId) => [...bookingKeys.all, 'detail', id] as const,
};
const BookingSchema = z.object({id: z.string(), seats: z.number().int()}).readonly();
type Booking = z.infer<typeof BookingSchema>;                  // 4.2 type follows schema, flows inward (core 10.7)
const RebookSchema = z.object({seats: z.number().int().positive()}).readonly();
type RebookInput = z.infer<typeof RebookSchema>;              // 4.6 one schema types the form values

export const useBooking = (id: BookingId) =>
  useQuery({
    queryKey: bookingKeys.detail(id),
    queryFn: async ({signal}) => {                            // 4.3 TanStack supplies the signal
      const res = await fetch(`/api/bookings/${id}`, {signal}); // threaded to fetch (core 9.5)
      if (!res.ok) throw new Error(`booking ${id}: HTTP ${res.status}`);
      return BookingSchema.parse(await res.json());           // 4.2 parse the unknown at the boundary
    },
  });

// RebookForm.tsx — a mutation-shaped form: a React 19 Action drives the rebook.
export function RebookForm({id}: {readonly id: BookingId}) {
  const qc = useQueryClient();
  const rebook = useMutation({
    mutationFn: (input: RebookInput) => postRebook(id, input),
    onSuccess: () => qc.invalidateQueries({queryKey: bookingKeys.detail(id)}), // 4.4 declared scope
  });
  const [state, formAction, isPending] = useActionState(async (_prev: {error?: string}, data: FormData) => {
    const parsed = RebookSchema.safeParse({seats: Number(data.get('seats'))}); // 4.6 same schema parses input
    if (!parsed.success) return {error: parsed.error.issues[0]?.message};
    await rebook.mutateAsync(parsed.data);                     // 4.4 invalidation fires on success
    return {};
  }, {});
  return (
    <form action={formAction}>                                 {/* 4.6 React 19 form Action */}
      <input type="number" name="seats" />
      {state.error && <p role="alert">{state.error}</p>}
      <button disabled={isPending}>Rebook</button>             {/* 4.6 isPending from useActionState */}
    </form>
  );
}
```

The key comes from a factory, never a literal (4.1); the response is zod-parsed and the domain type is `z.infer`, not hand-written (4.2, [core 10.7](../typescript/10-api-design.md)); `signal` flows from `queryFn` into `fetch` so navigation away cancels the request (4.3, [core 9.5](../typescript/09-concurrency.md)); the mutation invalidates exactly the detail it changed (4.4). The form is a React 19 Action: `<form action={formAction}>` over `useActionState` gives `isPending` and the returned error state for free (4.6), the one `RebookSchema` parses the input there just as it does at the wire (4.2) — a rich multi-field validation UI would instead reach for react-hook-form over the same schema (4.6).

## Rules

### 4.1 — Query keys come from a key factory, never string literals.

**Reasoning, step by step:**
1. A query key *is* the cache's primary key: it identifies the entry to read, write, and invalidate. Scattering string-literal keys — `['booking', id]` in the hook, `['bookings', id]` in the invalidation — is two spellings of one identity, and they drift the instant one is edited and the other forgotten, so the mutation invalidates a key nothing is stored under and the view goes stale silently.
2. So centralize keys in a per-resource factory object — `bookingKeys.all`, `bookingKeys.detail(id)` — built with `as const` so the tuples are literal types. Every read and every invalidation imports the same function; renaming the shape is one edit, and a typo is a compile error, not a missed cache hit.
3. Nest keys so invalidation can be coarse or fine: `detail(id)` extends `all`, so `invalidateQueries({queryKey: bookingKeys.all})` clears every booking query while `detail(id)` clears one. The hierarchy is the factory's job, declared once, not reconstructed at each call site.

```tsx
useQuery({queryKey: ['booking', id], queryFn});                 // bad — literal; the invalidator must guess this spelling
useQuery({queryKey: bookingKeys.detail(id), queryFn});         // good — one schema for the cache, greppable, typo-proof
```
**Enforcement:** review; query keys are factory calls, never inline array literals; `@tanstack/eslint-plugin-query` flags structural mistakes.

### 4.2 — Parse every response with zod at the fetch boundary.

**Reasoning, step by step:**
1. The server is an external system, and its JSON arrives as `unknown` ([core 3.2](../typescript/03-the-type-system.md)) no matter what the OpenAPI doc claims — a renamed field, a null where a number was promised, a shape from last deploy all type-check as `any` and detonate three components deep. The `queryFn` is the boundary; the data must be parsed into a domain type there, before any component touches it.
2. Parse with a zod schema and derive the type from it — `type Booking = z.infer<typeof BookingSchema>` — exactly as the core guide mandates at every wire edge ([core 10.7](../typescript/10-api-design.md)). The schema is the single source of truth; a hand-written `interface Booking` beside it is a second truth that drifts. Call `.readonly()` so the parsed value is frozen and the inferred type is `readonly` from birth.
3. Never `as`-cast a response into shape. `(await res.json()) as Booking` is a lie the compiler believes and the runtime disproves — it asserts a shape instead of checking one, so the bug surfaces as a crash in render rather than a caught parse error at the seam. The cast moves the failure away from its cause; the parse keeps it at the boundary.

```tsx
return (await res.json()) as Booking;                          // bad — asserted, not verified; drift crashes downstream
return BookingSchema.parse(await res.json());                  // good — verified at the boundary, type is z.infer
```
**Enforcement:** review; every `queryFn`/`mutationFn` parses its response through a schema; `as`-cast responses rejected (core 10.7).

### 4.3 — Query functions accept and thread the AbortSignal.

**Reasoning, step by step:**
1. TanStack Query passes an `AbortSignal` into every `queryFn` through its context argument, and aborts it when the query is no longer needed — the component unmounts, the key changes, the user navigates away mid-flight. That signal is only useful if you take it; ignore it and the request runs to completion with nowhere to deliver, wasting a connection and racing a newer query.
2. So destructure `{signal}` from the `queryFn` argument and pass it straight to `fetch` (or your client), continuing the core guide's rule that cancellation flows down to the I/O primitive ([core 9.5](../typescript/09-concurrency.md), 9.11). The signal accepted at the top must reach the actual wait, or it is decoration.
3. This makes navigation cancel in-flight work for free: route away from a detail page and its load aborts, so a slow response cannot resolve into an unmounted tree or clobber the page you moved to. The framework owns the lifetime; honoring its signal is how you cooperate.

```tsx
queryFn: async () => fetch(`/api/bookings/${id}`),             // bad — signal dropped; request outlives the component
queryFn: async ({signal}) => fetch(`/api/bookings/${id}`, {signal}), // good — cancels on unmount / key change
```
**Enforcement:** review; every `queryFn` that performs I/O threads the supplied `signal` to the underlying call (core 9.5).

### 4.4 — Mutations declare exactly which queries they invalidate.

**Reasoning, step by step:**
1. A mutation changes server state, which makes some cached queries stale — and the framework cannot know which. Leaving them stale is a correctness bug: the user rebooks, the seat count on screen still shows the old value, and they distrust the app. So every mutation must, on success, invalidate the queries its write affected.
2. State that scope explicitly and narrowly with the key factory (4.1): `invalidateQueries({queryKey: bookingKeys.detail(id)})` in `onSuccess` refetches the one view that changed. The set of affected keys is knowable from what the mutation wrote; name it, do not approximate it.
3. The two failure modes are opposite and both real. Invalidating nothing leaves stale views — a correctness bug. Invalidating everything (`queryClient.invalidateQueries()` bare) refetches the whole cache on every write — a performance bug that scales with how much the app has loaded. The right scope is the keys the write touched, neither less nor more.

```tsx
onSuccess: () => qc.invalidateQueries(),                       // bad — refetches the entire cache on every write
onSuccess: () => qc.invalidateQueries({queryKey: bookingKeys.detail(id)}), // good — exactly the affected view
```
**Enforcement:** review; every mutation declares a scoped invalidation; bare `invalidateQueries()` over the whole cache is rejected.

### 4.5 — Optimistic updates carry their rollback; reach for the layer that owns the optimism.

**Reasoning, step by step:**
1. Optimism shows the expected result before the server confirms, so the UI feels instant — but the server can still reject, and an optimistic value left standing after a rejection shows a change that never happened. The discipline is the same in both layers: the optimistic value must be undone if the write fails. Which layer you reach for depends on *what is optimistic*.
2. For UI-local optimism — the row you just submitted appearing in a list, the name updating beside the form — lead with `useOptimistic`. It owns the *view* layer: you derive an optimistic state from the real value plus the pending action, and React reverts it automatically when the Action settles or errors, with no snapshot to manage ([react.dev: useOptimistic](https://react.dev/reference/react/useOptimistic)). The rollback is the platform's, not yours.
3. For optimism that must persist in the *cache* — so every consumer of the query key, not just this component, sees the pending write — that lives in TanStack's `onMutate`/`onError`/`onSettled`, the cache layer's own rollback. In `onMutate`: cancel in-flight queries for the key (so a settling refetch cannot overwrite the optimistic write), snapshot the current value with `getQueryData`, write the optimistic value, and return the snapshot as context. In `onError`: restore the snapshot with `setQueryData`. In `onSettled`: invalidate (4.4) so the cache reconciles with the server's truth either way. The two are complementary, not rival: `useOptimistic` owns the component's view, `onMutate` owns the shared cache entry.
4. If you cannot write the rollback in whichever layer you chose, do not do the optimistic update — show a pending state (4.8) and wait for the server. A non-optimistic mutation that is always correct beats an optimistic one that is sometimes a lie.

```tsx
// view layer (useOptimistic): React reverts on settle/error — no manual snapshot (react.dev)
const [shownName, addOptimistic] = useOptimistic(name, (_prev, next: string) => next);
const submit = (formData: FormData) => { addOptimistic(String(formData.get('name'))); return rename.mutateAsync(formData); };

// cache layer (TanStack onMutate): persists the pending write for every consumer of the key
onMutate: async (next) => {
  await qc.cancelQueries({queryKey: bookingKeys.detail(id)});  // stop a refetch from clobbering the optimistic write
  const previous = qc.getQueryData(bookingKeys.detail(id));    // snapshot for rollback
  qc.setQueryData(bookingKeys.detail(id), previous ? {...previous, ...next} : previous); // merge, keep id
  return {previous};
},
onError: (_e, _next, ctx) => qc.setQueryData(bookingKeys.detail(id), ctx?.previous), // restore on failure
```
**Enforcement:** review; UI-local optimism uses `useOptimistic` (the platform reverts it); a cache-level `onMutate` optimistic write has a matching `onError` rollback from snapshotted context and an `onSettled` invalidation.

### 4.6 — Mutation-shaped forms use form Actions + `useActionState`; complex validation UIs use react-hook-form. One zod schema, either way.

**Reasoning, step by step:**
1. The constant across both paths is the schema. A form has two needs that usually drift apart: validating what the user typed, and typing the values the submit handler receives. Write them separately — a manual `validate` beside a hand-written `FormValues` interface — and they disagree the moment one changes. One zod schema serves both: it validates, and `z.infer<typeof Schema>` is the value type. Single source, no drift, identical to the wire-boundary discipline (4.2). The schema parses the input in *both* paths below; the choice between them is about the form's UI machinery, never about whether validation is zod's job.
2. For a mutation-shaped form — submit, await the write, show pending and error — lead with a form Action. Pass the function as `<form action={fn}>` and wrap it in `useActionState`, which returns `[state, formAction, isPending]`: the pending flag and the returned error state are built in, so there is no hand-rolled `isSubmitting` or `submitError` ([react.dev: useActionState](https://react.dev/reference/react/useActionState), [form actions](https://react.dev/blog/2024/12/05/react-19#actions)). The action receives the `FormData`, parses it with the same `Schema.safeParse`, and returns either the field errors or the result — React auto-wraps the submission in a Transition and resets the form on success.
3. For a complex multi-field validation UI — cross-field rules, per-keystroke feedback, arrays of fields, focus-on-error — react-hook-form still out-delivers the platform. `zodResolver(Schema)` validates and `z.infer` types `useForm<…>`; uncontrolled inputs via `register` mean typing in a thirty-field form costs one input's render, not thirty (reach for `Controller` only to wrap a library input that cannot take a `ref`). This is the boundary, stated honestly: the action path owns the submit-and-mutate shape; RHF owns the rich validation surface. Both validate with the one schema.
4. Validation errors render keyed by field and surface to assistive tech with `role="alert"` (REACT-4, cross-ref [07 accessibility](./07-accessibility.md)) — from `formState.errors` under RHF, from the action's returned `state` under `useActionState`. The schema's messages are the form's messages, so there is no second list of error strings to maintain in either path.

```tsx
// mutation-shaped form: form Action + useActionState — pending/error are built in (react.dev)
const [state, formAction, isPending] = useActionState(async (_prev: FormState, data: FormData) => {
  const parsed = RebookSchema.safeParse({seats: Number(data.get('seats'))}); // same schema parses here
  if (!parsed.success) return {errors: z.flattenError(parsed.error).fieldErrors};
  await postRebook(id, parsed.data);
  return {errors: {}};
}, {errors: {}});
<form action={formAction}>
  <input type="number" name="seats" />
  {state.errors.seats && <p role="alert">{state.errors.seats[0]}</p>}
  <button disabled={isPending}>Rebook</button>                {/* isPending from the hook, not a flag */}
</form>
// complex multi-field validation UI: react-hook-form over the same schema
// const {register, formState} = useForm<RebookInput>({resolver: zodResolver(RebookSchema)});
```
**Enforcement:** review; mutation-shaped forms use `<form action>` + `useActionState` (pending/error from the hook, no hand-rolled flags); complex validation UIs use `zodResolver`; either way one zod schema both validates and types, and per-field `useState` for inputs is rejected.

### 4.7 — Wrap each route or feature in an error boundary.

**Reasoning, step by step:**
1. A render-time throw with no boundary above it unmounts the *entire* React tree — one widget's bad data blanks the whole page to nothing. A thrown query error (4.2's parse failure, an HTTP throw) is exactly such a throw. The blast radius of a failure must be bounded to the feature that failed, not the application.
2. So place an error boundary at each route and around each independently-failing feature. A crash in the recommendations panel renders that panel's fallback; the booking form beside it keeps working. The boundary is where you decide what a failure looks like locally — an inline retry, a degraded placeholder — instead of a white screen globally.
3. Pair the boundary with TanStack's `QueryErrorResetBoundary` and `useQueryErrorResetBoundary` so a "Try again" button both resets the boundary and refetches the failed query, rather than just re-rendering the same cached error. Set `throwOnError: true` (or a predicate `throwOnError: (error) => …` to forward only some failures) on queries whose errors should reach the boundary, so error UX is structural, not a flag checked in every component.

```tsx
<QueryErrorResetBoundary>
  {({reset}) => (
    <ErrorBoundary onReset={reset} fallbackRender={({resetErrorBoundary}) =>
      <button onClick={resetErrorBoundary}>Try again</button>}>
      <BookingPanel id={id} />                                  {/* its crash stays here; the page survives */}
    </ErrorBoundary>
  )}
</QueryErrorResetBoundary>
```
**Enforcement:** review; every route and independently-failing feature sits under an error boundary; query-driven boundaries pair with `QueryErrorResetBoundary`.

### 4.8 — Derive loading UX from query state, never from bespoke flags.

**Reasoning, step by step:**
1. TanStack Query already models the lifecycle as a discriminated union — `isPending`, `isError`, `isSuccess` with the narrowed `data`/`error`. Re-deriving it into hand-rolled `const [loading, setLoading] = useState(false)` flags duplicates state the cache already owns, and the duplicate desynchronizes: a `setLoading(false)` is forgotten on an error path and the spinner spins forever. Loading is derivable from the query during render, so derive it (REACT-1, REACT-5); a flag mirrored by an effect is exactly the second source of truth REACT-5 forbids. Read the query's state; do not shadow it.
2. Branch on that union the same way state management branches on its status union ([03.2](./03-state-management.md)): `if (query.isPending) return <Skeleton/>; if (query.isError) return <Error/>;` and below those guards `query.data` is defined, so the success path needs no `?.` and no non-null assertion. The union makes the impossible state — pending *and* error at once — unrepresentable, and TypeScript narrows `data` for free.
3. Where the eventual layout is known, render a skeleton shaped like the content, not a centered spinner: the skeleton reserves the final dimensions so content does not jump in (no layout shift, cross-ref [08 performance](./08-react-performance.md)), and it tells the user *what* is loading. A spinner is the fallback only when the shape is genuinely unknown.

```tsx
if (booking.isPending) return <BookingSkeleton />;             // shape of the content, not a spinner
if (booking.isError) return <BookingError error={booking.error} />;
return <BookingDetail booking={booking.data} />;              // data is defined here — union narrowed it
```
**Enforcement:** review; views branch on `isPending`/`isError`/`isSuccess`; parallel `useState` loading/error flags shadowing a query are rejected (cross-ref 03.2).

## Cross-references

- Discriminated-union view state, `status` over boolean soup, server state belongs to TanStack Query: [03-state-management.md](./03-state-management.md).
- Thin route components and lazy boundaries where loading UX lands: [05-structure-and-routing.md](./05-structure-and-routing.md); skeletons and layout-shift cost: [08-react-performance.md](./08-react-performance.md). `role="alert"` for form errors and accessible loading announcements: [07-accessibility.md](./07-accessibility.md).
- zod at the wire boundary, `z.infer` as the single source of type truth, `unknown` then parse: [core 10.7](../typescript/10-api-design.md) and [core 3.2](../typescript/03-the-type-system.md).
- `{ signal }` cancellation accepted at the top and threaded to the I/O primitive: [core 9.5](../typescript/09-concurrency.md) and 9.11.
