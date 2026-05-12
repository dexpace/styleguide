# 07 — Pythonic Idioms

Python's idioms exist because they make code shorter, clearer, or safer. Used deliberately, they remove ceremony. Used carelessly, they produce code that looks Pythonic and reads like a puzzle.

## Rules

### 7.1 — Context managers (`with`) for every paired-resource lifecycle.

**Reasoning, step by step:**
1. `with open(path) as f: data = f.read()` closes the file on exit — normal *or* exceptional. Always safer than manual `try/finally`.
2. The protocol: `__enter__` returns the value; `__exit__(exc_type, exc, tb)` cleans up. Returning `True` from `__exit__` suppresses the exception (rarely correct; usually a footgun).
3. Use for: files, locks, transactions, temporary state changes, sockets, subprocess handles.
4. **`contextlib.contextmanager`** for ad-hoc context managers:
   ```python
   @contextmanager
   def temporary_dir() -> Iterator[Path]:
       d = Path(tempfile.mkdtemp())
       try:
           yield d
       finally:
           shutil.rmtree(d)
       ```
5. **`async with`** for async resources. Coroutine-safe equivalent (chapter 09).

### 7.2 — Generators and `yield` for streaming.

**Reasoning, step by step:**
1. A generator function (using `yield`) returns a lazy iterator. Memory is bounded by the consumer's pull rate, not the producer's volume.
2. Use for: reading large files, producing potentially unbounded sequences, transforming an iterable lazily.
3. `yield from inner_iterator()` for delegation. Cleaner than a `for x in inner: yield x` loop.
4. **Trap:** generators are single-pass. Iterating twice doesn't restart — you get an empty iterator the second time.
5. **Generator vs comprehension:**
   - `(x * 2 for x in items)` — generator expression. Lazy. Pass to functions expecting iterables.
   - `[x * 2 for x in items]` — list comprehension. Materialized. Use when you need a list.

### 7.3 — Comprehensions for transforms; loops for side effects.

**Reasoning, step by step:**
1. `[user.email for user in users if user.active]` is the natural Pythonic shape for "transform + filter."
2. The comprehension stops being clearer when (a) it spans more than 2 lines, (b) it has multiple `if`/`for` clauses that obscure intent, (c) the expression is a complex method chain.
3. **Side effects in comprehensions are a code smell.** `[print(x) for x in items]` is wrong — that's a `for` loop.
4. Dict/set comprehensions: `{k: v for k, v in pairs}`, `{x.id for x in items}`. Same rules.

### 7.4 — EAFP (Easier to Ask Forgiveness than Permission) — but only when natural.

**Reasoning, step by step:**
1. Python's design favors EAFP: try the operation, catch the exception if it fails.
2. `try: x = d["key"]; except KeyError: x = default` *or* `x = d.get("key", default)`. The latter is shorter; both are Pythonic.
3. EAFP wins when (a) the success case is dominant, (b) the check would be a TOCTOU race (file existence between check and open).
4. LBYL (Look Before You Leap) wins when (a) the precondition is cheap and definitive (`if x is None`), (b) the exception path has cost (allocations, logging).
5. **Don't catch broadly to enable EAFP.** Catch the specific exception you mean (`KeyError`, `ValueError`), not `Exception`.

### 7.5 — `pathlib.Path` over `os.path`.

**Reasoning, step by step:**
1. `pathlib` gives a typed, object-oriented path API. `Path("/a") / "b" / "c.txt"` reads naturally.
2. `os.path` is string-based and platform-fragile. Don't reach for it in new code.
3. Convert at the boundary: APIs that hand you a `str` path become a `Path` immediately. APIs that want a `str` get `str(path)`.
4. Common idioms: `path.read_text()`, `path.write_bytes(data)`, `path.iterdir()`, `path.glob("*.json")`.

### 7.6 — f-strings for formatting. `%`-style only for logging.

**Reasoning, step by step:**
1. `f"Hello, {name}! You have {len(cart)} items."` is shorter, scoped at the use site, and the fastest formatter Python has.
2. `%`-style and `str.format` are older and less ergonomic. Use them only for compatibility (rare) or **logging** (intentional — see chapter on logging).
3. Logging: `logger.info("user %s loaded", user_id)` — the formatting is lazy, only happens if the level is enabled. f-strings format eagerly, even at DEBUG.
4. **Anti-pattern:** complex expressions inside f-strings. If `f"{complex.expression.with[many].parts}"` is hard to read, lift it to a `val` above.

### 7.7 — `match`/`case` for exhaustive matching (Python 3.10+).

**Reasoning, step by step:**
1. Python 3.10's structural pattern matching is the right tool for: discriminated unions, deeply-nested shape checks, multi-way branching by type or value.
2. `match`/`case` is *structural* — `case Approved(receipt_id=r):` binds `r` to the field.
3. Pair with `assert_never` for exhaustiveness:
   ```python
   match result:
       case Approved(): ...
       case Declined(): ...
       case _: assert_never(result)   # mypy errors if a variant is added and not handled
   ```
