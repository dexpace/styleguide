# 08 — Build and Distribution

How a Bun package goes from source to a thing other code installs or a process the orchestrator runs. The build is part of the codebase, and it earns the same discipline as the code it ships: a checked-in script, a locked input set, a public surface that is reviewed when it changes, and an artifact that is byte-identical from the same source. A library and a service want opposite builds — the library hands its consumer an untouched module graph, the service hands the orchestrator one self-contained file — so this chapter splits on that line and then converges on the supply-chain rules both owe. The `package.json` `exports` map is where the core guide's public-surface contract ([../typescript/10-api-design.md](../typescript/10-api-design.md), 10.2) stops being a convention and becomes a resolution error; the module-graph shape it publishes is [../typescript/12-module-organization.md](../typescript/12-module-organization.md).

## What good looks like

```jsonc
// package.json — a library done right. The exports map is the contract; everything else serves it.
{
  "name": "@dexpace/booking",
  "version": "2.1.0",
  "license": "MIT",                            // SPDX id — explicit, so the tarball carries its terms
  "repository": {                              // links the package back to its source (provenance, 8.6)
    "type": "git",
    "url": "git+https://github.com/dexpace/booking.git"
  },
  "type": "module",
  "exports": {                                  // the locked public surface — 8.3, enforcing 10.2
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" }
  },
  "files": ["dist"],                            // publish allowlist — nothing else reaches the tarball
  "sideEffects": false,                          // import is free; bundlers may drop unused modules (12.6)
  "publishConfig": {                           // public scoped publish with signed provenance — 8.6
    "access": "public",
    "provenance": true
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json",      // plain tsc: .js + .d.ts + sourcemaps, no bundler — 8.1
    "check": "bun run tsc --noEmit && bun run eslint . && bun test",
    "api:update": "bun run api-extractor run --local", // dev: regenerate the committed report — 8.5
    "api": "bun run api-extractor run",          // verify: fail on any drift from the committed report — 8.5
    "lint:publish": "bun run publint && bun run attw --pack .", // package-health gate before anything ships — 8.4
    "prepublishOnly": "bun run build && bun run api && bun run lint:publish"
  }
}
```

The `exports` map names one entry point and makes every other path a resolution error for consumers (8.3, the mechanical form of 10.2); `files` keeps the tarball to `dist` so no test or config leaks; `sideEffects: false` keeps imports tree-shakeable (12.6). The publish gate is three boring tools in series — `publint` and `arethetypeswrong` verify the package resolves under ESM/CJS/types (8.4), `api-extractor` (the verifying `api`, not the dev-only `api:update` that regenerates the report) diffs the public surface (8.5) — and they run in `prepublishOnly` so a broken package cannot leave a laptop. `publishConfig` makes the scoped package public and turns on signed provenance, and `repository` links the tarball back to the source the attestation points at (8.6). There is no `engines` or `packageManager` field here: on Bun the runtime is pinned out-of-band by a committed `.bun-version` ([./01-runtime-and-toolchain.md](./01-runtime-and-toolchain.md)), so the same Bun runs in CI and on every laptop and the artifact is reproducible (8.9).

## Rules

### 8.1 — Libraries build with plain `tsc`: declarations and sourcemaps, no bundler between source and consumers.

**Reasoning, step by step:**
1. A library's consumer has a bundler, and that bundler optimizes best against the original module graph — one `.js` per source module, with the `import`/`export` edges intact — because that is what it tree-shakes, code-splits, and dead-code-eliminates against. Pre-bundling the library collapses that graph into one blob the consumer's tooling can no longer see into, so it ships more code, not less. A library is also runtime-agnostic: its consumer may run on Node, Bun, a browser, or an edge worker, so the build must not bake in one runtime's assumptions — which is exactly why it stays `tsc`, not `Bun.build`, even in a Bun shop.
2. So the library build is `tsc` and nothing else: it emits one `.js` per module, the matching `.d.ts` declarations, and `.js.map` sourcemaps that point back at the original `.ts`. The output mirrors the input, which is exactly what a downstream graph wants — and it resolves the same wherever the consumer runs.
3. Declarations and sourcemaps are not optional. The `.d.ts` is the typed half of the contract the consumer compiles against; the sourcemap is what makes a stack trace in the consumer's app land on your real source line instead of a column in a minified bundle.

```jsonc
// tsconfig.build.json — emit the graph, not a bundle
{ "compilerOptions": { "declaration": true, "sourceMap": true, "outDir": "dist", "module": "nodenext" } }
```
**Enforcement:** review that a library has no bundler in its `build` script; `attw` (8.4) confirms the emitted `.d.ts`/`.js` resolve for consumers.

