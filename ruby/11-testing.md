# 11 — Testing

Correctness is this guide's first value; a test suite is its primary proof. The type system via `srb tc` is the first suite (chapter [03](./03-type-safety-and-nil-discipline.md)); Minitest is the second — and the two are not interchangeable: a type regression passes every runtime test, a logic regression passes every type check. Tests are code that runs on every push; they earn their keep by failing precisely on real regressions and passing quietly otherwise.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

require "test_helper"
require "minitest/property"

class MoneyTest < Minitest::Test
  extend T::Sig

  test "addition is commutative" do
    a = Money.new(cents: 150, currency: "GBP")
    b = Money.new(cents: 75,  currency: "GBP")

    result_ab = a.add(b)
    result_ba = b.add(a)

    assert_equal result_ab, result_ba                    # positive: order does not matter (11.10)
    assert_equal Money.new(cents: 225, currency: "GBP"), result_ab
  end

  test "add raises on currency mismatch" do
    gbp = Money.new(cents: 100, currency: "GBP")
    usd = Money.new(cents: 100, currency: "USD")

    error = assert_raises(Money::CurrencyMismatch) { gbp.add(usd) }

    assert_match(/GBP.*USD/, error.message)             # negative space: error surface (11.10)
  end

  test "round trips through serialisation" do
    property(
      gen_amount:   -> { rand(1..1_000_000) },          # bounded generator (11.7, 11.11)
      gen_currency: -> { %w[GBP USD EUR].sample },
    ) do |amount:, currency:|
      original = Money.new(cents: amount, currency:)

      assert_equal original, Money.parse(original.to_h) # round-trip invariant (11.7)
    end
  end
end
```

The suite tests `Money` three ways. The first `test "..."` block follows AAA paragraphs separated by blank lines (11.2) and asserts both positive space (commutativity) and the exact value (11.10). The second asserts the negative space — a `CurrencyMismatch` is raised and its message names both currencies — pairing the error-boundary check with chapter [08](./08-error-handling.md) (11.3, 11.10). The property test fuzzes the round-trip invariant `Money.parse(m.to_h) == m` over bounded random input, proving the codec law over the full input space not just one lucky example (11.7). The clock is never read; `rand` is the only randomness and it is bounded (11.8, 11.11). `srb tc` would catch a wrong `sig` before any of these run (11.9).

## Rules

### 11.1 — Use Minitest with `test "..." do` blocks; require test helpers explicitly.

**Reasoning, step by step:**
1. Minitest is the Shopify and core-Ruby default; it is fast, standard, and ships with Ruby — no separate install, no magic, no framework lock-in. It is the single framework for every dexpace Ruby project; introduce no second runner.
2. `test "describes behaviour" do ... end` beats `def test_name` on readability: the description is a freeform string visible in `--verbose` output and CI logs. The `def` form forces identifier encoding on English prose, losing punctuation and natural phrasing.
3. Require test helpers and subject files explicitly at the top of every test file. Rely on no autoload magic; a test file must state every dependency it uses. Load-order problems surface immediately rather than hiding behind lucky require order.
4. Organize tests as `FooTest < Minitest::Test`, one test class per production class, in a `test/` directory mirroring `lib/` — the same file-to-class rule as chapter [12](./12-module-organization.md) applied to the test tree.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

require "test_helper"
require_relative "../lib/order"

class OrderTest < Minitest::Test
  test "calculates total from line items" do
    # arrange, act, assert below
  end
end
```

**Enforcement:** `rubocop` with `Minitest/TestMethodName` cop; review rejects `def test_*`; single framework enforced at the project bootstrap level.

### 11.2 — AAA structure: arrange, act, assert as blank-line-separated paragraphs; one behaviour per test.

**Reasoning, step by step:**
1. Every test has three phases: set up the world (arrange), run the one operation under test (act), check the outcome (assert). Separating them with blank lines makes the structure visible before a single expression is read — the same paragraph discipline as chapter [05](./05-methods.md).
2. One behaviour per test: a test that asserts five unrelated facts fails at the first and hides the rest. Split it; each failure then names exactly what broke and under which condition.
3. Multiple assertions on the *same* behaviour are correct and expected (11.10). The rule is one *behaviour*, not one `assert_*` call.
4. The test name is the failure message read at 2 am and in CI logs. Shape it as `<verb-phrase> when <condition>` — `raises CurrencyMismatch when currencies differ` tells you what broke; `money test 3` tells you nothing.

