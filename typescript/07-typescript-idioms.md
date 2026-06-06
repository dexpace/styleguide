# 07 — TypeScript Idioms

TypeScript's surface is large, and much of it is sugar that earns its keep. Used deliberately, these idioms make the type the documentation and let the compiler carry the weight. Used carelessly, they produce code that looks expressive and reads like a riddle. This chapter is the runtime-neutral idiom layer: it assumes the type discipline of chapter 03 and the data-modeling stance of chapter 06, and it shows the shapes that follow from them.

## What good looks like

```ts
interface RankConfig {
  readonly weights: Record<string, number>;
  readonly minScore: number;
}
interface Candidate {
  readonly id: string;
  readonly score: number | null;
}

const config = { weights: { recency: 0.7 }, minScore: 0.5 } satisfies RankConfig;

function rank(raw: readonly Candidate[]): readonly Candidate[] {
  const scored = raw.map((c) => ({ ...c, score: c.score ?? 0 }));
  const eligible = scored.filter((c) => c.score >= config.minScore);
  const ranked = [...eligible].sort((a, b) => b.score - a.score);
  for (const c of ranked) {
    log.debug(`ranked ${c.id} at ${c.score}`); // effect loop, not a transform
  }
  return Object.freeze(ranked);
}
```

`config` uses `satisfies` (7.1): checked against `RankConfig` yet keeping the literal type, so `config.weights.recency` stays a known key. `score ?? 0` (7.3) treats only `null`/`undefined` as absent, never a real `0`. The pipeline names each stage — `scored`, `eligible`, `ranked` (7.6) — instead of one fused chain; the `sort` copies first because it mutates in place (7.10). The terminal `for…of` (7.5) runs effects, and `Object.freeze` returns an immutable result.

## Rules

### 7.1 — Use `satisfies` to validate a value without widening it.

**Reasoning, step by step:**
1. A type annotation (`const c: RankConfig = …`) checks the value but then *erases* what you wrote: the variable's type becomes `RankConfig`, so literal keys and narrow members are gone. A bare literal keeps full inference but is checked against nothing, so a typo or a wrong member slips through.
2. `satisfies` is the union of both: the compiler verifies the value conforms to the type, and the variable keeps its inferred literal type. Reach for it on config objects, lookup tables, and route maps — wherever you want the error *and* the precision.

```ts
const a: RankConfig = { weights: { recency: 1 }, minScore: 0 }; // safe, but keys widen to string
const b = { weights: { recency: 1 }, minScore: 0 } satisfies RankConfig; // safe AND b.weights.recency known
```

**Enforcement:** review; prefer `satisfies` over annotation for literal config — see [03-the-type-system.md](./03-the-type-system.md).

### 7.2 — Use `as const` to freeze a literal into its narrowest shape.

**Reasoning, step by step:**
1. By default a literal widens: `{ kind: "a" }` infers `{ kind: string }`, and `["a", "b"]` infers `string[]`. `as const` makes every member `readonly` and every literal its own type, so the value becomes a precise, immutable description of itself.
2. This is the sanctioned replacement for `enum` (banned, see 03): an `as const` object plus a `keyof typeof` / indexed-access derivation gives you named constants and a closed type that tracks the object, with zero runtime emit.

```ts
const Suit = { hearts: "H", spades: "S" } as const;
type SuitCode = (typeof Suit)[keyof typeof Suit]; // "H" | "S"
```

**Enforcement:** review; `as const` for fixed maps and tuples — see 03's enum-replacement idiom in [03-the-type-system.md](./03-the-type-system.md) and the literal-config rule 4.3 in [04-variables-and-declarations.md](./04-variables-and-declarations.md).

### 7.3 — Default with `??`; never default with `||`.

**Reasoning, step by step:**
1. `||` falls back on every falsy value: `0`, `''`, `false`, `NaN` — legitimate values, not the absence of one. `count || 10` silently rewrites a real `0` to `10`; `name || 'anon'` erases a deliberate empty string, and the bug is invisible until the value happens to be falsy.
2. `??` falls back only on `null` and `undefined` — the two things that actually mean "no value." If you genuinely want "falsy → fallback," that is a different operation; write the condition out so the intent is visible.

```ts
const retries = options.retries ?? 3; // 0 stays 0
const label = options.label ?? "untitled"; // "" stays ""
```

