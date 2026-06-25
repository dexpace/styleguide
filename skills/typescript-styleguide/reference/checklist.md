# TypeScript styleguide — full checklist

One section per chapter. Read on demand for a full audit; the SKILL digest covers the quick edit.

### 01 — Formatting and Tooling

- `gts` is the toolchain; no standalone Prettier or ESLint config. Bootstrap with `bunx gts init`.
- Prettier defaults as shipped by gts are final; never override `printWidth`, `singleQuote`, anything.
- tsconfig extends the gts base and adds exactly six strictness flags: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `isolatedModules`, `verbatimModuleSyntax`, `erasableSyntaxOnly`. Platform `module`/`moduleResolution`/`lib` live with the runtime guide.
- TypeScript 5.8 or newer; pin a caret range and bump deliberately.
- `bun install` against a committed `bun.lock`; `--frozen-lockfile` in CI. Pin Bun via `.bun-version`.
- One ESLint overlay extending gts with `strict-type-checked` + `stylistic-type-checked`; every rule traces to a chapter.
- `max-lines-per-function` is 70 (`skipComments: true`, blank lines count).
- `max-depth` is 3, `max-params` is 3.
- Pre-commit runs `gts lint` and `tsc --noEmit`; CI runs the same.
- Every `eslint-disable` carries a same-line reason.

### 02 — Naming Conventions

- Apply Google's identifier casing verbatim: `lowerCamelCase` members, `UpperCamelCase` types, `CONSTANT_CASE` deep constants.
- Encode visibility with keywords, not glyphs: no `I` prefix, no leading/trailing underscore; use `private`/`protected`/`#`.
- Reserve `CONSTANT_CASE` for deeply immutable, conceptually constant values; a mutable singleton stays camel.
- Name files in kebab-case (`user-client.ts`); case-insensitivity-safe, ecosystem norm.
- Spell names out beyond the idiomatic set (`id`, `url`, `ctx`, `i`); `request` not `req`.
- Draw resource verbs from the client taxonomy: `get`/`list`/`create`/`upsert`/`update`/`delete`/`begin`, each with its documented contract.
- Omit the `Async` suffix; the `Promise` return already announces asynchrony.
- Write booleans as predicates (`is`/`has`/`can`/`should`); avoid negative stems.
- Design the name for the call site; write the call first; don't restate module context.
- Carry units in quantity names (`timeoutMs`, `sizeBytes`); unit last as a suffix.
- Name type parameters by role clarity: a single letter when obvious, else a `T`-prefixed name.

### 03 — The Type System

- Treat the strict flag family as law; never soften a flag per-project.
- Ban `any`; receive external input as `unknown` and narrow inward before the interior sees it.
- Ban `@ts-ignore`; allow `@ts-expect-error` only with a reason, only in tests and declared bridges.
- Require a why-comment on every `as`; prefer `satisfies`, guards, or a parse.
- Represent absence as `undefined`; convert external `null` to `undefined` at the boundary.
- Prefer optional `?` over `| undefined` in object types (`exactOptionalPropertyTypes` enforces the difference).
- Narrow with the weakest tool that works: discriminant, then `typeof`/`instanceof`/`in`, then a custom guard.
- Unit-test every custom type guard (`x is T`) with positive and negative cases.
- Brand domain primitives in high-rigor modules; generate only through a validating constructor (the one sanctioned `as`).
- Put `readonly`/`ReadonlyArray`/`Readonly<T>` in every public signature.
- Constrain every generic; add no gratuitous type parameters; annotate variance (`in`/`out`) on public generic interfaces.
- Write erasable syntax only: no `enum`, runtime `namespace`, parameter properties, or `import =`. Consume a codegen `enum` only at the boundary, convert to a union inside.

### 04 — Variables and Declarations

- Default to `const`; spend a `let` only when reassignment is the visible point; never `var`.
- Non-null `!` is banned outside tests and `// bridge:` lines; prove presence with `invariant`.
- Write `as const` on literal configuration so discriminants and tuples keep their narrow types.
- Declare at first use, one declaration per line.
- No chained assignment (`a = b = c`).
- Destructure at boundaries with defaults; at most two levels deep.
- No module-level mutable state; own state in an instance, a store, or arguments. A frozen `const` is fine.
- No shadowing, including outer scopes, imports, and built-ins.

### 05 — Functions

- 70 lines, hard cap, aim 10–30; every callable counts including arrow callbacks. Split on the "and".
- One level of abstraction per function: orchestrate named steps or do primitive work, never both.
- Guard clauses first; happy path flush left; an `else` is usually a missed guard.
- Step-down rule: callers above callees. Use top-level `function` declarations; reserve arrows for callbacks.
- Options object for ≥3 parameters or any boolean.
- Use the project-wide `invariant(cond, msg): asserts cond` helper throwing `InvariantViolation`.
- Assert aggressively: 2+ per function on average; positive and negative space; pair assertions.
- Pure by default; push effects to the edges; name the effect with an effect verb.
- No control-flag parameters that fork the body; split into two named functions.
- Overloads only when the return type depends on input shape; else a union parameter or generic.
- Explicit return types on every exported function.
- Paragraph the body with blank lines: guards, computation, postconditions, return.

