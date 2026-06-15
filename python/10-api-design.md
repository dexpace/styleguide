# 10 — API Design

Designing the surface other code calls. Python's lack of enforcement makes API discipline more important, not less.

## What good looks like

```python
from collections.abc import Iterator
from datetime import datetime
from typing import Protocol

__all__ = ["Clock", "ReportService", "Report"]


class Clock(Protocol):
    def now(self) -> datetime: ...


class ReportService:
    def __init__(self, endpoint: str, credential: TokenCredential, *, clock: Clock, timeout: float = 30.0) -> None:
        self._endpoint = endpoint
        self._credential = credential
        self._clock = clock
        self._timeout = timeout

    def fetch_report(self, report_id: ReportId, *, timeout: float | None = None) -> Report:
        stamped = self._clock.now().isoformat()
        return self._get(report_id, deadline=timeout or self._timeout, at=stamped)

    def list_reports(self, *, since: datetime | None = None) -> Iterator[Report]:
        yield from self._page(since=since)
```

`ReportService` takes its binding target and credential positionally, then a `*`, then keyword-only configuration (10.13); `clock` is a `Protocol` seam, not a concrete class (10.3); `fetch_report` makes `timeout` keyword-only and lets the per-call value override the per-client default (10.2, 10.15). `__all__` declares the public surface (10.1), every signature is fully typed including returns (10.6), and `list_reports` returns a lazy `Iterator` rather than an eager list (10.9).

## Rules

### 10.1 — `__all__` declares the public surface of every published module.

**Reasoning, step by step:**
1. Python has no `public`/`private` modifier. `__all__` is the convention: a tuple/list of names that constitute the module's API.
2. Without `__all__`, every top-level name is implicitly public. Refactoring becomes a guessing game.
3. Names not in `__all__` are conventionally `_private` (leading underscore). Importing them from outside the module is a "you bought it, you own it" decision.
4. **Pattern:**
   ```python
   __all__ = ["UserId", "User", "load_user", "UserNotFound"]
   ```
5. `from module import *` respects `__all__` — only the named symbols are imported.

**Enforcement:** review; ruff `F822` flags undefined names in `__all__`, every published module declares it.

### 10.2 — Keyword-only arguments for public functions past two positionals.

**Reasoning, step by step:**
1. Restated from chapter 05 §5.2: positional arguments lose meaning at the call site after the obvious first one or two.
2. `*` after the last positional forces keyword:
   ```python
   def fetch(url: str, *, timeout: float = 5.0, retries: int = 3) -> Response: ...
   ```
3. **Hard rule:** booleans are *always* keyword-only at call sites. `set_visible(True)` is OK because the function name implies the parameter; `with_retries(True)` is not.
4. Same-typed adjacent parameters must be named: `crop(image, 10, 20, 30, 40)` — which two are width/height? Force keywords.

**Enforcement:** review; ruff `FBT001`/`FBT002` flag boolean positional arguments, `*` separator on publics past two positionals.

### 10.3 — `Protocol` for dependency-inversion seams.

**Reasoning, step by step:**
1. When a function takes a dependency it doesn't construct, the parameter's type should be a `Protocol` describing required behavior — not the concrete class.
2. Reasons: testability (any Protocol-conforming fake works), looser coupling, no inheritance imposed on callers.
3. **Pattern:**
   ```python
   class Clock(Protocol):
       def now(self) -> datetime: ...

   def stamp(message: str, clock: Clock) -> str:
       return f"{clock.now().isoformat()}: {message}"
   ```
4. The Protocol can be defined in the same module as the function, in a `protocols.py` sibling, or in the consumer module — wherever it reads best.
5. Avoid ABCs for this. ABCs require callers to inherit; Protocols just require the right shape.

**Enforcement:** review; dependency parameters type as a `Protocol`, mypy structural-typing check confirms fakes conform.

### 10.4 — Generics for "the type the caller picks."

