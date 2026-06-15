# 08 — Error Handling

Exceptions are Pythonic; we use them. Discipline around exceptions — what to catch, when to chain, when to wrap — is what separates serviceable Python from production-grade Python.

## What good looks like

```python
from dataclasses import dataclass


class PaymentError(Exception):
    """Base for every payment failure."""


@dataclass
class CardDeclined(PaymentError):
    card_last4: str
    reason: str
    request_id: str

    def __str__(self) -> str:
        return f"card ****{self.card_last4} declined: {self.reason} (request {self.request_id})"


class GatewayUnavailable(PaymentError):
    """The payment gateway could not be reached."""


def charge_card(card: Card, amount: int, request_id: str) -> Receipt:
    if amount <= 0:                                          # 8.7 caller mistake
        raise ValueError(f"amount must be positive, got {amount}")

    try:
        response = gateway.submit(card.tokenize(), amount)
    except GatewayTimeout as e:                              # 8.3 narrow catch
        raise GatewayUnavailable(f"gateway unreachable (request {request_id})") from e  # 8.4, 8.5

    if response.code == "DECLINED":
        raise CardDeclined(card.last4, response.reason, request_id)  # 8.2 typed, context-carrying
    return response.to_receipt()
```

`charge_card` raises rather than returning a sentinel (8.14), and the choice holds throughout. `PaymentError` is a typed hierarchy with context fields (8.2); the `except` narrows to the one expected fault (8.3); `from e` chains the gateway failure forward (8.4); the boundary wraps so the domain never sees a raw `GatewayTimeout` (8.5); the input check splits the caller's mistake from the operational failures (8.7); messages name the request id and mask the card (8.12). The calling request handler logs once and maps each type to a status code.

## Rules

### 8.1 — Raise specific exceptions. Never bare `raise Exception(...)`.

**Reasoning, step by step:**
1. `Exception` is the parent of almost everything. Catching it (or its grandchildren) catches things you didn't mean to catch.
2. Raise the most specific built-in exception that fits: `ValueError` for bad value, `TypeError` for bad type, `KeyError` for missing dict key, `LookupError` for general lookup failure, `OSError` for I/O.
3. For domain failures: define a custom hierarchy (8.2).
4. **Anti-pattern:** `raise Exception("oh no")`. Callers can't distinguish your error from anything else.

**Enforcement:** review; ruff `TRY002`/`TRY003` flag raising bare `Exception` and long inline messages.

### 8.2 — Custom exception hierarchies for domains.

**Reasoning, step by step:**
1. Group domain exceptions under a single root: `class PaymentError(Exception): ...`. Subclasses for specific failures: `class CardDeclined(PaymentError): ...`.
2. Callers can catch the root for generic handling, or specific subclasses for precise recovery.
3. Each exception carries the relevant context: the input, the correlation ID, the failure mode.
4. **Pattern:**
   ```python
   class PaymentError(Exception):
       """Base for all payment-related failures."""

   @dataclass
   class CardDeclined(PaymentError):
       card_last4: str
       reason: str
       request_id: str

       def __str__(self) -> str:
           return f"card ****{self.card_last4} declined: {self.reason} (request {self.request_id})"
   ```
5. Hierarchy depth: two levels is usually right. Five-level exception trees are no easier to navigate than two-level ones.

**Enforcement:** review; domain failures subclass a single root exception and declare context fields.

### 8.3 — Catch only what you can handle. Never bare `except:`.

**Reasoning, step by step:**
1. Bare `except:` catches *everything* — including `KeyboardInterrupt`, `SystemExit`, and bugs in finally clauses. It's almost always wrong.
2. `except Exception:` is broader than you want but at least doesn't swallow control-flow exceptions. Use only at boundaries (top of a worker, end of a request handler) where logging + re-raising is the goal.
3. Catch specific exceptions: `except (KeyError, ValueError):`. Each catch is a deliberate decision about what recovery means.
4. **Re-raise** if you can't recover: `except SpecificError: cleanup(); raise`. Logging + dropping is *not* recovery.

**Enforcement:** ruff `E722` (bare `except`) and `BLE001` (blind `except Exception`) outside boundaries.

### 8.4 — Exception chaining with `raise ... from`.

**Reasoning, step by step:**
1. Wrapping an exception to add context: `raise PaymentError(...) from caught_error`. The original is preserved in `__cause__`.
2. **Always chain** when wrapping. Without `from`, you lose the debug information that says *what actually went wrong*.
3. `raise NewError(...) from None` suppresses the chain — only use when the underlying cause is genuinely irrelevant (rare).
4. Tracebacks show both exceptions: "the above exception was the direct cause of the following exception" or "during handling of the above exception, another exception occurred."

**Enforcement:** ruff `B904` requires `from` (or `from None`) on every `raise` inside an `except`.

