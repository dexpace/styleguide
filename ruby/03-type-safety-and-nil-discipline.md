# 03 — Type Safety & Nil Discipline

Sorbet's runtime-checked signatures are the first test suite. This chapter closes every hole through which an untyped, unproven, or `nil` value could reach domain logic: wrong types are rejected at the call site, absent values are represented honestly, and the parse boundary is the one place raw input becomes a typed value object.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  class Money < T::Struct
    extend T::Sig

    const :cents, Integer

    class << self
      extend T::Sig

      sig { params(raw: T.untyped).returns(Money) }
      def parse(raw)
        value = Integer(raw) # raises ArgumentError on non-integer input
        raise ArgumentError, "cents must be non-negative, got #{raw}" unless value >= 0

        new(cents: value) # sanctioned entry point; raw is proven valid above (3.9)
      end
    end

    sig { params(other: Money).returns(Money) }
    def add(other)
      Money.new(cents: cents + other.cents)
    end

    sig { params(factor: Integer).returns(Money) }
    def multiply(factor)
      raise ArgumentError, "factor must be non-negative, got #{factor}" unless factor >= 0

      Money.new(cents: cents * factor)
    end

    sig { returns(String) }
    def to_s
      "$#{format("%.2f", cents / 100.0)}"
    end
  end

  class LineItem < T::Struct
    extend T::Sig

    const :sku, Sku
    const :price, Money
    const :quantity, Integer
  end

  class Order < T::Struct
    extend T::Sig

    const :id, OrderId
    const :customer, Customer
    const :lines, T::Array[LineItem]
    const :status, OrderStatus
  end

  module OrderCalculator
    extend T::Sig

    sig { params(order: Order).returns(Money) }
    def self.total(order)
      raise ArgumentError, "order has no lines" if order.lines.empty?

      subtotal = order.lines.reduce(Money.new(cents: 0)) do |acc, line|
        acc.add(line.price.multiply(line.quantity))
      end

      raise ArgumentError, "subtotal must be non-negative" unless subtotal.cents >= 0

      subtotal
    end
  end
end
```

`Money.parse` is the single parse-constructor (3.9): it takes `T.untyped` input, validates once, and returns a typed, immutable value object — every downstream signature takes `Money`, so nothing re-validates. `Money`, `LineItem`, and `Order` are `T::Struct` value objects with typed `const` fields (3.11) rather than untyped hashes; crossing a boundary with a `Hash` is an unparsed input. `OrderStatus` is a `T::Enum` (3.10), not a free symbol. Every method carries a `sig` (3.2), and `T.must`/`T.unsafe` appear nowhere (3.7). `order.lines.empty?` is a guard that raises rather than returning `nil` (3.4), keeping `nil` out of the return type entirely.

## Rules

### 3.1 — Set `# typed: strict` on every file.

**Reasoning, step by step:**
1. Sorbet's level ladder ends at `strict`. At `strict`, every method must carry a `sig`, every constant must be typed, and every instance variable must be declared. Below `strict`, the checker fills gaps with `T.untyped`, which is a hole as large as the absence of Sorbet altogether.
2. `# typed: true` is a bridge level for files that cannot yet carry full signatures — it is not a resting state. Every file at `true` is a tracked deviation, recorded with a `TODO` naming who owns the migration and by what date.
3. The root README's correctness-first ordering means: if you can't type a file, that is a signal the design needs attention, not a licence to leave the safety level low.
4. `srb tc` is the typecheck gate in CI (chapter 01); it fails on any `strict`-level violation. A green CI run is a proof that no `strict` contract is broken across the repository.

**Enforcement:** `# typed: strict` is the header standard; `srb tc` in CI rejects violations. Files at `# typed: true` are tracked in the deviations ledger (see [README.md](./README.md)).

### 3.2 — Write a `sig` on every method; treat it as a precondition and postcondition in one.

