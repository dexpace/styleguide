# 09 — Concurrency

Ruby 4.0 offers three concurrency models — threads, Ractors, and Fibers — each suited to a different workload. Choose the right model deliberately, bound every pool and queue, replace `Timeout.timeout` with per-call deadlines, and tear down every resource on exit; the rules here keep concurrency correct, bounded, and leak-free.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

require "async"
require "async/semaphore"
require "concurrent-ruby"

class OrderFulfillmentWorker
  extend T::Sig

  MAX_CONCURRENT = T.let(8, Integer)
  FETCH_TIMEOUT  = T.let(5.0, Float)

  sig { params(order_ids: T::Array[String]).returns(T::Array[FulfillmentResult]) }
  def self.fulfill_batch(order_ids)
    raise ArgumentError, "order_ids exceeds cap of 500" if order_ids.size > 500 # 9.5, 9.7

    results = Concurrent::Array.new # 9.10 — thread-safe collection

    Async do |task|
      semaphore = Async::Semaphore.new(MAX_CONCURRENT) # 9.5 — bounded fan-out

      order_ids.each do |order_id|
        semaphore.async do
          task.with_timeout(FETCH_TIMEOUT) do # 9.6 — per-call deadline, not Timeout.timeout
            inventory = Inventory.fetch(order_id) # I/O call deadlined above
            result    = Fulfillment.process(order_id, inventory)
            results << FulfillmentResult.new(order_id:, result:) # 9.8 — Data value, safe across fibers
          end
        rescue => error
          results << FulfillmentResult.new(order_id:, error: error.message)
        end
      end
    end

    raise "expected #{order_ids.size} results, got #{results.size}" unless results.size == order_ids.size # postcondition

    results.freeze
  end
end
```

This exemplar picks Fibers + `async` because the work is I/O-bound — the GVL is irrelevant and Ractors would be overkill (9.1, 9.4). Fan-out is bounded to `MAX_CONCURRENT` via `Async::Semaphore` (9.5); each I/O call carries an explicit per-call deadline via `task.with_timeout` rather than the state-corrupting `Timeout.timeout` (9.6). Results accumulate in a `Concurrent::Array` rather than a hand-rolled lock (9.10). Each `FulfillmentResult` is an immutable `Data` value object — safe to share across fiber and thread boundaries without synchronization (9.8). The postcondition asserts result count before returning (root rule 8).

## Rules

### 9.1 — Know the GVL; match the model to the workload.

**Reasoning, step by step:**
1. The Global VM Lock (GVL, formerly GIL) prevents two Ruby threads from executing bytecode simultaneously on one process. For CPU-bound work, threads give no parallelism — only one runs at a time.
2. I/O releases the GVL. A thread blocked on a network call, disk read, or `sleep` yields the lock so another thread can run. Multiple threads therefore give genuine concurrency for I/O-bound work.
3. True CPU parallelism requires Ractors (9.2) or multiple processes. Spinning up threads to parallelize a heavy computation achieves nothing and adds synchronization cost.
4. Choose the model at design time: Fibers + `async` for high-volume I/O (9.4), threads + `Mutex`/`Queue` for moderate I/O with shared state (9.3), Ractors for CPU-parallel work (9.2), multiple processes for isolation without Ractor restrictions.

**Enforcement:** review; code-review rejection of threads used for CPU-bound parallel work without Ractors or multiple processes.

### 9.2 — Use Ractors for CPU parallelism; share only frozen objects.

**Reasoning, step by step:**
1. A Ractor is Ruby 4.0's unit of true parallelism: each has its own GVL, so `N` Ractors on an `N`-core machine can execute bytecode simultaneously.
2. Ractors enforce their own isolation: they cannot access mutable objects from outside their scope. Only frozen objects, `Ractor.make_shareable` values, and primitives cross Ractor boundaries safely. This restriction is the price of parallelism; pay it by modeling inter-Ractor data as immutable `Data` value objects (9.8).
3. Bound the Ractor count to roughly the machine's core count (`Etc.nprocessors`). Spawning one Ractor per work item is unbounded fan-out (9.5) and overwhelms the scheduler.
4. Ractors are the heavy option; reach for them only when profiling shows the GVL is the bottleneck. I/O-bound work belongs to Fibers or threads, not Ractors.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

WORKERS = T.let(Etc.nprocessors, Integer)

sig { params(skus: T::Array[String]).returns(T::Array[Money]) }
def self.price_all(skus)
  partitions = skus.each_slice((skus.size.fdiv(WORKERS)).ceil).to_a

  ractors = partitions.map do |partition|
    Ractor.new(partition.freeze) do |ids|
      ids.map { |id| PricingEngine.calculate(id) } # PricingEngine must be Ractor-safe
    end
  end

  ractors.flat_map(&:take)
end
```

