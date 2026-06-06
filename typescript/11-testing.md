# 11 — Testing

Correctness is this guide's first value, and a test suite is its only proof. The type system is the first test suite (chapter [03](./03-the-type-system.md)); the runtime tests below are the second, and the two are not interchangeable — a type regression passes every runtime test, a logic regression passes every type check. This chapter covers both, plus property-based tests that fuzz invariants no example ever reaches. Tests are code that runs on every push: they earn their keep by failing meaningfully on real regressions and passing quietly otherwise.

## What good looks like

```ts
import {describe, it, expect, expectTypeOf} from 'vitest';
import {fc, test as propTest} from '@fast-check/vitest';
import {encodeCursor, decodeCursor, type Cursor} from './cursor.js';

describe('cursor codec', () => {
  it('round-trips a known cursor through encode then decode', () => {
    const cursor: Cursor = {offset: 40, pageSize: 20};

    const restored = decodeCursor(encodeCursor(cursor));

    expect(restored).toEqual(cursor);            // positive space: the value survives
    expect(encodeCursor(cursor)).not.toContain(':'); // negative space: opaque, no delimiter leaks (11.9)
  });

  propTest.prop([fc.record({offset: fc.nat(), pageSize: fc.integer({min: 1, max: 100})})])(
    'decode is the inverse of encode for every valid cursor', // round-trip property (11.5)
    cursor => {
      expect(decodeCursor(encodeCursor(cursor))).toEqual(cursor);
    },
  );

  it('exposes Cursor as a readonly type, not a mutable one', () => {
    expectTypeOf<Cursor>().toEqualTypeOf<{readonly offset: number; readonly pageSize: number}>();
  }); // type-level regression guard (11.6)
});
```

One module, tested three ways. The example-based `it` names the behaviour under its condition (11.2) and asserts both positive and negative space (11.9). The `fast-check` property proves the round-trip law over generated input, not one lucky case (11.5). The `expectTypeOf` test fails the build if `Cursor` ever loses its `readonly`, a correctness bug no runtime assertion can see (11.6). Imports are explicit — no globals (11.1) — and nothing touches a clock, a socket, or the filesystem, so the suite is deterministic (11.8).

## Rules

### 11.1 — Run Vitest with globals off; colocate unit tests, isolate integration tests.

**Reasoning, step by step:**
1. Vitest is the runner: native ESM, TypeScript without a separate transform, and the `expectTypeOf` and `@fast-check/vitest` integrations the rest of this chapter depends on. It is not a per-project choice. Globals stay off (`globals: false`): `import {describe, it, expect} from 'vitest'` is explicit about where the symbols come from, the same stance chapter [12](./12-module-organization.md) takes on every other import; an auto-injected `it` is ambient magic, banned by root rule 2.
2. Unit tests sit beside their subject as `foo.test.ts`. The test is the first consumer of `foo.ts` (11.12), and colocation keeps them moving together — rename, move, or delete the module and its test follows. Integration and end-to-end tests cross process or network boundaries, run on a different cadence, and live under a top-level `tests/`, not next to a single module.

**Worked example:**
```ts
import {describe, it, expect} from 'vitest'; // explicit; vitest config sets globals: false
// src/cursor.ts        → src/cursor.test.ts   (unit, colocated)
// tests/api.e2e.test.ts                        (integration, isolated)
```
**Enforcement:** `vitest.config.ts` with `globals: false`; `no-undef` flags an unimported `describe`/`it`/`expect`; review of test placement.

### 11.2 — Arrange, act, assert; one behaviour per test; name the behaviour under its condition.

**Reasoning, step by step:**
1. Every test has three sections — set up the world, run the one operation under test, check the outcome — separated by blank lines. The single act is what the test is about; everything above it is arrange, everything below is assert. One behaviour per test: a test that asserts five unrelated facts fails at the first and hides the rest; split it, and each failure names exactly what broke. Multiple assertions on the *same* behaviour are fine (11.9).
2. The name is the failure message you read at 2 am, and it surfaces in CI logs, IDE trees, and flakiness dashboards — public surface, treated with the care of an exported identifier (chapter [02](./02-naming-conventions.md)). `returns undefined when the id is unknown` tells you what broke and under which condition; `test cursor 2` tells you nothing. Shape: `<verb-phrase> when <condition>`, reading aloud as English.

**Worked example:**
```ts
it('returns undefined when the id is unknown', () => {
  const repo = new FakeUserRepository();   // arrange

  const found = repo.find(UserId('absent')); // act

  expect(found).toBeUndefined();             // assert
});
```
**Enforcement:** review for AAA shape and behavioural names; one logical behaviour per `it`.