**Enforcement:** `@typescript-eslint/prefer-nullish-coalescing` (error).

### 7.4 — Reach for `?.`, but never chain past a lying type.

**Reasoning, step by step:**
1. `a?.b` short-circuits to `undefined` when `a` is nullish — one clean read of an optional step, replacing a nested `&&` guard. One `?.` answers a real question: this link may be absent.
2. Two or more `?.` in one expression (`a?.b?.c?.d`) is a signal, not a convenience: either the type is wider than reality, or the model is wrong. Parse the input into a known shape so the deep path becomes non-optional. Do not chain past the problem.

```ts
const bad = resp?.data?.user?.address?.city; // smell: 4-deep chain hides a model to parse
const city = parseUserOrThrow(resp).address.city; // fix: parse once, then the path is non-optional
```

**Enforcement:** review flags expressions with two or more `?.`; the fix is narrowing or parsing, see [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).

### 7.5 — Build transforms with `map`/`filter`/`reduce`; use `for…of` only for effects and early exit.

**Reasoning, step by step:**
1. A transform takes data and returns new data. `map`/`filter`/`reduce` say that directly and compose into a readable sequence of stages.
2. `for…of` is the right tool when the loop body performs an *effect* (I/O, logging, mutation of something external) or needs to `break`/`return` early — things the pipeline operators cannot express.
3. `.forEach` is discouraged: it cannot `await`, cannot `break`, and returns nothing, so it cannot sit mid-pipeline — it is a `for…of` with strictly fewer capabilities (ts.dev/style concurs). `for…in` is outright banned: it iterates inherited and string keys in unspecified order over an array's *indices*, not its values — a bug waiting for a prototype.

```ts
const names = users.filter((u) => u.active).map((u) => u.name); // transform
for (const u of users) await notify(u); // effect + await: for…of, never forEach/for…in
```

**Enforcement:** review; `no-restricted-syntax` bans `ForInStatement` and discourages `.forEach`.

### 7.6 — Name the steps once a chain passes about three stages.

**Reasoning, step by step:**
1. A short chain reads fine inline. Past roughly three stages it becomes one long expression with no place to rest the eye or hang a name.
2. Splitting into named intermediate `const`s makes each stage's *meaning* explicit. The names are the documentation, and concision serves clarity, never the reverse — a fused chain that saves three lines but costs a re-read is a net loss.

```ts
const out = rows.filter((r) => r.ok).map(norm).sort(byScore).slice(0, 10); // write-only
const normalized = rows.filter((r) => r.ok).map(norm); // the names ARE the pipeline
const topTen = [...normalized].sort(byScore).slice(0, 10);
```

**Enforcement:** review; chains beyond ~3 stages get named intermediates.

### 7.7 — Use `Map`/`Set` for dynamic keys; reserve objects for known shapes.

**Reasoning, step by step:**
1. An object is a *record*: a fixed set of named fields, each with its own type. A `Map` is a *collection*: arbitrary keys added and removed at runtime.
2. Using an object as a dynamic dictionary blurs the two — you lose `.size`, ordered iteration, and non-string keys, and inherit prototype-key hazards (`__proto__`). Using a `Map` for a known shape blurs it the other way — you lose the named, individually-typed fields that make a record self-documenting. Match the structure to the role: keys known at author time → object; keys discovered at runtime → `Map`/`Set`.

```ts
interface User { id: string; name: string } // known shape: object
const byId = new Map<string, User>(); // dynamic keys discovered at runtime: Map
```

