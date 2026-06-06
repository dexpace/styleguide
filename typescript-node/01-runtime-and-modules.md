# 01 — Runtime and Modules

This guide extends [`typescript/`](../typescript/01-formatting-and-tooling.md) for code that runs on a Node.js server; where it is stricter, it wins for Node code. This chapter fixes the two things every Node service decides before any domain code: which runtime it pins to, and how its modules resolve. The core guide already chose ESM and pnpm at the language level (core [01](../typescript/01-formatting-and-tooling.md), [12](../typescript/12-module-organization.md)); here we make those choices concrete against a specific Node version, a specific module resolver, and a specific deployment shape — one stable process the orchestrator can kill and replace.

## What good looks like

```jsonc
// package.json
{
  "name": "@dexpace/orders",
  "type": "module",
  "engines": {"node": ">=24"},
  "packageManager": "pnpm@9.12.0",
  "imports": {
    "#config": "./dist/config/index.js",
    "#domain/*": "./dist/domain/*.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node ./dist/main.js",
    "test": "vitest run"
  }
}
```

```typescript
// src/main.ts — the only entry point; compiled to dist/main.js for `node` to run
import {randomUUID} from 'node:crypto';
import {parseArgs} from 'node:util';
import {loadConfig} from '#config';
import {startServer} from '#domain/server';

const config = loadConfig(process.env);
const server = await startServer({...config, instanceId: randomUUID()});

const {values} = parseArgs({options: {drainMs: {type: 'string'}}});
process.once('SIGTERM', () => void server.close(Number(values.drainMs ?? 10_000)));
```

The `.nvmrc` alongside this pins the same major (`24`) for local shells. This demonstrates 1.1 (`engines` floor at the Active LTS), 1.2 (`packageManager` for corepack), 1.3 (`node:` on every builtin), and 1.4 (`#config` / `#domain/*` subpaths instead of `../../config`). The `start` script runs compiled JS (1.5), with no `--experimental-*` flag in sight (1.6), and a single `node` process owns the whole service (1.8).

## Rules

### 1.1 — Run a Node Active LTS line, floor pinned at 24.

**Reasoning, step by step:**
1. Only the Active LTS and Maintenance lines receive security patches. A service on an end-of-life runtime accrues unpatched CVEs in the V8 engine and the HTTP stack — attack surface you cannot fix without an upgrade you have already deferred.
2. Old runtimes also lack platform APIs the later chapters assume: `AbortSignal.any` (composing cancellation), explicit-resource-management `using` (deterministic cleanup, ch. on resources), and a stable `node:test`/`fetch` surface. Writing around their absence is wasted effort.
3. Encode the floor in `package.json` so an install on an old runtime fails loudly: `"engines": {"node": ">=24"}`. That `>=24` is the lower guard only — it admits odd non-LTS majors (25, 27…) that the security-patch premise excludes. The operative runtime is the current Active LTS line, pinned *exactly* by `.nvmrc` and the CI image; the engine floor merely rejects anything below it. Non-LTS Current lines are never run in production. Commit a `.nvmrc` containing the exact LTS major (`24`) so `nvm use` selects the same runtime in every local shell and CI step.
4. **Worked example:** a CI runner provisions Node 20. `pnpm install` aborts with `Unsupported engine`, not three weeks later with a missing-API crash in production. The floor turns a latent incident into a build error.
5. Track the line: bump the floor when a new LTS goes Active, as a reviewed change, never drifting silently behind.

**Enforcement:** `"engines"` in `package.json`; `.nvmrc` committed; CI provisions via corepack/nvm and respects the engine floor.

### 1.2 — Pin the package manager (pnpm) through corepack's `packageManager` field.

**Reasoning, step by step:**
1. Core [01](../typescript/01-formatting-and-tooling.md) chose pnpm and the `packageManager` field for the language. On a server the stake is higher: the install that runs in the production image must be byte-identical to the one a developer tested, or "works on my machine" becomes a deployment.
2. `"packageManager": "pnpm@9.12.0"` lets corepack provision that exact pnpm on every machine — laptop, CI, build container — so one resolver computes the dependency tree everywhere.
3. Commit `pnpm-lock.yaml` and install with `pnpm install --frozen-lockfile` in CI and image builds. The frozen flag makes a drifted lockfile a hard error instead of a silent re-resolution that pulls a different transitive version into your artifact.
4. **Worked example:** the lockfile pins `undici@6.19.2`. A fresh, unfrozen install months later would resolve `6.21.0`; `--frozen-lockfile` refuses, so the image you ship contains the version you tested, and any bump is a reviewed lockfile diff.