**Reasoning, step by step:**
1. A `sig` does what a type annotation in a static-only system cannot: Sorbet wraps the method at runtime and checks every argument type and the return type on every call. The sig is not documentation — it is a runtime assertion on both ends of the call.
2. Because the check fires at call time, an incorrect argument is rejected at the caller, not three stack frames later. This is the Ruby parallel to TypeScript's `explicit-module-boundary-types` (chapter 05's 5.11) applied to all methods, not just exported ones.
3. A missing `sig` under `# typed: strict` is a compile error. There is no opt-out; any method without a sig breaks the typecheck gate. This guarantees the assertion density rule 8 in the README is met for type arguments on every method in the codebase.
4. Write the sig immediately above the method definition, with no blank line between them. The sig and the `def` are a unit.

Worked example:
```ruby
# good
sig { params(order: Order, discount: T.nilable(Money)).returns(Money) }
def apply_discount(order, discount)
  return order.total unless discount
  order.total.subtract(discount)
end

# bad — no sig; Sorbet treats the method as T.untyped, and every argument becomes unchecked
def apply_discount(order, discount)
  order.total.subtract(discount || Money.new(cents: 0))
end
```

**Enforcement:** `srb tc` under `# typed: strict` rejects any method without a `sig`. Code review confirms sig accuracy matches behavior.

### 3.3 — Declare every instance variable with `T.let` in `initialize`, in declaration order.

**Reasoning, step by step:**
1. Without `T.let`, Sorbet infers instance variable types from assignment sites. When an `@ivar` is assigned conditionally — in a branch, a guard, or a callback — Sorbet widens the inferred type to include `NilClass`, producing spurious `T.nilable` types that infect every reader.
2. Declaring `@ivar = T.let(value, Type)` in `initialize` pins the type to what you intend, making any later assignment that violates it a type error rather than a surprise widening.
3. Declare ivars in one consistent order so reviewers and Sorbet have a single, predictable place to find the shape of the object. This also stabilizes the object's shape for YJIT (chapter 15).
4. `T.let` does not add runtime cost beyond a single type check at assignment time in development; in production the Sorbet runtime can be configured to elide checks on internal-only ivars.

This rule governs classes that hold their own instance variables — value objects declare typed `const` fields on `T::Struct` instead (3.11), never hand-rolled ivars.

Worked example:
```ruby
# A stateful class: every ivar it will ever hold is declared in `initialize`.
class InventoryLedger
  extend T::Sig

  sig { params(starting: Integer).void }
  def initialize(starting:)
    @on_hand     = T.let(starting, Integer)
    @adjustments = T.let([], T::Array[Integer])
  end
end
```

**Enforcement:** `srb tc` under `# typed: strict` enforces typed ivar declarations. Code review confirms declaration order matches `attr_*` order.

### 3.4 — Treat `T.nilable` as a last resort; raise or return an empty collection rather than `nil`.

**Reasoning, step by step:**
1. `nil` as a sentinel value is a lie: the caller asked for a `Money` and received a `NilClass`. Every caller must then check, every check is a duplication, and every forgotten check is a `NoMethodError` in production.
2. A method that cannot produce a value has three honest choices: raise a typed error (chapter 08), return an empty collection, or return a `Result` type (a discriminated struct). None of these leak `nil` across the boundary.
3. `T.nilable(T)` is `T.any(T, NilClass)` under the hood. It forces every reader to handle both branches, propagating the uncertainty downstream. A non-nilable type makes the uncertainty impossible to propagate — the compiler refuses.
4. Legitimate uses of `T.nilable` are narrow: a field that is genuinely absent in the domain model (a `Customer` whose `middle_name` may not exist), an optional keyword argument, or an interface to an external system that emits `nil`. Each use should survive the question: "does `nil` genuinely model something in this domain?"

