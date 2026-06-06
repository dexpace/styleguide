## TypeScript on Node.js

Binding style rules for **server-side TypeScript on Node.js**. This guide *extends* [typescript/](../typescript/) for the Node runtime. It is additive: where this guide is stricter, it wins for Node code; the core guide remains the canonical language baseline.

The core guide is platform-agnostic. Node has its own contract — a single-threaded event loop, a process supervised by something that can restart it, an ecosystem of untyped dependencies, untrusted I/O at every edge. Those impose rules the language guide cannot address.

### Authorities

This guide extends and defers to an ordered chain. Where they conflict, the higher authority wins:

1. **[Node.js official documentation](https://nodejs.org/docs/latest/api/)** — canonical. The API reference, the event-loop and stream semantics, the lifecycle and signal contracts. Where our guidance collides with documented runtime behaviour, the runtime wins.
2. **[goldbergyoni/nodebestpractices](https://github.com/goldbergyoni/nodebestpractices)** — community canon. It fills the gaps the official docs leave open on production operation: error policy, security, project structure. Defer to it only where the official docs are silent.
3. **This guide's overlay** — the dexpace Node layer: crash-only error policy, event-loop budgets, parse-every-boundary, dependency discipline, named lifecycles, ESM-native modules.

The core [typescript/](../typescript/) authority chain (Google → ts.dev → gts → its overlay) still governs the *language* layer. This guide adds the runtime layer on top; it does not replace it.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Runtime & Modules](./01-runtime-and-modules.md) | ESM-only on Node, `node:` builtins, subpath `#imports`, CJS bridges, engines pin, one process per container |
| 02 | [Concurrency & the Event Loop](./02-concurrency-and-event-loop.md) | No blocking in request paths, no `*Sync`, event-loop lag metrics, `worker_threads` for CPU, streams with backpressure |
| 03 | [HTTP Services](./03-http-services.md) | Fastify (schema-first), thin handlers, centralized error mapping, correlation IDs, timeouts, honest health checks |
| 04 | [Persistence](./04-persistence.md) | Pooled clients, bounded pools, parameterized queries only, transactions as units, rows parsed at the edge, migrations |
| 05 | [Serialization & Validation](./05-serialization-and-validation.md) | zod at every boundary, types from `z.infer`, never trust `JSON.parse`, null-vs-absent semantics, no entity leakage |
| 06 | [Logging](./06-logging.md) | Structured JSON (pino), one logger, correlation via `AsyncLocalStorage`, PII redaction, levels, no `console.log` |
| 07 | [Node Performance](./07-node-performance.md) | `--cpu-prof`/`--heap-prof`/clinic, allocation hygiene, stream over buffer, connection reuse, slowest-resource-first, measure before tuning |
| 08 | [Build & Distribution](./08-build-and-distribution.md) | `tsc` for libraries / `esbuild` for services, locked `exports`, publint + attw gate, api-extractor surface diff, npm provenance, changesets, reproducible builds |

---

## The NODE rules

These add to [the 12 rules in the core guide](../typescript/README.md#the-12-rules-in-typescript). Each extends one core rule to the runtime's edges. When a NODE rule conflicts with a core one, the NODE rule wins for Node code.

### NODE-1. Crash-only error policy.

**Step-by-step reasoning:**
1. An `unhandledRejection` or `uncaughtException` means the process reached a state you did not design. You cannot reason about it, so you cannot safely continue from it.
2. Continuing is the dangerous option, not the safe one. A process limping on with a half-applied write or a corrupt in-memory cache produces silent wrong answers, which are worse than downtime.
3. The handler does exactly three things: log the error with its full `cause` chain, flush the logger, exit with code 1. No recovery logic, no "try once more".
4. A supervisor (systemd, Kubernetes, pm2) owns liveness and restarts a clean process. Restartability is a design requirement, not an afterthought.
5. This extends [core 8](../typescript/08-error-handling.md): operational errors are still handled in place where they occur. Only programmer errors that escape all the way to the process boundary trigger the crash.

### NODE-2. The event loop has a budget.

**Step-by-step reasoning:**
1. One thread serves every request. A synchronous operation that takes 200ms does not slow one client by 200ms; it freezes every concurrent client for 200ms.
2. So the loop is a bounded, shared resource and is treated like one. No `fs.*Sync`, `child_process.execSync`, or synchronous crypto on any request path.
3. CPU-bound work (hashing, compression, large parses) moves to `worker_threads` or out of process entirely. The request thread stays free to dispatch.
4. The budget is measured, not assumed. Event-loop lag is sampled, exported as a metric, and alerted before it surfaces as latency to clients.
5. This extends [core 15](../typescript/15-performance.md), performance from the outset, applied to the runtime's defining constraint.

### NODE-3. Every boundary is parsed.

**Step-by-step reasoning:**
1. Anything crossing from outside into the domain is untyped until proven otherwise: environment variables, HTTP requests, queue messages, third-party responses, database rows.
2. A raw `JSON.parse` result is `any` wearing a disguise. Trusting its shape is the same mistake as `any`, just relocated to runtime.
3. Each boundary value passes through a zod schema *first*. Invalid input is rejected at the edge, before any domain code touches it.
4. The domain type is derived from the schema with `z.infer`, never hand-written alongside it. One source of truth means the validator and the type cannot drift apart.
5. This extends [core 10.7](../typescript/10-api-design.md), parse at boundaries, from the API surface to every edge the runtime touches, including the process environment itself.

### NODE-4. Dependencies are attack surface.

**Step-by-step reasoning:**
1. Every dependency is code you did not write, running with your process's full privileges. The transitive tree is the real surface, not the top-level list.
2. Keep the set minimal. Prefer the `node:` standard library or a few lines of owned code over pulling a package and its subtree.
3. Pin versions with a committed lockfile and audit the tree (`pnpm audit`) in CI. An unaudited install is an unreviewed code change.
4. Every new dependency is justified in the PR that adds it: what it does, why a standard-library or in-house alternative will not, who maintains it.
5. This is the Node face of the practices in [security.md](../security.md).

### NODE-5. Named lifecycles.

**Step-by-step reasoning:**
1. Startup and shutdown are explicit and ordered, never implicit in import side effects. A named bootstrap acquires resources in sequence: config, then clients, then the server that depends on them.
2. Shutdown is the reverse, triggered by `SIGTERM`: stop accepting connections, drain in-flight work, close pools and consumers, then exit.
3. Shutdown is deadline-bounded. If draining exceeds a hard timeout, force exit rather than hang forever and let the orchestrator kill you.
4. Every resource (pool, consumer, timer, server) binds its open and its close to these named phases. A resource with no owning phase is a leak waiting to happen.
5. Verify it: send `SIGTERM` to the running process and confirm clean drain logs. This extends [core 13](../typescript/13-resource-management.md), resource management, to the process scope.

### NODE-6. ESM-native.

**Step-by-step reasoning:**
1. One module system. This is an ESM codebase: `import`/`export` only, no mixing `module.exports` into application code.
2. Builtins are imported with the `node:` prefix, which is unambiguous and immune to package shadowing. See [chapter 01](./01-runtime-and-modules.md) for the mechanics.
3. Internal aliases use subpath `#imports`, not deep relative chains or path-mapping hacks. See [chapter 01](./01-runtime-and-modules.md) for the mechanics.
4. CommonJS is quarantined to declared, documented bridge modules at the edges where a CJS-only dependency forces it. Those bridges are the only place `require`/`createRequire` appears.
5. This extends [core 12](../typescript/12-module-organization.md), module organization, into the runtime's loader.

---

Zero technical debt holds here as everywhere: what ships meets the design goals. **Perfection over technical debt — debt never gets paid.** A Node service runs unattended for months; the shortcut taken today is the page at 3am later.

## Influences

- **[Node.js documentation](https://nodejs.org/docs/latest/api/)** — canonical runtime reference: event loop, streams, modules, process lifecycle.
- **[Node.js Best Practices (goldbergyoni)](https://github.com/goldbergyoni/nodebestpractices)** — community canon for production Node operation.
- **[zod](https://zod.dev/)** — boundary parsing and `z.infer`-derived types.
- **[pino](https://getpino.io/)** — structured, low-overhead logging.
- **TigerBeetle Tiger Style** — limits on everything, crash on unknown state, zero technical debt.
