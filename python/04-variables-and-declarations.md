# 04 — Variables & Declarations

How values come into being and how long they live. Python's flexibility around mutability is its biggest footgun; the rules here close the obvious traps.

## What good looks like

```python
from dataclasses import dataclass
from typing import ClassVar, Final

MAX_RETRIES: Final[int] = 3
_RETRYABLE_CODES: Final[frozenset[int]] = frozenset({429, 503})


@dataclass(frozen=True, slots=True)
class RetryPolicy:
    attempts: int = MAX_RETRIES
    backoff: float = 0.5
    RETRYABLE: ClassVar[frozenset[int]] = _RETRYABLE_CODES


def expand(codes: list[int], extra: list[int] | None = None) -> list[int]:
    if extra is None:
        extra = []
    seen: set[int] = set()
    return [c for c in [*codes, *extra] if not (c in seen or seen.add(c))]


def first_retryable(codes: list[int]) -> int | None:
    if (hit := next((c for c in codes if c in _RETRYABLE_CODES), None)) is not None:
        return hit
    return None
```

`MAX_RETRIES` and `_RETRYABLE_CODES` are `Final` module constants in `SCREAMING_SNAKE_CASE` (4.3), the immutable set is a `frozenset` rather than a mutable `set` (4.2); `RetryPolicy` is a frozen, slotted dataclass (4.9) whose `RETRYABLE` is a `ClassVar`, not a per-instance field (4.4). `expand` defaults its collection argument to `None` and materializes inside (4.1) and annotates the widening local `seen` (4.6); `first_retryable` binds the walrus result once and reuses it (4.5). The module declares names and compiles values only — no side effects on import (4.7).

## Rules

### 4.1 — Never use mutable default arguments.

**Reasoning, step by step:**
1. `def append(item, items=[]):` evaluates the `[]` once at function definition time. Every call without an `items` argument shares the *same list*. State leaks across invocations.
2. This is the most well-known Python bug and somehow still ships.
3. **Fix:** default to `None`, materialize inside:
   ```python
   def append(item: T, items: list[T] | None = None) -> list[T]:
       if items is None:
           items = []
       items.append(item)
       return items
   ```
4. Same trap for `dict`, `set`, custom mutable objects. The fix is always: `None` default + initialize inside.
5. Ruff's `B006` rule catches this. Enable it; treat warnings as errors.

**Enforcement:** Ruff `B006` (mutable default argument), errors in CI.

### 4.2 — `tuple` over `list` when contents don't change.

**Reasoning, step by step:**
1. `tuple` is immutable. `list` is mutable. Picking the right one documents intent and prevents accidental mutation.
2. For constants and configured values: `tuple` (or `frozenset`/`frozendict`-equivalent via `types.MappingProxyType`).
3. For collections that *do* change: `list`/`set`/`dict`.
4. Performance side: tuples are slightly cheaper to construct and iterate. For hot paths, this matters.
5. **Anti-pattern:** `tuple` because "I want immutability," then casting to `list` at every use site. Pick the right type and stop.

**Enforcement:** review; mypy flags mutation of a `tuple`/`frozenset`-typed value.

### 4.3 — Module-level constants use `Final` and `SCREAMING_SNAKE_CASE`.

**Reasoning, step by step:**
1. `MAX_RETRIES: Final[int] = 3` — mypy enforces no reassignment.
2. `SCREAMING_SNAKE_CASE` for the name, by convention.
3. For lazily-initialized "constants" (regexes, parsers, configs): use `functools.lru_cache` on a no-arg function, or a module-level call evaluated at import time.
4. **Anti-pattern:** module-level mutable state (`_cache: dict = {}`). Either make it explicitly thread-safe (`threading.Lock`) or move it into a class.

**Enforcement:** mypy `Final` reassignment error; review for `SCREAMING_SNAKE_CASE` and module-level mutable state.

### 4.4 — `ClassVar` distinguishes class attributes from dataclass fields.

**Reasoning, step by step:**
1. In a `@dataclass`, plain class-level annotations become *instance* fields. `ClassVar` marks them as class-level (shared, not per-instance).
2. ```python
   @dataclass
   class Config:
       name: str
       MAX_SIZE: ClassVar[int] = 1024  # class attribute, not a field
   ```
3. Forget the `ClassVar` and you've added a per-instance field by accident.
4. mypy enforces correct `ClassVar` usage; trust it.

**Enforcement:** mypy validates `ClassVar` against dataclass field semantics.

### 4.5 — Walrus operator (`:=`) only when it removes a duplicate computation.