Worked example:
```ruby
# bad — nil leaks; every caller must nil-check before using the result
sig { params(sku: Sku).returns(T.nilable(Inventory)) }
def find_inventory(sku)
  @store[sku]
end

# good — raise on miss; the return type is non-nilable and trustworthy
sig { params(sku: Sku).returns(Inventory) }
def find_inventory(sku)
  @store.fetch(sku) { raise KeyError, "no inventory for SKU #{sku}" }
end
```

**Enforcement:** `srb tc` propagates `T.nilable` to all call sites, making every unchecked use a type error. Code review questions every new `T.nilable` return type.

### 3.5 — Use `&.` (safe navigation) only where `nil` is a legitimate, documented value.

**Reasoning, step by step:**
1. `&.` silences a `NoMethodError` on `nil`. When `nil` is not a legitimate value at that point, `&.` is a patch over a bug: it hides the nil rather than surfacing the upstream failure that produced it.
2. `&.` on a non-nilable Sorbet type is a type error: the checker knows the receiver cannot be `nil`, and it rejects the operator. This is the right outcome — the compiler is detecting a confused nil model, not a valid access pattern.
3. Reserve `&.` for receivers that are declared `T.nilable` or `T.untyped` at the boundary of external input. Once a value is parsed into a typed domain object, it should not need safe navigation.
4. The presence of `&.` on a domain object is a smell that the sig is too wide or the parse boundary is too far downstream.

Worked example:
```ruby
# bad — order is typed Order, non-nilable; &. hides a code path that should never exist
total = order&.total

# good — total is legitimately nilable here because the order has not been fetched yet
sig { params(order_id: OrderId).returns(T.nilable(Money)) }
def pending_total(order_id)
  order = Order.find_by(id: order_id) # returns T.nilable(Order) from the ORM
  order&.total
end
```

**Enforcement:** `srb tc` rejects `&.` on non-nilable types. Code review flags `&.` on domain objects that should never be nil.

### 3.6 — Use `Hash#fetch` and `Array#fetch` for keys and indices that must exist.

**Reasoning, step by step:**
1. `hash[key]` returns `nil` on a missing key with no indication that anything is wrong. The `nil` propagates silently until something calls a method on it, at which point the error is three frames from the mistake.
2. `hash.fetch(key)` raises `KeyError` immediately and locally, naming the missing key. The stack trace points to the lookup, not to the downstream `NoMethodError`. This is the Ruby expression of "parse, don't validate": turn the implicit contract (this key must exist) into an explicit runtime assertion at the point of access.
3. `fetch` with a block is the sanctioned form for providing computed defaults. `fetch` with a second argument is the form for static defaults. Neither silently returns `nil`.
4. Under Sorbet, `hash[key]` on a `T::Hash[K, V]` returns `T.nilable(V)`, forcing a nil check. `hash.fetch(key)` returns `V` — the type is non-nilable because the contract is enforced by the runtime exception.

Worked example:
```ruby
RATES = T.let({ "US" => 0.08, "EU" => 0.20 }.freeze, T::Hash[String, Float])

# bad — returns nil on unknown jurisdiction; nil propagates into arithmetic
rate = RATES[jurisdiction]

# good — raises KeyError immediately; the return type is Float, not T.nilable(Float)
rate = RATES.fetch(jurisdiction) { raise KeyError, "no tax rate for #{jurisdiction}" }
```

**Enforcement:** RuboCop `Style/HashFetch` (where available) or code review. Sorbet's `T::Hash` return types reveal every unchecked `[]` as `T.nilable`.

### 3.7 — Ban `T.must` and `T.unsafe` outside declared bridge points; every bridge use carries a why-comment.

