# 09 — Concurrency & Async

Python concurrency has three shapes: `asyncio` (I/O-bound, cooperative), `threading` (I/O-bound, preemptive but GIL-bound), `multiprocessing` (CPU-bound, separate processes). Pick deliberately.

## Rules

### 9.1 — Default to `asyncio` for new I/O-bound code.

**Reasoning, step by step:**
1. asyncio gives structured concurrency (3.11+), cooperative cancellation, and integrates with the broader async ecosystem (httpx, asyncpg, motor, aiokafka).
2. Threading is older, less expressive about lifecycles, and the GIL means it only helps for I/O-bound work anyway.
3. Multiprocessing is for CPU-bound work and brings its own complexity (pickling, IPC, lifecycle).
4. **Decision:** I/O-bound new code → asyncio. CPU-bound → multiprocessing or a native extension. Mixed → asyncio for the I/O, `asyncio.to_thread` for the blocking pieces.

### 9.2 — `asyncio.TaskGroup` over bare `asyncio.gather` (Python 3.11+).

**Reasoning, step by step:**
1. `asyncio.TaskGroup` provides structured concurrency: when the block exits, every task is awaited or cancelled. Errors from any task cancel the others; all errors are aggregated into an `ExceptionGroup`.
2. `asyncio.gather` has subtler semantics: by default it propagates the first error and *doesn't* cancel siblings. `return_exceptions=True` hides errors as values. Both are footguns.
3. **Pattern:**
   ```python
   async with asyncio.TaskGroup() as tg:
       t1 = tg.create_task(load_user(uid))
       t2 = tg.create_task(load_order(oid))
   # both done here; errors raised as ExceptionGroup
   user, order = t1.result(), t2.result()
   ```
4. Pre-3.11: use `asyncio.gather` with explicit error handling, or migrate.

### 9.3 — `asyncio.timeout` over `asyncio.wait_for` (Python 3.11+).

**Reasoning, step by step:**
1. `asyncio.timeout` is a context manager: `async with asyncio.timeout(5.0): await something()`. Cancellation propagates correctly; works with TaskGroup.
2. `asyncio.wait_for(coro, timeout=5)` works but has rougher edges around cancellation propagation.
3. **Rule:** every external async I/O has a timeout. Wrap external calls with `asyncio.timeout`. Pick a number; document the choice.
4. The timeout should match the user-perceived SLA, not "infinity minus a bit."

### 9.4 — Never `asyncio.create_task` and drop the reference.

**Reasoning, step by step:**
1. `asyncio.create_task(coro)` schedules a coroutine and returns a `Task`. Python only weakly references the task — if you drop the reference, the task can be garbage-collected mid-execution.
2. Symptoms include: silent task disappearance, mysterious cancellation, lost results.
3. **Fix:** hold the task reference. Use `TaskGroup` (which holds tasks for you), or stash tasks in a set and remove on completion:
   ```python
   _background: set[asyncio.Task] = set()

   def fire_and_forget(coro: Coroutine) -> None:
       task = asyncio.create_task(coro)
       _background.add(task)
       task.add_done_callback(_background.discard)
   ```
4. **Better:** don't fire-and-forget. Own the lifecycle with a TaskGroup or service-level scope.

### 9.5 — Cancellation is cooperative. Honor `CancelledError`.

**Reasoning, step by step:**
1. Cancelling a task raises `CancelledError` at the next `await`.
2. `try/except` in async code must re-raise `CancelledError` if it catches it:
   ```python
   try:
       await something()
   except asyncio.CancelledError:
       cleanup()
       raise
   except Exception:
       handle()
   ```
3. Long CPU-only sections without `await` don't notice cancellation. Insert `await asyncio.sleep(0)` periodically, or split the work.
4. `except Exception:` does *not* catch `CancelledError` in Python 3.8+ (it became `BaseException`-derived). Be explicit if you need to catch it.

### 9.6 — `asyncio.Lock`, `asyncio.Semaphore`, `asyncio.Queue` for coroutine-safe sync.

**Reasoning, step by step:**
1. `threading.Lock` blocks the underlying thread — inside an async function, that's a starvation hazard.
2. `asyncio.Lock` is suspension-aware. `async with lock: ...` is the idiomatic shape.
3. `asyncio.Semaphore` for bounding concurrent operations: `async with semaphore: await heavy()` limits in-flight to the semaphore's value.
4. `asyncio.Queue` for producer-consumer between coroutines. **Bound the queue:** `asyncio.Queue(maxsize=N)`.

### 9.7 — `asyncio.to_thread` for unavoidable blocking calls.

**Reasoning, step by step:**
1. Some libraries are sync-only (legacy DB drivers, `requests`, image processing). Don't let them block the event loop.
2. `await asyncio.to_thread(sync_fn, *args)` runs the call in a worker thread and awaits the result. The event loop stays responsive.
3. Bound the worker pool. The default `ThreadPoolExecutor` is unbounded by default in `to_thread`; set a custom executor with a real cap for high-load systems.
4. Each `to_thread` call has a thread-context-switch cost. For tight loops over a sync library, batch the work first.

