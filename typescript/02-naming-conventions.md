# 02 — Naming Conventions

Google's TypeScript casing rules are the floor. This chapter takes its *identifier* casing verbatim (file naming deviates — see 2.4) and adds the project layer: a client verb taxonomy ported from Python, predicate booleans, unit-bearing quantities, and the discipline of naming for the call site. Casing is mechanical and lint-enforced; the rest is taste, and taste is what makes an API legible. Where this chapter and the Google guide collide on casing, Google wins.

## What good looks like

```ts
const DEFAULT_PAGE_SIZE = 50; // 2.3 — deeply immutable and conceptually constant
const HTTP_STATUS = {ok: 200, notFound: 404} as const; // 2.3 — frozen lookup, conceptually fixed

interface User {
  id: string;
  email: string;
  isVerified: boolean; // 2.8 — a question, not a state
}

interface CreateUserRequest {
  email: string;
}

class UserClient {
  async getUser(id: string): Promise<User> { /* ... */ } // 2.6 — one item, throws when absent
  async listUsers(pageSize = DEFAULT_PAGE_SIZE): Promise<readonly User[]> { /* ... */ } // 2.6/2.7 — many; Promise says async
  async createUser(request: CreateUserRequest): Promise<User> { /* ... */ } // 2.6 — throws when it exists
  async upsertUser(request: CreateUserRequest): Promise<User> { /* ... */ } // 2.6 — idempotent create-or-update
  async deleteUser(id: string): Promise<void> { /* ... */ } // 2.6 — idempotent, no-ops when absent
  isStaleByAge(user: User, maxAgeMs: number): boolean { /* ... */ } // 2.5/2.8/2.10 — predicate, unit suffix
  isOk(status: number): boolean { return status === HTTP_STATUS.ok; } // 2.8 — predicate reading the frozen lookup
}
```

Properties, parameters, and methods are `lowerCamelCase`; the type and class are `UpperCamelCase`; the module-level constants are `CONSTANT_CASE`, including the deeply immutable `HTTP_STATUS` lookup. Every resource verb on the client comes from the taxonomy (2.6) and carries its documented contract, so a reader knows `getUser` throws, `upsertUser` is idempotent, and `deleteUser` does not throw before reading a line of the body. The predicates `isVerified`, `isStaleByAge`, and `isOk` read as yes/no questions (2.8), and `maxAgeMs` states its unit at the call site so no caller passes seconds by accident.

## Rules

### 2.1 — Apply Google's identifier casing table verbatim.

**Reasoning, step by step:**
1. `lowerCamelCase` for variables, functions, properties, parameters, and methods.
2. `UpperCamelCase` for classes, interfaces, types, type aliases, and enum-like `as const` maps.
3. `CONSTANT_CASE` only for module-level constants that are deeply immutable (see 2.3).
4. This is Google's table, taken as-is. The value is uniformity across every dexpace TypeScript repository, not novelty.

| Construct | Case | Example |
|---|---|---|
| Variable, function, property, parameter, method | `lowerCamelCase` | `pageSize`, `getUser` |
| Class, interface, type, type alias | `UpperCamelCase` | `UserClient`, `CreateUserRequest` |
| Module-level deep constant | `CONSTANT_CASE` | `DEFAULT_PAGE_SIZE` |

**Enforcement:** `@typescript-eslint/naming-convention` (the `gts` baseline).

### 2.2 — Encode visibility with keywords, not glyphs.

**Reasoning, step by step:**
1. Drop the `I` prefix on interfaces. `User`, not `IUser`. The prefix is Hungarian notation that leaks an implementation distinction the consumer should not care about.
2. Use no leading or trailing underscore on any name. `_user` and `user_` both signal a privacy convention the language already expresses.
3. Reach for the `private`, `protected`, and `#name` keywords when you need encapsulation. The reader sees `private balance`, not `_balance`, and the compiler checks it rather than honoring a convention.

**Enforcement:** `@typescript-eslint/naming-convention` with `leadingUnderscore: forbid` and a custom interface filter rejecting the `^I[A-Z]` pattern.

