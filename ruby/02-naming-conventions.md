# 02 — Naming Conventions

Airbnb's Ruby guide is the floor; this chapter extends it with the dexpace layer. Casing rules are mechanical and cop-enforced; predicate contracts, bang discipline, effect-verb taxonomy, and call-site design are where correctness and legibility are actually won.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  TAX_RATE_PCT = T.let(8, Integer).freeze

  class Order
    extend T::Sig

    sig { returns(T::Boolean) }
    def fulfilled?
      line_items.all?(&:shipped?)
    end

    sig { params(other: Order).returns(T::Boolean) }
    def ==(other)
      id == other.id
    end

    sig { returns(Money) }
    def subtotal
      line_items.reduce(Money.zero) { |running_total, line| running_total + line.amount }
    end

    sig { void }
    def charge_card
      CardGateway.submit(token: customer.payment_token, amount: subtotal)
    end
  end

  class LineItem
    extend T::Sig

    sig { returns(T::Boolean) }
    def shipped?
      !shipped_at.nil?
    end

    sig { returns(Money) }
    def amount
      sku.unit_price * quantity
    end
  end
end
```

`TAX_RATE_PCT` names a constant in `SCREAMING_SNAKE_CASE` and carries its unit in the name (2.3). `fulfilled?` and `shipped?` return real booleans — every branch of the predicate lands on `true` or `false`, never a truthy object (2.5). `charge_card` carries an effect verb announcing the side effect at the call site (2.8); `subtotal` and `amount` read as nouns because they are pure transforms (2.8). The `reduce` block uses mnemonic names `running_total` and `line`, not `a, b` (2.9). `==` receives `other`, the canonical binary-operator parameter name (2.9). No name carries `is_`, `has_`, or `get_` (2.7).

## Rules

### 2.1 — Use `snake_case` for methods, variables, symbols, file names, and directories.

**Reasoning, step by step:**
1. Ruby's entire standard library and the vast majority of gems use `snake_case` for these identifiers; deviating forces every reader to context-switch.
2. File names and directory names mirror the constant they contain in `snake_case` form: `Commerce::Order` lives in `commerce/order.rb`. Consistent mapping means `Zeitwerk` can autoload without a manual map (see [12-module-organization.md](./12-module-organization.md)).
3. Symbol names used as keyword argument keys, hash keys, and method names share the same rule — `:order_id`, not `:orderId` or `:OrderId` — so the whole identifier space is uniform.

**Enforcement:** RuboCop `Naming/MethodName`, `Naming/VariableName`, `Naming/FileName`.

### 2.2 — Use `CamelCase` for classes and modules; keep acronyms as all-caps when established.

**Reasoning, step by step:**
1. `CamelCase` (also called `PascalCase`) is Ruby's single convention for constants that name types. Every constant that is a class or module uses it, so `Order`, `LineItem`, and `Customer` are unambiguous at a glance.
2. Established acronyms stay uppercase when the broader ecosystem already treats them that way: `HTTPClient`, `XMLParser`, `SKUCatalog`. A reader who knows HTTP does not benefit from `HttpClient`; an unusual acronym (`Skus`) that would read oddly mid-word may be lowercased on a case-by-case basis.
3. The test is recognizability at the call site: prefer the form a reader of the Ruby ecosystem will expect for that abbreviation.

**Enforcement:** RuboCop `Naming/ClassAndModuleCamelCase`.

### 2.3 — Use `SCREAMING_SNAKE_CASE` for constants; carry the unit or scale in the name.

**Reasoning, step by step:**
1. `SCREAMING_SNAKE_CASE` signals "this value is fixed for the lifetime of the process." It marks the contract immediately, before the reader reaches the assignment.
2. A constant encoding a quantity must carry its unit. `CONNECT_TIMEOUT_MS`, `MAX_RETRIES`, `TAX_RATE_PCT` — the unit or scale is part of the name, not a comment, because a comment is invisible at the call site where a mistake is made.
3. Magic numbers are banned: every bare numeric literal that has a conceptual identity gets a named constant. `MINIMUM_ORDER_CENTS = 500` is clear; `500` scattered through charging logic is not.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

CONNECT_TIMEOUT_MS = T.let(5_000, Integer).freeze
MAX_LINE_ITEMS     = T.let(50,    Integer).freeze
TAX_RATE_PCT       = T.let(8,     Integer).freeze
```

**Enforcement:** RuboCop `Naming/ConstantName`; unit-in-name discipline enforced at review.

### 2.4 — One class or module per file; the filename is the `snake_case` of the constant path.

