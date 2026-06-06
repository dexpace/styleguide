# 10 — API Design

Designing the surface other code imports. A package's public API is a promise: every name you export is a contract you keep for every caller, in every refactor, until a major version lets you break it. This chapter is about exporting the least, exporting it deliberately, and shaping what you do export so the call site reads as a family. The verb taxonomy ([chapter 02](./02-naming-conventions.md)), branded parse-mint boundary ([chapter 03](./03-the-type-system.md)), failure documentation ([chapter 08](./08-error-handling.md)), and cancellation ([chapter 09](./09-concurrency.md)) are the raw material; here they compose into a surface.

## What good looks like

```ts
// index.ts — the package contract. These names are public; everything else is internal.
export {UserClient} from './user-client.js';
export {UserSchema, type User, type UserId, type CreateUser} from './user.js';
export type {ListUsersOptions} from './user-client.js';
export {UserNotFoundError} from './errors.js';
// user.ts — zod schema at the wire boundary; the type is inferred, never hand-written (10.7).
import {z} from 'zod';
export type UserId = string & {readonly __brand: 'UserId'};
export const UserSchema = z.object({
  id: z.uuid().transform((s): UserId => s as UserId), // parse-mint (ch. 03)
  email: z.email(),
}).readonly();
export type User = z.infer<typeof UserSchema>; // single source of type truth
export const CreateUserSchema = z.object({email: z.email()}).readonly();
export type CreateUser = z.infer<typeof CreateUserSchema>; // input type, also schema-derived
// user-client.ts — a symmetric verb family over one resource.
export interface ListUsersOptions {
  /** Page size. @default 50 */ readonly pageSize?: number;
  /** Cancellation. */ readonly signal?: AbortSignal;
}
export class UserClient {
  /** @throws {UserNotFoundError} no user with this id. */
  async getUser(id: UserId, options?: {readonly signal?: AbortSignal}): Promise<User> { /* ... */ }
  listUsers(options?: ListUsersOptions): AsyncIterable<User> { /* ... */ } // pagination is iteration (10.10)
  async createUser(input: CreateUser, options?: {readonly signal?: AbortSignal}): Promise<User> { /* ... */ }
}
```

`index.ts` is the whole contract (10.2): the `.js` deep imports stay internal. The verbs are drawn from the taxonomy and read as a family — `get`/`list`/`create` share parameter shape and an options-tail (10.6). Every export is named (10.1), the input is an `interface` and the output a `readonly` inferred type (10.4, 10.7), `pageSize` documents its default at the option (10.5), the throwing operation declares its failure (10.8), and `signal` threads cancellation through every call (10.8).

## Rules

### 10.1 — Export named symbols only; never `export default`.

**Reasoning, step by step:**
1. A default export has no canonical name at the import site. Each caller invents one — `import client from './x'` versus `import c from './x'` — so the same symbol wears a dozen names across the codebase, and a grep for the real name finds none of them.
2. Defaults also rename silently under refactor. Rename the exported class and every named importer breaks loudly at compile time, which is what you want; the default importers keep compiling against a name that no longer means what it did.
3. A named export is one identifier everywhere — defined once, imported under that spelling, greppable, and renamed atomically by an editor across the whole repository. This is the Google position, taken verbatim.

