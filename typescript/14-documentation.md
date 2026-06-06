# 14 — Documentation

TSDoc is part of the public API; it ships in the `.d.ts`, surfaces in the IDE, and feeds the docs site. This chapter covers what to document, what to leave to the signature, and how to keep both from drifting apart. The rule is narrow: a comment exists to carry intent the code cannot, and it is wrong the moment it stops matching the code.

## What good looks like

A fully documented public function: a why-summary, `@param` only where the name and type fall short, the failure modes, and one compilable call site.

```ts
/**
 * Reserves `quantity` units of `sku` against the caller's cart, holding them
 * for {@link RESERVATION_TTL_MS} so checkout can complete without a race.
 *
 * The hold is best-effort: stock can still sell out between reservation and
 * capture, which is why {@link captureReservation} re-checks. Prefer this over
 * decrementing stock directly — a raw decrement leaks units when a cart is
 * abandoned.
 *
 * @param quantity - units to hold; must be a positive integer (whole units, not grams)
 * @throws {@link OutOfStockError} fewer than `quantity` units are available — show the back-in-stock prompt
 * @throws {@link SkuRetiredError} the SKU is no longer sold — drop it from the cart
 * @example
 * const cartId = await getCartId(userId);
 * const hold = await reserveStock(cartId, "SKU-1024", 2);
 * await captureReservation(hold.id);
 */
export function reserveStock(
  cartId: CartId,
  sku: Sku,
  quantity: number,
): Promise<Reservation>;

// Bad sibling: restates the types, narrates the mechanics, hides the throws.
/** Reserves stock. Takes a cartId, a sku, and a quantity, and returns a reservation. */
export function reserveStockBad(cartId: CartId, sku: Sku, quantity: number): Promise<Reservation>;
```

The good version earns every line: the summary says *why* a caller reaches for it over a raw decrement (14.3), `@param` appears only on `quantity` because that is the one argument whose contract — positive, integer, whole units — the type cannot state (14.2), both failure modes are documented with what the caller should do (14.5), and the `@example` is a real call site that compiles (14.4). The bad sibling re-types the signature in prose, narrates what the diff already shows, and says nothing about the two ways it can throw.

## Rules

### 14.1 — Use TSDoc syntax for every public API.

Public symbols — anything exported from a package's `index.ts` (chapter 10) — carry a TSDoc comment, the `/** … */` block with `@`-tags. It is the format `tsc`, `typedoc`, and every editor parse for hover-cards and the generated reference; a plain `//` comment or a bare block is invisible to all of them.

**Reasoning, step by step:**
1. TSDoc is the one comment dialect the toolchain reads. `tsc` lifts it into the emitted `.d.ts`, so a consumer sees it on hover even without your source. `typedoc` renders it into the docs site. A non-TSDoc comment reaches none of these surfaces.
2. The audience for a public symbol is someone who will never open its source — they get the signature and the hover-card. If the contract is not in the TSDoc, it does not exist for them.
3. Private and module-internal symbols are documented only when the name does not already carry the meaning; do not paste a TSDoc block on `const double = (n: number) => n * 2`.

**Enforcement:** lint; ESLint with the TSDoc plugin validates tag syntax, review confirms every export from a barrel carries a block.

### 14.2 — Never restate the type in prose.

The signature is the source of truth for shape; prose that repeats it is noise that can fall out of sync. A TSDoc line earns its place by adding what the type cannot say: intent, constraints, units, valid ranges, the meaning of a sentinel.

