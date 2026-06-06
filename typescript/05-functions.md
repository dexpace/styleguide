# 05 — Functions

Functions are the unit of reasoning. Every rule here keeps that unit small, named, and honest: one job, one level of abstraction, asserted at both ends, with the happy path flush left. The cap is the floor of the discipline; aim far below it.

## What good looks like

```ts
function settleInvoice(invoice: Invoice, options: SettleOptions): Receipt {
  invariant(invoice.lines.length > 0, 'invoice has no lines');
  invariant(invoice.status === 'open', `invoice ${invoice.id} is not open`);

  const subtotal = sumLines(invoice.lines);
  const tax = taxFor(subtotal, options.jurisdiction);
  const total = cents(subtotal + tax);

  invariant(total >= subtotal, 'total cannot shrink below subtotal');
  invariant(total >= tax, 'total cannot shrink below tax');

  return {invoiceId: invoice.id, subtotal, tax, total, paidAt: options.now};
}

function sumLines(lines: readonly Line[]): Cents {
  let total = 0;
  for (const line of lines) {
    invariant(line.amount >= 0, 'line amount is negative');
    total += line.amount * line.quantity;
  }
  return cents(total);
}

function taxFor(subtotal: Cents, jurisdiction: Jurisdiction): Cents {
  const rate = RATES[jurisdiction];
  invariant(rate !== undefined, `no rate for ${jurisdiction}`);
  return cents(Math.round(subtotal * rate));
}
```

`Cents` is minted in one place — a parse-constructor that validates, then performs the lone sanctioned `as` (3.9). Nothing else in the chapter casts to it:

```ts
function cents(n: number): Cents {
  invariant(Number.isInteger(n) && n >= 0, 'cents must be a non-negative integer');
  return n as Cents; // sanctioned: validated immediately above (3.4, 3.9)
}
```

The public function sits above the two helpers it calls, so the file reads top-down (5.4). Guard clauses assert the preconditions and leave the happy path flush left (5.3). The single object parameter keeps the call site legible (5.5), and a postcondition pair checks the sum is approached from both of its parts before it returns (5.7). Each function holds one level of abstraction — `settleInvoice` orchestrates named steps and touches no arithmetic itself (5.2). The helpers reach a `Cents` only through the `cents` mint, the single parse-constructor that validates and casts (3.9); `taxFor` guards the rate lookup because `noUncheckedIndexedAccess` (1.3) types the miss as `undefined`, so the flag forces the guard.

## Rules

### 5.1 — 70 lines. Hard cap. Aim for 10–30.

**Reasoning, step by step:**
1. A function you can hold in your head fits on a screen. Past ~30 lines it stops being one idea and becomes a list of them.
2. The cap is 70 lines, lint-enforced — blank lines counted, comments excluded. It is set at Go's level deliberately, not scaled down for TypeScript; hitting it is already a smell.
3. The test is linguistic: if the honest one-sentence summary of a function needs an "and," it is two functions wearing one name. Split on the "and."
4. Every callable counts — top-level declarations, methods, and arrow callbacks alike. A 70-line arrow is the same defect as a 70-line method.

Worked example: a `handleRequest` that parses input, *and* authorizes, *and* writes the database, *and* formats the response is four functions. Extract `parseRequest`, `authorize`, `persist`, `formatResponse`; the orchestrator drops to a dozen readable lines.

**Enforcement:** ESLint `max-lines-per-function: ['error', {max: 70, skipComments: true, skipBlankLines: false}]`.

### 5.2 — One level of abstraction per function.

**Reasoning, step by step:**
1. A function either *orchestrates* named steps or *does* primitive work. Mixing the two forces the reader to shift altitude mid-body, re-deriving what a high-level call means while parsing low-level arithmetic.
2. Pick the altitude from the name. `settleInvoice` names a workflow, so its body is calls; `sumLines` names a computation, so its body is a loop.
3. When a high-level body sprouts a tight loop or a bit-twiddle, that fragment wants a name. Extract it; the helper's name documents the step you removed.

Worked example: in the exemplar, `settleInvoice` never multiplies or rounds — it delegates to `sumLines` and `taxFor`. Inlining either would drop two levels of abstraction into one body and the workflow would vanish into the arithmetic.

**Enforcement:** review; surfaced by 5.1's line cap and `max-depth` (a body mixing levels nests deeper).

### 5.3 — Guard clauses first; happy path flush left.

**Reasoning, step by step:**
1. Handle the exceptional at the top and return early. What survives the guards is the main case, and it reads as a straight column down the left margin.
2. Early return beats nesting: every `if` you invert into a guard is an indentation level the happy path never pays for. An `else` is usually a guard you failed to take.
3. Deep nesting hides the spine of a function inside pyramids of braces. Flat code has one obvious path.

