# 06 — Classes & Data Modeling

Model domain state as immutable value objects and modules of functions; reserve a stateful class for the narrow category of lifecycle resources you open and close. Every structural choice here follows from one root question: does this thing have a lifecycle, or is it data?

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  # Value object — Data.define gives ==, hash, to_h, with, and frozen by default (6.3).
  # Parse-constructor is the single entry point; invalid Money cannot exist (6.4).
  class Money < T::Struct
    extend T::Sig

    const :cents,    Integer
    const :currency, String, default: "USD" # ISO 4217 code; defaulted so cents-only construction stays valid

    class << self
      extend T::Sig

      sig { params(raw: T.untyped).returns(Money) }
      def parse(raw)
        value = Integer(raw) # raises ArgumentError on non-integer input
        raise ArgumentError, "cents must be non-negative, got #{value}" unless value >= 0

        new(cents: value) # sanctioned: both guards above prove validity (6.4)
      end
    end

    sig { params(other: Money).returns(Money) }
    def add(other)
      Money.new(cents: cents + other.cents)
    end

    sig { returns(String) }
    def to_s
      "$#{format("%.2f", cents / 100.0)}"
    end
  end

  # Closed set — T::Enum; free symbols are invisible to the type system (6.8).
  class OrderStatus < T::Enum
    enums do
      Pending   = new("pending")
      Paid      = new("paid")
      Cancelled = new("cancelled")
    end
  end

  # Data object — T::Struct; no mutable setters, Sorbet tracks every field (6.3, 6.10).
  class Order < T::Struct
    extend T::Sig

    const :id,       String
    const :lines,    T::Array[LineItem]
    const :status,   OrderStatus
    const :paid_at,  T.nilable(Time)   # nil only on non-Paid statuses — genuine domain absence
  end

  # Module of functions — no lifecycle, so no class (6.2).
  module OrderCalculator
    extend T::Sig

    class << self
      extend T::Sig

      sig { params(order: Order).returns(Money) }
      def total(order)
        raise ArgumentError, "order has no lines" if order.lines.empty?

        subtotal = order.lines.reduce(Money.new(cents: 0)) do |acc, line|
          acc.add(line.price.multiply(line.quantity))
        end

        raise ArgumentError, "subtotal must be non-negative" unless subtotal.cents >= 0

        subtotal
      end

      sig { params(order: Order).returns(String) }
      def status_label(order)
        case order.status
        when OrderStatus::Pending   then "Awaiting payment"
        when OrderStatus::Paid      then "Payment received"
        when OrderStatus::Cancelled then "Cancelled"
        else T.absurd(order.status) # exhaustiveness: breaks at runtime if a variant is unhandled
        end
      end
    end
  end
end
```

`Money` is a `T::Struct` carrying `cents` and a defaulted `currency`, with a single `parse` class method in a `class << self` block — the one sanctioned entry point that validates and constructs, so an invalid instance cannot be represented (6.4, 6.7). `OrderStatus` is a `T::Enum`; a free symbol `:pending` would be invisible to Sorbet and omit exhaustiveness checking (6.8). `Order` has only `const` fields; no setter can break an invariant after construction (6.10). `paid_at` is `T.nilable(Time)` because `nil` has genuine domain meaning: an order that has not been paid has no `paid_at` — not a missing value but an impossible one for that state. `OrderCalculator` is a module of class methods in a single `class << self` block — no lifecycle, no instance state, no class (6.2, 6.7). The `case` in `status_label` is closed by `T.absurd`, so adding an `OrderStatus` member without updating the switch is a runtime failure at the first unhandled call (6.8).

## Rules

### 6.1 — Data and functions, not objects; reserve classes for lifecycle resources.

**Reasoning, step by step:**
1. Root rule 1 is "data and functions, not objects." A domain record is pure data: give it a typed value object (`T::Struct`, `Data.define`) and group its transformations into a module of functions. The module takes the data as an argument and returns new data — no `self`, no hidden state, trivially testable.
2. A class earns its keep only when an instance owns a lifecycle: something you open and must close, or that holds mutable runtime state behind an invariant — a database connection, a thread pool, a socket, a file handle. The test is whether `open`/`close` (or `start`/`stop`) are meaningful. `Money` and `Order` do not pass the test; a `ConnectionPool` does.
3. A class that holds only class methods and no instance state is a module written in the wrong idiom. A class that holds only data and no mutating instance methods is a value object — model it with `T::Struct` or `Data.define` (6.3), not a hand-rolled class.
4. Keeping data and behaviour separate makes the two independently testable and composable: functions compose cleanly because their inputs and outputs are typed values, not entangled object graphs.

Worked example:
```ruby
# bad — Order is data; wrapping it in a class that holds only class methods is noise
class OrderService
  def self.total(order)
    order.lines.reduce(Money.new(cents: 0)) { |acc, line| acc.add(line.price) }
  end
