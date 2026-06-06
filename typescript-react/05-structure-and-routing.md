# 05 — Structure and Routing

How a React app is carved into folders, where a feature's boundary sits, and how routes hang off that structure. The core guide already settled the module graph for TypeScript at large; this chapter ports those rules into the shapes a React codebase grows — feature folders with colocated components, hooks, and queries, and route components that stay thin. It governs where files live, not what they export.

## What good looks like

```text
src/
├── features/
│   └── booking/                 # one feature, everything it needs
│       ├── BookingCard.tsx      # PascalCase component file
│       ├── BookingCard.test.tsx # colocated test (5.5)
│       ├── BookingCard.stories.tsx
│       ├── useBooking.ts        # camelCase hook, lives with its feature
│       └── api/
│           └── bookSeat.ts      # query/mutation, colocated (5.1)
├── common/                      # cross-feature code only; kept thin
│   ├── Button.tsx
│   └── formatMoney.ts
└── routes/
    └── BookingRoute.tsx         # thin: composes feature, owns boundaries (5.4)
```

The tree groups by feature, not by kind (5.1, core [12.3](../typescript/12-module-organization.md)): a booking change edits one directory. Component files are PascalCase and folders camelCase (5.3), tests and stories sit beside the component they cover (5.5), and `common/` holds only what two features already share. Routes live apart and stay thin, composing the feature rather than reaching into it (5.4).

## Rules

### 5.1 — Everything a feature needs lives in its feature folder.

**Reasoning, step by step:**
1. A feature folder (`features/booking/`) holds the feature's components, hooks, API queries, types, and tests side by side. A change to booking opens one directory, and every file a reader needs is within arm's reach.
2. Kind folders (`components/`, `hooks/`, `queries/`) scatter one feature across the tree: a single booking change edits four directories, and each mixes unrelated features that share only a noun. This is the core guide's rule for grouping by feature ([12.3](../typescript/12-module-organization.md)), carried into React's component-and-hook shapes.
3. Colocation is what makes a feature deletable: removing booking is `rm -r features/booking/` rather than an archaeology dig across the tree.
4. `common/` exists only for code two or more features genuinely share. Promotion into it is deliberate, not anticipatory — a file moves to `common/` after the second consumer exists, never before. A `common/` filled with single-use code is just kind-folders wearing a different name.
5. A feature folder exposes no `index.ts` barrel, and neither does `common/`: callers import the file they mean (`import { BookingCard } from '../booking/BookingCard'`), not a re-export hub. This is core [12.4](../typescript/12-module-organization.md) ported to feature folders — internal barrels invite folder-level cycles and defeat tree-shaking, so the only barrel in a React app is the package root the core rule already permits. A feature is a folder convention, not a published surface.

**Enforcement:** review; a folder named for a layer rather than a feature, a `common/` entry with one importer, or an `index.ts` inside a feature or `common/` is a design discussion in the PR.

### 5.2 — No cross-feature imports except through `common/`.

**Reasoning, step by step:**
1. A feature boundary is a module boundary. `features/booking/` may import from itself and from `common/`, and nothing else; it must never reach into `features/checkout/`.
2. Direct feature-to-feature imports recreate the tangle the folder structure was meant to prevent: booking and checkout grow a hidden two-way dependency, and neither can be changed or deleted in isolation.
3. When two features need the same thing, that thing is not owned by either — it belongs in `common/`, where its shared status is explicit. This is the same pressure the core guide applies to cycles ([12.5](../typescript/12-module-organization.md)): coupling is a confession that a shared piece wants its own home.
4. Gate it mechanically. A lint path rule forbids `features/*/` importing a sibling `features/*/`, and `madge --circular` fails the build on any cycle, so the boundary holds without depending on reviewer vigilance.

**Enforcement:** an `import/no-restricted-paths` zone (from `eslint-plugin-import`) bars one feature from importing another — the actual enforcer of this rule, since a one-way feature-to-feature import is not a cycle; `madge --circular` (the core [12.5](../typescript/12-module-organization.md) gate, cross-referenced) covers the cycle case a path rule cannot.

### 5.3 — PascalCase component files; camelCase folders and non-component files.

**Reasoning, step by step:**
1. A file's name tells the reader its kind before they open it. `BookingCard.tsx` is a component; `useBooking.ts` is a hook; `bookSeat.ts` is a plain module. The casing carries the signal at a glance in the file tree.
2. React reverses the core guide's kebab-case file rule (core [2.4](../typescript/02-naming-conventions.md)) for component files. A component file's name is its import name — `import { BookingCard } from './BookingCard'` — and React components are PascalCase symbols, so the file is PascalCase too; the default-or-named export matches the filename exactly, and a symbol is always findable from its path and vice versa. Non-component files stay camelCase. This is a deliberate override, recorded in the README ledger, not an additive convention.
3. Folders and non-component files are camelCase (`features/booking/`, `useBooking.ts`, `api/bookSeat.ts`). The single capital-letter distinction is reserved for files that export a component, which keeps the one thing that is special looking special.
4. Hooks keep the `use` prefix that React's rules-of-hooks lint depends on; the prefix is not style but a contract the linter and the runtime both read.

**Enforcement:** a `unicorn/filename-case` rule (from `eslint-plugin-unicorn`) pinned to PascalCase for `*.tsx` components and camelCase elsewhere — the same rule core [2.4](../typescript/02-naming-conventions.md) sets to `kebabCase`, reconfigured here per the override above; review on the import-name-equals-file-name invariant.