### 8.2 — Services bundle with `Bun.build` into one artifact, or compile to a single-file executable; the build script is boring either way.

**Reasoning, step by step:**
1. A service is the opposite case: nothing downstream consumes its module graph, so there is no graph to preserve. What the orchestrator wants is one thing it can copy into an image and start, with the fewest module resolutions at boot. Bundling collapses the graph on purpose here, which is the right move precisely because it was the wrong one for a library (8.1). The default is `Bun.build` to one JS artifact: one entry in, one `dist/server.js` out, dependencies inlined, run by a Bun runtime that is already in the image. Cold start drops because the loader resolves one file instead of walking `node_modules`, and the deploy is one artifact to ship and one thing to hash (8.9).
2. The build is a checked-in script, not a long shell incantation, because `Bun.build` is a JS API, not a flag soup — and a build nobody can read is a build nobody can audit. The script is a few lines: `entrypoints`, `outdir`, `target: "bun"` (the runtime the artifact runs on), `sourcemap: "linked"`, and stop. Same source, same output; no clever plugin chain.
3. For a service that must run where Bun is not installed — a scratch container, an operator's laptop, a CI tool — `bun build --compile` produces a single-file executable that embeds the Bun runtime, all imported files, and all dependencies, so there is no `node_modules` and no runtime to provision at the target at all. The trade-off is the choice: a `--compile` binary is large (it carries a whole runtime, tens of megabytes) and cross-compiled per platform with `--target` (e.g. `bun-linux-x64`), but its cold start is the best on offer (`--bytecode` precompiles, roughly halving startup) and it needs nothing on the host. Choose the plain `Bun.build` artifact when the image already has Bun (smaller layer, the common service case); choose `--compile` when "nothing else installed" is the requirement (CLIs you hand out, sidecars in minimal images).

```ts
// build.ts — the default service build, one boring checked-in script (run: bun run build.ts)
await Bun.build({
  entrypoints: ["./src/server.ts"],
  outdir: "./dist",
  target: "bun",
  sourcemap: "linked",
});
```
```jsonc
// package.json (service) — pick one; the compile path is one boring line
{ "scripts": {
  "build": "bun run build.ts",                                              // default: one JS artifact for a Bun image
  "build:bin": "bun build src/server.ts --compile --minify --bytecode --sourcemap --outfile dist/server" // self-contained binary
} }
```
**Enforcement:** review that the build is the whole script — `Bun.build` config legible, or the `--compile` flags legible; the artifact is a single file plus its sourcemap, and the `--compile` vs `Bun.build` choice is justified by where the service runs.

### 8.3 — The `exports` field is locked: explicit entry points only, and deep imports are resolution errors.

**Reasoning, step by step:**
1. Without an `exports` field, a package's entire file tree is reachable: a consumer can write `import 'pkg/dist/internal/http-pool.js'` and now that internal file is part of your public contract whether you meant it or not, because someone depends on it. The core guide's 10.2 says `index.ts` is the only front door; `exports` is what makes that wall real instead of aspirational.
2. So `exports` lists exactly the entry points the package promises and nothing else. Any path not listed — every deep import into `dist/` — fails to resolve at the consumer's `import`, at install-time tooling, and in the type-checker. The internal modules become genuinely internal: free to move, rename, or split without a major bump, because no caller could have reached them.
3. List the `types` condition first and the `import` condition for the ESM entry; an ESM-only library needs no `require` condition (12.1). One entry, one contract — multiple entry points are deliberate, each its own reviewed promise (10.3), not a convenience door punched through the wall.

```jsonc
"exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }
// consumer: import {bookSeat} from '@dexpace/booking'        // resolves — the one front door
//           import x from '@dexpace/booking/dist/seat-map.js' // ERR_PACKAGE_PATH_NOT_EXPORTED
```
**Enforcement:** `exports` lists only intended entry points; `publint` (8.4) flags a malformed map; a new entry point is reviewed as a contract change (10.3).

### 8.4 — `publint` and `arethetypeswrong` gate publishing: package health is verified in CI, not by consumer bug reports.

**Reasoning, step by step:**
1. A package can typecheck and test green on your machine and still be broken for consumers: an `exports` condition pointing at a file that does not exist, a `.d.ts` that resolves under `node` module resolution but not `bundler`, an ESM default that a CJS consumer cannot import. None of this shows up in your own build — it shows up as a stranger's GitHub issue a week after release.
2. So two tools run in CI before publish. `publint` checks the package's structure — `exports`, `main`/`module`/`types`, file references, ESM/CJS correctness — against the rules every consumer's resolver applies. `arethetypeswrong` (`attw`) simulates resolving the published types under every module-resolution mode and reports which combinations break.
3. They gate, not warn. A red `publint` or `attw` fails the publish job the same way a failed test does, so a structurally broken package cannot reach the registry. This is package health moved left: caught in the pipeline that built the tarball, not in the consumer's that installed it.

