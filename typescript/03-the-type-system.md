# 03 — The Type System

The type system is the first test suite, and it runs on every keystroke. This chapter is about not lying to it: every rule below closes a hole through which an untyped, unproven, or unchecked value could reach your domain logic. Get the boundary right and the interior takes care of itself.

## What good looks like

```ts
import {z} from 'zod';

// A branded primitive: a `string` the compiler refuses to confuse with any other string (3.9).
type UserId = string & {readonly __brand: 'UserId'};
const UserIdSchema = z.uuid().transform((s): UserId => s as UserId); // sanctioned `as`
const UserSchema = z.object({
  id: UserIdSchema,
  email: z.email(),
  roles: z.array(z.enum(['admin', 'member'])).readonly(),
});
type User = Readonly<z.infer<typeof UserSchema>>; // inferred from the schema: one source of truth
type ParseResult =
  | {readonly kind: 'ok'; readonly user: User}
  | {readonly kind: 'parse-error'; readonly issues: ReadonlyArray<string>};
export function parseUser(raw: unknown): ParseResult {
  const result = UserSchema.safeParse(raw);
  if (!result.success) {
    return {kind: 'parse-error', issues: result.error.issues.map(i => i.message)};
  }
  return {kind: 'ok', user: result.data};
}
```

The module takes `unknown` and hands back a discriminated union, never a throw and never an `any` (3.2). The branded `UserId` (3.9) cannot be swapped for a bare string. The lone `as` is confined to the schema's `transform`, where the value is already proven, and it carries a why-comment (3.4). `roles` is a literal union, not an `enum` (3.12), and every public surface is `readonly` (3.10). Absence, were any field optional, would be `undefined` (3.5), not `null`.

## Rules

### 3.1 — Treat the strict flag family as law.

**Reasoning, step by step:**
1. `strict: true` is the baseline; on top of it the tsconfig adds six flags (chapter [01](./01-formatting-and-tooling.md)), and each closes a specific hole. `noUncheckedIndexedAccess`: `arr[i]` returns `T | undefined`, forcing the bounds check Tiger Style demands. `exactOptionalPropertyTypes`: `{a?: number}` no longer accepts `{a: undefined}`, so absent and present-but-undefined stay distinct. `noImplicitOverride`: overriding without the `override` keyword is an error.
2. `isolatedModules` and `verbatimModuleSyntax`: every file transpiles in isolation and `import type` is explicit, which is what esbuild and SWC require. `erasableSyntaxOnly`: the subject of 3.12. None of these are knobs to soften per-project; a weakened flag is a deviation argued in the ledger, not a local convenience.

**Worked example:**
```ts
const name = items[0].name;             // error: Object is possibly 'undefined'
const safe = items[0]?.name ?? 'none';  // good — noUncheckedIndexedAccess forces the check
```
**Enforcement:** `tsconfig.json` `strict` plus the six flags; see chapter [01](./01-formatting-and-tooling.md).

### 3.2 — Ban `any`; accept `unknown` and narrow inward.

**Reasoning, step by step:**
1. `any` is not a type, it is the absence of one. It disables every check on the value *and on everything derived from it*, so one `any` quietly infects a call chain.
2. `unknown` is the honest top type: it holds anything, but the compiler forbids every operation until you prove what it is, and that proof is narrowing (3.7). So external input (`JSON.parse`, `fetch`, payloads) should be *received as* `unknown` — annotate the boundary variable, because `JSON.parse` and `res.json()` hand back `any`, which `no-unsafe-assignment` then flags — so it stays `unknown` at the boundary and is parsed into a domain type (3.4, 3.9) before the interior ever sees it.

**Worked example:**
```ts
function handle(raw: any): string {return raw.user.naem;}          // bad — typo never caught; any spreads
// parseUserOrThrow(raw: unknown): User — parses, returning User or throwing on invalid input.
function handle(raw: unknown): User {return parseUserOrThrow(raw);} // good — unknown forces a parse
```
**Enforcement:** `@typescript-eslint/no-explicit-any`, plus the `no-unsafe-*` family (`no-unsafe-assignment`, `no-unsafe-member-access`, `no-unsafe-call`, `no-unsafe-return`, `no-unsafe-argument`) from `strict-type-checked`.

### 3.3 — Ban `@ts-ignore`; allow `@ts-expect-error` only with a reason, only in tests and declared bridges.

**Reasoning, step by step:**
1. `@ts-ignore` silences whatever error sits on the next line, including a *new* one a later edit introduces; it rots silently. `@ts-expect-error` is the honest cousin: it is itself an error when the next line has none, so removing the underlying problem also flags the now-stale suppression.
2. Permit it only where suppression is legitimate: a test exercising a deliberate type failure, or a declared bridge to untyped third-party code. Both require an inline reason string.

**Worked example:**
```ts
// in a *.test.ts, asserting misuse is rejected:
// @ts-expect-error — a number where UserId is required must not compile
parseUser(42);
```
**Enforcement:** `@typescript-eslint/ban-ts-comment` with `ts-ignore: true` and `ts-expect-error: 'allow-with-description'`.

