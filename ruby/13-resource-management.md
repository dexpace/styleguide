# 13 — Resource Management

File handles, database connections, sockets, locks, and thread stacks outlive the call that opens them unless something explicitly closes them — and that something must run on the exceptional path too. Ruby's block form and `ensure` are the language's answer: a block-taking method guarantees close; `ensure` is the fallback when no block form exists. Neither delegates cleanup to the garbage collector, neither leaves ordering to chance.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

class OrderExporter
  extend T::Sig

  MAX_CONNECTIONS = T.let(10, Integer)
  EXPORT_TIMEOUT  = T.let(30.0, Float)

  sig { params(orders: T::Array[Order], dest: String).returns(Integer) }
  def self.export(orders, dest)
    raise ArgumentError, "orders cannot be empty" if orders.empty?

    written = T.let(0, Integer)

    ConnectionPool.new(size: MAX_CONNECTIONS) do # 13.3 — bounded pool, named constant
      db_conn = Sequel.connect(ENV.fetch("DATABASE_URL"))
      db_conn
    end.with do |db| # 13.1 — block form; pool releases on any exit
      Tempfile.create(["order_export", ".csv"]) do |tmp| # 13.1 — block form; file closed on any exit
        Timeout.new(EXPORT_TIMEOUT).wrap do # 13.5 — deadline on every external I/O
          orders.each do |order|
            row = ExportRow.build(order, db)
            tmp.write(row.to_csv)
            written += 1
          end
        end
        FileUtils.mv(tmp.path, dest)
      end
    end

    raise "expected #{orders.size} rows, wrote #{written}" unless written == orders.size # postcondition (13.2, 13.7)

    written
  end
end
```

The pool is bounded by `MAX_CONNECTIONS`, a named constant (13.3); both the connection and the temp file use block form so they close on `return`, `raise`, or normal exit (13.1); the `Timeout` wrapper enforces a deadline on the I/O loop rather than letting a hung write hold the file handle forever (13.5); teardown order is innermost-first — file closes before the connection is returned to the pool (13.7); and a postcondition asserts row count before returning (13.2).

## Rules

### 13.1 — Use block form for every closable resource.

**Reasoning, step by step:**
1. `File.open(path) { |f| ... }`, `Tempfile.create(["name", ".ext"]) { |f| ... }`, `pool.with_connection { |c| ... }` — the block form guarantees the underlying `close` runs when the block exits by any path: normal return, uncaught exception, or explicit `raise`. Manual open/close pairs have no such guarantee; an exception between them leaks the handle.
2. This is the same guarantee `ensure` would give, but co-located with the acquisition rather than separated by however many lines of body code come between them. The block form is therefore harder to accidentally break than a hand-written `begin/ensure`.
3. Ruby's standard library and all well-designed gems expose a block form for exactly this reason. When you see `IO.open`, `TCPServer.open`, `PG::Connection.open`, or `Sequel.connect` without a block, reach for the block variant first.
4. Nest block-form acquisitions rather than acquiring multiple resources at the same level. Nesting establishes the correct teardown order (13.7) by construction: the innermost block exits first, then the one wrapping it.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(path: String, sku: Sku).returns(LineItem) }
def read_line_item(path, sku)
  File.open(path, "r") do |f| # guaranteed close; no file handle survives this method
    raw = JSON.parse(f.read)
    LineItem.parse(raw.fetch(sku.to_s))
  end
end
```

**Enforcement:** RuboCop `Style/AutoResourceCleanup`; review for any `.open` or `.new` on a closable type that is not paired with a block or an `ensure` clause.

### 13.2 — When no block form exists, acquire then release in `ensure`; make close idempotent.

**Reasoning, step by step:**
1. Some third-party clients expose only a manual `open`/`close` API. When that is the case, place the acquisition before a `begin` and the release in the corresponding `ensure`. The `ensure` clause runs whether the body returns normally or raises — it is the manual equivalent of a block's guarantee.
2. Order matters: acquire first, then `begin`. Placing the acquisition inside the `begin` means a failure during acquisition skips the `ensure` and leaves nothing to close — which is fine — but if you guard with a nil check in `ensure`, the pattern is explicit and safe either way.
3. Make the close call idempotent: check for nil or a closed flag before calling `close`. A double-close on many clients (sockets, DB adapters) raises; an idempotent wrapper prevents that from turning a teardown into a second exception.
4. Release in `ensure`, not in `rescue`. A `rescue` clause handles an error; the `ensure` clause handles cleanup. Mixing them means cleanup is skipped on a normal return or on a re-raise that bypasses the `rescue`.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(cfg: InventoryConfig).returns(T::Array[Sku]) }
def fetch_skus(cfg)
  client = T.let(nil, T.nilable(InventoryClient))
  client = InventoryClient.new(cfg)
  client.connect

  begin
    client.skus.to_a
  ensure
    client.close if client&.connected? # idempotent; safe on reconnect failure
  end
