# 05 — Functions

Functions are the unit of reasoning. Python makes them easy to write; we add a few rules so they stay easy to read.

## What good looks like

```python
from collections.abc import Iterator
from pathlib import Path


def read_orders(path: Path, *, limit: int = 10_000) -> Iterator[Order]:
    """Stream orders from a file, one per line, bounded by `limit`."""
    with path.open() as f:
        for i, line in enumerate(f):
            if i >= limit:
                return
            yield Order.from_line(line)


def total_due(order: Order, *, tax_rate: float = 0.0) -> Cents:
    """Pure: line subtotal plus tax. Same inputs, same output."""
    assert tax_rate >= 0.0, f"tax_rate must be non-negative, got {tax_rate}"
    subtotal = sum(item.price * item.qty for item in order.items)
    return Cents(round(subtotal * (1.0 + tax_rate)))


async def settle(path: Path, *, tax_rate: float) -> Cents:
    grand_total = Cents(0)
    for order in read_orders(path):           # streaming — bounded memory
        grand_total += total_due(order, tax_rate=tax_rate)  # pure helper
    await ledger.record(grand_total)          # I/O — at the boundary
    return grand_total
```

`read_orders` yields rather than building a list (5.11) and keeps each function under the 50-line ceiling doing one thing (5.1); `tax_rate` and `limit` are keyword-only with documented defaults (5.2, 5.3); every parameter and return is hinted (5.4); `total_due` is pure with side effects pushed to `settle`'s boundary (5.9), and `async def` is honored with `await` (5.12).

## Rules

### 5.1 — One function, one purpose. 50 lines is the ceiling.

**Reasoning, step by step:**
1. A function should do *one* thing — described by its name without "and."
2. Hard cap: 50 lines including blanks and the closing dedent. Aim 10–25. Docstrings don't count.
3. If a function approaches 50, extract a helper, lift a comprehension out, or split into smaller functions.
4. Top-level functions, methods, lambdas, and nested functions all count.

**Enforcement:** Ruff `PLR0915` (too-many-statements) plus a line-count check in review.

### 5.2 — Keyword-only arguments for everything past the obvious positional.

**Reasoning, step by step:**
1. `f(a, b, c, d, e)` at the call site tells the reader nothing. `f(a, b, retries=3, timeout=5, verbose=True)` does.
2. Python supports keyword-only arguments via `*`:
   ```python
   def fetch(url: str, *, timeout: float = 5.0, retries: int = 3) -> Response: ...
   ```
   `timeout` and `retries` must be named at call sites.
3. **Rule:** anything that isn't part of the function's primary identity is keyword-only. Booleans are *always* keyword-only at call sites.
4. **Anti-pattern:** five positional arguments. After two or three, force keyword.

**Enforcement:** Ruff `PLR0913` (too-many-arguments) and `FBT` (boolean-positional) flags; review for the `*` separator.

### 5.3 — Default arguments document the contract. No surprises in the default.

**Reasoning, step by step:**
1. Defaults appear in help text, IDE completion, and the function's `__defaults__`. They're part of the API.
2. Pick defaults that callers want most of the time. `timeout: float = 5.0` is a real choice; document it.
3. Mutable defaults are banned (chapter 04 §4.1). Use `None` + initialize.
4. Sentinel defaults for "absent vs `None`": `_MISSING = object()` then `if value is _MISSING:`. Rare but sometimes necessary.

**Enforcement:** Ruff `B006` (mutable-default) blocks mutable defaults; review for default choice.

### 5.4 — Type-hint every parameter and the return.

**Reasoning, step by step:**
1. Restated from chapter 03 §3.1 because it bears repeating: public function signatures get full type hints.
2. `-> None` for void functions. Explicit, not omitted.
3. Private functions (`_underscored`, defined inside another function) may skip hints if the type is unambiguous. The strict-mode trade-off is local.

**Enforcement:** mypy `--strict` (`disallow_untyped_defs`) on public functions.

### 5.5 — Decorators: explicit, type-preserving, sparingly applied.

**Reasoning, step by step:**
1. A decorator modifies a function. Stack three of them and the reader has to mentally trace the wrapping order.
2. Common safe decorators: `@functools.wraps` (always inside your own decorators), `@property`, `@classmethod`, `@staticmethod`, `@dataclass`, `@functools.lru_cache`.
3. Custom decorators must use `@functools.wraps(func)` so the wrapped function keeps its name, docstring, and signature.
4. **Anti-pattern:** decorator that swallows the return value, changes the signature implicitly, or has side effects on import. Type-preserve and document.
5. Type a decorator's signature so mypy understands what it does:
   ```python
   def retry[**P, R](func: Callable[P, R]) -> Callable[P, R]: ...
   ```

**Enforcement:** mypy checks `ParamSpec`-preserving signatures; review for `@functools.wraps` and stacking depth.

### 5.6 — `*args` and `**kwargs`: only for true forwarding.

**Reasoning, step by step:**
1. `*args` and `**kwargs` are escape hatches. They hide the API and disable type checking.
2. Acceptable uses: (a) decorators that forward to the wrapped function, (b) inheritance where you call `super().__init__(*args, **kwargs)`, (c) factory functions that pass through to a constructor.
3. Otherwise: write the real parameters. mypy and IDE completion will thank you.
4. Even when you use them, type them: `def wrap(*args: int, **kwargs: str) -> None`. Not `Any`.