**Enforcement:** review; Ractor count bounded to core count; non-frozen objects crossing boundaries are a Ractor runtime error — treat them as a compile-time concern during review.

### 9.3 — Use `Mutex#synchronize` for shared mutable state; prefer immutable sharing and message passing.

**Reasoning, step by step:**
1. When threads share mutable state, a `Mutex` makes every read-modify-write atomic. Without it, two threads can interleave their reads and writes, producing lost updates or corrupt data.
2. A lock is an admission that you have mutable shared state. Prefer eliminating that state: immutable `Data` value objects need no synchronization (9.8); a `Queue` or `SizedQueue` makes producer-consumer coordination explicit without a bare `Mutex`.
3. Protect the smallest possible critical section — only the statements that actually touch shared state (9.9). A coarse lock that spans an I/O call couples concurrency and latency and risks deadlock.
4. When in doubt, reach for `concurrent-ruby` primitives (`Concurrent::Map`, `Concurrent::Array`) that have synchronization built in rather than wrapping a stdlib collection yourself (9.10).

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class InventoryTracker
  extend T::Sig

  sig { void }
  def initialize
    @mutex   = T.let(Mutex.new, Mutex)
    @reserve = T.let(Concurrent::Map.new, Concurrent::Map) # 9.10 — prefer concurrent-ruby
  end

  sig { params(sku: String, qty: Integer).void }
  def reserve(sku, qty)
    @mutex.synchronize { @reserve[sku] = (@reserve[sku] || 0) + qty } # 9.9 — minimal section
  end
end
```

**Enforcement:** review; bare `Mutex.new` + manual synchronization should be replaced by `Concurrent::Map`/`Concurrent::Array` where possible; `@mutex.synchronize` never spans an I/O call.

### 9.4 — Use Fibers + `async` for high-concurrency I/O.

**Reasoning, step by step:**
1. Threads are OS-managed and carry a fixed memory overhead (~1 MB stack) and a context-switch cost. For workloads with thousands of concurrent I/O waits, that overhead compounds fast.
2. Fibers are Ruby's cooperative coroutines: they yield control explicitly (or via `Fiber.scheduler`) rather than being preempted, and they live entirely in user space. The `async` gem (socketry/async) provides a scheduler that parks a Fiber at each I/O wait and resumes it when data arrives — multiplexing thousands of Fibers over a small thread pool.
3. `Async` blocks compose naturally: an outer `Async do` creates a task; inner `semaphore.async` blocks spawn child tasks. All child tasks complete before the outer block returns — structured concurrency without manual joining.
4. Reserve threads for integrations that cannot use the Fiber scheduler (C extensions with blocking calls, background work that must span process-level isolation). Keep I/O-heavy hot paths on Fibers.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(customer_ids: T::Array[String]).returns(T::Array[Customer]) }
def self.load_customers(customer_ids)
  results = Concurrent::Array.new

  Async do |task|
    semaphore = Async::Semaphore.new(16)
    customer_ids.each do |id|
      semaphore.async { results << CustomerRepository.fetch(id) }
    end
  end

  results.sort_by(&:id)
end
```

**Enforcement:** review; new `Thread.new` for I/O-bound fan-out is rejected in favour of `Async`/`Semaphore`.

### 9.5 — Bound every pool and queue.

**Reasoning, step by step:**
1. Spawning one thread or Fiber per work item is an unbounded fan-out: for a list of 10,000 orders you get 10,000 concurrent database connections, file descriptors, and memory allocations. Under load this exhausts the dependency.
2. Use `Concurrent::FixedThreadPool` (not `Concurrent::CachedThreadPool` or raw `Thread.new`) for thread-based fan-out. Use `Async::Semaphore` for Fiber-based fan-out. In both cases, declare the bound as a named constant and document why you chose that number.
3. Use `SizedQueue` instead of `Queue` for producer-consumer channels. An unbounded `Queue` lets producers race arbitrarily ahead of consumers; `SizedQueue` applies backpressure when the buffer is full.
4. Bound at every level: the pool, the queue feeding it, and the total in-flight work. Cascade the bounds so no single layer can accumulate unbounded state.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

LINE_ITEM_WORKERS = T.let(4, Integer)
LINE_ITEM_QUEUE   = T.let(200, Integer)

pool  = Concurrent::FixedThreadPool.new(LINE_ITEM_WORKERS)
queue = SizedQueue.new(LINE_ITEM_QUEUE) # blocks producer when full

