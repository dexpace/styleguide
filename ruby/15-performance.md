# 15 — Performance

Performance in Ruby is won at design time, not in micro-optimization. Know the resource hierarchy, write code YJIT compiles well, and let the profiler name the hot path before you touch it.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

# before: p99 order-summary latency 38ms; stackprof blamed serial DB round-trips + GC pressure.
sig { params(order_ids: T::Array[Integer]).returns(T::Array[OrderSummary]) }
def summarize_orders(order_ids)
  # One bulk query instead of N per-row fetches (15.11); symbols for label keys (15.12).
  rows = Order.where(id: order_ids).includes(:line_items).to_a

  rows.map do |order|
    total = order.line_items.reduce(Money::ZERO) { |sum, li| sum + li.subtotal }
    OrderSummary.new(
      id:         order.id,
      customer:   order.customer_id,
      total:      total,
      item_count: order.line_items.size,  # size, not count (15.8)
      status:     order.status,
    )
  end
end

# after: p99 38ms -> 4ms; GC time down 6x; one query replaces N+1.
sig { params(skus: T::Array[String]).returns(String) }
def sku_report(skus)
  buf = +""  # mutable string; << mutates in place (15.6)
  skus.each { |sku| buf << sku << "\n" }
  buf.freeze
end
```

`summarize_orders` wins by eliminating the N+1 fetch in one `includes` call — a network/disk win that dwarfs any CPU tuning (15.2). `size` reads the already-loaded association length without re-querying (15.8). `sku_report` builds with `<<` so no intermediate `String` is allocated on each iteration (15.6). Both examples obey object-shape discipline: `OrderSummary.new` is called with the same keyword arguments in the same order every time, so YJIT's inline caches see one shape (15.4). The measurement note in the comment is the ledger entry (15.1) that earned these deviations from the straightforward form.

## Rules

### 15.1 — Measure first; profile the hot path, not a guess.

**Reasoning, step by step:**
1. Intuition about Ruby performance is wrong at least half the time. YJIT, the GC, object shapes, and the C-extension boundary interact non-locally. A hypothesis is not a result.
2. For CPU: `stackprof` with wall-clock or CPU mode names the frame that actually burns time. For memory: `memory_profiler` reports allocation counts and sources. For micro-comparisons: `benchmark-ips` (iterations-per-second) measures a function in isolation with a warm runtime. Use all three: they answer different questions.
3. Capture the profile before changing any code, apply the fix, capture again. The delta is the only proof. Store a benchmark beside the code it guards — a committed `bench/` file turns "we made this faster once" into a regression guard the next refactor must beat.
4. Never touch a cold path. The GC's generational minor collection makes short-lived allocation cheap in code the profiler never flags; clean code there beats a saved allocation everywhere the flamegraph is flat.

**Enforcement:** A PR claiming a performance win attaches before/after numbers (`stackprof` flamegraph, `benchmark-ips` output, or `memory_profiler` totals) in the description. Optimization without a committed profile or benchmark does not merge.

### 15.2 — Optimize the slowest resource first: network > disk > memory > CPU.

**Reasoning, step by step:**
1. Resource costs differ by orders of magnitude: a network round-trip at 5–50ms dwarfs a CPU operation at nanoseconds. One eliminated query beats a thousand micro-optimizations. Choose the layer to work on from the profile, not from the code.
2. The order is fixed: **network > disk > memory > CPU**. If the profiler shows DB time, fix the query or add a batch. If it shows GC, reduce allocations. Only reach for CPU tricks when the design layer is sound and CPU is genuinely the constraint.
3. This is root rule 11 restated precisely. Architecture is chosen once, at design time, and is expensive to retrofit; get the resource hierarchy right first.

**Enforcement:** Design review against the four-resource hierarchy. A CPU micro-fix proposed before a profile names CPU as the bottleneck is sent back; fix the slowest resource named in the profile.

### 15.3 — Ship under a JIT and write code it compiles well.

**Reasoning, step by step:**
1. YJIT is Ruby's stable production compiler; enable it with `--yjit` or `RUBY_YJIT_ENABLE=1`. Ruby 4.0 also ships ZJIT, YJIT's successor, compiled into the binary but not enabled at runtime by default — evaluate it, but keep YJIT as the default until ZJIT matches it. A JIT compiles call sites monomorphically: a site that always dispatches to the same method with the same argument types becomes near-native code, while a megamorphic site falls back to interpretation.
2. The JIT rewards predictable, simple call sites. Avoid heavy `method_missing`, `send` with dynamic names, and `define_method` in hot paths — each defeats the inline cache and forces a generic dispatch.
3. Benchmark with the JIT you ship enabled. A micro-benchmark run with `--disable-yjit` measures a different runtime; the numbers do not transfer. Profile in the environment you ship.

**Enforcement:** Benchmarks capture the `ruby --yjit` baseline. Reviewers flag `send`/`method_missing` on measured hot paths and redirect to a direct call or a dispatch table.

### 15.4 — Assign every instance variable in `initialize`, in a consistent order.

**Reasoning, step by step:**
1. Ruby 3.2+ tracks object shapes: the combination of an object's ivar names and their assignment order. YJIT caches ivar accesses by shape. Two objects of the same class that assigned ivars in different orders, or assigned different subsets, get different shapes; YJIT's cache misses and the access degrades to a hash lookup.
2. Assign every ivar your class will ever use inside `initialize`, in one deterministic order. Never conditionally define an ivar outside `initialize`; that forks the shape into a transition chain.
3. This mirrors chapter 03's `T.let` ordering discipline: the Sorbet annotation and the shape-stability rule agree — declare all state upfront, in order, always.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class Inventory
  sig { params(sku: Sku, quantity: Integer, reserved: Integer).void }
  def initialize(sku, quantity, reserved)
    @sku      = T.let(sku,      Sku)      # consistent order: same shape every construction
    @quantity = T.let(quantity, Integer)
    @reserved = T.let(reserved, Integer)
    # never: if condition then @reserved = ... end  — that forks the shape
  end
end
```