```jsonc
"scripts": { "lint:publish": "bun run publint && bun run attw --pack ." } // both must pass before the publish step runs
```
**Enforcement:** `publint` + `attw --pack` as a required CI job gating publish; a failure blocks the release, no override.

### 8.5 — api-extractor produces an API report on libraries: the public-surface diff is a reviewed artifact, and unintended breaks fail CI.

**Reasoning, step by step:**
1. This is the Bun analog of the JVM guide's `binary-compatibility-validator` ([../kotlin-jvm/08-build-and-distribution.md](../kotlin-jvm/08-build-and-distribution.md), 8.5): the public surface is too important to be reviewed by reading a diff of implementation files. A widened return type, a removed overload, a newly-optional parameter — each is a contract change that a reviewer skimming a feature PR will miss, because the change is in a `.d.ts` they did not open.
2. So `api-extractor` rolls the package's public types into a single committed `*.api.md` report. The report is the surface, flattened. When a PR changes the public API, it changes that file, and the change shows up as a reviewable diff a human signs off on — an intentional addition gets the report updated and approved, an accidental one gets caught at review.
3. CI fails when the generated report does not match the committed one, exactly as the JVM validator fails on a stale `*.api` snapshot. The build forces the author to either regenerate the report (a deliberate surface change, reviewed) or fix the leak (an unintended one). The diff is the gate; an unreviewed break cannot merge.

```jsonc
"scripts": { "api": "bun run api-extractor run" } // CI runs the verifying mode; a mismatch with the committed report fails
```
**Enforcement:** committed `*.api.md` report; `api-extractor run` in CI fails on any undocumented surface change; the report diff is reviewed like code.

### 8.6 — Publish with provenance and 2FA; commit the lockfile always.

**Reasoning, step by step:**
1. Supply-chain integrity runs in both directions, and both directions need a guarantee. Outbound: a consumer installing your package wants proof the tarball was built from the source it claims and not swapped by a compromised token. Inbound: your build wants proof the dependency tree it resolves is the exact one that was reviewed.
2. Outbound is npm provenance plus 2FA. `bun publish` carries the tarball, the access flag, and 2FA (`--auth-type`, `--otp`), but as of this writing it does not emit a provenance attestation — there is no `--provenance` flag. So the publish step in CI runs `npm publish --provenance` (from a CI runner with an OIDC identity), which attaches a signed, verifiable link from the artifact back to the commit and workflow that produced it. This is a registry-tooling choice, not a workflow regression: `bun` builds, installs, and audits the package; the one publish call that generates the attestation goes through `npm` until Bun ships provenance. 2FA on the publish step means a stolen token alone cannot push a release, and a laptop cannot generate provenance — which is the point: publishing moves to CI (BUN-4). When `bun publish` gains `--provenance`, the call swaps and the rest of the pipeline is unchanged.
3. Inbound is the committed lockfile, no exceptions. `bun.lock` pins every transitive dependency to an exact version and integrity hash, so `bun install` resolves the reviewed tree and not a freshly-floated one. An uncommitted or `.gitignore`d lockfile means every install is an unreviewed code change ([../security.md](../security.md), BUN-4).

```jsonc
"scripts": { "publish:ci": "npm publish --provenance --access public" } // CI-only; provenance via npm until bun publish supports it; 2FA enforced
```
**Enforcement:** `--provenance` on the CI publish step (via `npm publish` — `bun publish` has no `--provenance` flag yet), 2FA required by the registry; `bun.lock` committed and never ignored (BUN-4, [../security.md](../security.md)).

### 8.7 — Dependency policy: minimal, pinned, audited — every new dep justified in the PR.

**Reasoning, step by step:**
1. Every dependency is code you did not write running with your process's full privileges, and the transitive tree is the real surface (BUN-4). The cheapest dependency to secure is the one you did not add: a few lines of owned code, a `node:` builtin, or a `Bun.*` builtin beats pulling a package and its subtree, every time the owned version is viable.
2. So a new dependency is justified in the PR that adds it — its size and transitive count, who maintains it and how actively, and why the standard library or a small in-house function will not do. The reviewer weighs that cost the way they weigh any other code being merged, because that is what it is.
3. Versions are pinned by the committed lockfile (8.6), and `bun audit` runs as a CI gate. `bun audit` checks the installed tree against the advisory database and exits non-zero when it finds a vulnerability, which is what makes it a gate rather than a report. Zero tolerance for a known vulnerability in a direct dependency — that fails the build outright; a transitive one is assessed and mitigated within one sprint, not ignored ([../security.md](../security.md)). An unaudited install is an unreviewed change, and this is the gate that reviews it.

