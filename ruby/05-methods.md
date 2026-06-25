# 05 — Methods

Methods are the unit of reasoning. Every rule here keeps that unit small, named, and honest: one job, one level of abstraction, asserted at both ends, with the happy path flush left. The 25-line cap is the floor of the discipline; aim far below it.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(order: Order, now: Time).returns(Receipt) }
def settle_order(order, now:)
  Assert.that(order.line_items.any?, "order #{order.id} has no line items")
  Assert.that(order.status == :open, "order #{order.id} is not open")

  subtotal = sum_line_items(order.line_items)
  tax      = tax_for(subtotal, jurisdiction: order.customer.jurisdiction)
  total    = Money.parse(subtotal.cents + tax.cents)

  Assert.that(total >= subtotal, "total cannot fall below subtotal")
  Assert.that(total >= tax,      "total cannot fall below tax")

  Receipt.new(order_id: order.id, subtotal:, tax:, total:, settled_at: now)
end

sig { params(items: T::Array[LineItem]).returns(Money) }
def sum_line_items(items)
  cents = items.reduce(0) do |acc, item|
    Assert.that(item.unit_price.cents >= 0, "negative unit price on sku #{item.sku}")
    acc + item.unit_price.cents * item.quantity
  end
  Money.parse(cents)
end

sig { params(subtotal: Money, jurisdiction: Symbol).returns(Money) }
def tax_for(subtotal, jurisdiction:)
  rate = TAX_RATES.fetch(jurisdiction) { Assert.fail("no tax rate for #{jurisdiction}") }
  Money.parse((subtotal.cents * rate).ceil)
end
```

`Money` is generated in exactly one place — `Money.parse`, the parse-constructor that validates the integer-cents value before returning an immutable value object. Nothing else creates a `Money`. The public method sits above the two helpers it calls, so the file reads top-down (5.4). Guard clauses assert the preconditions and leave the happy path flush left (5.3). The keyword argument `now:` keeps the call site legible without a positional slot that callers can transpose (5.4 rule). A postcondition pair verifies the sum from both of its parts before return (5.12). Each method holds one level of abstraction — `settle_order` orchestrates named steps and touches no arithmetic itself (5.2). The helpers reach a `Money` only through `Money.parse`, the single parse-constructor that validates (per chapter 03).

## Rules

### 5.1 — 25 lines. Hard cap. Aim for 5–15.

**Reasoning, step by step:**
1. A method you can hold in your head fits on half a screen. Past ~15 lines it stops being one idea and starts being a list of them.
2. The cap is 25 lines, cop-enforced (blank lines counted, comments excluded). It is set deliberately tight — the Ruby-scaled sibling of Go's 70 and Kotlin's 60. Hitting it is already a smell; brushing against it is a prompt to extract.
3. The test is linguistic: if the honest one-sentence summary of a method needs an "and," it is two methods wearing one name. Split on the "and."
4. Every callable counts — public methods, private helpers, module methods, and block bodies alike. A 25-line block passed to `each` is the same defect as a 25-line public method.

Worked example: a `process_checkout` that parses the payload, *and* validates inventory, *and* charges the card, *and* persists the order is four methods. Extract `parse_checkout_payload`, `assert_inventory_available`, `charge_payment`, `persist_order`; the orchestrator drops to a handful of readable lines.

**Enforcement:** RuboCop `Metrics/MethodLength: Max: 25` (CountAsOne for heredocs; blank lines counted).

### 5.2 — One level of abstraction per method.

**Reasoning, step by step:**
1. A method either *orchestrates* named steps or *performs* primitive work. Mixing the two forces the reader to shift altitude mid-body — re-deriving what a high-level call means while parsing arithmetic or string slicing.
2. Pick the altitude from the name. `settle_order` names a workflow, so its body is calls to named helpers. `sum_line_items` names a computation, so its body is a loop over values.
3. When an orchestrator sprouts arithmetic, a regex, or a conditional that belongs in a helper, that fragment wants a name. Extracting it makes the helper's name document the step you removed, replacing a comment.
4. Depth is a symptom: a body mixing abstraction levels grows extra nesting as the low-level logic branches. If `max-depth` fires, first ask whether the culprit deserves its own method.

Worked example: in the exemplar, `settle_order` never multiplies or calls `.ceil` — it delegates to `sum_line_items` and `tax_for`. Inlining either would drop two levels of abstraction into one body and the workflow would vanish into arithmetic.

**Enforcement:** review; surfaced by 5.1's line cap and RuboCop `Metrics/BlockNesting` (mixing levels nests deeper).

### 5.3 — Guard clauses first; happy path flush left.

**Reasoning, step by step:**
1. Handle the exceptional at the top and return (or raise) early. What survives the guards is the main case, and it reads as a straight column down the left margin.
2. Early return beats nesting: every `if` you invert into a guard is an indentation level the happy path never pays for. An `else` branch is usually a guard you failed to take.
3. Deep nesting hides the spine of a method inside pyramids of `end` keywords. Flat code has one obvious path. Ruby's `return` at the top of a method is idiomatic and clear; use it without hesitation.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(sku: Sku, cart: Cart).returns(Money) }
def price_for(sku, cart)
  item = cart.item_for(sku)
  return Money.parse(0) if item.nil? || item.quantity.zero?   # guard

  Money.parse(item.unit_price.cents * item.quantity)          # happy path, flush left
end
```