**Reasoning, step by step:**
1. `T.must(x)` asserts to Sorbet that `x` is non-nil. The compiler believes you and removes `NilClass` from the type. If you are wrong, the runtime raises `TypeError` — but at the `T.must` call site, not where the nil was introduced. The assertion has no proof; it is a claim.
2. `T.unsafe(x)` is worse: it tells Sorbet to treat `x` as `T.untyped`, disabling all checking on the value and everything derived from it. It is the Ruby parallel to TypeScript's `any` cast.
3. Both break the invariant that Sorbet's types are trustworthy. One `T.unsafe` call opens a hole that the type system cannot see into; every downstream use of that value is effectively untyped. The type system's value is totality — a single escape undermines the guarantee for the entire call chain.
4. Bridge points are specific, named: a gem that has no Sorbet RBI types yet, a codegen boundary where types are generated after the fact, a known Sorbet inference limitation that cannot be worked around. Each bridge is in the deviations ledger with an owner and a plan to close it.

Worked example:
```ruby
# bad — T.must with no justification; the nil assumption is undocumented
price = T.must(line_item.price)

# good — if price can be nil, the sig should reflect it and the caller should handle it
# good — if price cannot be nil, fix the sig of the method that returns it

# bridge (only in declared bridge file, e.g. lib/legacy_bridge.rb):
# T.unsafe needed here because LegacyGem has no RBI and dynamic dispatch
# cannot be statically typed. Tracked: https://linear.app/dexpace/PLAT-1234
legacy_result = T.unsafe(LegacyGem.call(params))
```

**Enforcement:** code review rejects `T.must` / `T.unsafe` without a why-comment. A RuboCop custom cop or `grep` gate in CI can detect unmarked uses.

### 3.8 — Require a why-comment on every `T.cast`; it is the single sanctioned cast.

**Reasoning, step by step:**
1. `T.cast(x, T)` tells Sorbet to treat `x` as `T` at the type level, with a runtime check that the value actually is `T`. Unlike `T.unsafe`, it does raise at runtime on mismatch — but the compiler still trusts the cast rather than verifying it from surrounding code.
2. The correct pattern for `T.cast` is: validate the value immediately above the cast, then cast with a comment citing what the validation proved. This is the Ruby mirror of the TypeScript "single sanctioned `as`" (TS chapter 03, rule 3.4): the cast is the seal on a proof you just established.
3. `T.cast` without a validation above it is an unproven claim, identical in danger to `T.must`. The why-comment forces the author to state the proof — and if they cannot state it, they should not be casting.
4. Alternatives to reach for first: `case/in` pattern matching (the compiler narrows the type inside the branch), a discriminant field on a `T::Struct`, or a Sorbet `sealed!` hierarchy with exhaustive matching.

Worked example:
```ruby
# bad — cast with no justification; proof is absent
price = T.cast(raw_price, Money)

# good — validate above, then cast with a comment naming the proof
sig { params(raw: T.untyped).returns(Money) }
def coerce_price(raw)
  raise TypeError, "expected Money, got #{raw.class}" unless raw.is_a?(Money)
  T.cast(raw, Money) # sanctioned: is_a? guard above proves the type at runtime
end

# best — avoid the cast entirely by parsing at the boundary (3.9)
Money.parse(raw)
```

**Enforcement:** code review rejects `T.cast` without an accompanying validation and why-comment. A grep gate can detect `T.cast` lines without a comment on the preceding line.

### 3.9 — Parse, don't validate: one constructor takes raw input; everything downstream takes the typed value.

**Reasoning, step by step:**
1. "Parse, don't validate" means: do the work of proving a value is valid exactly once, at the boundary, and encode the proof in the type. Downstream code takes `Money`, not `Integer` with a side-comment that it's non-negative — the type is the proof, and it travels with the value forever.
2. A parse-constructor takes `T.untyped` (or the raw external type), validates completely, and returns a typed, immutable value object. It is the only place that raw input is interrogated. Every other method in the system takes the typed form and can trust it without re-checking.
3. `Money.parse` is the canonical example: it takes raw input, validates that it is a non-negative integer, and constructs a `Money`. The `Money` type carries the guarantee. A method that takes `Money` never asks "is this non-negative?" because the type already answers.
4. The parse-constructor is the single point where `T.cast` or a direct `T.let` on a newly-constructed object is sanctioned — because the validation is immediately above it. Everywhere else in the system, values arrive already typed.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