Worked example:

```ruby
test "applies discount when order qualifies" do
  customer = Customer.new(tier: "gold")
  order    = Order.new(customer:, items: [LineItem.new(sku: Sku.new("ABC"), qty: 2, price: 500)])

  discounted = order.apply_discount

  assert_equal 900, discounted.total_cents
end
```

**Enforcement:** `Minitest/MultipleAssertions` cop set to warn above four (same-behaviour pairs are exempt); review for AAA shape and behavioural test names.

### 11.3 — Use descriptive assertions; `assert_equal`, `assert_predicate`, `refute_nil` over bare `assert`.

**Reasoning, step by step:**
1. A failure message must locate the bug without a debugger. `assert x == y` prints `Expected false to be truthy` — meaningless. `assert_equal expected, actual` prints `Expected 100, got 75` — the bug is on the screen.
2. Always pass `expected` before `actual` to `assert_equal`; Minitest's diff output is backwards when the order is wrong and every failure message misleads the reader.
3. Use the most specific assertion available: `assert_predicate order, :valid?` prints `Expected #<Order ...> to be valid?`; `assert_equal true, order.valid?` prints a useless bool diff. `refute_nil`, `assert_empty`, `assert_includes`, `assert_raises` — each produces a targeted message.
4. Never use `assert_nothing_raised`; it was removed from Minitest 6+ because it inverts the test structure. Assert the *positive outcome* directly: if the operation succeeds, assert its return value (11.5).

Worked example:

```ruby
# bad: opaque failure
assert order.line_items.any? { |li| li.sku == target_sku }

# good: precise failure message
assert_includes order.line_items.map(&:sku), target_sku
assert_predicate order, :open?
refute_nil inventory.find(sku: Sku.new("WIDGET-1"))
```

**Enforcement:** `Minitest/AssertPredicate`, `Minitest/RefuteNil`, `Minitest/AssertEqual` cops; review rejects bare `assert x == y` or `assert_equal` with reversed argument order.

### 11.4 — One aspect per test case; split compound tests so failures are independent.

**Reasoning, step by step:**
1. A test that checks three unrelated properties of a result is really three tests glued together. When property one fails, properties two and three never run — the test suite gives a partial picture of what is broken and a partial picture of what works.
2. Split each independent aspect into its own `test "..." do` block. The total number of tests grows; the cost is zero. The benefit is precise, independent failure reporting on every push.
3. Aspects that are genuinely part of the same behaviour belong together (11.10). The split criterion is independence: would a future change that breaks property A leave property B intact? If yes, they are independent and belong in separate tests.
4. The split also enforces the one-act rule: each test has exactly one `act` line. Multiple unrelated act+assert pairs in one test are the smell; splitting restores the one-act structure.

Worked example:

```ruby
# bad: compound test — one failure hides the rest
test "order is processed" do
  result = order.process

  assert_equal :confirmed, result.status
  assert_equal 2, result.line_items.count
  assert_predicate result, :paid?
end

# good: three independent tests, each naming its aspect
test "status is confirmed after processing" do ... end
test "line item count is preserved after processing" do ... end
test "order is marked paid after processing" do ... end
```

**Enforcement:** `Minitest/MultipleAssertions` as a guide; review rejects multi-act tests or tests with clearly independent assertion groups.

### 11.5 — Assert a positive outcome; never use `assert_nothing_raised`.

**Reasoning, step by step:**
1. `assert_nothing_raised` was removed from Minitest because it inverts the test model: a test that passes only when nothing goes wrong is checking absence of explosion, not presence of correctness. It provides no signal about what the code actually did.
2. If an operation is expected to succeed, assert what success looks like: the return value, a side effect, a state transition. The positive assertion proves the operation did the right thing; an absence-of-exception check proves only that it did not raise.
3. If the test is genuinely checking that a code path is exception-safe under a specific precondition, structure it as: call the method, then assert the outcome. The lack of an exception is implicit in reaching the assertion.