### 5.4 — Keep route components thin.

**Reasoning, step by step:**
1. A route component is a seam, not a place for logic. Its job is to compose feature components, wire the URL params, and own the Suspense and error-boundary seam — then hand off. `BookingRoute.tsx` renders `<BookingCard>`; it does not contain booking logic.
2. The route does not hold the data. We use TanStack Query, not router loaders (REACT-3, [04](./04-data-fetching-and-forms.md)): components and hooks call `useQuery` where the data is used, and the cache — not the route — owns dedup, freshness, and the result. What the route may do at the URL seam is *prefetch* (`queryClient.prefetchQuery`) on navigation, warming the cache the feature's `useQuery` then reads. The route reads the params and decides which boundary wraps the feature; the feature fetches its own data.
3. This is also where `lazy()` boundaries belong. Code-splitting at the route means each route's feature bundle loads on navigation, and the split lives at the one edge that already maps to a URL. The Suspense and error-boundary placement that pairs with `lazy()` is covered in [08-react-performance.md](./08-react-performance.md).
4. A thin route stays readable and swappable: the feature can be reorganized without touching routing, and the route can be re-pointed without understanding the feature's internals.

**Enforcement:** review; a route component holding business logic or direct data manipulation beyond the load-and-compose seam is sent back.

### 5.5 — Colocate tests, stories, and styles with their component.

**Reasoning, step by step:**
1. What changes together lives together. `BookingCard.tsx`, `BookingCard.test.tsx`, `BookingCard.stories.tsx`, and any `BookingCard.module.css` sit in the same folder, so editing the component surfaces its test and story in the same view.
2. A parallel `__tests__/` or `stories/` tree forces the reader to navigate two structures at once and lets a test drift out of sync with its subject unnoticed. Colocation makes the staleness visible: a renamed component with an un-renamed neighbor is obvious in the diff. This extends the core guide's colocation rule ([12.3](../typescript/12-module-organization.md)) to React's stories and styles.
3. Deleting a component takes its test, story, and styles with it automatically, because they share a folder rather than living in a separate tree someone must remember to prune.
4. The naming mirror (`X.tsx`, `X.test.tsx`, `X.stories.tsx`) means tooling globs find them by pattern, and a reader infers the whole set from any one member.

**Enforcement:** review; Vitest (the runner, core [11.1](../typescript/11-testing.md)) and Storybook globs configured to discover `*.test.tsx` and `*.stories.tsx` beside source, not in a separate tree.

### 5.6 — One styling system per project.

**Reasoning, step by step:**
1. A project picks a single styling approach and uses it everywhere. Mixing Tailwind in one feature and CSS Modules in another doubles the mental model, the build config, and the ways a given style can be expressed or overridden.
2. Runtime CSS-in-JS is discouraged. Libraries that serialize styles during render add work to the hot path — every render may re-compute and re-inject CSS — and that cost compounds in lists and frequently-updating trees. The render path should paint, not assemble stylesheets.
3. Tailwind and CSS Modules are both acceptable: each resolves to static CSS at build time, so the render path carries no styling computation. The choice between them is a project-level taste call, not a per-feature one.
4. The decision is recorded in the project README so it is a settled fact a new contributor reads once, rather than a pattern they reverse-engineer from whichever file they opened first.

**Enforcement:** the chosen system is named in the README; review rejects a second styling mechanism and any runtime CSS-in-JS introduced without a charter amendment.

### 5.7 — GraphQL conventions: not adopted.

**Reasoning, step by step:**
1. This is a charter-ledger entry, not a live rule. dexpace does not use GraphQL today, so the codebase has no operations to name and no schema conventions to enforce.
2. The upstream guide specifies an `In{Feature}` naming scheme for operations colocated in a feature — `BookingQuery` defined in the booking feature. That convention is sound but addresses a problem this project does not have.
3. Recording the decision as a numbered rule keeps the ledger honest: a future reader sees that GraphQL was considered and deliberately left out, rather than wondering whether it was simply forgotten.
4. The trigger to revisit is concrete. If dexpace adopts GraphQL, this rule is reopened and the upstream `In{Feature}` operation-naming convention is the starting point for the replacement.

**Enforcement:** none today; this entry is the record that the question was settled, to be reopened only on a GraphQL adoption.

## Cross-references

- Grouping by feature, the thin shared layer, the package-root-only barrel that 5.1 ports to feature folders, and colocated tests this chapter ports to React: core guide [12.3](../typescript/12-module-organization.md), [12.4](../typescript/12-module-organization.md).
- Treating an import cycle as a bug, behind the `madge --circular` gate that backs 5.2: core guide [12.5](../typescript/12-module-organization.md).
- The kebab-case file rule that 5.3 overrides for PascalCase component files: core guide [02-naming-conventions.md](../typescript/02-naming-conventions.md).
- The TanStack Query cache and error-boundary layer behind the thin route's data seam (5.4): [04-data-fetching-and-forms.md](./04-data-fetching-and-forms.md).
- The colocated tests and stories (5.5) that this chapter places beside their component: [06-testing-react.md](./06-testing-react.md).
- The `lazy()` boundaries, Suspense, and error boundaries that pair with thin routes (5.4): [08-react-performance.md](./08-react-performance.md).