**Enforcement:** `packageManager` in `package.json`; `pnpm-lock.yaml` committed; `pnpm install --frozen-lockfile` in CI and image builds.

### 1.3 — Prefix every builtin import with `node:`.

**Reasoning, step by step:**
1. `import {readFile} from 'fs'` and `import {readFile} from 'node:fs'` both resolve the builtin today, but only the prefixed form is unambiguous. A bare `fs`, `path`, or `crypto` specifier can be shadowed by an npm package of that name landing in `node_modules` — accidentally or via a supply-chain attack.
2. The prefix is explicit intent the resolver cannot misread: `node:` names are reserved and can never be overridden by a package, so the import means the builtin and nothing else.
3. It is greppable and future-proof: `grep -r "from 'node:"` enumerates every platform dependency in one pass, and newer builtins (`node:test`, `node:sqlite`) are *only* reachable through the prefix.
4. **Worked example:** a transitive dependency publishes a package literally named `crypto`. Code importing `'crypto'` silently binds to it; code importing `'node:crypto'` is untouched. The prefix is a one-token defense against a whole class of shadowing.

**Enforcement:** ESLint `n/prefer-node-protocol` (error); review rejects any unprefixed builtin import.

### 1.4 — Use `#imports` subpaths for internal cross-area references.

**Reasoning, step by step:**
1. Deep relative paths (`../../../domain/order`) are noise that breaks the moment a file moves, and they obscure whether an import crosses an architectural boundary or stays local.
2. Subpath imports are a package.json standard, not bundler magic: declare an `imports` map whose keys begin with `#` (the reserved sigil for internal-only specifiers, never exposed to consumers), and Node, `tsc`, and Vitest all resolve them identically given the `nodenext` resolution the core tsconfig already mandates ([typescript/01](../typescript/01-formatting-and-tooling.md)).
3. Point the map at compiled output so the runtime resolves real files: `"#domain/*": "./dist/domain/*.js"`. The source imports `#domain/order`; the running program loads `dist/domain/order.js`.
4. **Worked example:**
   ```jsonc
   // package.json
   "imports": {"#domain/*": "./dist/domain/*.js"}
   ```
   ```typescript
   import {placeOrder} from '#domain/order'; // not '../../domain/order'
   ```
   Move the importing file to a different depth and the specifier is unchanged — the boundary is named, not counted in `../`.
5. Reserve `#` keys for genuine cross-area boundaries (domain, config, infrastructure); within one small area, plain relative imports stay clearer than a subpath.

**Enforcement:** `imports` map in `package.json`; review prefers a declared subpath over any `../../` that crosses an area boundary.

### 1.5 — Compile with `tsc` for production; type stripping is a dev-loop convenience only.

**Reasoning, step by step:**
1. Node can run TypeScript directly by stripping types at load, which is excellent for a fast inner loop — edit, `node src/main.ts`, see the result, no build step.
2. A production artifact must not depend on that. Run `tsc` to emit plain `.js` plus `.js.map` sourcemaps into `dist/`, and let `node ./dist/main.js` execute ordinary JavaScript. The deployed thing is already parsed and checked; startup does no transform, and a crash maps back to TypeScript line numbers through the sourcemap.
3. Compiling ahead of time also runs the full type check as a release gate. Type stripping deliberately does *not* type-check — it only erases — so shipping stripped source would mean shipping code the type checker never approved.
4. **Worked example:** `"start": "node ./dist/main.js"` after `"build": "tsc"`. Dev uses `node --watch src/main.ts` for speed; the image runs the compiled output. The two paths never diverge in what reaches users.
5. Revisit when on-the-fly stripping stabilizes as a supported production mode with equivalent diagnostics; until then, compiled JS is the artifact.