**Enforcement:** review for non-forwarding use; mypy `disallow_any_explicit` blocks `Any`-typed varargs.

### 5.7 — Expression-bodied lambdas only when the call site benefits.

**Reasoning, step by step:**
1. `sorted(xs, key=lambda x: x.age)` is fine — short, contextual, no name needed.
2. `lambda x, y: x + y` for `reduce`? Use `operator.add`. Use the stdlib.
3. Anything more than ~1 line of body: write a named `def`. Lambdas can't have annotations, can't have docstrings, can't be debugged easily.
4. **Lint:** Ruff's `E731` (don't assign a lambda) catches `f = lambda x: ...` — that should always be a `def`.

**Enforcement:** Ruff `E731` (lambda-assignment).

### 5.8 — `functools` is your friend.

**Reasoning, step by step:**
1. `@functools.lru_cache(maxsize=N)` for memoizing pure functions. **Bound `maxsize`** (chapter 15 §15.5) — unbounded caches are leaks.
2. `@functools.cache` is `lru_cache(maxsize=None)` — only use when you can prove the input set is bounded.
3. `functools.partial(func, fixed_arg=value)` for partial application. Clearer than a lambda wrapping the call.
4. `functools.reduce` exists. Use it for *associative* reductions. For others, a `for` loop is clearer.
5. `functools.singledispatch` for ad-hoc polymorphism by first-argument type. Useful, niche.

**Enforcement:** Ruff `B019` (cached-method on instance) plus review for bounded `maxsize`.

### 5.9 — Pure functions where you can; side effects at the boundary.

**Reasoning, step by step:**
1. A pure function: same input → same output, no observable side effect. Trivially testable, parallelizable, cache-able.
2. Push I/O, logging, and state mutation to the edges. A thin shell calls pure helpers.
3. **Pattern:**
   ```python
   async def load_and_process(user_id: UserId) -> Report:
       raw = await http.fetch(user_id)         # I/O — shell
       report = build_report(raw)              # pure — testable
       await audit.log(user_id, report)        # I/O — shell
       return report
   ```
4. Not religion. A logger call inside a function is fine. A function that does I/O, parsing, validation, and persistence is not.

**Enforcement:** review; I/O confined to shell functions, pure helpers free of side effects.

### 5.10 — `@property` for cheap, idempotent read-only views. Not for I/O.

**Reasoning, step by step:**
1. `@property` makes a method look like an attribute. Callers don't see `obj.size()` — they see `obj.size`.
2. Therefore: properties must be *cheap* and *idempotent*. Callers expect attribute-access cost.
3. Doing I/O in a property is a bug-magnet. Reading from a database in `obj.value` surprises every caller.
4. `@functools.cached_property` for "compute once, cache, behave like an attribute." Useful when the computation is expensive and the value doesn't change after first access.

**Enforcement:** review; no I/O or mutation inside `@property` bodies.

### 5.11 — Generators for streaming. `yield` over building a list when the list is large or unbounded.

**Reasoning, step by step:**
1. A function that yields produces a lazy iterator. Memory is bounded; consumers pull when ready.
2. `def read_lines(path: Path) -> Iterator[str]: ...` is better than building a list of millions of lines.
3. `yield from` for delegating to an inner iterator.
4. **Trap:** generators are *single-pass*. Iterating twice doesn't restart. If the caller needs multi-pass, `list(generator())` or rethink the API.
5. See chapter 07 for the Pythonic-idiom flavor of this rule.

**Enforcement:** review; `Iterator`/`Iterable` return hint for streaming, no list materialization of unbounded input.

### 5.12 — `async def` is part of the signature. Don't pretend.

**Reasoning, step by step:**
1. An `async def` function returns a coroutine. Callers must `await` it.
2. mypy distinguishes `Coroutine[Any, Any, T]` from `T` — type-check accordingly.
3. Don't mix sync and async unintentionally: a function should be one or the other.
4. Bridge with care: `asyncio.run(coro)` at the program entry, `asyncio.to_thread(sync_fn, ...)` inside async code for blocking calls.

**Enforcement:** Ruff `RUF006`/`ASYNC` rules plus mypy coroutine-type checking; review for un-awaited coroutines.

## Worked example

```python
from collections.abc import Iterator
from functools import cached_property, lru_cache
from pathlib import Path


@lru_cache(maxsize=128)
def parse_iso_date(s: str) -> date:
    return date.fromisoformat(s)


def read_records(path: Path, *, limit: int = 10_000) -> Iterator[Record]:
    with path.open() as f:
        for i, line in enumerate(f):
            if i >= limit:
                return
            yield Record.from_line(line)


class UserView:
    def __init__(self, user: User) -> None:
        self._user = user

    @cached_property
    def display_name(self) -> str:
        return f"{self._user.first} {self._user.last}".strip()


# bad
def append(item, items=[]):                     # mutable default!
    ...

def f(a, b, c, d, e, f, g):                     # 5.2 — keyword-only past two args
    ...

result = (lambda x, y: x + y)(1, 2)             # 5.7 — use a def or operator.add
```

## Cross-references

- Async functions in depth: chapter 09.
- Generators and iteration patterns: chapter 07.
- Memoization and cache-size bounds: chapter 15.