**Enforcement:** RuboCop `Metrics/BlockNesting: Max: 3`; aim for ≤ 2. Reviewer rejects inversion of guard into deep `else`.

### 5.4 — Keyword arguments over positional; never bare positional defaults.

**Reasoning, step by step:**
1. A call site must read without opening the signature. `ship(order, true, false, 3)` is a row of unlabeled values; `ship(order, expedited: true, insured: false, attempts: 3)` says what each means.
2. Keyword arguments are named at the call site and order-free. Adding a new keyword argument is backward-compatible; inserting a positional breaks every caller.
3. Never give a positional parameter a default value — the default is invisible at the call site and becomes a silent time bomb when the signature changes. A keyword with a default is visible in the call because callers omit it deliberately.
4. An options hash (`def foo(options = {})`) is the defeated form: it loses type coverage, destroys IDE navigation, and lets unknown keys pass silently. Use keyword arguments.

Worked example: `settle_order(order, now:)` vs `settle_order(order, Time.now)` — the keyword call says what `now` is; the positional call is a riddle unless you memorize the signature.

**Enforcement:** RuboCop `Style/OptionHash` (bans options hashes); `Style/KeywordParameters`; reviewer rejects any positional default.

### 5.5 — ≤ 4 parameters; a behaviour-forking boolean is two methods.

**Reasoning, step by step:**
1. A method with five or more parameters is usually doing more than one thing. Extract a value object or split the responsibility before reaching the fourth parameter.
2. A configuration boolean can live as a keyword argument: `search(query, fuzzy: true)`. But a behaviour-forking boolean — one whose truth value selects between two fundamentally different algorithms — is two methods hiding in one body. Split them.
3. `parse(input, strict: true)` sounds like configuration, but if the `strict` branch runs a completely different code path, the shared name is a lie. `parse_strict(input)` and `parse_lenient(input)` each tells the truth; shared work moves to a private helper both call.
4. Each extracted method now obeys rules 5.1 and 5.2 on its own; the fork that prevented that is gone.

Worked example:

```ruby
# Before — behaviour fork masquerading as configuration
sig { params(input: String, strict: T::Boolean).returns(Money) }
def parse_amount(input, strict:)
  if strict
    # entirely different validation and parsing path
  else
    # lenient path with fallbacks
  end
end

# After — two honest names, shared private core
sig { params(input: String).returns(Money) }
def parse_amount_strict(input)
  raw = parse_amount_raw(input)
  Assert.that(raw[:remainder].empty?, "trailing characters: #{raw[:remainder]}")
  Money.parse(raw[:cents])
end

sig { params(input: String).returns(Money) }
def parse_amount_lenient(input)
  Money.parse(parse_amount_raw(input).fetch(:cents))
end
```

**Enforcement:** RuboCop `Metrics/ParameterLists: Max: 4`; reviewer splits behaviour-forking booleans.

### 5.6 — Prefer `next` (and `return`) over nested conditional bodies in loops.

**Reasoning, step by step:**
1. A conditional body inside a loop that wraps most of its content is an indentation tax on every line of the happy case. Inverting it into a `next` keeps the meaningful work flush left.
2. `next` in a block is the guard-clause principle (5.3) applied inside an iteration: skip the degenerate cases at the top; everything below is the normal case.
3. The same principle applies inside `map`/`select`/`reduce` blocks — guard with `next` rather than wrapping the transform in an `if`.

Worked example:

```ruby
# Before — nested conditional buries the work
order.line_items.each do |item|
  if item.in_stock?
    if item.quantity > 0
      process_line_item(item)
    end
  end
end

# After — guards, then work flush left
order.line_items.each do |item|
  next unless item.in_stock?
  next unless item.quantity > 0

  process_line_item(item)
end
```

