## TypeScript on Bun

Binding style rules for **server-side TypeScript on Bun**. This guide *extends* [typescript/](../typescript/) for the Bun runtime. It is additive: where this guide is stricter, it wins for Bun code; the core guide remains the canonical language baseline.

The core guide is platform-agnostic. Bun has its own contract — a single-threaded event loop on JavaScriptCore, a process supervised by something that can restart it, an ecosystem of untyped dependencies, untrusted I/O at every edge, and a runtime that executes TypeScript by stripping types without ever checking them. Those impose rules the language guide cannot address.

### Authorities

This guide extends and defers to an ordered chain. Where they conflict, the higher authority wins:

1. **[Bun official documentation](https://bun.com/docs)** — canonical. The API reference, the runtime and bundler semantics, the package manager and test runner contracts, the JavaScriptCore execution model. Where our guidance collides with documented runtime behaviour, the runtime wins.
2. **Community best practices** — the production canon (the Node best-practices tradition, carried where Bun's `node:` compatibility makes it apply). It fills the gaps the official docs leave open on production operation: error policy, security, project structure. Defer to it only where the official docs are silent.
3. **This guide's overlay** — the dexpace Bun layer: crash-only error policy, event-loop budgets, parse-every-boundary, dependency discipline, named lifecycles, typecheck as a separate gate.

The core [typescript/](../typescript/) authority chain (Google → ts.dev → gts → its overlay) still governs the *language* layer. This guide adds the runtime layer on top; it does not replace it.

### When Bun, and when not

Bun is the dexpace default server runtime. Two facts shape how this guide treats it. First, **there is no LTS policy**: you pin the exact version (`.bun-version` plus the CI image) and upgrade deliberately, never floating. Second, Bun targets **near-complete Node compatibility** — most `node:` builtins work, so the runtime-agnostic doctrine carries directly, while the native surface (`Bun.serve`, `Bun.SQL`, `bun:sqlite`, `Bun.build`) is what this guide leans into. Node-only projects are the exception, not the rule; the retired Node guide lives at the tag `node-guide-final` for anyone who still needs it.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Runtime & Toolchain](./01-runtime-and-toolchain.md) | `bun install` + committed `bun.lock`, exact version pin (`.bun-version`), `bunfig.toml`, native TS execution + `tsc --noEmit` gate, `node:` builtins, `moduleResolution: "bundler"`, ESM-only, one process per container |
| 02 | [Concurrency & the Event Loop](./02-concurrency-and-event-loop.md) | No blocking in request paths, no `*Sync`, event-loop lag measured, bounded Bun Workers for CPU, Web Streams with backpressure |
| 03 | [HTTP Services](./03-http-services.md) | Hono on `Bun.serve`, thin handlers, zod validator middleware, centralized error mapping, correlation IDs, timeouts, `app.request()` injection testing, honest health checks |
| 04 | [Persistence](./04-persistence.md) | `Bun.SQL` (Postgres) with Drizzle, bounded pools, parameterized tagged-template queries, `bun:sqlite` for embedded, transactions as units, rows parsed at the edge, migrations |
| 05 | [Serialization & Validation](./05-serialization-and-validation.md) | zod at every boundary, types from `z.infer`, never trust `JSON.parse`, `Bun.env` parsed once, null-vs-absent semantics, Bun-native file/S3 binary boundaries, no entity leakage |
| 06 | [Logging](./06-logging.md) | Structured JSON (pino on Bun), one logger, correlation via `AsyncLocalStorage`, PII redaction, levels, stdout-JSON fallback, no `console.log` |
| 07 | [Bun Performance](./07-bun-performance.md) | `bun:jsc` profiling, `Bun.nanoseconds()`, mitata micro-benchmarks, allocation hygiene, Web Streams over buffer, Bun fetch pooling, JSC-vs-V8 carry/don't-carry, slowest-resource-first, measure before tuning |
| 08 | [Build & Distribution](./08-build-and-distribution.md) | `Bun.build` for bundling, `bun build --compile` single-file executables, locked `exports`, publint + attw gate, api-extractor surface diff, `bun publish` to npm, changesets, reproducible builds |

---

## Runtime Substitutions

Bun is a drop-in for much of the family's tooling, not all of it. Each row below records a core or family rule, the Bun substitution that replaces it for server code, and the intent that is preserved unchanged. The *intent* is the contract; the tool is the implementation.

| Core / family rule | Bun substitution | Preserved intent |
|---|---|---|
| esbuild services (retired Node guide, tag `node-guide-final`) | `Bun.build` | One bundler for service builds, locked outputs, reproducible artifacts |
| V8 guidance (core [15.2–15.3](../typescript/15-performance.md)) | JavaScriptCore profile (chapter [07](./07-bun-performance.md)) | Allocation hygiene, batching, and measure-first carry; V8 hidden-class specifics do not — see chapter 07's carry/don't-carry table |

`bun install` and `bun test` are family defaults since the core flip (core [1.5](../typescript/01-formatting-and-tooling.md), [11.1](../typescript/11-testing.md)) — they are recorded there, not substitutions here.

**Counter-substitution.** Server code uses `bun test` (core [11.1](../typescript/11-testing.md)), but React component tests keep **Vitest + Testing Library + MSW**: `bun test` cannot run MSW and its DOM story is immature. That deviation belongs to the UI layer and is recorded in the [typescript-react README](../typescript-react/README.md) ledger, not here.

---

## The BUN rules

These add to [the 12 rules in the core guide](../typescript/README.md#the-12-rules-in-typescript). Each extends one core rule to the runtime's edges. When a BUN rule conflicts with a core one, the BUN rule wins for Bun code.

### BUN-1. Crash-only error policy.

**Step-by-step reasoning:**
1. An `unhandledRejection` or `uncaughtException` means the process reached a state you did not design. You cannot reason about it, so you cannot safely continue from it.
2. Continuing is the dangerous option, not the safe one. A process limping on with a half-applied write or a corrupt in-memory cache produces silent wrong answers, which are worse than downtime.
3. The handler does exactly three things: log the error with its full `cause` chain, flush the logger, exit with code 1. No recovery logic, no "try once more".
4. A supervisor (systemd, Kubernetes, pm2) owns liveness and restarts a clean process. Restartability is a design requirement, not an afterthought.
5. This extends [core 8](../typescript/08-error-handling.md): operational errors are still handled in place where they occur. Only programmer errors that escape all the way to the process boundary trigger the crash.

### BUN-2. The event loop has a budget.

**Step-by-step reasoning:**
1. One thread serves every request. A synchronous operation that takes 200ms does not slow one client by 200ms; it freezes every concurrent client for 200ms.
2. So the loop is a bounded, shared resource and is treated like one. No `fs.*Sync`, `execSync`, or synchronous crypto on any request path.
3. CPU-bound work (hashing, compression, large parses) moves to bounded Bun Workers or out of process entirely. The request thread stays free to dispatch.
4. The budget is measured, not assumed. Event-loop lag is sampled, exported as a metric, and alerted before it surfaces as latency to clients.
5. This extends [core 15](../typescript/15-performance.md), performance from the outset, applied to the runtime's defining constraint.

### BUN-3. Every boundary is parsed.

**Step-by-step reasoning:**
1. Anything crossing from outside into the domain is untyped until proven otherwise: environment variables, HTTP requests, queue messages, third-party responses, database rows.
2. A raw `JSON.parse` result is `any` wearing a disguise. Trusting its shape is the same mistake as `any`, just relocated to runtime.
3. Each boundary value passes through a zod schema *first*. Invalid input is rejected at the edge, before any domain code touches it.
4. The domain type is derived from the schema with `z.infer`, never hand-written alongside it. One source of truth means the validator and the type cannot drift apart.
5. This extends [core 10.7](../typescript/10-api-design.md), parse at boundaries, from the API surface to every edge the runtime touches, including `Bun.env` itself.

### BUN-4. Dependencies are attack surface.

**Step-by-step reasoning:**
1. Every dependency is code you did not write, running with your process's full privileges. The transitive tree is the real surface, not the top-level list.
2. Keep the set minimal. Prefer the `node:` standard library, Bun's native APIs, or a few lines of owned code over pulling a package and its subtree.
3. Pin versions with a committed `bun.lock` and audit the tree with `bun audit` in CI — it scans the lockfile against the registry and exits non-zero on a vulnerability. An unaudited install is an unreviewed code change.
4. Every new dependency is justified in the PR that adds it: what it does, why a standard-library, Bun-native, or in-house alternative will not, who maintains it.
5. This is the Bun face of the practices in [security.md](../security.md).

### BUN-5. Named lifecycles.

**Step-by-step reasoning:**
1. Startup and shutdown are explicit and ordered, never implicit in import side effects. A named bootstrap acquires resources in sequence: config, then clients, then the server that depends on them.
2. Shutdown is the reverse, triggered by `SIGTERM`: stop accepting connections, drain in-flight work, close pools and consumers, then exit.
3. Shutdown is deadline-bounded. If draining exceeds a hard timeout, force exit rather than hang forever and let the orchestrator kill you.
4. Every resource (pool, consumer, timer, server) binds its open and its close to these named phases. A resource with no owning phase is a leak waiting to happen.
5. Verify it: send `SIGTERM` to the running process and confirm clean drain logs. This extends [core 13](../typescript/13-resource-management.md), resource management, to the process scope.

### BUN-6. Typecheck is a separate gate.

**Step-by-step reasoning:**
1. Bun executes TypeScript by stripping types and running the result — *"Bun does not perform typechecking."* A file with a type error runs anyway, right up until the moment the lie reaches runtime.
2. So type safety is not a side effect of running the code; it is a step you must add. `tsc --noEmit` is a non-negotiable CI gate that fails the build on any type error, exactly because the runtime will not.
3. The family's `erasableSyntaxOnly` stance is what makes type-stripping safe: no `enum`, no constructor parameter properties, nothing that needs runtime emit. Type-space and value-space stay cleanly separable, so stripping is lossless for behaviour.
4. This is a *gate*, not a runtime behaviour — it lives in CI and pre-commit, alongside `gts lint`, never in the hot path. The ESM and `node:`-prefix module mechanics that the Node guide once called out separately fold into [chapter 01](./01-runtime-and-toolchain.md); Bun makes module-system mixing a non-issue.
5. This extends [core 3](../typescript/03-the-type-system.md), the type system as the first test suite — but on Bun the suite only runs if you wire it up.

---

Zero technical debt holds here as everywhere: what ships meets the design goals. **Perfection over technical debt — debt never gets paid.** A Bun service runs unattended for months; the shortcut taken today is the page at 3am later.

## Influences

- **[Bun documentation](https://bun.com/docs)** — canonical runtime reference: the runtime, bundler, package manager, test runner, and JavaScriptCore execution model.
- **Node.js best-practices tradition** — community canon for production server operation, carried where Bun's `node:` compatibility makes it apply.
- **[zod](https://zod.dev/)** — boundary parsing and `z.infer`-derived types.
- **[pino](https://getpino.io/)** — structured, low-overhead logging.
- **TigerBeetle Tiger Style** — limits on everything, crash on unknown state, zero technical debt.