end
```

**Enforcement:** Review; any method that opens a resource without a block form must have an `ensure` clause on the containing `begin`, and the close call must be nil-guarded or idempotent.

### 13.3 — Size connection pools with a named constant; never one connection per request.

**Reasoning, step by step:**
1. Opening a new database or HTTP connection for every request is unbounded: at peak load, the number of open connections equals the number of concurrent requests. Every connection consumes a file descriptor, memory on both client and server, and an authenticated session on the upstream. The upstream's connection limit is a hard ceiling; exceed it and requests fail.
2. A bounded pool amortizes connection cost across requests, queues excess requestors, and caps the load placed on the upstream. The pool size is a design parameter — derive it from the upstream's `max_connections` and the expected concurrency, then record it as a named constant so it appears in config review.
3. Name the constant at module scope with a `T.let` type annotation. A literal `10` embedded in a `ConnectionPool.new(size: 10)` call is a magic number that reviewers cannot cross-reference to a capacity decision.
4. Pair the pool with a checkout timeout (13.5) so a saturated pool fails fast rather than queuing indefinitely and exhausting caller threads.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

module Database
  extend T::Sig

  MAX_CONNECTIONS    = T.let(10, Integer)
  CHECKOUT_TIMEOUT   = T.let(2.0, Float)

  POOL = T.let(
    ConnectionPool.new(size: MAX_CONNECTIONS, timeout: CHECKOUT_TIMEOUT) { Sequel.connect(ENV.fetch("DATABASE_URL")) },
    ConnectionPool,
  )

  sig { params(blk: T.proc.params(db: Sequel::Database).returns(T.untyped)).returns(T.untyped) }
  def self.with(&blk)
    POOL.with(&blk) # 13.1 — block form; returns connection to pool on any exit
  end
end
```

**Enforcement:** Review and grep; `ConnectionPool.new`, `redis-client`, or any DB adapter must reference a named `MAX_*` constant for the size argument; bare numeric literals in pool constructors are rejected.

### 13.4 — Bound every cache by size or TTL; an unbounded memo is a memory leak.

**Reasoning, step by step:**
1. A memoized method (`@cache ||= compute`) that caches by an argument grows without bound as the argument space grows. In a long-lived process serving many unique keys — order IDs, SKUs, customer IDs — this is a slow, invisible memory leak that only surfaces under sustained load.
2. Replace unbounded per-instance memos with a bounded LRU cache when the key space is large or unbounded. Set a `max_size` derived from the expected working set and available memory, not from the number of items you have seen so far.
3. Pair the size bound with a TTL where the data has a meaningful freshness window. TTL-based expiry is a clock read; event-driven invalidation is a distributed-systems problem. Do both: size prevents memory exhaustion; TTL prevents stale data.
4. A module-level or class-level cache (`@@cache`, or a class-side `Hash`) shared across threads requires a `Mutex` or a thread-safe cache implementation. An unsynchronized shared hash corrupts under concurrent writes (09-concurrency.md).

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

require "lru_redux"

class InventoryCache
  extend T::Sig

  MAX_ENTRIES = T.let(5_000, Integer)
  TTL_SECONDS = T.let(60, Integer)

  sig { void }
  def initialize
    @store = T.let(LruRedux::TTL::Cache.new(MAX_ENTRIES, TTL_SECONDS), LruRedux::TTL::Cache)
  end

  sig { params(sku: Sku).returns(T.nilable(Integer)) }
  def fetch(sku)
    @store[sku.to_s]
  end

  sig { params(sku: Sku, qty: Integer).void }
  def store(sku, qty)
    @store[sku.to_s] = qty
  end