```ts
export default class UserClient {}        // bad — caller picks the name; grep finds nothing
export class UserClient {}                // good — one name, greppable, atomically renamable
```
**Enforcement:** `import/no-default-export` (the `gts` baseline); allowed only where a framework demands it (a route module's default), confined to those files.

### 10.2 — `index.ts` is the contract; everything else is internal.

**Reasoning, step by step:**
1. A package needs exactly one front door. `index.ts` re-exports the public surface and nothing else; every other module is an implementation detail the package author is free to move, rename, or split without telling a single caller.
2. The barrel re-exports only what callers need — `export {UserClient} from './user-client.js'`, `export type {User} from './user.js'`. A symbol that never appears in `index.ts` is private to the package even though TypeScript marks it `export` (the `export` is for sibling modules, not the world).
3. Re-export at the boundary only. A barrel per feature folder, not per directory; deep barrel chains create import cycles and defeat tree-shaking ([chapter 12](./12-module-organization.md)). The hard wall — making `import 'pkg/internal/x'` fail to resolve — is the `package.json` `exports` field, which lands in the [Node guide's build chapter](../typescript-node/08-build-and-distribution.md); reference it, do not restate it here.

```ts
// index.ts — the entire public surface, re-exported from one place
export {UserClient} from './user-client.js';
export {UserSchema, type User} from './user.js';
// user-client.ts imports './http-pool.js' freely; it is never re-exported, so it stays internal
```
**Enforcement:** review that `index.ts` is the sole barrel; `package.json` `exports` (Node guide) blocks deep imports at the package wall.

### 10.3 — Export the least; start private and promote deliberately.

**Reasoning, step by step:**
1. Every export is a promise kept forever. Once a symbol is in `index.ts` and shipped, removing or renaming it is a breaking change (10.9) that costs a major version and a migration for every caller. The cheapest API to maintain is the one you never exported.
2. So default to private. A new helper, type, or class is unexported until a caller outside the package genuinely needs it; only then is it promoted into the barrel. Promotion is a one-line, reversible-while-unreleased decision; demotion after release is a breaking change. The asymmetry says start closed.
3. This is the inverse of the convenience reflex: "export it in case someone wants it" trades a permanent contract for a hypothetical caller. Wait for the real second caller, then export.

```ts
function normalizeEmail(raw: string): string {} // internal: used only inside the package — not exported
export class UserClient {}                       // public: the one symbol a caller actually needs
```
**Enforcement:** review of every addition to `index.ts`; a new public export is a contract change, reviewed as one.

### 10.4 — Accept interfaces; return concrete `readonly` types.

**Reasoning, step by step:**
1. A function's input contract should be the narrowest shape it actually uses, and its output the fully-known thing it actually produces. These pull in opposite directions: be liberal in what you accept, strict in what you return.
2. So accept an `interface` describing only the members you touch. A function that reads a name and an id takes `{readonly id: string; readonly name: string}`, not the 30-field `User` class — any value with those members satisfies it, including a test fake, without imposing a type on the caller (this is consumer-defined interfaces, ported from [Kotlin 10.2](../kotlin/10-api-design.md)).
3. Return the concrete type, fully specified and `readonly` (3.10). The caller gets every field and a compiler guarantee the value will not mutate under them. Never return a wide `unknown` or a bare interface where a concrete `readonly` type is known — that pushes a narrowing burden onto every call site.

```ts
function domainOf(user: {readonly email: string}): string {} // narrowest input — any value with `email` qualifies
function loadUser(id: UserId): Promise<Readonly<User>> {}     // concrete, fully-known, immutable output
```
**Enforcement:** review; `@typescript-eslint/prefer-readonly` and the readonly-signature rule (3.10) on returns.

### 10.5 — Options objects carry documented defaults; callers state only deviations.

**Reasoning, step by step:**
1. Past two parameters, a positional list loses meaning at the call site (chapter 05): `fetch(url, 5000, 3, true)` is unreadable. Collect optionals into a single options object so each is named where it is passed.
2. The default for each option lives in exactly one place — the implementation — and is documented at the option with a `@default` tag (root rule 2's carve-out: library options follow documented defaults, callers pass only what differs). The TSDoc sits on the option field, so the default is visible on hover at the call site without opening the source.
3. Make the whole options object optional and every field within it optional and `readonly`, so the zero-config call `client.list()` works and a caller overrides one field without restating the rest. This is the inverse of an options *bag* the caller must always construct; here the common path passes nothing.

```ts
export interface RetryOptions {
  /** Attempts before giving up. @default 3 */ readonly maxAttempts?: number;
  /** Base backoff in ms. @default 100 */ readonly backoffMs?: number;
}
function withRetry<T>(fn: () => Promise<T>, options: RetryOptions = {}): Promise<T> {
  const {maxAttempts = 3, backoffMs = 100} = options; // defaults in one place, matching the docs
}
```
**Enforcement:** `max-params 3` (chapter 01) forces the object; review that each option's `@default` matches the implementation.

### 10.6 — Keep API symmetry: parallel operations share vocabulary, parameter shape, and return shape.

**Reasoning, step by step:**
1. Operations that do parallel things should read as variations on one form. When `getUser`, `listUsers`, and `createUser` share their verb taxonomy ([chapter 02](./02-naming-conventions.md)), their parameter order, and the shape of their options tail, a caller learns the family once and predicts the rest. Asymmetry — `getUser(id)` beside `fetchUserList()` beside `userCreate(data, opts)` — forces the call site to relearn each method.
2. Symmetry is concrete: the same noun spelling across the family, the resource argument first, the options object last, cancellation via the same `signal` field everywhere (10.8), and parallel return shapes — `get` returns one, `list` enumerates many (10.10), `create` returns the created one. A reader who has called `getUser` can then write `getOrder` without consulting the signature; divergence must be a deliberate, documented exception, not an accident of who wrote which method.

```ts
async getUser(id: UserId, options?: CallOptions): Promise<User> {}             // resource arg first, options tail
async getOrder(id: OrderId, options?: CallOptions): Promise<Order> {}          // same verb, different noun — predictable
listUsers(options?: ListOptions & CallOptions): AsyncIterable<User> {}         // cross-verb: same options tail, same cancellation
```
**Enforcement:** review against the verb taxonomy (chapter 02); the surface is checked as a family, not method by method.

### 10.7 — Put a zod schema at every external boundary; `z.infer` is the single source of type truth for wire data.

**Reasoning, step by step:**
1. Data crossing the wire — a `fetch` response, a request body, a queue message — arrives as `unknown` (3.2). It must be parsed into a domain type before the interior touches it, and the parser is a zod schema that validates shape and mints any brands (3.9) in one pass.
2. The TypeScript type for that wire data is *derived from the schema*, never written alongside it: `type User = z.infer<typeof UserSchema>`. A hand-written `interface User` maintained next to the schema is two sources of truth that drift the moment one is edited and the other forgotten — and the drift is silent, because each compiles. One declaration, inferred, cannot drift from itself.
3. Parse at the boundary, then trust the type inside. The schema runs once where the data enters; downstream code consumes the inferred type with no re-validation. Call `.readonly()` on the schema: it freezes the parsed value at runtime *and* makes `z.infer` yield a readonly type, so the data is immutable from birth (3.10) with no separate `Readonly<>` wrapper to remember.

```ts
const UserSchema = z.object({id: z.uuid(), email: z.email()}).readonly();
type User = z.infer<typeof UserSchema>;                 // type follows schema — no hand-written twin
export async function fetchUser(url: string): Promise<User> {
  return UserSchema.parse(await (await fetch(url)).json()); // parse the unknown at the boundary
}
```
**Enforcement:** review; wire types are `z.infer`, never hand-written beside a schema; boundary input typed `unknown` then parsed (3.2).

### 10.8 — Document failure modes and accept cancellation on every public operation.

**Reasoning, step by step:**
1. A thrown error is invisible in a signature and an uncancellable call is a resource leak waiting to happen; both are hidden behaviour a caller discovers in production. Every public operation makes both explicit: how it can fail, and how to stop it.
2. Declare failure one of two ways ([chapter 08](./08-error-handling.md)): a `@throws` tag per error type a caller might reasonably catch, or a `Result<T, E>` return that puts the failures in the type itself. Prefer `Result` where it fits, because it cannot drift from the implementation; with `@throws`, list only the errors a caller would act on and keep them in step with the code.
3. Accept cancellation through a `{ signal }: AbortSignal` option on every async operation ([chapter 09](./09-concurrency.md)), threaded down to the underlying I/O. The field lives in the same options object as everything else (10.5) and carries the same name across the whole family (10.6), so cancelling any call looks identical.

```ts
/**
 * Charges a card.
 * @throws {CardDeclinedError} the issuer declined — retry with another card.
 * @throws {GatewayUnavailableError} gateway unreachable — safe to retry.
 */
chargeCard(card: Card, amount: Cents, options?: {readonly signal?: AbortSignal}): Promise<Receipt>;
```
**Enforcement:** review; `@throws` or a `Result` signature on every throwing public (chapter 08), `{ signal }` on every public async operation (chapter 09).

### 10.9 — Deprecate with `@deprecated` and a migration path; breaking changes are MAJOR.

**Reasoning, step by step:**
1. A public symbol cannot simply vanish — removing or renaming it breaks every caller. Removal is a two-release process: mark the old symbol `@deprecated`, keep it working as a thin shim for one major cycle, then delete it. The `@deprecated` TSDoc tag is load-bearing: editors strike the symbol through at every call site and the linter can flag uses, so callers see the warning where they call, not in a changelog they never read.
2. The tag must carry a migration path. "Deprecated, use `createUser`, removed in v3" names the replacement and the removal version; a bare `@deprecated` is a dead end that tells the caller to leave but not where to go.
3. The version math is semver, non-negotiable: removing or renaming a public symbol, narrowing a parameter type, or changing return semantics is a breaking change and a MAJOR bump — no exceptions (see the [git guide's Release Discipline](../git-and-code-review.md)). Adding a new optional option (10.5) or a new method is a MINOR; a bug fix that preserves the contract is a PATCH.

```ts
/** @deprecated Use {@link createUser} instead. Removed in v3.0. */
export function addUser(input: CreateUser): Promise<User> {
  return createUser(input); // thin shim for one major cycle, then deleted
}
```
**Enforcement:** review; `@deprecated` with a named replacement and removal version; breaking changes gated behind a MAJOR bump per the git guide.

### 10.10 — Stream results as `AsyncIterable`; pagination is iteration.

**Reasoning, step by step:**
1. A result set that could be large or unbounded — every user, every log line, a paginated endpoint — must not be returned as one eager array. That materializes the whole thing in memory before the caller sees the first element and offers no way to stop early. Return an `AsyncIterable<T>` instead: pull-based, backpressure-aware, consumed with `for await`.
2. Pagination is an implementation detail the iterable hides. The async generator fetches page one, yields its items, and fetches page two only when the consumer has drained page one — so a caller who `break`s after ten items never fetches page two. The continuation-token plumbing lives inside the generator; the caller just iterates.
3. The iterable threads cancellation like every other operation (10.8): the `{ signal }` aborts the in-flight page fetch, and the `for await` loop stops. Return `AsyncIterable<T>` (the minimal consumable contract), not a concrete generator type, so the implementation is free to change.

```ts
async function* listUsers(options?: ListUsersOptions): AsyncIterable<User> {
  let cursor: string | undefined;
  do {
    const page = await fetchPage(cursor, {signal: options?.signal}); // options-shape even internally; next page pulled only when asked
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor !== undefined);
}
for await (const u of listUsers()) if (u.isAdmin) break; // break stops early; later pages never fetched
```
**Enforcement:** review; unbounded or paginated results return `AsyncIterable<T>`, never an eager array; the `{ signal }` aborts the in-flight fetch.

## Cross-references

- Client verb taxonomy (`get`/`list`/`create`/`upsert`/`update`/`delete`/`begin`) and call-site naming: [02-naming-conventions.md](./02-naming-conventions.md).
- Branded primitives, parse-mint, `unknown` at the boundary, `readonly` signatures: [03-the-type-system.md](./03-the-type-system.md).
- `max-params 3` forcing options objects: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md); options-object construction: [05-functions.md](./05-functions.md).
- `@throws` versus `Result`, documenting failure modes: [08-error-handling.md](./08-error-handling.md). `{ signal }` cancellation: [09-concurrency.md](./09-concurrency.md).
- Barrels at the boundary, import cycles, tree-shaking: [12-module-organization.md](./12-module-organization.md). Semver and breaking-change discipline: [git-and-code-review.md](../git-and-code-review.md).
- Deep-import blocking via `package.json` `exports`: [typescript-node build chapter](../typescript-node/08-build-and-distribution.md).
