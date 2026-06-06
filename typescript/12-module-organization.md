# 12 — Module Organization

How code is grouped into files, how files are grouped into folders, and how a package exposes itself to the world. The shape of the module graph is the first thing a new reader sees and the last thing a refactor can cheaply change, so it earns the same deliberation as the types. This chapter is about the boundaries between files; chapter 10 is about the API those boundaries publish.

## What good looks like

```text
packages/booking/
├── package.json          # "type": "module", "sideEffects": false
├── src/
│   ├── index.ts          # the only barrel: the package's public surface
│   └── features/
│       └── booking/      # one feature, all its parts together
│           ├── booking-service.ts        # public entry for the feature
│           ├── booking-service.test.ts   # colocated (ch. 11)
│           ├── seat-map.ts               # internal helper, no barrel
│           └── types.ts                  # feature-local types
└── shared/               # cross-feature primitives only; kept thin
    ├── invariant.ts
    ├── money.ts
    └── result.ts
```

```ts
// booking-service.ts — reads top-down: public API first, helpers below.
import { randomUUID } from 'node:crypto';

import { invariant } from '../../shared/invariant.js';
import type { Money } from '../../shared/money.js';
import type { Booking, Seat, SeatMap } from './types.js';

export async function bookSeat(map: SeatMap, seat: Seat, price: Money): Promise<Booking> {
  invariant(price.amount > 0, `price must be positive, got ${price.amount}`);
  const held = reserve(map, seat); // callee defined below (step-down, 05)
  return { id: randomUUID(), seat: held, price };
}

function reserve(map: SeatMap, seat: Seat): Seat {
  invariant(map.free.has(seat.id), `seat ${seat.id} already taken`);
  return { ...seat, status: 'held' };
}
```

The tree groups by feature, not by kind (12.3), with the lone barrel at the package root (12.4) and `shared/` kept thin. The file declares `bookSeat` before the `reserve` it calls, so the reader meets each function before its dependencies (the step-down rule, 05); `import type` keeps `Money` and `Seat` out of the runtime graph (12.2); the imports fall into three blank-line-separated groups — `node:`, external, internal (12.7); and the file covers one concept, the booking flow (12.9).

## Rules

### 12.1 — Ship ESM; reach for CJS only at a declared interop bridge.

**Reasoning, step by step:**
1. ESM is the language's module system and the platform's direction: static `import`/`export` give tools a graph they can analyze, tree-shake, and check for cycles before the code runs. CommonJS `require` is a runtime function call, so the graph is only knowable by executing the program.
2. Declare it once: `"type": "module"` in `package.json` makes every `.ts`/`.js` an ES module, and the ambiguity about what a bare `.js` means disappears.
3. Some dependency or host still speaks CJS. Confine the seam to a named bridge module (`vendor-bridge.ts`) that does the `createRequire` or dynamic `import()` dance, documents why, and re-exports a clean ESM surface. The rest of the codebase never sees the seam.
4. One file carries the impedance mismatch so the other three hundred stay pure ESM.

**Enforcement:** `"type": "module"` in `package.json`; review rejects `require`/`module.exports` outside a file whose name and header declare it a bridge.

### 12.2 — Write `import type` for every type-only import.

**Reasoning, step by step:**
1. With `verbatimModuleSyntax` on (chapter [01](./01-formatting-and-tooling.md)), the emitter does not guess which imports survive to runtime: a plain `import` is kept, an `import type` is erased. The source therefore states the runtime module graph exactly, with no compiler inference in the middle.
2. This makes a value import visually distinct from a type import. A reader scanning the header sees which modules actually load at runtime and which are pure compile-time references.
3. It also prevents an accidental runtime dependency: importing a type as a value would drag its whole module into the bundle, and a cycle that exists only in type-space would become a real one at runtime.
4. Prefer the top-level `import type { Foo }` form over inline `import { type Foo }` when every name in the statement is a type; the statement-level keyword is the louder signal.

**Enforcement:** `verbatimModuleSyntax: true` in `tsconfig.json` makes a missing `type` keyword a compile error; `@typescript-eslint/consistent-type-imports` autofixes the form.

### 12.3 — Group files by feature, not by technical kind.

**Reasoning, step by step:**
1. A feature folder (`features/booking/`) holds the feature's components, logic, types, and tests side by side. Everything a change to booking touches lives in one directory.
2. Kind folders (`services/`, `models/`, `components/`, `hooks/`) scatter one feature across the tree: a single booking change edits four directories, and each directory mixes unrelated features that share only a noun.
3. Change travels together. Feature-shaped layout means a diff is local, a code-owner is obvious, and deleting a feature is `rm -r` on one folder rather than an archaeology dig.
4. Cross-feature primitives that genuinely have no home — money types, a `Result` helper, an `invariant` — live in a sibling `shared/`. Keep it thin: every entry there is a thread pulling all features toward a common center, and a fat `shared/` is just kind-folders wearing a different name.

**Enforcement:** review; a folder named for a layer rather than a feature is a design discussion in the PR.

### 12.4 — Put barrels only at the package boundary.

**Reasoning, step by step:**
1. A barrel is an `index.ts` that re-exports its neighbors. At the package root it is exactly right: it is the one curated public surface, the contract chapter 10 describes, the single file a consumer imports from.
2. Internal barrels — an `index.ts` in every folder — are the opposite. They invite cycles: folder A's barrel imports B's barrel, which transitively imports A's, and a graph that was acyclic between files becomes a tangle between directories.
3. They also defeat tree-shaking and slow tooling: importing one symbol from a barrel makes the bundler and type-checker pull and analyze the whole re-export set, even when you wanted one function.
4. Internally, import the file you mean: `import { reserve } from './seat-map.js'`, not `from './index.js'`. The path names the dependency precisely, and the graph stays legible.

