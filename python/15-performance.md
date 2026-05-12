# 15 — Performance

Design-time is the best time for 1000× improvements. Code-time is the best time for 10× ones. Profile-time is the only honest time for everything below 10×.

## Rules

### 15.1 — Design for the slowest resource first: network > disk > memory > CPU.

**Reasoning, step by step:**
1. A request that does 10 sequential network calls is hopeless regardless of how tight the Python is. Batch, parallelize, or cache the network *first*.
2. Allocation pressure rarely wins from instruction-level tuning; reduce allocations first.
3. Order of priority: eliminate the round-trip; batch the round-trips; cache the response; reduce allocations; tune the inner loop.
4. CPU optimizations are last because (a) they're the smallest absolute wins on most server workloads, (b) the Python interpreter is the speed ceiling, (c) they're easiest to do wrong.

### 15.2 — Profile before you optimize. `py-spy`, `cProfile`, `memray`, `scalene`.

**Reasoning, step by step:**
1. Intuition about Python performance is wrong roughly half the time. The GIL, allocations, attribute lookup, and built-in function dispatch all interact in non-local ways.
2. **`py-spy`** — sampling profiler, no instrumentation, attaches to running processes. Best default for "where is time going?"
3. **`cProfile`** — stdlib, instruments the program. Heavier overhead but exact call counts.
4. **`memray`** — allocation profiling. Use when you suspect memory pressure.
5. **`scalene`** — CPU + memory in one tool, distinguishes Python vs native time.
6. **Rule:** before "fixing" a perf issue, capture a profile that *shows the issue*. After the fix, capture another. The delta is the proof.

### 15.3 — `__slots__` for hot-path classes. Skip the per-instance `__dict__`.

**Reasoning, step by step:**
1. Python classes default to a per-instance `__dict__`, which costs memory and slows attribute access.
2. `class X: __slots__ = ("a", "b")` declares fixed attributes. No `__dict__`. Faster access, lower memory.
3. `@dataclass(slots=True)` (Python 3.10+) generates it for you.
4. Trade-off: no dynamic attribute assignment. For value types this is a feature.
5. Multi-inheritance + slots is finicky — multi-inherit two slot-classes with overlapping slots and you'll get errors. Most value types don't multi-inherit.

### 15.4 — Generators over lists for large or streaming data.

**Reasoning, step by step:**
1. `[x.value for x in items]` materializes the full list — memory grows with `len(items)`.
2. `(x.value for x in items)` is lazy — one element in memory at a time, but iterable only once.
3. For chained transforms, lazy beats eager when the intermediate sizes are large. For small inputs (~100 elements), eager is faster — comprehension overhead is dominated by lazy-iterator overhead.
4. `itertools` operations are lazy by default: `chain`, `groupby`, `islice`, `accumulate`, `pairwise`.

### 15.5 — `functools.lru_cache` with bounded `maxsize`.

**Reasoning, step by step:**
1. `@lru_cache(maxsize=128)` memoizes a pure function. Cheap win for repeated calls with overlapping inputs.
2. **Always bound `maxsize`.** `maxsize=None` is unbounded — a memory leak.
3. Cache key is the arguments. Mutable arguments don't hash; the cache decorator errors.
4. Memoize *pure* functions only. Side-effecting cached calls produce stale results and confusion.
5. Inspect with `func.cache_info()` (hits, misses, current size).

### 15.6 — Avoid global state. Module-level mutable singletons are a code smell.

**Reasoning, step by step:**
1. Module-level `_cache: dict = {}` is shared by every caller, every test, every thread. Mutation is invisible.
2. If state is needed: encapsulate in a class, inject the class as a dependency. Tests can substitute.
3. Acceptable module-level mutables: compiled regexes (immutable after compile), connection pools (configured at import, lifecycle managed elsewhere), `functools.lru_cache` (designed for this).
4. **Anti-pattern:** module-level counters incremented from inside functions. That's per-process global state with no concurrency story.

### 15.7 — String operations: `"".join(parts)`, f-strings, no `+` in loops.

