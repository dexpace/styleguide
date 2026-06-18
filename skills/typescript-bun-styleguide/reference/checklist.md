# typescript-bun checklist

Bun runtime additions on top of `typescript-styleguide`. Additive only — never weaker than a core rule. One heading per chapter.

### 01 — Runtime and Toolchain

- Pin Bun exactly in a committed `.bun-version`; CI provisions that exact version from a pinned image; upgrades are reviewed, changelog-read diffs.
- `bun install` writes a committed `bun.lock`; `bun install --frozen-lockfile` in CI and image builds; a drifted or edited lockfile fails the build.
- Prefix every builtin import with `node:` (`node:fs`, `node:crypto`); a bare specifier can be shadowed by a package.
- Use `#imports` subpaths in `package.json` for cross-area references; reserve `#` keys for genuine boundaries, plain relative imports stay local.
- Run TS natively; `tsc --noEmit` is the type gate, not a build step — Bun strips types and never checks them. Entry points are `.ts` run by Bun, no `tsc` emit.
- Ship on a pinned stable release; no `--canary` or untagged Bun on any production path.
- ESM only (`"type": "module"`); quarantine `require`/`createRequire` to one `// bridge:` module per CJS dependency.
- One process per container; scale by adding replicas, not forking workers; this layer owns `moduleResolution: "bundler"`.

### 02 — Concurrency and the Event Loop

- No `*Sync` API on a request path (`readFileSync`, `crypto.*Sync`, `execSync`); `*Sync` is for startup only, before `Bun.serve()`.
- Event-loop lag is a first-class metric: sample with an interval-drift sampler (Bun has no verified `monitorEventLoopDelay`), export p99, alert on the budget (default p99 < 50ms).
- CPU-bound work goes to bounded Bun `Worker`s sized from `availableParallelism()`, with the queue bounded by `invariant`.
- Honor backpressure end to end: `stream.pipeline`/async iterators, never a bare `.pipe()` without error and teardown wiring.
- `unhandledRejection`/`uncaughtException` log with context, flush, `process.exit(1)` — never a `return` that resumes serving (crash-only).
- Graceful shutdown is ordered and deadline-bounded: stop accepting, drain in-flight, close pools last; register with `process.once`; force exit on overrun under the orchestrator grace.
- Long-lived loops yield per batch (`await setImmediate()` / `Bun.sleep(0)`) and check `signal.throwIfAborted()`.
- In-flight tracking is bounded by a named ceiling; shed with `503` + `Retry-After` over an unbounded accept queue.
- Timers and intervals tie to the shutdown `AbortController` and clear on abort; `unref()` is a liveness hint, not cleanup.

### 03 — HTTP Services

- Hono on `Bun.serve` is the framework; route I/O is inferred from the validator via `c.req.valid`, not hand-typed.
- Elysia only for Bun-only experiments; Express/Fastify are legacy Node estates (tag `node-guide-final`); NestJS is rejected (DI/decorator magic, breaks `erasableSyntaxOnly`).
- Every route declares request AND response schemas; the outbound `.parse()` is a leak tripwire.
- Handlers are thin: parse already-validated input, call one plain domain function, return; no Hono types in domain code. Test via `app.request()`, not a live socket or MSW.
- One centralized `app.onError(mapError)` maps domain errors to `problem+json`; handlers never craft a 5xx; unknown errors map to a generic 500 carrying only the correlation id.
- Every request carries a `correlationId`, minted or accepted once at the edge, propagated through `AsyncLocalStorage` via `store.run`.
- Set server timeouts explicitly: `idleTimeout` on the `Bun.serve` config and a per-route `timeout(...)`, kept under the upstream LB idle timeout.
- Rate-limit at the edge with bounded store state (LRU max size or Redis TTL), `429` + `Retry-After`.
- Health endpoints are honest: liveness does no dependency I/O (restart signal); readiness checks dependencies and fails first during drain (traffic signal).

### 04 — Persistence

- `Bun.SQL` + Drizzle is the Postgres layer; raw `Bun.SQL` tagged templates for measured hot paths; `bun:sqlite` off the request path only; Prisma needs written justification (codegen + separate engine).
- Repositories are plain functions taking a `db`/`tx` handle, not classes; no active-record `save()`/`delete()`.
- Transactions are explicit `db.transaction` scopes; `tx` threads down to each write; no ambient/decorator transactions, no global handle inside a scope.
- Migrations are versioned, forward-only, reviewed SQL (`drizzle-kit generate`); no `down` in prod, no `push`/auto-sync; CI runs them on a fresh database.
- `Bun.SQL` pools are explicitly sized and bounded with `connectionTimeout`, `idleTimeout`, `maxLifetime`; opened once at boot, closed on `SIGTERM`, never per-request.
- Every list query carries a `LIMIT`; cursor pagination over `OFFSET`; expose as `AsyncIterable<T>`.
- N+1 is a bug, not a tuning opportunity: batch with `inArray`/joins; the query count of a request is part of review.
- Rows are external input — parse at the repository boundary with zod/`z.infer`; entities never cross inward or reach the API surface.
- Wrap driver errors into typed domain errors at the repository with `cause` intact; no SQLSTATE or SQL text above the boundary.