end

# good — module of functions; same logic, honest idiom (6.2)
module OrderCalculator
  class << self
    extend T::Sig

    sig { params(order: Order).returns(Money) }
    def total(order)
      order.lines.reduce(Money.new(cents: 0)) { |acc, line| acc.add(line.price) }
    end
  end
end
```

**Enforcement:** Code review rejects a class that has no instance state and no lifecycle. RuboCop `Metrics/ClassLength` surfaces god objects; a zero-instance-method class that holds only `def self.*` is a module in disguise.

### 6.2 — Prefer a module of functions over a class that holds only class methods; use `extend self` only for simple utility modules.

**Reasoning, step by step:**
1. Shopify's Ruby guide is direct: a class whose public interface is entirely class methods is a module. Classes carry the overhead of an implicit `Class` hierarchy, a `new` constructor you must suppress, and `inherited` hooks — none of which a namespace of functions needs.
2. `module Foo; extend self; end` makes every instance method callable as a module method, which is correct for simple utility namespaces. For modules that need `sig` on class methods, a single `class << self; extend T::Sig; end` block is clearer — it makes `private` work for class-level methods and collects them in one place (6.7).
3. `module_function` creates a private instance method and a public module-level copy. The resulting dual presence is confusing under Sorbet, which cannot resolve the module-function arity unambiguously. Prefer `extend self` or `class << self` — one idiom, one declaration.
4. A module of functions is also the composition primitive: modules `include`/`prepend`/`extend` cleanly; a class-method bag composes only by inheritance or by calling it directly, both of which introduce coupling.

Worked example:
```ruby
# bad — class pretending to be a namespace of functions
class SkuFormatter
  def self.normalize(sku)
    sku.upcase.tr("-", "_")
  end
end

# good — module; honest idiom, no suppressed constructor
module SkuFormatter
  extend T::Sig

  class << self
    extend T::Sig

    sig { params(sku: String).returns(String) }
    def normalize(sku)
      sku.upcase.tr("-", "_")
    end
  end
end
```

**Enforcement:** Code review rejects a class with only class methods. RuboCop `Style/ModuleFunction` governs `module_function`; the project default is `extend self` or `class << self`.

### 6.3 — Use `Data.define` for immutable value objects; use `T::Struct` when Sorbet must track the type; never a mutable record where either works.

**Reasoning, step by step:**
1. Ruby 3.2+'s `Data.define` constructs a frozen value object with `==`, `hash`, `to_h`, and `with` generated for free. It is the lightest, most idiomatic immutable record: no boilerplate, no explicit `freeze`, keyword constructor enforced.
2. Under `# typed: strict`, `Data.define` fields are not individually typed by Sorbet — the accessor returns `T.untyped`. Use `T::Struct` with `const` fields when Sorbet must track each field's type. At `# typed: strict`, `T::Struct` is the default; `Data.define` is acceptable only in `# typed: true` files that have not yet been migrated.
3. `Struct` with `keyword_init: true` is permitted only when you need `Struct`-specific behaviour (`members`, positional construction, subclassing for test doubles). Use it with `keyword_init: true`; never without it. Never a plain `Struct` where `Data.define` or `T::Struct` suffices — `Struct` fields are mutable by default.
4. A hand-rolled class with `attr_reader` and a `freeze` call in `initialize` is acceptable only when the class adds methods beyond data access. For pure data, `T::Struct` or `Data.define` is always shorter and always correct.

