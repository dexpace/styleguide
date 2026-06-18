# Python styleguide — full checklist

One section per chapter. Read on demand for a full audit.

### 01 — Formatting & Tooling

- Ruff formats and lints; mypy type-checks; CI runs all three, pre-commit mirrors them locally.
- `# fmt: off` / `# noqa` / `# type: ignore` carry a TODO with owner and date.
- Line length 100, hard; wrap at semantic boundaries, not mid-expression.
- Function-size cap 50 lines, hard; aim 10–25. Signature, blanks, closing count; docstrings don't.
- `__all__` declares every exporting module's surface, sorted, just below imports.
- Imports explicit, sorted, grouped (stdlib / third-party / local); no `from foo import *` outside re-exports.
- Comprehensions/generators/ternaries when they fit (≤2 lines, one `if`); else a `for` loop. No nested ternaries.
- Conditionals: no parens around the condition; `is None` not truthy checks when `None` differs from empty.
- Two blank lines between top-level defs, one between methods; no trailing whitespace; final newline.
- `pre-commit` runs Ruff + mypy with pinned `rev`s; `pyproject.toml` is the single config source.
- Tooling drift is a bug: pin versions, format on save / commit / CI.

### 02 — Naming Conventions

- `snake_case` everything except classes; `PascalCase` for classes and class-shaped types; `SCREAMING_SNAKE_CASE` constants.
- `_name` for internal; `__name` only for genuine name-mangling; never invent dunders.
- Module names `snake_case`, short, singular, no stdlib collisions.
- Booleans read as English: `is_`, `has_`, `should_`; avoid negative-form names.
- Functions are verbs, classes are nouns, properties name state; beware `*Manager`/`*Helper`/`*Util`.
- Single-character names only in tight scope (`i`, `x`, `_`); `user` over `u` otherwise.
- Test names are full sentences in `snake_case`.
- Constants module-level, `SCREAMING_SNAKE_CASE`, `Final`-annotated; no fake constants inside functions.
- No Hungarian/type prefixes, no `IUser`, no `*Async` class suffix.
- Type variables short and meaningful; variance in the name (`T_co`, `T_contra`).
- Service-client classes end in `Client`, only `Client`.
- Resource methods follow the verb taxonomy: `get_` (raises if missing), `list_` (pageable), `create_`, `upsert_`, `update_`, `delete_` (no-ops if missing), `<noun>_exists` (bool), `begin_` (poller).

### 03 — Type Hints

- Type-hint every public signature, return included; public = importable.
- Modern syntax: `|` over `Optional`, `list[X]` over `List[X]`, PEP 695 `type` and inline generics.
- No `Any` in public APIs; replace with `object`, a type variable, or a `Protocol`.
- `Protocol` for behavior contracts; ABCs only when you need shared implementation or runtime `isinstance`.
- `Final` for module constants and non-overridable attributes.
- `Literal` for discriminators and string flags; `StrEnum` for larger closed sets.
- `TypedDict` for JSON-shaped dicts, dataclasses for domain objects.
- Generics (`def f[T](...)`) over `Any` for "the type the caller picks"; constrain when meaningful.
- `Self` for fluent / chained returns and factory methods.
- `from __future__ import annotations` in new modules for forward refs and faster imports.
- No `# type: ignore` without `[error-code]` and a reason; same for `# noqa`; track the count.
- Prefer `cast` (visible claim) over `# type: ignore` (silent); fix the types first, keep both rare.

### 04 — Variables & Declarations

- Never mutable default arguments; default to `None`, materialize inside.
- `tuple`/`frozenset` over `list`/`set` when contents don't change.
- Module constants use `Final` and `SCREAMING_SNAKE_CASE`; no module-level mutable state.
- `ClassVar` marks class attributes so they aren't dataclass fields.
- Walrus `:=` only when it removes a duplicate computation and the binding is reused.
- Annotate locals when the type isn't obvious or widens later (`xs: list[int] = []`); don't over-annotate.
- No module-level side effects on import; declare names, don't perform actions.
- `del` only for genuine unbinding, not as a memory hint.
- `__slots__` (`slots=True`) on value-shaped and hot-path dataclasses.
- One assignment per line; chained assignment only for shared values or unpacking.

### 05 — Functions

