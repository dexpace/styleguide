# 02 — Naming Conventions

PEP 8's naming rules are good enough. We follow them and add a small number of project conventions.

## Rules

### 2.1 — `snake_case` for everything except classes.

**Reasoning, step by step:**
1. Functions, methods, variables, parameters, module names: `snake_case`.
2. Classes (and class-shaped things — `NamedTuple`, `Enum`, `TypedDict`, `Protocol`): `PascalCase`.
3. Module-level constants: `SCREAMING_SNAKE_CASE`.
4. Type variables: `T`, `K`, `V` for canonical use; `TItem`, `TKey`, `TValue` (PascalCase with `T` prefix) when you need descriptive names. PEP 484 allows both.
5. Acronyms in class names are treated as words: `HttpClient`, `XmlParser`. Not `HTTPClient` (that's the stdlib's choice for `urllib`; new code in our style uses `HttpClient`).

### 2.2 — Underscore prefixes signal visibility intent. Two leading underscores trigger name-mangling — use sparingly.

**Reasoning, step by step:**
1. `_name` — convention: "internal, don't import from outside the module." Not enforced by the language.
2. `__name` (two leading, no trailing) — triggers name-mangling in classes (`obj._Class__name`). Use only when you're inheriting and need to avoid collisions; rare.
3. `__name__` (two leading and trailing) — reserved for dunders. Don't invent your own.
4. **Rule:** `_` for internal. `__` only when name-mangling is genuinely needed (subclassing scenarios). Never invent new dunders.

### 2.3 — Module names: `snake_case`, short, descriptive.

**Reasoning, step by step:**
1. PEP 8: lowercase, optional underscores. `payment_client.py`, not `paymentClient.py` or `PaymentClient.py`.
2. Short and singular. `user.py` not `users.py`, `payment.py` not `payments.py` — unless the module genuinely covers the *collection*.
3. No Python keywords or stdlib collisions. `email.py` next to the stdlib `email` package will hurt.

### 2.4 — Boolean variables and functions: `is_*`, `has_*`, `should_*`.

**Reasoning, step by step:**
1. `is_active`, `has_default_card`, `should_retry` reads as English.
2. `not active` reads as "this isn't active." `not is_active` reads as "not is active" — slightly more awkward, but clearer about the negation.
3. Negative-form names compound badly. `not_ready` → `not not_ready` is a double-negative head-scratcher.

### 2.5 — Functions describe *actions*, classes describe *things*, properties describe *state*.

**Reasoning, step by step:**
1. Functions: verb-first. `parse_iso_date()`, `load_user()`, `close()`.
2. Classes: noun. `UserRepository`, `PaymentRequest`, `JsonParser`.
3. Properties / class attributes: noun or `is_`/`has_` boolean. `user.email`, `request.is_authenticated`.
4. **Beware:** `*Manager`, `*Helper`, `*Util`, `*Handler` — these often signal a class with no single responsibility. Consider a top-level function or a split into smaller classes.

### 2.6 — Single-character variables only in tight scope.

**Reasoning, step by step:**
1. `for i in range(n):` is fine — the scope is one line.
2. `for u in users: load(u)` is borderline — the scope is small but `u` carries no domain meaning.
3. `for user in users:` is the safe default. Three extra letters cost nothing.
4. Exception: `x`, `y`, `z` for coordinates; `i`, `j`, `k` for indices; `n` for counts; `_` for "ignored." These are the canonical short names.

### 2.7 — Test names: long, descriptive, sentence-shaped.

**Reasoning, step by step:**
1. `test_returns_404_when_user_does_not_exist` is what you want to see in a test report.
2. `test_user_404` reads like a code, not a sentence.
3. Use `snake_case` (Python convention) — backticks for spaces aren't a Python feature.
4. Test names appear in CI output and flakiness dashboards. Treat them as public API of the test suite.

### 2.8 — Constants: module-level, `SCREAMING_SNAKE_CASE`, type-annotated.

**Reasoning, step by step:**
1. Module-level constants get a type annotation: `MAX_RETRIES: Final[int] = 3`.
2. `typing.Final` documents and enforces that the binding doesn't get reassigned (mypy checks).
3. `SCREAMING_SNAKE_CASE` makes constant-vs-variable visible at the call site.
4. **Anti-pattern:** writing `MAX_RETRIES = 3` inside a function. That's not a constant — it's a re-evaluated local. Move to module scope, or just use the literal.

### 2.9 — Avoid Hungarian notation and type prefixes.

