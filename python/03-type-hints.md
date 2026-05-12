# 03 — Type Hints

Python's type system is gradual and optional. We make it mandatory at module boundaries. The cost is small; the payoff is large.

## Rules

### 3.1 — Type-hint every public function signature. Return type included.

**Reasoning, step by step:**
1. A public function's signature is its contract. Without types, the contract lives in docstrings (which rot) or in the head of whoever wrote it (which leaves).
2. Public = anything that could be imported by another module. Private = leading underscore, defined inside a function, or in a `_internal` module.
3. mypy strict mode (`disallow_untyped_defs`) flags missing hints. Run it in CI.
4. Return type matters even when "obvious": it documents and lets mypy catch implementation drift.
5. **Anti-pattern:** typing arguments but not return values, or vice versa. Both, always.

### 3.2 — Use modern syntax: `|` over `Optional`, `list[X]` over `List[X]`.

**Reasoning, step by step:**
1. PEP 604 (Python 3.10+) allows `str | None` in place of `Optional[str]`. Shorter and more obvious.
2. PEP 585 (Python 3.9+) allows `list[int]` in place of `List[int]`. No more importing from `typing`.
3. PEP 695 (Python 3.12+) adds the `type` statement and inline generic syntax: `def first[T](xs: list[T]) -> T: ...`. Use it for new code; the readability win is real.
4. Don't mix styles within a module. Pick the modern one.

### 3.3 — No `Any` in public API. `Any` is a silent type-checker disable.

**Reasoning, step by step:**
1. `Any` means "I don't know" — and mypy will let any operation pass on a value of type `Any`. It propagates: every value derived from an `Any` is also `Any`.
2. Public function signatures should never use `Any`. Replace with: `object` (for "any value, but I'll check"), `T` (generic), `Protocol` (structural).
3. Private uses of `Any` exist (FFI, dynamic loading) and are sometimes unavoidable. Comment them with `# type: ignore[reason]` or `# Any: <why>`.
4. mypy strict mode rejects `Any` returns by default.

### 3.4 — `Protocol` for behavior contracts. ABCs only when you need inheritance.

**Reasoning, step by step:**
1. `Protocol` (PEP 544) is *structural* typing — any class with the right methods is accepted, no inheritance needed.
2. ABCs (`abc.ABC`, `@abc.abstractmethod`) require inheritance. Subclasses must declare they implement the ABC.
3. Pick `Protocol` for: dependency-inversion seams, "duck-typed but checked" interfaces, function parameters describing required behavior.
4. Pick ABC when: you need shared implementation in the base, or you genuinely want `isinstance` to enforce the relationship at runtime (rare; use `@runtime_checkable` on a Protocol if you do).
5. **Pattern:**
   ```python
   class Closeable(Protocol):
       def close(self) -> None: ...

   def with_resource(r: Closeable) -> None:
       try:
           ...
       finally:
           r.close()
   ```
   Any class with a `close()` method satisfies `Closeable` — no inheritance.

### 3.5 — `Final` for module constants and intentionally-non-overridable attributes.

**Reasoning, step by step:**
1. `MAX_RETRIES: Final[int] = 3` says: this value never changes. mypy enforces.
2. On class attributes: `class Config: timeout: Final[int] = 30` prevents subclasses or instances from overriding.
3. Use for: configuration constants, computed-once values, class attributes that genuinely shouldn't be touched.
4. Don't use for every local — `Final` is annotation overhead; reserve for the cases where the immutability matters.

### 3.6 — `Literal` for enum-like discriminators and string flags.

**Reasoning, step by step:**
1. `Literal["asc", "desc"]` is a tight type for a string flag with two valid values.
2. mypy enforces — passing `"random"` is a type error.
3. **Discriminator pattern** for sum types:
   ```python
   class Approved(TypedDict):
       kind: Literal["approved"]
       receipt: Receipt

   class Declined(TypedDict):
       kind: Literal["declined"]
       reason: str

   ChargeResult = Approved | Declined
   ```
   mypy narrows on `result["kind"] == "approved"` automatically.
4. For larger sets, prefer `enum.StrEnum` (3.11+) — gives you string values *and* a closed type.