### 8.5 — Wrap exceptions at module boundaries.

**Reasoning, step by step:**
1. A `psycopg2.OperationalError` from your repository should not propagate to your domain logic untouched. The domain doesn't know about Postgres.
2. Boundary functions catch and wrap:
   ```python
   try:
       row = db.fetchone(sql, params)
   except psycopg2.OperationalError as e:
       raise StorageUnavailable(...) from e
   ```
3. The wrapping function preserves the cause (`from e`) so debug context isn't lost.
4. **Anti-pattern:** wrapping every exception into a custom one that carries no extra information. Either add information (correlation ID, request context) or don't wrap.

**Enforcement:** review; adapter layers translate dependency exceptions into domain types with `from e`.

### 8.6 — Never silently swallow. If you must continue, log loudly.

**Reasoning, step by step:**
1. `except SomeError: pass` is almost always a bug. Errors are *information* — discarding them discards information.
2. If you genuinely want to continue: `logger.exception("expected failure mode, continuing")`. The `logger.exception` includes the stack trace.
3. **`contextlib.suppress`** for the rare case where the error is expected and uninteresting: `with suppress(FileNotFoundError): path.unlink()`. Reads as "yes, I know this might fail, and I don't care."
4. The set of suppressed exceptions in `suppress()` should be tiny — usually one. `suppress(Exception)` is just bare `except`.

**Enforcement:** ruff `S110`/`SIM105` flag `except: pass`; review forbids broad `suppress`.

### 8.7 — Validate inputs with `if not ...: raise`. Use `assert` for invariants.

**Reasoning, step by step:**
1. **Input validation** is for caller mistakes. Raise `ValueError`/`TypeError` with a useful message:
   ```python
   if not 0 <= probability <= 1:
       raise ValueError(f"probability must be in [0, 1], got {probability}")
   ```
2. **`assert`** is for *our* invariants — things that should always be true if the program is correct. `assert n > 0, "n must be positive after normalization"`.
3. `assert` is removed when Python runs with `-O`. Therefore: never rely on `assert` for security checks or input validation that must run in production.
4. Two distinct intents, two distinct tools.

**Enforcement:** ruff `S101` flags `assert` in non-test modules; review confirms validation uses `raise`.

### 8.8 — `finally` for cleanup that *must* happen. `with` for paired resources.

**Reasoning, step by step:**
1. `with` (chapter 07 §7.1) is the right tool when there's an `__enter__`/`__exit__` protocol — files, locks, transactions, subprocess handles.
2. `try/finally` for cleanup that doesn't fit the context-manager protocol: releasing a lock acquired in a complex condition, flushing a buffer, restoring global state.
3. `finally` runs even on exception. Don't put anything in `finally` that can itself fail unexplained; if it can, log loudly.

**Enforcement:** ruff `B012` flags control flow in `finally`; review prefers `with` over manual `try/finally`.

### 8.9 — Re-raising preserves the traceback. Don't reconstruct.

**Reasoning, step by step:**
1. `except SomeError as e: raise e` re-raises but *loses the original traceback frame*. The traceback starts at the `raise e` line, not the original throw.
2. Bare `raise` (no value) preserves the original traceback. Use it: `except SomeError: cleanup(); raise`.
3. If you genuinely need a different exception, use `raise NewError(...) from e`.

**Enforcement:** ruff `TRY201` flags `raise e`; use bare `raise` to re-raise.

### 8.10 — Top-of-process exception handler: log with correlation, exit with a code.

**Reasoning, step by step:**
1. Every long-running process has a top-level `try/except Exception:` that catches anything that escaped the request handler / event loop.
2. Top-level handler responsibilities: (a) log with correlation ID and full traceback, (b) increment a metric, (c) reply with an opaque error to the client, (d) decide whether to crash or continue.
3. For batch jobs and CLI tools: log, exit with a non-zero code. Don't silently succeed.
4. For services: log, return 500 with a correlation ID, *keep serving other requests*.

**Enforcement:** review; entrypoints carry one boundary handler that logs with correlation and sets the exit code.

### 8.11 — `Result`-style sealed unions: optional discipline for high-rigor modules.

