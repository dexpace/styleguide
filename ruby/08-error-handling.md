# 08 — Error Handling

Errors are values, handled explicitly. Every failure is a typed `StandardError` subclass raised with a class and a message, chained through `cause` on rethrow, and rescued precisely. Programmer errors — broken invariants, unreachable states, bugs — raise `Assert::InvariantViolation` and crash fast; operational errors are caught by callers who can recover.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  class Error < StandardError; end

  class PaymentError < Error
    sig { returns(String) }
    attr_reader :order_id

    sig { params(message: String, order_id: String, cause: T.nilable(Exception)).void }
    def initialize(message, order_id:, cause: nil)
      super(message)
      @order_id = order_id
      set_backtrace(cause&.backtrace) if cause
    end
  end

  class CardDeclinedError < PaymentError
    sig { returns(String) }
    attr_reader :decline_code

    sig { params(order_id: String, decline_code: String).void }
    def initialize(order_id:, decline_code:)
      super("order #{order_id}: card declined (#{decline_code})", order_id:)
      @decline_code = decline_code
    end
  end

  class GatewayUnavailableError < PaymentError; end
end

module Payments
  class ChargeService
    sig { params(order: Order, card_token: String).returns(Receipt) }
    def charge(order, card_token:)
      Assert.that(order.total.cents > 0, "order #{order.id} total must be positive")

      response = gateway.submit(card_token, order.total)
      build_receipt(order, response)
    rescue GatewayClient::TimeoutError => error
      raise Commerce::GatewayUnavailableError.new(
        "gateway unreachable for order #{order.id}",
        order_id: order.id,
        cause: error,
      )
    end

    private

    sig { params(order: Order, response: GatewayClient::Response).returns(Receipt) }
    def build_receipt(order, response)
      case response.status
      when "approved"
        Receipt.new(order_id: order.id, amount: order.total, authorized_at: response.timestamp)
      when "declined"
        raise Commerce::CardDeclinedError.new(
          order_id: order.id,
          decline_code: response.decline_code,
        )
      else
        Assert.fail("unknown gateway status '#{response.status}' for order #{order.id}")
      end
    end
  end
end
```

`Commerce::Error` sits at the root of a typed hierarchy so callers rescue at any depth they need (8.1). `PaymentError` carries `order_id` as a typed field (8.1). `GatewayClient::TimeoutError` is caught and immediately re-raised as a domain error with `cause:` set to preserve the original stack (8.10). `Assert.that` guards the precondition — a positive total is a programming contract, not an operational case (8.12). The `case` statement uses `Assert.fail` for the unreachable branch; the runtime enforces exhaustiveness rather than a silent fall-through (8.12). No `rescue` block is empty, no modifier `rescue` appears, and no exception surfaces for a normal case — the happy path is `build_receipt`, and only a declined card or an unreachable gateway raises (8.3, 8.5, 8.4).

## Rules

### 8.1 — Build a typed `StandardError` hierarchy rooted at one project base.

**Reasoning, step by step:**
1. A bare `raise "order failed"` gives a debugger a string and nothing else: no order id, no context, no hierarchy. The caller cannot rescue precisely because there is no class to match.
2. Define one project-level base, e.g. `Commerce::Error < StandardError`. Per-domain trees hang off it: `PaymentError < Commerce::Error`, `InventoryError < Commerce::Error`. Callers rescue the root for broad handling or a leaf for precise recovery.
3. Carry the identifying inputs as Sorbet-typed `attr_reader` fields — ids, the offending value, a correlation id that ties the failure to a request. These survive serialization and appear in structured logs.
4. Keep the tree two levels deep. A five-level hierarchy navigates no better than a two-level one; the depth tax outweighs any precision gain.
5. Introduce a new class only when callers must distinguish it. A new subclass is a new rescue clause the caller is responsible for handling; prefer the existing class and a richer message when the distinction would never drive different handling.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  class Error < StandardError; end

  class InventoryError < Error
    sig { returns(Sku) }
    attr_reader :sku

    sig { params(message: String, sku: Sku).void }
    def initialize(message, sku:)
      super(message)
      @sku = sku
    end
  end

  class InsufficientStockError < InventoryError
    sig { returns(Integer) }
    attr_reader :requested, :available

    sig { params(sku: Sku, requested: Integer, available: Integer).void }
    def initialize(sku:, requested:, available:)
      super(
        "sku #{sku}: requested #{requested}, only #{available} available",
        sku:,
      )
      @requested = requested
      @available = available
    end
  end
end
```

**Enforcement:** review; all domain failures extend `Commerce::Error`; no bare `raise StandardError` or `raise RuntimeError` in application code.