**Reasoning, step by step:**
1. Python 3.12+ has PEP 695 syntax: `def first[T](xs: list[T]) -> T:`. Use it.
2. Pre-3.12: declare a `TypeVar` at module scope, then use it in signatures. Verbose, works.
3. Constrain when meaningful: `T: bound=Comparable` for sortable, `T: int | float` for numeric.
4. **Don't reach for generics prematurely.** Many functions don't need them — passing `Iterable[str]` is more honest than `Iterable[T]` if the function only handles strings.

**Enforcement:** review; PEP 695 syntax on 3.12+, mypy checks the type variable is actually bound by a parameter.

### 10.5 — Default arguments document the contract. Mutable defaults are banned.

**Reasoning, step by step:**
1. Restated from chapters 04 and 05: defaults are part of the API. Pick the value most callers want.
2. Never mutable defaults. Use `None` and initialize inside. (Chapter 04 §4.1.)
3. Add new optional parameters via keyword-only + default. Adding a *positional* parameter with a default is binary-compatible but call-site-incompatible if callers used positional invocation.

**Enforcement:** ruff `B006` blocks mutable defaults; review for new optionals added as keyword-only.

### 10.6 — Type-hint everything in the public surface. Return types included.

**Reasoning, step by step:**
1. Restated from chapter 03: every public function signature is fully typed. mypy strict mode enforces.
2. Type stubs (`.pyi`) when the implementation can't carry hints (C extensions, generated code) but the package publishes a typed API.
3. PEP 561: a typed package includes a `py.typed` marker file. Without it, type checkers won't inspect your stubs.

**Enforcement:** mypy strict mode (`disallow_untyped_defs`); CI checks the `py.typed` marker ships in the wheel.

### 10.7 — Deprecation: `warnings.warn` with `DeprecationWarning`, `stacklevel=2`.

**Reasoning, step by step:**
1. Removing a public function is a breaking change. Deprecate first:
   ```python
   def old_name(*args, **kwargs):
       warnings.warn(
           "old_name is deprecated; use new_name instead. Removed in 3.0.",
           DeprecationWarning,
           stacklevel=2,
       )
       return new_name(*args, **kwargs)
   ```
2. `stacklevel=2` makes the warning point at the *caller*, not the deprecated function. Always.
3. Document the removal release. "Deprecated forever" is a lie callers learn not to trust.
4. Keep the deprecated function shimmed for at least one release cycle.

**Enforcement:** review; tests assert the `DeprecationWarning` fires with `stacklevel=2`, removal release named in the message.

### 10.8 — Versioning: follow semver, document breaking changes.

**Reasoning, step by step:**
1. `MAJOR.MINOR.PATCH` — major = breaking, minor = backward-compatible additions, patch = backward-compatible fixes.
2. Breaking changes need a major bump. No exceptions.
3. Removing a public symbol, narrowing a parameter type, or changing return semantics is breaking. Adding new parameters with defaults (kwarg-only!) is not.
4. Document breaking changes in `CHANGELOG.md` with migration paths.

**Enforcement:** review; CI runs an API-diff tool (e.g. `griffe check`) that fails the build when a breaking change lands without a major bump.

### 10.9 — Generators / iterators at the API boundary: lazy by default, materialize at the consumer.

**Reasoning, step by step:**
1. Return an iterator/generator when the result *could* be large or infinite. The consumer chooses when to materialize.
2. Return a `list`/`tuple` when the result is bounded and the consumer will iterate multiple times.
3. **Document the contract:** `Iterator[T]` is single-pass; `Iterable[T]` may be re-iterable; `list[T]` is materialized.
4. Don't accept `list` when `Iterable` would do — be liberal in what you accept, strict in what you produce.

**Enforcement:** review; return type spells `Iterator[T]`/`Iterable[T]`/`list[T]` to state the pass-and-materialization contract, mypy enforces.

