# 06 — Classes and Data Modeling

TypeScript's type system is structural and its classes are optional. This chapter says what to model with plain data and free functions, what to model with classes, and how to shape both so the compiler rejects the states your domain forbids. The governing idea: in the type system, elegance and correctness are the same move. A type that admits only legal values is also the type that reads cleanly.

## What good looks like

```ts
type OrderId = string & {readonly __brand: 'OrderId'};

// One union member per legal state; no other shape is constructible.
type Order =
  | {readonly status: 'pending'; readonly id: OrderId; readonly total: number}
  | {readonly status: 'paid'; readonly id: OrderId; readonly total: number; readonly paidAt: Date}
  | {readonly status: 'cancelled'; readonly id: OrderId; readonly reason: string};

function assertNever(x: never): never {  // defined once — see 6.5
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`);
}

// Parse-then-generate: validate the raw string, then the one sanctioned `as` (6.11).
function toOrderId(raw: string): OrderId {
  if (raw === '') throw new Error('OrderId must be non-empty');
  return raw as OrderId;  // sanctioned: validated immediately above
}

function describe(order: Order): string {
  switch (order.status) {
    case 'pending': return `Order ${order.id} awaiting ${order.total}`;
    case 'paid': return `Order ${order.id} paid at ${order.paidAt.toISOString()}`;
    case 'cancelled': return `Order ${order.id} cancelled: ${order.reason}`;
    default: return assertNever(order);
  }
}

// Factory validates then assigns; an invalid Order cannot exist.
function createPendingOrder(id: string, total: number): Order {
  if (total <= 0) throw new RangeError(`total must be positive, got ${total}`);
  return {status: 'pending', id: toOrderId(id), total};
}
```

This exemplar demonstrates the headline rule (6.1) — `paidAt` and `reason` live only on the states that own them, so a "paid and cancelled" order cannot be typed. It uses a `type` for the union (6.2), a literal `status` discriminant with an exhaustive `switch` closed by `assertNever` (6.5), `readonly` fields throughout (6.6), a `create*` factory that validates and a constructor-free object that only assigns (6.8), and a branded `OrderId` (6.11).

## Rules

### 6.1 — Make illegal states unrepresentable.

**Reasoning, step by step:**
1. Every optional field doubles the state space. A record with `paidAt?: Date` and `cancelledAt?: Date` has four combinations, but only three are legal; the fourth, both set, is a bug the type invites.
2. Model the variants as union members instead of optional fields on one bag. Each member carries exactly the fields that state owns, so the illegal combinations fail to typecheck rather than fail in production. The type system does the work a runtime invariant check would otherwise do.

```ts
// bad — permits paidAt AND cancelledAt set together
interface Order {status: string; paidAt?: Date; cancelledAt?: Date}

// good — each state owns its fields; the illegal combination is unconstructible
type Order =
  | {status: 'paid'; paidAt: Date}
  | {status: 'cancelled'; cancelledAt: Date};
```

**Enforcement:** Code review against the optional-field bag; `typescript-eslint` strictness plus exhaustiveness checking (6.5) catch the downstream access.

### 6.2 — Use `interface` for object shapes and `type` for everything else.

**Reasoning, step by step:**
1. Google's TypeScript guide splits these by job. An `interface` declares the shape of an object: a record with named fields. A `type` alias names everything else the type system can express — unions, intersections, mapped types, conditional types, tuples, primitives.
2. The split is a signal, not a preference. `interface Order` tells the reader "object with these fields"; `type Order = A | B` tells them "one of these variants." Mixing the two erases the signal.

```ts
interface User {readonly id: UserId; readonly email: string}

type Result<T> = {ok: true; value: T} | {ok: false; error: Error};
type Handlers = Readonly<Record<string, (event: Event) => void>>;
```

**Enforcement:** `@typescript-eslint/consistent-type-definitions` set to `interface`; reviewers flag a `type` used for a plain object shape.

### 6.3 — Model data as plain objects and free functions; reserve classes for stateful lifecycle resources.

**Reasoning, step by step:**
1. Root rule 1 is "data and functions, not objects." A domain record is data: type it with an `interface`, transform it with free functions that take it as a parameter and return new data. No `this`, no hidden state, trivially testable.
2. A class earns its keep only when an instance owns a lifecycle — something you open and close, or that holds mutable runtime state behind an invariant: a connection, a cache, a pool, a server, a client (see chapter 13). The test is whether the thing has a lifecycle. A `Money` or `Order` does not and is data; a `DbConnection` does and is a class.

```ts
// data + free function: no lifecycle, so no class
interface Money {readonly cents: number; readonly currency: 'USD' | 'EUR'}
const add = (a: Money, b: Money): Money =>
  ({cents: a.cents + b.cents, currency: a.currency});