### 8.2 — Never rescue `Exception`; rescue `StandardError` or a named subclass.

**Reasoning, step by step:**
1. `Exception` is the root of Ruby's exception tree, not `StandardError`. Below `StandardError` sit `SignalException` (including `Interrupt`), `NoMemoryError`, `SystemExit`, and `ScriptError`. Rescuing `Exception` catches process signals and out-of-memory conditions that the runtime uses to tear down — your rescue block runs instead of the teardown, and the process neither exits cleanly nor propagates the signal.
2. A bare `rescue` without a class already means `rescue StandardError`. Prefer naming the class anyway when you are at a boundary that logs or wraps; name it precisely (a subclass) when only that subclass needs handling. The bare form is acceptable in a narrow `rescue => error` that immediately re-raises.
3. Never write `rescue Exception => error` in application code. The one place it belongs is a top-level crash reporter that re-raises after logging — and that belongs in a framework or process supervisor, not in a business-logic method.

Worked example:

```ruby
# Avoid — catches Interrupt, NoMemoryError, etc.
def reserve_inventory(sku, quantity)
  Inventory.reserve!(sku, quantity)
rescue Exception => error   # wrong: catches everything
  logger.error(error)
end

# Prefer — catches only recoverable failures
sig { params(sku: Sku, quantity: Integer).returns(T::Boolean) }
def reserve_inventory(sku, quantity)
  Inventory.reserve!(sku, quantity)
  true
rescue Commerce::InsufficientStockError
  false
end
```

**Enforcement:** RuboCop `Lint/RescueException`; review rejects `rescue Exception` outside the designated crash-reporter module.

### 8.3 — No exceptions for flow control.

**Reasoning, step by step:**
1. An exception is for the exceptional. Using `raise`/`rescue` to signal an ordinary "not found" or "zero result" conflates a result with a failure and pays a full stack-trace capture for a routine branch — a multi-microsecond overhead at call frequency.
2. Check the condition before you operate. `raise ZeroDivisionError` after dividing is always avoidable: `if divisor.zero?` is the check that costs nothing and reads as the contract. Catching `ZeroDivisionError` to handle the zero case means the exception carries information the caller already had.
3. Absence is `nil`. A lookup that may miss returns `T.nilable(Order)`, and the caller uses `&.` or an explicit nil check. Reserve `raise` for genuine failures (8.12) and never as a `return`-with-extra-steps from a conditional.
4. A yes/no question returns a boolean. `#in_stock?` returns `false` for "no"; it does not raise `OutOfStockError` that the caller catches to read the answer.

Worked example:

```ruby
# Avoid — exception used as a conditional branch
sig { params(order_id: String).returns(Money) }
def order_total(order_id)
  begin
    Order.find!(order_id).total
  rescue ActiveRecord::RecordNotFound
    Money.parse(0)   # wrong: catching a lookup result as if it were a failure
  end
end

# Prefer — nil check, no exception in the happy path
sig { params(order_id: String).returns(Money) }
def order_total(order_id)
  order = Order.find(order_id)
  return Money.parse(0) if order.nil?

  order.total
end
```

**Enforcement:** review; callers that rescue a class to read a "not found" branch are rejected; `T.nilable` return + nil check is the canonical pattern.

### 8.4 — Use implicit `begin`: method-level `rescue`/`ensure` without an explicit `begin` block.

**Reasoning, step by step:**
1. Ruby's `def`/`end` is already an implicit `begin`/`end`. An explicit `begin` nested inside the method body adds a redundant keyword and an extra indentation level that the entire method body pays for.
2. With implicit `begin`, `rescue` and `ensure` appear at the method level, flush with `def`. The structure is visible in the signature rather than buried inside the body.
3. The only legitimate use of an explicit `begin`/`end` inside a method body is when only a sub-expression — not the whole method — needs the rescue. If the rescue scope is smaller than the method, make it a private method and rescue at that method's level.

Worked example:

```ruby
# Avoid — redundant explicit begin inside a method
sig { params(order: Order).returns(Receipt) }
def finalize(order)
  begin
    charge_and_persist(order)
  rescue Commerce::PaymentError => error
    handle_payment_failure(order, error)
  end
end

# Prefer — implicit begin; rescue at method level
sig { params(order: Order).returns(Receipt) }
def finalize(order)
  charge_and_persist(order)
rescue Commerce::PaymentError => error
  handle_payment_failure(order, error)
end
```

**Enforcement:** RuboCop `Style/RedundantBegin`; reviewer rejects explicit `begin` at the outermost scope of a method body.

### 8.5 — No empty rescue, no `rescue nil`, no modifier rescue.

