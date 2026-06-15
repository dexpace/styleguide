# 04 — Variables & Declarations

How values come into existence, how long they live, and what they are allowed to do. Every rule here keeps scope narrow, state explicit, and mutation an opt-in you have to justify — because in Ruby, the runtime will not do it for you.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

class Order
  extend T::Sig

  MINIMUM_LINE_COUNT = T.let(1, Integer)
  ZERO_MONEY        = T.let(Money.new(cents: 0), Money)

  sig { returns(T::Array[LineItem]) }
  attr_reader :line_items

  sig { returns(Customer) }
  attr_reader :customer

  sig { params(line_items: T::Array[LineItem], customer: Customer).void }
  def initialize(line_items:, customer:)
    raise ArgumentError, "order must have at least #{MINIMUM_LINE_COUNT} line item" if line_items.empty?

    @line_items = T.let(line_items.freeze, T::Array[LineItem])
    @customer   = T.let(customer, Customer)
    @total      = T.let(nil, T.nilable(Money))
  end

  sig { returns(Money) }
  def total
    @total ||= compute_total
  end

  class << self
    extend T::Sig

    sig { params(raw: T::Hash[String, T.untyped]).returns(Order) }
    def parse(raw)
      line_items = raw.fetch("line_items").map { |l| LineItem.parse(l) }
      customer   = Customer.parse(raw.fetch("customer"))
      new(line_items:, customer:)
    end
  end

  private

  sig { returns(Money) }
  def compute_total
    line_items.reduce(ZERO_MONEY) { |sum, item| sum + item.subtotal }
  end
end
```

`MINIMUM_LINE_COUNT` and `ZERO_MONEY` are frozen constants declared at the top of the class, typed with `T.let`, turning magic literals into named facts (4.11). Instance variables are declared once at initialization with `T.let` and never reassigned by a writer (4.9). `@total` memoizes through `||=` a pure, idempotent computation; the memoizing block is a simple method call — no `return` inside a `begin` (4.6). `raw.fetch` over `[]` surfaces missing keys explicitly (cross-ref: 03). `class << self` groups the parse-constructor (4.10). `line_items:, customer:` uses shorthand hash keys (4.8). No `$globals`, no `@@class` variables, no redundant `self.` on the readers (4.2, 4.3, 4.10).

## Rules

### 4.1 — Keep local-variable scope minimal; declare near first use.

**Reasoning, step by step:**
1. A variable declared at the top of a method forces every reader to track a name that means nothing yet. Declare it on the line where it first has a value so the intent and the assignment are visible together.
2. Short-lived names reduce the mental stack. A name whose live range is two lines is easy to reason about; one whose range spans thirty lines requires re-reading the whole body.
3. Intermediate locals should capture a named concept, not a temporary step. If a name is too generic to have an honest single-word label (`tmp`, `result`, `data`), extract the computation to a method that earns a real name.

Worked example:

```ruby
# bad — subtotal declared before it has a value; live range covers the whole body
sig { returns(Money) }
def build_summary
  subtotal = Money.new(cents: 0)
  label    = ""
  line_items.each { |item| subtotal += item.subtotal }
  label = customer.display_name
  Summary.new(subtotal:, label:)
end

# good — declared at first use; scopes are as tight as the computation allows
sig { returns(Money) }
def build_summary
  subtotal = line_items.reduce(Money.new(cents: 0)) { |sum, item| sum + item.subtotal }
  label    = customer.display_name
  Summary.new(subtotal:, label:)
end
```

**Enforcement:** review; the live-range smell surfaces as a gap between declaration and first meaningful use.

### 4.2 — No `$global` variables and no Perl special variables.

**Reasoning, step by step:**
1. `$global` variables are process-wide mutable state with no owner. Any file can write them; any test can poison them; any thread can race on them. They are the worst possible scope.
2. Perl special variables (`$1`, `$2`, `$~`, `$;`, `$,`, `$\`) are global state disguised as punctuation. They are silently overwritten by any subsequent regex or method call in the same thread, making code that reads them after any intervening call unreliably racy.
3. Pass state explicitly: return values, keyword arguments, or instance variables on a well-scoped object. Name the dependency so it is visible in the signature.

Worked example:

```ruby
# bad — $1 is global state, overwritten by the next regex anywhere in the call stack
if sku_string =~ /\ASKU-(\d+)\z/
  sku_id = $1.to_i
end

# good — named captures, no global state, result scoped to this match
if (m = sku_string.match(/\ASKU-(?<id>\d+)\z/))
  sku_id = m[:id].to_i
