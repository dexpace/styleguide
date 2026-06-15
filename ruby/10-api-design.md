# 10 — API Design

Designing the surface other code imports. A module's public API is a promise: every method you expose is a contract you keep for every caller, through every refactor, until a major version lets you break it. This chapter covers exporting the least, shaping what you do export so the call site reads as a family, and refusing to let raw or undeclared values cross the boundary in either direction.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  module Pricing
    extend T::Sig

    # Public surface: three methods. Everything else is private.

    sig { params(order: Order).returns(Money) }
    def self.subtotal(order:)
      raise ArgumentError, "order has no lines" if order.lines.empty?

      order.lines.reduce(Money.new(cents: 0)) do |acc, line|
        acc.add(line_total(line: line))
      end
    end

    sig { params(order: Order, coupon_code: String).returns(Money) }
    def self.discounted_total(order:, coupon_code:)
      base   = subtotal(order: order)
      rate   = discount_rate(coupon_code: coupon_code)
      result = base.multiply(1 - rate)

      raise ArgumentError, "discounted total must be non-negative" unless result.cents >= 0

      result.freeze
    end

    sig { params(order: Order).returns(T::Hash[String, T.untyped]) }
    def self.to_h(order:)
      {
        "subtotal_cents" => subtotal(order: order).cents,
        "line_count"     => order.lines.size,
      }.freeze
    end

    class << self
      extend T::Sig

      private

      sig { params(line: LineItem).returns(Money) }
      def line_total(line:)
        line.price.multiply(line.quantity)
      end

      sig { params(coupon_code: String).returns(Float) }
      def discount_rate(coupon_code:)
        DISCOUNT_RATES.fetch(coupon_code) do
          raise KeyError, "unknown coupon code: #{coupon_code.inspect}"
        end
      end

      DISCOUNT_RATES = T.let(
        { "SAVE10" => 0.10, "SAVE20" => 0.20 }.freeze,
        T::Hash[String, Float],
      )
    end
  end
end
```

The public surface is exactly three methods: `subtotal`, `discounted_total`, and `to_h` (10.1). Every public method carries a `sig` (10.2). Arguments are keyword-only, so call sites read as `Pricing.subtotal(order: order)` without positional guesswork (10.3). The module accepts the narrowest duck-typed input — `Order` is a value object — and returns frozen concrete values; `to_h` returns a frozen hash so callers cannot reach back into module state (10.4). Raw `String` coupon codes are resolved inside the module through a `fetch` that raises on miss, never passed deeper as untyped data (10.5). `subtotal`/`to_h` form a symmetric serialization pair: one produces the canonical value, the other emits a hash representation (10.7). Guard assertions appear before the happy path on both public methods (10.2, 10.8).

## Rules

### 10.1 — Minimize the public surface: everything that is not part of the contract is `private` or `protected`.

**Reasoning, step by step:**
1. Every public method is a promise. Once a caller depends on a method, removing or renaming it is a breaking change — it demands a major-version bump and a migration note for every consumer (10.8). A smaller surface is fewer promises to keep.
2. The correct default is private. Define a method without visibility; then ask whether any caller *outside this module* needs it. If the answer is no, add `private`. Promote to public only when a genuine external need exists.
3. This is especially sharp in Ruby because everything is public by default. Explicit `private` / `protected` declarations are the only fence between "implementation detail" and "API contract." Without them, every helper method is implicitly published.
4. A module with a large public surface invites callers to depend on internals. Each dependency you did not intend is a refactor you cannot safely do.

Worked example:

```ruby
module Inventory
  extend T::Sig

  sig { params(sku: Sku).returns(Integer) }
  def self.available_count(sku:)           # public — part of the contract
    reserved = reserved_count(sku: sku)
    on_hand(sku: sku) - reserved
  end

  class << self
    extend T::Sig

    private

    sig { params(sku: Sku).returns(Integer) }
    def on_hand(sku:)                      # private — implementation detail
      LEDGER.fetch(sku, 0)
    end

    sig { params(sku: Sku).returns(Integer) }
    def reserved_count(sku:)               # private — implementation detail
      RESERVATIONS.fetch(sku, 0)
    end
  end