- One purpose; 50-line ceiling, aim 10–25; all function kinds count.
- Keyword-only past the obvious positional; booleans always keyword-only.
- Default arguments document the contract; no mutable defaults; sentinel for absent-vs-`None`.
- Type-hint every parameter and the return (`-> None` explicit).
- Decorators explicit, type-preserving (`@functools.wraps`, `ParamSpec`), sparingly stacked.
- `*args`/`**kwargs` only for true forwarding; type them, never `Any`.
- Lambdas only when the call site benefits; never assign a lambda — write a `def`.
- Reach for `functools` (`lru_cache` with bounded `maxsize`, `partial`, `reduce`, `singledispatch`).
- Pure functions where you can; push I/O, logging, mutation to a thin shell.
- `@property` for cheap idempotent reads only — no I/O; `cached_property` for compute-once.
- Generators (`yield`) for streaming / large / unbounded; remember single-pass.
- `async def` is part of the signature; don't mix sync and async unintentionally.

### 06 — Classes & Data Modeling

- `@dataclass(frozen=True, slots=True)` for value-shaped types; no hand-written `__init__`/`__eq__`/`__hash__`.
- `Enum`/`StrEnum`/`IntEnum` for closed sets; no stringly-typed constant clusters.
- `Protocol` over ABC; inheritance only for genuine shared behavior.
- Sum types via `Literal` discriminator + union; exhaustive `match` with `assert_never`.
- No inheritance for code reuse — compose and inject; reject `Base*`/`Abstract*` parents that only share a dependency.
- Constructors store, validate, raise; no I/O; complex creation via classmethod factories.
- `@classmethod` only where `cls` is used; `@staticmethod` rarely — prefer module functions.
- Avoid multiple inheritance; small orthogonal mixins only, reject diamonds.
- Dunders only when the class genuinely *is* that protocol; keep `__eq__`/`__hash__` paired.
- Pick carrier by consumer: `dataclass` default, `NamedTuple` for positional/tuple use, `TypedDict` for dict shapes.
- `@property`/`cached_property` for derived state, plain attributes for stored; no getter/setter pairs.
- Equality and hashing from `frozen=True` or deliberately absent (`eq=False`), never inconsistent.
- Multi-source construction through `@classmethod` factories returning `Self`, not `__init__` with sentinel args.

### 07 — Pythonic Idioms

- `with` for every paired-resource lifecycle; `contextlib.contextmanager` for ad-hoc; `async with` for async.
- Generators / `yield` / `yield from` for streaming; generator expr lazy, list comprehension materialized.
- Comprehensions for transforms, loops for side effects; cap span at 2 lines, no side effects inside.
- EAFP when the success path dominates; catch the specific exception, not `Exception`.
- `pathlib.Path` over `os.path`; convert `str` paths at the boundary.
- f-strings for formatting; lazy `%`-style only for logging.
- `match`/`case` for discriminated unions and structural branching, with `assert_never`; not for a two-way `if`.
- Implement the dunders your class is part of; never for cleverness; keep `__eq__`/`__hash__` paired.
- Decorators for cross-cutting concerns, `@functools.wraps` + `ParamSpec`; no import-time side effects.
- `enumerate`/`zip(strict=True)`/`itertools` over manual indexing; `islice` to bound unbounded sources.
- Truthiness: `is None`/`is not None` explicit; `if items:` only when empty and missing are the same.
- Hint with `collections.abc` (`Iterable`, `Mapping`, `Callable`), not deprecated `typing` aliases.
- Reach for stdlib first (`pathlib`, `dataclasses`, `functools`, `itertools`, `contextlib`, `enum`).

### 08 — Error Handling

- Raise specific exceptions; never bare `raise Exception(...)`.
- Custom domain hierarchies under a single root, carrying context fields; two levels is usually enough.
- Catch only what you can handle; never bare `except:`; `except Exception` only at boundaries that log + re-raise.
- Chain with `raise ... from cause`; `from None` only when the cause is genuinely irrelevant.
- Wrap dependency exceptions into domain types at module boundaries, preserving the cause.
- Never silently swallow; log loudly or use a tight `contextlib.suppress`.
- `if not ...: raise` for input validation (runs in prod); `assert` for internal invariants only (stripped under `-O`).
- `finally` for must-run cleanup that doesn't fit `with`; no control flow in `finally`.
- Re-raise with bare `raise` to preserve the traceback, not `raise e`.
- One top-of-process handler: log with correlation, set exit code, keep serving.
- `Result`-style sealed unions optional per module; one error style per module, transition at the boundary.
- Error messages include the caller's invisible inputs; mask secrets.
- Prefer existing exception types; a new one earns its place only if callers catch it programmatically.
- Don't raise for normal "no" answers; predicates return `bool`, may-miss lookups return `T | None`, failures raise.
- Document raised exceptions in the `Raises:` docstring section.

