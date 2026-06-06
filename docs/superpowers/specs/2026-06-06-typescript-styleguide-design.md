# TypeScript Styleguide Family — Design Spec

**Date:** 2026-06-06
**Status:** Approved (design review complete; pending spec review)
**Owner:** Omar (decisions on TS specifics delegated to Claude with reasoning recorded; revisit as TS experience grows)

## Goal

Add a TypeScript guide family to the styleguide repo:

```
typescript/          — platform-agnostic core (15 chapters, mirrors kotlin//python/ spine)
├── typescript-node/  — server runtime extension (8 chapters, mirrors kotlin-jvm/ shape)
└── typescript-react/ — UI extension (8 chapters, peer of typescript-node/)
```

Built on Google's TypeScript Style Guide at the core, adopting everything portable from the
repo's existing guides (Tiger Style overlay, 12 root rules, kotlin/python chapter spine).

## Values

**Correctness > performance > simplicity > expressiveness (no hidden behaviour).**

- Correctness implies every feasible kind of testing: unit, property-based, type-level,
  mutation, component, e2e.
- "No hidden behaviour" is root rule 2 (*explicit over implicit*) in TS terms: no `any`
  escape hatches, no implicit coercion, no decorator/DI magic, no type-space syntax that
  generates value-space code.
- This maps onto the root README's `correctness > performance > developer experience` and the
  clarity hierarchy (clarity → simplicity → concision → maintainability → consistency).

## Architecture

### Extension semantics (verbatim from kotlin → kotlin-jvm precedent)

Extensions are *additive*; they never delete or weaken a core rule. Where an extension is
stricter, it wins for that runtime. Extensions cross-reference core chapters instead of
restating them.

### Authority chains