### 3.4 — Require a why-comment on every type assertion (`as`).

**Reasoning, step by step:**
1. An assertion is an unproven claim to the compiler: "trust me, this is a `T`." The compiler stops checking and believes you. When you are wrong, the failure surfaces far from the lie.
2. Reach for the proven alternatives first: `satisfies` checks a value against a type *without* widening it, and a guard or a parse (zod) establishes the type with a runtime check the compiler can see. When an assertion is genuinely unavoidable, such as generating a brand after validation (3.9) or narrowing a value the compiler cannot follow, it carries a comment stating why the claim holds.

**Worked example:**
```ts
const bad = {timeout: 30} as Config;                    // bad — as hides the missing `retries`
const good = {timeout: 30, retries: 3} satisfies Config; // good — checked, type preserved
```
**Enforcement:** `@typescript-eslint/consistent-type-assertions`; assertions reviewed for an accompanying why-comment.

### 3.5 — Represent absence as `undefined`; let `null` in only where an external contract forces it, and convert at the boundary.

**Reasoning, step by step:**
1. JavaScript has two empties and TypeScript inherits both; two absence values double the cases every narrowing must handle (`x == null` versus `x === undefined`). Pick one. `undefined` is the language's native absence: a missing property, an unset variable, and a `void` return all already are it.
2. Some external contracts (JSON APIs, database drivers, the DOM) emit `null`. Convert `null` to `undefined` at the boundary, the same place you parse, so the interior sees exactly one absence value.

**Worked example:**
```ts
const middleName = dto.middleName ?? undefined; // null converted once, here; the interior speaks undefined
```
**Enforcement:** review at boundary modules; `??` conversion is the idiom (chapter [07](./07-typescript-idioms.md)).

### 3.6 — Prefer optional `?` over `| undefined` in object types.

**Reasoning, step by step:**
1. `{name?: string}` and `{name: string | undefined}` look interchangeable but are not. The first lets the key be absent; the second *requires* the key, with the value possibly `undefined`.
2. Under `exactOptionalPropertyTypes` (3.1) the difference is enforced: `{name: undefined}` will not satisfy the optional form, which is exactly right, since absent means absent. Use `?` for "this field may not be there" and reserve the explicit `| undefined` for the rare case where the key must exist as a signal even when empty.

**Worked example:**
```ts
interface Patch {readonly displayName?: string}
const p: Patch = {}; // ok — the `| undefined` form would reject {} and demand the key
```
**Enforcement:** `exactOptionalPropertyTypes` makes the misuse a compile error.

### 3.7 — Narrow with the weakest tool that works: discriminant, then `typeof`/`instanceof`/`in`, then a custom guard.

**Reasoning, step by step:**
1. Narrowing is how `unknown` and union types become usable, and the tools form a preference order from safest to most error-prone. A discriminant property (`switch (x.kind)`) is exhaustive-checkable and impossible to get subtly wrong; prefer it for any union you control (chapter [06](./06-classes-and-data-modeling.md)).
2. `typeof`, `instanceof`, and `in` are built-in, compiler-verified narrowings for primitives, classes, and property presence; they need no tests because the compiler implements them. A custom guard (`x is T`, 3.8) is the last resort, the only narrowing whose correctness the compiler *cannot* verify, so the only one you must test.

**Worked example:**
```ts
type Shape = {kind: 'circle'; r: number} | {kind: 'square'; side: number};
function area(s: Shape): number {
  switch (s.kind) {            // discriminant — exhaustive, weakest sufficient tool
    case 'circle': return Math.PI * s.r ** 2;
    case 'square': return s.side ** 2;
  }
}
```
**Enforcement:** `@typescript-eslint/switch-exhaustiveness-check` on discriminated `switch`.

### 3.8 — Unit-test every custom type guard (`x is T`).

**Reasoning, step by step:**
1. A custom guard is a function returning `x is T`. When it returns `true`, the compiler *believes* the value is `T` and stops checking. A wrong guard is therefore a type-system lie that spreads downstream as silent `any`-grade unsoundness.
2. Unlike `typeof` or a discriminant (3.7), the compiler cannot verify the body matches the claim; nothing stops a guard from returning `true` for a value missing half its fields. The only defense is tests: positive cases that must pass, and negative cases (wrong type, missing field, `null`) that must fail.

**Worked example:**
```ts
function isUser(x: unknown): x is User {
  return typeof x === 'object' && x !== null && 'id' in x && 'email' in x;
}
expect(isUser({id: 'u1', email: 'a@b.c'})).toBe(true);  // in *.test.ts
expect(isUser({id: 'u1'})).toBe(false);                 // negative space is mandatory
```
**Enforcement:** code review requires a colocated test; see chapter [11](./11-testing.md).

### 3.9 — Brand domain primitives in high-rigor modules.

**Reasoning, step by step:**
1. In a structural type system every `string` is interchangeable, so `UserId`, `OrderId`, and a raw email are one type and the compiler will happily pass one where another is meant. A brand attaches a phantom tag: `type UserId = string & {readonly __brand: 'UserId'}`. It costs nothing at runtime (the field never exists) but makes the values nominally distinct.
2. A branded value can only be *created* through a parsing constructor that validates and generates it. That constructor is the single place an `as` is sanctioned (3.4), because the value is proven the line before.