### 09 — Concurrency & Async

- Default to `asyncio` for new I/O-bound code.
- `asyncio.TaskGroup` over bare `asyncio.gather` for fan-out (structured cancellation + aggregation).
- `asyncio.timeout` over `wait_for`; every external async call has a documented deadline.
- Never `create_task` and drop the reference; hold it or use a `TaskGroup`.
- Cancellation is cooperative; re-raise `CancelledError` if you catch it; yield in long CPU sections.
- `asyncio.Lock`/`Semaphore`/`Queue(maxsize=N)` for coroutine-safe sync; no `threading` primitives in `async def`.
- `asyncio.to_thread` (bounded executor) for unavoidable blocking calls.
- Threading only when forced (GIL-bound, I/O); multiprocessing only when CPU-bound; justify against the decision tree.
- `concurrent.futures` with explicit `max_workers`, context-managed.
- Producer-consumer with backpressure via bounded `Queue`; document `maxsize`.
- `contextvars.ContextVar` for async-safe request context, not `threading.local`.
- `asyncio.run` at the program entry point only, never in library code.
- Separate sync and async clients by module path (`acme.x` vs `acme.x.aio`); never `def`/`async def` on one class.
- Async uses `async`/`await`; no `@asyncio.coroutine` or `yield from`-based coroutines.

### 10 — API Design

- `__all__` declares every published module's public surface.
- Keyword-only for public functions past two positionals; booleans always keyword-only.
- `Protocol` for dependency-inversion seams, not concrete classes.
- Generics for "the type the caller picks"; don't reach for them prematurely.
- Defaults document the contract; no mutable defaults; new optionals added keyword-only.
- Type-hint the whole public surface; ship `py.typed`.
- Deprecate with `warnings.warn(..., DeprecationWarning, stacklevel=2)`; name the removal release.
- Follow semver; breaking changes need a major bump and a documented migration.
- Iterators/generators lazy by default at the boundary; spell the pass/materialization contract in the return type.
- `async def` is part of the contract; don't expose sync+async of the same name unless both are first-class.
- Don't expose mutable internals; return `tuple`/`Mapping`/`MappingProxyType`.
- Pipeline composition via a bounded `list[Step]` fold of typed callables.
- Client constructor: ≤2 positionals (binding target + credential), then `*`, then keyword-only optionals.
- No options-bag objects on methods; per-call options are individual keyword-only args (`**kwargs` to policies is fine).
- Clients immutable after construction; per-call kwarg names mirror the constructor's.
- `list_*` returns a pageable iterator, never `None`, never an unbounded eager list.
- Long-running operations: `begin_*` prefix returning a poller.
- Conditional requests via `etag` + `match_condition` keyword-only pair.
- Accept `Model | Mapping[str, Any]` for input; normalize the `Mapping` path through the validator.
- Validate client parameters at the constructor; don't re-validate service-parameter semantics.
- Identify the library in `User-Agent`; emit OpenTelemetry spans for every I/O method.

### 11 — Testing

- `pytest`, not `unittest`; tests as top-level functions.
- Test names are full sentences in `snake_case`; files `tests/test_*.py`.
- Arrange / act / assert, blank-line separated; one concern per test.
- `@pytest.mark.parametrize` with `ids=` for table-driven cases; no copy-pasted near-identical tests.
- Fixtures over `setUp`/`tearDown`, scoped deliberately; yield-fixtures for cleanup; share via `conftest.py`.
- Tests independent — any order, parallel (`pytest -n auto`); no shared mutable state.
- Use real implementations; mock only at the genuine external boundary.
- Hand-rolled Protocol-conforming fakes for domain types; mocks for external I/O.
- Inject time / randomness / IDs (`Clock`, seeded random), never pull from the wild.
- `hypothesis` property tests for round-trip / idempotence / monotonicity invariants.
- Async tests via `pytest-asyncio` (`asyncio_mode = "auto"`); no real sleeps.
- No `time.sleep` in tests; poll, fake the clock, or use async test machinery.
- Assertion density mirrors production: 2+ per test on average; assert positive and negative space.
- Coverage is a floor (~80%), not a ceiling; test boundaries, edge cases, error paths, regressions.

