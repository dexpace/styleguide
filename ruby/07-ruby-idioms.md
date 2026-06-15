# 07 — Ruby Idioms

Ruby's standard library is wide, and most of its expressive power is genuinely useful — when chosen deliberately. This chapter names the idioms that make intent clear, pipelines readable, and code honest, and it names the traps that make the same surface clever instead of clear.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(orders: T::Array[Order]).returns(T::Array[LineItem]) }
def discounted_items(orders)
  eligible = orders.select { |o| o.status == "confirmed" }
  line_items = eligible.flat_map(&:line_items)
  discounted = line_items.select { |li| li.discount_cents > 0 }
  discounted.map { |li| li.tap { |x| Metrics.increment("discount.applied", sku: x.sku) } }
end

sig { params(payload: T::Hash[Symbol, T.untyped]).returns(Money) }
def extract_total(payload)
  case payload
  in { total_cents: Integer => cents, currency: String => currency }
    Money.new(cents:, currency:)
  in { total_cents: Integer => cents }
    Money.new(cents:, currency: "USD")
  end
end
```

`eligible`, `line_items`, and `discounted` name the three pipeline stages so each transformation is legible on its own (7.1). The `&:line_items` shorthand names a single no-argument method call (7.2). `tap` threads inspection through the chain without breaking it (7.3). `case/in` pattern matching in `extract_total` deconstructs the hash structurally and handles optional fields without conditional gymnastics (7.4). Every hash uses shorthand symbol keys (7.10), and no literal array sugar is in sight (7.13).

## Rules

### 7.1 — Build pipelines with `map`/`select`/`reject`/`reduce`/`find`; reach for `each` only for effects or early exit.

**Reasoning, step by step:**
1. `map`, `select`, `reject`, `reduce`, and `find` declare intent: a reader knows at a glance whether a block transforms, filters, accumulates, or searches.
2. `each` is a blank verb — it conveys nothing about what the loop produces. When a loop transforms and assigns into an outer collection, replace it with `map` or `reduce` so the structure is visible.
3. Name each pipeline stage with a local variable when the chain exceeds roughly three steps. The name is the documentation; a long fused method chain forces the reader to mentally execute every step before understanding the whole.
4. `each` is correct for pure effects (writing to a log, enqueueing a job) and for `break`/`next` early exits — things the functional operators cannot express.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid: blank verb, mutates outer array
results = []
line_items.each { |li| results << li.amount if li.fulfilled? }

# prefer: intent named, no mutation
fulfilled_amounts = line_items.select(&:fulfilled?).map(&:amount)
```

**Enforcement:** review; RuboCop `Style/MapIntoArray` and `Style/EachWithObject` surface common `each`-replaces-`map` patterns.

### 7.2 — Use `&:symbol` when the block is a single no-argument method call.

**Reasoning, step by step:**
1. `items.map { |i| i.amount }` says the same thing as `items.map(&:amount)` — one token shorter, no named variable the reader has to track, intent instantly clear.
2. The idiom applies exactly when the block takes one argument and calls one no-argument method on it, with no branching and no further computation.
3. Do not stretch it: `items.map { |i| i.amount * 100 }` has arithmetic and needs the block form; forcing `&:amount` and chaining a `map(&:* 100)` is unreadable.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

skus       = line_items.map(&:sku)
fulfilled  = line_items.select(&:fulfilled?)
totals     = orders.map(&:total_cents).reduce(0, :+)
```

**Enforcement:** RuboCop `Style/SymbolProc` (enabled).

### 7.3 — Use `tap` for side-effecting inspection inside a chain; use `then` to pipe a value through an expression.

**Reasoning, step by step:**
1. `tap` yields the receiver to a block and returns the receiver unchanged. It is the right hook for a log statement, a metric increment, or a debugger breakpoint that must not break the chain — the chain does not know the tap was there.
2. `then` (aliased `yield_self`) yields the receiver to a block and returns the block's return value. Use it to fold an expression that does not fit the receiver's own API into a pipeline: `raw.then { |s| JSON.parse(s) }` keeps the chain linear where a variable assignment would break it.
3. Do not use `tap` to mutate the receiver and carry the mutation forward invisibly; that is a hidden side effect, not an inspection. Mutation belongs in an explicit assignment.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

order
  .tap { |o| Rails.logger.info("processing order #{o.id}") }
  .line_items
  .select(&:in_stock?)
  .map(&:sku)
  .then { |skus| Inventory.reserve(skus) }
```