**Enforcement:** RuboCop and Sorbet `# typed: strict` flag uninitialized ivars. Reviewers reject ivar assignments outside `initialize` on objects in hot paths.

### 15.5 — Freeze string literals; reuse buffers; allocate less.

**Reasoning, step by step:**
1. `# frozen_string_literal: true` at the top of every file interns string literals: the same literal produces the same object reference instead of a fresh allocation each time. This is mandatory per chapter 01, and its performance benefit is free.
2. The GC cost is the *rate*, not the instance. One allocation is trivially cheap; the same allocation inside the inner loop of a hot path runs millions of times per second and accumulates pause time. Hoist allocations out of loops; build output into a pre-allocated buffer; reuse rather than reallocate.
3. Mutable working strings created with `+""` (an unfrozen duplicate of an empty literal) or `String.new` serve as reusable buffers. Freeze the final result before returning it across a boundary.

**Enforcement:** `frozen_string_literal: true` is enforced by `rubocop` (`Style/FrozenStringLiteralComment`). Reviewers flag allocation-per-iteration patterns on measured hot paths.

### 15.6 — Build strings with `<<`; never `String#+` in a loop.

**Reasoning, step by step:**
1. `String#+` allocates a new `String` object on every call. Inside a loop, each iteration discards the previous string and hands the GC a new object to collect. For N iterations the cost is O(N) allocations and up to O(N²) bytes copied.
2. `String#<<` mutates the receiver in place: no new object, no copy of the accumulated result, O(1) per append. Use it whenever a string is being assembled incrementally.
3. When the string must be immutable at the boundary, freeze it after assembly — do not use `String#+` to avoid mutability. Build with `<<` into a mutable buffer, return `buf.freeze`.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(items: T::Array[LineItem]).returns(String) }
def csv_lines(items)
  buf = +""
  items.each do |item|
    buf << item.sku.to_s << "," << item.quantity.to_s << "\n"
  end
  buf.freeze
end
```

**Enforcement:** Review; `rubocop` `Performance/StringConcatenationInLoop` flags `String#+` inside a block or loop body.

### 15.7 — Use `.lazy` for large, streamed, or infinite sequences.