**Enforcement:** RuboCop `Style/Next` (enforces `next` over wrapped conditionals); reviewer enforces the pattern in block bodies.

### 5.7 — No single-line method definitions; avoid endless methods except trivial pure one-liners.

**Reasoning, step by step:**
1. `def price; @price; end` on one line suppresses the visual boundary between methods and makes the file harder to scan. One body per line is the rule; the `end` goes on its own line.
2. Ruby 3's endless method (`def total = subtotal + tax`) is acceptable for a trivial pure expression with no guard clause, no local variable, and no side effect — the kind you would write as a read-only `attr_reader` if the value were stored. Anything beyond that goes on its own lines.
3. The bar for "trivial" is high: if the body contains a method call with arguments, a conditional, or any local binding, use the traditional `def`/`end` form so the body can be read and `sig`-annotated properly.

Worked example:

```ruby
# Acceptable endless — pure delegation, trivially readable
sig { returns(Integer) }
def cents = amount.cents

# Unacceptable endless — has logic; use traditional form
# def total = line_items.sum(&:subtotal) + tax   ← no: too much for one expression

sig { returns(Money) }
def total
  subtotal = line_items.sum(&:subtotal)
  Money.parse(subtotal + tax.cents)
end
```

**Enforcement:** RuboCop `Style/SingleLineMethods`; reviewer rejects endless methods with conditional or multi-step logic.

### 5.8 — Omit redundant `return`; reserve explicit `return` for guards.

**Reasoning, step by step:**
1. In Ruby the last expression in a method is its return value. Writing `return value` at the bottom of a method adds a keyword that the language already infers, and signals to the reader that something exceptional is happening — a signal that turns to noise if used everywhere.
2. Reserve `return` for early exits: guards at the top of a method that escape before the main body runs. That gives `return` a single, recognizable role: "stop here, the rest does not apply."
3. This asymmetry is load-bearing: when a reviewer sees `return` mid-method, it should mean "guard or unusual exit" — not "this author types `return` everywhere." Consistency makes the meaning visible.

Worked example:

```ruby
# Guard exit — explicit `return` is correct
sig { params(order: Order).returns(T.nilable(Discount)) }
def discount_for(order)
  return nil if order.customer.loyalty_tier == :standard   # guard: explicit return
  return nil unless order.total >= Money.parse(10_000)     # guard: explicit return

  Discount.new(rate: LOYALTY_RATE)                         # happy path: implicit return
end
```

**Enforcement:** RuboCop `Style/RedundantReturn`; reviewer enforces the guard-only idiom for explicit `return`.

### 5.9 — `def` with parentheses when it takes parameters; omit them when it takes none.

**Reasoning, step by step:**
1. Parentheses after `def` signal "this method has a contract to inspect." Their absence signals "nothing to pass." The visual distinction is a scanning aid — readers know at a glance whether to look for arguments.
2. This is consistent with Sorbet `sig` blocks: a `sig { params(...).returns(...) }` always accompanies a `def` with parentheses, while a `sig { returns(...) }` always accompanies a `def` without. The two forms reinforce each other.
3. Never omit parentheses on a definition that takes parameters, even if all parameters have defaults. The parentheses are not about whether callers must provide values; they are about whether there is a parameter list at all.

Worked example:

```ruby
sig { returns(Money) }
def subtotal          # no params, no parens
  line_items.sum(&:amount)
end

sig { params(rate: Float).returns(Money) }
def apply_discount(rate)   # has params, has parens
  Money.parse((subtotal.cents * (1 - rate)).ceil)
end
```

**Enforcement:** RuboCop `Style/DefWithParentheses` and `Style/MethodDefParentheses`.

### 5.10 — Pure by default; side effects live at the edges.