**Reasoning, step by step:**
1. `if (n := len(items)) > 10: print(f"got {n}")` — `n` is bound once, used twice. Clearer than computing `len(items)` twice.
2. `if (m := pattern.match(line)) is not None: process(m.group(1))` — same idea: bind, test, use.
3. **Anti-pattern:** `if (x := foo()):` when you don't reuse `x`. Just write `if foo():`.
4. **Anti-pattern:** walrus inside complex expressions for one-character savings. If the assignment isn't obvious to a fast reader, lift it out.

**Enforcement:** review; a walrus whose binding is used once is rejected.

### 4.6 — Type inference exists, but type annotations on locals are sometimes worth it.

**Reasoning, step by step:**
1. Most local variables don't need annotations. mypy infers `x = 42` as `int`.
2. Annotate locals when (a) the type isn't obvious from the right-hand side, (b) the variable's type widens later (`results: list[Result] = []`), (c) you want to prevent accidental retyping.
3. `xs: list[int] = []` is the canonical pattern. Otherwise mypy infers `list[Any]`.
4. Don't over-annotate. Three annotations per line is noise.

**Enforcement:** mypy `--disallow-any-generics` surfaces inferred `list[Any]`; review for over-annotation.

### 4.7 — Module-level side effects on import: forbidden.

**Reasoning, step by step:**
1. Importing a module should *not* hit the network, open a database connection, write a file, or print anything to stdout.
2. Imports happen at unpredictable times — at startup, at test collection, when a tool introspects the module.
3. Put initialization inside functions or class constructors. Module-level code should declare names, not perform actions.
4. Acceptable module-level work: building lookup tables, compiling regexes, setting up `__all__` and constants. Cheap, deterministic, side-effect-free.

**Enforcement:** review; import-time linters flagging I/O at module scope.

### 4.8 — `del` for genuine deletion. Not for "go away please."

**Reasoning, step by step:**
1. `del x` unbinds `x`. Useful inside `__slots__` classes, in code that owns large buffers, or to make a name unavailable to subsequent code.
2. `del x` is *not* a way to "free memory" — the object is freed when refcount hits zero, which may not be immediately after `del`.
3. Use `del` sparingly. Most "I want to forget this variable" intentions are better served by smaller functions where the variable simply goes out of scope.

**Enforcement:** review; `del` used for genuine unbinding, not as a memory-management hint.

### 4.9 — `__slots__` for hot-path classes and dataclasses (3.10+).

**Reasoning, step by step:**
1. `__slots__` declares a fixed set of attributes. Instances skip the per-object `__dict__` — faster attribute access, lower memory.
2. `@dataclass(slots=True)` (3.10+) generates `__slots__` for you. Use it on every value-shaped dataclass.
3. Trade-off: no dynamic attribute assignment (`obj.new_attr = 1` raises `AttributeError`). For value types this is a *feature*; for plugin-shaped classes, it's a constraint.
4. Multi-inheritance + slots is finicky. Most value types don't multi-inherit, so this rarely matters.

**Enforcement:** review; value-shaped dataclasses carry `slots=True`.

### 4.10 — One assignment per line. Chained assignment only when it reads clearly.

**Reasoning, step by step:**
1. `a = b = c = 0` is fine — three names bound to the same value, clear intent.
2. `a, b, c = compute()` is fine — unpacking from a tuple-return.
3. `a = compute_b(b := compute_c(c := init()))` is not fine. Lift each step into its own line.
4. **Rule:** assignment chains are OK when they share a value or are a destructuring. Not OK when they smuggle in walrus operators or function calls.

**Enforcement:** review; chained assignment limited to shared values and unpacking.

## Worked example

```python
from typing import Final, ClassVar
from dataclasses import dataclass

MAX_RETRIES: Final[int] = 3
_DEFAULT_TIMEOUT: Final[float] = 5.0


@dataclass(frozen=True, slots=True)
class PaymentConfig:
    timeout: float = _DEFAULT_TIMEOUT
    retries: int = MAX_RETRIES
    REQUIRED_FIELDS: ClassVar[tuple[str, ...]] = ("amount", "currency")


# bad
def add(item, items=[]):                       # 4.1 — mutable default!
    items.append(item)
    return items

maxRetries = 3                                  # 4.3 — should be SCREAMING_SNAKE + Final
print("module loaded")                          # 4.7 — side effect on import
```

## Cross-references

- Dataclasses + `slots=True`: chapter 06.
- `Final` and type discipline: chapter 03.
- Resource initialization patterns: chapter 13.