Worked example:

```ts
function priceFor(sku: Sku, cart: Cart): Cents {
  const item = cart.items.get(sku);
  if (item === undefined || item.quantity === 0) return cents(0); // guard
  return cents(item.unitPrice * item.quantity);                   // happy path, flush left
}
```

**Enforcement:** ESLint `max-depth: ['error', 3]`; aim for ≤ 2.

### 5.4 — Step-down rule: callers above callees.

**Reasoning, step by step:**
1. Code is read top-down. Put each function above the ones it calls, so a reader meets the intent before the detail and can stop reading once the altitude is low enough.
2. This requires top-level `function` declarations: hoisting makes the caller-above-callee order legal even though the callee is defined later. A `const fn = () =>` is not hoisted and would force callees first, inverting the file.
3. Named declarations also survive in stack traces — an anonymous arrow assigned to a `const` shows up far less reliably across tooling than a real function name.
4. Reserve arrows for callbacks, where they are passed inline and never need to read top-down: `items.map(toRow)`, `promise.then(handleResult)`.

Worked example: the exemplar lists `settleInvoice`, then `sumLines`, then `taxFor` — the order a reader descends. As `const` arrows the leaves would have to come first and the entry point last.

**Enforcement:** review; `func-style: ['error', 'declaration', {allowArrowFunctions: true}]` keeps named functions as declarations while permitting arrow callbacks.

### 5.5 — Options object for ≥ 3 parameters or any boolean.

**Reasoning, step by step:**
1. A call site must read without opening the signature. `connect(host, 443, 10_000, true, false)` is a row of unlabeled values; `connect(host, {port: 443, timeoutMs: 10_000, tls: true, redirect: false})` says what each means.
2. A bare boolean at a call site is an unlabeled switch — `setVisible(true)` is fine because the name carries it, but `render(true)` is a coin flip until you read the parameter name. An options object turns the flag into a labeled key: `render({preview: true})`.
3. Three or more positional parameters also invite transposition between same-typed arguments; named keys make order irrelevant and additions backward-compatible.

Worked example: `SettleOptions` in the exemplar bundles `jurisdiction` and `now` so `settleInvoice(invoice, {jurisdiction, now})` is self-describing, where `settleInvoice(invoice, jurisdiction, now)` would not be.

**Enforcement:** ESLint `max-params: ['error', 3]`; booleans go in an options object regardless of count.

### 5.6 — The `invariant` helper.

**Reasoning, step by step:**
1. An assertion that also *narrows* is worth more than one that only checks: a runtime guarantee that teaches the compiler. TypeScript's assertion functions do exactly this — after `invariant(x !== undefined, …)`, the type of `x` loses `undefined` for the rest of the scope.
2. Define it once, project-wide, and use it everywhere. A bespoke `if (!x) throw` at each site neither narrows the type nor reads as an assertion.
3. The thrown error is its own class so callers can distinguish a broken invariant (a programmer error) from an operational failure they might recover from.

Define it in full once, project-wide:

```ts
export class InvariantViolation extends Error {
  override readonly name = 'InvariantViolation';
}

export function invariant(cond: unknown, msg: string): asserts cond {
  if (!cond) throw new InvariantViolation(msg);
}
```

**Enforcement:** review; the helper is the single sanctioned assertion primitive (see [08-error-handling.md](./08-error-handling.md)).

### 5.7 — Assert aggressively: 2+ per function on average.

**Reasoning, step by step:**
1. Assertions are executable preconditions and postconditions. Check arguments at entry and results before return; the average across a module should land at two or more per function.
2. Assert *positive and negative space*: not just that the expected holds, but that the impossible is absent — a total must never fall below its subtotal *and* never fall below its tax.
3. Pair assertions: verify one property two independent ways, so that when the two derivations disagree a bug surfaces at the assertion instead of three layers downstream.

Worked example: `settleInvoice` opens with two preconditions (non-empty lines, open status) and closes with a postcondition pair where the sum is approached from both of its parts: `total >= subtotal` and `total >= tax` together rule out a negative tax or a negative subtotal.

**Enforcement:** review and code review; density is a target, not a lint rule. The `invariant` helper (5.6) is the mechanism.

### 5.8 — Pure by default.