### 3.7 — `TypedDict` for JSON-shaped dictionaries. Dataclasses for everything else.

**Reasoning, step by step:**
1. `TypedDict` types a `dict` with known string keys and per-key types. Useful for JSON request/response shapes, config dicts, kwargs bags.
2. Dataclasses are for *objects* — bound methods, identity, lifetime. TypedDict is for *records*.
3. **Rule of thumb:** if it crosses the wire as JSON, `TypedDict` (or a Pydantic model — see [chapter 14 in kotlin-jvm](../kotlin-jvm/05-serialization.md) for the equivalent JVM patterns; we apply a similar split here, see this guide's serialization chapter).
4. Inside the domain: `@dataclass(frozen=True, slots=True)`.

### 3.8 — Generics over `Any`. Type variables for "the type the caller picks."

**Reasoning, step by step:**
1. `def first(xs: list) -> Any` loses type information at every call. mypy can't help downstream.
2. `def first[T](xs: list[T]) -> T:` (PEP 695, 3.12+) — caller gets back the type they passed in.
3. Constrain type variables when needed: `T: bound=Comparable`, `T: int | float`.
4. Pre-3.12: `T = TypeVar("T")` at module scope, then `def first(xs: list[T]) -> T`. Verbose but works.

### 3.9 — `Self` for fluent APIs and chained calls (Python 3.11+).

**Reasoning, step by step:**
1. `def with_header(self, k: str, v: str) -> Self` says "I return my own type" — including in subclasses.
2. Pre-3.11: a `TypeVar` bound to the class. Verbose.
3. With `Self`, subclasses get correct types automatically: `SubRequest.with_header(...)` returns `SubRequest`, not `Request`.

### 3.10 — Forward references and `from __future__ import annotations`.

**Reasoning, step by step:**
1. `from __future__ import annotations` makes all annotations strings, lazily evaluated. Fixes forward references; speeds up imports.
2. Adopt this in new modules. PEP 563 was deferred-permanent; PEP 649 (Python 3.13+) replaces it with a more uniform mechanism — same source-code shape.
3. Side effect: annotations are strings at runtime. Libraries that introspect type hints at runtime (Pydantic, FastAPI) handle this with `typing.get_type_hints`; you do the same if you must inspect annotations.

### 3.11 — No `# type: ignore` without a reason and an owner.

**Reasoning, step by step:**
1. `# type: ignore` disables mypy on that line. It's a sometimes-necessary escape valve.
2. **Required form:** `# type: ignore[error-code]  # <why>`. Specific error code, comment with reasoning.
3. Track `# type: ignore` count in CI — it shouldn't grow unbounded.
4. Same goes for `# noqa`: always with a code and a reason.

### 3.12 — `cast` is honest; `# type: ignore` is silent. Prefer `cast`.

**Reasoning, step by step:**
1. `typing.cast(T, value)` tells mypy "trust me, this is a T." It's a runtime no-op but a visible source-level claim.
2. `# type: ignore` says "stop checking" — broader, more dangerous.
3. When you have a value that mypy can't narrow but you know the type, `cast` documents the claim at the exact place.
4. Either way, the underlying issue is usually that your types are imprecise. Fix the types first; reach for `cast` only when refactoring isn't feasible.

## Worked example

```python
from __future__ import annotations

from collections.abc import Iterable
from dataclasses import dataclass
from typing import Final, Literal, Protocol, Self


MAX_RETRIES: Final[int] = 3


class UserReader(Protocol):
    def find(self, user_id: UserId) -> User | None: ...


@dataclass(frozen=True, slots=True)
class UserId:
    raw: str

    def __post_init__(self) -> None:
        if not self.raw:
            raise ValueError("UserId must be non-empty")


def load_users[T: User](reader: UserReader, ids: Iterable[UserId]) -> list[T]:
    return [reader.find(i) for i in ids if reader.find(i) is not None]  # type: ignore[return-value]  # narrowing limitation
```

## Cross-references

- Dataclasses + Protocols + ABCs: chapter 06.
- mypy configuration: chapter 01.
- Pydantic / kwargs / TypedDict for JSON boundaries: chapter 14 (serialization equivalent) and chapter 10 (API design).
