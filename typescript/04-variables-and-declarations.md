# 04 — Variables and Declarations

How values come into being and how long they live. The type system protects only what you let it see; a `let` you never reassign, a `!` that papers over a gap, a widened literal each throw away information the compiler had.

## What good looks like

```ts
// retry.config.ts — zero mutable module state, every fact derivable
import {invariant} from './invariant.js';
export const RETRY = {
  maxAttempts: 5,
  baseDelayMs: 100,
  backoff: 'exponential',
} as const;
// Types derived from the single source of truth, never declared twice.
export type Backoff = (typeof RETRY)['backoff']; // 'exponential', not string
export type RetryConfig = typeof RETRY;
export type RetryKey = keyof typeof RETRY; // 'maxAttempts' | 'baseDelayMs' | 'backoff'
const ENV_ATTEMPTS = process.env['RETRY_MAX_ATTEMPTS']; // one external input, read once
/** Resolve attempts from the environment, else the literal default. */
export function resolveAttempts(): number {
  if (ENV_ATTEMPTS === undefined) return RETRY.maxAttempts;
  const parsed = Number.parseInt(ENV_ATTEMPTS, 10);
  invariant(Number.isInteger(parsed) && parsed > 0, `bad RETRY_MAX_ATTEMPTS: ${ENV_ATTEMPTS}`);
  return parsed; // precondition checked, fails loud
}
/** Read one config field by key; the generic returns the exact per-key type, which can't drift from RETRY. */
export function fieldOf<K extends RetryKey>(key: K): RetryConfig[K] {
  return RETRY[key]; // 4.3 — keyof typeof keeps callers honest, no `!`
}
```

Every binding is `const` (4.1); nothing leans on `!` (4.2); `as const` keeps `backoff` as `'exponential'` not `string` (4.3); names are declared at first use, one per line (4.4); the derived types can't drift from the literal; and the module holds no mutable state, reading its one external input into a `const` (4.7).

## Rules

### 4.1 — Default to `const`; spend a `let` only when the reader can see why; never `var`.

**Reasoning, step by step:**
1. A `const` is a fact: after its line the binding never changes, so the reader carries one value forward instead of re-deriving it at each use. A `let` asks "what is this now?" and makes the reader scan ahead for the reassignment.
2. Spend a `let` only when reassignment is the point: an accumulator no pipeline expresses, or a value built across branches. `var` is strictly worse than either: function-scoped and hoisted, it leaks out of its block and reads as defined before its line, buying nothing `let` doesn't.

```ts
let total = sum(items);   // bad — let, never reassigned
const total = sum(items); // good — a fact; let only when a branch/loop reassigns it
```

**Enforcement:** `no-var` and `prefer-const` (gts baseline), errors in CI.

### 4.2 — The non-null assertion `!` is banned outside tests and declared bridges.

**Reasoning, step by step:**
1. `value!` is an unchecked claim that this isn't `null` or `undefined`. It erases the case the type told you to handle, silently, with no message when wrong.
2. Every `!` is one of two things: a *missing model*, where the value really is always present and the type should say so; or a *latent bug*, where it can be absent and the `!` will one day dereference nothing. The `!` fixes neither and hides both.
3. When the value is genuinely present, prove it. `invariant(x !== undefined, '…')` narrows `x` for the compiler and fails loud with a message if you were wrong, the trade the Kotlin guide makes banning `!!` for `requireNotNull` (kotlin/03 §3.2). The narrow exceptions are test setup with a known-configured fixture, and a declared bridge to untyped code carrying a `// bridge:` comment; otherwise `?.` and `??` handle absence, where `!` only denies it.

```ts
const port = config.get('port')!; // bad — missing model or latent bug, silent on failure
const port = config.get('port'); invariant(port !== undefined, 'port missing'); // good — narrow with a message
```

**Enforcement:** `@typescript-eslint/no-non-null-assertion`, error in CI; overridden only in `*.test.ts` and on `// bridge:` lines.

### 4.3 — Write `as const` on literal configuration.

**Reasoning, step by step:**
1. Without it the compiler widens silently and lossily: `{kind: 'retry'}` becomes `{kind: string}` and `[1, 2, 3]` becomes `number[]`. A discriminant widened to `string` can no longer drive a discriminated union, and a tuple widened to an array forgets its length and member positions.
2. `as const` freezes the literal to its narrowest type, so it serves as the single source for derived types (`typeof CONFIG`, `keyof typeof CONFIG`) instead of a second declaration that drifts. It is the right tool, not the banned `as SomeType` (chapter 03): `as const` asserts the narrowest type the value already has, where `as SomeType` overrides the inferred type with a different one.

```ts
const ROUTES = {list: {method: 'GET'}};            // bad — method widens to string
const ROUTES = {list: {method: 'GET'}} as const;   // good — method stays 'GET', keys known
```

