## TypeScript Code Style

Binding TypeScript rules for all dexpace projects; extends [Google's TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html). Target **TypeScript 5.8+**, ESM-only, `gts` for tooling. Prioritize **correctness**, **explicitness**, **simplicity** — never cleverness, never `any`, never abstraction for its own sake.

This guide is platform-agnostic. It covers the TypeScript language, its type system, and runtime-neutral idioms. Runtime-specific rules live in the companion guides: server concerns in [typescript-bun](../typescript-bun/), UI concerns in [typescript-react](../typescript-react/).

### Authorities

This guide extends and defers to an ordered chain. Where they conflict, the higher authority wins:

1. **[Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)** — canonical. The official source of truth for casing, syntax, and taste. Where our guidance collides with it, **the official guide wins**, except for the deliberate deviations recorded in the ledger below.
2. **[ts.dev/style](https://ts.dev/style/)** — the community adaptation of the Google guide. It **fills the gaps** the official guide leaves open. Defer to it only where Google is silent.
3. **[gts v7](https://github.com/google/gts)** — the tooling embodiment of the above. Its Prettier defaults and lint baseline are final; formatting is a non-discussion.
4. **This guide's overlay** — the dexpace layer on top: the Tiger Style discipline (assertion density, bounded everything, no recursion, zero debt), the 70-line function cap, the erasable-syntax stance (emits no runtime code), and `Result`-as-discriminated-union discipline.

### Values

**Correctness > performance > developer experience.** This is the root README's ordering; this guide refines "developer experience" into **simplicity > expressiveness**. When rules conflict, that order decides.

- **Correctness** comes first because a fast, simple, expressive program that computes the wrong answer is worthless. It implies every feasible kind of testing: unit, property-based, type-level, mutation, component, e2e. The type system is the first test suite, and most of chapter 03 is about not lying to it.
- **Performance** comes before simplicity because the right architecture is chosen once, at design time, and is expensive to retrofit. It is designed in, not bolted on. Work with the grain of V8. Optimize the slowest resource first: network > disk > memory > CPU.
- **Simplicity** is the simplest approach that accomplishes the goal: no abstraction for its own sake, no cleverness. When two correct, fast-enough designs differ, the simpler one wins.
- **Expressiveness** comes last because clarity is worth nothing if the code is wrong, slow, or needlessly complex; rule 2 makes it concrete.

All four are held to one standard of elegant, well-structured code, enforced through the chapters' rules and exemplars rather than aspired to as a rank in the priority order.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Formatting & Tooling](./01-formatting-and-tooling.md) | gts only; ESLint overlay + tsconfig flags; TS ≥ 5.8; bun install; pre-commit `gts lint`; 70-line cap, `max-depth 3`, `max-params 3` |
| 02 | [Naming Conventions](./02-naming-conventions.md) | Google casing, kebab-case files, client verb taxonomy, no `I` prefix, no `Async` suffix, names designed for the call site |
| 03 | [The Type System](./03-the-type-system.md) | `any` banned (`unknown` + narrowing), `as` needs a why, `satisfies`/guards/parse, absence = `undefined`, tested type guards, branded primitives, `readonly` |
| 04 | [Variables & Declarations](./04-variables-and-declarations.md) | `const` default, `let` justified, `var` banned, non-null `!` banned outside bridges, `as const` config |
| 05 | [Functions](./05-functions.md) | 70-line cap, one level of abstraction, guard clauses, step-down rule, options object for ≥3 params, `invariant(): asserts`, 2+ assertions |
| 06 | [Classes & Data Modeling](./06-classes-and-data-modeling.md) | Make illegal states unrepresentable; `interface` + free functions; classes only for lifecycle; discriminated unions; parse, don't validate |
| 07 | [TypeScript Idioms](./07-typescript-idioms.md) | `satisfies`, `as const`, `?.`/`??` (no `\|\|` defaults), pipeline `map`/`filter`/`reduce`, `Map`/`Set`, `structuredClone`, name the steps |
| 08 | [Error Handling](./08-error-handling.md) | `Error` subclasses, mandatory `cause` chaining, `catch (e: unknown)` + narrow, opt-in `Result` unions, programmer vs operational errors |
| 09 | [Concurrency & Async](./09-concurrency.md) | async/await only, `no-floating-promises`, `AbortSignal.timeout()`, bounded fan-out via a worker-pool helper, documented races |
| 10 | [API Design](./10-api-design.md) | Named exports only, `index.ts` is the contract, accept interfaces / return concrete, zod at boundaries, `@deprecated` + semver, API symmetry |
| 11 | [Testing](./11-testing.md) | bun test, colocated `*.test.ts`, fast-check property tests, `expectTypeOf` type-level, fakes over mocks, MSW, determinism |
| 12 | [Module Organization](./12-module-organization.md) | ESM only, `import type` discipline, feature folders, barrels at the boundary only, `madge --circular`, no module side effects |
| 13 | [Resource Management](./13-resource-management.md) | `using`/`await using`, `Symbol.dispose`, AbortController as lifecycle handle, bounded pools/queues/caches |
| 14 | [Documentation](./14-documentation.md) | TSDoc on public API, never restate types, why-comments, `@example` on non-obvious publics |
| 15 | [Performance](./15-performance.md) | V8 monomorphism, no `delete`/sparse arrays, allocation hygiene, serial-await elimination, measure first, network > disk > memory > CPU |

---

## Cross-Cutting Concerns

Security, performance, and git practices are covered in the [root-level code style guide](../README.md). The cross-cutting docs are language-agnostic; this guide adapts them to TypeScript. Runtime-specific concerns are in [typescript-bun](../typescript-bun/) and [typescript-react](../typescript-react/).

## The 12 Rules in TypeScript

The root README defines twelve rules for every dexpace project. Here they are in TypeScript's vocabulary.

1. **Data and functions, not objects.** Model state as plain objects typed by `interface`; group behaviour into free functions and small interfaces. Reserve `class` for stateful lifecycle resources — things you open and close. No inheritance for code reuse.
2. **Explicit over implicit.** Code says what it does at the call site and does nothing it did not say. No `any`. No decorator or DI magic. No type-space syntax that emits runtime code. Every dependency is a parameter, visible in the signature. Library options follow their documented defaults; callers pass only what differs, supplied through an options object.
3. **Immutable by default.** `const` over `let`, `readonly` fields, `ReadonlyArray<T>` in public signatures, frozen config. Update by spreading into a new value, never by mutation. Mutability is the choice you have to type — that's the right way around.
4. **Errors are values, handled explicitly.** Typed `Error` subclasses per domain, with mandatory `cause` chaining and context fields on rethrow. Opt-in `Result<T, E>` discriminated unions where a failure is expected, never mixed within a module. `catch (e: unknown)`, then narrow. No swallowing; `no-floating-promises` makes a dropped promise a lint error.
5. **Composition over inheritance.** `extends` is for `Error` hierarchies and nothing else. Closed polymorphism is a discriminated union; code reuse is delegation. Small interfaces composed together, never a deep class tree.
6. **Transform, don't mutate.** Build pipelines from `map`/`filter`/`reduce`; reach for `for…of` only for effects or early exit. Functions take input and return new output. State changes are explicit, localized, and named — chains past ~3 stages get named intermediate `const`s.
7. **Always say why.** TSDoc comments explain reasoning, not mechanics; `@example` on non-obvious public API. Enforcement notes in the guides name their rule. If you can't say why a line exists, question whether it should.
8. **Assert aggressively.** An `invariant(cond, msg): asserts cond` helper backs this — assertions narrow types, so a runtime check also informs the compiler. Minimum two per function on average: preconditions at entry, postconditions at exit. Split compound assertions; assert positive and negative space.
9. **Limits on everything.** Functions cap at 70 lines (lint-enforced); nesting at `max-depth 3`, aiming for two. Bound every loop, queue, retry, pool, cache, and fan-out. Timeouts are mandatory on external I/O via `AbortSignal.timeout()`. No recursion in library code — all execution provably bounded.
10. **Small functions, breathing room.** Aim for 10–30 lines, one level of abstraction each. Guard clauses first so the happy path stays flush left. Separate logical sections with blank lines — whitespace is free, cramped code is unreadable.
11. **Performance from the outset.** Design-time is when 1000× improvements are cheap. Work with the grain of V8 — stable object shapes for monomorphism. Batch over serial `await`s. Optimize the slowest resource first: network > disk > memory > CPU.
12. **Zero technical debt.** What exists meets the design goals. **Perfection over technical debt — debt never gets paid.** Do it right the first time; the second chance may never come.

## Deviations from Upstream

This guide takes Google's TypeScript Style Guide as canonical. Three entries below are genuine deviations — places Google takes a position we override (banning `enum` and constructor parameter properties, and kebab-case files where Google specifies snake_case); the fourth, the 70-line function cap, is an addition Google does not address. Each is recorded so it can be revisited surgically.

| Rule | Upstream position | Our position | Why |
|---|---|---|---|
| `enum` | Google allows (bans only `const enum`) | Banned — use literal unions or `as const` maps | Hidden runtime emit (numeric enums even emit reverse-lookup objects); JSON-boundary friction (string enums are nominal, can't take `JSON.parse` output without a cast); platform direction (native type-stripping and `isolatedModules` can't execute it). See [03-the-type-system.md](./03-the-type-system.md). |
| Constructor parameter properties | Google allows | Banned — declare and assign explicitly | Hidden field declaration and assignment buried inside a constructor signature; "no hidden behaviour" by definition. See [03-the-type-system.md](./03-the-type-system.md). |
| Function size | No upstream cap | 70-line hard cap, lint-enforced | Owner decision; Tiger Style discipline; deliberately set at Go's level, not scaled down for TS. See [05-functions.md](./05-functions.md). |
| File naming | Google: `snake_case` files | kebab-case files | ecosystem norm, case-sensitivity-safe; See [./02-naming-conventions.md](./02-naming-conventions.md). |

## Influences

- **[Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)** — canonical authority. Where our guidance collides, the official guide wins (save the deviations above).
- **[ts.dev/style](https://ts.dev/style/)** — the community adaptation of the Google guide; fills the gaps the official guide leaves open.
- **[gts](https://github.com/google/gts)** — Google's opinionated TypeScript tooling; the single formatter and lint baseline for this guide.
- **[Effective TypeScript (Dan Vanderkam)](https://effectivetypescript.com/)** — community canon for idiomatic, type-safe patterns.
- **[total-typescript.com](https://www.totaltypescript.com/)** — modern type-system technique: branded types, discriminated unions, `satisfies`, inference design.
- **TigerBeetle Tiger Style** — assertion density, the 70-line function cap, limits on everything, no recursion, zero technical debt.

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **module / package level or larger** — never mix two styles within the same module. A half-migrated module is more confusing than either end state.