**Reasoning, step by step:**
1. An empty `rescue` block discards the exception and ships whatever corrupt state follows. The error is information; dropping it is a choice to remain ignorant of bugs.
2. `rescue nil` (`result = do_thing rescue nil`) is the worst form: it rescues *every* `StandardError` including bugs in `do_thing`, swallows the exception silently, and returns `nil` so callers assume success. A typo, a missing method, a wrong argument — all hidden.
3. The modifier rescue (`expr rescue fallback`) is similarly untargeted. It reads compact but catches everything, cannot log, cannot chain cause, and cannot distinguish the expected failure from a programmer mistake. Use a full `rescue` clause with a named class.
4. The one acceptable near-miss: a rescue that catches the single expected error class and immediately re-raises anything else is honest. Document why the failure is tolerable.

Worked example:

```ruby
# Avoid — swallows everything including bugs
cached_price = fetch_price(sku) rescue nil

# Avoid — silent, untargeted
begin
  sync_inventory(sku)
rescue
end

# Prefer — targeted, explained
begin
  cache.delete(sku)
rescue Redis::CommandError => error
  # Cache eviction is best-effort; a Redis blip does not fail the request.
  # Re-raise anything unexpected so bugs surface.
  raise unless error.message.include?("READONLY")
end
```

**Enforcement:** RuboCop `Lint/SuppressedException` (empty rescue), `Style/RescueModifier` (modifier rescue); review rejects `rescue nil`; any deliberate swallow requires a named class and a why-comment.

### 8.6 — Raise with a class and a message as two arguments; omit `RuntimeError` on bare messages.

**Reasoning, step by step:**
1. `raise SomeError.new("message")` instantiates the error before passing it to `raise`. `raise SomeError, "message"` delegates instantiation to `raise`, which sets the backtrace at the correct call depth. Prefer the two-argument form.
2. A bare `raise "message"` raises `RuntimeError`, which no caller can rescue precisely — the message is the only distinguishing feature and messages are not a stable API. The two-argument form `raise Commerce::Error, "message"` gives the caller a class to match and documents intent.
3. `raise RuntimeError, "message"` is redundant: when the class is `RuntimeError`, omit it and write `raise "message"`. But never use the bare form in application code (see point 2); use a named class.
4. When re-raising the current exception inside a rescue block (`raise` with no arguments), omit both class and message — the bare `raise` re-raises `$!` with the original backtrace intact.

Worked example:

```ruby
# Avoid — instantiates error before raise; backtrace may point at wrong line
raise Commerce::InventoryError.new("sku #{sku} not found")

# Avoid — RuntimeError; no stable class for callers to rescue
raise "sku #{sku} not found"

# Prefer — class + message as two arguments
raise Commerce::InsufficientStockError, "sku #{sku}: #{available} in stock, #{requested} requested"

# Prefer — bare raise inside rescue to re-raise unchanged
rescue Commerce::Error => error
  log_error(error)
  raise
end
```

**Enforcement:** RuboCop `Style/RaiseArgs` (enforces two-argument form); review rejects bare string raises in application code.

### 8.7 — Never `return` from an `ensure` block.

**Reasoning, step by step:**
1. Ruby's `ensure` block runs regardless of whether an exception is in flight. A `return` inside `ensure` discards any in-flight exception — silently. The exception was the signal that something went wrong; the `return` makes the method appear to succeed.
2. The discard is invisible at the call site: the caller receives a return value and has no indication that an exception was swallowed. This is the worst kind of swallow (8.5) because it appears in what looks like cleanup code.
3. `ensure` is for side effects: closing resources, releasing locks, decrementing counters. It must not compute a return value. If you need to return a fallback when an exception occurs, use `rescue`; let `ensure` stay unconditional cleanup.

Worked example:

```ruby
# Avoid — return in ensure discards any in-flight exception
sig { params(order: Order).returns(Receipt) }
def process(order)
  charge(order)
rescue Commerce::PaymentError => error
  raise Commerce::OrderFailedError.new("order #{order.id} failed", order_id: order.id, cause: error)
ensure
  return fallback_receipt(order)   # wrong: silently discards the PaymentError
end

# Prefer — ensure is cleanup only; rescue handles the failure path
sig { params(order: Order).returns(Receipt) }
def process(order)
  charge(order)
rescue Commerce::PaymentError => error
  raise Commerce::OrderFailedError.new("order #{order.id} failed", order_id: order.id, cause: error)
ensure
  Audit.record_attempt(order.id)   # cleanup only, no return
end
```

**Enforcement:** RuboCop `Lint/EnsureReturn`; reviewer rejects any `return` expression inside an `ensure` block.