Worked example:

```ruby
# bad: proves nothing about what the method did
assert_nothing_raised { order.process }

# good: proves the method produced the expected state
result = order.process
assert_equal :confirmed, result.status
```

**Enforcement:** `Minitest/NoAssertionInBlock` and `Minitest/AssertNothingRaised` cops; review rejects `assert_nothing_raised` in any form.

### 11.6 — Prefer fakes over mocks; a fake tests behaviour, a mock tests implementation.

**Reasoning, step by step:**
1. A mock that stubs `inventory.find` to return a canned value passes precisely when the code calls `find` in exactly the way you mocked it — and passes even when the logic is wrong in every other dimension. It couples the test to the call shape of the production code; refactor the internals and the mock breaks, even when behaviour is unchanged.
2. A fake is a real, in-memory implementation of the same interface: `FakeInventory` backed by a `Hash` stores and retrieves `Sku` values for real, exercises the actual call path, and survives any internal refactor that preserves behaviour.
3. Write a fake for every owned interface that crosses a test boundary: `FakeInventory`, `FakeOrderRepository`, `FakeCustomerNotifier`. Name doubles for what they are — never `MockInventory` for a hand-rolled fake; the name lies to the next reader.
4. Reserve true test doubles (stubs, mocks via `Minitest::Mock`) for genuine externals: a third-party payment gateway, a system clock, an SMS provider. The seam is the boundary of your own code; double across it, never inside it.

Worked example:

```ruby
class FakeInventory
  extend T::Sig

  sig { void }
  def initialize
    @stock = T.let({}, T::Hash[Sku, Integer])
  end

  sig { params(sku: Sku, qty: Integer).void }
  def add(sku, qty) = @stock[sku] = (@stock[sku] || 0) + qty

  sig { params(sku: Sku).returns(T.nilable(Integer)) }
  def available(sku) = @stock[sku]  # real behaviour, in memory
end
```

**Enforcement:** review; `Minitest::Mock` usage justified at an external boundary; `FakeX` naming convention enforced in review.

### 11.7 — Property-based tests for pure functions and value objects; bound the iteration count.

**Reasoning, step by step:**
1. An example test proves one input; a property test proves a law over thousands of generated inputs — including the empty, the maximal, and the adversarial edge a human never reaches. For anything that transforms or validates data, that breadth is the difference between "works on my three cases" and "works."
2. The canonical properties are few and reusable: **round-trip** (`parse(to_h(x)) == x`) for every codec and value object; **idempotence** (`f(f(x)) == f(x)`) for normalizers; **commutativity** for commutative operations; **bounds** (the output always lands in its declared range). Each maps directly to an invariant the production `sig` already asserts.
3. Bound generators explicitly: `rand(1..1_000_000)` rather than unbounded `rand`. Set a fixed iteration count (`run_count: 100` or similar) so CI runtime is predictable and a future contributor cannot accidentally widen it. Pin and log the random seed on failure so the shrunk counterexample is reproducible.
4. This is mandatory for: codecs, parsers, serializers, and any value object (`Money`, `Sku`, `LineItem`) with parse-constructor invariants (chapter [06](./06-classes-and-data-modeling.md)). These are precisely the functions whose failure modes hide in the input space and where the type system cannot help: a well-typed `Money.parse` can still have wrong logic.

Worked example:

```ruby
test "Money round-trips through to_h and parse for any valid amount" do
  100.times do
    amount   = rand(1..999_999)
    currency = %w[GBP USD EUR JPY].sample

    original  = Money.new(cents: amount, currency:)
    recovered = Money.parse(original.to_h)

    assert_equal original, recovered
  end
end
```

**Enforcement:** review; value objects and codec modules ship a round-trip property test; iteration count is explicit and bounded in source.

### 11.8 — Determinism: inject the clock and randomness; never read `Time.now` inside a unit under test.

