# 01 — Runtime and Toolchain

This guide extends [`typescript/`](../typescript/01-formatting-and-tooling.md) for code that runs on a Bun server; where it is stricter, it wins for Bun code. This chapter fixes the two things every Bun service decides before any domain code: which runtime it pins to, and how its modules resolve. The core guide already chose ESM and a reproducible install at the language level (core [01](../typescript/01-formatting-and-tooling.md), [12](../typescript/12-module-organization.md)); here we make those choices concrete against a specific Bun version, a specific module resolver, and a specific deployment shape — one stable process the orchestrator can kill and replace.

## What good looks like

```jsonc
// package.json
{
  "name": "@dexpace/orders",
  "type": "module",
  "imports": {
    "#config": "./src/config/index.ts",
    "#domain/*": "./src/domain/*.ts"
  },
  "scripts": {
    "dev": "bun --watch src/main.ts",
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "lint": "gts lint"
  }
}
```

```toml
# bunfig.toml — keep it minimal; only override what the project actually needs
[test]
coverage = true
```

```
# .bun-version — the exact Bun the team and CI run (pinned; see 1.1)
1.3.0
```

```typescript
// src/main.ts — the only entry point; Bun runs this .ts file directly
import {randomUUID} from 'node:crypto';
import {loadConfig} from '#config';
import {startServer} from '#domain/server';

const config = loadConfig(Bun.env);
const server = await startServer({...config, instanceId: randomUUID()});

process.once('SIGTERM', () => void server.close(10_000));
```

The `.bun-version` alongside this pins the exact runtime (`1.3.0`) for local shells and CI. This demonstrates 1.1 (exact Bun pin via `.bun-version`), 1.2 (`bun install` with a committed `bun.lock`), 1.3 (`node:` on every builtin), and 1.4 (`#config` / `#domain/*` subpaths instead of `../../config`). Bun runs the `.ts` entry directly with `tsc --noEmit` as the type gate (1.5), on a pinned stable release rather than a canary (1.6), and a single Bun process owns the whole service (1.8).

## Rules

### 1.1 — Pin Bun exactly; upgrades are deliberate, changelog-read changes.