// class: owns an open/close lifecycle, so state is justified
class ConnectionPool {
  private readonly idle: Connection[] = [];
  async acquire(): Promise<Connection> {/* ... */}
}
```

**Enforcement:** Review heuristic — a class with no field that changes after construction and no `dispose`/`close` should be an interface plus functions.

### 6.4 — Use `extends` only for `Error` hierarchies; compose by explicit delegation everywhere else.

**Reasoning, step by step:**
1. Root rules 1 and 5 ban inheritance for code reuse: a subclass is permanently coupled to its parent's internals, and a deep tree turns every parent change into a survey of descendants. The single sanctioned use of `extends` is the `Error` hierarchy, because `instanceof` dispatch and the platform's error model are built on the prototype chain (see chapter 08).
2. Reuse without inheritance is delegation: hold the collaborator as a field and forward the calls you mean to expose. The forwarding is explicit, so the surface is exactly what you wrote, with no inherited members leaking through.

```ts
class NotFoundError extends Error {}  // the only sanctioned extends: Error

// reuse by delegation, not inheritance: hold inner, forward what you expose
class CachingUserRepo {
  private readonly inner: UserRepo;
  private readonly cache: Cache;
  constructor(inner: UserRepo, cache: Cache) {
    this.inner = inner;
    this.cache = cache;
  }
  async find(id: UserId): Promise<User | undefined> {
    const cached = await this.cache.get(id);
    if (cached !== undefined) return cached;
    return this.inner.find(id);
  }
}
```

**Enforcement:** `@typescript-eslint/no-extraneous-class`; review rejects any `extends` whose base is not an `Error`.

### 6.5 — Model closed polymorphism as a discriminated union with an exhaustive `switch`.

**Reasoning, step by step:**
1. A discriminated union is the sum type: a finite, compiler-known set of variants, each tagged by a shared literal discriminant. Name it `kind` or, better, a domain term like `status` or `type`.
2. Branch on the discriminant with a `switch`. Inside each `case`, the compiler narrows to that one variant and grants access to its fields.
3. Close the `switch` with `default: return assertNever(x)`. Because every handled case narrows the subject, the value reaching `default` has type `never`; add a variant without a case and that `never` breaks, turning an omission into a compile error. Define `assertNever` once and import it everywhere:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`);
}

type Shape = {kind: 'circle'; radius: number} | {kind: 'square'; side: number};

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'square': return shape.side ** 2;
    default: return assertNever(shape);  // never: breaks if a variant is added
  }
}
```

**Enforcement:** `@typescript-eslint/switch-exhaustiveness-check` plus the `assertNever` `never` guard; both fail when a variant is unhandled.

### 6.6 — Make fields `readonly` by default and accept `Readonly<T>` in public signatures.

**Reasoning, step by step:**
1. Root rule 3 is immutable by default. A `readonly` field cannot be reassigned after construction, so a value's shape is fixed for its lifetime and a class of accidental-mutation bugs disappears. Mutability is the exception you type explicitly — a non-`readonly` field, true only inside the lifecycle classes of 6.3.
2. At the boundary, accept `Readonly<T>` and `ReadonlyArray<T>` (or `readonly T[]`). The signature promises the caller you will not mutate what they hand you, and the compiler holds you to it (see chapter 03).

```ts
interface Order {readonly id: OrderId; readonly lines: ReadonlyArray<OrderLine>}

function total(order: Readonly<Order>): number {
  return order.lines.reduce((sum, line) => sum + line.amount, 0);
}
```

**Enforcement:** `@typescript-eslint/prefer-readonly`; chapter 03 governs `readonly` in public types.

### 6.7 — Prefer the `private` modifier over `#private` fields.

**Reasoning, step by step:**
1. Google prefers TypeScript's `private` modifier. It is compile-time only and erasable: it emits no runtime code, costs nothing, and disappears under native type-stripping (chapter 01's erasable-syntax stance). A `#private` field is a runtime construct — a hard-private slot enforced by the engine that has a real cost and changes the emitted output.
2. Reach for `#` only when runtime privacy is itself the requirement: a library whose internals must stay unreachable even by reflective or bracket access. For ordinary application code, `private` is the default.

```ts
class RateLimiter {
  private tokens: number;  // default: erasable, zero-cost (not #tokens)
  constructor(max: number) {
    this.tokens = max;
  }
}
```

**Enforcement:** Review default is `private`; a `#` field requires a comment justifying the runtime-privacy requirement.

### 6.8 — Constructors assign; factories validate.