end
```

**Enforcement:** Review; any `Hash` used as a cache must declare a `MAX_*` constant and a TTL where applicable; unbounded `Hash` caches are rejected. See [04-variables-and-declarations.md](./04-variables-and-declarations.md) for memoization caveats.

### 13.5 — Set a deadline on every external I/O call.

**Reasoning, step by step:**
1. A hung TCP connection holds its socket descriptor, its connection-pool slot, and the calling thread's stack until the OS-level timeout fires — which can be tens of minutes. During that time, the pool slot is unavailable to other callers, threads pile up waiting for slots, and the service degrades silently.
2. Set a per-call timeout at the client level, not with `Timeout.timeout`. Ruby's `Timeout.timeout` raises `Timeout::Error` from a background thread, which can interrupt arbitrary C-extension code mid-write and corrupt client state. Use the client's own timeout option — `Net::HTTP#read_timeout`, `redis.timeout`, `pg`'s `:connect_timeout` — or `Async::Task#with_timeout` if you are in an async context (09-concurrency.md).
3. A single global timeout is not sufficient. A 30-second global timeout and a 5-second per-call timeout together mean a batch of ten calls can still block for 30 seconds. Set the timeout at the granularity of a single network call.
4. Expose the timeout as a named constant. A magic `5` in a client constructor is invisible to capacity planning; a `FETCH_TIMEOUT = T.let(5.0, Float)` is greppable and reviewable.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class PricingClient
  extend T::Sig

  CONNECT_TIMEOUT = T.let(2.0, Float)
  READ_TIMEOUT    = T.let(5.0, Float)

  sig { void }
  def initialize
    @http = T.let(
      Net::HTTP.new(ENV.fetch("PRICING_HOST"), 443).tap do |h|
        h.use_ssl       = true
        h.open_timeout  = CONNECT_TIMEOUT
        h.read_timeout  = READ_TIMEOUT
      end,
      Net::HTTP,
    )
  end

  sig { params(sku: Sku).returns(Money) }
  def price_for(sku)
    response = @http.get("/prices/#{sku}")
    raise "unexpected status #{response.code}" unless response.code == "200"

    Money.parse(response.body)
  end
end
```

**Enforcement:** Review; every HTTP, DB, Redis, gRPC, or socket client must set `open_timeout`/`read_timeout` (or equivalent) to a named constant; bare `Timeout.timeout` wrappers are rejected. See [09-concurrency.md](./09-concurrency.md) for async deadline API.

### 13.6 — Never rely on finalizers or the GC for resource cleanup.

**Reasoning, step by step:**
1. The garbage collector reclaims memory on its own schedule; it has no awareness of file descriptors, sockets, or database connections. A handle kept alive inside an object that is no longer referenced will not be closed until GC eventually runs — and under low allocation pressure, GC may not run for a long time.
2. `ObjectSpace.define_finalizer` schedules a proc to run when the object is collected, but finalizers are non-deterministic, run on a separate thread, may coalesce, and at process exit are not guaranteed to run at all. A finalizer is not a substitute for an explicit close.
3. Explicit cleanup through a block form (13.1) or `ensure` clause (13.2) is the only deterministic path. Every resource that must be released has an owner who releases it; the GC is not that owner.
4. `ObjectSpace.define_finalizer` may be useful as a leak detector — logging a warning when an object is collected without having been explicitly closed. Mark any such use with a comment that names its diagnostic purpose so it is not confused with actual cleanup.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class InventorySocket
  extend T::Sig

  sig { params(host: String, port: Integer).void }
  def initialize(host, port)
    @socket = T.let(TCPSocket.new(host, port), TCPSocket)
    @closed = T.let(false, T::Boolean)

    # Leak detector only — not a substitute for explicit close (13.6).
    ObjectSpace.define_finalizer(self, self.class.method(:warn_leak).to_proc)
  end

  sig { void }
  def close
    return if @closed

    @closed = true
    @socket.close
  end

  class << self
    extend T::Sig

    sig { params(_id: Integer).void }
    def warn_leak(_id)
      warn "[InventorySocket] collected without explicit close — resource leak"
    end
  end
end
```

**Enforcement:** Review and grep for `ObjectSpace.define_finalizer`; any occurrence must carry a comment marking it as a diagnostic-only leak detector; its absence from all resource-release paths is verified.

### 13.7 — Release resources in reverse acquisition order; teardown is exception-safe and idempotent.

