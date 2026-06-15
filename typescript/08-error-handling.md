# 08 — Error Handling

Errors are values, handled explicitly. TypeScript erases types, so a thrown value has no type the compiler tracks: `catch` always hands you `unknown`, and a signature never tells you what flies out. This chapter closes that gap. Model failures as typed `Error` subclasses with a mandatory `cause` chain, narrow every `catch`, and reach for an opt-in `Result` union where a failure is part of the contract. Programmer errors crash fast; operational errors are handled.

## What good looks like

```ts
/** Base for every payment failure. Carries a correlation id for the log trail. */
export class PaymentError extends Error {
  readonly correlationId: string;
  constructor(message: string, correlationId: string, options?: ErrorOptions) {
    super(message, options);
    this.correlationId = correlationId;
    this.name = new.target.name; // every subclass reports its own name
  }
}

export class CardDeclinedError extends PaymentError {
  readonly cardLast4: string;
  readonly declineCode: string;
  constructor(cardLast4: string, declineCode: string, correlationId: string) {
    super(`card ****${cardLast4} declined: ${declineCode}`, correlationId);
    this.cardLast4 = cardLast4;
    this.declineCode = declineCode;
  }
}

export class GatewayUnavailableError extends PaymentError {}

/** Boundary: wraps the gateway's failures, never lets a raw one leak upward. */
export async function chargeCard(card: Card, amount: Cents, correlationId: string): Promise<Receipt> {
  invariant(amount > 0, `amount must be positive, got ${amount}`); // 8.7 programmer error

  let response: GatewayResponse;
  try {
    response = await gateway.submit(card.token, amount);
  } catch (e: unknown) {
    throw new GatewayUnavailableError('payment gateway unreachable', correlationId, { cause: toError(e) });
  }

  switch (response.status) {
    case 'approved': return response.receipt;
    case 'declined': throw new CardDeclinedError(card.last4, response.declineCode, correlationId);
    default: return assertNever(response.status); // 8.9 exhaustive; a new status fails to compile
  }
}
```

This module throws rather than returning a `Result` (8.6), and the choice holds throughout. `PaymentError` is a typed hierarchy with context fields (8.1); `cause` carries the gateway failure forward (8.2); the `catch` narrows `unknown` through `toError` (8.3); the boundary wraps so the domain never sees a raw fault (8.5); `invariant` splits the programmer error from the operational ones (8.7); the `switch` is exhaustive (8.9). The calling HTTP handler logs once and maps each error type to a status code.

## Rules

### 8.1 — Define domain `Error` subclasses with typed context fields.

**Reasoning, step by step:**
1. A bare `throw new Error('payment failed')` gives a debugger a string and nothing else: no order id, no amount, no correlation id. The context died at the throw site.
2. Subclass `Error` per domain, with a root (`PaymentError`) and specific leaves (`CardDeclinedError`). The caller catches the root for generic handling or a leaf for precise recovery.
3. Carry the identifying inputs as `readonly` fields — ids, the offending input, the `correlationId` that ties the failure to a request. These survive serialization and show up in structured logs.
4. Set `this.name = new.target.name` in the base constructor so every subclass reports its own class name in stack traces without restating it. Keep the tree two levels deep; a five-level error hierarchy navigates no better than a two-level one.

```ts
export class LineItemInvalidError extends OrderError {
  readonly lineIndex: number;
  readonly quantity: number;
  constructor(orderId: string, lineIndex: number, quantity: number) {
    super(`order ${orderId}: line ${lineIndex} has invalid quantity ${quantity}`, orderId);
    this.lineIndex = lineIndex;
    this.quantity = quantity;
  }
}
```

**Enforcement:** review; domain failures extend `Error` and declare context fields, no bare `new Error`.

### 8.2 — Pass `cause` on every rethrow.

**Reasoning, step by step:**
1. When you catch one error and throw another, the original is the evidence. Drop it and the new stack trace starts at your rethrow; the actual fault is gone. A chain without `cause` is amnesia.
2. `Error` accepts `{ cause }` as its second argument (ES2022) and V8 prints the chain. Always pass it: `throw new PaymentDeclinedError(msg, { cause: e })`.
3. The cause must be an `Error`, so run the caught value through `toError` (8.3) first. Inner layers wrap with `cause`; the boundary error explains *what*, the cause explains *why*.