line_items.each { |item| pool.post { process_line_item(item) } }
pool.shutdown
pool.wait_for_termination # 9.11 — deterministic teardown
```

**Enforcement:** `rubocop` custom cop banning `Thread.new` inside loops; review rejects `Queue.new` where `SizedQueue.new` belongs; pool size and queue bound must be named constants.

### 9.6 — Never use `Timeout.timeout`; apply per-call deadlines instead.

**Reasoning, step by step:**
1. `Timeout.timeout` raises `Timeout::Error` from a background thread at an arbitrary point in the protected block's execution — mid-assignment, mid-rescue, mid-transaction commit. There is no safe place for an asynchronous exception to arrive; it leaves objects in partially mutated states and is essentially un-rescueable.
2. The `async` gem's `task.with_timeout(seconds)` cancels by unwinding at the next Fiber yield point — a cooperative, deterministic cancellation that does not corrupt intermediate state.
3. For blocking I/O in threads, configure the client library's own timeout option (`:read_timeout`, `:connect_timeout`, `:open_timeout`). Every mature I/O library — `net-http`, `redis`, `pg`, `faraday` — exposes these options; use them rather than wrapping the call in `Timeout.timeout`.
4. Set two timeouts on every external call: a connect timeout and a read/operation timeout. They model different failure modes: a slow DNS resolution versus a connection accepted but data never arriving.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# bad — raises at an arbitrary bytecode boundary; can corrupt state
Timeout.timeout(5) { inventory_client.fetch(sku) }

# good — deadline honoured at the next Fiber yield; safe cooperative cancellation
Async do |task|
  task.with_timeout(5.0) { inventory_client.fetch(sku) }
end

# good — per-call library timeout for blocking thread I/O
pg_conn = PG.connect(host: "db", connect_timeout: 3, options: "-c statement_timeout=5000")
```

**Enforcement:** review and custom RuboCop cop banning `Timeout.timeout` in application code; I/O client constructors must include timeout options.

### 9.7 — Cap retries with bounded backoff.

**Reasoning, step by step:**
1. Unbounded retries under backpressure make a bad situation worse: a downstream dependency that is slow gets hammered with an exponentially growing retry storm from every caller.
2. Apply a fixed maximum retry count and exponential backoff with jitter. Jitter desynchronizes retriers so they do not thundering-herd on the same interval.
3. Declare the bounds as named constants — `MAX_RETRIES`, `BASE_DELAY_SECONDS`, `MAX_DELAY_SECONDS` — so they are visible in code review and tuneable without hunting through inline literals.
4. After exhausting retries, raise a typed `StandardError` subclass (chapter 08) that propagates up; do not swallow the final failure or return a sentinel `nil`.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

MAX_RETRIES       = T.let(3, Integer)
BASE_DELAY        = T.let(0.5, Float)
MAX_DELAY         = T.let(8.0, Float)

sig { params(order_id: String).returns(Inventory) }
def self.fetch_with_retry(order_id)
  attempts = 0

  begin
    InventoryClient.fetch(order_id)
  rescue InventoryClient::TransientError => error
    attempts += 1
    raise if attempts > MAX_RETRIES

    delay = [BASE_DELAY * (2**(attempts - 1)) + rand(0.1), MAX_DELAY].min
    sleep(delay)
    retry
  end
end
```

**Enforcement:** review; retry loops without a max-attempts cap are rejected; delay constants must be named.

### 9.8 — Use immutable `Data` value objects across thread and Ractor boundaries.

**Reasoning, step by step:**
1. Sharing a mutable object across threads requires synchronization. Sharing a frozen object requires nothing — immutability is the free lock.
2. Ruby 4.0's `Data.define` produces frozen-by-construction value objects; every instance is deeply immutable. Model cross-thread payloads, Ractor messages, and queue items as `Data` values rather than `Hash` or `Struct`.
3. A `Data` value also survives Ractor transfer without `make_shareable` — the runtime already knows it is shareable. A plain `Hash` or mutable `Struct` raises a `Ractor::IsolationError` at the transfer site.
4. Document every place a value crosses a concurrency boundary and name the invariant that holds: "frozen `LineItem` — safe for Ractor transfer" rather than leaving the reader to reason about it.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# Value objects are frozen-by-construction; no synchronization needed at transfer
LineItem = Data.define(:sku, :quantity, :unit_price)
item     = LineItem.new(sku: "SKU-001", quantity: 2, unit_price: Money.new(cents: 1000, currency: "USD"))

# Ractor transfer is safe — Data is inherently shareable
ractor = Ractor.new(item) { |li| li.quantity * li.unit_price.amount }
result = ractor.take
```

**Enforcement:** review; `Hash` or mutable `Struct` crossing a thread or Ractor boundary is rejected in favour of `Data.define`; every such boundary has a comment naming the invariant.

### 9.9 — Minimize the critical section; never hold a lock across I/O.