**Reasoning, step by step:**
1. A pure method — same input, same output, no observable effect — is trivially testable, safe to call from property tests, and safe to memoize. Make it the default; the testable surface of the codebase grows for free.
2. Side effects belong at the edges: a thin shell does the I/O or the mutation and delegates the logic to a pure core. This is the same instinct as root rule 6 (transform, don't mutate) applied at the method boundary.
3. Name the effect. A side-effecting method carries an effect verb: `write_ledger`, `emit_event`, `persist_order`, `notify_customer`. A name like `data` or `process` hiding a database write is a lie at the call site (chapter 02).
4. Inject time, randomness, and external state as parameters so the core remains pure and the tests remain fast. `Time.now` inside a method is a hidden dependency; `now:` is explicit.

Worked example:

```ruby
# pure core — no I/O, injectable time
sig { params(order: Order, now: Time).returns(Receipt) }
def build_receipt(order, now:)
  settle_order(order, now:)
end

# thin shell — does the I/O, delegates to the pure core
sig { params(order: Order).returns(Receipt) }
def record_sale(order)
  receipt = build_receipt(order, now: Time.now)  # pure core
  LedgerWriter.write(receipt)                     # effect, at the edge
  receipt
end
```

**Enforcement:** review; effect verbs enforced at code review (naming taxonomy in [02-naming-conventions.md](./02-naming-conventions.md)).

### 5.11 — `public_send` over `send`.

**Reasoning, step by step:**
1. `send` bypasses Ruby's visibility rules: it can call `private` and `protected` methods as if they were public. Using it in application code erases the encapsulation the visibility modifiers were meant to enforce.
2. `public_send` calls only public methods and raises `NoMethodError` on a private or protected receiver — exactly the behaviour visibility is supposed to provide.
3. If you find yourself reaching for `send` to access a private method, the real fix is to ask why that method is private. Either make it public, extract it, or redesign the boundary. `send` to a private method is a design smell, not a technique.
4. The only legitimate use of `send` in application code is when `public_send` is genuinely insufficient — typically in test helpers that deliberately exercise private internals, or in framework-level introspection. Annotate those sites with a comment explaining why.

Worked example:

```ruby
# Prefer
sig { params(processor: Processor, event: Symbol).void }
def dispatch(processor, event)
  processor.public_send(event)  # respects visibility; raises if not public
end

# Avoid
def dispatch(processor, event)
  processor.send(event)  # bypasses private/protected — do not use
end
```

**Enforcement:** RuboCop custom cop or `Lint/SendWithMixinArgument`; reviewer rejects bare `send` in non-test application code.

### 5.12 — Assert aggressively: 2+ per method on average.

**Reasoning, step by step:**
1. Assertions are executable preconditions and postconditions. Check arguments at entry and results before return. The average across a module should land at two or more per method; anything lower means whole regions of behaviour are unguarded.
2. The runtime-checked `sig` covers types. Add explicit value-level assertions on top: the `sig` cannot express that an array must be non-empty, that a monetary total must be non-negative, or that a computed sum must be at least as large as any of its parts.
3. Assert *positive and negative space*: not just that the expected holds, but that the impossible is absent. A total must never fall below its subtotal *and* never fall below its tax — both assertions rule out separate failure modes (negative tax, negative subtotal).
4. Pair-assert: verify one property two independent ways so that when the two derivations disagree, a bug surfaces at the assertion rather than three layers downstream in the customer's receipt.
5. Define an `Assert` module once, project-wide. Give it a dedicated error class so callers can distinguish a broken invariant (programmer error) from an operational failure they might recover from.

Define it once, project-wide:

```ruby
# frozen_string_literal: true
# typed: strict

module Assert
  class InvariantViolation < StandardError; end

  sig { params(condition: T::Boolean, message: String).void }
  def self.that(condition, message)
    raise InvariantViolation, message unless condition
  end

  sig { params(message: String).returns(T.noreturn) }
  def self.fail(message)
    raise InvariantViolation, message
  end
end
```

`Assert.that` is the Ruby parallel to the TypeScript `invariant(): asserts` helper. Use it for preconditions, postconditions, and any invariant that `sig` cannot express. `Assert.fail` is the unconditional form for exhaustive case analysis and unreachable branches.

**Enforcement:** review and code review; density is a target, not a lint rule. `Assert.that` is the sole sanctioned assertion primitive (see [08-error-handling.md](./08-error-handling.md)).

## Cross-references

- Line-length cap, `Metrics/MethodLength`, `Metrics/BlockNesting`, and `Metrics/ParameterLists` cop configuration: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming, predicate suffixes, and the naming taxonomy for side-effecting methods: [02-naming-conventions.md](./02-naming-conventions.md).
- `sig` on every method, `T.let`/`T.cast` discipline, and `Money.parse` as the parse-constructor: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- Local variable scope, `attr_reader`, and memoization caveats that affect method bodies: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- `Data.define` value objects, `class << self` grouping, and module-of-functions pattern: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Enumerable pipelines, `next` in block idioms, and pattern matching as an alternative to branching: [07-ruby-idioms.md](./07-ruby-idioms.md).
- `Assert::InvariantViolation`, `StandardError` subclasses, and the error taxonomy: [08-error-handling.md](./08-error-handling.md).
- Pure method design and deterministic testing with injected `Time`: [11-testing.md](./11-testing.md).
- Public API surface, `sig` on every public method, and keyword-argument contracts at boundaries: [10-api-design.md](./10-api-design.md).