class Money < T::Struct
  extend T::Sig

  const :cents, Integer

  class << self
    extend T::Sig

    sig { params(raw: T.untyped).returns(Money) }
    def parse(raw)
      value = Integer(raw) # raises ArgumentError on non-integer input
      raise ArgumentError, "cents must be non-negative, got #{value}" unless value >= 0

      new(cents: value)
    end
  end
end

# downstream — no validation; the type carries the proof
sig { params(price: Money, quantity: Integer).returns(Money) }
def line_total(price, quantity)
  raise ArgumentError, "quantity must be positive, got #{quantity}" unless quantity > 0

  Money.new(cents: price.cents * quantity)
end
```

**Enforcement:** code review: every `T.untyped` parameter must be in a parse-constructor or a declared bridge. Domain methods taking raw scalars where a value object exists are rejected.

### 3.10 — Model closed sets with `T::Enum`; never use free-floating symbols or strings for domain states.

**Reasoning, step by step:**
1. A free symbol like `:pending` or a string like `"shipped"` is invisible to the type system. Sorbet types it as `Symbol` or `String`, meaning every value of that type is accepted — including `:pendingg` or `"shiped"`. The compiler cannot catch a typo, an impossible state, or an exhaustiveness gap.
2. `T::Enum` creates a nominal type. Sorbet knows every member at compile time and rejects any value that is not a declared member. A `case` on a `T::Enum` value can be checked for exhaustiveness by adding a final `else T.absurd(status)` branch, which fails at runtime if a new member is added without updating the switch.
3. `T::Enum` values are singleton objects, not raw strings — they cannot be confused with arbitrary user input, they are identity-comparable, and they carry no encoding ambiguity.
4. At a system boundary (JSON deserialization, database column), parse the raw string into the enum using `MyEnum.deserialize(raw)`, which raises on unknown values. The enum member is the type; the raw string is input.

Worked example:
```ruby
class OrderStatus < T::Enum
  enums do
    Pending  = new("pending")
    Paid     = new("paid")
    Shipped  = new("shipped")
    Cancelled = new("cancelled")
  end
end

sig { params(status: OrderStatus).returns(String) }
def status_label(status)
  case status
  when OrderStatus::Pending   then "Awaiting payment"
  when OrderStatus::Paid      then "Payment received"
  when OrderStatus::Shipped   then "On its way"
  when OrderStatus::Cancelled then "Cancelled"
  else T.absurd(status) # exhaustiveness: fails at runtime if a new variant is unhandled
  end
end
```

**Enforcement:** `srb tc` rejects passing a raw `Symbol` or `String` where a `T::Enum` is required. `T.absurd` in `case` exhaustiveness guards is checked at code review. See [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).

### 3.11 — Use `T::Struct` or `Data.define` for typed value objects at boundaries; a `Hash` crossing a boundary is unparsed input.

**Reasoning, step by step:**
1. A `Hash` has the type `T::Hash[K, V]` — every key is the same type and every value is the same type. A `LineItem` has typed, named fields: `sku: Sku`, `price: Money`, `quantity: Integer`. The struct is the proof that the hash has been parsed; the type names each field and enforces its type individually.
2. A `Hash` that crosses a module or method boundary carries no proof of its shape. The callee must fetch keys and hope they exist, producing `T.nilable` values and triggering nil checks that duplicate validation the caller should have done at parse time.
3. `T::Struct` integrates with Sorbet: `const` fields are typed, immutable by default, and checked at construction. `Data.define` is Ruby 3.2+'s built-in immutable value object, suitable where Sorbet integration is not required. Both are frozen, preventing mutation after construction.
4. `T::Struct` is the right choice when Sorbet must track the type. `Data.define` is acceptable for small, internal value objects in files at `# typed: true` that have not been migrated. At `# typed: strict`, reach for `T::Struct`.