### 8.8 — Never `return` from a `begin`/`end` used in an assignment context.

**Reasoning, step by step:**
1. A `begin`/`end` used on the right-hand side of an assignment — typically to memoize or to initialize a variable — may contain a `rescue` clause. A `return` inside that `rescue` skips the assignment entirely and exits the method, so the variable is never set.
2. This is the silent memoization bug: `@result ||= begin; compute; rescue Error; return default; end` — on error, `@result` stays `nil`, the method returns `default`, and every subsequent call hits the memoized `nil` and re-computes. The bug is invisible until load reveals the pattern.
3. Assign the fallback instead of returning it. Inside a rescue nested in an assignment `begin`/`end`, the last expression of the `rescue` clause becomes the assigned value. Use that, never `return`.

Worked example:

```ruby
# Avoid — return inside a begin/end assignment skips the assignment
sig { returns(T.nilable(ExchangeRate)) }
def rate
  @rate ||= begin
    ExchangeRateService.fetch(base: "USD", target: currency)
  rescue ExchangeRateService::UnavailableError
    return nil   # wrong: @rate is never set; nil is returned from the method instead
  end
end

# Prefer — rescue clause evaluates to the fallback; assignment completes
sig { returns(T.nilable(ExchangeRate)) }
def rate
  @rate ||= begin
    ExchangeRateService.fetch(base: "USD", target: currency)
  rescue ExchangeRateService::UnavailableError
    nil   # @rate is assigned nil; subsequent calls short-circuit on @rate correctly
  end
end
```

**Enforcement:** review; any `return` inside a `begin`/`end` that appears on the right side of an assignment or `||=` is rejected.

### 8.9 — Name the rescue variable `error`, not `e`.

**Reasoning, step by step:**
1. `e` is a single-character binding that carries no information. A reader encountering `rescue => e` must look at the usages to determine what kind of failure it represents; `rescue => error` is self-describing.
2. In a method that rescues multiple classes, meaningful names make each clause readable in isolation: `rescue Commerce::CardDeclinedError => declined_error` vs `rescue Commerce::CardDeclinedError => e`.
3. A short name signals the binding is throwaway — that the error is about to be discarded. A full name signals it will be used: logged, chained, or re-raised with context. The naming convention enforces honest intent.

Worked example:

```ruby
# Avoid
rescue Commerce::PaymentError => e
  raise Commerce::OrderFailedError.new("order #{order.id}", order_id: order.id, cause: e)

# Prefer
rescue Commerce::PaymentError => error
  raise Commerce::OrderFailedError.new("order #{order.id}", order_id: order.id, cause: error)
```

**Enforcement:** RuboCop custom naming convention or review; `e` as a rescue variable name is rejected at code review.

### 8.10 — Chain causes on rethrow; attach context fields.

**Reasoning, step by step:**
1. When you catch one error and raise another, the original error is the evidence of what went wrong. Discard it and the new stack trace starts at your raise site; the actual fault is gone. A chain without `cause` is amnesia.
2. Ruby sets `$!` to the current exception inside a rescue block. Pass it explicitly as the `cause:` keyword when constructing the new error — do not rely on Ruby's implicit cause assignment, which only works when you raise inside the rescue; explicit is always clear.
3. The cause must be the original exception, not a string. Wrap raw third-party exceptions in a typed domain class (8.1) before propagating; the cause chain carries the raw exception, the domain class names the failure in your vocabulary.
4. Attach context fields on the new error class: the inputs that triggered the failure, a correlation id that ties the chain to the request. These fields are indexable in structured logs without parsing message strings.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(order: Order).returns(Receipt) }
def persist_order(order)
  DB.insert(:orders, order.to_h)
rescue Sequel::UniqueConstraintViolation => error
  raise Commerce::DuplicateOrderError.new(
    "order #{order.id} already exists",
    order_id: order.id,
    cause: error,     # original stack preserved in the chain
  )
rescue Sequel::DatabaseError => error
  raise Commerce::PersistenceError.new(
    "failed to persist order #{order.id}",
    order_id: order.id,
    cause: error,
  )