**Enforcement:** review; an `eslint-plugin-import/no-internal-modules` or `no-cycle` rule flags barrels below the package root.

### 12.5 — Treat an import cycle as a bug, not a style nit.

**Reasoning, step by step:**
1. Module A imports B and B imports A — directly or through a chain. The graph has a loop, which means there is no order in which the two modules are fully initialized before the other reads them.
2. ESM tolerates some cycles by leaving a binding temporarily undefined, so a cycle often *works by accident* until a reordering, a new top-level read, or a bundler change turns it into a `ReferenceError` or a silent `undefined`.
3. A cycle is a coupling confession: two modules that need each other are really one concept split across two files, or they share a third thing that wants its own module. The fix is to extract that shared piece, not to paper over the loop.
4. Gate it in CI. `madge --circular src` fails the build the moment a cycle appears, so the design pressure lands on the PR that introduced it, while the fix is still cheap.

**Enforcement:** `madge --circular` (or `eslint-plugin-import/no-cycle`) as a required CI check.

### 12.6 — Forbid import-time side effects; declare `"sideEffects": false`.

**Reasoning, step by step:**
1. Importing a module must be free: it may define functions, classes, and constants, and nothing else. No network call, no registry mutation, no console write, no clock read at the top level.
2. Import-time effects make load order load-bearing. The behavior of the program then depends on which file was imported first, which is invisible at every call site and brittle under any tree-shake or reorder.
3. They also break tree-shaking. A bundler must keep a module with side effects even when none of its exports are used, because dropping it might drop an effect something relied on.
4. Declaring `"sideEffects": false` in `package.json` is the explicit promise that the package has none, letting the bundler drop unused modules entirely. Code that genuinely needs an effect (registering a global, a polyfill bridge) exposes an explicit `init()` the caller invokes, and lists that one file in the `sideEffects` array.

**Enforcement:** `"sideEffects": false` in `package.json`; review rejects top-level statements that do work rather than declare it.

### 12.7 — Order imports in three groups: `node:` → external → internal.

**Reasoning, step by step:**
1. An import header is prose, and prose has paragraphs. Three groups, each separated by a blank line, let the eye find a given import by its kind without reading every line.
2. The order goes from most foundational to most local: built-ins first (`node:crypto`, `node:fs`), then third-party packages, then this project's own modules. A reader scans outward-in, the way dependencies actually nest.
3. Within each group, alphabetical order removes the last decision and makes merge conflicts in the import block trivial to resolve.
4. The `node:` protocol prefix on built-ins is mandatory, not optional: it disambiguates a core module from a same-named package and lets the first group be matched by a simple rule.

**Enforcement:** `eslint-plugin-import/order` (or the `gts` baseline) with three `groups`, `newlines-between: always`, and alphabetization; autofixed on save.

### 12.8 — Keep relative imports shallow: at most two levels up.

**Reasoning, step by step:**
1. A relative import is a measure of distance between two files. `./` and `../` are fine; `../../foo.js` is the edge; `../../../foo.js` means the importer and the imported are far apart in a tree that claims they are related.
2. Deep `../../..` chains are also fragile: they break the moment a file moves, and they read as noise that hides which module is actually being named.
3. The chain is a symptom, and the cure is structural. Either the two files belong in the same feature folder (move them together) or the imported thing is shared and belongs in `shared/` (promote it). Restructure the tree so the distance shrinks.
4. Do not paper over depth with a path alias. An alias hides the smell without fixing the layout, and now the real coupling is invisible behind a tidy-looking `@/foo`. Aliases earn their keep for a stable package root, not for escaping a tree that grew wrong.

**Enforcement:** `eslint-plugin-import/no-relative-parent-imports` tuned to a two-level depth, or review on any import with three or more `../`.

### 12.9 — Write each module so it reads top-down as a story.

**Reasoning, step by step:**
1. A file has one concept and presents it like an article: the public, most abstract entry point first, then the helpers it leans on, in decreasing order of abstraction. A reader who stops after the first export already understands what the file is for.
2. This is the step-down rule (chapter [05](./05-functions.md)) applied at file scope: callers sit above callees, so each name is introduced before it is used. Top-level `function` declarations make the order legal through hoisting; a file of `const fn = () =>` would force callees first and invert the narrative.
3. One concept per file is what makes the story coherent. When a file starts describing two things — a service *and* an unrelated cache, a parser *and* a formatter — that is two articles bound into one pamphlet, and each wants its own file.
4. Around 300 lines per *file* (distinct from the 70-line *function* cap of 05) is the consider-splitting signal, not a hard cap. It is the point at which a file usually stopped being one story; check whether a second concept has crept in, and if so, give it its own module.

**Enforcement:** review against the step-down rule; `max-lines` set to a warn at ~300 as a prompt to look, not a gate.

## Cross-references

- `verbatimModuleSyntax` and the tsconfig flags behind `import type`: chapter [01](./01-formatting-and-tooling.md).
- The step-down rule (callers above callees) that 12.9 extends to file scope: chapter [05](./05-functions.md).
- Named exports only and the `index.ts` public-surface contract that 12.4 serves: chapter 10.
- Colocated `*.test.ts` files in the feature folder: chapter 11.