**Enforcement:** review; `noUncheckedIndexedAccess` (chapter 01's six flags) makes every index-signature read `T | undefined`, friction that already pushes object-as-dictionary toward `Map`, see [03-the-type-system.md](./03-the-type-system.md).

### 7.8 — Clone with `structuredClone`, not a JSON round-trip.

**Reasoning, step by step:**
1. `JSON.parse(JSON.stringify(x))` was the old deep-copy trick, and it is lossy: it silently drops `undefined` and functions, and stringifies `Date`, `Map`, `Set`, and `RegExp` into garbage. The loss is invisible — the copy looks right until a `Date` field is now a string or an optional field has vanished.
2. `structuredClone(x)` is the built-in deep clone: it preserves `Date`, `Map`, `Set`, typed arrays, and cyclic references. It throws on truly non-cloneable values (functions, DOM nodes), which is correct — a clone that cannot be made should fail loudly, not lie.

```ts
const original = { at: new Date(), tags: new Set(["a"]), note: undefined };
const copy = structuredClone(original); // at is a Date, tags is a Set, note exists
```

**Enforcement:** review; `no-restricted-syntax` note flags `JSON.parse(JSON.stringify(…))`.

### 7.9 — Compose strings with template literals; type string contracts with template literal types.

**Reasoning, step by step:**
1. `` `${base}/${id}` `` reads in document order and cannot mismatch a `+` or coerce silently the way `base + "/" + id` can. In value space, template literals are the default for any string assembled from parts.
2. In type space, template literal types turn a string format into a *contract* the compiler checks: `` type Route = `/api/${string}` `` rejects any route that does not start with `/api/`. This pushes string-shape validation to compile time, where a malformed route is a type error instead of a 404.

```ts
type Route = `/api/${string}`;
const ok: Route = "/api/users"; // accepted; "users" would be a type error (no "/api/" prefix)
```

**Enforcement:** `prefer-template` (error); template literal types for string formats by review.

### 7.10 — Update immutably with spread; push nested updates into small named helpers.

**Reasoning, step by step:**
1. A shallow update is clean: `{ ...user, name }` returns a new object with one field changed and never mutates the original (rule 3, immutability). The clarity collapses with depth: `{ ...s, a: { ...s.a, b: { ...s.a.b, c } } }` is write-only — nobody can verify it at a glance, and one missing spread is a silent shared-reference bug.
2. Extract the nested update into a small named function: the name states the intent and confines the spreading to one verifiable place. Note that `.sort()` and `.reverse()` mutate the receiver, so spread first (`[...xs].sort()`) when the source must survive.

```ts
const next = { ...state, a: { ...state.a, b: { ...state.a.b, c: 1 } } }; // write-only
const setC = (s: State, c: number): State => ({ ...s, a: setB(s.a, c) }); // intent named, spread contained
```

**Enforcement:** review; deep spread chains become named helpers — see [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).

### 7.11 — Branch on a discriminant with an exhaustive `switch`, never an `if`-chain over `kind`.

**Reasoning, step by step:**
1. A discriminated union is a closed set of variants. A `switch` on the discriminant maps one `case` to each variant, flush and scannable. An `if`/`else if` chain over `event.kind` expresses the same logic with no exhaustiveness guarantee: add a variant and the chain silently falls through to nothing.
2. A `default: return assertNever(x)` arm makes the union closed at compile time — adding a variant without a `case` becomes a type error, not a runtime surprise. The compiler proving you handled every case is the whole point of modeling state as a union.

```ts
switch (s.kind) {
  case "circle": return Math.PI * s.r ** 2;
  case "square": return s.side ** 2;
  default: return assertNever(s); // type error if a variant is unhandled
}
```

**Enforcement:** `@typescript-eslint/switch-exhaustiveness-check` (error) plus `assertNever` — see [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).

### 7.12 — Write no clever one-liners; if it needs a second read, it needs a second line.

**Reasoning, step by step:**
1. A line packing a ternary inside a `reduce` inside a destructure is shorter to type and longer to understand. The reader pays the cost the writer saved. Clarity is the goal; brevity is only a means, and when they conflict brevity loses (README values: simplicity over expressiveness).
2. The test is mechanical: if a competent reader must read the line twice to be sure what it does, split it into named steps. Lines are free; re-reads are not.

```ts
const t = (u.tier ?? (u.vip ? "gold" : "std")).toUpperCase().slice(0, 3); // clever
const tier = u.tier ?? (u.vip ? "gold" : "std"); // second read avoided: each step named
const tag = tier.toUpperCase().slice(0, 3);
```

**Enforcement:** review; clarity over brevity is the tie-breaker, see README values.

## Cross-references

- `any` banned, `as` needs a why, branded primitives, `satisfies`/guards/parse: [03-the-type-system.md](./03-the-type-system.md); `const` default, `as const` config, non-null `!` banned: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Guard clauses, options object, one level of abstraction: [05-functions.md](./05-functions.md); discriminated unions, `assertNever`, illegal states unrepresentable: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `for…of` with `await` and `no-floating-promises`: [09-concurrency.md](./09-concurrency.md); V8 monomorphism, no `delete`, allocation hygiene of spread and clone: [15-performance.md](./15-performance.md).