**Reasoning, step by step:**
1. `s += part` in a loop is O(n²) — each append allocates a new string. Use `"".join(parts)`.
2. f-strings are the fastest Python formatter. Use them.
3. `str.format` and `%`-style are slower; `%`-style is used for logging because of laziness, not speed.
4. For binary data, `bytes`/`bytearray` and `b"".join` for the same reasons.

### 15.8 — `dict`/`set` for O(1) lookup. `list.in` for unbounded scans.

**Reasoning, step by step:**
1. `if x in some_list:` is O(n). For repeated lookups, build a `set` once and reuse.
2. `dict` lookup is O(1) average; constants matter for tight loops.
3. `frozenset` for immutable membership tests. Slightly faster than `set` for read-only.
4. **Anti-pattern:** scanning a list for membership inside a loop over another list. That's O(n²). Build a set first.

### 15.9 — `asyncio` overhead: cheap, not free. Don't over-spawn.

**Reasoning, step by step:**
1. A coroutine launch is microseconds; an `await` is similar. Millions per second on a single thread.
2. Launching one task per element of a 10M-element list, each doing a microsecond of work, is wasteful — the scheduling cost dominates.
3. **Pattern:** batch work into chunks, launch per chunk:
   ```python
   async def process_all(items: list[Item]) -> None:
       async with asyncio.TaskGroup() as tg:
           for chunk in batched(items, 1024):
               tg.create_task(process_chunk(chunk))
   ```
4. `itertools.batched` (3.12+) for the chunking.

### 15.10 — GIL realities: threading helps I/O, not CPU.

**Reasoning, step by step:**
1. The GIL serializes Python bytecode execution. Only one thread runs Python at a time.
2. I/O releases the GIL during the wait (most stdlib I/O calls do). Threading wins for I/O parallelism.
3. CPU-bound work in threads is *slower* than serial — context switches without parallelism.
4. For CPU parallelism: `multiprocessing`, or call into a native extension that releases the GIL (NumPy, Polars, much of scipy).
5. Python 3.13's experimental free-threaded build (PEP 703) removes the GIL but isn't yet default. Don't depend on it.

### 15.11 — Native extensions for hot paths: numpy, polars, pyarrow, Cython.

**Reasoning, step by step:**
1. Numeric and columnar workloads in pure Python are slow. `numpy`/`pandas`/`polars`/`pyarrow` are 10-1000× faster for vectorized operations.
2. For custom hot paths: Cython, `mypyc`, Rust extensions (`pyo3`), C extensions. The cost is build complexity; the win is order-of-magnitude.
3. Don't optimize prematurely. Measure first; reach for native extensions when profiles say so.

### 15.12 — Cold-start matters for some workloads. Lazy imports help.

**Reasoning, step by step:**
1. Import time adds to cold-start. A large project can spend hundreds of milliseconds in imports before any code runs.
2. Lazy imports (`import X` inside the function that uses it) defer that cost — at the price of slower first-call.
3. `importlib.util.LazyLoader` for true lazy module loading. Useful for plugin systems and CLI tools.
4. For long-running services, cold-start doesn't matter much. For CLI tools and serverless, it does.

### 15.13 — `dataclass` performance: `slots=True`, `frozen=True`, careful with `__hash__`.

**Reasoning, step by step:**
1. `@dataclass(slots=True, frozen=True)` is the fast and safe default for value types.
2. Without `slots=True`, you pay for a per-instance `__dict__`. Significant memory across millions of instances.
3. `__hash__` is generated when `frozen=True`. Hashing iterates fields each call — for very wide dataclasses on a hot path, cache the hash if you can.
4. Don't reach for `attrs`/`pydantic` for hot-path value types. They have richer features but more overhead. Stdlib `dataclass` is the right tool for the inner loop.

## Cross-references

- Generators in API design: chapter 10.
- asyncio + structured concurrency: chapter 09.
- Memory profiling tools: also see [performance.md](../performance.md) cross-cutting guide.