### 9.8 — Threading: only when forced. Multiprocessing: only when CPU-bound.

**Reasoning, step by step:**
1. **Threading** in Python is constrained by the GIL — only one thread runs Python bytecode at a time. Useful for I/O parallelism in sync code, useless for CPU parallelism.
2. **Multiprocessing** spawns separate Python processes — each with its own GIL. Useful for CPU-bound work, but inter-process communication is expensive (pickling).
3. **Decision tree:**
   - I/O-bound, new code → asyncio.
   - I/O-bound, can't go async (legacy library, ecosystem) → threading with a bounded pool.
   - CPU-bound → multiprocessing, or a C extension, or `concurrent.futures.ProcessPoolExecutor`.
   - "I want it faster" without measurement → profile first.
4. Python 3.13's PEP 703 (per-interpreter GIL) and PEP 684 are experimental — not yet a default option.

### 9.9 — `concurrent.futures` for high-level sync parallelism.

**Reasoning, step by step:**
1. `ThreadPoolExecutor` and `ProcessPoolExecutor` give a uniform `submit` / `map` / `as_completed` API.
2. **Always bound the pool size.** Default is `os.cpu_count()` for processes; for threads, it's `min(32, cpu_count + 4)` — sometimes wrong for your workload.
3. `with ThreadPoolExecutor(max_workers=8) as pool:` — context-managed shutdown.
4. For async code, prefer asyncio primitives. Use `concurrent.futures` only in sync code or to bridge.

### 9.10 — Producer-consumer with backpressure: bounded `Queue`, suspend on full.

**Reasoning, step by step:**
1. An unbounded queue is a memory leak with a delay.
2. Bounded `asyncio.Queue(maxsize=N)`: `put()` suspends when full. Producers naturally backpressure.
3. For multi-producer/multi-consumer: use queues + tasks managed by a TaskGroup. The producers and consumers are tasks; the queue mediates.
4. Choose `maxsize` from the slowest consumer's catch-up time. Document the value.

### 9.11 — `contextvars` for async-safe context (replaces `threading.local` for asyncio).

**Reasoning, step by step:**
1. `threading.local` is per-thread. In asyncio, all coroutines share a thread — `threading.local` doesn't isolate them.
2. `contextvars.ContextVar` is per-context. Each task has its own context; copies inherit the parent's values at task creation.
3. Use for: request IDs, user identity, tenant context, anything that should "follow" a request through async calls.
4. Pattern:
   ```python
   request_id: ContextVar[str] = ContextVar("request_id")
   request_id.set("abc-123")  # inside the request handler
   # any coroutine descended from here can read request_id.get()
   ```
5. For logging integration with `contextvars`, see the logging chapter.

### 9.12 — `asyncio.run` at the program entry point only.

**Reasoning, step by step:**
1. `asyncio.run(main())` is the top-level entry into async code. It creates an event loop, runs the coroutine, closes the loop.
2. Calling `asyncio.run` inside a function called from async code is wrong — there's already a loop.
3. Tests: use `pytest-asyncio` or `anyio` plugins. They handle the loop lifecycle.
4. Libraries: never call `asyncio.run`. Take the coroutine, let the caller run it.

### 9.13 — Separate sync and async clients. Never mix `async def` and `def` in the same class.

**Reasoning, step by step:**
1. A class with both `def get(self)` and `async def get_async(self)` is two classes pretending to be one. Callers can't tell what they're getting; subclassing breaks; mypy can't help with the shape.
2. Provide two classes — same name, different module path. Sync `BookingClient` in `acme.booking`; async `BookingClient` in `acme.booking.aio`. Callers explicitly import the variant they want:
   ```python
   from acme.booking import BookingClient            # sync
   from acme.booking.aio import BookingClient        # async
   ```
3. The async client lives in a sibling `.aio` submodule (the Azure SDK convention — broadly sound). Sub-submodules of `.aio` mirror the sync side: `acme.booking.aio.payments` mirrors `acme.booking.payments`.
4. **Don't name the class `BookingClientAsync`.** That's the same class-name suffix anti-pattern the Azure SDK explicitly rejects. The *module path* carries the sync/async distinction; the class name stays clean.
5. The two classes share documentation conventions, method names, and parameter names. The only difference is the body and the `async`/`await` keywords.

### 9.14 — Async clients use `async`/`await`. Not `yield from` coroutines, not `asyncio.coroutine`.

**Reasoning, step by step:**
1. `@asyncio.coroutine` and `yield from`-based coroutines were removed in Python 3.11. Don't write new code with them; migrate legacy code when touched.
2. `async def` + `await` is the only blessed shape.
3. **From Azure SDK guidelines:** "DO use the `async`/`await` keywords. Do not use the yield from coroutine or asyncio.coroutine syntax."

## Cross-references

- Resource lifecycle and cancellation: chapter 13.
- Logging in async code (contextvars + structlog): chapter on logging.
- Performance trade-offs of asyncio vs threading: chapter 15.
- Client constructor signature (same shape for sync and async): chapter 10.