**Reasoning, step by step:**
1. Chained `Enumerable` methods (`map`, `select`, `find`) are eager by default: each stage materializes a full intermediate array. For large or infinite sequences this is both a memory and a latency problem — you pay for the whole collection before consuming the first element.
2. Prepend `.lazy` to the chain. Lazy enumerators pull one element at a time through every stage; no intermediate array is built. Call `.first(n)` or `.take(n).to_a` at the end to force evaluation of only the elements you need.
3. Use `lazy` when: the source is a large collection, an `IO`/`Enumerator`, or infinite; or when you consume only a prefix. Do not use it on small, finite collections where eager is clearer and the GC pressure is negligible.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(inventory: T::Enumerable[Inventory], limit: Integer).returns(T::Array[Sku]) }
def low_stock_skus(inventory, limit)
  inventory
    .lazy
    .select { |inv| inv.quantity < inv.reorder_threshold }
    .map(&:sku)
    .first(limit)
end
```

**Enforcement:** Review; `rubocop` `Performance/Count` and similar cops flag materialized chains where lazy would avoid a full traversal.

### 15.8 — Prefer `size` over `count` or `length` where equivalent.

**Reasoning, step by step:**
1. On Ruby `Array`, `Hash`, and `String`, `size` and `length` are aliases: O(1) cached-length reads. `count` without a block is also O(1) on `Array` and `Hash`; but on `ActiveRecord::Relation` and many `Enumerable` sources, `count` executes a `SELECT COUNT(*)` or iterates the entire collection. The distinction matters enormously at the data layer.
2. Prefer `size` as the default: it signals "read the cached length" and never triggers a query or a full traversal on any standard Ruby type. Reserve `count { |e| predicate }` for the counted-predicate form, where it is semantically distinct.
3. When working with ActiveRecord, `size` checks whether the association is already loaded and returns the cached length if so, falling back to a `COUNT` only when unloaded. `count` always issues a query. Load the association once and use `size`.

**Enforcement:** RuboCop `Performance/Size` (prefer `size` over `count` for arrays/strings without a block). Reviewers replace `count` on already-loaded associations with `size`.

### 15.9 — Use the most specific string method; avoid `gsub` or a regex when a simpler method suffices.

**Reasoning, step by step:**
1. `gsub` compiles and executes a regex on every call. When the operation is an exact-string replacement (`sub`), a single-character translation (`tr`), a character deletion (`delete`), or a prefix/suffix check (`start_with?` / `end_with?`), the specialized method is faster and communicates intent more precisely.
2. The performance difference is not noise: regex engine startup and backtracking overhead is real, especially when the pattern could be expressed as a literal. Use the specialized form and save the regex for problems that actually require it.
3. Hierarchy: `start_with?` / `end_with?` for prefix/suffix checks; `delete` for character-set removal; `tr` for single-character translation tables; `sub` for a single exact-string replacement; `gsub` only when you need global replacement of a pattern that is genuinely a regex.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sku.gsub("-", "_")          # bad — regex for a literal character translation
sku.tr("-", "_")            # good — O(n) single-pass translation

label.gsub(/^Order: /, "")  # bad — regex for a literal prefix removal
label.delete_prefix("Order: ")  # good — purpose-built, no regex

code.start_with?("SKU-")   # good — no regex needed for a prefix check
```

**Enforcement:** RuboCop `Performance/StringReplacement`, `Performance/StartWith`, `Performance/EndWith`. Reviewers flag `gsub` with a string-literal pattern and redirect to the specialized method.

### 15.10 — Use `Hash#fetch` with a default block; use `Hash.new` for default factories.

**Reasoning, step by step:**
1. `hash[key] || default` recomputes `default` on every miss and mishandles falsy stored values: a stored `false` or `0` triggers the default incorrectly. `hash.fetch(key) { default }` evaluates its block only on miss and raises `KeyError` on unrecoverable absence, making the miss explicit.
2. `Hash.new { |h, k| h[k] = expensive_default(k) }` computes and caches the default on first access. The alternative — `hash[key] ||= expensive_default(key)` — recomputes on every falsy value and allocates the result without caching it into the hash. The factory hash pays the cost once per key.
3. Memoization via `||=` is appropriate for simple, truthy, idempotent defaults on instance variables (chapter 04 rule). For hash values, `fetch` + a block or a factory hash is the correct tool and is consistent with the chapter 03 `Hash#fetch` over `[]` discipline.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# Grouping line items by SKU — build with a factory hash, not repeated ||= chains.
sig { params(items: T::Array[LineItem]).returns(T::Hash[Sku, T::Array[LineItem]]) }
def group_by_sku(items)
  groups = Hash.new { |h, k| h[k] = [] }
  items.each { |item| groups[item.sku] << item }
  groups
