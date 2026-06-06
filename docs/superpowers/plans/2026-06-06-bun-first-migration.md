# Bun-First Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `typescript-node/` with `typescript-bun/` (in-place transformation), flip the core guide's toolchain defaults to Bun, and record React's Vitest counter-substitution — one atomic migration landing on main.

**Architecture:** Directory rename first (preserves git rename detection), then per-file rewrites that keep runtime-agnostic doctrine and replace only what Bun changes; core/react/index touchpoints updated in the same change; dev commits reshaped into docs + one `feat:` migration commit at publish. Spec: `docs/superpowers/specs/2026-06-06-typescript-bun-design.md` (the **Spec** — its conversion map is authoritative for keep/rewrite splits).

**Tech Stack:** Markdown only. Verification: the family gates (`grep`/`wc`) + zero `typescript-node` references outside docs history + zero pnpm/Vitest references outside React's recorded substitution. Branch: `feat/bun-first` (spec already on it; tag `node-guide-final` = pre-migration main).

---

## Ground rules (every transform task)

- Source file already lives at its **typescript-bun/** path (Task 0 renames the directory).
- KEEP-rules are preserved verbatim except: cross-reference paths (`typescript-node/` → `typescript-bun/`), "Node" runtime wording where Bun now applies, and renumber-nothing (rule counts and numbers stay stable unless the task says otherwise).
- REWRITE-rules replace content but keep the slot (same `### N.M` number, new imperative title allowed).
- Format contract unchanged: exemplar-first, **Reasoning, step by step:**, **Enforcement:**, Cross-references, blank line after each `###`, balanced fences.
- Authors verify Bun claims against bun.com/docs (WebFetch) before asserting; hedge honestly where the platform is moving (the Spec flags `monitorEventLoopDelay` support and `bun publish` provenance).
- **No git commands in transform agents** — controller owns git (per [[feedback-parallel-subagents]] protocol).
- Verify per file: `grep -c '^### '` = stated count · exemplar = 1 · Reasoning = Enforcement = count · no `typescript-node/` link remains in the file · no `pnpm`/`Vitest`/`fastify`/`undici`/`piscina`/`esbuild` token unless the task says it stays (e.g. node-compat notes).

### Task 0: Directory rename (controller)

- [ ] `git mv typescript-node typescript-bun && git commit -m "refactor: rename typescript-node to typescript-bun (transform pending)"`
- [ ] Verify: `ls typescript-bun/*.md | wc -l` → 9; `ls typescript-node 2>&1` → no such file.

### Task 1: typescript-bun/README.md

**Counts after:** BUN- rules = 6; substitutions table ≥ 4 rows; 'debt never gets paid' ×1; 8-row index (titles updated: 01 → "Runtime & Toolchain", 07 → "Bun Performance").

- [ ] KEEP: charter shape (extends `../typescript/`, additive, stricter-wins, core chain governs language layer); chapter-index structure; numbered-reasoning rule format; zero-debt closing; Influences shape.
- [ ] REWRITE: authorities → Bun official docs (https://bun.com/docs) canonical for the runtime; community best practices secondary; core chain unchanged. NODE-1…6 → **BUN-1…6**: 1 crash-only (keep reasoning, Bun processes supervised identically) · 2 event-loop budget (keep) · 3 every boundary parsed (keep) · 4 dependencies are attack surface — `bun audit`, committed `bun.lock` · 5 named lifecycles (keep) · 6 **typecheck is a separate gate** — Bun strips types and never checks them; `tsc --noEmit` gates CI (new; ESM-native content folds into ch. 01).
- [ ] ADD: **Runtime Substitutions ledger** (table: core/family rule → Bun substitution → preserved intent): pnpm+corepack (ts 1.5) → `bun install`+`bun.lock` · Vitest (ts 11.1) → `bun test` · esbuild services (bun 8.2) → `Bun.build` · V8 guidance (ts 15.2–15.3) → JSC profile (ch. 07) · pointer to React's counter-substitution (react keeps Vitest+MSW for component tests). ADD when-Bun paragraph: no LTS policy — pin exactly, upgrade deliberately; 90%+ node-compat; the retired Node guide lives at tag `node-guide-final`.
- [ ] Verify + report numbers.

### Task 2: typescript-bun/01-runtime-and-modules.md → retitle `# 01 — Runtime and Toolchain`

**Counts after:** 8 rules.

- [ ] KEEP (adjust wording only): 1.3 `node:` prefix (Bun implements node builtins; greppability reasoning unchanged) · 1.4 `#imports` subpaths (Bun resolves them) · 1.7 ESM-only + `// bridge:` quarantine (note Bun's transparent CJS interop makes bridges rarer; rule stands) · 1.8 one process per container.
- [ ] REWRITE: 1.1 LTS floor → **pin Bun exactly** (`.bun-version` + CI image; no LTS policy means the floor IS the pin; upgrades are deliberate, changelog-read) · 1.2 corepack/pnpm → **`bun install`** (committed `bun.lock`, `--frozen-lockfile` in CI; strict-install intent preserved) · 1.5 tsc-compile-for-prod → **native TS execution** (Bun runs `.ts` directly by stripping erasable syntax — cite ts/03 §3.12; `tsc --noEmit` is the CI gate, BUN-6) · 1.6 no-experimental-flags → no canary builds in prod; pin stable releases.
- [ ] REWRITE exemplar: `package.json` (scripts: `dev: bun --watch src/main.ts`, `typecheck: tsc --noEmit`, `test: bun test`, `lint: gts lint`), `bunfig.toml` minimal, `.bun-version`, entry module with `node:` import + `#imports`. ADD platform-tsconfig note: `moduleResolution: "bundler"` here (the override that core 01 no longer carries — cite ts/01).
- [ ] Verify + report.

### Task 3: typescript-bun/02-concurrency-and-event-loop.md

**Counts after:** 9 rules.

- [ ] KEEP: 2.1 `*Sync` ban · 2.4 backpressure via `stream.pipeline`/async iterators (note Web Streams are native; both forms fine) · 2.5 crash-only handlers · 2.6 shutdown contract · 2.7 yield in long loops (`setImmediate`/`Bun.sleep(0)`) · 2.8 bounded in-flight + load-shed · 2.9 timers tied to lifecycle signals.
- [ ] REWRITE: 2.2 lag metric — author VERIFIES `monitorEventLoopDelay` under Bun (bun.com/docs node-compat); if supported, keep with a Bun-verified note; if not, substitute an honest interval-drift sampler (~6-line helper) with the same p99 budget · 2.3 piscina → **Bun `Worker`** with a hand-rolled bounded pool (~12-line snippet; sized to `navigator.hardwareConcurrency`/cgroup quota; same maxQueue reasoning).
- [ ] REWRITE exemplar: same shutdown skeleton, `Bun.serve` server handle (`server.stop()` for stop-accepting), crash-only handlers unchanged.
- [ ] Verify + report.

### Task 4: typescript-bun/03-http-services.md

**Counts after:** 9 rules.

- [ ] KEEP: 3.3 request AND response schemas mandatory · 3.4 thin handlers · 3.5 centralized error mapping (Hono `app.onError`) · 3.6 correlation contract (`correlationId` canonical key, ALS — works on Bun) · 3.7 explicit timeouts (rewrite option surface: `Bun.serve` `idleTimeout`, Hono `timeout` middleware; keep stated defaults) · 3.8 rate limiting bounded · 3.9 honest health.
- [ ] REWRITE: 3.1 Fastify → **Hono on `Bun.serve`** — decision ledger: already family-blessed (was the edge option), ~38M weekly downloads vs Elysia's ~461K, multi-runtime, fast cadence (BUN-4 applied to frameworks); zod validator middleware (`@hono/zod-validator`) = parse-don't-validate · 3.2 alternatives — Elysia acceptable for Bun-only experiments (Eden type-safety noted; adoption/maturity reasoning); Express/Fastify = legacy Node estates (tag pointer); **NestJS rejection carries verbatim** (root rule 2 + erasableSyntaxOnly).
- [ ] ADD inside 3.3 or 3.4 (keep count 9): **`app.request()` injection testing** — Hono apps test by direct invocation, no network, no MSW (which `bun test` cannot run anyway — ledger note).
- [ ] REWRITE exemplar: one Hono route — `zValidator('json', Schema)`, typed thin handler, `app.onError(mapError)`, correlation middleware (ALS, same key), served via `Bun.serve({fetch: app.fetch, idleTimeout})`.
- [ ] Verify + report (also: `fastify` token count → 0 outside the legacy-estates line).

### Task 5: typescript-bun/04-persistence.md

**Counts after:** 9 rules.

- [ ] KEEP: 4.2 repos as plain functions · 4.3 explicit transactions · 4.4 forward-only migrations (drizzle-kit) · 4.6 bounded lists/cursors · 4.7 N+1 · 4.8 rows parsed at boundary · 4.9 errors wrapped with cause.
- [ ] REWRITE: 4.1 → **`Bun.SQL` + Drizzle** primary for Postgres (tagged-template queries are parameterized by construction — injection-safe by API shape; Drizzle's bun-sql driver for schema-derived types); **`bun:sqlite`** for embedded (synchronous API — note the 2.1 carve-out: acceptable only off the request path or for CLI/dev tooling); raw `Bun.SQL` acceptable on hot paths; Prisma stance carries · 4.5 pool sizing → `Bun.SQL` option surface (`max`, idle/connection timeouts; author verifies exact option names against bun.com/docs).
- [ ] REWRITE exemplar: Drizzle schema (bun-sql driver) + plain-function repo + one explicit transaction.
- [ ] Verify + report (`pg Pool`/`node-postgres` tokens → 0).

### Task 6: typescript-bun/05-serialization-and-validation.md

**Counts after:** 8 rules.

- [ ] KEEP: 5.2–5.8 wholesale (schemas, JSON.parse confinement, dates, null-vs-absent, unknown fields, outbound checking, money/i64/binary).
- [ ] REWRITE: 5.1 env config → **`Bun.env`** parsed once at startup (same zod/treeify/freeze pattern; note `process.env` also works — `Bun.env` is the idiomatic read). Exemplar: swap the read, keep the shape.
- [ ] Verify + report.

### Task 7: typescript-bun/06-logging.md

**Counts after:** 8 rules.

- [ ] KEEP: 6.2 ALS context (works on Bun; `correlationId` key) · 6.3 redact · 6.4 log-once · 6.5 levels · 6.6 no-console (note: the lint addition now rides the bun overlay) · 6.7 child loggers · 6.8 bounded lines.
- [ ] REWRITE: 6.1 pino → **pino on Bun** (compatible ≥ Bun 1.3; transports caveat: they spawn worker threads — author verifies current status; fallback recorded: base pino JSON to stdout with identical fields, no transports, ship via the platform's log collector).
- [ ] Exemplar: unchanged shape; add the fallback note comment.
- [ ] Verify + report.

### Task 8: typescript-bun/07-node-performance.md → rename file to `07-bun-performance.md`, retitle `# 07 — Bun Performance`

**Counts after:** 8 rules. (Controller renames the file in Task 0? No — agent cannot git-mv. **Controller renames in Task 15 commit prep**; agent edits in place at the old name and reports; controller `git mv` before committing.)

- [ ] KEEP: 7.1 lag-as-first-metric (cross-ref bun 02's verified mechanism) · 7.4 streams for large payloads · 7.5 schema-derived serialization economics (Hono/`Bun.serve` framing; cite ts 15.8) · 7.6 load-test before claims (`oha`/`bombardier` or autocannon-on-node note — author picks one and states it) · 7.8 optimization ledger.
- [ ] REWRITE: 7.2 tool-per-question → **`bun:jsc` profiling** (`profile()`, heap snapshot via `generateHeapSnapshot`), `Bun.nanoseconds()` timing, **mitata** micro-benchmarks (replaces vitest bench; same warm-JIT caveats) · 7.3 undici pools → **Bun fetch pooling** (`Bun.serve`/fetch keep-alive semantics; explicit `maxRequests`-style bounds where the API offers them — author verifies surface; the default-is-not-a-strategy reasoning carries) · 7.7 memory → JSC heap vs RSS; `--smol` flag trade-off; container limits reasoning carries with JSC numbers.
- [ ] ADD to scope para or 7.2: the **JSC ≠ V8 carry table** (carries: allocation hygiene, batching, measure-first, ledger — cites ts/15; does not carry: hidden-class/`delete`/sparse-array V8 specifics — JSC has its own shape optimizations, measure instead of folklore).
- [ ] Verify + report (`undici`/`--cpu-prof`/`speedscope` tokens → 0 unless in a "on Node you would" contrast line).

### Task 9: typescript-bun/08-build-and-distribution.md

**Counts after:** 9 rules.

- [ ] KEEP: 8.1 libraries build with plain tsc (consumers are runtime-agnostic) · 8.3 exports locked · 8.4 publint + attw · 8.5 api-extractor · 8.8 changesets/semver · 8.9 reproducible builds.
- [ ] REWRITE: 8.2 esbuild → **`Bun.build`** for services (one boring checked-in script) + **single-file executables** `bun build --compile` with trade-offs (size, startup, no node_modules at runtime; when to choose) · 8.6 provenance → **`bun publish`**; author VERIFIES provenance support — if absent, the recorded fallback is `npm publish --provenance` in CI only (registry tooling, not workflow regression) · 8.7 `pnpm audit` → **`bun audit`**; lockfile = `bun.lock`.
- [ ] REWRITE exemplar package.json: scripts wire `bun run` equivalents; `publishConfig.provenance` stays; engines → document Bun pin instead.
- [ ] Verify + report.

### Task 10: Core flip — typescript/01-formatting-and-tooling.md

- [ ] REWRITE rule 1.5 (pnpm/corepack) → `bun install` + committed `bun.lock` (strictness intent: isolated installs, fail-on-undeclared via lockfile discipline; note Bun's installer behavior honestly — author verifies hoisting/strictness claims and words them precisely).
- [ ] REWRITE exemplar: `packageManager` field → remove (corepack-specific); add `.bun-version` mention; scripts run via `bun run`; pre-commit = `gts lint && tsc --noEmit` unchanged (runner-agnostic).
- [ ] MOVE the platform-override prose: `module: nodenext` exemplar line and its comment are replaced by: "platform module settings live with the runtime guide (`typescript-bun/01`, `typescript-react/` tooling); core mandates only the six strictness flags." (The `lib`-additions parenthetical from ch. 13 stays — still a platform-settings example.)
- [ ] Verify: rule count 10; `pnpm` tokens 0; `corepack` 0; six flags intact; `nodenext` 0 in core 01.

### Task 11: Core flip — typescript/11-testing.md

- [ ] REWRITE 11.1: Vitest → **`bun test`** (explicit imports from `bun:test`; colocated `*.test.ts`; `tests/` for integration unchanged). 11.4 MSW → scope to React (component tests, react/06); server HTTP tests use direct app invocation (cite bun/03). 11.8 determinism: fake timers via `bun:test` (`setSystemTime`), seeded fast-check unchanged. 11.6: `expect-type` standalone (or `expectTypeOf` import from its package) + `tsc --noEmit`-driven type tests — exact tool named after author verifies the 2026 standalone package. 11.11 Stryker → **accepted gap on bun** (no runner; coverage via `bun test --coverage` lcov; honesty framing kept: coverage reported never targeted; revisit note).
- [ ] Exemplar: imports flip to `bun:test` + the type-test import; fast-check property unchanged (`@fast-check/vitest` → plain `fast-check` with `test`/`fc.assert` form — rewrite the property block accordingly).
- [ ] Verify: rule count 12; `vitest`/`Vitest` tokens 0; `MSW` only in the React-scoping line; `Stryker` only in the gap note.

### Task 12: Core forward-refs + ledgers — typescript/10-api-design.md, typescript/15-performance.md, typescript/README.md

- [ ] 10.2 + cross-refs: `../typescript-node/08-build-and-distribution.md` → `../typescript-bun/08-build-and-distribution.md`.
- [ ] 15.8 + cross-refs: node/07 forward-ref → `../typescript-bun/07-bun-performance.md`; 15.2–15.3 gain a half-clause: V8-specific — on Bun (JSC) see the bun guide's carry table.
- [ ] typescript/README: charter/index mentions of companion guides → typescript-bun; toolchain words (pnpm→bun install, Vitest→bun test) in scope cells for 01/11.
- [ ] Verify: `grep -rn 'typescript-node' typescript/ | wc -l` → 0.

### Task 13: React counter-substitution — typescript-react/README.md + 06-testing-react.md

- [ ] README: ledger gains a row: component-test runner | family default `bun test` | **Vitest + MSW** | `bun test` runs no Service-Worker interception and its DOM story is immature; Playwright unchanged; revisit. Companion-guide mentions → typescript-bun.
- [ ] 06: scope para + 6.3 state explicitly that component tests run on **Vitest** (the recorded substitution; cite README ledger + core 11's bun default); MSW rules unchanged otherwise.
- [ ] Verify: react rule counts unchanged (README ✓, 06 → 8); `typescript-node` tokens → 0 in react/.

### Task 14: CLAUDE.md + root README row

- [ ] CLAUDE.md: architecture line (family = typescript ← typescript-bun + typescript-react; node retired at tag `node-guide-final`); hard-rules line gains "bun-first: bun install/bun test/Bun.build; tsc --noEmit is the typecheck gate; React keeps Vitest+MSW (recorded substitution)"; Git section unchanged.
- [ ] Root README: replace the "TypeScript on Node" row with: `| TypeScript on Bun | [\`typescript-bun/\`](./typescript-bun/) | Extends \`typescript/\`; [Bun docs](https://bun.com/docs) | Bun ≥ 1.3 pinned (no LTS), native TS execution + tsc gate, Hono on Bun.serve, Bun.SQL + Drizzle, bun test, JSC perf. |`
- [ ] Verify: root README has bun row, no node row; `grep -rn 'typescript-node' --include='*.md' . | grep -v docs/superpowers | grep -v node-guide-final` → 0.

### Task 15: Family sweep + publish (controller + workflow)

- [ ] Controller: `git mv typescript-bun/07-node-performance.md typescript-bun/07-bun-performance.md` (if Task 8's agent edited in place under the old name).
- [ ] Workflow sweep: family gates (counts per the task statements above); link integrity (all relative links resolve, incl. renamed 07); zero stale tokens (`typescript-node` outside docs/tag-notes; `pnpm|corepack|vitest|fastify|undici|piscina|esbuild` outside recorded substitutions/contrast lines — emit the grep results for review); coherence (substitution ledger ↔ core flips ↔ react counter-substitution all mutually consistent); spec-coverage critic vs the Spec.
- [ ] Fix findings; commit dev fixes.
- [ ] Publish reshape on `feat/bun-first-publish` from main: cherry-pick spec commit; commit plan (`docs: add bun-first migration plan`); single migration commit `feat: bun-first runtime -- typescript-bun replaces typescript-node` = full final tree delta (body: direction, transform summary, substitutions, react exception, tag pointer). Equivalence proof: `git diff feat/bun-first feat/bun-first-publish` → empty.
- [ ] Swap branch names; `git push origin node-guide-final`; push branch; ff-merge to main; push main; delete branches. Gates re-run on merged main. No co-author trailers anywhere.

---

## Self-Review (run inline — completed)

1. **Spec coverage:** transform map → Tasks 1–9; core flips → 10–12; react → 13; integration → 14; tag/atomic publish/sweep → Task 0 + 15. Author-verification items (lag API, Bun.SQL options, pino transports, bun publish provenance, fetch-pool surface, type-test package) each live in their owning task with honest fallbacks. ✓
2. **Placeholders:** none — every rewrite names its replacement content; verification items specify both branches (verified / fallback). ✓
3. **Consistency:** rule counts stated per task and re-checked in Task 15; `correlationId`, `bun.lock`, `.bun-version`, `node-guide-final`, `07-bun-performance.md` names used identically across tasks. ✓