### 05 — Serialization and Validation

- Parse `Bun.env` once at startup into a frozen `Readonly` config; fail the boot listing every problem (`z.treeifyError`); no `Bun.env`/`process.env` access outside the config module.
- Every request and response body has a zod schema; the type is `z.infer`, never a hand-written twin.
- Raw `JSON.parse` never escapes a boundary module; parse, validate, and type in one move; the intermediate is `unknown`, not `any`.
- Dates are ISO-8601 strings on the wire (`z.iso.datetime()`); `Temporal` in the domain, `date-fns` until then, never `moment`.
- Null versus absent is a per-field contract decision (`exactOptionalPropertyTypes`); choose `.optional()` vs `.nullable()` deliberately.
- Unknown fields: strip by default on reads, `z.strictObject()` on commands/writes; `z.looseObject()` only at a commented proxy boundary.
- Outbound serialization is schema-checked too (dev/test), driven by `app.request()` injection tests; a leaked field fails the parse.
- BigInt, binary, and money get explicit wire encodings: money as integer minor units or decimal string into `Cents`, i64 as a string, binary as base64 — never a JSON `number`.

### 06 — Logging

- pino on Bun, structured JSON; logs are data first; constant message strings, variable data in the object; no `console` wrapper, no second logger.
- No in-process `transport` in production (Bun `worker_threads` gaps); base pino writes NDJSON to stdout, the collector ships and formats it.
- `AsyncLocalStorage` carries request context (the MDC analog); bind with `store.run` at the boundary, read through a `log()` accessor; never `enterWith` mid-handler.
- `redact` PII and secret paths at the base logger (authorization, cookie, set-cookie, x-api-key, `*.password`, `*.token`, ...); masking is config, not call-site discipline.
- Log once, at the boundary; inner layers enrich via `cause` and stay silent; pass the whole error object, not `e.message`.
- Levels mean something: `error` pages, `warn` is degraded-but-serving, `info` marks state transitions, `debug` off in prod; expected outcomes are not `error`.
- `console.*` is banned in server code (`no-console` lint on top of core); diagnostics go through `log()`.
- Child loggers per component (`child({ component })`) for grep-able provenance; stable low-cardinality fields on the child, per-event data on the call.
- Log lines are bounded: log handles and counts, not payloads; truncate large fields, log arrays by length or a bounded sample.

### 07 — Bun Performance

- Read core chapter 15 first; carry allocation hygiene, batched awaits, measure-first, and the optimization ledger; drop V8-specific shape folklore — JSC is not V8, so measure on JSC.
- Event-loop lag is the GC-pause analog: measure it, budget it, alert on it (interval-drift sampler).
- Profile with JSC's own tools first and name the tool to the question: `bun:jsc` `startSamplingProfiler()` for CPU, `generateHeapSnapshot()`/`heapStats()` for memory, mitata for micro-comparisons (state its warm-JIT caveats), `Bun.nanoseconds()` for spans.
- Bound and deadline every outbound `fetch`: keep Bun's default keep-alive, add `AbortSignal.timeout(ms)` and a call-site concurrency bound; the global 256 cap is not a strategy; `keepalive: false` needs a recorded reason.
- Stream large payloads with backpressure (`Bun.file(path).stream()`, `pipeThrough`/`pipeTo`); never buffer an unbounded payload fully into memory.
- Schema-derived JSON serialization on hot routes only where a profile measures it; default to plain `JSON`; on Hono shape the output to the response schema.
- Load-test before shipping a performance claim: `oha` (or `bombardier`), a stated scenario, p50/p99 + RSS in the PR; never the average, never a Node-based tool against Bun.
- Watch RSS against the JS heap (native-side leaks); no `--max-old-space-size` (V8 knob); use `--smol` for constrained containers as a recorded trade-off; cap the container above measured peak RSS.
- The optimization ledger applies: every Bun-specific tuning constant carries the measurement that earned it, written at the site.

### 08 — Build and Distribution

- Libraries build with plain `tsc`: one `.js` per module, `.d.ts` declarations, sourcemaps — no bundler between source and consumers.
- Services bundle with `Bun.build` into one artifact (`target: "bun"`), or `bun build --compile` for a single-file executable where Bun is not installed; the build script is boring and checked in.
- The `exports` field is locked: explicit entry points only; deep imports are resolution errors; `types` first, then `import`.
- `publint` + `arethetypeswrong` (`attw --pack`) gate publishing in CI; a failure blocks the release, no override.
- `api-extractor` produces a committed `*.api.md` report on libraries; CI fails on undocumented surface change; the diff is reviewed like code.
- Publish with provenance (`npm publish --provenance` from CI until `bun publish` supports it) and 2FA; commit `bun.lock` always.
- Dependency policy: minimal, pinned, audited (`bun audit` as a CI gate); every new dependency justified in the PR (size, maintenance, why no `node:`/`Bun.*`/owned alternative).
- Version with changesets: every consumer-facing bump traces to a changelog entry; breaking is MAJOR with no exceptions.
- Reproducible builds: same inputs, same artifact; only `bun install --frozen-lockfile` touches the network; the Bun version is pinned via committed `.bun-version`.
