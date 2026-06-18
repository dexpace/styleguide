---
name: python-styleguide
description: Use when writing, editing, or reviewing Python (.py) source in a dexpace project — enforces the dexpace Python styleguide (full type hints, dataclasses + Protocols, structured asyncio, 50-line function cap). Also use before committing Python, or when asked to review Python against the styleguide.
---

# Python styleguide

Extends PEP 8 / PEP 20 / PEP 484+604 / PEP 695 plus the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html); where they conflict, the PEPs win, except the deviations below. Target Python 3.12+.

## When this applies

Editing `*.py`, or reviewing Python. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data + functions: `@dataclass` for state, plain functions and `Protocol` for behavior. No inheritance for code reuse, no deep hierarchies.
2. Explicit over implicit: every dependency in the signature or constructor, every error raised. No module-level side effects on import. Library options follow documented defaults.
3. Immutable by default: `frozen=True` dataclasses, `tuple`/`frozenset` over mutable collections, no mutable default arguments.
4. Errors are values, handled explicitly: custom exception hierarchies, no bare `except:`, no silent `pass`. Chain with `raise ... from cause`.
5. Composition over inheritance: `Protocol` seams and mixins where reuse is needed without an "is-a"; ABCs only for genuine shared implementation.
6. Transform, don't mutate: pure helpers in, new output out; push I/O and state changes to the boundary.
7. Always say why: comments and docstrings explain reasoning, not mechanics.
8. Assert aggressively: ≥2 checks per function on average; validate at every public boundary, split compound checks, fail fast.
9. Limits on everything: bound every loop, queue, retry, timeout, task list, cache. No unbounded `while True`, no recursion where iteration works.
10. Small functions: one thing each; **50-line hard cap, aim 10–25**; blank lines between logical sections.
11. Performance from the outset: optimize the slowest resource first (network > disk > memory > CPU); profile before tuning.
12. Zero technical debt: `__all__` declares the surface, semver is a promise, do it right the first time.

## Language hard rules

- Type every public signature, return included; `|` over `Optional`, `list[X]` over `List[X]`, PEP 695 `type` aliases and `def f[T](...)` generics. No `Any` in public APIs — use `object`, a `Protocol`, or a type variable. mypy strict is the gate.
- `@dataclass(frozen=True, slots=True)` value types plus `Protocol` contracts; no inheritance for reuse. Validate invariants in `__post_init__`, multi-source construction through `@classmethod` factories returning `Self`.
- Model sealed sets as `Literal`-discriminated unions or `StrEnum`; `match` over them stays exhaustive with `assert_never` so mypy flags a missing variant.
- Errors: raise specific exceptions, never bare `Exception`; custom domain hierarchies; chain with `raise ... from`; `contextlib.suppress` for the deliberate ignore; `assert` for internal invariants only (stripped under `-O`), `if not ...: raise` for input validation. Don't return `None`/`bool` to signal failure.
- **Assertion density is an explicit overlay** the language doesn't ask for — Python culture is "validate at the boundary, trust internally". Keep the density: validate at every public boundary, split compound checks, fail fast with a clear message.
- Structured concurrency: `asyncio.TaskGroup` over bare `gather`, `asyncio.timeout` on every external call, hold task references, re-raise `CancelledError`. `threading`/`multiprocessing` only when forced, justified against the decision tree.
- Idioms used deliberately: context managers, generators, comprehensions (≤2 lines, one `if`), EAFP, `pathlib`, f-strings, `match`/`case`, `collections.abc` hints.
- Resources: `with`/`async with` on every paired lifecycle, never `__del__` for cleanup; bounded pools and queues; `secrets`/`os.urandom` for cryptographic randomness; timeouts on external I/O.
- Ruff is lint + format; mypy strict; `pyproject.toml` is the single source of config.

## Before you finish — verify

```bash
ruff format .       # then check it stuck
ruff check .        # lint clean
mypy --strict .     # or the project's mypy config
pytest              # pytest, not unittest
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/python/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/python/01-formatting-and-tooling.md)
- [02 Naming Conventions](https://github.com/dexpace/styleguide/blob/main/python/02-naming-conventions.md)
- [03 Type Hints](https://github.com/dexpace/styleguide/blob/main/python/03-type-hints.md)
- [04 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/python/04-variables-and-declarations.md)
- [05 Functions](https://github.com/dexpace/styleguide/blob/main/python/05-functions.md)
- [06 Classes & Data Modeling](https://github.com/dexpace/styleguide/blob/main/python/06-classes-and-data-modeling.md)
- [07 Pythonic Idioms](https://github.com/dexpace/styleguide/blob/main/python/07-pythonic-idioms.md)
- [08 Error Handling](https://github.com/dexpace/styleguide/blob/main/python/08-error-handling.md)
- [09 Concurrency & Async](https://github.com/dexpace/styleguide/blob/main/python/09-concurrency.md)
- [10 API Design](https://github.com/dexpace/styleguide/blob/main/python/10-api-design.md)
- [11 Testing](https://github.com/dexpace/styleguide/blob/main/python/11-testing.md)
- [12 Package Organization](https://github.com/dexpace/styleguide/blob/main/python/12-package-organization.md)
- [13 Resource Management](https://github.com/dexpace/styleguide/blob/main/python/13-resource-management.md)
- [14 Documentation](https://github.com/dexpace/styleguide/blob/main/python/14-documentation.md)
- [15 Performance](https://github.com/dexpace/styleguide/blob/main/python/15-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