| Guide | Canonical authority | Supplementary | Conflict rule |
|---|---|---|---|
| `typescript/` | [Google TS Style Guide](https://google.github.io/styleguide/tsguide.html) | [ts.dev/style](https://ts.dev/style/) (community adaptation), [gts](https://github.com/google/gts) (tooling embodiment) | Google official wins; ts.dev/style fills gaps; dexpace overlay on top |
| `typescript-node/` | Node.js official docs | [goldbergyoni/nodebestpractices](https://github.com/goldbergyoni/nodebestpractices) | Core guide is baseline; Node guide wins where stricter |
| `typescript-react/` | [react.dev](https://react.dev) Rules of React (correctness, non-negotiable) | [react-typescript-style-guide.com](https://react-typescript-style-guide.com/) (conventions) | React's rules > community conventions > dexpace overlay |

### Rule format (identical to kotlin/python)

- Chapters: `# NN — Title`. Rules: `### N.M — imperative title`.
- Every rule: **Reasoning, step by step** (numbered), worked good/bad examples,
  cross-references, and an enforcement note naming the compiler flag / gts / eslint rule
  where one exists.
- Tiger Style overlay throughout: 2+ assertions per function, bounded everything,
  no recursion in library code, zero technical debt.

### Deviations ledger (new repo pattern)

Each guide README carries a "Deviations from upstream" table: rule, upstream position, our
position, reasoning link. Initial entries listed per guide below. (Motivation: the go/ guide's
explicit-vs-defaults confusion would have been prevented by such a ledger.)

## Toolchain (decided)

**gts is the single tool.** `npx gts init`, `gts lint`, `gts fix`; its Prettier defaults are
final (formatting is a non-discussion). Exactly **one** overlay file extending gts's flat
ESLint config:

- `typescript-eslint` `strict-type-checked` + `stylistic-type-checked` — gts alone does not
  enable type-aware correctness rules (`no-floating-promises`, `no-misused-promises`), and
  correctness is the top value.
- `max-lines-per-function: 70` (`skipComments: true`, blank lines counted — mirrors python's
  counting rule).
- The handful of dexpace rules named in chapters (e.g. `prefer-nullish-coalescing`,
  `switch-exhaustiveness-check`), each traceable to a chapter rule.

Extensions may add **correctness-justified** plugins to their own overlay
(typescript-react/: `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`). Style-only
plugins remain banned everywhere — gts owns style.

**tsconfig** extends gts's base and adds:

| Flag | Why |
|---|---|
| `noUncheckedIndexedAccess` | Indexing returns `T \| undefined` — forces the bounds check Tiger Style wants |
| `exactOptionalPropertyTypes` | Absent ≠ explicitly-undefined; correctness at boundaries |
| `noImplicitOverride` | Explicit override intent |
| `isolatedModules` | Transpiler-safe TS (Vitest/esbuild reality) |
| `verbatimModuleSyntax` | `import type` discipline, erasability |
| `erasableSyntaxOnly` | See decision record below |

TypeScript ≥ 5.8 (required by `erasableSyntaxOnly`), prefer latest stable.
**pnpm** mandated: strict `node_modules` makes undeclared dependencies fail (explicitness),
fastest installs (performance).

## Decision record: erasable syntax only

**Decision:** ban `enum`, `namespace`, and constructor parameter properties; enforce all three
mechanically with the `erasableSyntaxOnly` compiler flag.

**Reasoning:**
1. All three are type-looking syntax that generates runtime code — "no hidden behaviour" by
   definition. Numeric enums even emit bidirectional lookup objects (`Color[0]` reverse maps).
2. Boundaries: string enums are nominal in a structural language; `JSON.parse` output can't be
   assigned without a cast. Literal unions round-trip JSON natively and plug into zod.
3. Toolchain: `const enum` is already broken under `isolatedModules`; Node's native
   type-stripping cannot execute non-erasable syntax. The platform direction is clear.
4. Performance (minor): enums emit poorly tree-shaken IIFEs; literal unions emit nothing.
5. Nothing is lost: exhaustiveness via `never` works identically; the `as const` object idiom
   restores symbol rename.

**Replacement idioms (standardized to prevent fragmentation):**
- Small closed set: bare literal union `type Color = 'red' | 'green'`.
- Need iteration or rename-ability:
  ```ts
  const Color = { Red: 'red', Green: 'green' } as const;
  type Color = (typeof Color)[keyof typeof Color];
  ```

**Carve-outs** (mirrors kotlin's "`!!` banned outside bridges"): consuming enums from
codegen/third-party libs (Prisma, gRPC, TS compiler API) is allowed at the boundary;
`declare namespace` in ambient `.d.ts` remains legal (erasable).

**Cost accepted:** two deviations from Google (enums, parameter properties); slightly more
verbose constructors.

## `typescript/` — 15 chapters

Mirrors the kotlin/python spine: same chapter number → same topic. Slot 03 is the
language-safety chapter (kotlin: nullability; python: type-hints; TS: the type system).

| Ch | Title | Key rules and decisions |
|---|---|---|
| 01 | formatting-and-tooling | gts only; overlay + tsconfig as above; TS ≥ 5.8; pnpm; pre-commit `gts lint`. |
| 02 | naming-conventions | Google casing verbatim (no `I` prefix, no underscores, CONSTANT_CASE for deep constants). Files kebab-case (ecosystem norm; case-sensitivity safe). Ports python's client verb taxonomy (get/list/create/upsert/delete/begin). No `Async` suffix (return type already says it). |
| 03 | the-type-system | `any` banned (`unknown` + narrowing); `@ts-ignore` banned, `@ts-expect-error` only with reason in tests/bridges; `as` requires why-comment, prefer `satisfies`/guards/parse. Absence = `undefined`; `null` only where external contracts force it, converted at boundary. Custom type guards must have tests (a wrong guard is a type-system lie). Branded types for domain primitives in high-rigor modules. `readonly`/`ReadonlyArray` in public signatures. |
| 04 | variables-and-declarations | `const` default, `let` justified, `var` banned. Non-null `!` banned outside tests/bridges — mirror of kotlin 3.2 (`!!`). `as const` for literal config. |
| 05 | functions | **70-line hard cap** (lint-enforced), aim 10–30. Top-level `function` declarations (hoisting clarity, named stack traces); arrows for callbacks. Options object for ≥3 params or any boolean (boolean traps). `invariant(cond, msg): asserts cond` helper (~10 lines) — assertions narrow types, so runtime checks also inform the compiler. 2+ assertions per function: preconditions at entry, postconditions at exit. |
| 06 | classes-and-data-modeling | Data = plain objects + `interface` (Google); `type` for unions/mapped/conditional. Classes only for stateful lifecycle resources; `extends` only for `Error` hierarchies. Discriminated unions are the sum type (`kind` discriminant, exhaustive `switch`, `assertNever`). `private` modifier over `#private` (Google; erasable) — `#` only when runtime privacy is required. Constructors assign only; factories hold logic; domain types validated at construction (parse, don't validate). |
| 07 | typescript-idioms | `satisfies`; `as const`; `?.`/`??` — `\|\|` banned for defaults (`0`/`''` bug class; `prefer-nullish-coalescing`). Pipelines: `map`/`filter`/`reduce` for transforms; `for…of` for effects/early-exit; `.forEach` discouraged (can't await/break; ts.dev agrees); `for…in` banned. `Map`/`Set` over object-as-dictionary. `structuredClone` over JSON round-trip. |
| 08 | error-handling | `Error` subclasses per domain; `cause` chaining mandatory on rethrow + context fields (correlation ID) — the wrap-with-context analog. `catch (e: unknown)` + narrow; no swallowing. Result pattern: hand-rolled ~10-line discriminated union, **opt-in per module, no mixing** (ports python §8.11; no library dep). Programmer errors → `invariant` (crash fast); operational errors → typed errors (handled). |
| 09 | concurrency | async/await only. **`no-floating-promises` is the flagship rule** (await, return, or explicit `void` with comment). AbortSignal threaded through long async APIs; `AbortSignal.timeout()` mandatory on external I/O (ports python asyncio.timeout). Bounded fan-out: `Promise.all` over serial awaits, concurrency-limited via ~15-line semaphore. Interleaving races documented: check-then-act across `await` is a race. |
| 10 | api-design | Named exports only; `index.ts` is the contract (deep imports blocked via `exports` field — node guide). Accept interfaces, return concrete. Options with documented defaults (root rule 2 carve-out). zod at every external boundary; `z.infer` is the single source of truth for external-data types. `@deprecated` + semver. |
| 11 | testing | **Vitest**; colocated `*.test.ts`; integration/e2e under `tests/`. Mandated: unit; property-based (fast-check) for codecs/parsers/invariant-bearing functions; type-level (`expectTypeOf`) for public generics and conditional types (type regressions are invisible to runtime tests). Recommended: Stryker mutation testing, nightly CI (honest metric; no coverage-% fetish). Fakes over mocks for owned code; MSW for HTTP. Determinism: fake timers, seeded fast-check (log seeds), no real network/clock/fs in unit tests. |
| 12 | module-organization | ESM only (`"type": "module"`); `import type` discipline (enforced by `verbatimModuleSyntax`). Feature folders over kind folders. Barrels only at package boundary; internal barrels banned (cycles, bundle cost). Cycles: `madge --circular` in CI (keeps "gts only" honest — no extra lint plugins). Module side effects banned (`sideEffects: false`). |
| 13 | resource-management | `using` / `await using` mandated for disposables; implement `Symbol.dispose` on owned resources — the `use{}`/`with` analog. AbortController as universal lifecycle handle (`addEventListener(..., {signal})` unifies cleanup and cancellation). Bounded pools/queues/caches (LRU + TTL; ports performance.md). |
| 14 | documentation | TSDoc; never restate types in prose (Google); why-comments (root rule 7); `@example` on non-obvious publics. |
| 15 | performance | V8 mental model: monomorphism (stable shapes — `interface` discipline buys this), no `delete`, no sparse arrays. Allocation hygiene in hot paths. Serial-await elimination (cross-ref 09). `vitest bench` micro; `--cpu-prof` real profiles; measure first; network > disk > memory > CPU (ports performance.md). |

README: authority chain, chapter index, the 12 root rules restated TS-natively, deviations
ledger.

## `typescript-node/` — 8 chapters

README adds NODE-1…6 (mirroring JVM-1…6): crash-only error policy; event-loop budget; every
boundary parsed; dependencies are attack surface; named lifecycles; ESM-native.

| Ch | Title | Key rules and decisions |
|---|---|---|
| 01 | runtime-and-modules | Node Active LTS (≥ 24); `engines` + `packageManager` pinned (corepack). `node:` prefix imports mandatory (explicit). Subpath `#imports` over bundler aliases. Compile with tsc for prod (no type stripping in prod yet). |
| 02 | concurrency-and-event-loop | `*Sync` APIs banned in request paths. CPU work → `worker_threads` via piscina (bounded pool). Backpressure via `stream.pipeline`/async iterators. `unhandledRejection` → log + exit 1 (crash-only; supervisor restarts). Graceful shutdown: SIGTERM → stop accepting → drain (deadline-bounded) → close pools → exit. |
| 03 | http-services | **Fastify** (schema-first validation = parse-don't-validate built-in; explicit plugin model, no DI container; ~2–3× express throughput). Express = legacy maintenance only. **NestJS rejected**: decorator/DI magic violates root rule 2 and is incompatible with `erasableSyntaxOnly`. Hono acceptable for edge. Correlation-ID middleware (ports kotlin-jvm 06). Centralized error mapping. |
| 04 | persistence | **Drizzle** primary (SQL visible, types from schema, no codegen engine, no query magic); raw `pg` acceptable; Prisma only with justification. Repositories as plain functions over a db handle (data + functions). Explicit transaction scopes. Forward-only versioned migrations. Pool sizes explicit and bounded. |
| 05 | serialization-and-validation | zod for env config (parse `process.env` at startup → frozen typed config, fail fast), all request/response bodies, all external JSON (never raw `JSON.parse` into a typed value). ISO-8601 strings at boundaries; Temporal when natively available, `date-fns` until then. |
| 06 | logging | **pino** (structured JSON, fastest, built-in `redact` for PII — ports security.md masking). AsyncLocalStorage for request context — the MDC analog (kotlin-jvm 06 parity). Log once at the boundary. `console.log` banned in server code. |
| 07 | node-performance | Event-loop lag as first-class metric (`monitorEventLoopDelay`) with stated budget. `--cpu-prof`/clinic/0x. undici pools (bounded). fastify schema-derived `fast-json-stringify`. autocannon load tests. One process per container; scale horizontally. |
| 08 | build-and-distribution | Libraries: plain tsc emit + `publint` + `arethetypeswrong`. Services: esbuild single artifact. `exports` field locked (no deep imports). api-extractor API reports — the binary-compatibility-validator analog. npm provenance. Lockfile committed. Minimal-dependency policy (security.md tie-in). |

## `typescript-react/` — 8 chapters

README adds REACT-1…6: render purity is law; hooks rules are compiler-checked; server state ≠
client state; a11y is correctness; effects synchronize, never derive; composition over
configuration.

| Ch | Title | Key rules and decisions |
|---|---|---|
| 01 | components-and-props | Function components only. `interface XProps`. **No `React.FC`** (implicit-children legacy, worse inference). Class components only via `react-error-boundary`. Composition (children/slots) over boolean-config props. |
| 02 | hooks | `eslint-plugin-react-hooks` added to the react overlay (justified: Rules of React are correctness, not style; includes React Compiler lint). `exhaustive-deps` as error. Effects synchronize with external systems only (react.dev canon: you might not need an effect). Custom hooks `use`-prefixed. Fetch effects take AbortSignal (cross-ref core 09/13). |
| 03 | state-management | Local-first `useState`; lift minimally. Discriminated-union state over boolean soup (`status: 'idle' \| 'loading' \| 'error'` — illegal states unrepresentable). **TanStack Query for server state** (it's a cache, not state). Context for static DI; Zustand only for genuinely global dynamic state; no Redux in new code. |
| 04 | data-fetching-and-forms | TanStack Query conventions (queryKey factories; zod-parse every response at the boundary — core 10). react-hook-form + zod resolver (one schema = validation + types). Error boundaries per route. |
| 05 | structure-and-routing | Ports upstream's feature folders (`features/x/` with colocated hooks/components/queries; `common/` only when shared). PascalCase component files; camelCase folders (upstream convention). No cross-feature imports except via `common/`. One styling system per project; runtime CSS-in-JS discouraged (perf); Tailwind or CSS Modules acceptable. GraphQL chapter **not adopted** (no dexpace GraphQL; ledger entry for revisit). |
| 06 | testing-react | Testing Library with role-based queries (a11y-aligned). user-event. **MSW** (network-level fakes; no fetch mocks). Playwright for critical-flow e2e. Component tests colocated. |
| 07 | accessibility | Added beyond upstream (a11y is correctness for UI): semantic HTML first; keyboard paths tested; jsx-a11y lint; role-query testing reinforces structurally. |
| 08 | react-performance | React Compiler first; manual `memo`/`useMemo` only when profiler proves it. `lazy()` at route boundaries. List virtualization. Bundle budget + analyzer in CI. |

## Deviations ledger (initial entries)

| Guide | Rule | Upstream position | Ours | Why |
|---|---|---|---|---|
| typescript/ | `enum` | Google allows (bans `const enum`) | Banned (erasable-syntax stance) | Hidden runtime emit; JSON boundary friction; platform direction |
| typescript/ | Constructor parameter properties | Google allows | Banned | Hidden field declaration+assignment inside a signature |
| typescript/ | Function size | No upstream cap | 70-line hard cap | Owner decision; Tiger Style; deliberately Go-level, not scaled down |
| typescript-react/ | GraphQL conventions | Upstream has a chapter | Not adopted | No dexpace GraphQL; revisit on adoption |

## Sizing targets

- `typescript/`: kotlin–python band (~1,500–2,500 lines total).
- Each extension: kotlin-jvm band (~900 lines).
- Every chapter: same density as kotlin/python (~8–14 rules, reasoning chains, worked examples).

## Repo integration

- Root README "Per-language guides" table: three new rows (typescript, typescript-node,
  typescript-react) with authorities and notes.
- Cross-cutting docs (security.md, performance.md) untouched — adding TS examples there is a
  separate follow-up.
- No .gitignore changes needed (already covers Node).

## Implementation phasing (three commits, mirroring repo history)

1. `feat: typescript style guide` — typescript/ (16 files) + root README row.
2. `feat: typescript-node style guide` — typescript-node/ (9 files) + root README row.
3. `feat: typescript-react style guide` — typescript-react/ (9 files) + root README row.

Each commit leaves the repo consistent (per "Applying Style Changes": whole-package
consistency per commit).

## Out of scope

- TS examples in security.md / performance.md (follow-up).
- typescript-deno/, typescript-bun/ runtime guides.
- Monorepo tooling guidance (nx/turborepo) — revisit if dexpace adopts a monorepo.
- CI pipeline definitions (the guides name the checks; wiring them is per-project).

## Open questions

None blocking. Owner note: decisions on TS specifics were delegated and are individually
reversible; the deviations ledger and per-rule reasoning exist precisely so future revisions
(post TS experience) can be made surgically.