**Reasoning, step by step:**
1. `@param userId - the user id` adds nothing to `userId: UserId`; the name and the branded type already say it. The comment is pure restatement, and now there are two places to update on a rename.
2. The type system cannot express "in `[0, 1]`", "cents, not dollars", "must be sorted ascending", or "empty string means anonymous". Those are exactly what `@param` and the summary are for.
3. Follow [the Google guidance on comments and documentation](https://google.github.io/styleguide/tsguide.html#comments-documentation): omit a `@param` entirely when the name and type are self-documenting. A signature where every parameter has a redundant `@param` trains readers to skip them all, hiding the one that matters.

**Enforcement:** review; reject `@param`/`@returns` lines that only echo the name or type, keep the ones that add a constraint, unit, or sentinel.

### 14.3 — Make comments say why, not what.

Root rule 7: always say why. The diff and the signature already show *what* the code does; a comment that narrates the mechanics duplicates them and rots when they change. The comment's job is the reasoning a future reader cannot reconstruct from the code alone.

**Reasoning, step by step:**
1. `// increment the retry counter` above `retries++` is dead weight — the line says that already. `// back off only on 5xx; 4xx will never succeed on retry` carries a decision the code cannot.
2. The *what* is verifiable by reading the code; the *why* is not. Spend comment budget on the part that is invisible: the constraint that forced this shape, the bug this guards against, the spec paragraph behind the magic number.
3. TODOs are a why with an owner and a date: `// TODO(omar 2026-08-01): drop once API v3 is the default`. An ownerless TODO is a wish nobody is accountable for.

**Enforcement:** review; delete mechanics-narrating comments on sight, keep the ones a reviewer could not have inferred from the diff.

### 14.4 — Put an `@example` on every non-obvious public API.

If correct use is not self-evident from the signature, the TSDoc carries one canonical call site under `@example`. It is the fastest thing a caller reads, and it doubles as a compile-checked contract when the docs build typechecks its fences.

**Reasoning, step by step:**
1. A signature shows the shape of a call but not the *shape of usage* — the order of operations, what to do with the return, which helper pairs with it. One worked example answers all three faster than prose.
2. Keep it to a single, realistic call site, not a tour of every option. The example is a starting point a reader adapts, so it must compile against the current API; a stale example is worse than none (14.8).
3. Obvious one-liners — a pure `clamp(value, min, max)` — do not need an example; the signature is the example. Reserve the tag for APIs with non-obvious sequencing or setup.

**Enforcement:** review; non-obvious exports carry a compilable `@example`, and the docs build typechecks example fences so they cannot drift.

### 14.5 — Document every failure mode with `@throws`.

Every error a public function can throw and a caller might reasonably catch gets a `@throws` tag naming the error type and what the caller should do about it. A thrown error is invisible in the return type, so without the tag the caller discovers the failure in production.

**Reasoning, step by step:**
1. `function chargeCard(...): Promise<Receipt>` admits nothing about `CardDeclinedError`. The signature documents the success type; `@throws` is the only place the failure contract can live — exactly the point made in [chapter 08, §8.10](./08-error-handling.md) and reinforced by API design in [chapter 10, §10.8](./10-api-design.md) (its documentation half — §10.8 also mandates cancellation, which 14.5 does not touch).
2. List only the errors a caller would act on, most-likely first, each with a recovery hint: `@throws {@link RateLimitedError} too many requests — retry after the \`retryAfter\` header`. Do not catalogue every `Error` that could theoretically escape.
3. Where failure is expected rather than exceptional, prefer a `Result` return (chapter 08) over `@throws`; the failure then lives in the type and cannot drift from the implementation. Use `@throws` for the genuinely exceptional path.

**Enforcement:** review; throwing publics carry a `@throws` per caught-able error or use a `Result` signature, checked against the implementation.

### 14.6 — Give every package a README that reaches first success in 30 seconds.

Each package ships a README whose top gets a new engineer from zero to one working call without reading source: what it is in a sentence, the install line, one runnable example, then links deeper. The first screen is the whole job; everything else is reference.

**Reasoning, step by step:**
1. A reader arrives needing one thing — to use the package — and decides in seconds whether the docs will get them there. Lead with install and a single copy-pasteable example, not architecture or history.
2. The 30-second target forces ruthless ordering: the one canonical example first, the API surface and edge cases linked, not inlined. If first success needs a paragraph of prose, the API is too hard to start with — fix the API.
3. The README is the package's front door; the per-symbol contracts live in TSDoc and the generated reference. The README links to them rather than restating them (14.7).

**Enforcement:** review; every publishable package has a README whose first screen is name, install, one example, and links — verified when the package is added.

### 14.7 — Link, don't duplicate.

Documentation states each fact in exactly one place and links to it from everywhere else. Repeat nothing already written in this styleguide, a sibling chapter, or another symbol's TSDoc; use `{@link Symbol}` and relative paths. Duplicated prose forks the moment one copy changes and silently rots.

**Reasoning, step by step:**
1. Two copies of the same explanation are two things to update and one you will forget. The forgotten copy becomes a confident lie — the worst kind of documentation.
2. TSDoc resolves `{@link reserveStock}` and `{@link Cart.total}` into navigable, rename-safe references; the IDE updates the link target under a refactor. A bare "see reserveStock" is dead text that breaks silently.
3. Prefer one authoritative home per fact: the contract on the symbol, the conceptual overview in the README, the rule in this guide. Everything else points at it. This is the documentation form of the zero-debt stance — one source, no copies to drift.

**Enforcement:** review; replace copied paragraphs with a link, prefer `{@link}` over prose cross-references so renames keep the docs honest.

### 14.8 — Treat stale comments as debt and fix them in the same commit.

A comment that no longer matches the code is worse than no comment: it actively misleads. When a change touches code, the surrounding TSDoc and comments are part of that change — update them or delete them in the same commit, never "later".

**Reasoning, step by step:**
1. A reader trusts a comment more than they trust their own reading of the code; a wrong comment spends that trust to point them at the wrong conclusion. Deleting an outdated comment strictly improves the file.
2. Documentation drifts only when an edit changes the code and leaves the prose behind. Closing that gap is free at edit time and expensive forever after, once nobody remembers which of the two is right.
3. This is rule 12 — zero technical debt — applied to prose. Stale documentation is debt that compounds the way stale code does, and like all debt the second chance to pay it may never come. The fix is the discipline of one commit: code and its documentation move together, or not at all.

**Enforcement:** review; a diff that changes behaviour and leaves its TSDoc or comments stale does not merge — update or delete in the same commit.

## Cross-references

- Branded types and the type-as-documentation contract that lets 14.2 stay terse: [chapter 03](./03-the-type-system.md).
- `@throws` and the `Result`-versus-throw decision behind 14.5: [chapter 08, §8.10](./08-error-handling.md).
- Naming as the first line of documentation: [chapter 02](./02-naming-conventions.md).
- What to expose before documenting it, and `@deprecated` with semver: [chapter 10, §10.9](./10-api-design.md).
- The package boundary a README documents: [chapter 12](./12-module-organization.md).