**Reasoning, step by step:**
1. A pure function (same input, same output, no observable effect) is trivially testable, cacheable, and safe to reorder. Make it the default and the testable surface of the codebase grows. Side effects live at the edges: a thin shell does the I/O and calls a pure core, the same instinct as root rule 6 (transform, don't mutate) applied to data.
2. Name the effect so the signature tells the truth. A side-effecting function carries an effect verb (`writeLedger`, `emitEvent`, `fetchUser`); a name like `data` or `process` hiding a database write is a lie at the call site.

Worked example:

```ts
async function recordSale(sale: Sale): Promise<Receipt> {
  const receipt = settleInvoice(sale.invoice, sale.options); // pure core
  await writeLedger(receipt);                                // effect, at the edge
  return receipt;
}
```

**Enforcement:** review; effect verbs are checked at code review (naming taxonomy in [02-naming-conventions.md](./02-naming-conventions.md)).

### 5.9 — No control-flag parameters.

**Reasoning, step by step:**
1. A boolean parameter that forks the body into two paths means two functions are hiding in one. The flag is a branch the caller is forced to understand. Split them: `getItems({recursive: true})` becomes `getItemsRecursive()` and `getItemsShallow()`, each name stating what it does, neither body carrying a branch the other doesn't need.
2. This differs from 5.5: 5.5 labels a *configuration* boolean an options object can hold; 5.9 forbids a *behaviour-forking* boolean entirely. If the flag selects between two algorithms, no options object redeems it.

Worked example: `parse(input, true /* strict */)` hides two parsers. Make them `parseStrict(input)` and `parseLenient(input)`; the shared work goes in a private helper both call.

**Enforcement:** review; surfaces as a code-review rejection of behaviour-selecting booleans.

### 5.10 — Overloads only when the return type depends on input shape.

**Reasoning, step by step:**
1. Every overload signature is another contract to keep honest, and the implementation signature is invisible to callers. Pay that cost only when nothing else expresses the API.
2. The one case that earns it: the return type genuinely varies with the *shape* of the input — `createElement('input')` returns `HTMLInputElement`, `createElement('div')` returns `HTMLDivElement`, and a union return would force every caller to narrow. When the return type is stable instead, a union parameter or a generic says the same thing with one signature: `area(shape: Circle | Square): number` needs no overloads.

Worked example: prefer `function format(value: string | number): string` over two overloads — the return is always `string`, so the union parameter is the honest, single-signature form.

**Enforcement:** ESLint `@typescript-eslint/unified-signatures` (collapses overloads a union would cover); review for return-type dependence.

### 5.11 — Explicit return types on every exported function.

**Reasoning, step by step:**
1. A public contract is written, not derived. Annotating the return type makes the boundary a deliberate decision and stops an accidental internal change from silently widening what callers receive.
2. Inference is for locals, where the definition is right there and a wrong inference is caught immediately; across a module boundary it is invisible. The annotation is also the first error site: if the body stops returning what the signature promises, the compiler points at the function, not at some distant call site that consumed the wrong type.

Worked example: every function in the exemplar is annotated — `: Receipt`, `: Cents`, `: Cents`. None relies on inference at the boundary, so each signature is a contract checked at the definition.

**Enforcement:** ESLint `@typescript-eslint/explicit-module-boundary-types`.

### 5.12 — Paragraph with blank lines.

**Reasoning, step by step:**
1. Group statements by thought and separate the groups with a blank line. Whitespace is free (root rule 10); a wall of contiguous statements forces the reader to find the seams that the author already knew.
2. Read the blank lines as paragraph breaks: guards, then computation, then postconditions, then return — four thoughts, three blank lines. The paragraphing is the function's outline made visible, and the cheapest documentation there is.

Worked example: `settleInvoice` is four paragraphs, one blank line between each — guards, then the three-step computation, then the postcondition pair, then the return. The structure is legible before a single expression is read.

**Enforcement:** review; `padding-line-between-statements` can require blanks around blocks, but judgement sets the paragraphs.

## Cross-references

- The line cap, `max-depth`, and `max-params` lint configuration: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming and the client verb taxonomy: [02-naming-conventions.md](./02-naming-conventions.md).
- Assertion functions, `asserts`, and branded types like `Cents`: [03-the-type-system.md](./03-the-type-system.md).
- `const` defaults and the banned non-null `!` outside bridges: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- `interface` + free functions over classes: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Pipelines, `satisfies`, and naming the steps: [07-typescript-idioms.md](./07-typescript-idioms.md).
- `InvariantViolation`, `Error` subclasses, and `Result` unions: [08-error-handling.md](./08-error-handling.md).
- `async`/`await` discipline and `no-floating-promises`: [09-concurrency.md](./09-concurrency.md).
- Explicit return types as the public contract, plus property and type-level tests for pure functions: [10-api-design.md](./10-api-design.md), [11-testing.md](./11-testing.md).
- Bounded loops and pools, hot-path inlining, and V8 monomorphism: [13-resource-management.md](./13-resource-management.md), [15-performance.md](./15-performance.md).
