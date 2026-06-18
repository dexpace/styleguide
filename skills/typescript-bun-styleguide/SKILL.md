---
name: typescript-bun-styleguide
description: Use when writing, editing, or reviewing TypeScript that runs on Bun (Bun.serve, Hono, Bun.SQL, bun test) in a dexpace project — extends typescript-styleguide with Bun-specific rules (runtime pinning, the tsc typecheck gate, HTTP services, persistence). Use alongside typescript-styleguide, not instead of it.
---

# TypeScript-on-Bun styleguide

## When this applies

Editing TypeScript that runs on Bun, or reviewing it. Triggers: `bun.lock`, `bunfig.toml`, `.bun-version`, `Bun.serve`, `bun:sqlite`, `Bun.SQL`, `Bun.env`, `Bun.build`, or Hono imports. Priority: **correctness > performance > developer experience.**

## Inherited

First apply `typescript-styleguide`; this layer adds, and where stricter overrides, for TypeScript on Bun.

## Language hard rules

These add to the core rules — they never restate or weaken them. Where a Bun rule conflicts with a core one, the Bun rule wins for Bun code.

- **Bun never type-checks.** It runs TypeScript by stripping types, so a type error runs anyway until it reaches runtime. `tsc --noEmit` is a non-negotiable CI and pre-commit gate, separate from execution. The core `erasableSyntaxOnly` stance (no `enum`, no parameter properties) is what makes strip-and-run lossless — keep it.
- Pin Bun exactly in a committed `.bun-version`; CI provisions that exact version from a pinned image. No range, no canary, no untagged build on a production path; upgrades are reviewed, changelog-read diffs.
- `bun install` with a committed `bun.lock`; `bun install --frozen-lockfile` in CI and image builds. ESM only; quarantine any CJS interop to a `// bridge:` module. Prefix every builtin import with `node:`. Use `#imports` subpaths for cross-area references. This layer owns `moduleResolution: "bundler"`.
- One process per container; scale by adding replicas, not forking workers.
- **Crash-only:** `unhandledRejection`/`uncaughtException` log with the full `cause` chain, flush, `process.exit(1)` — never resume serving. The supervisor restarts a clean process.
- The event loop has a budget. No `*Sync` on a request path; CPU work goes to bounded Bun `Worker`s; long loops `await setImmediate()`/`Bun.sleep(0)` per batch; sample event-loop lag and alert on it. Honor backpressure with `stream.pipeline`/async iterators, never a bare `.pipe()`.
- Graceful shutdown is ordered and deadline-bounded: stop accepting, drain in-flight, close pools last; register with `process.once`; force exit on overrun.
- HTTP: Hono on `Bun.serve`, thin handlers over plain domain functions, a zod validator on every request and response, one centralized `onError` map, a `correlationId` on every request, explicit `idleTimeout` and per-route timeouts, bounded rate-limit state, honest liveness/readiness endpoints. Test routes via `app.request()` injection, never a live socket.
- Persistence: `Bun.SQL` + Drizzle for Postgres, `bun:sqlite` off the request path only. Repositories are plain functions over a `db`/`tx` handle. Transactions are explicit scopes that thread `tx`. Bound every list query and the pool; parse rows at the boundary; wrap driver errors into typed domain errors; N+1 is a bug.
- Parse every boundary with zod, types from `z.infer`. Parse `Bun.env` once at startup into a frozen config and fail the boot listing every problem. Never let raw `JSON.parse` escape a boundary. Money as integer minor units or a decimal string, i64 as a string, binary as base64 — never a JSON `number`.
- Logging: structured JSON via pino, no in-process transport; carry context through `AsyncLocalStorage`; `redact` PII at the logger; log once at the boundary; `console.*` is banned in server code.
- Build: libraries with plain `tsc` (declarations + sourcemaps, no bundler); services with `Bun.build` or `bun build --compile`. Lock `exports`; gate publish with `publint` + `attw`; `api-extractor` diffs the public surface; publish with provenance and a committed lockfile; reproducible builds, no network beyond the lockfile.

Examples use zod v4 top-level forms only: `z.email()`, `z.uuid()`, `z.url()`, never the chained `z.string()` equivalents.

## Before you finish — verify

```bash
bun install --frozen-lockfile   # exact bun.lock tree; drift fails the build
tsc --noEmit                    # the type gate — Bun never type-checks
bun test                        # server tests; route wiring via app.request()
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/typescript-bun/README.md)
- [01 Runtime & Toolchain](https://github.com/dexpace/styleguide/blob/main/typescript-bun/01-runtime-and-toolchain.md)
- [02 Concurrency & the Event Loop](https://github.com/dexpace/styleguide/blob/main/typescript-bun/02-concurrency-and-event-loop.md)
- [03 HTTP Services](https://github.com/dexpace/styleguide/blob/main/typescript-bun/03-http-services.md)
- [04 Persistence](https://github.com/dexpace/styleguide/blob/main/typescript-bun/04-persistence.md)
- [05 Serialization & Validation](https://github.com/dexpace/styleguide/blob/main/typescript-bun/05-serialization-and-validation.md)
- [06 Logging](https://github.com/dexpace/styleguide/blob/main/typescript-bun/06-logging.md)
- [07 Bun Performance](https://github.com/dexpace/styleguide/blob/main/typescript-bun/07-bun-performance.md)
- [08 Build & Distribution](https://github.com/dexpace/styleguide/blob/main/typescript-bun/08-build-and-distribution.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