**Reasoning, step by step:**
1. Later acquisitions depend on earlier ones: a cursor depends on its transaction; a transaction depends on its connection. Releasing in reverse order — cursor first, transaction second, connection last — ensures nothing is released while something layered on it is still alive.
2. Block-form nesting gives reverse order by construction: the innermost block closes first, then the wrapper. `ensure` clauses on nested `begin` blocks do the same. Write acquisition order top-down and rely on the language to reverse it.
3. When writing teardown by hand — for example, releasing multiple resources in a single `ensure` — list the closes bottom-up and annotate the dependency: `cursor.close; txn.rollback; conn.close` with a comment naming why cursor precedes txn. A later edit that reverses the order without reading the comment is a latent bug.
4. Each close call must be safe to call twice. Guard with a `@closed` flag or rely on a method that is documented as idempotent. An exception from a close call in `ensure` replaces the original exception and hides the real failure.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

sig { params(order_id: String).returns(T::Array[LineItem]) }
def load_line_items(order_id)
  conn = Database.checkout
  txn  = conn.transaction(savepoint: true)

  begin
    txn.fetch("SELECT * FROM line_items WHERE order_id = ?", order_id)
       .map { |row| LineItem.parse(row) }
  ensure
    release("transaction") { txn.rollback } # txn first — layered on conn
    release("connection") { conn.close }    # conn last — base resource
  end
end

# Releases a resource, logging a failure instead of masking the in-flight exception.
sig { params(label: String, block: T.proc.void).void }
def release(label, &block)
  block.call
rescue StandardError => error
  warn("#{label} release failed: #{error.message}")
end
```

**Enforcement:** Review; composite teardown is reverse-order; any `ensure` that closes multiple resources is annotated with the dependency order; block nesting is the default for ≥ 2 resources.

### 13.8 — Join or shut down every thread, Ractor, and Fiber on exit.

**Reasoning, step by step:**
1. A thread that is not joined on process exit holds its stack, its locals, and every resource it has opened. On a clean shutdown, Ruby's `at_exit` hooks fire, but a non-joined thread may still be mid-write to a socket when the process terminates, leaving the remote end with a partial message.
2. Track every thread or Ractor you spawn in a collection and join it in a teardown path that is guaranteed to run. A spawned thread with no reference to join is a resource that can never be recovered.
3. Fiber teardown is the caller's responsibility. A Fiber left suspended holds its stack and its local bindings. Call `.resume` until it finishes, or in `async` contexts call `.stop` on the task.
4. Supervisors and worker pools must expose a `shutdown` method that signals workers to drain their queue and join each worker thread before the method returns. Callers invoke `shutdown` in an `ensure` clause or `at_exit` hook (13.9), not by relying on Ruby's GC.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class FulfillmentWorkerPool
  extend T::Sig

  sig { params(concurrency: Integer).void }
  def initialize(concurrency)
    @queue   = T.let(Queue.new, Queue)
    @workers = T.let(
      Array.new(concurrency) { Thread.new { drain } },
      T::Array[Thread],
    )
  end

  sig { params(order: Order).void }
  def submit(order)
    @queue.push(order)
  end

  sig { void }
  def shutdown
    @workers.size.times { @queue.push(:stop) } # sentinel per worker
    @workers.each(&:join)                        # 13.8 — join every thread; no leaked stacks
  end

  private

  sig { void }
  def drain
    loop do
      item = @queue.pop
      break if item == :stop

      Fulfillment.process(item)
    end
  end
end
```

**Enforcement:** Review; any `Thread.new` or `Ractor.new` whose handle is not stored for a later `join` or `wait` is rejected; worker pools must expose a `shutdown` that joins all workers. See [09-concurrency.md](./09-concurrency.md).

### 13.9 — Use `at_exit` only sparingly and idempotently; prefer explicit lifecycle ownership.

**Reasoning, step by step:**
1. `at_exit` hooks execute in reverse registration order, after all threads have terminated. That ordering is fragile: a hook registered by a gem you depend on may run before or after yours, and the hook receives no signal about which exit path fired or whether an exception is in flight.
2. An `at_exit` hook that closes a resource without checking whether it is already closed will raise if another hook or an `ensure` clause closed it first. Make every `at_exit` body idempotent and nil-guarded.
3. Prefer explicit lifecycle ownership. A class that opens a resource in its constructor should expose a `close` method the caller invokes in an `ensure` clause. The caller knows when its work is done; `at_exit` does not.
4. Reserve `at_exit` for process-level resources with no owning object — a PID file, a lock file, a metrics flush — where every other cleanup path would require threading the resource through the entire call stack. Document why explicit ownership was not feasible.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