### 10.10 — `async def` is part of the contract. Don't pretend it isn't.

**Reasoning, step by step:**
1. An `async def` returns a coroutine. Callers must `await`.
2. Don't expose both sync and async versions of the same function unless both are first-class. If you must: name them `do_thing` (sync) and `do_thing_async` (async). Document.
3. Generally: pick one. Mixed APIs invite confusion.

**Enforcement:** review; ruff `ASYNC` lint catches blocking calls in `async def`, the `_async` suffix convention checked at review.

### 10.11 — Don't expose mutable internals.

**Reasoning, step by step:**
1. Returning an internal `list` lets the caller mutate your state. Return a copy or a `tuple`.
2. `types.MappingProxyType(dict)` for "read-only view of a dict you own."
3. Accepting `list` is fine — you can copy or treat it as `Iterable`. But returning a mutable reference is hostile.

**Enforcement:** review; public return types are `tuple`/`Mapping`/`MappingProxyType`, mypy flags a mutable return that aliases internal state.

### 10.12 — Pipeline composition via callables: `list[Step]` + `functools.reduce` or a small loop.

**Reasoning, step by step:**
1. For a transformation that's the composition of small steps, model it as `list[Step]` and fold:
   ```python
   Step = Callable[[Request], Request]

   def run_pipeline(steps: list[Step], initial: Request) -> Request:
       result = initial
       for step in steps:
           result = step(result)
       return result
   ```
2. Each step is one function with one responsibility. Composable, testable, swappable.
3. Use in place of: deep inheritance hierarchies, classes-with-overridable-hooks, decorator chains that hide ordering.
4. Bound the list. `len(steps) > 100` deserves a comment about why.

**Enforcement:** review; each `Step` is a typed `Callable` alias, composition is a fold over a bounded `list[Step]`.

### 10.13 — Service-client constructor signature: positional required, keyword-only optional.

**Reasoning, step by step:**
1. The canonical client constructor takes two positional arguments — the *binding target* (endpoint/URL/resource identifier) and the *credential* — then a `*`, then keyword-only optional arguments.
2. **Pattern:**
   ```python
   class PaymentClient:
       def __init__(
           self,
           endpoint: str,
           credential: TokenCredential,
           *,
           api_version: str | None = None,
           transport: Transport | None = None,
           max_retries: int = 3,
           timeout: float = 30.0,
           application_id: str | None = None,
           **kwargs: Any,  # forwarded to pipeline policies
       ) -> None:
           ...
   ```
3. Two positional args, no more. The rest is keyword-only. The reader knows what to type without consulting the signature.
4. Optional construction paths go through `@classmethod` factories (chapter 06 §6.13) — `from_connection_string`, `from_environment`, `from_url`.
5. **Anti-pattern:** five positional arguments for a client constructor. Force keywords.
6. **From Azure SDK guidelines:** "DO provide a constructor that takes positional binding parameters (e.g., service name or URL), a positional `credential` parameter, a `transport` keyword-only parameter, and keyword-only arguments for pipeline policies."

**Enforcement:** review; client `__init__` declares at most two positionals before `*`, alternative paths are `@classmethod` factories.

### 10.14 — No options-bag objects. Pass keyword-only arguments individually.