**Worked example:**
```ts
type UserId = string & {readonly __brand: 'UserId'};
function toUserId(raw: string): UserId {
  if (!/^[0-9a-f-]{36}$/.test(raw)) throw new Error(`invalid UserId: ${raw}`);
  return raw as UserId; // sanctioned: validated immediately above
}
```
**Enforcement:** convention in domain modules; the brand makes mismatches compile errors.

### 3.10 — Put `readonly`, `ReadonlyArray`, and `Readonly<T>` in every public signature.

**Reasoning, step by step:**
1. Immutability at the API boundary is the contract, not the caller's discipline. A mutable parameter type invites a callee to mutate the caller's data, and a mutable return invites the reverse.
2. Mark every field `readonly`, every array parameter `ReadonlyArray<T>` (or `readonly T[]`), and wrap object returns in `Readonly<T>`. The cost is keystrokes; the payoff is a class of aliasing bugs the compiler now refuses. This pairs with root rule 3 (immutable by default): update by spreading into a new value, never by mutating in place.

**Worked example:**
```ts
function top(items: number[]): number {}              // bad — callee may sort() in place
function top(items: ReadonlyArray<number>): number {} // good — signature promises not to mutate
```
**Enforcement:** `@typescript-eslint/prefer-readonly`; review of public signatures.

### 3.11 — Constrain every generic; add no gratuitous type parameters; annotate variance on public generic interfaces.

**Reasoning, step by step:**
1. An unconstrained `<T>` says nothing, so the body can do nothing with it but pass it through; if you need a property, constrain it (`<T extends {id: string}>`), which documents the requirement and unlocks the member access. A type parameter that appears exactly once is not generic, it is obfuscation: `function f<T>(x: T): void` is just `function f(x: unknown): void` with ceremony. Delete it.
2. On public generic interfaces, annotate variance with `in` and `out`. It documents how the parameter flows, lets the compiler check your intent, and speeds structural comparison.

**Worked example:**
```ts
function logId<T>(x: T): void {}                       // bad — T used once, constrains nothing
function id<T extends {readonly id: string}>(x: T) {} // good — constrained, parameter earns its place
interface Reader<out T> {read(): T;}                  // public generic: variance declared
```
**Enforcement:** `@typescript-eslint/no-unnecessary-type-parameters`; review of public generics for constraints and variance.

### 3.12 — Write erasable syntax only.

**Reasoning, step by step:**
1. `enum`, runtime `namespace`, constructor parameter properties, and `import =` aliases are type-looking syntax that emits *runtime* code. That is hidden behaviour by definition, in violation of root rule 2. A numeric `enum` even emits a bidirectional lookup object (`Color[0]` reverse-maps to the name).
2. They fight the boundary. A string `enum` is nominal in a structural language, so `JSON.parse` output cannot be assigned to it without a cast; a literal union round-trips JSON natively and plugs straight into zod (3.2).
3. They fight the toolchain. `const enum` is already broken under `isolatedModules`, and Node's native type-stripping cannot execute non-erasable syntax. The platform direction is settled. `erasableSyntaxOnly` (3.1) makes all four a compile error.
4. Nothing is lost. Exhaustiveness via `never` works identically on unions, and the replacement idioms below restore both small closed sets and rename-able, iterable sets.

**Worked example:**
```ts
enum Color {Red, Green}        // bad — emits runtime code; banned
// either:
type Color = 'red' | 'green';  // good — small closed set: a bare literal union
// or, when you need iteration:
const Color = {Red: 'red', Green: 'green'} as const; // good — rename-able, iterable
type Color = (typeof Color)[keyof typeof Color];
```
**Carve-out:** consuming an `enum` from a codegen or third-party library (Prisma, gRPC, the TS compiler API) is allowed *at the boundary*; convert to a domain union inside. `declare namespace` in an ambient `.d.ts` stays legal because it is erasable. This rule is the source of two Deviations ledger entries (`enum`, constructor parameter properties); see the [README](./README.md).
**Enforcement:** `erasableSyntaxOnly` (TypeScript ≥ 5.8) makes every banned form a compile error.

## Cross-references

- Flags and lint overlay: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md). Call-site naming: [02-naming-conventions.md](./02-naming-conventions.md). Non-null `!` and `as const`: [04-variables-and-declarations.md](./04-variables-and-declarations.md). `invariant(): asserts`: [05-functions.md](./05-functions.md).
- Illegal states unrepresentable, discriminated unions, parse-don't-validate: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md). `satisfies`, `as const`, `??` over `||`: [07-typescript-idioms.md](./07-typescript-idioms.md). `catch (e: unknown)` and `Result` unions: [08-error-handling.md](./08-error-handling.md).
- Testing guards and `expectTypeOf`: [11-testing.md](./11-testing.md). `import type` discipline: [12-module-organization.md](./12-module-organization.md). Monomorphism and stable shapes: [15-performance.md](./15-performance.md).