### 11.3 — Fake your own interfaces; reserve `vi.mock` for true externals.

**Reasoning, step by step:**
1. For code you own, write a fake: a hand-rolled, in-memory implementation of your own interface. A `FakeUserRepository` backed by a `Map` is fully behavioural (it stores, finds, and deletes for real), so the test exercises real call paths instead of restating the implementation in mock expectations. A mock that returns canned values passes precisely when the code is wrong in the way you mocked it; the fake has no such blind spot. This is the same stance the Kotlin and Python guides take: real implementations wherever possible, doubles only at the genuine seam.
2. `vi.mock` is for true externals only: a third-party SDK, a module with import-time side effects you cannot otherwise sever. The seam is the boundary of your own code; mock across it, never inside it. Name a double for what it is (`FakeUserRepository`, `StubClock`), never `MockClock` for a hand-rolled fake — the name lies to the next reader.

**Worked example:**
```ts
class FakeUserRepository implements UserRepository {
  private readonly byId = new Map<UserId, User>();
  save(user: User): void {this.byId.set(user.id, user);}
  find(id: UserId): User | undefined {return this.byId.get(id);} // real behaviour, in memory
}
```
**Enforcement:** review; `vi.mock` calls justified against an external boundary, not an owned interface.

### 11.4 — Fake HTTP at the network with MSW; never monkey-patch `fetch`.

**Reasoning, step by step:**
1. Reassigning `globalThis.fetch` to a mock tests your wiring against your own assumptions, not against HTTP. It skips URL construction, headers, status handling, and serialization — exactly the layer most likely to be wrong.
2. Mock Service Worker (MSW) intercepts at the network layer, so the real `fetch` runs and your code exercises the genuine request/response path against a handler you control. The fake is the *server*, not the client. The handlers are reusable across unit, integration, and component tests, and the same definitions can back a dev server — one source of truth for what the upstream returns.

**Worked example:**
```ts
const server = setupServer(
  http.get('https://api.example.com/users/:id', () =>
    HttpResponse.json({id: 'u1', email: 'a@b.c'})),
);
// real fetch runs; the network is faked, not the client
```
**Enforcement:** review; no assignment to `globalThis.fetch`; HTTP doubles go through MSW.

### 11.5 — Property-based tests are mandatory for codecs, parsers, serializers, and invariant-bearing functions.

**Reasoning, step by step:**
1. An example test proves one input behaves; a property test proves a law holds across thousands of generated inputs, including the empty, the maximal, and the adversarial edge a human never thinks to type. For anything that transforms or validates data, that breadth is the difference between "works on my three cases" and "works."
2. Use `fast-check`. The canonical properties are few and reusable: **round-trip** (`decode(encode(x))` equals `x`) for every codec and serializer; **idempotence** (`f(f(x))` equals `f(x)`) for normalizers and dedupers; **order-insensitivity** (a sort or merge yields the same result however its input is permuted); **bounds** (the output of a clamp or a parser always lands in its declared range). Each maps directly to an `invariant` the production code already asserts (chapter [05](./05-functions.md)). This is mandatory, not optional, for the four categories named — precisely the functions whose failure modes hide in the input space, and where the type system cannot help: a `parse(raw: string): Cursor` is well-typed even when its logic is wrong.

**Worked example:**
```ts
propTest.prop([fc.array(fc.integer())])('sorting is order-insensitive', xs => {
  const shuffled = [...xs].reverse();
  expect(sort(xs)).toEqual(sort(shuffled)); // same multiset, same result
});
```
**Enforcement:** review; codecs, parsers, serializers, and invariant-bearing functions ship with a `fast-check` property covering at least one canonical law.

### 11.6 — Type-level tests are mandatory for public generics and conditional types.

**Reasoning, step by step:**
1. A public generic or conditional type is an API surface, and it can regress silently: a refactor widens an inferred return, a conditional collapses to `never`, a mapped type quietly drops `readonly`. No runtime test sees any of it, because the broken type still compiles and still runs. A type regression is a correctness bug, and the only test that catches it lives in the type space.
2. Assert the type with `expectTypeOf`. Pin what the generic infers, that a conditional resolves to the branch it should, and that the negative case is rejected — `expectTypeOf(badCall).toBeNever()` or a `@ts-expect-error` line (chapter [03](./03-the-type-system.md) §3.3) proving misuse does not compile. The type-level test asserts negative space too (11.9). This is mandatory for every exported generic and conditional type — the constructs whose correctness the compiler enforces for callers but not for *you*; the test is how you hold yourself to the contract you publish.