**Reasoning, step by step:**
1. A test that depends on the wall clock, an unseeded random source, or the real filesystem is a flake waiting to happen: it passes locally, fails on slow CI, and erodes trust until a red build means nothing. Determinism is the precondition for a suite anyone believes.
2. Pass the clock as a parameter — `sig { params(clock: Time).returns(T::Boolean) }` — so tests provide a fixed `Time.new(2026, 1, 1, 0, 0, 0, "UTC")` in one line. A unit under test that reads `Time.now` directly is an API smell (see chapter [10](./10-api-design.md)): the hidden dependency is the design problem, not the test scaffold.
3. When you cannot refactor the caller (a third-party hook, a legacy boundary), use `Time.stub :now, fixed_time do ... end` — the Minitest `stub` block is scoped and never leaks. Prefer injection; reach for stub as the last resort.
4. Pin any random seed used inside a property test. Log it on failure so the sequence that found the counterexample is reproducible. `srand(seed)` before the block, rescue and re-raise after logging the seed.

Worked example:

```ruby
test "order is expired when current time is past deadline" do
  deadline = Time.new(2026, 1, 1, 12, 0, 0, "UTC")
  now      = Time.new(2026, 1, 1, 13, 0, 0, "UTC")
  order    = Order.new(expires_at: deadline)

  assert_predicate order.expired?(clock: now), :itself
end
```

**Enforcement:** `rubocop` custom cop banning `Time.now` and `Date.today` in `test/`; review requires injected clock or scoped `stub`; property seeds logged on failure.

### 11.9 — `srb tc` is the first test suite; a type error is a test failure caught before runtime.

**Reasoning, step by step:**
1. Sorbet's `srb tc` with `# typed: strict` and runtime-checked `sig` blocks is the first suite to run on every push — before Minitest, not after. A sig on a public method is a machine-checked assertion on every argument and return value; it catches an entire class of bugs that no runtime test can see until the wrong type reaches a branch that exercises it.
2. Treat a `srb tc` failure as a test failure: it blocks merge, it is not a "type warning," and it is never silenced with `T.untyped` without a recorded justification. Chapter [03](./03-type-safety-and-nil-discipline.md) governs `T.cast` and `T.must`; the same discipline applies in test files — test helpers carry sigs too.
3. Sigs are executable specs: `sig { params(order: Order, clock: Time).returns(T::Boolean) }` specifies the contract more precisely than a YARD comment (chapter [14](./14-documentation.md)). When the sig is wrong, `srb tc` finds it at commit time. When the implementation diverges from the sig, the runtime-checked wrapper finds it at the first call site in any environment.
4. Write `# typed: strict` on every test file. Test helpers are production-quality code that carry the same type discipline as `lib/`.

Worked example:

```ruby
# typed: strict

class OrderTest < Minitest::Test
  extend T::Sig

  test "discount applied for gold tier" do
    customer = T.let(Customer.new(tier: "gold"), Customer)
    order    = T.let(Order.new(customer:, total_cents: 1000), Order)

    result = order.apply_discount

    assert_equal 900, result.total_cents
  end
end
```

**Enforcement:** `srb tc` runs as the first CI step, before `bundle exec ruby -Itest`; a type error is a hard block; `# typed: strict` enforced by `rubocop-sorbet`'s `Sorbet/StrictSigil` cop on all files including `test/`.

### 11.10 — Test the negative space: assert errors at boundaries and pair-assert properties.

**Reasoning, step by step:**
1. A test that only asserts the happy path verifies that the code succeeds when everything goes right. The bug usually hides in the space you forgot to assert: the extra write, the swallowed error, the leaked delimiter. Negative space is not optional — it is the other half of correctness.
2. At every error boundary defined in chapter [08](./08-error-handling.md), pair a positive test with a negative one: assert the expected `StandardError` subclass is raised, check its message identifies the violating input, and verify no partial side effect leaked (no half-written record, no charged payment without a confirmation).
3. Pair-assert a property two independent ways. After computing a discount total: assert it equals the expected value (positive) *and* assert it is strictly less than the original total (negative). When the two derivations disagree, a bug surfaces at the assertion rather than three layers downstream.
4. `assert_raises` returns the exception — use it, assert the class and the message. Never rescue and ignore; a rescue that swallows the exception is not a negative-space assertion, it is a hole.