end
```

**Enforcement:** review; every wrap-and-rethrow passes `cause:`; a rethrow without `cause:` is a review rejection.

### 8.11 — Prefer a standard-library exception where one fits; introduce a new class only when callers must distinguish it.

**Reasoning, step by step:**
1. Ruby ships `ArgumentError`, `KeyError`, `TypeError`, `RangeError`, `StopIteration`, and others. When the failure is exactly what one of these describes — a caller passed the wrong type, a key is absent, a value is out of range — use the standard class. Callers already know how to rescue it; documentation already names it.
2. Introduce a new class only when callers must distinguish it for handling. A new subclass is a new contract: the caller is responsible for rescuing it. Pay that cost only when the distinction drives different behaviour at some call site.
3. Wrap standard-library exceptions at layer boundaries (8.10): a `KeyError` from a hash lookup inside a repository stays inside the repository and surfaces as a typed domain error to the layer above.
4. `ArgumentError` and `TypeError` are for caller mistakes — wrong arity, wrong type. They are programmer errors (8.12); raise them from validation helpers but do not rescue them in business logic.

Worked example:

```ruby
# Good — ArgumentError for caller mistake, standard class suffices
sig { params(rate: Float).void }
def set_discount_rate(rate)
  raise ArgumentError, "rate must be between 0 and 1, got #{rate}" unless rate.between?(0.0, 1.0)

  @rate = rate
end

# Good — new class because callers must distinguish "declined" from "unavailable"
raise Commerce::CardDeclinedError.new(order_id: order.id, decline_code: response.decline_code)
raise Commerce::GatewayUnavailableError.new("gateway timeout", order_id: order.id, cause: error)
```

**Enforcement:** review; new exception classes require a stated reason why existing classes cannot be rescued precisely enough.

### 8.12 — `ensure` for cleanup; assertions raise `Assert::InvariantViolation`; separate programmer errors from operational errors.

**Reasoning, step by step:**
1. A programmer error is a bug: a violated precondition, an unreachable branch reached, a computation that produces an impossible result. Retrying cannot fix it — the code is wrong. It must crash loudly and close to the fault so the bug surfaces immediately. Use `Assert.that` and `Assert.fail` (defined in chapter 05) for this purpose.
2. An operational error is an expected failure of a correct program: a card declined, a gateway timeout, an item out of stock. It is part of the contract and must be raised as a typed domain error (8.1) or returned as a nilable value (chapter 03), then handled by callers.
3. Never demote a programmer error to a handled operational error. Catching `Assert::InvariantViolation` in business logic hides bugs; it belongs only in a top-level crash reporter. Never promote an operational error to a crash; a missing inventory item is not a bug.
4. `ensure` is unconditional cleanup: close files, release locks, delete temp rows, decrement gauges. Pair it with block-form resource methods (chapter 13) where possible. An `ensure` that can itself raise must rescue narrowly inside the `ensure` body; an exception escaping an `ensure` replaces the in-flight exception — another form of silent discard.
5. Distinguish the two error paths at the type level: `Assert::InvariantViolation` is not a subclass of `Commerce::Error`. A caller rescuing `Commerce::Error` does not accidentally catch a programming bug; a caller rescuing `StandardError` at a boundary should re-raise `Assert::InvariantViolation` explicitly.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(order: Order, sku: Sku, quantity: Integer).returns(LineItem) }
def add_line_item(order, sku, quantity)
  # Programmer error — Assert crashes; no rescue expected
  Assert.that(quantity > 0, "quantity must be positive, got #{quantity}")
  Assert.that(order.status == :draft, "order #{order.id} is not a draft; cannot add items")

  # Operational error — typed class; callers may rescue
  inventory = Inventory.find(sku)
  raise Commerce::InsufficientStockError.new(
    sku:,
    requested: quantity,
    available: inventory.on_hand,
  ) if inventory.on_hand < quantity

  LineItem.new(order_id: order.id, sku:, quantity:, unit_price: inventory.price)
ensure
  # Cleanup unconditionally; no return, no raise that could replace in-flight error
  Audit.log_line_item_attempt(order.id, sku, quantity)
end
```

**Enforcement:** review; `Assert::InvariantViolation` is never rescued in business logic; operational errors are typed `Commerce::Error` subclasses; `ensure` blocks contain no `return` (8.7) and must narrow any rescue within them.

## Cross-references

- `Assert.that`, `Assert.fail`, `Assert::InvariantViolation`, and assertion density: [05-methods.md](./05-methods.md).
- `sig` on every method, `T.nilable` for absent values, parse-at-boundaries discipline: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `T::Enum`, sealed modules, and discriminated unions for exhaustive case analysis: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Block-form resource management, `ensure` teardown, and deterministic cleanup: [13-resource-management.md](./13-resource-management.md).
- YARD `@raise` tags and contract changes as public API: [14-documentation.md](./14-documentation.md).
- Minimal public surface, keyword-argument boundaries, and error contracts in public APIs: [10-api-design.md](./10-api-design.md).
- Effect-verb naming for error-raising methods: [02-naming-conventions.md](./02-naming-conventions.md).