**Enforcement:** review; `tap` mutation is rejected at code review as a hidden side effect.

### 7.4 — Use `case/in` for structural decomposition; use one-line `=>` and `in` patterns where they read.

**Reasoning, step by step:**
1. `case/in` pattern matching deconstructs hashes, arrays, and `Data` objects structurally. It replaces a tree of `dig`/`[]`/`fetch` calls and `if` guards with one declarative match expression that is both shorter and harder to misread.
2. `Hash` patterns (`in { key: }`) and array patterns (`in [first, *rest]`) reduce entire parse-and-validate blocks to a single match arm. Add `=> variable` to bind matched values directly in the pattern.
3. One-line `=>` (the rightward assignment pattern) binds a value from a deconstruct without branching — useful at the top of a method where the input shape is asserted, not selected from multiple cases.
4. Pin operator `^` matches against an existing variable; use it to guard against a specific runtime value in a structural pattern.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

sig { params(event: T::Hash[Symbol, T.untyped]).returns(T.nilable(Order)) }
def handle_event(event)
  case event
  in { type: "order.created", payload: { order_id: String => id, customer_id: String => cid } }
    Order.find_by(id:, customer_id: cid)
  in { type: "order.cancelled", payload: { order_id: String => id } }
    Order.cancel(id)
  else
    nil
  end
end
```

**Enforcement:** review; prefer `case/in` over nested `fetch`/`dig` chains for structural decomposition.

### 7.5 — Never use `for`; use block iterators.

**Reasoning, step by step:**
1. `for x in collection` leaks the loop variable into the enclosing scope after the loop finishes. Block iterators (`each`, `map`, etc.) scope the block parameter to the block; no leakage.
2. `for` is syntactic sugar over `collection.each` but adds indirection without adding information. Every reader has to recall that equivalence; a block iterator states the pattern directly.
3. `for` provides no `break` value, no block-level rescue, and no way to pass the block further — strictly less capable and more surprising than its iterator equivalent.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid
for item in line_items
  process(item)
end

# prefer
line_items.each { |item| process(item) }
```

**Enforcement:** RuboCop `Style/For` (enabled, `EnforcedStyle: each`).

### 7.6 — Use `{ ... }` for single-line blocks and `do ... end` for multi-line; never chain `do...end`; never write multi-line `{ }`.

**Reasoning, step by step:**
1. `{ ... }` has higher precedence than `do ... end`. On a single line this is irrelevant; in a method chain, a `do ... end` block binds to the last receiver in the chain, not to `map` — silent re-parsing of intent. Use `{ }` for any block that participates in a chain.
2. Multi-line `{ }` is visually ambiguous: the closing `}` floats far from its opening line and readers must scan to find it, unlike `do`/`end` which reads like prose structure.
3. A `do ... end` block attached mid-chain is the mirror error: `items.map do ... end.select { }` attaches `do...end` to `items`, not `map`. Make the chain use `{ }` throughout.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# single-line: brace block
totals = orders.map { |o| o.total_cents }

# multi-line: do...end
orders.each do |order|
  notify(order)
  Metrics.increment("order.notified")
end

# chain: brace throughout
orders
  .select { |o| o.confirmed? }
  .map { |o| o.total_cents }
```

**Enforcement:** RuboCop `Style/BlockDelimiters` (`EnforcedStyle: line_count_based`).

### 7.7 — Never monkey-patch core classes; use a scoped refinement with a why-comment if unavoidable.

**Reasoning, step by step:**
1. Opening `String`, `Integer`, `Array`, or any core class globally changes behaviour for every gem and every caller in the process — including ones written months later by someone who does not know the patch exists. The impact radius is the entire Ruby process.
2. Refinements (`refine` + `using`) scope the patch to the files that explicitly opt in with `using`. The impact radius is the one file. A why-comment on the `using` line documents the tradeoff so it can be revisited.
3. Even refinements are a last resort. Before reaching for one, ask: can an adapter method, a plain module function, or a decorator class express the same thing without patching?

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

module CentsConversion
  refine Integer do
    sig { returns(Money) }
    def to_money
      Money.new(cents: self, currency: "USD")
    end
  end
end

# In the one file that genuinely needs it:
using CentsConversion # why: legacy report adapter; remove when report is retired (2026-Q3)
```

**Enforcement:** RuboCop `Style/MonkeyPatchingProtection`; global reopens of core classes are rejected at review.