**Reasoning, step by step:**
1. A constructor's only job is to assign its arguments to fields. It does no I/O and no branching. That keeps construction cheap, total, and predictable — `new` never fails for a reason the caller has to decode.
2. Validation lives in a `create*` factory function that parses raw inputs with zod or explicit invariants, then either returns a fully valid object or throws. This is "parse, don't validate": once past the factory, validity is a type-level fact, not a hope.
3. Because the only path to a value runs through the factory, an invalid instance is unrepresentable at runtime as well as in the type system, so downstream code never re-checks.

```ts
import {z} from 'zod';
const emailSchema = z.email();

interface User {readonly id: UserId; readonly email: string}

function createUser(id: string, rawEmail: string): User {
  const email = emailSchema.parse(rawEmail);  // throws on invalid input
  return {id: toUserId(id), email};           // constructor-free: parse-generate then assign (3.9, 6.11)
}
```

**Enforcement:** zod at boundaries (chapter 10); review rejects validation logic inside a constructor and `new` calls that bypass the factory for domain types.

### 6.9 — Make value objects frozen plain objects compared by a structural helper.

**Reasoning, step by step:**
1. A value object's identity is its content: two `Money` values with the same cents and currency are the same value. Model it as a plain object, frozen with `Object.freeze` so its fields cannot drift after creation. `Object.freeze` is shallow, so a value object holds only primitives (or `ReadonlyArray`/nested frozen values), never a mutable object that freezing would leave writable.
2. Structural sameness has no language operator here — `===` on objects is reference identity. Compare with a free `equals` helper that checks the fields, not an `.equals` method bolted onto the data. This preserves 6.3: the value stays plain data, the comparison stays a free function beside it.

```ts
interface Money {readonly cents: number; readonly currency: string}

const money = (cents: number, currency: string): Money =>
  Object.freeze({cents, currency});

const moneyEquals = (a: Money, b: Money): boolean =>
  a.cents === b.cents && a.currency === b.currency;
```

**Enforcement:** Review — value types are frozen at their factory; equality is a sibling helper, never a method on the record.

### 6.10 — Lift co-traveling optionals into the union.

**Reasoning, step by step:**
1. Two or more optional fields that are always present together, or always absent together, are a variant in disguise. The type permits every combination of present and absent, but the domain allows only some.
2. Each independent optional multiplies the state space again. Three co-traveling optionals admit eight shapes where the domain has two, and the extra six are latent bugs.
3. Lift the cluster into a discriminated union (6.5): one member for "present," one for "absent." The fields that travel together now live or die together, enforced by the compiler.

```ts
// bad — trackingId and shippedAt are a hidden "shipped" state
interface Shipment {destination: string; trackingId?: string; shippedAt?: Date}

// good — the cluster becomes a variant
type Shipment =
  | {status: 'preparing'; destination: string}
  | {status: 'shipped'; destination: string; trackingId: string; shippedAt: Date};
```

**Enforcement:** Review trigger — a second optional field added to a shape prompts the question "do these co-travel?"; if yes, model a union.

### 6.11 — Brand IDs at domain edges so they cannot be interchanged.

**Reasoning, step by step:**
1. A bare `string` ID is assignable to any other `string` ID. `cancelOrder(userId)` compiles, and the swap surfaces only as a production incident.
2. A branded type intersects the primitive with a unique phantom tag (chapter 03's branded-types rule). `OrderId` and `UserId` become distinct types: passing one where the other is required is a compile error, while the runtime value stays a plain `string`. Apply the brand once, inside the parsing factory of 6.8 where the raw value enters the domain, so internal code never re-casts.

```ts
type OrderId = string & {readonly __brand: 'OrderId'};
type UserId = string & {readonly __brand: 'UserId'};

function cancelOrder(id: OrderId): void {/* ... */}

declare const userId: UserId;
cancelOrder(userId);  // compile error: UserId is not assignable to OrderId
```

**Enforcement:** Branding convention defined in chapter 03; the assignability error is the enforcement, with the cast confined to factories.

## Cross-references

- Branded primitives, `readonly` in public types, and tested type guards: [03-the-type-system.md](./03-the-type-system.md).
- `readonly` field defaults and immutable declarations: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- `invariant()` assertions and the function-size cap on factories: [05-functions.md](./05-functions.md).
- `Result` as a discriminated union and `Error` subclass hierarchies: [08-error-handling.md](./08-error-handling.md).
- zod at boundaries and accept-interfaces / return-concrete: [10-api-design.md](./10-api-design.md).
- Lifecycle classes, `using`/`Symbol.dispose`, and bounded pools: [13-resource-management.md](./13-resource-management.md).