Worked example:

```ruby
test "raises InsufficientStock when quantity exceeds available" do
  inventory = FakeInventory.new
  inventory.add(Sku.new("WIDGET"), 5)
  order = Order.new(items: [LineItem.new(sku: Sku.new("WIDGET"), qty: 10)])

  error = assert_raises(Inventory::InsufficientStock) { order.reserve(inventory) }

  assert_match(/WIDGET/, error.message)         # message names the SKU
  assert_equal 5, inventory.available(Sku.new("WIDGET"))  # no stock was deducted
end
```

**Enforcement:** review; error-boundary methods in chapter [08](./08-error-handling.md) require a paired `assert_raises` test; `assert_raises` result is always used; review rejects empty rescue in tests.

### 11.11 — Tests are fast and isolated: bound data, no shared mutable state, no order dependence.

**Reasoning, step by step:**
1. Every test must run alone, in any order, and pass. A test that passes only after another has run is not a test — it is a fragment of one. Minitest randomizes test order by default; any order dependence is caught immediately. Never override the seed to paper over it.
2. Shared mutable fixtures — a module-level array a test pushes into, a repository instantiated once for the class — leak state between tests: one test's write is another's surprise, and a failure in test A masks the true cause in test B. Build every mutable fixture fresh: in the `test "..."` block, in `setup`, or from a factory method.
3. Immutable shared data (a frozen constant, a fixed `Sku` value) is safe to hoist. The rule is mutable state: if a test can change it, no other test may share it.
4. Bound data sizes and iteration counts. A property test that generates unbounded arrays or runs 10 000 iterations in CI is a slow test; slow tests are skipped. Cap generators (`rand(1..100)` not `rand`), set explicit run counts, and keep the whole suite well under 30 seconds.

Worked example:

```ruby
class OrderTest < Minitest::Test
  STANDARD_SKU = T.let(Sku.new("STANDARD").freeze, Sku)  # safe: immutable

  def setup
    @inventory = FakeInventory.new   # fresh mutable fixture per test
    @customer  = Customer.new(tier: "standard")
  end

  test "reserves stock against fresh inventory" do
    order = Order.new(customer: @customer, items: [LineItem.new(sku: STANDARD_SKU, qty: 1)])
    @inventory.add(STANDARD_SKU, 10)

    order.reserve(@inventory)

    assert_equal 9, @inventory.available(STANDARD_SKU)
  end
end
```

**Enforcement:** `Minitest/GlobalExpectations` cop; review bans module-level `@instance_variables` or mutable class-level variables shared across tests; CI runs `--seed random` on every push; iteration counts are numeric literals in source, not computed.

## Cross-references

- Formatting, `frozen_string_literal`, 2-space indent, 100-col limit, `rubocop` baseline: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Effect-verb naming, predicate `?` convention, test class names: [02-naming-conventions.md](./02-naming-conventions.md).
- `srb tc` as the first test suite, `sig` discipline, `T.let`/`T.cast` justification, `# typed: strict` on all files: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- Guard clauses, AAA paragraph discipline, assertion density, pure-by-default: [05-methods.md](./05-methods.md).
- `Data.define` value objects with parse-constructor invariants, `T::Enum`, illegal-state modeling: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `StandardError` subclasses, `raise` with class + message, `cause` chaining, never rescue `Exception`: [08-error-handling.md](./08-error-handling.md).
- Injected clocks, `Mutex`, bounded queues — concurrency determinism: [09-concurrency.md](./09-concurrency.md).
- Minimal public surface, `sig` on every public method, parse at boundaries — the test-as-first-caller principle: [10-api-design.md](./10-api-design.md).
- One file per class, `test/` mirroring `lib/`, Zeitwerk autoloading: [12-module-organization.md](./12-module-organization.md).
- YARD on public API, why-comments, no restating a `sig` in prose: [14-documentation.md](./14-documentation.md).
