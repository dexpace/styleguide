# Bun-First Migration — typescript-bun Replaces typescript-node

**Date:** 2026-06-06
**Status:** Approved in design review (pending spec review)
**Owner:** Omar. Direction set by owner: "we almost never want to use npm, only bun — if the
node docs are not needed anymore, delete them or move them to bun and update them."
TS/runtime technical decisions delegated, reasoning recorded, individually reversible.

## Goal

Make Bun dexpace's default server runtime and toolchain across the TypeScript family:

1. **Transform** `typescript-node/` → `typescript-bun/` — the server doctrine *moves* and is
   updated for Bun; nothing is duplicated because the source guide is retired in the same
   change.
2. **Flip core defaults** in `typescript/` where they encode the npm/Node workflow
   (package manager, test runner, runtime forward-refs).
3. **React keeps Vitest + MSW** for component tests as a recorded substitution (bun test
   cannot run MSW and its DOM story is immature).
4. Family becomes: `typescript/` ← `typescript-bun/` (server) + `typescript-react/` (UI).

The retired Node guide stays retrievable: tag the pre-migration commit
**`node-guide-final`** and note it in the bun README ("Node-only projects are the
exception; the retired guide lives at this tag").

## Verified runtime facts (2026-06-06, primary sources)

- Bun v1.3.x; production-capable since 1.3 (Oct 2025); **no LTS policy** → exact version
  pinning (`.bun-version` + CI image), upgrades deliberate.
- Bun executes TypeScript natively by stripping types — it **never type-checks**; the
  family's `erasableSyntaxOnly` stance is what makes that safe, and `tsc --noEmit`
  becomes a non-negotiable CI gate.
- 90%+ Node compatibility (`node:` builtins largely work); native: `Bun.SQL` (Postgres),
  `bun:sqlite`, Redis, S3, `Bun.serve`, `Bun.build`, single-file executables.
- Engine is **JavaScriptCore, not V8**: core 15's allocation hygiene / batching /
  measure-first carry; hidden-class/V8 specifics do not.
- `bun test`: fast, Jest-flavored; lcov/text coverage; **no MSW**, **no Stryker runner**;
  fast-check and `expect-type` are runner-agnostic and keep working.
- Hono (mid-2026): ~38M weekly downloads, 322 contributors, multi-runtime, fast cadence;
  Elysia Bun-only at ~461K weekly. Framework throughput is not the practical bottleneck.

## Decisions (reasoned, owner-reversible)

1. **Transform, don't author from scratch.** Each node chapter is converted in place
   (rename directory, rewrite runtime-specific rules, keep the doctrine that is
   runtime-agnostic). Per-chapter conversion map below. This honors "Applying Style
   Changes": the whole package migrates in one atomic change — no half-Node state.
2. **Core flips (typescript/):**
   - 01: pnpm/corepack → **`bun install`** + committed `bun.lock` (same strict,
     reproducible intent — recorded in the deviations/toolchain prose); pre-commit runs
     `bun run` equivalents; the `module: nodenext` **platform override moves out of core**
     into the runtime guides (bun: `moduleResolution: "bundler"`; react tooling already
     bundler-resolved) — core keeps exactly the six strictness flags and says platform
     settings live with the runtime.
   - 11: Vitest → **`bun test`** as the default runner (explicit imports from `bun:test`);
     fast-check + `expect-type` mandates unchanged; MSW reference scoped to the React
     guide; **mutation testing recorded as an accepted gap** (no Stryker runner for bun;
     revisit) instead of a nightly recommendation.
   - Forward-refs: 15.8 and 10.2 point at `typescript-bun/07` / `typescript-bun/08`.
   - gts survives untouched — lint/format are runtime-independent.
3. **BUN-1…6** (replacing NODE-1…6; same spine, one swap): crash-only · event-loop budget ·
   every boundary parsed · dependencies are attack surface (`bun audit`) · named
   lifecycles · **typecheck is a separate gate** (replaces "ESM-native", which Bun makes a
   non-issue; ESM/`node:`-prefix rules live on in ch. 01).
4. **HTTP: Hono on `Bun.serve`** — already blessed in node/03, ~80× Elysia's adoption,
   multi-runtime (BUN-4 reasoning applies doubly to frameworks); zod validator middleware
   keeps parse-don't-validate; **`app.request()` injection testing** replaces MSW
   server-side. Elysia noted acceptable for Bun-only experiments; NestJS rejection carries
   over verbatim reasoning (erasable-syntax + root rule 2).
5. **Persistence: `Bun.SQL` + Drizzle** (tagged-template queries are parameterized by
   construction); `bun:sqlite` for embedded. Repos/transactions/migrations/N+1/row-parsing
   doctrine carries through the transform.
6. **Logging: pino retained** on Bun (≥1.3 compatible; family consistency), with a
   stdout-JSON fallback note if transports misbehave; `correlationId` canonical key and
   ALS context carry unchanged.
7. **Publishing: libraries still ship to the npm registry** ("only bun" is the workflow,
   not the registry) via `bun publish`; publint + attw + api-extractor + provenance gates
   carry; author verifies `bun publish` provenance support and falls back to
   `npm publish --provenance` in CI if absent (recorded either way).
8. **React substitution (typescript-react/):** component tests keep Vitest + Testing
   Library + MSW (bun test lacks both); recorded in the react README ledger as a runtime
   substitution mirroring the bun ledger mechanism. Playwright unchanged.

## Per-chapter conversion map (typescript-node/NN → typescript-bun/NN)

| File | Keep (doctrine) | Rewrite (runtime) |
|---|---|---|
| README | extension semantics, authority-chain shape, zero-debt close | Bun docs canonical; BUN-1…6; **Runtime Substitutions ledger** (bun install/bun test/Bun.build/JSC rows + react's Vitest counter-substitution pointer); when-Bun paragraph (no LTS → pin exactly); `node-guide-final` tag pointer |
| 01 runtime-and-modules → runtime-and-toolchain | `node:` prefix, no-experimental-flags, one-process-per-container, ESM rules | LTS floor → exact pin (`.bun-version`); tsc-compile-for-prod → **native TS execution + `tsc --noEmit` gate**; pnpm/corepack → `bun install`/`bun.lock`; add `bunfig.toml` minimal + `moduleResolution: "bundler"` platform note |
| 02 concurrency-and-event-loop | sync-ban, crash-only, shutdown contract, yield, bounded in-flight, timer lifecycles | piscina → **Bun Workers** (bounded hand-rolled pool); `monitorEventLoopDelay` → verify Bun support, else Bun-native lag measure (author verifies, hedges honestly); Web Streams-native backpressure |
| 03 http-services | schemas-mandatory, thin handlers, central error map, correlation contract, timeouts, rate limits, honest health | Fastify → **Hono on `Bun.serve`** (new exemplar); type-provider wiring → zod validator middleware; add `app.request()` testing rule; `Bun.serve` option surface |
| 04 persistence | repos-as-functions, explicit tx, forward-only migrations, pool bounds, N+1, row parsing, error wrapping | pg Pool → **`Bun.SQL`** (+ pool options); add `bun:sqlite`; Drizzle-on-Bun wiring |
| 05 serialization-and-validation | all of it (zod, dates, null-vs-absent, unknown fields, outbound, money/i64/binary) | `process.env` → `Bun.env` (same parse-once pattern); Bun-native file/S3 binary boundaries |
| 06 logging | pino, redact, ALS/`correlationId`, log-once, levels, no-console, child loggers, bounds | pino-on-Bun compatibility note + stdout-JSON fallback; transport caveat |
| 07 node-performance → bun-performance | lag-as-first-metric, tool-per-question, streams, schema-derived serialization economics, load-test-before-claims, memory headroom, ledger | undici pools → Bun fetch pooling; `--cpu-prof` → **`bun:jsc` profiling** + `Bun.nanoseconds()`; mitata micro-benchmarks; JSC-vs-V8 carry/don't-carry table (cross-ref core 15) |
| 08 build-and-distribution | exports-locked, publint/attw, api-extractor, provenance/2FA, dep policy, changesets, reproducibility | esbuild → **`Bun.build`**; add single-file executables (`bun build --compile`) trade-offs; `pnpm audit` → `bun audit`; lockfile = `bun.lock`; `bun publish` (with CI provenance fallback note) |

Format contract unchanged (exemplar-first, Reasoning, Enforcement, Cross-references,
gates); the transform must leave every gate green and every cross-reference (core ↔ bun ↔
react) resolving — including updating **all** family references to `typescript-node/`
(core 10.2/15.8, react ledger, root README row, CLAUDE.md).

## Repo integration & process

- Tag `node-guide-final` on the pre-migration commit, then one atomic migration commit:
  `feat: bun-first runtime — typescript-bun replaces typescript-node` (directory rename +
  rewrites + core flips + react ledger note + root README/CLAUDE.md updates), preceded by
  this spec + its plan as docs commits. No co-author trailers; sole author Omar.
- Same execution pipeline as the family: plan → parallel transforms (disjoint files,
  controller owns git) → per-file spec+quality review → family sweep (links, coherence,
  substitution-ledger accuracy) → merge to main.

## Out of scope

- typescript-deno/; converting react component tests to bun test; Bun macros; WebSocket
  doctrine (revisit on first real use); deleting the spec/plan history of the node guide.

## Open questions

None blocking. Author-verification items (Bun `monitorEventLoopDelay` support, `bun publish`
provenance) are flagged in the conversion map with honest fallbacks either way.