end
```

**Enforcement:** code review; every non-private method in a module is treated as a published contract and reviewed as one.

### 10.2 — Write a `sig` on every public method; the sig is the written, runtime-checked contract.

**Reasoning, step by step:**
1. A public method without a `sig` is an undocumented promise. Callers must read the body to know what it accepts and what it returns; there is no machine-readable statement of the contract.
2. Under `# typed: strict`, Sorbet wraps every method that carries a `sig` and checks arguments and the return value at runtime on every call. The sig is not documentation — it is an assertion that fires on every invocation, far denser than any hand-written precondition.
3. Inference is for locals. Inside a method body, Sorbet can infer local types from assignments; that is fine, because the definition and the use are in the same file and any wrong inference is caught immediately. Across a module boundary, inferred types are invisible to callers and cannot catch a wrong argument until the call fails at runtime — the wrong runtime. The sig makes the boundary explicit.
4. `# typed: strict` rejects any method without a `sig`; this rule is therefore mechanically enforced for every method in the repository. Public methods have the added obligation of accuracy: the sig must match the semantics, not just compile.

**Enforcement:** `srb tc` under `# typed: strict` rejects any method without a `sig`. Code review confirms the sig accurately reflects the method's semantics and failure modes.

### 10.3 — Use keyword arguments on every public method; positional arguments are order-dependent and backward-incompatible to extend.

**Reasoning, step by step:**
1. A positional argument is a position: callers must know the order, and adding a new required positional anywhere but the end is a breaking change. A keyword argument is a name: callers pass it by name, order is irrelevant, and adding a new keyword with a default is always backward-compatible.
2. Call sites with keyword arguments are self-documenting. `Pricing.discounted_total(order: order, coupon_code: code)` names each argument where it is passed; `Pricing.discounted_total(order, code)` is a positional guessing game the reader resolves by opening the signature.
3. Keyword arguments also prevent transposition bugs. `process(customer, order)` and `process(order, customer)` compile identically when both arguments share a type; `process(customer:, order:)` makes the swap a `NoMethodError` before a test runs.
4. The only sanctioned positional argument is the implicit `self` receiver. All other arguments are keywords. For existing APIs, adding keyword alternatives before removing positionals satisfies the no-weakening principle (see 10.8).

Worked example:

```ruby
# bad — positional; adding a third arg breaks every call site
sig { params(order: Order, discount: Money).returns(Money) }
def self.apply_discount(order, discount) = ...

# good — keyword; backward-compatible to extend with new optional keywords
sig { params(order: Order, discount: Money).returns(Money) }
def self.apply_discount(order:, discount:) = ...
```

**Enforcement:** RuboCop `Style/ArgumentsForwarding` and code review; public method signatures using positional arguments (other than the receiver) are rejected.

### 10.4 — Accept the narrowest duck-typed interface; return a concrete frozen value.