### 2.3 — Reserve `CONSTANT_CASE` for deeply immutable, conceptually constant values.

**Reasoning, step by step:**
1. `CONSTANT_CASE` makes two promises at once: the binding never changes and the value it points to never changes.
2. A frozen lookup table qualifies. `const HTTP_STATUS = { ok: 200, notFound: 404 } as const` is deeply immutable and conceptually fixed.
3. A mutable singleton never qualifies, even at module scope. `const retryableStatus = new Set([429, 503])` keeps its binding but mutates its contents, so it stays `lowerCamelCase`.
4. The test is the value, not the location. Ask "could a field of this ever change after construction?" If yes, it is camel. Google allows `CONSTANT_CASE` on intent alone; we hold the stricter deeply-immutable line.

**Enforcement:** review; `@typescript-eslint/naming-convention` cannot judge deep immutability.

### 2.4 — Name files in kebab-case.

**Reasoning, step by step:**
1. Use `user-service.ts`, not `userService.ts` or `UserService.ts`. The file name need not match the exported symbol's case; `user-client.ts` exporting `UserClient` is correct and expected.
2. Kebab-case is case-insensitivity-safe. A `UserService.ts` import resolves on a case-insensitive macOS checkout and breaks on a case-sensitive Linux CI runner; lowercase-only names cannot drift this way.
3. It is the ecosystem norm. Most published TypeScript packages ship kebab-case modules, so the convention reads as ordinary to any contributor.
4. This departs from Google's snake_case files; kebab-case is the stronger ecosystem norm — recorded in the charter ledger.

**Enforcement:** `eslint-plugin-unicorn`'s `filename-case` set to `kebabCase`.

### 2.5 — Spell names out beyond the idiomatic abbreviation set.

**Reasoning, step by step:**
1. Names are read far more often than they are typed, so optimize for the reader, who pays the cost every time.
2. The idiomatic set is allowed everywhere: `id`, `url`, `ctx`, and the loop counter `i`. These are universal and carry no ambiguity.
3. Everything else gets spelled out. `req` becomes `request`, `usr` becomes `user`, `cfg` becomes `config`. The saved keystrokes are not worth the re-read.
4. Never drop letters to invent a shorthand. `Sbx` for `Sandbox` and `Usr` for `User` are unsearchable and force the reader to decode them.

**Enforcement:** review; editor autocomplete removes the typing-cost argument.

### 2.6 — Draw resource verbs from the client taxonomy.

**Reasoning, step by step:**
1. When a method operates on a resource, pick a verb from the fixed set below. The reader then knows the contract, including whether it throws, without reading the body.
2. Use the documented verb against its documented semantics, and never invent a synonym. `fetchUser`, `readUser`, and `findUser` are all wrong when `getUser` is the established name; synonyms read like a disorganized API.
3. This is the Python taxonomy (§2.12) rendered in `lowerCamelCase`. The same service shipped in Go (`GetUser`), Kotlin (`getUser`), and TypeScript (`getUser`) uses the same verb for the same operation.

| Verb | Semantics |
|---|---|
| `get<Noun>` | Fetch one resource. Contract: throw when absent, or return `undefined` if the signature says so. Pick one and document it. |
| `list<Noun>` | Enumerate many. Returns an array or async iterable, never a single item, never `undefined`. |
| `create<Noun>` | Create new. Throws when it already exists. |
| `upsert<Noun>` | Create or update. Idempotent. |
| `update<Noun>` | Modify existing. Throws when absent. |
| `delete<Noun>` | Remove. No-ops when absent; does not throw on "not found." |
| `begin<Noun>` | Start a long-running operation. Returns a poller or handle. |

**Enforcement:** review; reinforced by the API-design chapter's surface checks.

### 2.7 — Omit the `Async` suffix.

**Reasoning, step by step:**
1. The return type already announces asynchrony. A method returning `Promise<User>` is async, and the signature says so at every call site, so `getUser` is complete.
2. `getUserAsync` is Hungarian notation: it encodes a type fact in the name, which then duplicates and can contradict the signature.
3. There is no sync twin to disambiguate. The project does not ship paired `getUser` and `getUserAsync`, so the suffix distinguishes nothing.

