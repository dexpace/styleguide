# TypeScript Styleguide Family Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author the three-guide TypeScript family (`typescript/` 15 chapters, `typescript-node/` 8 chapters, `typescript-react/` 8 chapters, plus READMEs and root-README rows) per the approved spec.

**Architecture:** Pure-markdown documentation in the repo's established shape: each guide extends a canonical upstream authority; chapters follow the kotlin/python spine; every chapter opens with a "What good looks like" exemplar; every rule carries step-by-step reasoning and an enforcement note. Three publish commits (one per guide) reshape the per-task work at the end.

**Tech Stack:** GitHub-flavored Markdown only — no build. Verification is mechanical (`grep`/`wc`/`ls`) plus read-through. Spec: `docs/superpowers/specs/2026-06-06-typescript-styleguide-design.md` (the **Spec**). Working branch: `feat/typescript-styleguide`.

---

## Chapter Format Contract (the "type definition" every task references)

Every chapter file MUST have this skeleton, in this order:

```markdown
# NN — Title

One-paragraph scope statement (2–4 sentences, what this chapter governs and why it matters).

## What good looks like

​```ts
// A complete, idiomatic 20–40-line exemplar embodying this chapter's rules in concert.
// Realistic domain (bookings, payments, search — match the repo's example flavor).
​```

One short paragraph naming which rules the exemplar demonstrates (link by rule number).

## Rules

### N.1 — Imperative rule title.

**Reasoning, step by step:**

1. Numbered why-chain (2–5 steps): how the rule serves correctness/performance/simplicity/expressiveness.
2. ...

​```ts
// Worked example: good (and bad where contrast teaches).
​```

**Enforcement:** named compiler flag / gts / eslint rule / CI check — or "review" if none exists.

### N.2 — ...

## Cross-references

- [typescript/NN-title.md](./NN-title.md) — why related.
- Root rules / cross-cutting docs where relevant.
```

**Voice:** house style — imperative headlines, confident declaratives, reasons over mechanics, no hedging, no filler. Match kotlin/python guides' density. Use `—` em-dashes sparingly and purposefully. Numbers and caps stated plainly ("70 lines. Hard cap.").

**Counting rules used by Verify steps:** rule count = `grep -c '^### '`; exemplar present = `grep -c '^## What good looks like'` (expect 1); reasoning blocks = `grep -c 'Reasoning, step by step'` (expect = rule count); line band = `wc -l`.

**Per-task commit convention (dev commits; reshaped in Task 39):** `docs(<guide>): draft <NN-chapter-name>`.

---

## File Structure

```
typescript/                  README.md + 01…15 (16 files)   ~1,500–2,500 lines total
typescript-node/             README.md + 01…08 (9 files)    ~900 lines total
typescript-react/            README.md + 01…08 (9 files)    ~900 lines total
README.md                    3 new table rows (one per phase)
```

Responsibilities: each chapter file owns exactly one concern (the spine topic); READMEs own authority chains, values, root-rule restatements, deviations ledgers, indexes; root README owns the per-language index.

---

# Phase A — `typescript/` core

### Task 1: typescript/README.md

**Files:** Create `typescript/README.md` · Mirrors: `kotlin/README.md` (structure), `python/README.md` (authority framing) · Spec: Architecture, Values, Toolchain, Deviations ledger.

- [ ] **Step 1: Read** `kotlin/README.md`, `python/README.md`, Spec sections above.
- [ ] **Step 2: Draft** (~110–150 lines) with these sections:
  - Title + one-line charter ("Binding TypeScript rules for all dexpace projects; extends Google's TS style guide").
  - **Authorities** (ordered): Google TS Style Guide (canonical) → ts.dev/style (community adaptation, fills gaps) → gts v7 (tooling embodiment) → this guide's overlay. Conflict rule stated.
  - **Values:** correctness > performance > simplicity > expressiveness — all held to the elegance standard; one paragraph from Spec Values (including "elegance is enforced, not aspired to").
  - **Chapter index:** table of 01–15 with one-line scopes (titles exactly as in Phase A tasks below).
  - **The 12 root rules, restated TS-natively** (one short paragraph each):
    1. Data + functions: `interface` + free functions; classes only for lifecycle resources.
    2. Explicit over implicit: no `any`, no decorator/DI magic, every dependency a parameter; documented defaults at options.
    3. Immutable by default: `const`, `readonly`, `ReadonlyArray`, frozen config; spread-to-update.
    4. Errors are values handled explicitly: typed `Error` subclasses + mandatory `cause`; opt-in `Result` unions; `no-floating-promises`.
    5. Composition over inheritance: `extends` only for `Error`; unions + delegation; small interfaces.
    6. Transform, don't mutate: pipeline `map`/`filter`/`reduce`; pure functions; named steps.
    7. Always say why: TSDoc why-comments; `@example`; enforcement notes name their rule.
    8. Assert aggressively: `invariant(): asserts` helper; 2+ per function; assertions narrow types.
    9. Limits on everything: 70-line functions; `max-depth 3`; bounded fan-out, pools, caches, timeouts (`AbortSignal.timeout`); no recursion in library code.
    10. Small functions, breathing room: aim 10–30 lines; guard clauses; blank-line paragraphs.
    11. Performance from the outset: V8 monomorphism; batching over serial awaits; network > disk > memory > CPU.
    12. Zero technical debt — **closing rule, verbatim framing:** "perfection over technical debt — debt never gets paid."
  - **Deviations from upstream** table: enums banned (Google allows) · parameter properties banned (Google allows) · 70-line function cap (no upstream cap) — each with one-line why + chapter link.
- [ ] **Step 3: Verify:** `grep -c '^|' typescript/README.md` ≥ 20 (two tables present); `grep -c 'debt never gets paid' typescript/README.md` → 1; `wc -l` in 110–170.
- [ ] **Step 4: Commit:** `git add typescript/README.md && git commit -m "docs(typescript): draft README"`

### Task 2: typescript/01-formatting-and-tooling.md

**Files:** Create `typescript/01-formatting-and-tooling.md` · Mirrors: `python/01-formatting-and-tooling.md`, `kotlin/01-formatting-and-tooling.md` · Spec: Toolchain (decided), ch 01 row.