Worked example:
```ruby
# bad — Hash at the boundary; callee cannot trust the shape
sig { params(line: T::Hash[Symbol, T.untyped]).returns(Money) }
def line_total(line)
  price = T.cast(line.fetch(:price), Money) # defensive cast; caller could pass anything
  price.multiply(Integer(line.fetch(:quantity)))
end

# good — T::Struct; callee trusts every field's type
class LineItem < T::Struct
  const :sku,      Sku
  const :price,    Money
  const :quantity, Integer
end

sig { params(line: LineItem).returns(Money) }
def line_total(line)
  line.price.multiply(line.quantity)
end
```

**Enforcement:** code review rejects `T::Hash[Symbol, T.untyped]` in any non-boundary method signature. `srb tc` flags every missing `const`-type declaration in `T::Struct` bodies.

### 3.12 — Type the negative space: a return type is non-nil unless `nil` genuinely models an absent domain value.

**Reasoning, step by step:**
1. Every `T.nilable` in a return type is a claim that `nil` carries domain meaning. If you cannot explain what `nil` means to a domain expert — not "the database returned nothing" but "the order has no discount because it is a full-price order" — then `nil` should not be in the return type.
2. The negative space of a type is what the type forbids. A `sig { returns(Money) }` forbids `nil`, `0`, `"zero"`, and every other non-`Money` value. The narrower the return type, the more the type system has proven and the less downstream code must check.
3. Making impossible states unrepresentable means reaching for the type that matches the domain model exactly. A method that always returns a value should declare it. A method that returns a value or raises should declare the non-nil return — the exception is not a return path from the type system's perspective. Only a method that models "value or absent" should return `T.nilable`.
4. Audit `T.nilable` return types in code review: ask whether `nil` genuinely models absence in the domain, or whether the method should raise, return an empty collection, or return a more specific type. The goal is for `T.nilable` in return position to be rare enough to be notable.

Worked example:
```ruby
# bad — nil as a sentinel for "not found"; every caller must check
sig { params(id: OrderId).returns(T.nilable(Order)) }
def find_order(id)
  @store[id]
end
# caller: order = find_order(id); return unless order; order.total ...

# good — raise on miss; return type is non-nilable; callers never handle nil
sig { params(id: OrderId).returns(Order) }
def fetch_order(id)
  @store.fetch(id) { raise KeyError, "order not found: #{id}" }
end

# also good — when "not found" is a valid domain outcome, use a result type
OrderResult = T.type_alias { T.any(Order, Symbol) } # :not_found is a domain value

sig { params(id: OrderId).returns(OrderResult) }
def find_order(id)
  @store.fetch(id, :not_found)
end
```

**Enforcement:** `srb tc` propagates every `T.nilable` return through callers, making every unchecked path a type error. Code review questions every `T.nilable` in return position.

## Cross-references

- `srb tc` as the typecheck gate, `rubocop` baseline, and `# frozen_string_literal: true`: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming, predicate `?`, and no `is_` prefix: [02-naming-conventions.md](./02-naming-conventions.md).
- `attr_reader`, `freeze` on constants, and `||=` caveats for nilable types: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Method sig as precondition, guard clauses, 2+ assertions per method, and assertion density: [05-methods.md](./05-methods.md).
- `T::Struct`, `Data.define`, `T::Enum`, `sealed!`, and making illegal states unrepresentable: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `Hash#fetch`, `case/in` pattern matching, and Enumerable pipelines over imperative loops: [07-ruby-idioms.md](./07-ruby-idioms.md).
- Typed `StandardError` subclasses, raise at the boundary, and `cause` chaining: [08-error-handling.md](./08-error-handling.md).
- `sig` on every public method as the public API contract: [10-api-design.md](./10-api-design.md).
- Testing parse-constructors and `T::Enum` exhaustiveness guards with negative-space cases: [11-testing.md](./11-testing.md).
- Stable object shapes, `T.let` in `initialize`, and YJIT monomorphism: [15-performance.md](./15-performance.md).
