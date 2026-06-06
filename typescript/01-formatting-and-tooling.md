# 01 — Formatting and Tooling

Formatting is not a matter of taste. We delegate it to one tool, `gts`, and spend our judgment on things that matter. This chapter fixes the toolchain, the compiler flags, and the lint shape every dexpace TypeScript project inherits before a line of domain code is written. Everything here is mechanical and enforced; the later chapters cover the decisions a tool cannot make for you.

## What good looks like

`bunx gts init` scaffolds the gts defaults; replace the generated configs with the dexpace overlay below. Pin the toolchain and layer the overlay on top:

```jsonc
// package.json — pin Bun out-of-band with a committed `.bun-version` file, not a field here
{
  "type": "module",
  "devDependencies": {"gts": "^7", "typescript": "^5.8", "typescript-eslint": "^8"},
  "scripts": {
    "lint": "gts lint",
    "fix": "gts fix",
    "compile": "tsc --noEmit",
    "test": "bun test"
  }
}
```

```jsonc
// tsconfig.json — gts base, plus exactly the six strictness flags
{
  "extends": "./node_modules/gts/tsconfig-google.json",
  "compilerOptions": {
    // platform module settings (module/moduleResolution, lib additions) live with the
    // runtime guide, not here — see the runtime-and-toolchain chapter
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true
  },
  "include": ["src/**/*.ts"]
}
```

```js
// eslint.config.js — one overlay on top of gts
import tseslint from 'typescript-eslint';
import gts from 'gts';

export default tseslint.config(
  ...gts,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    // projectService gives type-checked rules a program without hand-maintained parserOptions.project
    languageOptions: {parserOptions: {projectService: true}},
    rules: {
      'max-lines-per-function': ['error', {max: 70, skipComments: true, skipBlankLines: false}],
      'max-depth': ['error', 3],
      'max-params': ['error', 3],
    },
  },
);
```

This demonstrates 1.5 (`bun install` against a committed lockfile, Bun pinned via `.bun-version`) and 1.6 (a single overlay layering the type-checked tiers on gts). The generated `.prettierrc.js` re-exports the gts defaults verbatim; per 1.2 we never edit it.

## Rules

### 1.1 — gts is the toolchain; no standalone Prettier or ESLint config.

**Reasoning, step by step:**
1. gts bundles Prettier, ESLint, and a TypeScript base config behind one opinionated dependency, so the formatter and linter cannot drift apart between projects.
2. Bootstrap with `bunx gts init`; it scaffolds the config files and scripts so every package starts identical.
3. `gts fix` runs locally and auto-fixes; `gts lint` runs in CI as the gate. A developer never argues with the output, and a reviewer never comments on it.
4. A hand-rolled `.prettierrc` or a parallel `.eslintrc` re-opens the argument gts exists to close. Delete them; extend gts instead.

**Enforcement:** `gts lint` in CI; absence of standalone formatter configs checked in review.

### 1.2 — Prettier defaults, as shipped by gts, are final.

**Reasoning, step by step:**
1. Quote style, semicolons, print width, and trailing commas carry no correctness weight, so the only cost they have is the time spent debating them.
2. gts ships Prettier with Google's defaults already chosen. Adopting them wholesale means zero per-project negotiation.
3. Any override — `printWidth`, `singleQuote`, anything — forks your formatting from every other dexpace repo and from the upstream baseline you inherit fixes from.

**Enforcement:** `gts lint` fails on unformatted code; review rejects any Prettier override.

### 1.3 — tsconfig extends the gts base and adds exactly six flags.