- [ ] **Step 1: Read** mirrors + Spec Toolchain section.
- [ ] **Step 2: Draft** (110–160 lines, 9–10 rules) per Format Contract. Exemplar: a complete minimal repo setup — `package.json` scripts (`lint`, `fix`, `compile`, `test`), `eslint.config.js` extending gts + the overlay, `tsconfig.json` extending gts base + the six flags. Rule inventory:
  - 1.1 gts is the toolchain — `npx gts init`; `gts lint` in CI; `gts fix` locally; no standalone Prettier/ESLint configs.
  - 1.2 Prettier defaults (via gts) are final — formatting is a non-discussion; zero overrides.
  - 1.3 tsconfig extends gts base and adds exactly six flags — table: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `isolatedModules`, `verbatimModuleSyntax`, `erasableSyntaxOnly`, each with its one-line why (from Spec table).
  - 1.4 TypeScript ≥ 5.8, latest stable preferred — `erasableSyntaxOnly` floor.
  - 1.5 pnpm, pinned via corepack `packageManager` — strict node_modules = undeclared deps fail.
  - 1.6 One ESLint overlay file extending gts — `strict-type-checked` + `stylistic-type-checked`; every added rule traceable to a chapter.
  - 1.7 `max-lines-per-function: 70` — `skipComments: true`, blank lines counted (python's counting rule, restated).
  - 1.8 `max-depth: 3` and `max-params: 3` — shape caps; aim depth ≤ 2; ≥3 params means options object (cross-ref 05).
  - 1.9 Pre-commit: `gts lint` + `tsc --noEmit` — broken style or types never enters history.
  - 1.10 `eslint-disable` requires a why-comment on the same line — undocumented suppressions are debt.
- [ ] **Step 3: Verify:** `grep -c '^### ' typescript/01-formatting-and-tooling.md` → 10; `grep -c '^## What good looks like'` → 1; `grep -c 'Reasoning, step by step'` → 10; `wc -l` in 110–170.
- [ ] **Step 4: Commit:** `git add typescript/01-formatting-and-tooling.md && git commit -m "docs(typescript): draft 01-formatting-and-tooling"`

### Task 3: typescript/02-naming-conventions.md

**Files:** Create `typescript/02-naming-conventions.md` · Mirrors: `python/02-naming-conventions.md` (verb taxonomy §2.11–2.12), `go/02-naming-conventions.md` (call-site framing) · Spec: ch 02 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (110–150 lines, 10–11 rules). Exemplar: a small client module whose names demonstrate casing, verb taxonomy, predicate booleans, unit suffixes. Rule inventory:
  - 2.1 Casing table (Google verbatim): `lowerCamelCase` variables/functions/properties/parameters; `UpperCamelCase` classes/interfaces/types; `CONSTANT_CASE` only module-level deep constants.
  - 2.2 No `I` prefix on interfaces; no leading/trailing underscores — visibility is a keyword, not a glyph.
  - 2.3 `CONSTANT_CASE` means deeply immutable AND conceptually constant — a frozen lookup table qualifies; a mutable singleton never does.
  - 2.4 Files kebab-case (`user-service.ts`) — case-sensitivity-safe across OSes; ecosystem norm.
  - 2.5 No abbreviations beyond the idiomatic set (`id`, `url`, `ctx`, `i` in tight loops) — names are read far more than typed.
  - 2.6 Client verb taxonomy: `get` (one, throws/undefined per contract), `list` (many), `create`, `upsert`, `update`, `delete`, `begin` (long-running) — port python §2.11 semantics table.
  - 2.7 No `Async` suffix — the return type already says `Promise`; suffixes are Hungarian.
  - 2.8 Booleans read as predicates: `is`/`has`/`can`/`should` — `enabled` is a state, `isEnabled` is a question.
  - 2.9 Names are designed for the call site — the audience is the reader of the caller; test every public name by writing its call site first.
  - 2.10 Quantities carry units: `timeoutMs`, `sizeBytes`, `ttlSeconds` — unit bugs are correctness bugs.
  - 2.11 Type parameters: single conventional letter (`T`, `K`, `V`) when role is obvious; descriptive `UpperCamelCase` (`TRow`, `TError`) when not.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 11; exemplar 1; reasoning 11; `wc -l` 110–160.
- [ ] **Step 4: Commit:** `git add typescript/02-naming-conventions.md && git commit -m "docs(typescript): draft 02-naming-conventions"`

### Task 4: typescript/03-the-type-system.md

**Files:** Create `typescript/03-the-type-system.md` · Mirrors: `kotlin/03-nullability.md` (safety-slot framing, the `!!`-style ban patterns), `python/03-type-hints.md` · Spec: ch 03 row, Decision record (erasable).

- [ ] **Step 1: Read** mirrors + Spec row + Spec decision record.
- [ ] **Step 2: Draft** (140–190 lines, 12 rules — richest chapter, like kotlin's nullability). Exemplar: a boundary module that parses unknown JSON into a branded domain type via zod + narrowing, no `any`, no assertions. Rule inventory:
  - 3.1 The strict flag family is law — what each of the six tsconfig additions catches, one example each (cross-ref 01.3).
  - 3.2 `any` is banned — `unknown` at boundaries + narrowing inward; the `no-explicit-any` + `no-unsafe-*` rule set enforces.
  - 3.3 `@ts-ignore` banned; `@ts-expect-error` only with a reason string, only in tests and declared bridges.
  - 3.4 Type assertions (`as`) require a why-comment — prefer `satisfies`, narrowing, or parsing; an assertion is an unproven claim.
  - 3.5 Absence is `undefined`; `null` only where an external contract forces it, converted at the boundary — one absence value simplifies every narrowing.
  - 3.6 Optional `?` over `| undefined` in object types — synergy with `exactOptionalPropertyTypes`.
  - 3.7 Narrowing toolbox, in preference order: discriminant → `typeof`/`instanceof`/`in` → custom guard — reach for the weakest tool that works.
  - 3.8 Custom type guards (`x is T`) MUST have unit tests — a wrong guard is a type-system lie that spreads (cross-ref 11).
  - 3.9 Branded types for domain primitives in high-rigor modules — `type UserId = string & { readonly __brand: 'UserId' }` + parse-constructor.
  - 3.10 `readonly` / `ReadonlyArray` / `Readonly<T>` in every public signature — immutability is the API's promise, not the caller's discipline.
  - 3.11 Generics: always constrained; no gratuitous type parameters; variance annotations (`in`/`out`) on public generic interfaces.
  - 3.12 Erasable syntax only — enums, runtime namespaces, parameter properties, `import =` aliases are compile errors (`erasableSyntaxOnly`); replacement idioms: literal union, `as const` object (show both, from Spec decision record); carve-out: consuming third-party/codegen enums at the boundary.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 12; exemplar 1; reasoning 12; `wc -l` 140–200.
- [ ] **Step 4: Commit:** `git add typescript/03-the-type-system.md && git commit -m "docs(typescript): draft 03-the-type-system"`

### Task 5: typescript/04-variables-and-declarations.md

**Files:** Create `typescript/04-variables-and-declarations.md` · Mirrors: `python/04-variables-and-declarations.md`, `kotlin/04-variables-and-declarations.md` · Spec: ch 04 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (90–130 lines, 8 rules). Exemplar: a config module — `as const` literal, derived types, no mutable module state. Rule inventory:
  - 4.1 `const` by default; `let` requires a reason the reader can see; `var` is banned (lint `no-var`).
  - 4.2 Non-null assertion `!` banned outside tests and declared bridges — the kotlin `!!` rule (kotlin/03 §3.2), same reasoning chain; `requireNonNull`-style: use `invariant(x !== undefined)` to narrow with a message.
  - 4.3 `as const` for literal configuration — widest-by-default literals are a silent loss of information.
  - 4.4 Declare at first use, one declaration per line — declarations are documentation of flow.
  - 4.5 No chained assignment (`a = b = c`) — hidden intermediate mutation.
  - 4.6 Destructure at boundaries with defaults; maximum two levels deep — deeper nesting hides shape.
  - 4.7 No module-level mutable state — module scope is global scope wearing a jacket; explicit stores/parameters instead.
  - 4.8 No shadowing (`no-shadow` via typescript-eslint) — two meanings for one name in one screen is a trap.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 90–140.
- [ ] **Step 4: Commit:** `git add typescript/04-variables-and-declarations.md && git commit -m "docs(typescript): draft 04-variables-and-declarations"`

### Task 6: typescript/05-functions.md

**Files:** Create `typescript/05-functions.md` · Mirrors: `python/05-functions.md`, `kotlin/05-functions.md` · Spec: ch 05 row (elegance mechanisms live here).

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (150–200 lines, 12 rules — flagship elegance chapter). Exemplar: a ~35-line module showing step-down order (public function → two helpers), guard clauses, options object, invariants, paragraphing. Rule inventory:
  - 5.1 **70 lines. Hard cap.** (`max-lines-per-function`; skipComments true, blanks counted). Aim 10–30. A function that needs "and" to describe needs splitting.
  - 5.2 One level of abstraction per function — a function either orchestrates named steps or does primitive work, never both in one body.
  - 5.3 Guard clauses first; happy path flush left; early return over nesting (`max-depth: 3`, aim ≤ 2).
  - 5.4 Step-down rule: callers above callees; file reads top-down — and therefore top-level `function` declarations (hoisting makes the order legal, names survive in stack traces); arrows for callbacks only.
  - 5.5 Options object for ≥ 3 parameters or any boolean parameter (`max-params: 3`) — call sites must read without consulting the signature; booleans at call sites are unlabeled switches.
  - 5.6 The `invariant` helper — define it in full (~10 lines, `asserts cond` signature, message required, throws `InvariantViolation`); assertions narrow types: runtime checks teach the compiler.
  - 5.7 Assert aggressively: 2+ per function average — preconditions at entry, postconditions at exit; positive and negative space; pair assertions (verify one property two ways).
  - 5.8 Pure by default — side-effecting functions are named with effect verbs and isolated at the edges (root rule 6).
  - 5.9 No control-flag parameters — a boolean that forks behavior means two functions are hiding in one.
  - 5.10 Overloads only when the return type depends on input shape; otherwise unions — overloads are N signatures to keep honest.
  - 5.11 Explicit return types on every exported function — inference is for locals; public contracts are written, not derived.
  - 5.12 Paragraph with blank lines — group statements by thought; whitespace is free (root rule 10).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 12; exemplar 1; reasoning 12; `wc -l` 150–210.
- [ ] **Step 4: Commit:** `git add typescript/05-functions.md && git commit -m "docs(typescript): draft 05-functions"`

### Task 7: typescript/06-classes-and-data-modeling.md

**Files:** Create `typescript/06-classes-and-data-modeling.md` · Mirrors: `python/06-classes-and-data-modeling.md` (richest python chapter), `kotlin/06-classes-and-data-modeling.md` · Spec: ch 06 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (140–190 lines, 11 rules). Exemplar: an order-state model — discriminated union making illegal states unrepresentable, `assertNever` exhaustive switch, factory with validation. Rule inventory:
  - 6.1 **Make illegal states unrepresentable** — headline rule; model variants as union members, not optional-field soup (`status` union vs `paidAt?: Date; cancelledAt?: Date` on one bag).
  - 6.2 `interface` for object shapes; `type` for unions, intersections, mapped and conditional types (Google).
  - 6.3 Data is plain objects + free functions; classes only for stateful lifecycle resources (connections, caches, servers).
  - 6.4 `extends` only for `Error` hierarchies — composition via explicit delegation everywhere else (root rules 1, 5).
  - 6.5 Discriminated unions are the sum type: literal `kind` discriminant; exhaustive `switch`; `assertNever(x: never)` helper defined in full (~5 lines).
  - 6.6 `readonly` fields by default; `Readonly<T>`/`ReadonlyArray` in public signatures (cross-ref 03.10).
  - 6.7 `private` modifier over `#private` (Google; erasable, zero-cost) — `#` only when runtime privacy is itself a requirement.
  - 6.8 Constructors assign; factories validate — `create*` factory parses inputs (zod or invariants) and returns a valid object or throws; invalid instances are unrepresentable at runtime too.
  - 6.9 Value objects are frozen plain objects compared structurally — equality helpers, not `.equals` methods.
  - 6.10 No optional-field explosion — two or more co-traveling optionals are a hidden variant: lift them into the union.
  - 6.11 Branded IDs at domain edges (cross-ref 03.9) — `OrderId` and `UserId` must not be interchangeable.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 11; exemplar 1; reasoning 11; `wc -l` 140–200.
- [ ] **Step 4: Commit:** `git add typescript/06-classes-and-data-modeling.md && git commit -m "docs(typescript): draft 06-classes-and-data-modeling"`

### Task 8: typescript/07-typescript-idioms.md

**Files:** Create `typescript/07-typescript-idioms.md` · Mirrors: `kotlin/07-kotlin-idioms.md` (richest kotlin chapter), `python/07-pythonic-idioms.md` · Spec: ch 07 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (140–190 lines, 12 rules). Exemplar: a transform pipeline — fetch result → parse → named steps (`const eligible = …; const ranked = …`) → frozen output; demonstrates `satisfies`, `??`, `for…of` for the effect at the end. Rule inventory:
  - 7.1 `satisfies` to validate without widening — annotation keeps the literal type, `satisfies` keeps both safety and inference.
  - 7.2 `as const` for fixed shapes (cross-ref 03.12 replacement idioms).
  - 7.3 `??` for defaults; `||` banned for defaults (`prefer-nullish-coalescing`) — `0`, `''`, `false` are values, not absences.
  - 7.4 Optional chaining `?.` — but two-plus `?.` in one expression means the type is lying or the model is wrong; fix the model.
  - 7.5 Pipelines for transforms: `map`/`filter`/`reduce`; `for…of` for effects and early exit; `.forEach` discouraged (no `await`, no `break`; ts.dev concurs); `for…in` banned.
  - 7.6 **Name the steps** — chains beyond ~3 stages get named intermediate `const`s; the names are the pipeline's documentation; concision serves clarity, never the reverse.
  - 7.7 `Map`/`Set` for dynamic keys; objects for known shapes — index-signature objects blur both roles (`noPropertyAccessFromIndexSignature` synergy).
  - 7.8 `structuredClone` over JSON round-trips — JSON silently drops `undefined`, `Date` becomes string.
  - 7.9 Template literals over concatenation; template literal types for string contracts (`type Route = \`/api/${string}\``).
  - 7.10 Spread for immutable update; nested update via narrow helper functions — deep spread chains are write-only code.
  - 7.11 Exhaustive `switch` on discriminants; no `if`-chains over `kind` (`switch-exhaustiveness-check` + `assertNever`).
  - 7.12 No clever one-liners — if it needs a second read, it needs a second line.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 12; exemplar 1; reasoning 12; `wc -l` 140–200.
- [ ] **Step 4: Commit:** `git add typescript/07-typescript-idioms.md && git commit -m "docs(typescript): draft 07-typescript-idioms"`

### Task 9: typescript/08-error-handling.md

**Files:** Create `typescript/08-error-handling.md` · Mirrors: `python/08-error-handling.md` (§8.11 Result port), `kotlin/08-error-handling.md` (sealed hierarchy framing) · Spec: ch 08 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (140–190 lines, 11 rules). Exemplar: a payment module — domain error hierarchy with `cause` + context fields, boundary wrap, exhaustive handling. Rule inventory:
  - 8.1 Domain `Error` subclasses with typed context fields (ids, inputs, correlationId) — generic `Error` says nothing.
  - 8.2 `cause` is mandatory on every rethrow — `throw new PaymentDeclinedError(msg, { cause: e })`; a chain without cause is amnesia.
  - 8.3 `catch (e: unknown)` and narrow — define `toError(e: unknown): Error` helper in full (~6 lines).
  - 8.4 Never swallow: no empty catch (`no-empty`), no catch-and-continue without handling — handle, wrap-and-rethrow, or let it fly.
  - 8.5 Wrap at boundaries, log once at the top — inner layers add context, the boundary decides representation.
  - 8.6 The Result union, opt-in per module, no mixing — define the ~10-line `Ok`/`Err`/`Result<T, E>` (mirror python §8.11 shape in TS: frozen objects, `ok` discriminant); a module either throws or returns Results, never both.
  - 8.7 Programmer errors vs operational errors — bugs hit `invariant` and crash fast; expected failures are typed errors/Results and are handled (Tiger split).
  - 8.8 Error messages carry context, never secrets — inputs, ids, limits; security.md masking rules apply.
  - 8.9 No exceptions for control flow — absence is `undefined`, predicates are booleans (`find`, not try/catch around `get`).
  - 8.10 Public API documents its failure modes — `@throws` TSDoc, or `Result` in the signature; undocumented throws are hidden behaviour.
  - 8.11 `AggregateError` for fan-out failures — when N independent operations fail, report all N, not the first (pairs with `Promise.allSettled`, cross-ref 09).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 11; exemplar 1; reasoning 11; `wc -l` 140–200.
- [ ] **Step 4: Commit:** `git add typescript/08-error-handling.md && git commit -m "docs(typescript): draft 08-error-handling"`

### Task 10: typescript/09-concurrency.md

**Files:** Create `typescript/09-concurrency.md` · Mirrors: `python/09-concurrency.md` (TaskGroup/timeout discipline), `kotlin/09-concurrency.md` (structured concurrency framing) · Spec: ch 09 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (150–200 lines, 12 rules). Exemplar: bounded parallel fetch — semaphore-limited `Promise.all`, `AbortSignal.timeout`, cancellation propagation, allSettled aggregation. Rule inventory:
  - 9.1 The event loop model in five sentences — single thread, interleaving at every `await`; "concurrency" is interleaved I/O.
  - 9.2 `async`/`await` only — `.then` chains only at interop edges; mixing styles hides sequencing.
  - 9.3 **No floating promises** (`no-floating-promises`) — every promise is awaited, returned, or explicitly `void`-ed with a why-comment; a floating promise is an error path that vanishes.
  - 9.4 No promise-constructor wrapping of async work (`no-async-promise-executor`, no `new Promise` around existing promises).
  - 9.5 Every long-running async API takes `{ signal }: { signal?: AbortSignal }` — cancellation is part of the signature, not an afterthought (kotlin cooperative-cancellation analog).
  - 9.6 `AbortSignal.timeout()` on every external I/O call — unbounded waits are unbounded queues (python `asyncio.timeout` port; root rule 9).
  - 9.7 Bounded fan-out — define the ~15-line `mapWithConcurrency(items, limit, fn)` semaphore in full; naked `Promise.all(items.map(...))` over unbounded N is a resource bomb.
  - 9.8 No `await` in a loop for independent work (`no-await-in-loop` guidance) — batch, then await; sequential awaits are hidden latency.
  - 9.9 `Promise.allSettled` when partial failure is acceptable — and every rejection inspected; aggregate via `AggregateError` (cross-ref 8.11).
  - 9.10 Interleaving races are real on one thread — check-then-act across an `await` is a race; re-validate after every await or restructure to single-assignment.
  - 9.11 Propagate signals downward — pass `signal` through every layer; check `signal.aborted` in CPU loops (`signal.throwIfAborted()`).
  - 9.12 Queues, buffers, and in-flight maps are bounded — declare the bound, assert it (root rule 9).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 12; exemplar 1; reasoning 12; `wc -l` 150–210.
- [ ] **Step 4: Commit:** `git add typescript/09-concurrency.md && git commit -m "docs(typescript): draft 09-concurrency"`

### Task 11: typescript/10-api-design.md

**Files:** Create `typescript/10-api-design.md` · Mirrors: `python/10-api-design.md` (richest python chapter), `kotlin/10-api-design.md` · Spec: ch 10 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (130–180 lines, 10 rules). Exemplar: a small client package surface — `index.ts` named exports, symmetric verb family, options with documented defaults, zod boundary schema with `z.infer`. Rule inventory:
  - 10.1 Named exports only; no default exports (Google) — defaults rename silently at import sites.
  - 10.2 `index.ts` is the contract — re-export the public surface; everything else is internal (deep-import blocking via `exports` field: typescript-node/08).
  - 10.3 Export the least — every export is a promise you keep forever; start private.
  - 10.4 Accept interfaces, return concrete readonly types — narrowest input contract, fully-known output.
  - 10.5 Options objects with documented defaults — defaults live in one place, documented at the option (root rule 2 carve-out); callers state only deviations.
  - 10.6 **API symmetry** — parallel operations share vocabulary, parameter shape, and return shape; `list`/`get`/`create` read as a family (cross-ref 02.6).
  - 10.7 zod schema at every external boundary; `z.infer` is the single source of type truth for external data — hand-written types for wire data drift.
  - 10.8 Document failure modes (`@throws` or `Result` signature) and async cancellation (`signal` option) on every public operation.
  - 10.9 `@deprecated` with migration path + semver discipline — breaking changes are MAJOR, no exceptions (git-and-code-review.md release rules).
  - 10.10 Streams of results are `AsyncIterable` — pull-based, backpressure-aware, `for await` consumable (pagination as iteration).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 10; exemplar 1; reasoning 10; `wc -l` 130–190.
- [ ] **Step 4: Commit:** `git add typescript/10-api-design.md && git commit -m "docs(typescript): draft 10-api-design"`

### Task 12: typescript/11-testing.md

**Files:** Create `typescript/11-testing.md` · Mirrors: `python/11-testing.md`, `kotlin/11-testing.md` · Spec: ch 11 row (correctness flagship).

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (150–200 lines, 12 rules). Exemplar: one module tested three ways — unit (example-based), property (fast-check round-trip), type-level (`expectTypeOf`) — ~40 lines total. Rule inventory:
  - 11.1 Vitest is the runner — explicit imports (globals off); unit tests colocated `foo.test.ts`; integration/e2e under `tests/`.
  - 11.2 Arrange-Act-Assert; one behavior per test; name = behavior under condition (`returns undefined when the id is unknown`).
  - 11.3 Fakes over mocks for owned code — hand-rolled implementations of your own interfaces; `vi.mock` only for true externals (port kotlin/python stance).
  - 11.4 MSW for HTTP — network-level fakes; never monkey-patch `fetch`.
  - 11.5 **Property-based tests are mandatory** for codecs, parsers, serializers, and any function with stated invariants — fast-check; properties: round-trip, idempotence, order-insensitivity, bounds.
  - 11.6 **Type-level tests are mandatory** for public generics and conditional types — `expectTypeOf`; a type regression is a correctness bug no runtime test sees.
  - 11.7 Custom type guards get truth-table tests (cross-ref 03.8) — wrong guards poison downstream narrowing.
  - 11.8 Deterministic always: `vi.useFakeTimers` for time; seeded fast-check with the seed logged on failure; no real network/clock/fs in unit tests.
  - 11.9 Tests assert positive AND negative space — what happened and what must not have (2+ assertions; Tiger discipline applies to tests too).
  - 11.10 No shared mutable fixtures; no test interdependence — every test runs alone and in any order.
  - 11.11 Mutation testing (Stryker) nightly, recommended — the honest metric; coverage % is reported, never targeted (coverage theater).
  - 11.12 The test is the first caller — unergonomic tests are an API smell; listen (cross-ref 02.9, 10.6).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 12; exemplar 1; reasoning 12; `wc -l` 150–210.
- [ ] **Step 4: Commit:** `git add typescript/11-testing.md && git commit -m "docs(typescript): draft 11-testing"`

### Task 13: typescript/12-module-organization.md

**Files:** Create `typescript/12-module-organization.md` · Mirrors: `python/12-package-organization.md`, `kotlin/12-module-organization.md` · Spec: ch 12 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (110–150 lines, 9 rules). Exemplar: a feature-folder tree (10–15 lines, annotated) + a module skeleton showing top-down narrative. Rule inventory:
  - 12.1 ESM only — `"type": "module"`; CJS only at declared interop bridges.
  - 12.2 `import type` for type-only imports — `verbatimModuleSyntax` makes the runtime graph visible in the source.
  - 12.3 Feature folders over kind folders — `features/booking/` with its components/logic/tests together, not `services/` + `models/` layers (changes travel together).
  - 12.4 Barrels only at the package boundary — internal barrels invite cycles and bloat bundles; import the file you mean.
  - 12.5 Import cycles are bugs — `madge --circular` gate in CI; a cycle is a hidden coupling confession.
  - 12.6 No import-time side effects; `"sideEffects": false` — importing a module must be free (explicitness + tree-shaking).
  - 12.7 Import groups: `node:` → external → internal, blank-line separated — three paragraphs, like prose.
  - 12.8 Relative imports stay shallow (≤ 2 levels) — `../../..` means the tree is wrong; restructure, don't alias around it.
  - 12.9 **A module reads top-down as a story** — public API first, helpers below (step-down, cross-ref 05.4); one concept per file; ~300 lines is the "consider splitting" signal.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–160.
- [ ] **Step 4: Commit:** `git add typescript/12-module-organization.md && git commit -m "docs(typescript): draft 12-module-organization"`

### Task 14: typescript/13-resource-management.md

**Files:** Create `typescript/13-resource-management.md` · Mirrors: `kotlin/13-resource-management.md` (`use {}` framing), `python/13-resource-management.md` (`with` framing) · Spec: ch 13 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (110–150 lines, 9 rules). Exemplar: a resource-owning service — `await using` a connection, `DisposableStack` composing teardown, AbortController tying listeners and timeout together. Rule inventory:
  - 13.1 `using` / `await using` for every disposable — the `use {}`/`with` analog; scope IS lifetime.
  - 13.2 Owned resources implement `Symbol.dispose`/`Symbol.asyncDispose` — disposal is part of the type, not a README note.
  - 13.3 `DisposableStack`/`AsyncDisposableStack` for composite teardown — reverse-order release, exception-safe, no try/finally pyramids.
  - 13.4 AbortController is the universal lifecycle handle — one controller cancels fetches, timers, listeners (`addEventListener(..., { signal })`).
  - 13.5 Every `setTimeout`/`setInterval` has an owner and a clear path — tie to a signal or return a disposer; orphan timers are leaks with alarms.
  - 13.6 Pools, caches, queues: bounded, with TTL where applicable — LRU + TTL (performance.md port); an unbounded cache is a slow OOM.
  - 13.7 Release in reverse acquisition order — teardown mirrors setup (DisposableStack gives this for free).
  - 13.8 Never rely on GC for resources — `FinalizationRegistry` is not a cleanup strategy; close explicitly.
  - 13.9 Tests assert cleanup — leak checks: fake timers drained, handles closed in `afterEach` (pairs with 11.9 negative space).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–160.
- [ ] **Step 4: Commit:** `git add typescript/13-resource-management.md && git commit -m "docs(typescript): draft 13-resource-management"`

### Task 15: typescript/14-documentation.md

**Files:** Create `typescript/14-documentation.md` · Mirrors: `python/14-documentation.md`, `kotlin/14-documentation.md` · Spec: ch 14 row.

- [ ] **Step 1: Read** mirrors + Spec row.
- [ ] **Step 2: Draft** (100–140 lines, 8 rules). Exemplar: a fully documented public function — TSDoc with why-summary, `@param` only where non-obvious, `@throws`, `@example`. Rule inventory:
  - 14.1 TSDoc syntax for all public APIs — tooling-readable, IDE-surfaced.
  - 14.2 Never restate types in prose (Google) — the signature is the source; prose adds intent, constraints, units.
  - 14.3 Comments say why (root rule 7) — the diff shows what; mechanics-narration is noise.
  - 14.4 `@example` on every non-obvious public API — one canonical call site, compilable.
  - 14.5 `@throws` for every documented failure mode (cross-ref 08.10, 10.8).
  - 14.6 Every package has a README that gets a new engineer to first success in 30 seconds — install, one example, link to deeper docs.
  - 14.7 Link, don't duplicate — repeat nothing from this styleguide or other docs; duplicated prose forks and rots.
  - 14.8 Stale comments are debt — wrong documentation is worse than none; update or delete in the same commit that changes the code.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–150.
- [ ] **Step 4: Commit:** `git add typescript/14-documentation.md && git commit -m "docs(typescript): draft 14-documentation"`

### Task 16: typescript/15-performance.md

**Files:** Create `typescript/15-performance.md` · Mirrors: `python/15-performance.md`, `kotlin/15-performance.md`, root `performance.md` · Spec: ch 15 row.

- [ ] **Step 1: Read** mirrors + Spec row + root `performance.md`.
- [ ] **Step 2: Draft** (120–160 lines, 10 rules). Exemplar: before/after of a hot path — shape-stable objects, batched awaits, allocation hoisted out of the loop, with the measurement that justified it. Rule inventory:
  - 15.1 Design-time first — the 1000× wins are architectural; resource order: network > disk > memory > CPU (performance.md is canonical; this chapter is V8/TS specifics).
  - 15.2 Monomorphism: keep object shapes stable — same properties, same order, same types; the `interface` discipline (06.2) is also a JIT strategy.
  - 15.3 Never `delete` a property; no sparse arrays — both poison hidden classes; set `undefined` or rebuild.
  - 15.4 Allocation hygiene in hot paths — closures, spreads, and array churn per iteration are GC pressure; hoist, reuse, or restructure.
  - 15.5 Batch independent awaits (cross-ref 09.8) — N serial round-trips is the canonical self-inflicted latency.
  - 15.6 Measure first, keep the benchmark — `vitest bench` for micro (with its caveats stated), `--cpu-prof` + speedscope for real profiles; a perf fix without a benchmark is a rumor.
  - 15.7 String building: `+=` is fine (V8 ropes); the myth to kill is join-always; measure when it matters.
  - 15.8 JSON is a hot-path cost — parse/stringify are O(payload); schema-derived serializers at scale (typescript-node/07).
  - 15.9 Author for tree-shaking — named exports, `sideEffects: false`, no import-time work (12.6) — dead code that ships is negative performance.
  - 15.10 The optimization ledger — every deliberate perf trick carries a comment with the measurement that justified it; unexplained cleverness gets simplified away (zero-debt: future readers must know why).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 10; exemplar 1; reasoning 10; `wc -l` 120–170.
- [ ] **Step 4: Commit:** `git add typescript/15-performance.md && git commit -m "docs(typescript): draft 15-performance"`

### Task 17: Root README row — typescript

**Files:** Modify `README.md` (per-language guides table, after the Python row).

- [ ] **Step 1: Add row** matching existing format exactly:
  `| TypeScript | [\`typescript/\`](./typescript/) | [Google TS Style Guide](https://google.github.io/styleguide/tsguide.html) + [ts.dev/style](https://ts.dev/style/) | TS ≥ 5.8, erasable syntax only, gts toolchain, strict type-aware lint, Vitest + fast-check + expectTypeOf. |`
- [ ] **Step 2: Verify:** `grep -c 'typescript/' README.md` ≥ 1; table renders (pipe count matches header).
- [ ] **Step 3: Commit:** `git add README.md && git commit -m "docs: index typescript guide in root README"`

# Phase B — `typescript-node/`

### Task 18: typescript-node/README.md

**Files:** Create `typescript-node/README.md` · Mirrors: `kotlin-jvm/README.md` (extension semantics, JVM-1…6 pattern) · Spec: Architecture, node guide intro.

- [ ] **Step 1: Read** `kotlin-jvm/README.md` + Spec.
- [ ] **Step 2: Draft** (~90–120 lines): extension declaration ("extends `typescript/`; additive; where stricter, wins for Node code"); authorities (Node.js docs canonical; goldbergyoni/nodebestpractices supplementary); chapter index (8 rows); NODE-1…6 restatements, one short paragraph each:
  - NODE-1 Crash-only error policy — unknown state is worse than a restart.
  - NODE-2 The event loop has a budget — blocking it blocks every request.
  - NODE-3 Every boundary is parsed — env, requests, queue messages: zod first.
  - NODE-4 Dependencies are attack surface — minimal, audited, pinned.
  - NODE-5 Named lifecycles — startup and shutdown are explicit, ordered, deadline-bounded.
  - NODE-6 ESM-native — one module system, `node:` prefixed builtins.
  Closing rule: zero debt (verbatim framing from typescript/README).
- [ ] **Step 3: Verify:** `grep -c 'NODE-' typescript-node/README.md` ≥ 6; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-node/README.md && git commit -m "docs(typescript-node): draft README"`

### Task 19: typescript-node/01-runtime-and-modules.md

**Files:** Create `typescript-node/01-runtime-and-modules.md` · Mirrors: `kotlin-jvm/01-java-interop.md` (boundary-chapter role) · Spec: node ch 01 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (90–120 lines, 8 rules). Exemplar: `package.json` (engines, packageManager, type, exports) + entry module with `node:` imports. Rules:
  - 1.1 Node Active LTS (≥ 24); `engines` enforced, `.nvmrc` present.
  - 1.2 `packageManager` pinned (corepack + pnpm) — reproducible everywhere.
  - 1.3 `node:` prefix on every builtin import — unambiguous, future-proof.
  - 1.4 Subpath `#imports` for internal cross-area paths — package.json-standard, no bundler magic.
  - 1.5 Compile with tsc for production — type stripping is for dev loops only (revisit when stable).
  - 1.6 No `--experimental-*` flags in production.
  - 1.7 ESM only; CJS interop quarantined to declared bridge modules (cross-ref typescript/12.1).
  - 1.8 One process per container — scale horizontally; no `cluster` module.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-node/01-runtime-and-modules.md && git commit -m "docs(typescript-node): draft 01-runtime-and-modules"`

### Task 20: typescript-node/02-concurrency-and-event-loop.md

**Files:** Create `typescript-node/02-concurrency-and-event-loop.md` · Mirrors: `kotlin-jvm/02-jvm-concurrency.md` · Spec: node ch 02 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (110–140 lines, 9 rules). Exemplar: graceful-shutdown skeleton (SIGTERM → stop accepting → drain with deadline → close pools → exit). Rules:
  - 2.1 `*Sync` APIs banned in request paths — one sync read stalls every in-flight request.
  - 2.2 Event-loop lag is a first-class metric — `monitorEventLoopDelay`, alert on budget breach (state a default budget: p99 < 50ms).
  - 2.3 CPU-bound work goes to `worker_threads` via piscina — bounded pool, sized explicitly.
  - 2.4 Backpressure honored end-to-end — `stream.pipeline`/async iterators; never raw `.pipe()` without error wiring.
  - 2.5 `unhandledRejection`/`uncaughtException`: log, flush, exit 1 — crash-only (NODE-1); the supervisor restarts; limping state corrupts.
  - 2.6 Graceful shutdown is ordered and deadline-bounded — the exemplar sequence, with `AbortSignal.timeout` on the drain.
  - 2.7 Long-lived loops yield — `setImmediate`/scheduler hooks in batch jobs; starvation is self-DoS.
  - 2.8 In-flight request tracking is bounded — load shedding over unbounded queueing (root rule 9).
  - 2.9 Timers and intervals tied to lifecycle signals (cross-ref typescript/13.5).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–150.
- [ ] **Step 4: Commit:** `git add typescript-node/02-concurrency-and-event-loop.md && git commit -m "docs(typescript-node): draft 02-concurrency-and-event-loop"`

### Task 21: typescript-node/03-http-services.md

**Files:** Create `typescript-node/03-http-services.md` · Mirrors: `kotlin-jvm/03-jvm-frameworks.md` · Spec: node ch 03 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (110–140 lines, 9 rules). Exemplar: one Fastify route — zod schema, typed handler, error mapping, correlation id. Rules:
  - 3.1 Fastify is the HTTP framework — schema-first validation (parse-don't-validate native), explicit plugin model, no DI container, measured throughput; the reasoning ledger from the Spec.
  - 3.2 Express is legacy-maintenance only; Hono acceptable at the edge; **NestJS rejected** — decorator/DI magic violates root rule 2 and `erasableSyntaxOnly` (deviations ledger entry).
  - 3.3 Every route declares request/response schemas — zod via type provider; unvalidated input never reaches a handler (NODE-3).
  - 3.4 Handlers are thin — parse → call domain function → map result; business logic lives in plain functions (data + functions).
  - 3.5 Centralized error mapping — one boundary translator from domain errors to HTTP problem responses; handlers never hand-craft 500s.
  - 3.6 Correlation ID middleware — accept inbound or generate; propagate via AsyncLocalStorage (cross-ref 06); every log line carries it.
  - 3.7 Timeouts on the server (headers, body, keep-alive, per-route) — stated defaults; slowloris is a config bug.
  - 3.8 Rate limiting at the edge with bounded state (security.md port).
  - 3.9 Health endpoints are honest — liveness vs readiness, readiness gates on dependencies.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–150.
- [ ] **Step 4: Commit:** `git add typescript-node/03-http-services.md && git commit -m "docs(typescript-node): draft 03-http-services"`

### Task 22: typescript-node/04-persistence.md

**Files:** Create `typescript-node/04-persistence.md` · Mirrors: `kotlin-jvm/04-persistence.md` · Spec: node ch 04 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (110–140 lines, 9 rules). Exemplar: a Drizzle schema + a repository of plain functions + an explicit transaction. Rules:
  - 4.1 Drizzle is the data layer — SQL visible in code, types derived from schema, no codegen engine (reasoning from Spec); raw `pg` acceptable; Prisma requires written justification.
  - 4.2 Repositories are plain functions taking a db handle — no repository classes, no active records (data + functions).
  - 4.3 Transactions are explicit scopes — `db.transaction(async (tx) => …)`; transactionality visible at the call site, never ambient.
  - 4.4 Migrations: versioned, forward-only, reviewed SQL — generated by drizzle-kit, read by humans.
  - 4.5 Connection pools: explicitly sized, bounded, with acquisition timeouts — defaults stated (performance.md pooling port).
  - 4.6 Queries are bounded — every list query has LIMIT; pagination by cursor (cross-ref typescript/10.10).
  - 4.7 N+1 is a bug — batch with `IN`/joins (performance.md port); the query count is part of review.
  - 4.8 Entities never cross the API boundary — map to DTOs at the edge (kotlin-jvm 04 parity); wire types come from zod (typescript/10.7).
  - 4.9 Database errors are wrapped into domain errors at the repository boundary with `cause` (cross-ref typescript/08.5).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–150.
- [ ] **Step 4: Commit:** `git add typescript-node/04-persistence.md && git commit -m "docs(typescript-node): draft 04-persistence"`

### Task 23: typescript-node/05-serialization-and-validation.md

**Files:** Create `typescript-node/05-serialization-and-validation.md` · Mirrors: `kotlin-jvm/05-serialization.md` · Spec: node ch 05 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: env config module — zod schema, parse at startup, frozen export, fail-fast error listing every missing var. Rules:
  - 5.1 `process.env` is parsed once, at startup, into a frozen typed config — invalid config fails the boot, listing every problem (not the first).
  - 5.2 Every request/response body has a zod schema (cross-ref 03.3) — and `z.infer` is the type (typescript/10.7).
  - 5.3 Raw `JSON.parse` never escapes a boundary module — parse → validate → typed value, one move.
  - 5.4 Dates are ISO-8601 strings on the wire — `Temporal` when natively available, `date-fns` until then; never `moment`.
  - 5.5 Null vs absent is a contract decision, stated per field — `exactOptionalPropertyTypes` makes the distinction real (typescript/03.6).
  - 5.6 Unknown fields: strip by default, reject where integrity matters — `.strict()` on commands, `.passthrough()` never silent.
  - 5.7 Outbound serialization is schema-checked too — responses validated in dev/test; you can't trust what you didn't check.
  - 5.8 BigInt, binary, and money: explicit wire encodings (string for money/i64) — floating point is not a currency type (correctness).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-node/05-serialization-and-validation.md && git commit -m "docs(typescript-node): draft 05-serialization-and-validation"`

### Task 24: typescript-node/06-logging.md

**Files:** Create `typescript-node/06-logging.md` · Mirrors: `kotlin-jvm/06-logging.md` (MDC parity chapter) · Spec: node ch 06 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: pino setup — redact paths, AsyncLocalStorage request context, one boundary log. Rules:
  - 6.1 pino, structured JSON — human formats are for terminals, not production.
  - 6.2 AsyncLocalStorage carries request context (correlation id, user id) — the MDC analog; context follows async hops automatically.
  - 6.3 `redact` paths for PII and secrets at the logger — masking is config, not per-call-site discipline (security.md port).
  - 6.4 Log once, at the boundary — inner layers attach context to errors (`cause` chain), the boundary writes one line (kotlin-jvm parity).
  - 6.5 Levels mean something: error = page someone; warn = degraded but serving; info = state transitions; debug = off in prod.
  - 6.6 `console.*` banned in server code — unstructured, unleveled, uncorrelated.
  - 6.7 Child loggers per component — `logger.child({ component })`; grep-able provenance.
  - 6.8 Log lines are bounded — no payload dumping; size caps stated (a log flood is an outage amplifier).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-node/06-logging.md && git commit -m "docs(typescript-node): draft 06-logging"`

### Task 25: typescript-node/07-node-performance.md

**Files:** Create `typescript-node/07-node-performance.md` · Mirrors: `kotlin-jvm/07-jvm-performance.md` · Spec: node ch 07 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: a measured optimization — event-loop lag graph reading → undici pool tuning → autocannon verification numbers. Rules:
  - 7.1 Event-loop lag is the JVM-pause analog — measure it, budget it, alert on it (cross-ref 02.2).
  - 7.2 Profile with `--cpu-prof` (speedscope) and heap snapshots; clinic/0x for triage — name the tool per question.
  - 7.3 undici with explicit keep-alive pools — bounded connections, stated timeouts; the default global agent is not a strategy.
  - 7.4 Streams for large payloads — buffering a 2GB file is an OOM with extra steps; pipeline + backpressure (cross-ref 02.4).
  - 7.5 Schema-derived JSON serialization on hot routes — fastify's `fast-json-stringify` path; measured, not assumed.
  - 7.6 Load test before shipping perf claims — autocannon, stated scenario, p50/p99 recorded in the PR (git-and-code-review test-plan port).
  - 7.7 Memory: watch RSS vs heap; container limits set `--max-old-space-size` explicitly — OOM-killer surprises are config debt.
  - 7.8 The optimization ledger applies (typescript/15.10) — every Node-specific trick carries its measurement.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-node/07-node-performance.md && git commit -m "docs(typescript-node): draft 07-node-performance"`

### Task 26: typescript-node/08-build-and-distribution.md

**Files:** Create `typescript-node/08-build-and-distribution.md` · Mirrors: `kotlin-jvm/08-build-and-distribution.md` · Spec: node ch 08 row.

- [ ] **Step 1: Read** mirror + Spec row.
- [ ] **Step 2: Draft** (110–140 lines, 9 rules). Exemplar: a library `package.json` — exports map, types, sideEffects, publish scripts with checks. Rules:
  - 8.1 Libraries build with plain tsc — declarations + source maps; no bundler between your source and your consumers.
  - 8.2 Services bundle with esbuild — one artifact, fast, reproducible; the script is checked in and boring.
  - 8.3 `exports` field is locked — explicit entry points; deep imports are compile errors for consumers (typescript/10.2 enforced).
  - 8.4 `publint` + `arethetypeswrong` gate publishing — package health is CI, not vibes.
  - 8.5 api-extractor API reports on libraries — the binary-compatibility-validator analog; surface diffs are reviewed diffs.
  - 8.6 npm publish with provenance; 2FA; lockfile committed always.
  - 8.7 Dependency policy: minimal, pinned, audited — every dep is supply-chain surface (NODE-4, security.md); justify each new one in the PR.
  - 8.8 Semver discipline with changesets or equivalent — version bumps trace to changelog entries (git-and-code-review release port).
  - 8.9 Reproducible builds — same input, same artifact; no network at build time beyond the lockfile.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–150.
- [ ] **Step 4: Commit:** `git add typescript-node/08-build-and-distribution.md && git commit -m "docs(typescript-node): draft 08-build-and-distribution"`

### Task 27: Root README row — typescript-node

**Files:** Modify `README.md` (after the TypeScript row).

- [ ] **Step 1: Add row:** `| TypeScript on Node | [\`typescript-node/\`](./typescript-node/) | Extends \`typescript/\`; Node.js docs + community best practices | Node ≥ 24 LTS, Fastify, Drizzle, pino, crash-only, event-loop budget. |`
- [ ] **Step 2: Verify:** `grep -c 'typescript-node/' README.md` ≥ 1.
- [ ] **Step 3: Commit:** `git add README.md && git commit -m "docs: index typescript-node guide in root README"`

# Phase C — `typescript-react/`

### Task 28: typescript-react/README.md

**Files:** Create `typescript-react/README.md` · Mirrors: `kotlin-jvm/README.md` (extension pattern), `typescript-node/README.md` (peer) · Spec: react guide intro.

- [ ] **Step 1: Read** mirrors + Spec.
- [ ] **Step 2: Draft** (~90–120 lines): extension declaration; authorities (react.dev Rules of React canonical for correctness; react-typescript-style-guide.com for conventions; conflict rule from Spec); chapter index (8 rows); REACT-1…6, one paragraph each:
  - REACT-1 Render purity is law — components and hooks are pure during render; the compiler assumes it.
  - REACT-2 Hooks rules are compiler-checked — react-hooks v6 recommended preset, `exhaustive-deps` as error.
  - REACT-3 Server state is not client state — caches are caches (TanStack Query); `useState` is for the client's own facts.
  - REACT-4 Accessibility is correctness — an unreachable feature is a broken feature.
  - REACT-5 Effects synchronize, never derive — derive in render; reach for an effect only to talk to a non-React system.
  - REACT-6 Composition over configuration — children and slots over boolean prop forests.
  Deviations ledger: GraphQL conventions not adopted (revisit on adoption). Closing rule: zero debt (verbatim framing).
- [ ] **Step 3: Verify:** `grep -c 'REACT-' typescript-react/README.md` ≥ 6; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-react/README.md && git commit -m "docs(typescript-react): draft README"`

### Task 29: typescript-react/01-components-and-props.md

**Files:** Create `typescript-react/01-components-and-props.md` · Spec: react ch 01 row · Upstream: react-typescript-style-guide.com component conventions.

- [ ] **Step 1: Read** Spec row + `typescript/05-functions.md` (shape rules apply to components).
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: one well-shaped component — `interface XProps`, destructured props, early returns, slots. Rules:
  - 1.1 Function components only — classes only via `react-error-boundary`.
  - 1.2 Props are `interface XProps`, exported beside the component — callers import the contract.
  - 1.3 No `React.FC` — implicit-children legacy, worse generics; type the props parameter, return `ReactNode` explicitly on exported components.
  - 1.4 Destructure props in the signature with defaults — the signature is the documentation.
  - 1.5 Composition over configuration (REACT-6) — `children`/named slots over `showX`/`variantY` prop forests; two boolean props that interact are a variant union begging to exist.
  - 1.6 Components follow function shape rules — 70-line cap applies; guard-clause early returns for loading/error states; one component per file.
  - 1.7 Internal component order (upstream port): hooks → derived values → effects → handlers → return — one predictable read.
  - 1.8 Props stay serializable where possible — functions and elements are fine; passing mutable class instances through props hides coupling.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-react/01-components-and-props.md && git commit -m "docs(typescript-react): draft 01-components-and-props"`

### Task 30: typescript-react/02-hooks.md

**Files:** Create `typescript-react/02-hooks.md` · Spec: react ch 02 row · Upstream: react.dev Rules of React.

- [ ] **Step 1: Read** Spec row.
- [ ] **Step 2: Draft** (110–140 lines, 9 rules). Exemplar: a custom hook — `use`-prefixed, AbortSignal-wired fetch effect with cleanup, named return object. Rules:
  - 2.1 react-hooks v6 `recommended` in the react overlay — Rules of React are correctness; compiler-powered rules (`purity`, `immutability`, `set-state-in-render`, `set-state-in-effect`, `refs`) on.
  - 2.2 `exhaustive-deps` is an error, never disabled — a lied-about dependency is a stale-closure bug on a timer.
  - 2.3 Effects synchronize with external systems only (REACT-5) — derivable data is computed in render; "derived state in an effect" is the canonical anti-pattern.
  - 2.4 Every effect returns cleanup; every fetch effect wires AbortSignal (typescript/09.5, 13.4) — unmount is a cancellation.
  - 2.5 Custom hooks: `use` prefix; single responsibility; return a named object (or tuple ≤ 2).
  - 2.6 Hooks compose top-down like functions (typescript/05.4) — page hook orchestrates feature hooks orchestrating primitives.
  - 2.7 `useRef` for non-render state; never read/write refs during render (compiler `refs` rule).
  - 2.8 No hook calls behind conditions — restructure with early-return components or split hooks (rules-of-hooks).
  - 2.9 Don't memoize by hand first — React Compiler covers the default case; `useMemo`/`useCallback` only with a profiler trace attached (cross-ref 08).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 9; exemplar 1; reasoning 9; `wc -l` 110–150.
- [ ] **Step 4: Commit:** `git add typescript-react/02-hooks.md && git commit -m "docs(typescript-react): draft 02-hooks"`

### Task 31: typescript-react/03-state-management.md

**Files:** Create `typescript-react/03-state-management.md` · Spec: react ch 03 row.

- [ ] **Step 1: Read** Spec row + `typescript/06-classes-and-data-modeling.md` (union modeling).
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: a feature's state — discriminated-union `status`, TanStack Query for the server slice, one lifted local state. Rules:
  - 3.1 Local first — state lives in the component that owns it; lift only when two components truly share it.
  - 3.2 Discriminated unions over boolean soup — `status: 'idle' | 'loading' | 'success' | 'error'`; `isLoading && isError` must be unrepresentable (typescript/06.1 in React).
  - 3.3 Server state is TanStack Query, period (REACT-3) — it is a cache with staleness semantics; copying it into `useState` forks the truth.
  - 3.4 Context for static dependencies (theme, auth client, flags) — low-frequency values; not a state manager.
  - 3.5 Zustand only for genuinely global, dynamic client state — and modeled as unions with actions; one store per domain.
  - 3.6 No Redux in new code — the boilerplate era is over; ledger entry if a legacy app needs it.
  - 3.7 State transitions are functions — `reducer`-shaped pure transitions for non-trivial state machines (`useReducer`); transitions are testable without rendering.
  - 3.8 URL is state too — filters, tabs, pagination live in the URL (shareable, restorable); router state is not duplicate-able into stores.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-react/03-state-management.md && git commit -m "docs(typescript-react): draft 03-state-management"`

### Task 32: typescript-react/04-data-fetching-and-forms.md

**Files:** Create `typescript-react/04-data-fetching-and-forms.md` · Spec: react ch 04 row.

- [ ] **Step 1: Read** Spec row + `typescript/10-api-design.md` (zod boundary).
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: one query + one mutation — queryKey factory, zod-parsed response, react-hook-form with zodResolver. Rules:
  - 4.1 Query keys come from factories — `bookingKeys.detail(id)`; string-literal keys drift and collide.
  - 4.2 Every response is zod-parsed at the fetch boundary — the server is an external system; `z.infer` types flow inward (typescript/10.7).
  - 4.3 Query functions take AbortSignal — TanStack passes it; wire it through to fetch (typescript/09.5).
  - 4.4 Mutations declare their invalidations — explicit `invalidateQueries` scope; "refetch everything" is a perf bug, stale views are a correctness bug.
  - 4.5 Optimistic updates carry rollback — onMutate snapshots, onError restores; optimism without rollback is lying to users.
  - 4.6 react-hook-form + zodResolver — one schema validates the form and types the values; uncontrolled inputs for performance.
  - 4.7 Error boundaries per route/feature — a widget's crash never takes the page; pair with query error resets.
  - 4.8 Loading UX from query states, not bespoke flags — `isPending`/`isError` drive the union (cross-ref 03.2); skeletons over spinners where layout is known.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-react/04-data-fetching-and-forms.md && git commit -m "docs(typescript-react): draft 04-data-fetching-and-forms"`

### Task 33: typescript-react/05-structure-and-routing.md

**Files:** Create `typescript-react/05-structure-and-routing.md` · Spec: react ch 05 row · Upstream: react-typescript-style-guide.com folder structure.

- [ ] **Step 1: Read** Spec row.
- [ ] **Step 2: Draft** (90–120 lines, 7 rules). Exemplar: annotated feature tree (`features/booking/` with components/hooks/api/types colocated; `common/`). Rules:
  - 5.1 Feature folders (upstream port) — everything a feature needs lives in its folder; `common/` only for genuinely shared code.
  - 5.2 No cross-feature imports except via `common/` — feature boundaries are module boundaries (madge gate applies, typescript/12.5).
  - 5.3 PascalCase component files (`BookingCard.tsx`), camelCase folders and non-component files — upstream convention, import-name parity.
  - 5.4 Route components are thin — compose feature components; data loading at the route seam (lazy boundaries land here, cross-ref 08).
  - 5.5 Colocate tests, stories, and styles with their component — travel-together principle (typescript/12.3).
  - 5.6 One styling system per project — runtime CSS-in-JS discouraged (render-path cost); Tailwind or CSS Modules acceptable; the choice is recorded in the project README.
  - 5.7 GraphQL conventions: not adopted (ledger) — if a project adopts GraphQL, upstream's `In{Feature}` naming gets revisited then.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 7; exemplar 1; reasoning 7; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-react/05-structure-and-routing.md && git commit -m "docs(typescript-react): draft 05-structure-and-routing"`

### Task 34: typescript-react/06-testing-react.md

**Files:** Create `typescript-react/06-testing-react.md` · Spec: react ch 06 row · Extends `typescript/11-testing.md`.

- [ ] **Step 1: Read** Spec row + `typescript/11-testing.md`.
- [ ] **Step 2: Draft** (100–130 lines, 8 rules). Exemplar: one component test — MSW handler, `getByRole`, user-event, negative-space assertion. Rules:
  - 6.1 Testing Library queries by role first — `getByRole('button', { name: … })`; role queries test what users (and screen readers) can reach (REACT-4 reinforced structurally).
  - 6.2 user-event over fireEvent — real interaction sequences (focus, keydown, click), not synthetic dispatches.
  - 6.3 MSW for all network — handlers per feature; tests never know fetch exists; no `vi.mock` of API modules (typescript/11.3–11.4).
  - 6.4 Test behavior, not implementation — no snapshot-everything, no asserting on state internals or class names; assert what the user sees and does.
  - 6.5 Async UI via `findBy*`/`waitFor` with bounded timeouts — no arbitrary sleeps (typescript/11.8 determinism).
  - 6.6 Playwright e2e for critical flows — auth, checkout, the money paths; few, fast, owned by the feature.
  - 6.7 Accessibility assertions in component tests — names, roles, focus order; the a11y chapter's rules get teeth here (cross-ref 07).
  - 6.8 Negative space in UI tests — assert the error did NOT render after success, the button IS disabled while pending (typescript/11.9).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 8; exemplar 1; reasoning 8; `wc -l` 100–140.
- [ ] **Step 4: Commit:** `git add typescript-react/06-testing-react.md && git commit -m "docs(typescript-react): draft 06-testing-react"`

### Task 35: typescript-react/07-accessibility.md

**Files:** Create `typescript-react/07-accessibility.md` · Spec: react ch 07 row (beyond upstream — a11y is correctness).

- [ ] **Step 1: Read** Spec row.
- [ ] **Step 2: Draft** (90–120 lines, 7 rules). Exemplar: an accessible form field group — label, description, error wiring via `aria-describedby`, keyboard-reachable action. Rules:
  - 7.1 Semantic HTML first — `button`, `nav`, `label`; ARIA is the fallback, not the default ("no ARIA is better than bad ARIA").
  - 7.2 Every interactive element is keyboard-reachable and visibly focused — tab through every new UI before review.
  - 7.3 Every input has a programmatic label; errors wire via `aria-describedby` — placeholder is not a label.
  - 7.4 jsx-a11y in the react overlay — mechanical floor (correctness-justified plugin, with react-hooks).
  - 7.5 Images, icons, and media carry text alternatives — decorative images say so (`alt=""`).
  - 7.6 Color is never the only signal — pair with text/icon; contrast meets WCAG AA.
  - 7.7 Role-query testing is the enforcement loop — if a test can't find it by role, a user can't either (cross-ref 06.1).
- [ ] **Step 3: Verify:** `grep -c '^### '` → 7; exemplar 1; reasoning 7; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-react/07-accessibility.md && git commit -m "docs(typescript-react): draft 07-accessibility"`

### Task 36: typescript-react/08-react-performance.md

**Files:** Create `typescript-react/08-react-performance.md` · Spec: react ch 08 row · Extends `typescript/15-performance.md`.

- [ ] **Step 1: Read** Spec row + `typescript/15-performance.md`.
- [ ] **Step 2: Draft** (90–120 lines, 7 rules). Exemplar: a measured fix — profiler trace identifies re-render storm → state colocation fix → trace after. Rules:
  - 8.1 React Compiler first — automatic memoization is the baseline; hand `memo`/`useMemo`/`useCallback` only with a profiler trace attached (the optimization ledger, typescript/15.10).
  - 8.2 Re-render hygiene by structure — colocate state (03.1), push state down, lift content up (children don't re-render when the wrapper's state changes).
  - 8.3 `lazy()` + Suspense at route boundaries — code arrives when the route does.
  - 8.4 Virtualize long lists — hundreds of rows is a windowing problem, not a rendering problem.
  - 8.5 Bundle budget in CI — analyzer diff on PRs; a dependency that doubles the bundle is a perf regression in review terms.
  - 8.6 Images and fonts are perf — dimensions set (no layout shift), modern formats, lazy below the fold.
  - 8.7 Measure with React DevTools Profiler + Web Vitals — LCP/INP/CLS are the user-truth metrics; optimize against traces, not vibes.
- [ ] **Step 3: Verify:** `grep -c '^### '` → 7; exemplar 1; reasoning 7; `wc -l` 90–130.
- [ ] **Step 4: Commit:** `git add typescript-react/08-react-performance.md && git commit -m "docs(typescript-react): draft 08-react-performance"`

### Task 37: Root README row — typescript-react

**Files:** Modify `README.md` (after the typescript-node row).

- [ ] **Step 1: Add row:** `| TypeScript + React | [\`typescript-react/\`](./typescript-react/) | Extends \`typescript/\`; [react.dev](https://react.dev) rules + [react-typescript-style-guide.com](https://react-typescript-style-guide.com/) | React 19+, React Compiler, TanStack Query, Testing Library + MSW + Playwright, a11y as correctness. |`
- [ ] **Step 2: Verify:** `grep -c 'typescript-react/' README.md` ≥ 1.
- [ ] **Step 3: Commit:** `git add README.md && git commit -m "docs: index typescript-react guide in root README"`

# Phase D — Family verification & publish shaping

### Task 38: Family-wide verification sweep

**Files:** none created — verification only.

- [ ] **Step 1: Counts.** `ls typescript/*.md | wc -l` → 16; `ls typescript-node/*.md | wc -l` → 9; `ls typescript-react/*.md | wc -l` → 9.
- [ ] **Step 2: Format contract sweep.** For every chapter file: `grep -L '^## What good looks like' typescript/0*.md typescript/1*.md typescript-node/0*.md typescript-react/0*.md` → empty output (no file lacks an exemplar); `grep -L 'Reasoning, step by step' …same globs…` → empty.
- [ ] **Step 3: Link check.** Extract local links and verify targets exist: `grep -rhoE '\]\((\./|\.\./)[^)]+\)' typescript* README.md | sort -u` then `ls` each target — every referenced path exists (fix any broken).
- [ ] **Step 4: Sizing bands.** `wc -l typescript/*.md | tail -1` → total in 1,600–2,700 (incl. README); `wc -l typescript-node/*.md | tail -1` and `wc -l typescript-react/*.md | tail -1` → each in 850–1,150.
- [ ] **Step 5: Spec-coverage read-through.** Open the Spec chapter tables side-by-side with each README index — every spec row has its chapter; every deviation in the Spec ledger appears in a README ledger. Fix gaps before proceeding.
- [ ] **Step 6: Root README.** Three rows present, table renders, row format matches existing rows.

### Task 39: Reshape into three publish commits

**Files:** history only. The branch holds the docs commits plus ~37 dev commits; the Spec mandates three publish commits (each phase's row lands with its phase). Strategy: replay tree snapshots — each phase-final dev commit's tree IS the correct cumulative state, so committing successive snapshots produces exactly the per-phase deltas. No interactive rebase needed.

- [ ] **Step 1: Confirm clean tree and capture SHAs.**
  `git status` → clean.
  `DOCS=$(git log --format=%H --reverse main..HEAD -- docs/ | tr '\n' ' ')` (the spec/refinement/plan commits)
  `T17=$(git log --format=%H -n1 --grep 'index typescript guide')`
  `T27=$(git log --format=%H -n1 --grep 'index typescript-node guide')`
  `T37=$(git log --format=%H -n1 --grep 'index typescript-react guide')` — expect three non-empty SHAs.
- [ ] **Step 2: Build the publish branch.** `git checkout -b feat/ts-styleguide-publish main` then `git cherry-pick $DOCS` (docs commits replay cleanly — they touch only `docs/`).
- [ ] **Step 3: Publish commit 1 (snapshot of T17).** `git checkout $T17 -- . && git add -A && git commit -m "feat: typescript style guide"` — body 3–5 lines (15-chapter core on Google TS guide + ts.dev/style + gts; erasable syntax only; exemplar-first chapter format), per git-and-code-review.md. Delta = `typescript/` + its README row only, because T17's tree predates the other phases.
- [ ] **Step 4: Publish commit 2 (snapshot of T27).** `git checkout $T27 -- . && git add -A && git commit -m "feat: typescript-node style guide"` — body: Node extension; Fastify, Drizzle, pino, crash-only, event-loop budget.
- [ ] **Step 5: Publish commit 3 (snapshot of T37).** `git checkout $T37 -- . && git add -A && git commit -m "feat: typescript-react style guide"` — body: React extension; compiler-first, TanStack Query, a11y as correctness.
- [ ] **Step 6: Prove equivalence, then swap branches.** `git diff feat/typescript-styleguide feat/ts-styleguide-publish` → **empty** (identical trees is the correctness proof). Then: `git branch -m feat/typescript-styleguide feat/typescript-styleguide-dev && git branch -m feat/ts-styleguide-publish feat/typescript-styleguide`. Keep the `-dev` branch until merge, then delete both per repo rules.
- [ ] **Step 7: Verify final history.** `git log --oneline main..feat/typescript-styleguide` → docs commits + exactly 3 feat commits; each feat commit's `--stat` touches only its guide directory + README.

---

## Self-Review (run after writing — completed inline)

1. **Spec coverage:** every Spec chapter row maps to Tasks 2–16 / 19–26 / 29–36; READMEs Tasks 1/18/28; ledger entries in Tasks 1, 21, 28, 33; root rows Tasks 17/27/37; phasing Task 39. ✓
2. **Placeholder scan:** every task carries its full rule inventory; helpers that chapters must define (invariant, assertNever, toError, Result, mapWithConcurrency) are named with size and signature in their owning task. ✓
3. **Consistency:** chapter filenames in tasks match README index titles (Task 1/18/28) and root rows; rule cross-references use final paths; verify-counts match each task's inventory length. ✓