**Reasoning, step by step:**
1. Python's idiom is exceptions. Sealed `Result[T, E]` unions are *not* Pythonic by default.
2. **Some modules benefit from them anyway** — payment processing, ETL pipelines, anything where every failure mode must appear in the signature.
3. Pattern (similar to Kotlin's, see [kotlin/08-error-handling.md](../kotlin/08-error-handling.md)):
   ```python
   @dataclass(frozen=True, slots=True)
   class Ok[T]:
       value: T
       kind: Literal["ok"] = "ok"

   @dataclass(frozen=True, slots=True)
   class Err[E]:
       error: E
       kind: Literal["err"] = "err"

   type Result[T, E] = Ok[T] | Err[E]
   ```
4. Don't mix idioms within a module. If a module uses `Result`, everything inside it does. If a module uses exceptions, everything inside it does. The transition happens at a boundary.

**Enforcement:** review; one error style per module, `Result` variants `frozen=True`, exhaustiveness via `assert_never`.

### 8.12 — Error messages: include the inputs the caller can't see.

**Reasoning, step by step:**
1. `"validation failed"` is useless. `"order {id}: line item {i} has negative quantity {qty}"` is debuggable.
2. Include the identifying inputs to the function, not just the symptom.
3. Don't include secrets in messages — keys, tokens, full PII. Mask them (chapter 13 §13.6 of the [security guide](../security.md)).
4. Messages travel into logs, stack traces, and sometimes user-facing surfaces. Treat them like a public API.

**Enforcement:** review; secret-scanning in CI; masking helpers from the [security guide](../security.md).

### 8.13 — Prefer existing exception types. Create new ones only when callers will catch them programmatically.

**Reasoning, step by step:**
1. Built-in exceptions (`ValueError`, `TypeError`, `KeyError`, `LookupError`, `OSError`, `RuntimeError`, `TimeoutError`) cover most failure modes. Use them.
2. A new exception type earns its existence by giving callers something to catch *separately* from the existing hierarchy. `class CardDeclined(PaymentError):` is justified — callers will catch it to retry with a different card. `class InvalidArgumentError(ValueError):` is not — callers won't disambiguate it from a regular `ValueError`.
3. **Rule:** before adding a new exception type, write the `except` block that *needs* it to be a separate type. If you can't, use a built-in.
4. **From Azure SDK guidelines:** "DO NOT create new exception types when a built-in exception type will suffice. YOU SHOULD NOT create a new exception type unless the developer can handle the error programmatically."

**Enforcement:** review; each new exception type is paired with an `except` block that catches it specifically.

### 8.14 — Don't raise for normal responses. Don't return `None`/`bool` to signal errors.

**Reasoning, step by step:**
1. Two failure modes get conflated in Python codebases:
   - "The operation completed and the answer is 'no'" — that's a *result*, not an error.
   - "The operation could not complete" — that's an error.
2. **`*_exists(id) -> bool`**: returns `True` or `False`. A 404 from the server is the `False` case — *not* an exception. Network failures and 5xx responses still raise.
3. **`get_user(id) -> User`**: must raise if the user doesn't exist. `get_*` says "the resource is there; fetch it." The lookup-may-miss variant is named `find_user` and returns `User | None`, or `try_get_user`.
4. **Anti-pattern:** `def create_user(...) -> bool` where `False` means "failed somehow." Use exceptions with specific types — callers can't distinguish "exists" from "validation failed" from "network down" otherwise.
5. **Anti-pattern:** returning `None` for "the operation didn't work." Callers stop type-checking the return; bugs slip through.

**Enforcement:** review; predicates return `bool`, may-miss lookups return `T | None`, failures raise.

### 8.15 — Document raised exceptions in docstrings.

**Reasoning, step by step:**
1. Exception raising isn't in the signature. The docstring's `Raises:` section is the contract.
2. List exceptions the caller might reasonably catch. Don't enumerate every `RuntimeError` that could theoretically escape.
3. Skip common Python exceptions (`ValueError`, `TypeError`) unless the function *intentionally* uses them as part of its contract.
4. **From Azure SDK guidelines:** "DO document the errors that are produced by each method."

**Enforcement:** review; public functions that raise carry a `Raises:` docstring section, checked against the body.

## Worked example

```python
from contextlib import suppress
from dataclasses import dataclass


class PaymentError(Exception):
    """Base for all payment failures."""


@dataclass
class CardDeclined(PaymentError):
    card_last4: str
    reason: str
    request_id: str


def charge_card(card: Card, amount: int) -> Receipt:
    if amount <= 0:
        raise ValueError(f"amount must be positive, got {amount}")

    try:
        response = gateway.submit(card.tokenize(), amount)
    except GatewayTimeout as e:
        raise PaymentError(f"timeout charging card ****{card.last4}") from e

    if response.code == "DECLINED":
        raise CardDeclined(
            card_last4=card.last4,
            reason=response.reason,
            request_id=response.request_id,
        )
    return response.to_receipt()


# tolerable failure: file may not exist
with suppress(FileNotFoundError):
    cache_path.unlink()


# bad
try:
    process()
except:                                         # 8.3 — bare except
    pass                                        # 8.6 — silent swallow

try:
    parse(s)
except Exception as e:
    raise ValueError("bad input")               # 8.4 — no `from e`; cause lost
```

## Cross-references

- `match`/`case` exhaustiveness with `assert_never`: chapter 07.
- Async cancellation and `asyncio.CancelledError`: chapter 09.
- Logging exceptions at the boundary: chapter on logging.