**Worked example:**
```ts
expectTypeOf<Awaited<Promise<User>>>().toEqualTypeOf<User>();    // conditional resolves correctly
expectTypeOf(parseUser).returns.toEqualTypeOf<ParseResult>();   // public generic's inferred return is pinned
```
**Enforcement:** review; exported generics and conditional types ship with an `expectTypeOf` test; `vitest --typecheck` runs them in CI.

### 11.7 — Give every custom type guard a truth-table test.

**Reasoning, step by step:**
1. A custom guard returning `x is T` is the one narrowing the compiler cannot verify (chapter [03](./03-the-type-system.md) §3.8): when it returns `true`, the compiler *believes* the value is `T` and stops checking. A wrong guard is a type-system lie that spreads downstream as silent, `any`-grade unsoundness — every narrowing it feeds is now unsound.
2. The test is a truth table: every positive case the guard must accept, and every negative case it must reject — wrong type, missing field, `null`, the empty object. Negative space is not optional here; a guard that returns `true` too often is exactly the bug, and only the negative rows catch it. This applies to the guard alone, not to `typeof`/`instanceof`/`in` narrowing, which the compiler implements and therefore needs no test (chapter [03](./03-the-type-system.md) §3.7).

**Worked example:**
```ts
it('isUser accepts a complete user and rejects malformed input', () => {
  expect(isUser({id: 'u1', email: 'a@b.c'})).toBe(true);  // positive row
  expect(isUser({id: 'u1'})).toBe(false);                 // missing field
  expect(isUser(null)).toBe(false);                       // negative space, mandatory
});
```
**Enforcement:** review requires a colocated truth-table test for every `x is T` guard; ports chapter [03](./03-the-type-system.md) §3.8.

### 11.8 — Make every test deterministic; control time, randomness, and I/O.

**Reasoning, step by step:**
1. A test that depends on the wall clock, an unseeded random source, the live network, or the real filesystem is a flake waiting to happen: it passes locally, fails on slow CI, and erodes trust until a red build means nothing. Determinism is the precondition for a suite anyone believes, and it reduces to one line: no real network, clock, or filesystem in a unit test.
2. Virtualize time with `vi.useFakeTimers()` and advance it explicitly with `vi.advanceTimersByTime(ms)` — never `await` a real `setTimeout` to "give it a moment." Inject clocks and ID generators rather than reading `Date.now()` or `crypto.randomUUID()` from the wild, the same injection the concurrency chapter relies on for testable timeouts (chapter [09](./09-concurrency.md)).
3. `fast-check` is seeded, and on failure prints the seed and the shrunk counterexample — log that seed in CI, or the shrink that found the bug is lost. HTTP goes through MSW (11.4), time through fake timers, persistence through a fake (11.3).

**Worked example:**
```ts
vi.useFakeTimers();
const promise = withTimeout(slowCall(), 1000);
vi.advanceTimersByTime(1000); // deterministic; no real wall-clock wait
await expect(promise).rejects.toThrow(TimeoutError);
```
**Enforcement:** `vi.useFakeTimers` for time-dependent tests; seeded `fast-check` with the seed logged; review forbids real network/clock/fs in unit tests.

### 11.9 — Assert positive and negative space.

**Reasoning, step by step:**
1. The assertion-density discipline (root rule 8: two-plus per function, positive *and* negative space) applies to tests, because a test is code and its assertions are its reason to exist. A test with one assertion checks one fact; often correct, but a complex outcome needs more. Positive space is what must have happened; negative space is what must *not* have. "The user was created" means `repo.find(id)` returns the user (positive) *and* no second record was written and no error was logged (negative). The bug usually hides in the space you forgot to assert — the extra write, the leaked delimiter, the swallowed error.
2. The pair-assertion is the sharpest form: verify the same property two independent ways. After a sort, assert the length is unchanged (nothing lost) *and* the order invariant holds (correctly arranged). Do not pad with unrelated assertions to hit a count; two assertions on one behaviour beat five across many.

**Worked example:**
```ts
it('createUser persists exactly one record', () => {
  service.createUser({email: 'a@b.c'});

  expect(repo.count()).toBe(1);                 // positive: it was written
  expect(repo.find(UserId('a@b.c'))).toBeDefined();
  expect(auditLog.errors).toHaveLength(0);      // negative: nothing went wrong
});
```
**Enforcement:** review; tests of non-trivial behaviour assert both what happened and what must not have.