```ts
try {
  await db.insert(order);
} catch (e: unknown) {
  throw new OrderPersistFailedError(order.id, { cause: toError(e) }); // chain preserved
}
```

**Enforcement:** review; every wrap-and-rethrow passes `{ cause }`.

### 8.3 — Type every `catch` as `unknown` and narrow it.

**Reasoning, step by step:**
1. `throw` accepts any value, so a `catch` binding is `unknown` (`useUnknownInCatchVariables`, on in strict mode). Treating it as `Error` is a lie the compiler stopped catching.
2. Narrow before use. For the common "I need an `Error` to chain or log" case, funnel through one shared helper defined once and imported everywhere — and the funnel must never throw from inside a `catch`, so guard the stringify: `String(e)` itself throws on a null-prototype object or a value whose `toString`/`Symbol.toPrimitive` throws.
```ts
export function toError(e: unknown): Error {
  if (e instanceof Error) return e;
  try {
    return new Error(String(e));
  } catch {
    return new Error('non-stringifiable thrown value', { cause: e });
  }
}
```
3. For richer narrowing that distinguishes your own error types, use `instanceof` against the specific class — never duck-typing on `.message` or a string `.code`.

```ts
try {
  return parseConfig(raw);
} catch (e: unknown) {
  if (e instanceof SyntaxError) throw new ConfigInvalidError(path, { cause: e });
  throw toError(e); // anything else: normalize and let it fly
}
```

**Enforcement:** `useUnknownInCatchVariables` (strict); `no-explicit-any` blocks `catch (e: any)`; `toError` in a shared module.

### 8.4 — Never swallow an error.

**Reasoning, step by step:**
1. An error is information. An empty `catch {}`, or a `catch` that logs and continues as if nothing happened, discards it and ships corrupt state forward.
2. A `catch` block has exactly three honest endings: handle the failure (recover to a known-good state), wrap-and-rethrow with `cause` (8.2), or don't catch at all and let it fly. "Log and continue" is none of these; if you cannot recover, rethrow, and logging belongs at the boundary (8.5).
3. The rare deliberate ignore, a best-effort cleanup that may legitimately fail, is narrowed to the one expected error and carries a comment saying why the failure is tolerable.

```ts
// bad — silent swallow, corrupt state continues
try { await reserve(seat); } catch { /* ignored */ }

// good — tolerable failure, narrowed and explained
try {
  await tempFile.delete();
} catch (e: unknown) {
  if (!(e instanceof FileNotFoundError)) throw e; // already gone is fine; anything else is not
}
```

**Enforcement:** `no-empty` (forbids empty blocks); review for log-and-continue.

### 8.5 — Wrap at boundaries; log once, at the top.

**Reasoning, step by step:**
1. A `PrismaClientKnownRequestError` from the repository must not surface in domain logic — the domain does not know about the ORM. Each layer translates the one below into its own vocabulary, wrapping with `cause` so nothing is lost.
2. Logging is a boundary concern. If every layer logs the same failure, one error becomes five log lines and the real one drowns. Inner layers add context to the error object; they do not log.
3. The outermost boundary (HTTP handler, queue consumer, CLI entry) logs once with the full `cause` chain and the `correlationId`, then decides the representation: a 500 with an opaque id, a non-zero exit. A typed hierarchy (8.1) makes the error-to-response mapping a `switch`, not a cascade of `instanceof`.

```ts
export async function handleCharge(req: Request): Promise<Response> {
  try {
    return Response.json(await chargeCard(req.card, req.amount, req.correlationId));
  } catch (e: unknown) {
    logger.error('charge failed', { err: e, correlationId: req.correlationId }); // logged once, here
    if (e instanceof CardDeclinedError) return Response.json({ code: 'declined' }, { status: 402 });
    return Response.json({ correlationId: req.correlationId }, { status: 500 });
  }
}
```

**Enforcement:** review; logging calls in `catch` blocks live only at boundary modules.

### 8.6 — Use the `Result` union opt-in per module; never mix it with throwing.