**Reasoning, step by step:**
1. A method's input contract should be the narrowest shape it actually uses. A method that reads an order's lines and status should accept anything that responds to `#lines` and `#status` — not a concrete `Order` class, if a test double or a lightweight struct also satisfies the usage. The narrower the input contract, the easier the method is to call from tests and from future callers.
2. Return the opposite: the fully-known, concrete thing the method actually produces, frozen before it leaves. Returning a wide type (an interface, an abstract module, a `T::Hash[Symbol, T.untyped]`) pushes a narrowing burden onto every call site. Return the concrete type; let the sig name it.
3. Freeze every collection, hash, and struct before returning it. Callers must not be able to mutate your return value and corrupt the module's internal state. `frozen?` being `true` on a returned collection is a runtime guarantee that the module's state is not shared.
4. This rule pairs with 3.9 (parse, don't validate): accept parsed typed values in, return frozen typed values out. The module is never the place where raw input is first interrogated.

Worked example:

```ruby
# bad — accepts more than needed; couples callers to a concrete class
sig { params(order: Order).returns(T::Array[Money]) }
def self.line_prices(order:)
  order.lines.map(&:price) # returns a mutable array
end

# good — accepts duck type; returns frozen concrete collection
sig { params(lines: T::Array[LineItem]).returns(T::Array[Money]) }
def self.line_prices(lines:)
  lines.map(&:price).freeze
end
```

**Enforcement:** code review; returned collections must call `.freeze` before they leave a public method; input types are widened to duck-typed modules where possible (see [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md)).

### 10.5 — Parse at the boundary: validate raw input once into a value object; never pass a raw `Hash` or scalar deeper into the system.

**Reasoning, step by step:**
1. Raw input — a `Hash` from JSON, a `String` from a form param, an `Integer` from a query string — is unproven. Every method that receives it must either trust it blindly or duplicate validation. Duplication means divergent rules, silent bugs, and a codebase that checks the same invariant in ten places and gets it wrong in two.
2. Parse once, at the entry point, into a typed value object (`Money.parse`, a `T::Struct`, a `Data.define` record). Every downstream method takes the typed form and can trust it without re-checking. The type is the proof; it travels with the value (see 3.9).
3. A `Hash` crossing a module boundary is a failure to parse. The callee cannot see the hash's shape without reading the caller; the sig lies (`T::Hash[Symbol, T.untyped]` is not a contract, it is an acknowledgement of defeat). Replace it with a `T::Struct` or `Data.define` that names and types each field.
4. Parse-constructors are the only place where `T.untyped` appears in a signature. They are named `parse` by convention, take raw input, validate fully, and return a typed immutable value. Every other method in the system takes the typed form.

Worked example:

```ruby
# bad — raw Hash passes through; callee must fetch and trust
sig { params(raw: T::Hash[Symbol, T.untyped]).returns(Money) }
def self.total_from_hash(raw:)
  price    = T.cast(raw.fetch(:price), Integer)
  quantity = T.cast(raw.fetch(:quantity), Integer)
  Money.new(cents: price * quantity)
end

# good — parse at the boundary; downstream takes typed LineItem
sig { params(raw: T.untyped).returns(LineItem) }
def self.parse_line_item(raw:)
  LineItem.new(
    sku:      Sku.parse(raw.fetch("sku")),
    price:    Money.parse(raw.fetch("price_cents")),
    quantity: Integer(raw.fetch("quantity")),
  )
end

sig { params(line: LineItem).returns(Money) }
def self.line_total(line:)
  line.price.multiply(line.quantity)   # no validation; the type carries the proof
end
```

**Enforcement:** code review rejects `T::Hash[Symbol, T.untyped]` in any non-boundary method signature; `T.untyped` in a sig is permitted only in named `parse` constructors (see [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md)).

### 10.6 — Do not return `nil` from a public method to mean "absent" where an empty collection or a raised error is clearer; when `nil` is legitimate, type it `T.nilable` and document it.

**Reasoning, step by step:**
1. `nil` as a sentinel is a lie: the caller asked for a `Money` and received `NilClass`. Every caller must then check; every forgotten check is a `NoMethodError` in production. A method that cannot produce a value has three honest choices: raise a typed error, return an empty collection, or return a discriminated type. None of these choices is `nil`.
2. An empty collection is nearly always clearer than `nil` for "no results." `[].each` is safe; `nil.each` is not. A caller iterating results never needs to nil-check the collection itself.
3. When raising is the right choice — the absence is exceptional, not normal — raise a typed `StandardError` subclass with a message. The caller that wants to handle the absence catches the specific class; the caller that treats it as a bug lets it propagate (see [08-error-handling.md](./08-error-handling.md)).
4. `T.nilable` in a return type is a published claim that `nil` carries domain meaning. Audit every occurrence: if you cannot explain what `nil` means to a domain expert — not "the query returned nothing" but "the order has no applied coupon" — then `nil` should not be in the return type. Where it is legitimate, document what `nil` means in a YARD comment above the `sig`.

Worked example:

```ruby
# bad — nil as sentinel; every caller nil-checks before using
sig { params(order_id: OrderId).returns(T.nilable(Order)) }
def self.find(order_id:)
  STORE[order_id]
end

# good — raise on miss; return type is non-nilable
sig { params(order_id: OrderId).returns(Order) }
def self.fetch(order_id:)
  STORE.fetch(order_id) { raise KeyError, "order not found: #{order_id}" }
end

# also good — nil is a domain value (coupon is genuinely optional)
# @return [T.nilable(Money)] the applied discount, or nil if no coupon was applied
sig { params(order: Order).returns(T.nilable(Money)) }
def self.applied_discount(order:)
  return nil unless order.coupon_code
  Money.parse(DISCOUNT_TABLE.fetch(order.coupon_code))
end
```

**Enforcement:** code review questions every `T.nilable` in a return type; `srb tc` propagates `T.nilable` to all call sites, making every unchecked use a type error (see [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md)).

### 10.7 — Keep API symmetry: paired operations share names and shapes.

**Reasoning, step by step:**
1. Operations that do parallel things should read as variations on one form. When `encode`/`decode`, `parse`/`to_h`, and `open`/`close` share naming patterns, argument shapes, and return conventions, a caller learns the family once and predicts the rest. Asymmetry — `parse` on one side and `serialize_to_hash` on the other — forces the caller to relearn each method.
2. Symmetry is concrete: paired verbs (`parse`/`to_h`, `encode`/`decode`, `acquire`/`release`), the same keyword argument names across a family, the same return type for parallel operations, and consistent error contracts (if `parse` raises `ArgumentError`, `encode` raises `ArgumentError` too, not `RuntimeError`).
3. Asymmetry that exists for a reason — a one-way operation, a method that has no inverse — is documented as such. The absence of a pair is a conscious design decision, not an oversight.
4. Name symmetry extends to resource pairs: if a module exposes `open`, it exposes `close`; if it exposes `begin_transaction`, it exposes `commit` and `rollback`. Callers should never be left holding a resource with no documented way to release it.

Worked example:

```ruby
module Serializer
  extend T::Sig

  sig { params(raw: T.untyped).returns(LineItem) }
  def self.parse(raw:)                    # inbound: raw → typed
    LineItem.new(
      sku:      Sku.parse(raw.fetch("sku")),
      price:    Money.parse(raw.fetch("price_cents")),
      quantity: Integer(raw.fetch("quantity")),
    )
  end

  sig { params(line: LineItem).returns(T::Hash[String, T.untyped]) }
  def self.to_h(line:)                    # outbound: typed → raw — symmetric verb pair
    {
      "sku"         => line.sku.to_s,
      "price_cents" => line.price.cents,
      "quantity"    => line.quantity,
    }.freeze
  end
end
```

**Enforcement:** code review; the API surface is reviewed as a family — verb pairing, argument shape, and error contract consistency are checked together.

### 10.8 — Deprecate deliberately: mark with a runtime warning and a `@deprecated` YARD tag; remove only on the next major version; version by semver.

**Reasoning, step by step:**
1. A public method cannot simply vanish — removing it breaks every caller with a `NoMethodError` they discover at runtime. Removal is a two-release process: mark the old method `@deprecated`, keep it as a thin delegation for one full major cycle, then delete it on the next major bump.
2. The `@deprecated` YARD tag is load-bearing: IDEs and documentation tools surface it at every call site. Pair it with a `warn` in the method body so callers see the warning in logs even if they do not read documentation. The warning message names the replacement and the removal version.
3. The version math is semver, non-negotiable. Removing or renaming a public method, narrowing a parameter type, or changing return semantics is a breaking change and demands a MAJOR bump. Adding a new optional keyword argument or a new method is a MINOR. A bug fix that preserves the contract is a PATCH.
4. A `@deprecated` tag without a named replacement and removal version is a dead end: it tells the caller to leave but not where to go or when the door closes.

Worked example:

```ruby
# @deprecated Use {Pricing.discounted_total} instead. Removed in v3.0.
sig { params(order: Order, code: String).returns(Money) }
def self.apply_coupon(order:, code:)
  warn "[DEPRECATED] Pricing.apply_coupon is deprecated and will be removed in v3.0. " \
       "Use Pricing.discounted_total(order:, coupon_code:) instead. " \
       "Called from #{caller_locations(1, 1)&.first}"
  discounted_total(order: order, coupon_code: code)
end
```

**Enforcement:** code review; every removal from the public surface must be preceded by a `@deprecated` cycle and gated behind a MAJOR version bump.

### 10.9 — Defaults follow the library's documented defaults; callers pass only what differs.

**Reasoning, step by step:**
1. Every optional keyword argument has a default. That default lives in exactly one place — the method signature — and is documented with a `@param` YARD note naming the default value. The caller passes only the keyword arguments that differ from the defaults; the zero-override call is always valid.
2. A caller should not need to supply every keyword to produce the ordinary result. APIs that require long keyword lists to get standard behavior shift the cost of default-management onto every call site, duplicating the default across the codebase and creating drift when the default changes.
3. Defaults must be stable. Changing a default is a behavior change for every caller who relied on the old default without passing it explicitly; in a library, this is a breaking change (MAJOR). Choose defaults carefully the first time. Document them in the sig's YARD note so callers understand what they inherit.
4. Keyword arguments with defaults must come after required keywords in the signature. Required keywords state what the caller must always supply; optional keywords with defaults state what the caller may override. Mixing the two without separating required from optional makes the signature harder to scan.

Worked example:

```ruby
# @param page_size [Integer] records per page. Default: 50.
# @param max_pages [Integer] pages to fetch before stopping. Default: 10.
sig { params(customer: Customer, page_size: Integer, max_pages: Integer).returns(T::Array[Order]) }
def self.recent_orders(customer:, page_size: 50, max_pages: 10)
  raise ArgumentError, "page_size must be positive, got #{page_size}" unless page_size > 0
  raise ArgumentError, "max_pages must be positive, got #{max_pages}" unless max_pages > 0

  fetch_pages(customer: customer, page_size: page_size, max_pages: max_pages).freeze
end

# common call — no overrides needed; defaults do the right thing
orders = Pricing.recent_orders(customer: customer)

# override call — only what differs from the defaults
bulk = Pricing.recent_orders(customer: customer, page_size: 200)
```

**Enforcement:** code review confirms that `@param` YARD notes name the default and that the default in the implementation matches the documentation (see [14-documentation.md](./14-documentation.md)).

### 10.10 — Keep argument shapes stable: required keywords first, keywords with defaults after; never reorder a published signature.

**Reasoning, step by step:**
1. Keyword argument order is part of the signature's readability contract, even though Ruby does not enforce it at the call site. Consistent ordering — required first, optional with defaults after — lets a reader scan any signature in the codebase and immediately locate what is mandatory.
2. Reordering keywords in a published signature is a confusing change even though it is not technically breaking for keyword arguments: existing call sites continue to compile, but anyone reading the old documentation or the old `sig` is now looking at a different layout than the new code. Reordering is a code-review rejection.
3. Required positionals do not exist on public methods in this guide (10.3). The only ordering question is among keyword arguments: required keywords first (no default), then optional keywords (with defaults). This is a stable, predictable contract.
4. Adding a new required keyword to a published signature is a breaking change even with keyword arguments, because existing callers that do not pass it will raise `ArgumentError`. Add new required keywords only with a MAJOR bump, or make them optional with a default that preserves existing behavior.

Worked example:

```ruby
# original published signature: required keyword first, optional with default after
sig { params(order: Order, currency: String).returns(Money) }
def self.convert(order:, currency: "USD") = ...

# bad — adding a required keyword breaks every caller that does not pass it
sig { params(order: Order, currency: String, rounding: Symbol).returns(Money) }
def self.convert(order:, rounding:, currency: "USD") = ...  # rounding is required; existing callers break

# good — new keyword is optional with a default; existing callers unaffected
sig { params(order: Order, currency: String, rounding: Symbol).returns(Money) }
def self.convert(order:, currency: "USD", rounding: :half_up) = ...
```

**Enforcement:** code review; published signatures are treated as contracts — reordering or adding a required keyword without a MAJOR bump is a rejection.

### 10.11 — Define value-object protocol methods; `Data.define` supplies most — do not hand-roll what it gives.

**Reasoning, step by step:**
1. A value object that does not define `==` is dangerous: two objects with identical fields compare unequal by identity, producing wrong results in `select`, `include?`, `uniq`, and every other equality-dependent operation. `hash` must be consistent with `==` or the object breaks as a `Hash` key. `to_h` enables round-trip serialization and interop. `to_s` makes debugging readable rather than showing `#<LineItem:0x…>`.
2. `Data.define` provides `==`, `hash`, `to_h`, `to_s`, `members`, and frozen instances for free. Using it means the protocol is correct, consistent, and tested by the Ruby core team. Hand-rolling these methods is error-prone — an off-by-one in `hash`, a missed field in `==` — and is a violation of rule 11 in the README (zero technical debt).
3. At `# typed: strict` with Sorbet, `T::Struct` with `const` fields is the preferred alternative: it provides Sorbet-typed fields, runtime type checking, and immutability. It also provides `==` and `to_h`. Choose `T::Struct` when Sorbet must track the field types; `Data.define` when a lightweight, stdlib-only value is sufficient.
4. Do not define protocol methods (`==`, `hash`, `to_h`, `to_s`) that `Data.define` or `T::Struct` already supply unless the custom behavior differs meaningfully from the generated one. If the generated `to_s` is insufficient for production logging, override only `to_s` and document why.

Worked example:

```ruby
# bad — hand-rolled value object; == and hash are error-prone and untested by the platform
class LineItem
  extend T::Sig

  sig { params(sku: Sku, price: Money, quantity: Integer).void }
  def initialize(sku:, price:, quantity:)
    @sku      = sku
    @price    = price
    @quantity = quantity
    freeze
  end

  sig { params(other: T.untyped).returns(T::Boolean) }
  def ==(other)
    other.is_a?(LineItem) && other.sku == @sku && other.price == @price # forgot quantity
  end
end

# good — T::Struct supplies ==, hash, to_h, and typed fields for free
class LineItem < T::Struct
  extend T::Sig

  const :sku,      Sku
  const :price,    Money
  const :quantity, Integer
end
```

**Enforcement:** code review rejects hand-rolled `==`/`hash` on value objects that could use `Data.define` or `T::Struct`; any override of a generated protocol method must carry a why-comment explaining the deviation (see [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md)).

## Cross-references

- `sig` on every method, `# typed: strict`, runtime checking, and `T.nilable` discipline: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `rubocop-airbnb` baseline, `srb tc` as the typecheck gate, and column/indent rules: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming, predicate `?`, and no `is_`/`get_` prefixes for public names: [02-naming-conventions.md](./02-naming-conventions.md).
- Keyword arguments in methods, guard clauses, 25-line cap, and 2+ assertions per method: [05-methods.md](./05-methods.md).
- `Data.define`, `T::Struct`, `T::Enum`, `class << self`, and making illegal states unrepresentable: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Typed `StandardError` subclasses, raise at the boundary, and `cause` chaining for error contracts: [08-error-handling.md](./08-error-handling.md).
- Zeitwerk autoloading, one class per file, and circular-dependency prevention: [12-module-organization.md](./12-module-organization.md).
- YARD on public API, `@param`/`@return`/`@deprecated` tags, and why-comments: [14-documentation.md](./14-documentation.md).