### 11.10 — Share no mutable fixtures; let no test depend on another.

**Reasoning, step by step:**
1. Every test must run alone, in any order, in parallel — Vitest parallelizes by default, so any cross-test coupling is already a future flake. A test that passes only after another has run is not a test, it is a fragment of one. A shared mutable fixture (a module-level array a test pushes into, a `FakeRepository` constructed once at the top and reused) leaks state between tests: one test's write becomes another's surprise, and a failure in test A masks the real cause in test B.
2. Build fixtures fresh — in the test, in `beforeEach`, or from a factory function — so each test starts from the same clean world.
3. Factory functions with defaults keep fresh setup terse: `makeUser({isActive: false})` returns a brand-new object each call, overriding only what the test cares about. Immutable shared data (a frozen constant, a parsed schema) is safe to hoist; mutable state never is.

**Worked example:**
```ts
function makeUser(overrides: Partial<User> = {}): User {
  return {id: UserId('u-1'), email: 'a@b.c', isActive: true, ...overrides}; // fresh each call
}
beforeEach(() => {repo = new FakeUserRepository();}); // clean world per test
```
**Enforcement:** review; no module-level mutable fixtures; setup in `beforeEach` or a factory, never hoisted shared state.

### 11.11 — Run mutation testing nightly; report coverage, never target it.

**Reasoning, step by step:**
1. Line coverage measures which lines ran, not whether an assertion would have caught a bug on them. A suite can execute every line and assert nothing meaningful — 100% coverage with 0% of the bugs caught. Targeting the percentage optimizes the metric and corrupts the suite: tests get written to touch lines, not to verify behaviour.
2. Mutation testing is the honest metric. Stryker introduces small faults (flips a `<` to `<=`, drops a branch, swaps a return) and reruns the suite; a mutant that survives is a behaviour no test actually checks, a precise to-do list of missing assertions that coverage only pretends to be. Runs are expensive, so schedule them nightly: run them, read the survivors, close the gaps. Coverage is still *reported*, a useful floor and a trend, but never the goal, and a build never passes or fails on a coverage number alone. The percentage answers "what ran"; mutation answers "what was checked," and only the second is correctness.

**Worked example:**
```jsonc
// stryker.config.jsonc — nightly CI job, not per-commit
{"testRunner": "vitest", "reporters": ["html", "clear-text"], "mutate": ["src/**/*.ts"]}
```
**Enforcement:** recommended nightly Stryker run; coverage reported as a trend, never set as a pass/fail target.

### 11.12 — Treat the test as the first caller; an unergonomic test is an API smell.

**Reasoning, step by step:**
1. The test is the first real consumer of the code under test, written before any other caller exists. What is awkward to test is awkward to call: a function that needs ten lines of setup, six mocks, and a global poked into place is telling you its dependencies are hidden and its surface is too wide.
2. Listen to that signal instead of papering over it with elaborate test scaffolding. The fix is in the production code, not the test: accept dependencies as parameters so a fake drops in (11.3), narrow the surface, split the function that does four things. This is the call-site principle of chapter [02](./02-naming-conventions.md) and the API-symmetry principle of chapter [10](./10-api-design.md), observed from the consumer's seat: the test is where a bad API first hurts. The corollary: when a test reads cleanly (construct, call, assert, with a fake passed in one line), the API is probably right. Test ergonomics are a design review you get free on every change.

**Worked example:**
```ts
// smell: untestable without global state — the API hides its clock
function isExpired(token: Token): boolean {return Date.now() > token.expiresAt;}
// fix: the dependency is a parameter, so the test passes a fixed clock in one line (11.8)
function isExpired(token: Token, now: number): boolean {return now > token.expiresAt;}
```
**Enforcement:** review; tests needing heavy scaffolding flag an API problem, fixed in the production code, not the test.

## Cross-references

- The type system as the first test suite, custom guards, branded primitives: chapter [03](./03-the-type-system.md) (§3.8 truth-table tests, §3.7 narrowing tools). `invariant`, `assertNever`, and the assertion-density discipline: chapter [05](./05-functions.md).
- Discriminated unions and making illegal states unrepresentable: chapter [06](./06-classes-and-data-modeling.md). `Result` unions and one error style per module: chapter [08](./08-error-handling.md).
- Injected clocks, `AbortSignal.timeout()`, and testable timeouts: chapter [09](./09-concurrency.md).
- Naming for the call site and API symmetry, the principles a hard-to-test API violates: chapters [02](./02-naming-conventions.md) and [10](./10-api-design.md).