**Enforcement:** review; its absence surfaces as widened types at the use site.

### 4.4 — Declare at first use, one declaration per line.

**Reasoning, step by step:**
1. A declaration at the line it is first needed documents the flow: the reader meets each name when it becomes relevant, and its live range stays short. Names hoisted to the top force the reader to hold a bag of valueless names and match each to its use below, the C-era habit block scoping was built to retire.
2. One name per line keeps diffs and `git blame` honest and leaves room for the annotation each name deserves. `const {a, b} = obj` is one declaration; destructuring (4.6) introduces several names at once.

```ts
let user, total;                 // bad — hoisted, comma-chained, live before they mean anything
const user = await loadUser(id); // good — each name declared, one per line, where first used
```

**Enforcement:** `one-var` (one declaration per statement); live-range tightening is a review concern.

### 4.5 — No chained assignment.

**Reasoning, step by step:**
1. `a = b = c` binds two names from one expression and reads ambiguously: it is right-associative, so `c` is assigned to `b` and that result to `a`, an order most readers reconstruct slowly. It also hides the write to `b`, easy to miss while the eye tracks the leftmost target, exactly the invisible state change 4.1 removes.
2. With `const` as the default it barely arises, since `const` can't be chained, and where it tempts you two named `const`s say the same thing in a readable order.

```ts
cursor = node = head; // bad — right-to-left, with a silent write to node in the middle
const node = head;    // good — one plain const per line (4.4); no chain, no hidden write
```

**Enforcement:** `no-multi-assign`, error in CI.

### 4.6 — Destructure at boundaries with defaults; at most two levels deep.

**Reasoning, step by step:**
1. Destructuring earns its place at a boundary, such as parameters or a parse result, where it names the fields you accept and shows the accepted shape in the signature. Apply defaults in the same pattern: `{timeout = 5_000} = options` states the fallback next to the name it guards, making it part of the contract rather than a `??` buried in the body.
2. Past two levels the pattern hides instead of documents: `const {a: {b: {c}}} = x` buries the shape of `x` in punctuation and couples the call site to three levels that can each change. Reaching a third level, destructure the intermediate on its own line.

```ts
const {data: {user: {email}}} = resp;  // bad — three levels, shape lost in brackets
function connect({host, port = 5432}: Opts): Pool { /* ... */ } // good — default at boundary
```

**Enforcement:** review; depth and default placement are read at the call site.

### 4.7 — No module-level mutable state.

**Reasoning, step by step:**
1. Module scope is global scope wearing a jacket. A `let` or mutable object at the top of a file is shared by every importer for the life of the process, with behaviour depending on which module mutated it first and on import order the bundler may change. It also poisons tests, since state carries between cases in the same process, killing the determinism chapter 11 demands.
2. Put state where it can be owned: a class instance you construct and dispose, a store passed as a parameter, or a value threaded through arguments. A module-level `const` holding a frozen value is fine; the mutability is banned, not the scope.

```ts
let requestCount = 0; // bad — mutable global; importers share it, tests bleed into each other
export class RequestMeter { private count = 0; } // good — state owned by an instance with a lifetime
```

**Enforcement:** review (a top-level `let`/`var` export is the tell); reinforced by the import-side-effect ban in chapter 12.

### 4.8 — No shadowing.

**Reasoning, step by step:**
1. Shadowing gives one name two meanings in one screen, so the reader must track which `id` each later line means, and the answer changes by scope depth. The trap is silent: a refactor that moves a line across the boundary, or deletes the inner binding, flips every reference to the other variable with no error, still compiling but now meaning something else.
2. The fix is a name, not a workaround: if the inner value differs it deserves a distinct name (`rawId` versus `id`); if it is the same value the inner declaration is redundant. The hazard includes outer scopes, imports, and built-ins, so a local `name` or `length` is the same trap with a longer fuse.

```ts
for (const user of user.delegates) audit(user);     // bad — inner user shadows the param
for (const delegate of user.delegates) audit(delegate); // good — distinct names
```

**Enforcement:** `@typescript-eslint/no-shadow` (the typescript-eslint variant, which understands type and enum scopes), error in CI.

## Cross-references

- `readonly`, branded primitives, why `as SomeType` needs a justification: [03-the-type-system.md](./03-the-type-system.md). `invariant(): asserts`, guard clauses, assertion density: [05-functions.md](./05-functions.md). Classes for owned mutable lifecycle state: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md). `as const` maps over `enum`, `?.`/`??` over `||`: [07-typescript-idioms.md](./07-typescript-idioms.md).
- Determinism that module state breaks: [11-testing.md](./11-testing.md). No side effects on import: [12-module-organization.md](./12-module-organization.md). Stable shapes for derived literals: [15-performance.md](./15-performance.md).