### 06 — Classes and Data Modeling

- Make illegal states unrepresentable; model variants as union members, not optional-field bags.
- Use `interface` for object shapes and `type` for everything else (unions, mapped, tuples).
- Model data as plain objects and free functions; reserve classes for stateful lifecycle resources.
- Use `extends` only for `Error` hierarchies; compose by explicit delegation everywhere else.
- Model closed polymorphism as a discriminated union with an exhaustive `switch` closed by `assertNever`.
- Make fields `readonly` by default; accept `Readonly<T>`/`ReadonlyArray<T>` in public signatures.
- Prefer the `private` modifier over `#private` fields; reach for `#` only when runtime privacy is the requirement.
- Constructors assign; `create*` factories validate (parse, don't validate).
- Make value objects frozen plain objects compared by a free `equals` helper, never an `.equals` method.
- Lift co-traveling optionals into the union.
- Brand IDs at domain edges so they cannot be interchanged.

### 07 — TypeScript Idioms

- Use `satisfies` to validate a value without widening it.
- Use `as const` to freeze a literal into its narrowest shape (the sanctioned `enum` replacement).
- Default with `??`; never default with `||`.
- Reach for `?.`, but never chain past a lying type; two-plus `?.` means parse the input.
- Build transforms with `map`/`filter`/`reduce`; use `for…of` only for effects or early exit. `for…in` banned, `.forEach` discouraged.
- Name the steps once a chain passes about three stages.
- Use `Map`/`Set` for dynamic keys; reserve objects for known shapes.
- Clone with `structuredClone`, not a JSON round-trip.
- Compose strings with template literals; type string contracts with template literal types.
- Update immutably with spread; push nested updates into small named helpers (spread before `.sort()`).
- Branch on a discriminant with an exhaustive `switch`, never an `if`-chain over `kind`.
- Write no clever one-liners; if it needs a second read, it needs a second line.

### 08 — Error Handling

- Define domain `Error` subclasses with typed `readonly` context fields; set `this.name = new.target.name`; keep the tree two levels deep.
- Pass `cause` on every rethrow; run the caught value through `toError` first.
- Type every `catch` as `unknown` and narrow it; `toError` must not throw from inside a catch.
- Never swallow an error: handle, wrap-and-rethrow with `cause`, or don't catch. The deliberate ignore is narrowed and commented.
- Wrap at boundaries; log once, at the top, with the full `cause` chain and `correlationId`.
- Use the `Result` union opt-in per module; never mix it with throwing. Freeze the wrapper.
- Separate programmer errors (crash with `invariant`/`assertNever`) from operational errors (typed error or `Result`).
- Put context in messages, never secrets; mask to the minimum identifying fragment; structured fields on the object.
- Don't use exceptions for control flow; absence is `T | undefined`, a yes/no is `boolean`.
- Document failure modes of every public API with `@throws` or a `Result` signature.
- Report fan-out failures with `AggregateError` over `Promise.allSettled`, never first-failure-wins.

### 09 — Concurrency

- Know the event-loop model: one thread, every `await` a suspension point; push CPU-bound spans off the loop.
- Use `async`/`await`; reserve `.then`/`.catch` for interop edges.
- Float no promises: `await`, `return`, or `void` with a why-comment. `no-floating-promises` + `no-misused-promises`.
- Never wrap async work in the `Promise` constructor; the only sanctioned use bridges a callback API with a sync executor.
- Make cancellation part of the signature: `{ signal }: { signal?: AbortSignal }` on every long-running async export.
- Put a timeout on every external I/O call via `AbortSignal.timeout(ms)`; combine with the caller's signal using `AbortSignal.any`.
- Bound fan-out with a project-wide worker pool (`mapWithConcurrency`); never naked `Promise.all` over unbounded input.
- Don't `await` in a loop for independent work; batch instead. Comment legitimate serial work.
- Reach for `Promise.allSettled` when partial failure is acceptable; inspect every rejection, aggregate into `AggregateError`.
- Treat interleaving as a real race: re-validate invariants after an `await`, or restructure to single ownership.
- Propagate signals downward to the I/O primitive; `signal.throwIfAborted()` in CPU loops between awaits.
- Bound every queue, buffer, and in-flight map with a named constant asserted via `invariant`.

### 10 — API Design

- Export named symbols only; never `export default` (except where a framework demands it).
- `index.ts` is the contract; everything else is internal. Re-export at the boundary only.
- Export the least; start private and promote deliberately.
- Accept interfaces (narrowest shape used); return concrete `readonly` types.
- Options objects carry documented `@default`s; callers state only deviations; the object and its fields are optional and `readonly`.
- Keep API symmetry: parallel operations share vocabulary, parameter shape, and return shape.
- Put a zod schema at every external boundary; `z.infer` is the single source of type truth; call `.readonly()`.
- Document failure modes (`@throws` or `Result`) and accept cancellation (`{ signal }`) on every public operation.
- Deprecate with `@deprecated` plus a migration path and removal version; breaking changes are MAJOR (semver).
- Stream large or paginated results as `AsyncIterable<T>`, never an eager array; the `{ signal }` aborts the in-flight fetch.

### 11 — Testing

- Run `bun test` with explicit imports from `bun:test`; colocate unit tests, isolate integration tests under `tests/`.
- Arrange, act, assert; one behaviour per test; name it `<verb-phrase> when <condition>`.
- Fake your own interfaces (real in-memory behaviour); reserve `mock.module` for true externals; name a fake a fake.
- Test server HTTP by invoking the app directly (`app.request`); reserve MSW for React component tests; never reassign `globalThis.fetch`.
- Property-based tests (`fast-check`) are mandatory for codecs, parsers, serializers, and invariant-bearing functions: round-trip, idempotence, order-insensitivity, bounds.
- Type-level tests (`expectTypeOf` from `expect-type`) are mandatory for public generics and conditional types; the gate is `tsc --noEmit`.
- Give every custom type guard a truth-table test with mandatory negative space.
- Make every test deterministic: control time via `setSystemTime`, seed `fast-check` and log the seed, no real network/clock/fs.
- Assert positive and negative space; the pair-assertion verifies one property two ways.
- Share no mutable fixtures; build fresh in `beforeEach` or a factory; let no test depend on another.
- Report coverage (lcov), never target it; mutation testing is a recorded accepted gap on Bun.
- Treat the test as the first caller; an unergonomic test is an API smell — fix the production code.

### 12 — Module Organization

- Ship ESM (`"type": "module"`); confine CJS to a named, documented bridge module.
- Write `import type` for every type-only import; prefer the statement-level form.
- Group files by feature, not by technical kind; keep `shared/` thin.
- Put barrels only at the package boundary; internally import the file you mean.
- Treat an import cycle as a bug; gate with `madge --circular` in CI.
- Forbid import-time side effects; declare `"sideEffects": false`; expose an explicit `init()` where an effect is genuinely needed.
- Order imports in three blank-line-separated groups: `node:` → external → internal, alphabetized; `node:` prefix mandatory.
- Keep relative imports shallow: at most two levels up; restructure rather than alias over depth.
- Write each module so it reads top-down as a story; one concept per file; ~300 lines is the consider-splitting signal.

### 13 — Resource Management

- `using`/`await using` for every disposable; the scope is the lifetime. Needs `"lib": ["es2023", "esnext.disposable"]` (runtime guide).
- Owned resources implement `Symbol.dispose`/`Symbol.asyncDispose`; disposal is idempotent; post-dispose use asserts via `invariant`.
- `DisposableStack`/`AsyncDisposableStack` for composite teardown; constructors that acquire commit with `.move()`.
- `AbortController` is the universal lifecycle handle; thread `signal` down; compose with `AbortSignal.any`; register listeners with `{ signal }`.
- Every `setTimeout`/`setInterval` has an owner and a clear cleanup path (signal, `defer`, or returned disposer).
- Pools, caches, and queues are bounded with a named numeric `max` and a TTL where staleness is allowed; bounded checkout timeout.
- Release resources in reverse acquisition order; `DisposableStack` gives this for free.
- Never rely on the garbage collector or `FinalizationRegistry` for resource release; close explicitly.
- Tests assert cleanup: spy the disposer, assert it ran once on happy and throwing paths; `afterEach` closes what the test opened; assert release order.

### 14 — Documentation

- Use TSDoc syntax for every public API (`/** … */` with `@`-tags); skip a block on self-documenting internals.
- Never restate the type in prose; a `@param` earns its place with a constraint, unit, range, or sentinel.
- Make comments say why, not what; TODOs carry an owner and a date.
- Put an `@example` on every non-obvious public API; one realistic, compiling call site.
- Document every failure mode with `@throws` naming the type and recovery, or prefer a `Result` return.
- Give every package a README that reaches first success in 30 seconds: name, install, one example, links.
- Link, don't duplicate; prefer `{@link Symbol}` over prose cross-references.
- Treat stale comments as debt; update or delete in the same commit as the code change.

### 15 — Performance

- Optimize the slowest resource first (network > disk > memory > CPU); design-time wins dwarf V8 tricks. Profile before touching code.
- Keep object shapes stable so V8 stays monomorphic; one construction site, one field order; `interface` + factory is the default.
- Never `delete` a property and never create a sparse array; assign `undefined` to clear, or use `Map`/`Set`.
- Keep allocations out of hot paths; hoist closures, collapse pipeline stages into one pass, reuse buffers — only where a profile shows GC pressure.
- Batch independent awaits with `Promise.all`; bound large fan-out with the chapter 09 worker pool.
- Measure first and keep the benchmark (`*.bench.ts`, `mitata`); a perf fix without before/after numbers does not merge.
- Build strings with `+=` (V8 ropes); reach for `join` only with a real array of parts; measure when it matters.
- Treat JSON parse/stringify as the hot-path cost it is; do less of it; `structuredClone` over a JSON round-trip clone.
- Author every module for tree-shaking: named exports, `"sideEffects": false`, no import-time work.
- Every deliberate optimization carries the measurement comment that justifies it; unexplained cleverness is simplified away.