end
```

**Enforcement:** RuboCop `Style/GlobalVars` and `Style/PerlBackrefs`; both are errors.

### 4.3 — No `@@class` variables; use a class instance variable or a frozen constant.

**Reasoning, step by step:**
1. `@@class` variables are shared across the entire inheritance hierarchy — a subclass writes `@@count` and the superclass's reader sees the change. This is non-local mutation with no syntactic warning and no scoping protection.
2. A class instance variable (`@count` inside `class << self`) is owned by exactly one class. Subclasses inherit nothing unless they opt in. The scope is visible and auditable.
3. If the value is fixed at load time, a frozen constant is even better: it is immutable, named, and accessible through `SCREAMING_SNAKE_CASE` so grep finds every reader.

Worked example:

```ruby
# bad — @@registry is shared across all subclasses of Inventory; a subclass write poisons the parent
class Inventory
  @@registry = T.let({}, T::Hash[String, Sku])
end

# good — @registry is owned by Inventory alone
class Inventory
  @registry = T.let({}.freeze, T::Hash[String, Sku])

  class << self
    extend T::Sig

    sig { returns(T::Hash[String, Sku]) }
    attr_reader :registry
  end
end
```

**Enforcement:** RuboCop `Style/ClassVars`; error.

### 4.4 — Freeze every mutable constant; `frozen_string_literal: true` does not cover arrays or hashes.

**Reasoning, step by step:**
1. `frozen_string_literal: true` freezes bare string literals so that `"dexpace"` becomes an immutable object. It does not freeze array or hash literals, which remain mutable and can be modified by any caller that holds a reference.
2. An unfrozen constant is an oxymoron. `STATUSES = ["open", "closed"]` is a constant *name* bound to a mutable object; any code can call `STATUSES << "pending"` and corrupt every reader that assumed the array was fixed.
3. `freeze` the object at the point of assignment. For nested structures, freeze each level or use `deep_freeze` via a library. For typed constants, wrap in `T.let` so Sorbet checks the type at load time.

Worked example:

```ruby
# bad — ALLOWED_CURRENCIES is a constant name on a mutable array; any caller can push to it
ALLOWED_CURRENCIES = T.let(["USD", "EUR", "GBP"], T::Array[String])

# good — frozen; a push raises FrozenError immediately, surfacing the bug at the mutation
ALLOWED_CURRENCIES = T.let(["USD", "EUR", "GBP"].freeze, T::Array[String])

# good — frozen hash; all keys and values are already frozen string literals
FEE_BY_CURRENCY = T.let({
  "USD" => Money.new(cents: 50),
  "EUR" => Money.new(cents: 45),
}.freeze, T::Hash[String, Money])
```

**Enforcement:** RuboCop `Style/MutableConstant`; error. `frozen_string_literal: true` is required in every file by rule 01.

### 4.5 — Use `||=` to lazily initialize, but never on booleans.

**Reasoning, step by step:**
1. `||=` is the idiomatic lazy-init pattern: `@x ||= expensive_call` assigns only if `@x` is `nil` or `false`. It is clear, concise, and widely understood in Ruby.
2. For boolean-typed values it is a latent bug. `@enabled ||= true` reads as "enable if not already set," but it will also re-enable if `@enabled` was explicitly set to `false`. The condition is falsiness, not absence.
3. For booleans, test for `nil` explicitly: `@enabled = true if @enabled.nil?`. This preserves `false` as a deliberate value rather than treating it the same as "never set."

Worked example:

```ruby
# bad — re-enables even when explicitly disabled; @enabled = false is overwritten
@enabled ||= true

# good — initializes only when unset; an explicit false is respected
@enabled = true if @enabled.nil?

# good — ||= is correct for non-boolean lazy init
@pricing_table ||= PricingTable.load(sku_ids)
```

**Enforcement:** review; RuboCop `Style/OrAssignment` has no boolean-specific mode — this is a manual code-review check. Sorbet's `sig` (chapter 03) narrows the type and makes the boolean case visible.

### 4.6 — Memoize only pure, idempotent results; never `return` from inside a memoizing `begin` block.

**Reasoning, step by step:**
1. Memoization with `@x ||= compute` is correct when `compute` is pure and idempotent: the same inputs always produce the same result and calling it twice has no side effects. Memoizing an effectful computation (a write, an enqueue, a network request) caches the *first* result and silently skips all future calls — a hidden no-op bug.
2. A `return` inside a `begin` block used for memoization skips the assignment. The pattern `@x ||= begin; ...; return y; end` exits the method before `@x` is set, so the next call recomputes from scratch, defeating the cache and executing any side effects again.
3. Extract the computation to a private method and memoize the method call: `@total ||= compute_total`. The private method may use early returns freely; the memoizing line is a one-liner with no branching.

Worked example:

```ruby
# bad — return inside begin skips the assignment; @breakdown is never cached
def breakdown
  @breakdown ||= begin
    items = line_items.select(&:taxable?)
    return Money.new(cents: 0) if items.empty?  # exits before @breakdown is set
    compute_breakdown(items)
  end