**Reasoning, step by step:**
1. A single constant per file makes the module structure navigable without an IDE: the directory tree is the namespace tree. `Commerce::Order::LineItem` is in `commerce/order/line_item.rb`.
2. Zeitwerk depends on this 1:1 mapping for autoloading. A file that defines two top-level constants will load the first and silently miss the second, causing subtle load-order bugs.
3. Keep the nesting shallow: three path segments is a smell; more than three is a problem. Deep nesting is a sign that the module structure is wrong, not that more files are needed.

Worked example:
```ruby
# commerce/order/line_item.rb
# frozen_string_literal: true
# typed: strict

module Commerce
  class Order
    class LineItem
      # ...
    end
  end
end
```

**Enforcement:** RuboCop `Naming/FileName`; Zeitwerk's eager-load verification in CI catches mismatches at runtime.

### 2.5 — Predicate methods end in `?` and return a real boolean; non-boolean methods never end in `?`.

**Reasoning, step by step:**
1. A caller reading `order.fulfilled?` has a firm contract: the return is `true` or `false`, not `nil`, not `[]`, not a string. That contract is what makes `if order.fulfilled?` and `orders.select(&:fulfilled?)` safe without a nil-guard.
2. Returning a truthy object from a `?` method breaks callers that test type equality or pass the result to a `T::Boolean` sig. Ensure every branch of a predicate lands on `!!something` or a Sorbet `T::Boolean` coercion — never a raw object.
3. A method that does not return a boolean must not end in `?`. `customer.address?` returning an `Address` or `nil` is a trap; name it `address` or `find_address` instead.

Worked example:
```ruby
sig { returns(T::Boolean) }
def valid?
  quantity > 0 && sku.active?
end
```

**Enforcement:** RuboCop `Naming/PredicateMethod`; Sorbet `T::Boolean` return type enforced by `srb tc`.

### 2.6 — Use `!` only when a non-bang counterpart exists; the bang marks the dangerous variant.

**Reasoning, step by step:**
1. A bang method is defined *relative* to its safe sibling. `save!` only makes sense because `save` exists and returns a falsy value on failure; `save!` raises instead. Without a counterpart the bang is meaningless noise.
2. "Dangerous" means: mutates the receiver in-place where the non-bang does not, or raises on failure where the non-bang returns `false`/`nil`. Both are legitimate uses; neither is present in the absence of a non-bang twin.
3. Never bang a method purely for emphasis. `charge_card!` without a `charge_card` non-raising version is a lie about the API's structure.

Worked example:
```ruby
sig { returns(T::Boolean) }
def fulfill
  return false unless inventory.reserve(line_items)
  transition_to(:fulfilled)
  true
end

sig { void }
def fulfill!
  raise FulfillmentError, "inventory unavailable" unless fulfill
end
```

**Enforcement:** Review; no automated cop covers the semantic contract, only the presence of the `!` character.

### 2.7 — No `is_`/`has_`/`get_` prefixes on any method.

**Reasoning, step by step:**
1. Ruby's `?` suffix already canonically marks a predicate. `is_empty?` is redundant: `empty?` says the same thing and is what every Rubyist expects. Layering `is_` or `has_` signals unfamiliarity with the idiom.
2. `get_` is a Java habit with no place in Ruby. A method named `order` that returns an order is its own contract; `get_order` adds a verb that states the obvious and clutters the namespace.
3. Use an effect verb — `fetch_`, `load_`, `find_` — when the operation goes to I/O or may fail. That communicates something real: `fetch_order(id)` tells the reader a network or database call happens. `get_order` tells them nothing.

**Enforcement:** RuboCop `Naming/PredicatePrefix` (catches `is_` and `has_` on boolean-returning methods); `get_` enforcement at review.

### 2.8 — Name side-effecting methods with an effect verb; name pure transforms as nouns or adjectives.

**Reasoning, step by step:**
1. A method name is a promise at the call site. A reader who sees `order.subtotal` expects a computation and no side effect; a reader who sees `charge_card` expects an external action. When the name matches the behaviour, the code is self-documenting and surprises are minimized.
2. Effect verbs that the guide endorses: `write_`, `charge_`, `fetch_`, `load_`, `send_`, `emit_`, `publish_`, `persist_`, `record_`. These all announce that something observable outside the object will happen.
3. Pure transforms — things that take input and return output without touching the world — read as nouns (`subtotal`, `full_name`) or past-participle adjectives (`normalized_sku`, `formatted_amount`). Never label them with a verb that implies action.

Worked example:
```ruby
# Pure transform — noun name
sig { returns(Money) }
def subtotal
  line_items.reduce(Money.zero) { |sum, line| sum + line.amount }
end

# Side-effecting — effect-verb name
sig { void }
def write_ledger(entry)
  LedgerStore.insert(order_id: id, entry:)
end
```