Worked example:
```ruby
# bad — mutable Struct; fields can be reassigned after construction
LineItem = Struct.new(:sku, :price, :quantity)

# acceptable at # typed: true — Data.define; frozen, ==, hash, with for free
LineItem = Data.define(:sku, :price, :quantity)
item = LineItem.new(sku: "SKU-1", price: Money.new(cents: 100), quantity: 2)
discounted = item.with(price: Money.new(cents: 80)) # returns a new instance; original unchanged

# preferred at # typed: strict — T::Struct; Sorbet tracks each field's type
class LineItem < T::Struct
  const :sku,      Sku
  const :price,    Money
  const :quantity, Integer
end
```

**Enforcement:** RuboCop `Style/MutableConstant` and code review reject mutable `Struct` fields. Sorbet `srb tc` rejects `T.untyped` field access in `# typed: strict` files; use `T::Struct` there.

### 6.4 — Write parse-don't-validate constructors that return frozen, valid instances.

**Reasoning, step by step:**
1. "Parse, don't validate" means: interrogate raw input exactly once, at the boundary, and encode the proof in the type. A `Money.parse` that takes `T.untyped`, validates completely, and returns a `Money` makes every downstream signature take `Money` — and `Money` already carries the non-negative guarantee. Downstream code never re-checks.
2. The parse-constructor is the single entry point. Make `new` private (or lean on `T::Struct`'s constructor, which Sorbet tracks) so no caller bypasses the parse step and constructs an invalid instance.
3. Raise `ArgumentError` with a message that names the offending value and the violated constraint. The exception message is the first diagnostics signal when bad data enters the system; make it actionable.
4. Pair assertions: check the constraint positively, then check it negatively where the two derivations are independent. A postcondition inside the constructor confirms the object came out valid even if the validation logic is more complex than it looks.

Worked example:
```ruby
class Money < T::Struct
  extend T::Sig

  const :cents, Integer

  class << self
    extend T::Sig

    sig { params(raw: T.untyped).returns(Money) }
    def parse(raw)
      value = Integer(raw) # raises ArgumentError on non-integer input
      raise ArgumentError, "cents must be non-negative, got #{value}" unless value >= 0

      result = new(cents: value) # sanctioned: both guards above prove validity (3.9)
      raise "postcondition: parsed cents #{result.cents} != validated #{value}" unless result.cents == value

      result
    end
  end
end

# downstream — no guards needed; the type carries the proof
sig { params(price: Money, quantity: Integer).returns(Money) }
def line_total(price, quantity)
  raise ArgumentError, "quantity must be positive, got #{quantity}" unless quantity > 0

  Money.new(cents: price.cents * quantity)
end
```

**Enforcement:** Code review rejects domain methods whose parameter types are raw `Integer`, `String`, or `T.untyped` where a value object exists. Parse-constructor coverage is verified by testing it with negative-space inputs (chapter 11).

### 6.5 — Compose via `include`/`prepend`/`extend` and `Forwardable`; subclass only for `StandardError` trees.

**Reasoning, step by step:**
1. Root rule 5: composition over inheritance. A subclass is permanently coupled to its parent's internals and its method resolution order; every parent change is a survey of descendants. The language supports deep inheritance; the guide does not.
2. For behavioural reuse, `include` a module. The module's methods become instance methods of the includer; they know nothing about the class they land in beyond the methods they call. That dependency is explicit in the module's method calls, not hidden in a parent.
3. For delegation — wrapping another object and forwarding a subset of its calls — use Ruby's `Forwardable` and `def_delegators`. The forwarded methods are enumerated explicitly; the caller sees exactly the surface you chose to expose, not the full interface of the wrapped object.
4. Inheritance is sanctioned for `StandardError` subclasses (chapter 08) because `rescue` and `is_a?` dispatch are built on the class hierarchy and there is no practical substitute.

Worked example:
```ruby
# bad — inheriting InventoryBase for code reuse; every parent change risks breaking Inventory
class Inventory < InventoryBase
  def allocate(sku, qty)
    super.tap { audit(sku, qty) }
  end
end

# good — include a module for shared behaviour; delegate to a collaborator for borrowed calls
module Auditable
  extend T::Sig

  sig { params(sku: Sku, quantity: Integer).void }
  def audit(sku, quantity)
    AuditLog.record(resource: self.class.name, sku: sku, quantity: quantity)
  end
end

class Inventory
  extend T::Sig
  include Auditable
  extend Forwardable

  def_delegators :@store, :fetch, :keys

  sig { params(store: T::Hash[Sku, Integer]).void }
  def initialize(store:)
    @store = T.let(store, T::Hash[Sku, Integer])
  end

  sig { params(sku: Sku, quantity: Integer).void }
  def allocate(sku, quantity)
    current = @store.fetch(sku) { raise KeyError, "unknown SKU: #{sku}" }
    raise ArgumentError, "insufficient stock for #{sku}" unless current >= quantity

    @store[sku] = current - quantity
    audit(sku, quantity)
  end
end
```

**Enforcement:** Code review rejects `<` unless the parent is a `StandardError` subclass, one of Sorbet's structural base types (`T::Struct`, `T::Enum` — the sanctioned way to model value objects and closed sets), or a Sorbet abstract/sealed module. RuboCop `Style/Delegation` encourages `def_delegators` over hand-rolled forwarding.

### 6.6 — Never use `@@` class variables; use a class instance variable for class-level state.

**Reasoning, step by step:**
1. `@@var` is shared across the entire inheritance hierarchy — the class that declares it and every subclass, including subclasses added after the fact in tests or plugins. A write in any subclass mutates the ancestor's slot. This is the widest-scoped mutable state in the language, and it is invisible at the assignment site.
2. A class instance variable (`@var` on the class object itself, accessed inside `class << self`) is scoped to exactly one class. Subclasses inherit the reader if you define one with `attr_reader` on the singleton, but each class has its own slot — writes do not propagate upward or sideways.
3. Under Sorbet `# typed: strict`, `@@var` cannot be typed precisely; `srb tc` infers it as `T.untyped` or requires a workaround. A class instance variable declared with `T.let` in the singleton is fully typed.
4. If the real intent is a process-global registry, use an explicit singleton module (`Module.new { extend self; ... }`) with documented mutability. The explicitness makes the shared state visible in the call graph.

Worked example:
```ruby
# bad — @@registry is shared with every subclass; a plugin that subclasses corrupts it
class SkuRegistry
  @@registry = {}

  def self.register(sku, handler)
    @@registry[sku] = handler
  end
end

# good — class instance variable; scoped to SkuRegistry only
class SkuRegistry
  extend T::Sig

  @registry = T.let({}, T::Hash[String, T.untyped])

  class << self
    extend T::Sig

    sig { params(sku: String, handler: T.untyped).void }
    def register(sku, handler)
      @registry[sku] = handler
    end

    sig { params(sku: String).returns(T.untyped) }
    def fetch(sku)
      @registry.fetch(sku) { raise KeyError, "no handler for SKU: #{sku}" }
    end
  end
end
```

**Enforcement:** RuboCop `Style/ClassVars` bans `@@`. Sorbet `srb tc` flags untyped class variables under `# typed: strict`.

### 6.7 — Group all class methods in a single `class << self` block.

**Reasoning, step by step:**
1. `class << self` opens the singleton class, giving `private` its expected scoping: `private :method_name` inside the block makes that class method private, which `def self.method` cannot do without a second `private_class_method` call.
2. A single block collects all class-level methods in one readable section. Scattered `def self.*` declarations interleaved with instance methods require the reader to mentally partition the file; the block makes the partition structural.
3. This is a recorded deviation from Airbnb's guide (which uses `def self.method`), aligned with Shopify's preference. The deviation is in the README deviations ledger.
4. Add `extend T::Sig` as the first line inside the block so Sorbet can attach `sig` blocks to class methods. Omitting it causes `srb tc` to flag every `sig` inside the block as unresolved.

Worked example:
```ruby
class Order < T::Struct
  extend T::Sig

  const :id,     String
  const :status, OrderStatus

  class << self
    extend T::Sig

    sig { params(id: String).returns(Order) }
    def pending(id)
      new(id: id, status: OrderStatus::Pending)
    end

    sig { params(raw: T::Hash[Symbol, T.untyped]).returns(Order) }
    def from_hash(raw)
      new(
        id:     raw.fetch(:id),
        status: OrderStatus.deserialize(raw.fetch(:status)),
      )
    end

    private

    sig { params(id: String).returns(T::Boolean) }
    def valid_id?(id)
      !id.empty?
    end
  end
end
```

**Enforcement:** RuboCop `Style/ClassMethodsDefinitions` (configured to `class << self`); code review rejects `def self.*` except inside a `class << self` block. This deviation is recorded in [README.md](./README.md).

### 6.8 — Make illegal states unrepresentable: `T::Enum` for closed sets, sealed abstract modules for closed hierarchies, `T.absurd` to exhaust them.

**Reasoning, step by step:**
1. A free symbol `:pending` or a string `"shipped"` is invisible to Sorbet. The type is `Symbol` or `String`; every value of that type is accepted, including typos and values added by future states. `T::Enum` creates a nominal type: Sorbet knows every member at compile time and rejects any non-member.
2. Close a `case` on a `T::Enum` or sealed hierarchy with `else T.absurd(value)`. Because every handled branch narrows the subject, the value reaching `else` has type `T.noreturn`; Sorbet proves the branch is unreachable. Add a member without a branch and `T.absurd` raises at runtime, turning an omission into a crash at the first unhandled call.
3. For a closed hierarchy of classes with shared behaviour, combine `abstract!` and `sealed!` on the parent module. `sealed!` restricts subclassing to the same file; `abstract!` requires every concrete subclass to implement the abstract interface. Sorbet checks both constraints at typecheck time.
4. Never reach for an optional field (`paid_at: T.nilable(Time)`) when the real model is a state transition. A `Paid` member carrying `paid_at: Time` makes the field mandatory on that state and absent on all others; no `nil` guard needed.

Worked example:
```ruby
class OrderStatus < T::Enum
  enums do
    Pending   = new("pending")
    Paid      = new("paid")
    Cancelled = new("cancelled")
  end
end

sig { params(status: OrderStatus).returns(String) }
def label(status)
  case status
  when OrderStatus::Pending   then "Awaiting payment"
  when OrderStatus::Paid      then "Payment received"
  when OrderStatus::Cancelled then "Cancelled"
  else T.absurd(status) # breaks at runtime if a new enum member is unhandled
  end
end

# Sealed hierarchy — closed polymorphism without an enum
module Discount
  extend T::Helpers
  sealed!
  abstract!
end

class PercentDiscount
  include Discount
  extend T::Sig

  const :rate, Float
end

class FixedDiscount
  include Discount
  extend T::Sig

  const :amount, Money
end
```

**Enforcement:** `srb tc` rejects passing a raw `Symbol`/`String` where `T::Enum` is required. `T.absurd` is checked at code review; every `case` on a `T::Enum` or sealed module must close with it. See [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).

### 6.9 — Define and depend on small, duck-typed interfaces declared as Sorbet abstract modules.

**Reasoning, step by step:**
1. A concrete dependency is a hard coupling: the caller knows the class, its constructor arguments, and every method it happens to expose. An abstract module narrows that coupling to the methods the caller actually uses.
2. Declare the interface as a Sorbet `abstract!` module with `sig { abstract.returns(...) }` stubs. Concrete implementations `include` it and Sorbet enforces that every abstract method is overridden. The caller takes the module type; swapping implementations is a one-line change.
3. Name interface modules by role — `Priceable`, `Fulfillable`, `Auditable` — not by implementation family (`AbstractOrder`, `BaseRepository`). The role name describes what the caller needs; the implementation name is a detail.
4. Keep interfaces small: one to three methods. A wide interface is several small interfaces merged. Callers that need only one method take the one-method module; they never pay for methods they do not use.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

module Priceable
  extend T::Sig
  extend T::Helpers

  abstract!

  sig { abstract.returns(Money) }
  def price; end
end

class LineItem < T::Struct
  include Priceable
  extend T::Sig

  const :sku,      Sku
  const :price,    Money
  const :quantity, Integer

  sig { override.returns(Money) }
  def price
    Money.new(cents: @price.cents * quantity)
  end
end

# Caller depends on the interface, not the concrete class.
sig { params(items: T::Array[Priceable]).returns(Money) }
def subtotal(items)
  items.reduce(Money.new(cents: 0)) { |acc, item| acc.add(item.price) }
end
```

**Enforcement:** Code review: a parameter typed as a concrete class where an abstract module would suffice is a coupling smell. `srb tc` enforces `override` on every implementation of an `abstract` method.

### 6.10 — Expose `attr_reader` only on value objects; never a setter that lets an invariant break after construction.

**Reasoning, step by step:**
1. A setter (`attr_writer`, `attr_accessor`, or a hand-written `def field=(value)`) is a gap in the invariant: the object was validated at construction, but the setter lets a caller mutate it to an invalid state after the fact. Value objects must be immutable; a mutable field on a value object undermines every downstream assumption.
2. `T::Struct` with `const` fields enforces this at the language level: Sorbet rejects any attempt to assign to a `const` field after construction. `Data.define` objects are frozen immediately; a mutation attempt raises `FrozenError`. For hand-rolled classes, freeze in `initialize` and expose `attr_reader` only.
3. If you need a "modified" value, model the transformation as a method that returns a new instance — `with` on `Data.define`, `with` on `T::Struct`, or an explicit factory method. Mutation is a loop where the loop runs once; transformation is a function from old value to new value.
4. For lifecycle classes that hold mutable state (chapter 13), setters may be warranted — but only when the class owns the mutation as part of its defined lifecycle contract, exposed through a narrow, named method, not a raw `attr_accessor`.

Worked example:
```ruby
# bad — attr_accessor on a value object; cents can be set to -100 after parse
class Money
  attr_accessor :cents

  def initialize(cents)
    raise ArgumentError, "negative cents" unless cents >= 0
    @cents = cents
  end
end

price = Money.new(100)
price.cents = -999 # invariant broken; no guard

# good — T::Struct const; the field is immutable after construction
class Money < T::Struct
  extend T::Sig
  const :cents, Integer
end

# good — transformation returns a new instance; original is untouched
sig { params(discount: Integer).returns(Money) }
def discounted(discount)
  raise ArgumentError, "discount exceeds price" if discount > cents

  Money.new(cents: cents - discount)
end
```

**Enforcement:** RuboCop `Style/AccessorGrouping` ensures readers are grouped; code review rejects `attr_writer` and `attr_accessor` on any `T::Struct` or `Data.define` value object. `T::Struct` with `const` makes the prohibition structural.

### 6.11 — Respect the Liskov Substitution Principle whenever you do inherit.

**Reasoning, step by step:**
1. The Liskov Substitution Principle: every subclass must be usable wherever its parent is accepted, with no surprises. A subclass that narrows accepted input, widens returned types, raises where the parent does not, or overrides a method to change the shared contract is not a subtype — it is a different thing wearing the parent's name.
2. In Ruby, the contract is not enforced by the type system alone: a subclass can freely override any method and change its behaviour. The only guarantees come from discipline. Test every subclass against the parent's test suite as well as its own (the Liskov test: substitute the subclass in the parent's tests and all should pass).
3. The practical rule: never override a method to change shared logic. Override to extend — call `super` first or last, then add, never replace. If you must replace the body entirely, the class hierarchy is modelling the wrong thing; extract a module or a collaborator instead.
4. This rule applies narrowly because rule 6.5 already restricts inheritance to `StandardError` trees and the narrow case of Sorbet abstract/sealed modules. LSP is the discipline that keeps those cases correct.

Worked example:
```ruby
# bad — override changes the shared logic; OrderWithSurcharge is not a valid Order substitute
class Order
  extend T::Sig

  sig { returns(Money) }
  def total
    lines.reduce(Money.new(cents: 0)) { |acc, line| acc.add(line.price) }
  end
end

class OrderWithSurcharge < Order
  sig { override.returns(Money) }
  def total
    # replaces parent logic entirely; callers of Order cannot rely on the contract
    Money.new(cents: super.cents + 500)
  end
end

# good — behaviour extension through composition, not override
class SurchargeCalculator
  extend T::Sig

  sig { params(order: Order, surcharge: Money).returns(Money) }
  def self.total_with_surcharge(order, surcharge)
    raise ArgumentError, "surcharge must be non-negative" unless surcharge.cents >= 0

    Money.new(cents: order.total.cents + surcharge.cents)
  end
end
```

**Enforcement:** Code review: any `override` that does not call `super` is a LSP smell requiring justification. Sorbet `sig { override.returns(...) }` enforces covariant return types statically; narrowing params is caught by `srb tc`.

### 6.12 — Keep classes small and single-responsibility; a god object is many objects wearing one name.

**Reasoning, step by step:**
1. A class that does many things is hard to name — its name either lies (claims one responsibility, delivers many) or becomes a bag (`Manager`, `Service`, `Helper`) that names nothing. If you cannot name the class in one domain noun without an "and," split it.
2. RuboCop `Metrics/ClassLength` caps class length. The cap is a floor, not a target; aim for a class whose entire body fits on a screen. A class that scrolls is a candidate for extraction.
3. Single responsibility means a class has one reason to change. When a class must change because pricing changes, *and* because serialization format changes, *and* because status reporting changes, it has three reasons — it is three classes. Extract each concern into its own type.
4. A long private method section is a sign that a class is doing its own internal bookkeeping that a collaborator should own. Long `initialize` parameter lists are a sign that too many dependencies were merged; break the class at its seams.

Worked example:
```ruby
# bad — Order does pricing, serialization, notification, and status reporting
class Order
  def total; ...; end             # pricing concern
  def to_json; ...; end           # serialization concern
  def notify_customer!; ...; end  # notification concern
  def status_report; ...; end     # reporting concern
end

# good — each class has one reason to change
class Order < T::Struct
  const :id,    String
  const :lines, T::Array[LineItem]
  const :status, OrderStatus
end

module OrderPricing
  class << self
    extend T::Sig

    sig { params(order: Order).returns(Money) }
    def total(order)
      order.lines.reduce(Money.new(cents: 0)) { |acc, line| acc.add(line.price) }
    end
  end
end

module OrderSerializer
  class << self
    extend T::Sig

    sig { params(order: Order).returns(String) }
    def to_json(order)
      { id: order.id, status: order.status.serialize }.to_json
    end
  end
end
```

**Enforcement:** RuboCop `Metrics/ClassLength` (cap configured in `.rubocop.yml`); `Metrics/MethodLength` (25 lines, chapter 05). Code review rejects any class that cannot be named without "and" or that carries concerns from more than one domain.

## Cross-references

- `# typed: strict`, `sig` on every method, `T.let` in `initialize`, `T::Struct` const fields, `T::Enum`, `T.absurd`, and `sealed!`: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `attr_reader`, class instance variables over `@@`, `freeze` on constants, and `||=` caveats: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Method size cap (25 lines), guard clauses, one level of abstraction, and `public_send` for duck-typed dispatch: [05-methods.md](./05-methods.md).
- `case/in` pattern matching, Enumerable pipelines, and `tap`/`then` for transformation chains: [07-ruby-idioms.md](./07-ruby-idioms.md).
- `StandardError` subclass trees, raise class + message, and `cause` chaining — the one sanctioned `<` hierarchy: [08-error-handling.md](./08-error-handling.md).
- `sig` on every public method as the API contract, accepting duck types, returning frozen concretes: [10-api-design.md](./10-api-design.md).
- Testing value objects with negative-space constructors, fakes over mocks for interface implementations: [11-testing.md](./11-testing.md).
- One class per file, Zeitwerk autoloading, and circular-dependency discipline: [12-module-organization.md](./12-module-organization.md).
- Lifecycle classes, block form for closables, and `ensure` teardown — the narrow case where a stateful class is warranted: [13-resource-management.md](./13-resource-management.md).
- Stable object shapes, `T.let` in `initialize`, and YJIT monomorphism for value objects: [15-performance.md](./15-performance.md).
