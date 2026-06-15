# 06 — Classes & Data Modeling

Python makes classes easy. We make most of them dataclasses, most contracts Protocols, and most "inheritance for reuse" something else.

## What good looks like

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Literal, Protocol, assert_never


class Currency(StrEnum):
    USD = "USD"
    EUR = "EUR"


@dataclass(frozen=True, slots=True)
class Money:
    minor_units: int
    currency: Currency

    def __post_init__(self) -> None:
        if self.minor_units < 0:
            raise ValueError("Money must be non-negative")


@dataclass(frozen=True, slots=True)
class Approved:
    receipt_id: str
    kind: Literal["approved"] = "approved"

@dataclass(frozen=True, slots=True)
class Declined:
    reason: str
    kind: Literal["declined"] = "declined"

ChargeResult = Approved | Declined


class PaymentGateway(Protocol):
    async def charge(self, amount: Money) -> ChargeResult: ...


async def settle(gateway: PaymentGateway, amount: Money) -> str:
    result = await gateway.charge(amount)
    match result:
        case Approved(receipt_id=rid): return rid
        case Declined(reason=why): raise RuntimeError(why)
        case _: assert_never(result)
```

`Money` is a `frozen=True, slots=True` value type validating its invariant in `__post_init__` (6.1, 6.6); the non-defaulted fields precede the defaulted `kind` discriminator in every variant; `Currency` is a `StrEnum` closed set (6.2); `ChargeResult` is a `Literal`-discriminated union narrowed by an exhaustive `match` with `assert_never` (6.4); `PaymentGateway` is a `Protocol`, so `settle` depends on a shape, not a base class (6.3, 6.5).

## Rules

### 6.1 — `@dataclass(frozen=True, slots=True)` for value-shaped types.

**Reasoning, step by step:**
1. A "value-shaped" type is one whose identity is its content: a `Point(x, y)`, a `UserId`, a `PaymentRequest`. Two with the same fields are equal.
2. `@dataclass` gives you `__init__`, `__repr__`, `__eq__` (and `__hash__` if `frozen=True`) for free. Writing these by hand is wasted typing and a future bug.
3. **Always `frozen=True`** for value types. An equal-but-mutable record breaks `__hash__` stability — put one in a `set` and mutate it, and the set is broken.
4. **Always `slots=True` (3.10+)** for hot-path value types — saves memory and speeds attribute access. Cost: no dynamic attribute assignment, which for value types is a feature.
5. Don't add behavior beyond simple derived properties. Dataclasses are records, not service objects.
6. **Pattern:**
   ```python
   @dataclass(frozen=True, slots=True)
   class UserId:
       raw: str

       def __post_init__(self) -> None:
           if not self.raw:
               raise ValueError("UserId must be non-empty")
   ```

**Enforcement:** review; value-shaped types carry `frozen=True, slots=True`, no hand-written `__init__`/`__eq__`/`__hash__`.

### 6.2 — `enum.Enum` (or `StrEnum`/`IntEnum`) for closed sets.

**Reasoning, step by step:**
1. When a value can only be one of a finite set (`OrderStatus`, `Currency`, `LogLevel`), make it an enum.
2. `enum.StrEnum` (3.11+) gives string-valued enums where the value *is* the wire format: `Currency.USD == "USD"`. Useful at JSON/serialization boundaries.
3. `enum.IntEnum` for integer-valued sets (HTTP status codes, OS signal numbers).
4. Plain `enum.Enum` for sets where values are arbitrary tokens.
5. **Anti-pattern:** stringly-typed constants (`STATUS_ACTIVE = "active"`). Use an enum. mypy + IDE completion both win.

**Enforcement:** mypy rejects an out-of-set literal where the enum is expected; review flags stringly-typed constant clusters.

### 6.3 — `Protocol` over ABC. Inheritance only when it's actually shared behavior.

**Reasoning, step by step:**
1. `Protocol` is structural typing — any class with the right methods is accepted. No inheritance needed.
2. ABC requires subclassing. Subclasses declare the relationship.
3. **Choose `Protocol` when:** you want dependency-inversion (the caller provides any object with the right shape) — most "interface" use cases.
4. **Choose ABC when:** you need shared *implementation* in the base, or you need `isinstance` checks to work at runtime (use `@runtime_checkable` on a Protocol if you only need that).
5. From the broader principle: prefer composition over inheritance. Inheritance is reserved for genuine "is-a" relationships and *shared implementation* (chapter 06 §6.4 of the Kotlin guide makes the same case).

**Enforcement:** review; `Protocol` for dependency-inversion seams, ABC reserved for shared implementation or `isinstance` needs.

### 6.4 — Sum types: `Literal` discriminator + union, or sealed-class-style hierarchies.

**Reasoning, step by step:**
1. Python doesn't have native sealed classes. We approximate with:
   - **Discriminated unions:**
     ```python
     @dataclass(frozen=True, slots=True)
     class Approved:
         receipt: Receipt
         kind: Literal["approved"] = "approved"

     @dataclass(frozen=True, slots=True)
     class Declined:
         reason: str
         kind: Literal["declined"] = "declined"

     ChargeResult = Approved | Declined
     ```
   - mypy narrows on `match result.kind:` or `if isinstance(result, Approved):`.
2. **Exhaustive `match`:** use `assert_never` from `typing` for the unreachable case:
   ```python
   from typing import assert_never

   match result:
       case Approved(receipt=r): handle(r)
       case Declined(reason=why): log(why)
       case _: assert_never(result)
   ```
   Adding a new variant fails the type check until you add a case.
3. For a fixed set of *behaviors* (not just data), an ABC with two subclasses works too. Pick by whether the variants share methods or just shapes.

**Enforcement:** mypy's `assert_never` fails the type check when a new variant is added without a `match` case.

### 6.5 — No inheritance for code reuse. Use composition + Protocol.

**Reasoning, step by step:**
1. Inheritance couples lifecycle. A subclass is a parent — any change to the parent affects every subclass.
2. The 1990s "Animal → Dog → Poodle" hierarchy is almost always wrong in real code. Real systems have orthogonal axes that don't form a tree.
3. Composition: take the dependency as a constructor arg, expose what you need on your own surface.
4. **Acceptable inheritance:** (a) extending stdlib/framework classes when forced (`Exception`, `Enum`, ABCs you're implementing), (b) sealed-style hierarchies where the variants share the parent as a tag, (c) genuinely shared implementation with a single subclass-author audience.
5. **Anti-pattern:** `class UserService(BaseService): ...` where `BaseService` exists only because three services happened to need a logger. Inject the logger.

**Enforcement:** review; reject `Base*`/`Abstract*` parents that exist only to share a dependency rather than model an "is-a".

### 6.6 — Constructors: simple, single-purpose, raise on invalid input.

**Reasoning, step by step:**
1. `__init__` (or `__post_init__` for dataclasses) should: (a) store the parameters, (b) validate invariants, (c) raise on violation. Nothing else.
2. No I/O in constructors. No "convenience" file reads, no network calls. Construction must be cheap and predictable.
3. For complex creation: use a classmethod factory (`User.from_dict(...)`, `Config.load(...)`). Document side effects in the factory's docstring.
4. **Pattern:** dataclasses with `__post_init__` for validation. Regular classes with `__init__` for everything else.

**Enforcement:** review; no I/O in `__init__`/`__post_init__`, invariants validated and raised at construction.

### 6.7 — `@classmethod` and `@staticmethod`: pick by what `self`/`cls` you need.

**Reasoning, step by step:**
1. `@classmethod` receives the class as `cls`. Use for: alternative constructors (`from_json`, `from_dict`, `default`), factory methods that need to know the subclass.
2. `@staticmethod` receives nothing. Use for: utility functions that *belong* to the class namespace but don't need `self` or `cls`. Honestly, this is rare — usually a module-level function is better.
3. **Anti-pattern:** `@staticmethod` for what should be a top-level function. The class wrapping adds nothing.

**Enforcement:** review; `@classmethod` only where `cls` is used, `@staticmethod` only where the class namespace earns its place over a module function.

### 6.8 — Multiple inheritance: avoid; if necessary, use mixins with clear MRO.

**Reasoning, step by step:**
1. Python's MRO (method resolution order) is C3 linearization. It's correct, but tracing it through five base classes is a debugging nightmare.
2. Mixins (small, focused classes that add one capability) are acceptable: `class User(LoggingMixin, JsonMixin, SerializableMixin): ...`. Keep mixins small and orthogonal.
3. Diamond inheritance (two parents share a grandparent) usually means you should be composing instead.
4. **Lint:** detekt-equivalent doesn't exist for this in Python tooling, but the rule stands. Code review catches it.

**Enforcement:** review; multiple inheritance limited to small orthogonal mixins, diamonds rejected in favor of composition.

### 6.9 — Dunders: implement only the ones you mean. Don't go overboard.

**Reasoning, step by step:**
1. Pythonic classes participate in language protocols via dunders: `__eq__`, `__hash__`, `__iter__`, `__contains__`, `__len__`, `__getitem__`, `__enter__`/`__exit__` (context manager), `__call__`.
2. Implement them when your class *is* that thing. `__iter__` if your class is iterable. `__len__` if it has a meaningful length.
3. **Anti-pattern:** implementing `__add__` because "it might be useful." Operator overloading is for types where the operation reads naturally — `Vector + Vector`, `Duration + Duration`. Not `User + Order`.
4. Keep dunders consistent: if you implement `__eq__`, you almost always need `__hash__`. Mismatch breaks Python's contracts (e.g., set membership).

**Enforcement:** review; a dunder appears only when the class genuinely *is* that protocol, and `__eq__`/`__hash__` stay paired.

### 6.10 — `NamedTuple` vs `dataclass` vs `TypedDict`: pick by use case.

**Reasoning, step by step:**
1. **`@dataclass(frozen=True, slots=True)`** — default choice. Mutable or immutable, fast, supports methods, integrates with type system.
2. **`NamedTuple`** — when the value should *also* be a tuple (positional access, iterable, indexable). Common for return values with positional meaning: `Coordinate(x, y)`. Slightly cheaper than dataclass for pure-data use.
3. **`TypedDict`** — when the value is a *dict* (JSON shape, kwargs bag, config). No methods, no constructor logic.
4. Pick by the consumer's needs: do they index, unpack, mutate, or pattern-match? Each implies a different choice.

**Enforcement:** review; the carrier matches the consumer — `dataclass` by default, `NamedTuple` for positional/tuple use, `TypedDict` for dict shapes.

### 6.11 — Properties for derived state. Direct attributes for stored state.

**Reasoning, step by step:**
1. If a value is derived (`user.full_name` from `first + last`), make it a `@property` or a `@cached_property`.
2. If a value is stored (`user.email`), it's a regular attribute.
3. Don't fake encapsulation with getter/setter methods. Python has properties for a reason.
4. Don't add a property "in case we need it later." YAGNI — and the moment you do need it, change the attribute to a property; callers don't see the difference.

**Enforcement:** review; derived state is a `@property`/`@cached_property`, stored state is a plain attribute, no getter/setter method pairs.

### 6.12 — `__hash__` and equality: consistent or absent.

**Reasoning, step by step:**
1. `__eq__` without `__hash__` means the class isn't hashable (can't be a dict key or set element). Sometimes correct, often a bug.
2. `@dataclass(frozen=True)` generates a consistent `__eq__` + `__hash__`. Use it.
3. `@dataclass(eq=False)` falls back to identity equality (`is`). Useful for entities where two records with the same fields *aren't* equal because they have different identities (databases, sessions).
4. The Python rule: equal objects must have equal hashes. Breaking this corrupts dicts and sets — silently. Use `frozen=True` and forget about it.

**Enforcement:** review; equality and hashing come from `frozen=True` or are deliberately absent (`eq=False` for identity), never inconsistently hand-rolled.

### 6.13 — Factory `@classmethod`s over polymorphic constructors.

**Reasoning, step by step:**
1. When a class can be constructed from multiple sources (a URL, a connection string, a dict, a parsed JSON blob), don't overload `__init__` with sentinel arguments and `if isinstance(...)` branches.
2. **Use classmethod factories.** Each takes one input type, validates it, and delegates to the canonical `__init__`. Reads as a sentence at the call site: `Client.from_connection_string(s)`, `Client.from_url(u, credential)`, `User.from_dict(payload)`.
3. **Pattern (service client — regular class, not a dataclass):**
   ```python
   from typing import Self

   class PaymentClient:
       def __init__(self, endpoint: str, credential: TokenCredential, *, timeout: float = 30.0) -> None:
           self._endpoint = endpoint
           self._credential = credential
           self._timeout = timeout

       @classmethod
       def from_connection_string(cls, conn_str: str, *, timeout: float = 30.0) -> Self:
           endpoint, credential = _parse_conn_str(conn_str)
           return cls(endpoint, credential, timeout=timeout)

       @classmethod
       def from_environment(cls, *, timeout: float = 30.0) -> Self:
           return cls.from_connection_string(os.environ["PAYMENTS_CONN_STR"], timeout=timeout)
   ```

   Note: a service client is *not* a frozen dataclass — it owns runtime state (the credential may rotate, the transport pool tracks connections). Use a regular class with `__init__`. Value-shaped types are the ones that get `@dataclass(frozen=True, slots=True)` (§6.1).
4. **Pattern (value type — dataclass):**
   ```python
   from typing import Self

   @dataclass(frozen=True, slots=True)
   class User:
       id: UserId
       email: Email

       @classmethod
       def from_dict(cls, payload: Mapping[str, Any]) -> Self:
           return cls(id=UserId(payload["id"]), email=Email(payload["email"]))
   ```
5. The canonical `__init__` (or `__post_init__` for dataclasses) is the *one* path that validates invariants. Factories construct inputs and delegate.
6. Factory methods may do small I/O (env lookup, parsing). Document side effects in the factory's docstring; keep them limited and predictable.
7. **Anti-pattern:** `def __init__(self, endpoint=None, conn_str=None, url=None): ... if conn_str: ...; elif url: ...`. The signature is a lie and mypy can't help.
8. Return type `Self` (Python 3.11+) — subclass factories return the subclass naturally.

**Enforcement:** review; multi-source construction goes through named `@classmethod` factories returning `Self`, not an `__init__` with optional sentinel arguments.

## Worked example

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Literal, Protocol, assert_never


class Currency(StrEnum):
    USD = "USD"
    EUR = "EUR"
    GBP = "GBP"


@dataclass(frozen=True, slots=True)
class UserId:
    raw: str

    def __post_init__(self) -> None:
        if not self.raw:
            raise ValueError("UserId must be non-empty")


@dataclass(frozen=True, slots=True)
class Approved:
    receipt_id: str
    kind: Literal["approved"] = "approved"

@dataclass(frozen=True, slots=True)
class Declined:
    reason: str
    kind: Literal["declined"] = "declined"

ChargeResult = Approved | Declined


class PaymentGateway(Protocol):
    async def charge(self, user_id: UserId, amount: int, currency: Currency) -> ChargeResult: ...


async def settle(gateway: PaymentGateway, user_id: UserId, amount: int) -> None:
    result = await gateway.charge(user_id, amount, Currency.USD)
    match result:
        case Approved(receipt_id=rid): logger.info("approved %s", rid)
        case Declined(reason=why): logger.warning("declined: %s", why)
        case _: assert_never(result)


# bad
class AbstractPayment:                          # 6.5 — open inheritance not needed
    pass

class Payment(AbstractPayment):
    def __init__(self):
        self.id = ""                            # 6.1 — should be a frozen dataclass
        self.amount = 0
```

## Cross-references

- `Protocol` and structural typing: chapter 03.
- Pattern matching as a refactor-safety tool: chapter 07.
- Sealed/discriminated unions in error handling: chapter 08.
- Client constructor signature and immutability: chapter 10.