4. **Anti-pattern:** `match` for what should be a simple `if/elif`. Two cases — write the `if`. Four cases with structural binding — write the `match`.

### 7.8 — Dunders: implement the protocol your class is *part of*.

**Reasoning, step by step:**
1. Dunders make a class participate in Python's protocols: iteration, containment, length, equality, ordering, formatting.
2. Implement when your class *is* that thing. Don't implement them for cleverness.
3. The big six to know: `__eq__` + `__hash__`, `__iter__`, `__len__`, `__contains__`, `__enter__` + `__exit__`, `__repr__`.
4. Less common but useful: `__getitem__` (indexable), `__call__` (callable), `__bool__` (truthiness — careful with this one), `__getattr__` (fallback attribute access).

### 7.9 — Decorators for cross-cutting concerns; not for cleverness.

**Reasoning, step by step:**
1. Decorators wrap a function in another function. Good uses: caching (`@lru_cache`), timing, logging, retry, transaction boundaries.
2. Stack decorators top-to-bottom in the order they wrap. `@log @retry @cache def f(): ...` means `log(retry(cache(f)))`.
3. Custom decorators always `@functools.wraps(func)` to preserve name, docstring, signature.
4. Type-preserve. Use `ParamSpec` (`Callable[P, R]`) to preserve the wrapped signature for mypy:
   ```python
   def retry[**P, R](fn: Callable[P, R]) -> Callable[P, R]: ...
   ```
5. **Anti-pattern:** decorator with side effects on import. Tests touching the module run the decorator's setup. Pure wrapping only.

### 7.10 — Iteration: `enumerate`, `zip`, `itertools`. Not manual indexing.

**Reasoning, step by step:**
1. `for i, item in enumerate(items):` over `for i in range(len(items)): item = items[i]`.
2. `for a, b in zip(xs, ys, strict=True):` over manual paired indexing. `strict=True` (3.10+) errors on length mismatch.
3. `itertools` is full of right answers: `chain`, `groupby`, `islice` (bounded slicing of any iterable), `pairwise` (3.10+), `accumulate`, `product`.
4. **Bound your iterators.** `itertools.islice(generator, max_items)` caps potentially-unbounded sources (Tiger Style rule §9 — see [root README](../README.md)).

### 7.11 — Truthiness: explicit on `None`, container truthiness for empty checks.

**Reasoning, step by step:**
1. `if x is None:` and `if x is not None:` — explicit. Always for "is this missing?"
2. `if items:` is OK for "are there any items?" (true for non-empty list, dict, set, str).
3. **Bug:** `if value:` when `value` could be `0` or `""` and you wanted "missing." Use `if value is not None:`.
4. Bool conversion of complex objects (`if user:`) is a smell unless you've defined `__bool__`. Be explicit.

### 7.12 — `collections.abc` for type hints; not the deprecated `typing` aliases.

**Reasoning, step by step:**
1. Take `Iterable[T]`, not `Sequence[T]`, as a function parameter — the looser type is more flexible for callers.
2. Return `list[T]` (concrete) rather than `Iterable[T]` (abstract) when callers will index or iterate twice. Be precise about return types.
3. Source the abstract types from `collections.abc`: `Iterable`, `Iterator`, `Sequence`, `Mapping`, `MutableMapping`, `Callable`. The `typing.*` versions are deprecated aliases.

### 7.13 — `pathlib`, `dataclasses`, `functools`, `itertools`, `collections.abc`, `contextlib`, `enum` are *the* stdlib modules.

**Reasoning, step by step:**
1. Reach for stdlib first. The above modules cover 80% of what bespoke utility classes get written for.
2. Third-party libraries that "replace" these usually add features you don't need and a dependency you do.
3. Notable exceptions where third-party is the right call: `pydantic` (validation at the boundary), `httpx`/`requests` (HTTP — stdlib `urllib` is awkward), `structlog` (structured logging — stdlib works but is bare).
4. Every dependency is an attack surface and a future upgrade. Lean on stdlib.

## Worked example

```python
from collections.abc import Iterator
from contextlib import contextmanager
from itertools import islice
from pathlib import Path
from typing import Literal, assert_never


@contextmanager
def temporary_workdir() -> Iterator[Path]:
    d = Path(tempfile.mkdtemp())
    try:
        yield d
    finally:
        shutil.rmtree(d)


def first_n_lines(path: Path, n: int) -> list[str]:
    with path.open() as f:
        return list(islice((line.rstrip() for line in f), n))


match result:
    case Approved(receipt_id=rid): logger.info("approved %s", rid)
    case Declined(reason=why): logger.warning("declined: %s", why)
    case _: assert_never(result)


# bad
for i in range(len(items)):                     # 7.10 — use enumerate
    print(items[i])

result = ""                                     # 7.6 — use f-string or join
for item in items:
    result += str(item) + ", "

with open(path) as f:                           # 7.5 — use pathlib
    data = f.read()
```

## Cross-references

- `async with` and async context managers: chapter 09.
- Generators in API design: chapter 10.
- `itertools.islice` and bounded iterators: chapter 13.
