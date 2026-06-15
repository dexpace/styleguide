# 13 — Resource Management

Resources (file handles, sockets, locks, connections, transactions, tasks) outlive the call that opens them unless something explicitly closes them. The explicit thing must be in your code.

## What good looks like

```python
import asyncio
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

import asyncpg
import httpx

POOL_MAX = 50  # downstream capacity, not a guess
HTTP_TIMEOUT = 5.0  # seconds, user-perceived SLA


@asynccontextmanager
async def fetcher() -> AsyncIterator[tuple[asyncpg.Pool, httpx.AsyncClient]]:
    pool = await asyncpg.create_pool(min_size=10, max_size=POOL_MAX)
    limits = httpx.Limits(max_connections=POOL_MAX, max_keepalive_connections=10)
    try:
        async with httpx.AsyncClient(limits=limits) as client:
            yield pool, client
    finally:
        await pool.close()  # deterministic, even on error


async def sync_orders(urls: list[str]) -> list[bytes]:
    async with fetcher() as (pool, client):
        async with asyncio.timeout(HTTP_TIMEOUT):
            async with asyncio.TaskGroup() as tg:
                tasks = [tg.create_task(client.get(url)) for url in urls]
        async with pool.acquire() as conn:
            await conn.executemany("INSERT INTO seen(url) VALUES($1)", [(u,) for u in urls])
    return [t.result().content for t in tasks]
```

Every paired resource enters through `async with` (13.1, 13.3); `fetcher` is an `@asynccontextmanager` whose `try/finally` closes the pool deterministically (13.2); the pool and HTTP client are bounded by `POOL_MAX` and `asyncio.timeout` guards the I/O (13.5, 13.7, 13.10); `TaskGroup` owns the fan-out task lifecycles (13.4). No `__del__`, no manual `close` in the happy path (13.9).

## Rules

### 13.1 — `with` on every paired resource. Always.

**Reasoning, step by step:**
1. `with open(path) as f: ...` closes the file on exit — normal *or* exceptional. Safer than manual `try/finally`.
2. Manual `try: f = open(path); ... finally: f.close()` is the same thing, verbose, and forgettable.
3. Use for: files, locks, transactions, subprocess handles, temporary state, sockets.
4. **Anti-pattern:** opening a resource and returning the open object from a function. Caller doesn't know to `with` it. Take a callback or be a context manager yourself.

**Enforcement:** `flake8-bugbear` B017/`pylint` consider-using-with; review for functions returning open handles.

### 13.2 — `contextlib.contextmanager` for ad-hoc context managers.

**Reasoning, step by step:**
1. `@contextmanager` turns a generator into a context manager — clearer than a class with `__enter__`/`__exit__` for simple cases.
2. **Pattern:**
   ```python
   from contextlib import contextmanager
   from collections.abc import Iterator
   from pathlib import Path
   import tempfile, shutil

   @contextmanager
   def temporary_workdir() -> Iterator[Path]:
       d = Path(tempfile.mkdtemp())
       try:
           yield d
       finally:
           shutil.rmtree(d)
   ```
3. Always `try/finally` around the `yield`. The `finally` runs even when the body raises.
4. For async resources: `@asynccontextmanager` + `async def` + `async with`.

**Enforcement:** review; every `@contextmanager` generator wraps its `yield` in `try/finally`.

### 13.3 — Async resources use `async with`.

**Reasoning, step by step:**
1. `async with httpx.AsyncClient() as client: ...` is the async equivalent of `with`. The `__aenter__` and `__aexit__` are coroutines.
2. Use for: async HTTP clients, async database pools, async file handles, async locks (`asyncio.Lock`).
3. Don't mix `with` and `async with` — if the resource has both, use the async form inside async code, the sync form inside sync code.

**Enforcement:** `ruff` ASYNC flags blocking calls in async code; review for sync `with` on async-capable resources.

### 13.4 — `asyncio.TaskGroup` owns task lifecycles.

**Reasoning, step by step:**
1. Restated from chapter 09: `TaskGroup` waits for all tasks on exit, cancels siblings on error.
2. Use as the *only* way to launch tasks you don't intend to outlive the current scope.
3. For tasks that *should* outlive: keep a strong reference, attach a `done_callback` for cleanup, document the lifecycle.

**Enforcement:** `ruff` RUF006 catches dangling `create_task`; review that ad-hoc tasks launch through a `TaskGroup`.

### 13.5 — `asyncio.timeout` is the default bound for I/O. No exceptions.

**Reasoning, step by step:**
1. Every async I/O without a timeout is a resource leak waiting to happen.
2. `async with asyncio.timeout(5.0): await http.get(url)` — the timeout is a hard contract.
3. Choose by user-perceived SLA. Don't pick "1 hour" because "it should be enough."

**Enforcement:** review; every external `await` sits inside an `asyncio.timeout` or carries a client-level deadline.

### 13.6 — `secrets` / `os.urandom` for cryptographic randomness; `random` for everything else.