### 7.8 — Avoid needless metaprogramming; write `ruby -w`-clean code; explicit beats clever.

**Reasoning, step by step:**
1. `define_method`, `method_missing`/`respond_to_missing?`, and dynamic `send` tricks defer errors to runtime and hide the API surface from Sorbet, from documentation tools, and from every reader who did not write the meta-code. The compiler cannot type-check what it cannot see.
2. `public_send` is the sanctioned dynamic dispatch: it respects `private`/`protected` visibility and raises `NoMethodError` on an unknown name — a loud failure rather than a silent dispatch to an unintended private method.
3. `ruby -w` enables extra warnings: undefined instance variables, ambiguous literals, unused variables. Code should produce zero warnings — every suppressed warning is a potential `nil` or dead branch hiding in production.
4. When metaprogramming genuinely earns its keep (a plugin system, a test double framework), contain it in one module, document the contract with a `sig`, and gate the file with a why-comment.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid: Sorbet cannot type the generated methods; errors are runtime-only
%w[draft confirmed cancelled].each { |s| define_method(:"#{s}?") { status == s } }

# prefer: explicit, typed, grep-able
sig { returns(T::Boolean) }
def draft? = status == "draft"

sig { returns(T::Boolean) }
def confirmed? = status == "confirmed"

sig { returns(T::Boolean) }
def cancelled? = status == "cancelled"
```

**Enforcement:** review; `Method#public_send` over `send` for dynamic dispatch; `define_method` and `method_missing` require a why-comment and architecture approval.

### 7.9 — Use squiggly heredocs `<<~` for multi-line strings.

**Reasoning, step by step:**
1. Plain `<<HEREDOC` preserves all leading whitespace, including the indentation of the surrounding code. The string carries invisible spaces that leak into error messages, SQL, and rendered templates.
2. `<<~` strips the common leading indentation, so the string content can be indented to match the surrounding code without the indentation becoming part of the value.
3. The closing delimiter goes on its own line at the base indentation level. Trailing `.strip` or `.chomp` is still available for final newline handling but is not needed for indentation.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

sig { params(order: Order).returns(String) }
def cancellation_notice(order)
  <<~MSG
    Your order #{order.id} has been cancelled.
    A refund of #{order.total_cents / 100.0} will appear within 5 business days.
    Contact support@dexpace.com with questions.
  MSG
end
```

**Enforcement:** RuboCop `Style/HeredocDelimiterNaming` and `Layout/HeredocIndentation` (enabled); `<<HEREDOC` without squiggly is a cop violation.

### 7.10 — Use shorthand symbol hash keys; prefer symbols over strings as hash keys; use `Hash#fetch` with a default for absent keys.

**Reasoning, step by step:**
1. `{ name: "Acme" }` is shorter than `{ :name => "Acme" }` and unambiguous: shorthand keys must be symbols, so the rocket form is reserved for non-symbol keys (`{ "content-type" => "application/json" }`) where the distinction matters visually.
2. Symbol keys are frozen and interned; they produce stable `object_id` values and zero GC pressure on repeated use as hash keys. String keys allocate a new object per literal.
3. `hash[:key]` returns `nil` on a missing key; the `nil` propagates silently until something blows up far from the original miss. `hash.fetch(:key)` raises `KeyError` immediately at the source. Supply a default block for keys that legitimately may be absent: `hash.fetch(:discount, 0)`.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

sig { params(attrs: T::Hash[Symbol, T.untyped]).returns(Order) }
def build_order(attrs)
  Order.new(
    id:       attrs.fetch(:id),
    customer: attrs.fetch(:customer),
    total:    attrs.fetch(:total_cents, 0),
  )
end
```

**Enforcement:** RuboCop `Style/HashSyntax` (`EnforcedStyle: ruby19`); `Hash#fetch` over `[]` on external/parsed input enforced at review.

### 7.11 — Use string interpolation over concatenation; wrap `@ivars` and `$globals` in `{}` inside interpolation; never call `.to_s` inside interpolation.

**Reasoning, step by step:**
1. String concatenation (`"Order " + id.to_s + " failed"`) requires explicit `.to_s` coercion, a `+` for every junction, and allocates an intermediate string per `+`. Interpolation (`"Order #{id} failed"`) coerces automatically via `to_s`, reads in document order, and allocates one string.
2. Without braces, `"#@total failed"` is legal but ambiguous — readers must know the interpolation ends at the first non-identifier character. `"#{@total} failed"` is unambiguous and grep-able.
3. `"#{value.to_s}"` calls `to_s` twice: interpolation already calls `to_s`. Remove the inner call; it is noise.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid
msg = "Order " + order.id.to_s + " for " + customer.name.to_s + " failed"