```jsonc
"scripts": { "audit": "bun audit --audit-level=high --prod" } // CI gate; exits non-zero on a known direct-dep vuln, failing the build
```
**Enforcement:** `bun audit` as a required CI check, zero known vulns in direct deps; every dependency addition justified in its PR (BUN-4, [../security.md](../security.md)).

### 8.8 — Version with changesets: every bump traces to a changelog entry, and breaking is MAJOR with no exceptions.

**Reasoning, step by step:**
1. A version number that is bumped by hand drifts from what actually changed: someone ships a breaking change and tags it MINOR because the diff "felt small," and a consumer's `^` range pulls it in and breaks at runtime. The version has to be derived from the changes, not asserted over them.
2. So each change that affects consumers ships with a changeset — a small file in the PR declaring its bump level (`patch`/`minor`/`major`) and the changelog line it contributes. At release, `changesets` aggregates them, computes the resulting version, and writes the changelog, so the version and the notes are a function of the merged changesets and cannot disagree with them.
3. The bump math is semver and non-negotiable, the same line the API report (8.5) draws and the git guide's release rules require ([../git-and-code-review.md](../git-and-code-review.md), Release Discipline): removing or renaming an export, narrowing a parameter, changing return semantics is a breaking change and a MAJOR bump — no exceptions. A new optional option or a new entry point is MINOR; a contract-preserving fix is PATCH.

```jsonc
"scripts": { "release": "bun run changeset version && bun run changeset publish" } // version + changelog derived from changesets
```
**Enforcement:** a changeset required on consumer-facing PRs; breaking changes gated behind a MAJOR bump per the git guide's release rules ([../git-and-code-review.md](../git-and-code-review.md)).

### 8.9 — Reproducible builds: same inputs, same artifact, no network beyond the lockfile.

**Reasoning, step by step:**
1. A reproducible build produces a byte-identical artifact from the same source. This is what makes a build cache trustworthy (a hit is provably the same output), what lets provenance (8.6) mean anything (the attestation is verifiable against a rebuild), and what rules out the whole class of "the artifact changed but the source didn't" mysteries.
2. So the build is hermetic with respect to the network: the only fetch is `bun install --frozen-lockfile` resolving the committed `bun.lock` (8.6), and nothing during compile reaches out for a floating version, a remote schema, or a "latest" anything. A build that phones the network is a build whose output depends on what the network said that minute.
3. The build environment is pinned, not inherited. The committed `.bun-version` ([./01-runtime-and-toolchain.md](./01-runtime-and-toolchain.md)) fixes the Bun version so the same Bun runs in CI and on every laptop, and archive metadata — the timestamps baked into the tarball and any zipped artifact — is normalized so two builds of the same commit differ in nothing (`tsc` emit is already timestamp-free, and `Bun.build` output is deterministic for a fixed input). Pin it in the committed `.bun-version`, where it is the source of truth, not in a CI script that drifts.

```jsonc
"scripts": { "ci": "bun install --frozen-lockfile && bun run build" } // hermetic: lockfile in, identical artifact out
```
**Enforcement:** `--frozen-lockfile` installs in CI, no network during compile; the Bun version pinned via the committed `.bun-version` ([./01-runtime-and-toolchain.md](./01-runtime-and-toolchain.md)).

## Cross-references

- The public-surface contract that 8.3 enforces mechanically (`index.ts` is the one front door): [../typescript/10-api-design.md](../typescript/10-api-design.md), 10.2; deliberate promotion of each public export: 10.3.
- The module-graph shape a library publishes (`sideEffects: false`, ESM, no internal barrels): [../typescript/12-module-organization.md](../typescript/12-module-organization.md).
- ESM-only packaging, `node:` builtins, and the `.bun-version` pin behind 8.9: [./01-runtime-and-toolchain.md](./01-runtime-and-toolchain.md).
- Dependencies as attack surface, lockfiles, and audit gates: BUN-4 in the [README](./README.md) and [../security.md](../security.md).
- Semver and breaking-change discipline behind 8.8: [../git-and-code-review.md](../git-and-code-review.md), Release Discipline.
- The JVM analog 8.5 mirrors (`binary-compatibility-validator` as a reviewed surface diff): [../kotlin-jvm/08-build-and-distribution.md](../kotlin-jvm/08-build-and-distribution.md), 8.5.