**Enforcement:** Review; the taxonomy is the checklist at code-review time.

### 2.9 — Name the binary-operator parameter `other`; name `reduce` block args mnemonically.

**Reasoning, step by step:**
1. `==`, `<=>`, `+`, `-`, and similar binary operators receive a conventional parameter named `other`. The convention is universal in Ruby; deviating forces every reader to re-learn it for your class.
2. `<<` and `[]` are exceptions: their semantics are too varied for `other` to add signal, so use a domain-appropriate name (`item`, `key`, `index`).
3. `reduce` block arguments deserve names that tell the story of the accumulation: `|running_total, line|` is plain; `|a, b|` demands the reader infer the relationship from context. Good block-arg names make the reduce legible without opening the block.

Worked example:
```ruby
sig { params(other: Money).returns(Money) }
def +(other)
  Money.new(cents: cents + other.cents)
end

sig { returns(Money) }
def total_charged
  line_items.reduce(Money.zero) { |invoice_total, line| invoice_total + line.amount }
end
```

**Enforcement:** Review.

### 2.10 — Use `_` or `_`-prefixed names for throwaway and unused variables.

**Reasoning, step by step:**
1. Ruby and RuboCop treat a variable named `_` as explicitly unused and suppress the "assigned but unused" warning. Use it for block parameters you must accept but do not use.
2. When multiple distinct discards appear in the same scope, prefix each with `_` followed by a descriptive suffix: `_index`, `_meta`. This preserves the intent signal while allowing multiple distinct bindings.
3. Never assign a real variable and leave it unused — that is noise, and likely signals a missing step or a dead path. Either use it or name it `_something` to document the deliberate discard.

Worked example:
```ruby
# Only the value matters; the key is discarded
inventory.each { |_sku, quantity| total += quantity }

# Multiple discards in one block
events.each_with_index { |event, _index| process(event) }
```

**Enforcement:** RuboCop `Lint/UnusedBlockArgument`, `Lint/UnusedMethodArgument`.

### 2.11 — Avoid nomenclature with discriminatory origins; prohibit magic numbers.

**Reasoning, step by step:**
1. Use `allowlist`/`denylist` instead of whitelist/blacklist; `primary`/`replica` instead of master/slave. These replacements are unambiguous, precise, and free of historical baggage. The Shopify guide and growing industry consensus have settled on these terms.
2. A bare numeric literal is a magic number when it has a conceptual identity beyond its arithmetic value. Extract it into a `SCREAMING_SNAKE_CASE` constant. The constant name is documentation that survives refactoring; the bare number is not.
3. True arithmetic constants (`0`, `1`, `-1`, `2` in an index expression) do not need names when their meaning is structurally obvious. Boundary values, business rules, limits, and timeouts always do.

Worked example:
```ruby
DENYLIST_COUNTRIES  = T.let(["KP", "IR"].freeze, T::Array[String]).freeze
MAX_RETRY_ATTEMPTS  = T.let(3, Integer).freeze
REPLICA_LAG_MS      = T.let(200, Integer).freeze

def retryable?(attempt_count)
  attempt_count < MAX_RETRY_ATTEMPTS
end
```

**Enforcement:** Review for terminology; RuboCop `Style/MagicNumber` (or `Lint/MagicNumber` depending on RuboCop version) for bare literals.

## Cross-references

- [01-formatting-and-tooling.md](./01-formatting-and-tooling.md) — the `rubocop-airbnb` baseline that enforces the mechanical casing rules from this chapter.
- [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md) — `T::Boolean` return types that enforce the predicate contract (2.5); `sig` blocks that make method contracts explicit.
- [04-variables-and-declarations.md](./04-variables-and-declarations.md) — `freeze` on constants (2.3); `attr_reader` naming conventions and the ban on `@@` class variables.
- [05-methods.md](./05-methods.md) — pure-by-default discipline that pairs with effect-verb naming (2.8); guard clauses and the 25-line cap.
- [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md) — `class << self` grouping and the one-class-per-file rule (2.4); `Data.define` value object naming.
- [07-ruby-idioms.md](./07-ruby-idioms.md) — `map`/`select`/`find`/`reduce` canonical names that inform mnemonic block-arg choices (2.9).
- [10-api-design.md](./10-api-design.md) — public surface naming; effect-verb taxonomy applied to service-layer APIs.
- [12-module-organization.md](./12-module-organization.md) — Zeitwerk autoloading and the filename-to-constant mapping (2.4).