# prefer
msg = "Order #{order.id} for #{customer.name} failed"

# ivar: always brace
note = "Balance is #{@balance_cents} cents"
```

**Enforcement:** RuboCop `Style/StringConcatenation` and `Style/RedundantInterpolation` (both enabled); `{}` around ivars and globals enforced by `Style/InterpolationCheck`.

### 7.12 — Use `Time` over `DateTime`; parse known formats with `Time.iso8601` over `Time.parse`.

**Reasoning, step by step:**
1. `DateTime` is a legacy class that models civil time in a combined date-and-time object without timezone awareness. `Time` in Ruby 4.0 is UTC-aware, supports nanosecond precision, and is the foundation for `ActiveSupport::TimeWithZone`. Use `Time` everywhere and keep timestamps in UTC.
2. `Time.parse(s)` accepts dozens of ambiguous formats and falls back to locale-dependent parsing when the format is unclear — a source of silent misparse. `Time.iso8601(s)` raises `ArgumentError` on anything that does not conform to ISO 8601; the failure is immediate and at the source.
3. Prefer `Time.now.utc` over `Time.now`; store and compare in UTC; convert to a local zone only at presentation boundaries.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid
created_at = DateTime.parse(params[:created_at])
expires_at = Time.parse(params[:expires_at])

# prefer
created_at = Time.iso8601(params.fetch(:created_at)).utc
expires_at = Time.iso8601(params.fetch(:expires_at)).utc
```

**Enforcement:** RuboCop `Style/DateTime` (prefer `Time`); review rejects `Time.parse` for known ISO 8601 inputs.

### 7.13 — Prefer plain literal arrays over `%w`/`%i`; use `first`/`last` over `[0]`/`[-1]`; use `&&`/`||`/`!` over `and`/`or`/`not`.

**Reasoning, step by step:**
1. `%w[draft confirmed cancelled]` is a second string-array literal syntax with its own escaping rules. Plain `["draft", "confirmed", "cancelled"]` is the same thing, grep-able as quoted strings, and requires no secondary syntax. This is our recorded deviation from Airbnb (see README deviations ledger).
2. `items[0]` returns `nil` on an empty array without signaling the access; `items.first` returns `nil` identically but reads as intent. More importantly, `items.first(n)` and `items.last(n)` generalize to a slice cleanly, while `[0]`/`[-1]` do not.
3. `and`/`or`/`not` have lower precedence than assignment, which produces non-obvious parse trees: `x = true and false` assigns `true` to `x`. `&&`/`||`/`!` have the precedence operators programmers expect from every C-family language. Use them in all boolean expressions; reserve `and`/`or` for flow control only (and even then, prefer `if`/`unless`).

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# avoid
STATUSES = %w[draft confirmed cancelled]
head = items[0]
tail = items[-1]
valid = user.active? and user.verified?  # assigns true, then evaluates and

# prefer
STATUSES = ["draft", "confirmed", "cancelled"].freeze
head = items.first
tail = items.last
valid = user.active? && user.verified?
```

**Enforcement:** RuboCop `Style/WordArray` (`EnforcedStyle: brackets`), `Style/SymbolArray` (`EnforcedStyle: brackets`), `Style/AndOr` (`EnforcedStyle: always`); `first`/`last` over index literals enforced at review.

## Cross-references

- Formatting baseline, 2-space indentation, 100-column limit, double quotes, trailing commas, and `rubocop-airbnb` cop configuration: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming, predicate `?` suffix, and no `is_`/`get_` prefixes: [02-naming-conventions.md](./02-naming-conventions.md).
- `# typed: strict`, `sig` on every method, `T.let`/`T.cast` with why-comments, and `Hash#fetch` as the nil-discipline tool: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `freeze` on constants, `||=` caveats, and no `@@class` variables: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- 25-line method cap, `public_send` over `send`, guard clauses, and pure-by-default methods: [05-methods.md](./05-methods.md).
- `Data.define` value objects, `class << self` for class methods, `T::Enum` for closed polymorphism: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `StandardError` subclasses, `raise` with class and message, and no silent rescue: [08-error-handling.md](./08-error-handling.md).
- Lazy enumerators, allocation hygiene, and `size` over `count` for performance: [15-performance.md](./15-performance.md).