**Reasoning, step by step:**
1. A critical section is the span of code serialized by a lock. The wider the section, the more threads queue behind it — coarse locks become the bottleneck under load.
2. I/O inside a lock compounds the problem: the lock is held for the entire I/O latency (tens to hundreds of milliseconds). Every other thread waiting on that lock is blocked for the duration, turning a network hiccup into application-wide stall.
3. I/O inside a lock also risks deadlock when two locks protect two resources and two threads acquire them in opposite order — a classic AB/BA deadlock, impossible to trigger without the second lock but trivially created by adding a blocking call.
4. Restructure to do all I/O outside the lock, then enter the lock for the pure in-memory update: fetch-then-lock, not lock-then-fetch.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# bad — I/O inside the lock; every thread serializes on network latency
@mutex.synchronize { @cache[sku] = InventoryClient.fetch(sku) }

# good — I/O outside the lock; lock protects only the in-memory write
inventory = InventoryClient.fetch(sku) # I/O without the lock
@mutex.synchronize { @cache[sku] = inventory }
```

**Enforcement:** review; `@mutex.synchronize` blocks containing I/O calls are rejected; critical sections touch only in-memory state.

### 9.10 — Prefer `concurrent-ruby` primitives over hand-rolled synchronization.

**Reasoning, step by step:**
1. Hand-rolled synchronization is hard to get right: it is easy to forget to release a lock on the error path, to miss a memory barrier, or to reach for the wrong primitive for the access pattern. `concurrent-ruby` has been battle-tested against all of those failure modes.
2. `Concurrent::Map` and `Concurrent::Array` are thread-safe drop-in collections that need no external lock for their own operations. Use them wherever multiple threads read or write the same collection.
3. `Concurrent::AtomicFixnum` and `Concurrent::AtomicBoolean` replace `@mutex.synchronize { @count += 1 }` patterns with a single atomic operation. `Concurrent::Future` and `Concurrent::Promise` cover deferred computation. `Concurrent::FixedThreadPool` is the bounded worker pool for 9.5.
4. When `concurrent-ruby` does not cover the pattern, reach for `Queue`/`SizedQueue` (stdlib) or `Async::Semaphore` before hand-rolling a `Mutex` solution.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# bad — hand-rolled mutex for a counter
@mutex  = Mutex.new
@filled = 0
@mutex.synchronize { @filled += 1 }

# good — single atomic operation, no external lock
@filled = Concurrent::AtomicFixnum.new(0)
@filled.increment

# good — thread-safe map for shared state
@sku_index = Concurrent::Map.new
@sku_index.put_if_absent(sku.id, sku)
```

**Enforcement:** review; custom `Mutex` + standard `Hash`/`Array` is replaced by `Concurrent::Map`/`Concurrent::Array`; `Concurrent::AtomicFixnum` for shared counters.

### 9.11 — Deterministic teardown: join threads, shut down pools, close queues.

**Reasoning, step by step:**
1. A thread or pool that is not joined before the process exits may be mid-operation — holding a database transaction open, writing to a file, or holding a lock. The OS will kill it mid-step, corrupting the operation.
2. A `SizedQueue` that is not closed leaves its consumer threads blocked on `deq` forever — a thread leak that prevents clean shutdown.
3. `Concurrent::FixedThreadPool` exposes `shutdown` (no new work accepted) and `wait_for_termination` (blocks until in-flight work drains). Call both. `Async` structured tasks drain automatically when the outer `Async do` block exits — that cleanup is free when you use structured concurrency.
4. Register pool and thread shutdown in an `at_exit` hook or inside an `ensure` block (chapter 13) so teardown happens even if the main thread raises. A leaked thread is a leaked resource (root rule 9).

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

pool  = Concurrent::FixedThreadPool.new(4)
queue = SizedQueue.new(100)

begin
  orders.each { |order| pool.post { fulfill(order) } }
ensure
  pool.shutdown
  pool.wait_for_termination(30) || warn("pool did not drain in 30 s")
  queue.close
end
```

**Enforcement:** review; every `Concurrent::FixedThreadPool` has a paired `shutdown` + `wait_for_termination` in `ensure`; every `SizedQueue` is `close`d on exit; cross-reference chapter 13.

## Cross-references

- Formatting, `frozen_string_literal`, and 100-col cap: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md)
- Effect-verb naming for concurrent workers and jobs: [02-naming-conventions.md](./02-naming-conventions.md)
- Sorbet `sig` on every concurrent method, immutable `T.let` constants: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md)
- `Data.define` frozen value objects as the cross-boundary payload type: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md)
- `StandardError` subclasses for retry exhaustion and concurrency errors: [08-error-handling.md](./08-error-handling.md)
- Block form for all closable resources, `ensure` cleanup, bounded caches: [13-resource-management.md](./13-resource-management.md)
- YJIT, object shapes, and avoiding allocation churn in concurrent hot paths: [15-performance.md](./15-performance.md)