end

# good — early return lives inside the private method; memoizing line is clean
sig { returns(Money) }
def breakdown
  @breakdown ||= compute_breakdown
end

private

sig { returns(Money) }
def compute_breakdown
  items = line_items.select(&:taxable?)
  return Money.new(cents: 0) if items.empty?
  items.reduce(Money.new(cents: 0)) { |sum, item| sum + item.tax }
end
```

**Enforcement:** review; the pattern is caught during code review. Thread-unsafe memoization in concurrent contexts is addressed in chapter 09.

### 4.7 — Treat the return value of `=` as a truth test only when the assignment is parenthesized.

**Reasoning, step by step:**
1. Ruby permits using the result of an assignment as a condition: `if m = string.match(pattern)`. Without parens this looks like a typo for `==`, causing reviewers to stop and re-read. The intent is not self-evident.
2. Wrapping the assignment in parens is a signal: `if (m = string.match(pattern))` tells the reader "yes, I mean assignment, not equality — I am capturing the result." It is the universal idiom for regex capture and conditional assignment-and-check.
3. Outside the parens idiom, assign to a local on one line and test it on the next; separate steps are clearer than a combined expression.

Worked example:

```ruby
# bad — reads as a likely typo for ==; reviewer must check the surrounding context
if m = order_ref.match(/\AOR-(?<id>\d+)\z/)
  process(m[:id])
end

# good — parens signal deliberate assignment; one idiomatic convention, one reader pause
if (m = order_ref.match(/\AOR-(?<id>\d+)\z/))
  process(m[:id])
end

# also good — split into two lines when the value is reused or the condition is complex
m = order_ref.match(/\AOR-(?<id>\d+)\z/)
process(m[:id]) if m
```

**Enforcement:** RuboCop `Lint/AssignmentInCondition` with `AllowSafeAssignment: true` (parens permitted, bare assignment flagged); default in `rubocop-airbnb`.

### 4.8 — Use shorthand self-assignment operators.

**Reasoning, step by step:**
1. `count = count + 1` is noisier than `count += 1` and forces the reader to verify that the left-hand name matches the right-hand name — a redundancy the operator eliminates.
2. The same holds for `<<`, `||=`, and `&&=`. Each operator conveys the update operation in its name: append, lazy-init, and conditional-clear. The expanded form buries the operation inside a repetition.
3. Shorthand operators also reduce the scope for transposition bugs where the left and right names diverge during a rename.

Worked example:

```ruby
# bad — verbose, repeats the variable name, one extra read per operator
total    = total + item.subtotal
tags     = tags + [new_tag]
@cache   = @cache || compute_cache
@session = @session && @session.refresh

# good — operator names the operation; left-hand is written once
total    += item.subtotal
tags     << new_tag
@cache   ||= compute_cache
@session &&= @session.refresh
```

**Enforcement:** RuboCop `Style/SelfAssignment`; error.

### 4.9 — Use `attr_reader` for trivial accessors; value objects are immutable so never add `attr_writer` or `attr_accessor`.

**Reasoning, step by step:**
1. `attr_reader` generates a one-line reader method — identical to writing `def x; @x; end` but without the noise. Use it for any instance variable that is part of the public interface.
2. `attr_writer` and `attr_accessor` open a mutation path. For `Data.define`-style or custom value objects, mutation is forbidden: the object's identity is its values at construction time (root rule 3). Adding a writer breaks that contract quietly because the type system cannot see it.
3. When an attribute genuinely needs to change over the object's lifetime, keep it as an explicit `def x=(value)` with a `sig` and a why-comment explaining the lifecycle — make the mutation opt-in and named, not a default affordance.

Worked example:

```ruby
# bad — attr_accessor on a value object; any caller can overwrite the unit price
class LineItem
  extend T::Sig
  attr_accessor :unit_price, :quantity, :sku  # mutation allowed; immutability violated
end

