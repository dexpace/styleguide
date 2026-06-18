---
name: typescript-styleguide
description: Use when writing, editing, or reviewing TypeScript (.ts) source in a dexpace project — enforces the dexpace TypeScript styleguide (no any/no enum, erasable syntax, Result unions, zod at boundaries, 70-line function cap). Also use before committing TypeScript, or when asked to review TypeScript against the styleguide.
---

# TypeScript styleguide

Extends the [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html) and [ts.dev/style](https://ts.dev/style/); where they conflict, Google wins, except the deviations below. TS ≥ 5.8, ESM-only, erasable syntax only (emits no runtime code), `gts` toolchain.

## When this applies

Editing `*.ts`, or reviewing TypeScript. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data and functions, not objects: model state as plain objects typed by `interface`; group behaviour into free functions and small interfaces. Reserve `class` for stateful lifecycle resources. No inheritance for code reuse.
2. Explicit over implicit: no `any`, no decorator or DI magic, no type-space syntax that emits runtime code. Every dependency is a parameter in the signature; library options follow documented defaults, callers pass only what differs via an options object.
3. Immutable by default: `const` over `let`, `readonly` fields, `ReadonlyArray<T>` in public signatures, frozen config. Update by spreading into a new value, never by mutation.
4. Errors are values, handled explicitly: typed `Error` subclasses per domain with mandatory `cause` chaining; opt-in `Result<T, E>` discriminated unions where failure is expected, never mixed within a module. `catch (e: unknown)`, then narrow. No swallowing; `no-floating-promises` is a lint error.
5. Composition over inheritance: `extends` is for `Error` hierarchies and nothing else. Closed polymorphism is a discriminated union; reuse is delegation. Small interfaces composed, never a deep class tree.
6. Transform, don't mutate: build pipelines from `map`/`filter`/`reduce`; reach for `for…of` only for effects or early exit. Take input, return new output; name intermediate steps past ~3 stages.
7. Always say why: TSDoc explains reasoning, not mechanics; `@example` on non-obvious public API. If you can't say why a line exists, question whether it should.
8. Assert aggressively: an `invariant(cond, msg): asserts cond` helper narrows types as it checks. ≥2 per function on average — preconditions at entry, postconditions at exit; split compound checks; assert positive and negative space.
9. Limits on everything: functions cap at **70 lines** (lint-enforced); nesting at `max-depth 3`, aim for two. Bound every loop, queue, retry, pool, cache, fan-out. Timeouts mandatory on external I/O via `AbortSignal.timeout()`. No recursion in library code.
10. Small functions, breathing room: aim 10–30 lines, one level of abstraction each. Guard clauses first so the happy path stays flush left; blank lines between logical sections.
11. Performance from the outset: design-time is when 1000× wins are cheap. Work with the grain of V8 — stable object shapes for monomorphism. Batch over serial `await`s. Optimize the slowest resource first: network > disk > memory > CPU.
12. Zero technical debt: do it right the first time. Perfection over technical debt — debt never gets paid.

## Language hard rules

- `any` is banned; accept `unknown` and narrow inward. `as` needs a why-comment; prefer `satisfies`, guards, and parse-don't-validate. Non-null `!` is banned outside tests and declared `// bridge:` lines — narrow with `invariant` instead.
- `enum` and constructor parameter properties are banned (`erasableSyntaxOnly`) — use literal unions or `as const` maps; assign constructor fields explicitly.
- Discriminated unions kept whole so narrowing works; close every `switch` with `default: return assertNever(x)`. Absence is `undefined`, never `null` past the boundary. `readonly` on every public shape. Brand domain primitives so IDs cannot be swapped.
- Default with `??`, never `||`; reach for `?.` but never chain past a lying type. Pipelines over loops; name the steps. `for…in` banned, `.forEach` discouraged.
- Errors: domain `Error` subclasses with context fields, mandatory `cause` on rethrow, `catch (e: unknown)` then narrow, opt-in `Result` per module. Crash programmer errors with `invariant`; handle operational errors as values. `AggregateError` for fan-out.
- Concurrency: `async`/`await` only, float no promises, `{ signal }` in the signature, `AbortSignal.timeout()` on every external call, bound fan-out through a worker-pool helper, treat interleaving across an `await` as a real race.
- zod **v4 top-level forms only** — `z.email()`, `z.uuid()`, `z.url()`, never the chained `.string().email()` style. Wire types come from `z.infer`, never hand-written beside the schema; parse `unknown` at every boundary.
- `const` default, `let` only when reassignment is visible, `var` banned. kebab-case files. No `I` prefix, no `Async` suffix. Named exports only; `index.ts` is the package contract. `import type` for type-only imports; ESM only, no import-time side effects. `gts` is law; `tsc --noEmit` is the typecheck gate.

## Before you finish — verify

```bash
bun install         # against a committed bun.lock; --frozen-lockfile in CI
gts lint            # gts is the toolchain; no standalone prettier/eslint config
tsc --noEmit        # the typecheck gate — bun test never type-checks
bun test            # colocated *.test.ts; deterministic, no real clock/net/fs
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/typescript/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/typescript/01-formatting-and-tooling.md)
- [02 Naming Conventions](https://github.com/dexpace/styleguide/blob/main/typescript/02-naming-conventions.md)
- [03 The Type System](https://github.com/dexpace/styleguide/blob/main/typescript/03-the-type-system.md)
- [04 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/typescript/04-variables-and-declarations.md)
- [05 Functions](https://github.com/dexpace/styleguide/blob/main/typescript/05-functions.md)
- [06 Classes & Data Modeling](https://github.com/dexpace/styleguide/blob/main/typescript/06-classes-and-data-modeling.md)
- [07 TypeScript Idioms](https://github.com/dexpace/styleguide/blob/main/typescript/07-typescript-idioms.md)
- [08 Error Handling](https://github.com/dexpace/styleguide/blob/main/typescript/08-error-handling.md)
- [09 Concurrency & Async](https://github.com/dexpace/styleguide/blob/main/typescript/09-concurrency.md)
- [10 API Design](https://github.com/dexpace/styleguide/blob/main/typescript/10-api-design.md)
- [11 Testing](https://github.com/dexpace/styleguide/blob/main/typescript/11-testing.md)
- [12 Module Organization](https://github.com/dexpace/styleguide/blob/main/typescript/12-module-organization.md)
- [13 Resource Management](https://github.com/dexpace/styleguide/blob/main/typescript/13-resource-management.md)
- [14 Documentation](https://github.com/dexpace/styleguide/blob/main/typescript/14-documentation.md)
- [15 Performance](https://github.com/dexpace/styleguide/blob/main/typescript/15-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