**Reasoning, step by step:**
1. `random.random()` is statistical-quality, not cryptographic. Don't use for tokens, nonces, secrets.
2. `secrets.token_bytes(32)`, `secrets.token_urlsafe(32)`, `secrets.compare_digest(a, b)` for security-sensitive operations.
3. `os.urandom(n)` is the same source. `secrets` is the wrapper.
4. **Constant-time comparison** for tokens/MACs: `secrets.compare_digest(expected, actual)` not `==`. (Restated from [security.md](../security.md).)

**Enforcement:** `bandit` B311 flags `random` in security contexts; review token comparisons for `compare_digest`.

### 13.7 — Connection pools: bounded, explicit, with timeouts.

**Reasoning, step by step:**
1. Pool size is a system parameter — picked from downstream capacity, expected concurrency, and RAM.
2. **httpx:** `httpx.Limits(max_connections=N, max_keepalive_connections=M)`. Document the values.
3. **asyncpg:** `asyncpg.create_pool(min_size=10, max_size=50)`. Same.
4. Monitor: pool exhaustion (waiters), checkout latency. Alert on saturated pool.

**Enforcement:** review; every pool constructor passes explicit `max_size`/`max_connections` from a named constant.

### 13.8 — Graceful shutdown: drain in-flight, refuse new, close pools.

**Reasoning, step by step:**
1. SIGTERM means "you have N seconds; finish what's in flight, then stop."
2. Shutdown sequence: stop accepting new work → wait for in-flight to drain (bounded) → close pools → exit.
3. A hung shutdown is worse than a forced one — the orchestrator will SIGKILL eventually.
4. **Pattern in asyncio:**
   ```python
   stop = asyncio.Event()

   def _handle_signal() -> None:
       stop.set()

   loop = asyncio.get_running_loop()
   for sig in (signal.SIGTERM, signal.SIGINT):
       loop.add_signal_handler(sig, _handle_signal)

   try:
       async with asyncio.TaskGroup() as tg:
           tg.create_task(serve())
           tg.create_task(stop.wait())
   finally:
       await asyncio.wait_for(close_resources(), timeout=30.0)
   ```

**Enforcement:** integration test that a SIGTERM drains in-flight work and closes pools within the bounded window.

### 13.9 — `del` is not a resource-management tool. Use context managers.

**Reasoning, step by step:**
1. `del x` unbinds the name. Whether the object is freed depends on reference counts and the GC.
2. `__del__` (the destructor dunder) is unreliable: order of finalization is undefined, exceptions in `__del__` are silently logged, cyclic references prevent finalization entirely.
3. **Never** rely on `__del__` for resource cleanup. Implement `__enter__`/`__exit__` (or `__aenter__`/`__aexit__`) instead.
4. `weakref.finalize` is acceptable for non-critical cleanup that *might* fire. Not for guaranteed cleanup.

**Enforcement:** review; no `__del__` used for resource cleanup, no `del` standing in for a context manager.

### 13.10 — Bounded everything.

**Reasoning, step by step:**
1. Restated from root rule §9: every loop, queue, retry, timeout, task list, cache must have a fixed upper bound.
2. Common bounds in Python:
   - `itertools.islice(iter, n)` for capping an iterator.
   - `asyncio.Queue(maxsize=N)` and `asyncio.Semaphore(N)` for capping concurrency.
   - `functools.lru_cache(maxsize=N)` for memoization (never `maxsize=None` in production unless inputs are provably finite).
   - `asyncio.timeout(seconds)` for every external call.
   - Pool `max_size` for every connection pool.
3. State the bound at the call site. Magic constants drift; named constants document.

**Enforcement:** review; `lru_cache(maxsize=None)`, unbounded queues, and uncapped iterators flagged in code review.

### 13.11 — `temporary*` for ephemeral files and directories.

**Reasoning, step by step:**
1. `tempfile.NamedTemporaryFile()`, `tempfile.TemporaryDirectory()` — context-managed, OS-level cleanup.
2. `delete=True` (default) for `NamedTemporaryFile` — file deleted on close.
3. On Windows: `NamedTemporaryFile(delete=False)` is sometimes needed because of file-locking rules. Then `os.unlink(path)` in a `finally`.

**Enforcement:** review; ephemeral files go through `tempfile`, and `delete=False` is paired with a `finally` unlink.

### 13.12 — Subprocess: `subprocess.run` with timeout, `capture_output=True`.

**Reasoning, step by step:**
1. `subprocess.run(args, timeout=30, check=True, capture_output=True, text=True)` is the safe default.
2. `shell=False` (the default). Never `shell=True` with user-provided input — command injection (see [security.md](../security.md)).
3. Pass arguments as a list: `["git", "status"]`, not `"git status"`.
4. **Async equivalent:** `asyncio.create_subprocess_exec` with explicit timeout via `asyncio.timeout`.

**Enforcement:** `bandit` B602/B603 flags `shell=True` and untrusted args; review for `timeout=` on every `subprocess.run`.

## Cross-references

- Async patterns: chapter 09.
- Security and credentials: [security.md](../security.md).
- Bounded caches: chapter 15.