**Enforcement:** `@typescript-eslint/naming-convention` custom filter rejecting the `Async$` suffix on methods.

### 2.8 — Write booleans as predicates.

**Reasoning, step by step:**
1. Prefix booleans with `is`, `has`, `can`, or `should`. `enabled` names a state; `isEnabled` asks a yes/no question, which is what a boolean answers.
2. The predicate form reads correctly inside a condition. `if (user.isVerified)` is a sentence; `if (user.verified)` is ambiguous between an adjective and a verb.
3. Avoid negative stems. `isNotReady` produces the double negative `!isNotReady`; prefer `isReady` and negate at the use site.
4. Apply the same rule to functions returning booleans. `canRetry(error)` and `hasCapacity(pool)` read as questions, matching their return type.

**Enforcement:** `@typescript-eslint/naming-convention` requires the `is`/`has`/`can`/`should` prefix on `boolean`-typed properties, variables, and parameters (typed linting selects these by type). Boolean-*returning* functions and methods cannot be selected by return type, so the predicate form there is review-enforced.

### 2.9 — Design the name for the call site.

**Reasoning, step by step:**
1. The audience for a public name is the reader of the caller, not the author of the callee. The name is read far more often where it is used than where it is defined.
2. Write the call site first to test a candidate name. If `store.upsertUser(request)` reads clearly without the surrounding definition, the name works.
3. The module context is already present at the call site, so do not repeat it. The surrounding `userStore.get(id)` supplies the noun; `userStore.getUserById(id)` stutters it.
4. A name that needs a comment to explain it at the call site is the wrong name. Rename until the call line stands alone.

**Enforcement:** review.

### 2.10 — Carry units in quantity names.

**Reasoning, step by step:**
1. A number with a physical unit names that unit: `timeoutMs`, `sizeBytes`, `ttlSeconds`. The unit lives in the name because the `number` type cannot hold it.
2. Unit confusion is a correctness-bug class, not a style nit. A caller passing seconds into a parameter that expects milliseconds compiles cleanly and fails in production.
3. Place the unit last, as a suffix on the concept: `maxAgeMs`, not `msMaxAge`. The concept leads, the unit qualifies. A branded type that encodes the unit (see the type-system chapter) makes the suffix redundant; absent that, the suffix is mandatory.

**Enforcement:** review; branded duration and size types make the unit type-checkable where they exist.

### 2.11 — Name type parameters by role clarity.

**Reasoning, step by step:**
1. Use a single conventional letter when the role is obvious from position. `T` for one generic type, `K` and `V` for a key and value pair, `E` for an element.
2. Use a descriptive `UpperCamelCase` name with a `T` prefix when the role is not obvious. `TRow`, `TError`, and `TResponse` carry meaning a bare letter cannot.
3. The prefix keeps a descriptive type parameter visually distinct from a concrete type. `TError` is plainly a parameter; `Error` is plainly the global. The reader never confuses the two.
4. Match the choice to the signature's complexity. `first<T>(items: readonly T[])` needs no name; `mapRows<TRow, TResult>(rows, fn)` earns both, because two interacting parameters would be opaque as `mapRows<A, B>`.

**Enforcement:** `@typescript-eslint/naming-convention` requiring either a single uppercase letter or a `T`-prefixed `UpperCamelCase` name for `typeParameter`.

## Cross-references

- [01-formatting-and-tooling.md](./01-formatting-and-tooling.md) — the `gts` lint baseline enforcing the casing rules here.
- [03-the-type-system.md](./03-the-type-system.md) — branded primitives that make units (2.10) type-checkable; `as const` maps (2.3).
- [10-api-design.md](./10-api-design.md) — the client surface and `Error` subclasses the verb taxonomy (2.6) governs.
- [12-module-organization.md](./12-module-organization.md) — module layout that kebab-case files (2.4) feed.
- [typescript-react/05-structure-and-routing.md](../typescript-react/05-structure-and-routing.md) — the React chapter that overrides the kebab-case file rule (2.4) to PascalCase component files.