**Reasoning, step by step:**
1. `client.do_thing(options=Options(retries=3, timeout=5))` reads worse than `client.do_thing(retries=3, timeout=5)`. The options-bag adds a layer of indirection callers see at every call site.
2. Options bags also break IDE completion (the bag's fields aren't surfaced as `do_thing`'s parameters), defeat keyword-argument validation, and accumulate orphan parameters over time.
3. **Rule:** every optional parameter that callers tune per call is keyword-only. Direct. Named in the function signature.
4. **Exception:** a small *configuration* dataclass that's shared by many methods and rarely changes (a `RetryPolicy`, a `TransportConfig`) is fine as a *constructor* parameter — callers see it once at construction, not at every call site.
5. **`**kwargs` is not an options bag.** Forwarding `**kwargs` to pipeline policies (as in 10.13's constructor signature) is the Python idiom for "I accept the open set of policy-level options without naming every one." It's the *typed bag passed as a single argument* (`Options(...)`) at every call site that this rule forbids.

**Enforcement:** review; per-call options are individual keyword-only parameters, no `options=`-style single-bag argument on methods.

### 10.15 — Clients are immutable after construction. Per-call kwargs override per-client kwargs.

**Reasoning, step by step:**
1. A client's configuration (endpoint, credential, retries, timeout, transport) is set at construction. No setters.
2. Per-call overrides go through the same kwarg names: `client.fetch(url, timeout=60)` overrides the client's default timeout for this call only.
3. The kwarg names at the method *must mirror* the kwarg names at the constructor. Don't rename `timeout` to `request_timeout` at one level — readers shouldn't have to learn two vocabularies.
4. **From Azure SDK guidelines:** "DO design the client to be immutable. ... There should not be any scenarios that require callers to change properties/attributes of the client."
5. **Anti-pattern:** `client.set_timeout(60)`. Either pass it to the constructor or pass it per call. Don't mutate the client.

**Enforcement:** review; no setters on the client, per-call kwarg names mirror the constructor's exactly.

### 10.16 — Collections return a pageable iterator protocol. Never raw `None`. Never an unbounded eager list.

**Reasoning, step by step:**
1. A `list_*` method returns *something iterable* even when there are zero results — never `None`, never raises for "no items."
2. **Return shape:** an iterator/iterable that supports both element-level iteration and page-level iteration:
   ```python
   for item in client.list_users():           # element-level
       ...

   for page in client.list_users().by_page(): # page-level for batch processing
       ...
   ```
3. Build a small `ItemPaged[T]` (or `AsyncItemPaged[T]`) protocol — the underlying server-driven paging is invisible to most callers, but reachable when they need it.
4. **Continuation tokens** belong on `.by_page(continuation_token=...)`, not on the top-level `list_*` method. The list method's signature is for filtering and selection — pagination is a separate concern.
5. **Anti-pattern:** `def list_users() -> list[User]` that pages internally and returns every result eagerly. For "small enough" collections this is fine; for unbounded ones it's a memory leak.

**Enforcement:** review; `list_*` return type is `ItemPaged[T]`/`AsyncItemPaged[T]`, never `None` and never an unbounded eager `list`.

### 10.17 — Long-running operations: `begin_*` prefix + poller return.

**Reasoning, step by step:**
1. A long-running operation (LRO) is one where the server returns immediately with an "in progress" handle, and the result becomes available later (often minutes).
2. Method name starts with `begin_`: `begin_create_index`, `begin_export`, `begin_restore`.
3. Return type is a *poller* — an object exposing `.result()` (blocks until done), `.status()` (current state), `.wait()` (block, no result), and `.cancel()` (request cancellation).
4. Async clients return an async poller whose methods are awaitable: `await poller.result()`, `await poller.wait()`.
5. **Sync pattern:**
   ```python
   poller = client.begin_export(index_id)
   # ... do other work ...
   result = poller.result(timeout=300)
   ```
6. **Async pattern:**
   ```python
   poller = await async_client.begin_export(index_id)
   async with asyncio.timeout(300):
       result = await poller.result()
   ```
7. **From Azure SDK guidelines:** "DO use a `begin_` prefix for all long running operations. DO return an object that implements the Poller protocol."

**Enforcement:** review; LRO methods carry the `begin_` prefix and return a poller, mypy checks the poller `Protocol` is satisfied.

### 10.18 — Conditional requests: `etag` + `match_condition` keyword arguments.

**Reasoning, step by step:**
1. For services that support optimistic concurrency (ETag/If-Match), expose two keyword-only kwargs at every method that supports them:
   - `etag: str | None = None` — the ETag value to send.
   - `match_condition: MatchCondition | None = None` — the condition (`IfMatch`, `IfNoneMatch`, `IfModifiedSince`, `IfUnmodifiedSince`).
2. If the caller passes a *model* with an `etag` attribute and *also* an explicit `etag` kwarg, the kwarg wins (explicit beats implicit).
3. Define `MatchCondition` once as a small enum; reuse across the SDK.
4. **Pattern:**
   ```python
   def update_user(
       self,
       user: User,
       *,
       etag: str | None = None,
       match_condition: MatchCondition | None = None,
   ) -> User: ...
   ```

**Enforcement:** review; concurrency-capable methods expose the `etag`/`match_condition` keyword-only pair, `MatchCondition` defined once as a shared enum.

### 10.19 — Accept `Mapping` (dict-like) alongside typed models for input parameters.

**Reasoning, step by step:**
1. Callers who want type safety pass a `User` dataclass. Callers who have a raw JSON-ish dict shouldn't have to construct one first.
2. Accept both: `def create_user(self, user: User | Mapping[str, Any]) -> User`. Internally, normalize: if it's a `Mapping`, run it through the validator/parser.
3. The typed path stays the recommended one (it's what completion shows). The `Mapping` path is the escape hatch.
4. Don't accept arbitrary kwargs as a "flat" alternative unless the model has very few fields — that's overload-explosion in disguise.

**Enforcement:** review; input parameters type as `Model | Mapping[str, Any]`, the `Mapping` path normalizes through the validator.

### 10.20 — Don't validate service parameters. Validate client parameters.

**Reasoning, step by step:**
1. **Client parameters** (endpoint URL syntax, credential type, transport config) are caller-controlled and not visible to the server. Validate them at the constructor. Raise on malformed.
2. **Service parameters** (request bodies, field values inside a model) are validated by the server. Don't reproduce that validation in the client — it goes stale the moment the service changes.
3. Exceptions: format-only checks (a UUID must look like a UUID before the request, a string must be non-empty when the type says so). Validate the *shape* the type promises; let the server validate the *semantics*.
4. **From Azure SDK guidelines:** "DO validate client parameters. ... DO NOT validate service parameters. Don't do null checks, empty string checks, or other common validating conditions on service parameters."

**Enforcement:** review; constructor validates client parameters and raises, methods forward service parameters without re-validating semantics.

### 10.21 — Identify your library in `User-Agent`; integrate distributed tracing.

**Reasoning, step by step:**
1. Every outgoing HTTP call from your client library should include a `User-Agent` header that identifies the library version and (optionally) the calling application.
2. **Pattern:** `User-Agent: <library-name>/<library-version> (<runtime>; <os>) <application-id>`. The `application_id` is a keyword-only constructor argument so callers can identify themselves further.
3. Integrate with distributed tracing (OpenTelemetry is the de facto standard): every client method that does I/O is a span. Span name matches the method name; span attributes carry the resource being acted on (entity type, ID, operation).
4. Tracing is *opt-in at the application level*: the application installs and configures an OpenTelemetry exporter. Your library always emits spans through the OpenTelemetry API; the exporter decides whether they're shipped anywhere. This means no runtime cost when tracing is unconfigured and zero-line-change activation when it is.

**Enforcement:** review; integration tests assert the `User-Agent` header and that every I/O method opens an OpenTelemetry span named for the method.

## Cross-references

- `Protocol` semantics: chapter 03, chapter 06.
- Deprecation cycle in published libraries: chapter 12.
- ABI stability is N/A in Python (no compiled binaries), but semver discipline is the analog.
- Service-client class naming and verb taxonomy: chapter 02 §§ 2.11, 2.12.
- Factory `@classmethod` constructors: chapter 06 §6.13.
- Sync/async client separation: chapter 09 §9.13.