**Reasoning, step by step:**
1. `str_name`, `b_is_active`, `i_count` are non-Pythonic. Type hints make this redundant.
2. `IUser` (Java/C# interface prefix) is wrong for Protocol classes — use `User` for the Protocol if it's the canonical contract.
3. Acceptable affixes: `is_`/`has_` for booleans (2.4), `on_*` for callbacks.
4. **Don't suffix classes with `Async`** (`BookingClientAsync` is wrong). Sync and async client classes share the class name and differ by module path (`acme.booking.BookingClient` vs `acme.booking.aio.BookingClient`). See [9.13](./09-concurrency.md).
5. A function-level `_async` suffix is acceptable only when a sync and async *function* coexist in the same module and the module isn't large enough to split. Prefer splitting.

### 2.10 — Type variables: short and meaningful.

**Reasoning, step by step:**
1. `T = TypeVar("T")` for generic "any type."
2. `K = TypeVar("K")`, `V = TypeVar("V")` for key/value.
3. `T_co = TypeVar("T_co", covariant=True)`, `T_contra = TypeVar("T_contra", contravariant=True)` — variance is in the name.
4. For complex generic signatures, descriptive names help: `TUser = TypeVar("TUser", bound=User)`. PEP 695's `type` statement and `def foo[T](...)` syntax (Python 3.12+) reduce the boilerplate.

### 2.11 — Service-client classes end in `Client`.

**Reasoning, step by step:**
1. A class that owns connections, credentials, retry policies, and the call surface to an external service is a *client*. Name it that way: `PaymentClient`, `BookingClient`, `IndexClient`.
2. Not `PaymentProxy`, `PaymentManager`, `PaymentService`, `PaymentAPI`. The suffix is `Client` and only `Client`. Consistency lets a reader find the entry point by completion in 1–2 keystrokes.
3. Specialized sub-clients (when a service has nested resources) follow the same rule. A `payments` attribute on a `BookingClient` returns a `PaymentClient`, not a `PaymentSubClient` or `Payments`.
4. **From Azure SDK guidelines:** "DO name service client types with a `Client` suffix."

### 2.12 — Method verb taxonomy for resource-shaped operations.

**Reasoning, step by step:**
1. When methods operate on resources (CRUD-ish APIs, REST clients, repositories), pick verbs from a known taxonomy. Reader instantly knows the semantics; the whole API surface behaves consistently.
2. **Verb → semantics:**

   | Verb | Semantics |
   |---|---|
   | `get_<noun>` | Fetch a resource. Raises if missing. Returns the entity. |
   | `list_<noun>` | Enumerate resources. Returns a pageable iterator (never `None`, never a raw list of "all"). |
   | `create_<noun>` | Create a new resource. Raises if it already exists. |
   | `upsert_<noun>` | Create or update. Idempotent. |
   | `update_<noun>` | Modify an existing resource. Raises if missing. |
   | `replace_<noun>` | Full replacement (PUT semantics). |
   | `delete_<noun>` | Remove a resource. Succeeds (no-ops) even if missing. |
   | `<noun>_exists` | Returns `bool`. Does NOT raise on "not found" — that's a normal response. Raises only on network/server errors. |
   | `begin_<noun>` | Long-running operation. Returns a poller. See [10.17](./10-api-design.md). |
   | `append_<noun>` | Append to a collection. |

3. **Rule:** don't invent new verbs when one of these fits. Don't reuse a verb against its documented semantics — `delete_user(id)` must not raise on missing.
4. **Anti-pattern:** `fetch_user`, `read_user`, `find_user` — pick `get_user` and be consistent. Synonyms read like a poorly-organized API.
5. **Cross-language consistency:** when an organization ships SDKs in multiple languages for the same service, align the verbs across languages — `GetUser` (Go), `getUser` (Kotlin), `get_user` (Python) all mean the same thing. The taxonomy here is the Python form.

## Worked example

```python
# good
from typing import Final, Protocol

MAX_RETRIES: Final[int] = 3
_DEFAULT_TIMEOUT: Final[float] = 5.0  # module-private


class UserReader(Protocol):
    def find(self, user_id: UserId) -> User | None: ...


def load_user(reader: UserReader, user_id: UserId) -> User:
    user = reader.find(user_id)
    if user is None:
        raise UserNotFound(user_id)
    return user


class BookingClient:
    def get_booking(self, booking_id: BookingId) -> Booking: ...       # 2.12 — raises if missing
    def list_bookings(self, *, customer_id: CustomerId | None = None) -> ItemPaged[Booking]: ...
    def create_booking(self, request: BookingRequest) -> Booking: ...   # raises if exists
    def delete_booking(self, booking_id: BookingId) -> None: ...        # no-ops if missing
    def booking_exists(self, booking_id: BookingId) -> bool: ...        # returns bool


# bad
maxRetries = 3                        # 2.1 — should be MAX_RETRIES
class IUser:                          # 2.5 / 2.9 — drop the prefix
    pass
def loadUserById(id):                 # 2.1 — should be snake_case; `id` shadows builtin
    pass
class BookingClientAsync: ...         # 2.9 / 9.13 — use acme.booking.aio.BookingClient
class BookingManager: ...             # 2.11 — should be BookingClient
def fetch_booking(): ...              # 2.12 — should be get_booking
```

## Cross-references

- `Protocol` and structural typing: chapter 06.
- `Final` and module constants: chapter 04.
- Test naming: chapter 11.
- Client constructor and method shapes that pair with this verb taxonomy: chapter 10.