**Reasoning, step by step:**
1. Bun has no LTS policy — there is no Active/Maintenance line to track, so there is no "floor" abstraction to lean on. With no LTS, the floor *is* the pin: the only safe contract is the exact version you tested. A `>=` range would silently admit a Bun release nobody on the team has run.
2. Encode the exact version in a committed `.bun-version` (e.g. `1.3.0`) and provision the same version from a pinned CI image / base container. Local shells, CI, and the production image then run byte-identical runtimes. There is no `engines`-style range to soften — one file, one version, everywhere (the `.bun-version` pin is mandated by the [bun migration spec](../docs/superpowers/specs/2026-06-06-typescript-bun-design.md); Bun's own docs configure behavior via `bunfig.toml`, not version).
3. Because Bun ships fast and is a single binary (runtime, installer, bundler, test runner all at once), an upgrade moves all of those at once. Treat a version bump as a reviewed change: read the changelog, run the suite, then move `.bun-version` and the CI image in the same diff. Never let the pin drift implicitly.
4. **Worked example:** Bun 1.3.4 lands with a fix you want. You bump `.bun-version` from `1.3.0` to `1.3.4`, update the CI image tag, read the changelog for behavior changes in the installer and test runner, and run the suite — one reviewed PR. The runtime never changes underneath a green build.
5. Never run a canary or untagged build in production (see 1.6); the pin always names a published stable release.

**Enforcement:** `.bun-version` committed with an exact version; CI provisions that exact Bun from a pinned image; review rejects any version range or unpinned upgrade.

### 1.2 — Install with `bun install`; commit `bun.lock` and use `--frozen-lockfile` in CI.

**Reasoning, step by step:**
1. Core [01](../typescript/01-formatting-and-tooling.md) chose `bun install` and a committed lockfile for the language. On a server the stake is higher: the install that runs in the production image must be byte-identical to the one a developer tested, or "works on my machine" becomes a deployment.
2. `bun install` resolves the dependency tree and writes a text-based `bun.lock` to the project root. Commit it. One installer (Bun itself, already pinned by 1.1) computes the tree on every machine — laptop, CI, build container — so there is no second resolver to disagree.
3. Install with `bun install --frozen-lockfile` in CI and image builds. Per Bun's docs the frozen flag installs the exact versions in `bun.lock` and, if `package.json` disagrees with the lockfile, exits with an error and does not update the lock. That turns a drifted lockfile into a hard build failure instead of a silent re-resolution that pulls a different transitive version into your artifact.
4. **Worked example:** the lockfile pins `hono@4.6.3`. A fresh, unfrozen install months later might resolve `4.7.0`; `--frozen-lockfile` refuses, so the image you ship contains the version you tested, and any bump is a reviewed lockfile diff.

**Enforcement:** `bun.lock` committed; `bun install --frozen-lockfile` in CI and image builds; review rejects a manually edited or uncommitted lockfile.

### 1.3 — Prefix every builtin import with `node:`.

**Reasoning, step by step:**
1. Bun implements the Node builtins (`fs`, `path`, `crypto`, …) with ~90%+ compatibility, and reaches them under the `node:` namespace. `import {readFile} from 'fs'` and `import {readFile} from 'node:fs'` both resolve the builtin today, but only the prefixed form is unambiguous. A bare `fs`, `path`, or `crypto` specifier can be shadowed by an npm package of that name landing in `node_modules` — accidentally or via a supply-chain attack.
2. The prefix is explicit intent the resolver cannot misread: `node:` names are reserved and can never be overridden by a package, so the import means the builtin and nothing else. (Bun's own native APIs live on the `Bun` global and under `bun:*` specifiers like `bun:sqlite`, which are a separate, unshadowable namespace — see ch. 04.)
3. It is greppable and future-proof: `grep -r "from 'node:"` enumerates every platform dependency in one pass, and newer builtins are *only* reachable through the prefix.
4. **Worked example:** a transitive dependency publishes a package literally named `crypto`. Code importing `'crypto'` silently binds to it; code importing `'node:crypto'` is untouched. The prefix is a one-token defense against a whole class of shadowing.

**Enforcement:** ESLint `n/prefer-node-protocol` (error); review rejects any unprefixed builtin import.

### 1.4 — Use `#imports` subpaths for internal cross-area references.

**Reasoning, step by step:**
1. Deep relative paths (`../../../domain/order`) are noise that breaks the moment a file moves, and they obscure whether an import crosses an architectural boundary or stays local.
2. Subpath imports are a package.json standard, not bundler magic: declare an `imports` map whose keys begin with `#` (the reserved sigil for internal-only specifiers, never exposed to consumers), and Bun resolves them natively, as do `tsc` and the test runner. Because Bun runs `.ts` directly (1.5), point the map at source: `"#domain/*": "./src/domain/*.ts"`.
3. The source imports `#domain/order`; Bun loads `src/domain/order.ts` and strips its types at load. There is no separate `dist/` mapping to keep in sync because there is no separate compile output to map to.
4. **Worked example:**
   ```jsonc
   // package.json
   "imports": {"#domain/*": "./src/domain/*.ts"}
   ```
   ```typescript
   import {placeOrder} from '#domain/order'; // not '../../domain/order'
   ```
   Move the importing file to a different depth and the specifier is unchanged — the boundary is named, not counted in `../`.
5. Reserve `#` keys for genuine cross-area boundaries (domain, config, infrastructure); within one small area, plain relative imports stay clearer than a subpath.

**Enforcement:** `imports` map in `package.json`; review prefers a declared subpath over any `../../` that crosses an area boundary.

### 1.5 — Run TypeScript natively; `tsc --noEmit` is the type gate, not the build.

**Reasoning, step by step:**
1. Bun executes TypeScript directly: its `ts` loader "strips out all TypeScript syntax, then behaves identically to the `js` loader," and Bun "does not perform typechecking" (Bun docs, runtime loaders). There is no `tsc` build step between source and runtime — `bun src/main.ts` parses, strips, and runs the file. This is the inner loop *and* the production path: the same `.ts` source ships and runs.
2. That this is safe is not luck — it rests on the family's erasable-syntax stance. Type stripping can only run code whose type syntax is purely erasable; `enum`, runtime `namespace`, constructor parameter properties, and `import =` all emit *runtime* code that a stripper cannot reproduce. Core [ts/03 §3.12](../typescript/03-the-type-system.md) bans exactly those forms, and `erasableSyntaxOnly` makes each a compile error. Because every dexpace file is already erasable, Bun's strip-and-run is total: nothing the runtime drops was load-bearing. This is BUN-6 — type stripping is the runtime, so erasable-only is a runtime correctness rule, not just a style one.
3. But stripping *erases* types; it never *checks* them. Shipping stripped source with no separate check would mean running code the type checker never approved. So `tsc --noEmit` becomes a non-negotiable CI gate (BUN-6): it type-checks the whole program and emits nothing, since Bun, not `tsc`, produces the runnable code. The two tools split cleanly — Bun runs, `tsc` verifies.
4. **Worked example:** `"dev": "bun --watch src/main.ts"` for the loop, `"typecheck": "tsc --noEmit"` in CI. A field-name typo that the stripper would happily run is caught by the `typecheck` job before merge. Production runs the identical `.ts` files Bun ran in dev; the paths never diverge in what reaches users.
5. Revisit only if Bun ever gains a type-checking mode with equivalent diagnostics; until then, `tsc --noEmit` owns correctness and Bun owns execution.

**Enforcement:** `tsc --noEmit` is a required CI gate; entry points are `.ts` run by Bun (no `tsc` build/emit step in `start`/`Dockerfile`); review rejects shipping without the typecheck gate green.

### 1.6 — Ship on a pinned stable release; no canary or untagged Bun in production.

**Reasoning, step by step:**
1. Bun publishes canary builds for testing unreleased fixes. A canary is Bun telling you the behavior is unproven and may change before it lands in a stable release. Building a service on one means a routine rebuild can pull in different, unreviewed behavior.
2. Canary and untagged builds carry no stability contract, so an incident traced to one has nothing stable to debug against — the answer is "it changed since you pulled it." The exact-pin discipline of 1.1 only holds if the thing you pin is a published stable version.
3. If a capability exists only in a canary, treat that as a signal it is not ready for a service that must stay up. Wait for it to ship in a stable release, or solve the problem on the current stable surface.
4. **Worked example:** trying `bun upgrade --canary` locally to confirm a fix is fine; pinning `.bun-version` or the CI image to a canary is not. The local check proves the fix is coming; the production pin waits for the stable release that carries it.

**Enforcement:** `.bun-version` and the CI/base image name a published stable release; no `--canary` or untagged Bun in `start`/`Dockerfile`/CI; review rejects canary builds on any production path.

### 1.7 — ESM only; quarantine CJS interop to declared bridge modules.

**Reasoning, step by step:**
1. Core [12.1](../typescript/12-module-organization.md) ships ESM and confines CommonJS to a named bridge. Bun's CJS interop is transparent — it can `import` most CJS packages directly and even allows `require` and `import` in the same file — which makes the seam *rarer* than on Node: many packages that needed a bridge before just work. The rule still stands, because "rarer" is not "never," and an unmanaged `require` is still an opaque hole in the module graph.
2. When a dependency genuinely forces a CJS seam (a package whose interop Bun cannot resolve cleanly, or one needing `createRequire(import.meta.url)`), that machinery is allowed to live in exactly one place per dependency. Mark each such file with a `// bridge:` header naming the dependency and why it needs the seam, and have it re-export a clean ESM surface. One file per CJS dependency carries the impedance mismatch; the other modules stay pure ESM and never learn `require` exists.
3. Confining interop this way keeps the module graph statically analyzable for the rest of the codebase — tree-shaking, cycle detection, and `verbatimModuleSyntax` all work — while still consuming the occasional package that needs the seam.
4. This is the loader-side discipline behind the ESM/`node:`-prefix rules: one module system, `import`/`export` only, with `require`/`createRequire` quarantined to the declared bridge and nowhere else. (Bun makes pure ESM-nativeness easy enough that BUN-6 spends its budget on the typecheck gate instead — but ESM discipline still lives here in ch. 01.)
5. **Worked example:**
   ```typescript
   // bridge: legacy-metrics is CJS-only (no "exports"); wrap it here, expose ESM.
   import {createRequire} from 'node:module';
   const require = createRequire(import.meta.url);
   const legacy = require('legacy-metrics') as {record(name: string): void};
   export const recordMetric = (name: string): void => legacy.record(name);
   ```
   Every other module imports `recordMetric` from this bridge and stays unaware of the `require` underneath.

**Enforcement:** `"type": "module"`; review rejects `require`/`createRequire`/`module.exports` outside a file whose `// bridge:` header declares it. See core [12](../typescript/12-module-organization.md).

### 1.8 — Run one process per container; scale by adding containers, not forking workers.

**Reasoning, step by step:**
1. A single process per container makes the unit of deployment and the unit of failure the same thing. The orchestrator (Kubernetes, ECS) already owns replication, rolling updates, health checks, and restart-on-crash; running its job a second time inside the container only duplicates the responsibility and splits the signals.
2. Scale horizontally: more load means more replicas the orchestrator schedules and load-balances across nodes, not more forked workers contending for one container's CPU quota. One process keeps memory accounting, log streams, and CPU limits legible per replica.
3. This is the runtime face of BUN-1 (crash-only): a process in unknown state is replaced, not nursed. With one process per container, an uncaught exception exits non-zero, the orchestrator starts a fresh replica, and there is no surviving worker holding corrupt state for the dead one. Forking would mean half-dead worker pools the supervisor cannot reason about.
4. **Worked example:** to handle 4× traffic, set replicas from 2 to 8 and the load balancer fans out. No process forking, no worker-count tuning per pod size — the deployment manifest is the scaling knob, and each replica is identical and disposable.
5. The exception is genuinely CPU-bound work offloaded to a bounded Bun Workers pool inside a request (covered with the concurrency chapter) — that is parallelizing a computation, not replicating the server, and the process is still one supervised unit.

**Enforcement:** one Bun entry per image; no in-container process forking in service code; replication declared in the orchestrator manifest, checked in review.

## Cross-references

- ESM discipline, the CJS bridge pattern, and the module graph: core [12-module-organization.md](../typescript/12-module-organization.md).
- The platform tsconfig override: this guide owns `moduleResolution: "bundler"` for Bun. Core [01-formatting-and-tooling.md](../typescript/01-formatting-and-tooling.md) keeps only the six strictness flags and states that platform settings live with the runtime; the `module`/resolution override no longer rides in core and is set here for Bun.
- Why strip-and-run is safe — erasable syntax only: core [ts/03 §3.12](../typescript/03-the-type-system.md), the rule that makes Bun's native TS execution (1.5, BUN-6) total.
- Crash-only error policy (BUN-1) that 1.8's one-process shape serves: core [08-error-handling.md](../typescript/08-error-handling.md).
- The supply-chain stakes behind the runtime and install pins in 1.1–1.2: [../security.md](../security.md).
</content>
</invoke>