**Reasoning, step by step:**
1. Where failure is *expected* and part of the contract (a parser, a payment pipeline, anything whose every failure mode must appear in the signature), make it a value the caller cannot ignore. Define the union once, frozen, with an `ok` discriminant — `Object.freeze` is shallow, so it locks the wrapper's `ok`/`value`/`error` slots while the payload's own immutability stays the payload's concern ([chapter 03](./03-the-type-system.md) §3.10):
```ts
export interface Ok<T> { readonly ok: true; readonly value: T; }
export interface Err<E> { readonly ok: false; readonly error: E; }
export type Result<T, E> = Ok<T> | Err<E>;

export const ok = <T>(value: T): Ok<T> => Object.freeze({ ok: true, value });
export const err = <E>(error: E): Err<E> => Object.freeze({ ok: false, error });
```
2. The `ok` discriminant drives exhaustive narrowing: `if (r.ok)` gives the compiler `value`, the `else` gives `error`. No unwrap can skip the failure branch.
3. A module either throws or returns `Result`, never both. Mixing forces the caller to narrow the union *and* wrap a `try` around it — the worst of each style. Pick one per module and hold it.
4. The transition happens at a module boundary: a throwing dependency becomes a `Result` at the adapter that calls it, and a `Result` is unwrapped (or rethrown) at the edge of the module that produced it. (This ports the Python guide's [§8.11 Result discipline](../python/08-error-handling.md).)

```ts
function parseAmount(raw: string): Result<Cents, 'empty' | 'not-a-number' | 'negative'> {
  if (raw.trim() === '') return err('empty');
  const n = Number(raw);
  if (Number.isNaN(n)) return err('not-a-number');
  return n < 0 ? err('negative') : ok(n as Cents);
}

const parsed = parseAmount(input);
if (!parsed.ok) return reject(parsed.error); // failure branch cannot be skipped
charge(parsed.value);
```

**Enforcement:** review; one error style per module, `Result` shapes frozen and `readonly`.

### 8.7 — Separate programmer errors from operational errors.

**Reasoning, step by step:**
1. A programmer error is a bug: a violated precondition, an unreachable branch reached, an impossible state. Retrying cannot fix it — the code is wrong. It must crash loudly and close to the fault so the bug is found.
2. An operational error is an expected failure of a correct program: a declined card, a timed-out request, a missing file. It is part of the contract and must be handled as a typed `Error` (8.1) or a `Result` (8.6).
3. Assert programmer errors with `invariant` (see [chapter 05](./05-functions.md)); it throws on a false condition and narrows the type for the compiler, and an impossible `switch` branch is `assertNever`. Never demote a bug to a handled error; that hides it. This is the Tiger split: assertions that crash for your mistakes, values that are handled for the world's expected failures.

```ts
function applyDiscount(price: Cents, pct: number): Cents {
  invariant(pct >= 0 && pct <= 100, `pct out of range: ${pct}`); // bug if violated — crash
  return (price * (100 - pct)) / 100 as Cents;
}
// an out-of-stock item is operational: a typed error or a Result, never an invariant.
```

**Enforcement:** review; `invariant`/`assertNever` for bugs, typed errors or `Result` for expected failures.

### 8.8 — Put context in error messages, never secrets.

**Reasoning, step by step:**
1. `'validation failed'` is useless. `` `order ${id}: line ${i} has negative quantity ${qty}` `` is debuggable. Include the identifying inputs the reader cannot otherwise see, not just the symptom. A message is a public API: it travels into logs, stack traces, alerting, and sometimes a user-facing surface.
2. Never interpolate secrets — tokens, credentials, API keys, full PANs, raw PII. Mask to the minimum identifying fragment: `card ****1234`, not the full number. The masking rules in [security.md](../security.md) apply.
3. Put structured fields on the error object (8.1), not only in the message string, so a log aggregator can index them without parsing prose.

```ts
// good — identifying context, secret masked
throw new CardDeclinedError(card.last4, code, correlationId);
// bad — leaks the full token into every log that touches this error
throw new Error(`charge failed for token ${card.fullToken}`);
```

**Enforcement:** review; secret-scanning in CI; masking helpers from `security.md`.

### 8.9 — Don't use exceptions for control flow.

**Reasoning, step by step:**
1. An exception is for the exceptional. Using `throw`/`catch` to signal an ordinary "not found" or "no" conflates a result with a failure and pays a stack-trace capture for a routine branch.
2. Absence is `undefined`. A lookup that may miss returns `T | undefined`, and the caller narrows with `?.`/`??` ([chapter 07](./07-typescript-idioms.md)). The may-miss variant is `get<Noun>(): T | undefined`, documented by its signature ([chapter 02](./02-naming-conventions.md)); the asserting variant keeps the same verb and throws when absent. (`Array.prototype.find` below is the unrelated array method, not this resource verb.)
3. A yes/no question returns a `boolean`. `hasAccess(user): boolean` returns `false` for "no"; it does not throw an `AccessDeniedError` the caller wraps a `try` around to read. Reserve `throw` for genuine failures (8.7) and `Result` (8.6) for expected-failure contracts; neither replaces a predicate or an `undefined`.

```ts
const user = users.find((u) => u.id === id); // good — absence is a value: User | undefined
if (user === undefined) return notFound();

try { return getUserOrThrow(id); } catch { return notFound(); } // bad — control flow via exceptions
```

**Enforcement:** review; `T | undefined`/`boolean` for routine branches, no `try` around a throwing lookup used as a predicate.

### 8.10 — Document the failure modes of every public API.

**Reasoning, step by step:**
1. A thrown error is invisible in the signature — `function chargeCard(...): Promise<Receipt>` admits nothing about `CardDeclinedError`. An undocumented throw is hidden behaviour the caller discovers in production.
2. A public function makes its failures part of its contract one of two ways: a `@throws` TSDoc tag per error type the caller might reasonably catch, or a `Result` return type (8.6) that puts the failures in the type itself.
3. Prefer the `Result` signature where it fits; it cannot drift from the implementation the way a comment can. With `@throws`, list only the errors a caller would act on, not every `Error` that could theoretically escape, and keep it in step with the code — a new throw is a contract change (see [chapter 14](./14-documentation.md)).

```ts
/**
 * Charges a card.
 * @throws {CardDeclinedError} the issuer declined the charge — retry with another card.
 * @throws {GatewayUnavailableError} the gateway was unreachable — safe to retry.
 */
export function chargeCard(card: Card, amount: Cents, correlationId: string): Promise<Receipt>;
```

**Enforcement:** review; `@throws` on throwing publics or a `Result` signature, checked against the implementation.

### 8.11 — Report fan-out failures with `AggregateError`.

**Reasoning, step by step:**
1. When N independent operations run together and several fail, reporting only the first throws away the other N−1. A batch that fails on items 3, 7, and 12 must say so, all of them, not stop at item 3.
2. Run the operations with `Promise.allSettled` (not `Promise.all`, which rejects on the first failure and abandons the rest), then collect the rejections.
3. If any failed, throw a single `AggregateError` holding every cause, with a message stating how many of how many failed. This pairs with the bounded fan-out rules in [chapter 09](./09-concurrency.md): the concurrency stays bounded, and aggregation is how its partial failures surface.

```ts
const results = await Promise.allSettled(orders.map((o) => persist(o)));
const failures = results.flatMap((r) => (r.status === 'rejected' ? [toError(r.reason)] : []));
if (failures.length > 0) {
  throw new AggregateError(failures, `${failures.length}/${orders.length} orders failed to persist`);
}
```

**Enforcement:** review; `Promise.allSettled` + `AggregateError` for independent fan-out, not first-failure-wins.

## Cross-references

- `invariant`, `assertNever`, and assertion density: [chapter 05](./05-functions.md).
- Discriminated unions and making illegal states unrepresentable: [chapter 06](./06-classes-and-data-modeling.md).
- `?.`/`??` for absence, exhaustive narrowing: [chapter 07](./07-typescript-idioms.md).
- `Promise.allSettled`, `AbortSignal.timeout()`, bounded fan-out: [chapter 09](./09-concurrency.md).
- `@throws` documentation and contract changes: [chapter 14](./14-documentation.md).
- Secret masking in messages and logs: [security guide](../security.md).