**Enforcement:** `build` script runs `tsc`; `start` runs from `dist/`; the production image contains compiled JS, checked in review and by the Dockerfile.

### 1.6 — Ship on the stable runtime surface; no `--experimental-*` flags in production.

**Reasoning, step by step:**
1. A flag named `--experimental-*` is Node telling you the behavior may change, or vanish, in a minor release. Building a service on one means a routine runtime bump can silently alter or remove a feature you depend on.
2. Experimental APIs emit runtime warnings and carry no semver guarantee, so an incident traced to one has no stable contract to debug against — the answer is "it changed."
3. If a capability exists only behind an experimental flag, treat that as a signal the capability is not ready for a service that must stay up. Wait for it to graduate, or solve the problem on the stable surface.
4. **Worked example:** prototyping behind `--experimental-vm-modules` is fine; depending on it in `start` is not. The prototype proves an idea; the production path waits for the API to stabilize before it ships.

**Enforcement:** no `--experimental-*` in `start`/`Dockerfile`/CI run commands; review rejects experimental flags on any production entry point.

### 1.7 — ESM only; quarantine CJS interop to declared bridge modules.

**Reasoning, step by step:**
1. Core [12.1](../typescript/12-module-organization.md) ships ESM and confines CommonJS to a named bridge. On the runtime side the seam is concrete: a dependency that is CJS-only must be reached through `createRequire(import.meta.url)` or a dynamic `import()`, and that machinery is allowed to live in exactly one place per dependency.
2. Mark each such file with a `// bridge:` header naming the dependency and why it needs the seam, and have it re-export a clean ESM surface. One file per CJS dependency carries the impedance mismatch; the other modules stay pure ESM and never learn `require` exists.
3. Confining interop this way keeps the module graph statically analyzable for the rest of the codebase — tree-shaking, cycle detection, and `verbatimModuleSyntax` all work — while still consuming the occasional CJS-only package.
4. This is where NODE-6 (ESM-native) lands in the loader: one module system, `import`/`export` only, with `require`/`createRequire` quarantined to the declared bridge and nowhere else.
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
1. A single process per container makes the unit of deployment and the unit of failure the same thing. The orchestrator (Kubernetes, ECS) already owns replication, rolling updates, health checks, and restart-on-crash; running its job a second time inside the container with `cluster` only duplicates the responsibility and splits the signals.
2. Scale horizontally: more load means more replicas the orchestrator schedules and load-balances across nodes, not more forked workers contending for one container's CPU quota. One process keeps memory accounting, log streams, and CPU limits legible per replica.
3. This is the runtime face of NODE-1 (crash-only): a process in unknown state is replaced, not nursed. With one process per container, an `uncaughtException` exits non-zero, the orchestrator starts a fresh replica, and there is no surviving worker holding corrupt state for the dead one. Forking would mean half-dead worker pools the supervisor cannot reason about.
4. **Worked example:** to handle 4× traffic, set replicas from 2 to 8 and the load balancer fans out. No `cluster.fork()`, no worker-count tuning per pod size — the deployment manifest is the scaling knob, and each replica is identical and disposable.
5. The exception is genuinely CPU-bound work offloaded to a bounded `worker_threads` pool inside a request (covered with the concurrency chapter) — that is parallelizing a computation, not replicating the server, and the process is still one supervised unit.

**Enforcement:** one `node` entry per image; no `cluster` import in service code; replication declared in the orchestrator manifest, checked in review.

## Cross-references

- ESM discipline, the CJS bridge pattern, and the module graph: core [12-module-organization.md](../typescript/12-module-organization.md).
- The tsconfig that makes `nodenext` resolution and ESM emit work: core [01-formatting-and-tooling.md](../typescript/01-formatting-and-tooling.md).
- Deterministic cleanup with `using` (an API the LTS floor in 1.1 unlocks): core [13-resource-management.md](../typescript/13-resource-management.md).
- Crash-only error policy (NODE-1) that 1.8's one-process shape serves: core [08-error-handling.md](../typescript/08-error-handling.md).
- The supply-chain stakes behind the runtime and package-manager pins in 1.1–1.2: [../security.md](../security.md).