module MetricsFlusher
  extend T::Sig

  @flushed = T.let(false, T::Boolean)

  # Reserve at_exit for the process-level metrics buffer, which has no single owner.
  at_exit do
    next if @flushed # idempotent (13.9)

    @flushed = true

    begin
      Metrics.flush # best-effort; a flush failure must not mask the real exit reason
    rescue StandardError => error
      warn("metrics flush failed at exit: #{error.message}")
    end
  end
end
```

**Enforcement:** Review and grep for `at_exit`; each occurrence must carry a comment explaining why explicit lifecycle ownership was not used; the body must be idempotent and nil-guarded.

### 13.10 — Wrap a raw resource in a small object that owns its lifecycle and exposes a `with_` block API.

**Reasoning, step by step:**
1. A raw handle — a socket, a file descriptor, a lock — requires callers to know how to open it, close it, and handle errors on both. Every call site duplicates that knowledge. A thin wrapper object centralizes the lifecycle and exposes a `with_` block method so callers use block form (13.1) without knowing the internals.
2. The wrapper's `with_` method is a `yield`-based method that acquires the resource, yields it to the caller's block, and releases it in an `ensure` clause. Internally it is exactly the pattern from 13.1 and 13.2; externally it is a single entry point the caller cannot misuse.
3. Keep the wrapper small: acquire, yield, release. No business logic inside the wrapper itself. Business logic belongs in the block the caller passes.
4. Annotate the `with_` method with a `sig` that reflects the block's parameter type so Sorbet can type-check the block body. The block parameter type is the resource's public interface, not the raw handle.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

class InventoryLock
  extend T::Sig

  LOCK_TIMEOUT = T.let(5.0, Float)

  sig { params(sku: Sku).void }
  def initialize(sku)
    @sku    = T.let(sku, Sku)
    @locked = T.let(false, T::Boolean)
  end

  sig do
    type_parameters(:R)
      .params(blk: T.proc.returns(T.type_parameter(:R)))
      .returns(T.type_parameter(:R))
  end
  def with_lock(&blk)
    acquire!

    begin
      yield
    ensure
      release! # 13.2 — ensure path; 13.7 — release after block
    end
  end

  private

  sig { void }
  def acquire!
    acquired = Redis.current.set("lock:#{@sku}", "1", nx: true, ex: LOCK_TIMEOUT.to_i)
    raise "could not acquire lock for #{@sku} within #{LOCK_TIMEOUT}s" unless acquired

    @locked = true
  end

  sig { void }
  def release!
    return unless @locked # idempotent (13.2)

    @locked = false
    Redis.current.del("lock:#{@sku}")
  end
end

# Call site: block form enforced by the wrapper.
InventoryLock.new(sku).with_lock do
  Inventory.decrement(sku, quantity)
end
```

**Enforcement:** Review; raw resource handles must not be passed across method boundaries without a wrapping object that owns the lifecycle; the wrapper must expose a `with_` block method and must not expose the raw handle as a public attribute.

## Cross-references

- Formatting, 100-col limit, trailing commas, `T.let` for typed constants: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- Named constants for magic numbers, `SCREAMING_SNAKE_CASE`: [02-naming-conventions.md](./02-naming-conventions.md).
- `T.nilable`, `sig` on every method, nil-guard patterns: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- Memoization caveats, unbounded `||=` cache hazards: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Method size cap, guard clauses, 2+ assertions per method: [05-methods.md](./05-methods.md).
- `Data.define` value objects safe to share across threads and Fibers: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Thread model, Ractor, Fiber, `async` deadlines, bounded queues: [09-concurrency.md](./09-concurrency.md).
- Error subclasses, `rescue => error`, `cause` chaining on teardown failures: [08-error-handling.md](./08-error-handling.md).
- Minitest cleanup in `teardown`, fake time, asserting close was called: [11-testing.md](./11-testing.md).
- Lazy enumerators, allocation hygiene, pool sizing to slowest resource: [15-performance.md](./15-performance.md).