### 12 — Package Organization

- `src/` layout for any project that publishes a package; tests run against the installed copy.
- `pyproject.toml` is the single config source; one build backend; apps lock, libraries declare bounds.
- `__init__.py` re-exports the public API via `__all__`; callers don't import internal paths.
- Group by feature, not technical layer; minimal shared `common`/`shared`.
- Modules flat where possible — aim two levels under the root; split ~500-line modules.
- No cyclic imports, ever; resolve by extracting a shared module or reworking the design (`import-linter`).
- Public vs private via underscore convention + `__all__`; `_internal`/`_generated` visible in the tree.
- Tests in `tests/`, mirroring `src/`; integration tests behind a marker.
- Generated code in its own `_generated` path, regenerated from config, never hand-edited.
- Namespace packages (PEP 420) only for plugin/extension architectures.
- README per package, docstring per module; describe what it does, not the name.
- One dependency tool per project (`uv`/`poetry`/`pdm`); pin Python with `.python-version`.

### 13 — Resource Management

- `with` on every paired resource; don't return open handles from functions.
- `contextlib.contextmanager` for ad-hoc managers, always `try/finally` around the `yield`.
- Async resources use `async with`; don't mix sync `with` on async-capable resources.
- `asyncio.TaskGroup` owns task lifecycles; long-lived tasks keep a strong reference + cleanup callback.
- `asyncio.timeout` is the default bound for I/O; pick the value from the SLA.
- `secrets`/`os.urandom` for cryptographic randomness; `secrets.compare_digest` for token comparison; `random` for the rest.
- Connection pools bounded and explicit (`max_size`/`max_connections`) from a named constant, with timeouts.
- Graceful shutdown: stop accepting → drain in-flight (bounded) → close pools → exit.
- `del`/`__del__` are not resource management; use context managers; `weakref.finalize` only for non-critical cleanup.
- Bound everything: `islice`, `Queue(maxsize=N)`, `Semaphore(N)`, `lru_cache(maxsize=N)`, `timeout`, pool `max_size`.
- `tempfile.NamedTemporaryFile`/`TemporaryDirectory` for ephemeral files; pair `delete=False` with a `finally` unlink.
- `subprocess.run(timeout=, check=True, capture_output=True)`, `shell=False`, args as a list.

### 14 — Documentation

- Every public function, class, and module has a docstring (Ruff `D` rules).
- Google-style docstrings (`convention = "google"`).
- Docstrings explain *why* and *when*, not *what*; delete one that only restates the signature.
- One-line imperative summary ending in a period, blank line, then details.
- Examples as an `Example` block or doctest; `pytest --doctest-modules` runs them.
- Document raised exceptions under `Raises:`, most-likely first.
- Type hints document data, docstrings document behavior; don't restate the type — add ranges, units, formats.
- Module docstring at the top of every module.
- Class docstrings describe purpose; document non-obvious `Attributes`; one home for `__init__` params.
- Deprecation note in the docstring paired with `warnings.warn(..., DeprecationWarning)`.
- In-body comments explain *why*; TODOs carry owner + date; no commented-out code.
- Generated docs (`pdoc` or `mkdocs` + `mkdocstrings`) build in CI; a broken build fails the PR.

### 15 — Performance

- Design for the slowest resource first: network > disk > memory > CPU.
- Profile before optimizing (`py-spy`, `cProfile`, `memray`, `scalene`); a perf PR carries before/after profiles.
- `__slots__` / `@dataclass(slots=True)` for hot-path classes.
- Generators over lists for large or streaming data; `itertools` is lazy.
- `functools.lru_cache` with bounded `maxsize`, pure functions only; inspect with `cache_info()`.
- Avoid global state; encapsulate and inject; module-level mutable singletons are a smell.
- `"".join(parts)` and f-strings; no `+=` string-building in loops.
- `dict`/`set`/`frozenset` for O(1) lookup; never repeated `in` over a list.
- `asyncio` is cheap, not free; batch fan-out into chunks (`itertools.batched`), don't over-spawn.
- GIL: threads for I/O parallelism, `multiprocessing` or native extensions for CPU.
- Native extensions (numpy/polars/pyarrow/Cython) for hot paths on profile evidence, not speculation.
- Defer heavy imports for cold-start-sensitive entrypoints (CLI, serverless).
- Hot-path value types are stdlib `dataclass(slots=True, frozen=True)`, not `attrs`/`pydantic`.