# good — attr_reader only; object is immutable after initialization
class LineItem
  extend T::Sig

  sig { returns(Money) }
  attr_reader :unit_price

  sig { returns(Integer) }
  attr_reader :quantity

  sig { returns(Sku) }
  attr_reader :sku

  sig { params(unit_price: Money, quantity: Integer, sku: Sku).void }
  def initialize(unit_price:, quantity:, sku:)
    @unit_price = T.let(unit_price, Money)
    @quantity   = T.let(quantity, Integer)
    @sku        = T.let(sku, Sku)
    freeze
  end
end
```

**Enforcement:** review; RuboCop `Style/AttrReadable` does not enforce immutability — this is a code-review gate enforced by the value-object contract in chapter 06.

### 4.10 — Drop redundant `self.` except where the language requires it.

**Reasoning, step by step:**
1. Inside an instance method, `self.total` and `total` both call the same reader. The `self.` is noise that the reader must parse without receiving information.
2. The language *requires* `self.` in three places: assignment to an instance accessor (`self.total = value` disambiguates from a local variable), defining class methods (`def self.parse`), and referencing the class itself (`self.class`). Everywhere else, omit it.
3. Conversely, in `class << self` blocks, every method is already a class method — no `self.` prefix is needed on the definition, and the block itself makes the class scope explicit.

Worked example:

```ruby
# bad — redundant self. on every read; adds visual noise, teaches nothing
sig { returns(Money) }
def subtotal
  self.unit_price * self.quantity  # self. not required; both are readers
end

# good — omit where the language does not require it
sig { returns(Money) }
def subtotal
  unit_price * quantity
end

# required — assignment to a writer; without self., Ruby creates a local variable
sig { params(price: Money).void }
def apply_discount(price)
  self.unit_price = price  # self. required here
end
```

**Enforcement:** RuboCop `Style/RedundantSelf`; error.

### 4.11 — Declare constants at the top of the class or module, frozen, typed with `T.let`; magic numbers become named constants.

**Reasoning, step by step:**
1. A constant declared at the top of its class is the first thing a reader sees. The class's fixed domain knowledge — limits, sentinel values, configuration — is visible before any method, so methods that reference the constant are readable without scrolling back.
2. Magic numbers and strings embedded in method bodies are opaque. `quantity > 99` and `status == "open"` repeat context the reader must reconstruct; `MAX_LINE_QUANTITY` and `STATUS_OPEN` state it once at the top and are grep-able across the codebase.
3. `T.let(value, Type)` gives Sorbet a concrete type for the constant, catching misassignments at load time. Without it, Sorbet infers `T.untyped` and the constant becomes a type-checking blind spot.
4. Freeze the value at the assignment site (see 4.4). Constants of frozen types (integers, symbols, `Data.define` objects) are already immutable; strings, arrays, and hashes are not.

Worked example:

```ruby
# bad — magic numbers embedded in methods; no single source of truth; Sorbet cannot type them
sig { returns(T::Boolean) }
def bulk_order?
  line_items.size > 10 && total.cents > 100_000
end

# good — named, typed, frozen constants at the top; methods are self-documenting
class Order
  extend T::Sig

  BULK_LINE_THRESHOLD  = T.let(10, Integer)
  BULK_AMOUNT_CENTS    = T.let(100_000, Integer)
  STATUS_OPEN          = T.let("open".freeze, String)

  sig { returns(T::Boolean) }
  def bulk_order?
    line_items.size > BULK_LINE_THRESHOLD && total.cents > BULK_AMOUNT_CENTS
  end
end
```

**Enforcement:** RuboCop `Style/MagicNumber` (via `rubocop-magic_numbers` gem); `T.let` on constants enforced at review; constants without `freeze` on mutable values caught by `Style/MutableConstant`.

## Cross-references

- Sorbet `T.let`, `T.cast`, `T.must`, and nil discipline: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `freeze`, double quotes, 2-space indent, 100-col limit, and `rubocop` baseline: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- `SCREAMING_SNAKE_CASE` naming for constants and effect-verb naming: [02-naming-conventions.md](./02-naming-conventions.md).
- `attr_reader` on value objects and `Data.define` immutable modeling: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `class << self` for class-method grouping: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Shorthand hash keys, `Hash#fetch`, and enumerable pipelines: [07-ruby-idioms.md](./07-ruby-idioms.md).
- Thread-safe memoization with `Mutex`: [09-concurrency.md](./09-concurrency.md).
- Parse-constructor pattern (`Order.parse`) and `sig` on public API: [10-api-design.md](./10-api-design.md).