end

# Safe lookup with an explicit miss block:
price = prices.fetch(sku) { Money::ZERO }
```

**Enforcement:** RuboCop `Style/FetchEnvVar` (analogous pattern); review for `hash[key] || default` on hot paths where the value could be falsy or the computation is non-trivial.

### 15.11 — Batch over per-row work; never issue N+1 queries or requests.

**Reasoning, step by step:**
1. A loop that issues one query or HTTP request per element transforms O(1) I/O cost into O(N) I/O cost. At N = 100 rows and 5ms per round-trip, 500ms of latency is manufactured from nothing. This is the single most common performance defect in Ruby applications.
2. Collect all ids or keys upfront, issue one bulk query (`WHERE id IN (...)`), then index the results in memory for O(1) lookup. For external APIs, batch the request if the API supports it; otherwise fan out with bounded concurrency (chapter 09) rather than serial iteration.
3. ORMs make N+1 invisible: `order.line_items` inside an `orders.each` loop is a query per order. Use eager loading (`includes`, `preload`, `eager_load`) at the query site; confirm the plan with `bullet` or SQL logs.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# bad — one query per order (N+1):
# orders.each { |order| process(order.customer) }

# good — one query for all customers, then index by id:
sig { params(orders: T::Array[Order]).returns(T::Array[Receipt]) }
def fulfill_orders(orders)
  customer_ids = orders.map(&:customer_id)
  customers    = Customer.where(id: customer_ids).index_by(&:id)

  orders.map do |order|
    customer = customers.fetch(order.customer_id)
    Receipt.new(order:, customer:)
  end
end
```

**Enforcement:** `bullet` gem in development/test raises on N+1 queries. Reviewers reject query-inside-loop patterns. SQL logs in tests catch unintentional N+1 introduced by refactors.

### 15.12 — Prefer symbols over strings for keys and labels.

**Reasoning, step by step:**
1. A symbol is interned: `:status` is one object for the lifetime of the process. A string `"status"` is allocated fresh at every string-literal evaluation (mitigated by `frozen_string_literal: true`, which interns string *literals* — but dynamic strings, string keys from JSON, and string construction still allocate). Symbols remove the allocation entirely.
2. Hash keys that are symbols allow Ruby to use a faster internal key-lookup path. The shorthand hash syntax (`status:`, `sku:`) and Sorbet's `T::Struct` field names are already symbol-keyed; stay consistent and use symbols for all internal keys and option labels.
3. The boundary exception: external data (JSON, query parameters, environment) arrives as strings. Convert at the boundary (`to_sym` or `symbolize_keys`) and work with symbols internally — or use `Hash#fetch` with string keys only in the adapter layer. Do not scatter `to_sym` across the codebase; do it once at the entry point.

**Enforcement:** RuboCop `Performance/InefficientHashSearch` and `Style/HashSyntax` (enforce shorthand symbol keys). Reviewers flag string keys on internal hashes and option sets.

## Cross-references

- Mandatory `frozen_string_literal: true`, `rubocop-airbnb` baseline, and the `srb tc` typecheck gate: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Sorbet `T.let` ordering and ivar initialization discipline that aligns with object-shape stability (15.4): [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `Hash#fetch` over `[]`, `||=` memoization caveats, and constant freezing: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Method size cap, guard clauses, and assertion density — the method-level discipline that keeps hot paths narrow: [05-methods.md](./05-methods.md).
- `Data.define` value objects and one-site construction — the shape-stability strategy (15.4): [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- `map`/`select`/`reduce`/`find`, lazy enumerators, and `&:sym` — the idiomatic pipeline tools this chapter tunes: [07-ruby-idioms.md](./07-ruby-idioms.md).
- Bounded concurrency, Ractors, and deadline timeouts for bounded fan-out behind rule 15.11: [09-concurrency.md](./09-concurrency.md).
- Resource pools, bounded caches, and deterministic teardown for allocation-reuse patterns (15.5): [13-resource-management.md](./13-resource-management.md).