**Reasoning, step by step:**
1. The gts base sets `strict` and the sensible defaults, but leaves off six strictness flags that close real holes in the type system. Add precisely these six, no more. (Platform settings — the `module`/`moduleResolution` pair the runtime dictates, and `lib` additions like `esnext.disposable` for ch13's `using` — are a separate matter, distinct from the six strictness flags; they live with the runtime guide, not in core.)
2. Each flag earns its place by turning a class of latent runtime bug into a compile error.
3. Adding strictness flags beyond this set is a guide change, not a per-project choice; the set is uniform so a file behaves identically in every repo.

| Flag | What it buys |
|---|---|
| `noUncheckedIndexedAccess` | Indexing an array or record yields `T \| undefined`, forcing the bounds check you would otherwise forget. |
| `exactOptionalPropertyTypes` | An absent property is not the same as one explicitly set to `undefined`; the distinction stops bugs at object boundaries. |
| `noImplicitOverride` | Overriding a base member requires the `override` keyword, so a renamed base method becomes an error instead of a silent new method. |
| `isolatedModules` | Each file must be transpilable alone, keeping the source safe for single-file transpilers like esbuild and SWC. |
| `verbatimModuleSyntax` | Type-only imports must say `import type`, so the emitter never guesses whether an import survives to runtime. |
| `erasableSyntaxOnly` | Type syntax may never emit runtime code, banning `enum`, namespaces with runtime code, parameter properties, and `import =`. |

**Enforcement:** the six `compilerOptions` in `tsconfig.json`; `tsc --noEmit` in CI.

### 1.4 — TypeScript 5.8 or newer; track the latest stable.

**Reasoning, step by step:**
1. `erasableSyntaxOnly` landed in 5.8, and rule 1.3 makes it mandatory, so 5.8 is the hard floor.
2. Newer compilers ship better inference and faster checks; staying current keeps the type system working for you rather than against you.
3. Pin a caret range (`^5.8.0`) and bump deliberately, as a reviewed change rather than an ambient surprise.

**Enforcement:** the `typescript` range in `package.json`; review on every bump.

### 1.5 — `bun install` against a committed `bun.lock`; `--frozen-lockfile` in CI.

**Reasoning, step by step:**
1. `bun install` records resolved versions in `bun.lock` (a text lockfile since Bun 1.2, superseding the old binary `bun.lockb`). Commit it: it pins the whole dependency graph so every machine resolves the same versions.
2. In CI, run `bun install --frozen-lockfile` (equivalently `bun ci`). Frozen mode installs exactly what the lockfile records and fails the build if `package.json` and `bun.lock` disagree — it never silently updates the lockfile, so an unreviewed dependency drift cannot slip through.
3. Be honest about isolation: by default `bun install` hoists into a flat `node_modules` (the npm/yarn layout), so a package can resolve a transitive dependency it never declared. The lockfile plus frozen mode guarantee *reproducibility* — the same versions on every machine — not pnpm-style strictness. Where strict per-package isolation matters, opt into Bun's isolated linker (`--linker isolated`, symlinks under `node_modules/.bun/`), which is the monorepo default; otherwise an undeclared import is caught by `tsc --noEmit` and review, not by the layout.
4. Pin the Bun version out-of-band with a committed `.bun-version` file (Bun has no LTS line), so every developer and the CI runner use one resolver and one runtime.

**Enforcement:** committed `bun.lock` and `.bun-version`; `bun install --frozen-lockfile` (`bun ci`) in CI.

### 1.6 — One ESLint overlay file, extending gts.

**Reasoning, step by step:**
1. Layer typescript-eslint's `strict-type-checked` and `stylistic-type-checked` on top of gts; the type-checked tiers catch bugs a syntax-only linter cannot see.
2. Keep the whole overlay in a single `eslint.config.js`. One file is the complete, greppable diff between dexpace and upstream gts.
3. Every rule you add must trace to a chapter of this guide. A rule with no chapter behind it is taste smuggled past the gate, and the next maintainer cannot tell why it is there.

**Enforcement:** the single `eslint.config.js`; review checks each added rule cites a chapter.

### 1.7 — `max-lines-per-function` is 70.

**Reasoning, step by step:**
1. A function you cannot see at once costs context on every read. Seventy lines is the hard ceiling, set deliberately at Go's level rather than scaled down for TypeScript.
2. Configure it with `skipComments: true` so documentation never pushes a function over; blank lines inside the body do count, because vertical sprawl is the thing being capped.
3. Approaching the cap is the signal to decompose — extract a helper, lift a pipeline stage, name an intermediate. The cap is a floor under readability, not a number to game.

**Enforcement:** `max-lines-per-function` in `eslint.config.js`. See [05-functions.md](./05-functions.md).

### 1.8 — `max-depth` is 3 and `max-params` is 3.

**Reasoning, step by step:**
1. Nesting past three levels hides the control flow; aim for two and let guard clauses keep the happy path flush left.
2. A function reaching for a fourth parameter is usually doing too much, or its arguments belong together. Three or more parameters means an options object, which also makes call sites self-documenting.
3. Both caps push the same way: toward small, flat, single-purpose functions whose shape you can read at a glance.

**Enforcement:** `max-depth` and `max-params` in `eslint.config.js`. See [05-functions.md](./05-functions.md).

### 1.9 — Pre-commit runs `gts lint` and `tsc --noEmit`.

**Reasoning, step by step:**
1. Broken style or a type error that reaches `main` blocks everyone who pulls it, turning one person's slip into the team's problem.
2. A pre-commit hook running `gts lint` and `tsc --noEmit` catches both before the commit object exists, so history stays green.
3. CI runs the same two commands as the authority. The hook is the fast local mirror, never a substitute for the gate.

**Enforcement:** pre-commit hook plus the identical CI step.

### 1.10 — Every `eslint-disable` carries a same-line reason.

**Reasoning, step by step:**
1. A suppression with no explanation is a black box; the next reader cannot tell whether it is load-bearing or stale debt left behind.
2. Require the reason inline — `// eslint-disable-next-line rule-name -- why this is safe here` — so the justification travels with the code it excuses.
3. A disable you cannot justify in one line is a disable you should not write. Fix the code instead.

**Enforcement:** `@eslint-community/eslint-comments/require-description` in the overlay; review on every suppression.

## Cross-references

- The 70-line cap, decomposition, and options objects: [05-functions.md](./05-functions.md).
- Why `enum` and parameter properties are banned (the `erasableSyntaxOnly` consequences): [03-the-type-system.md](./03-the-type-system.md).
- `import type` discipline that `verbatimModuleSyntax` enforces: [12-module-organization.md](./12-module-organization.md).
- Naming and file casing the formatter leaves untouched: [02-naming-conventions.md](./02-naming-conventions.md).
