## Python Code Style

Binding style rules for Python projects, target **Python 3.12+** (with 3.13 features noted where relevant). Prioritize **correctness**, **explicitness**, **simplicity** — never cleverness, never abstraction for its own sake.

This guide extends and defers to:

- **[PEP 8](https://peps.python.org/pep-0008/)** — Python's canonical style guide.
- **[PEP 20 (The Zen of Python)](https://peps.python.org/pep-0020/)** — the worldview ("Explicit is better than implicit. Simple is better than complex. Errors should never pass silently.").
- **[PEP 484](https://peps.python.org/pep-0484/) + [PEP 604](https://peps.python.org/pep-0604/) + [PEP 695](https://peps.python.org/pep-0695/)** — type hints, union syntax, modern generics.
- **[Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)** — docstring conventions and additional taste.

Where our guidance conflicts with PEP 8 or PEP 20, **the PEPs win.** This guide adds project-specific conventions: ruthless type hints, Tiger-style discipline (50-line function cap, assertion density, bounded loops), structured concurrency via `asyncio.TaskGroup`, and a strong preference for `@dataclass(frozen=True, slots=True)` + `Protocol` over inheritance.

### Style Priorities

1. **Clarity** — code's purpose is clear to the reader.
2. **Simplicity** — the simplest approach that accomplishes the goal.
3. **Concision** — high signal-to-noise ratio.
4. **Maintainability** — easy to modify correctly over time.
5. **Consistency** — matches the surrounding codebase.

Resolve rule conflicts in this order. Consistency is the tiebreaker, never an override.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Formatting & Tooling](./01-formatting-and-tooling.md) | Ruff (lint + format), mypy strict, line length, function-size cap, pre-commit |
| 02 | [Naming Conventions](./02-naming-conventions.md) | `snake_case`, `PascalCase`, `_private`, `__dunder`, module names |
| 03 | [Type Hints](./03-type-hints.md) | Type every public signature, `\|` over `Optional`, `Protocol` for structural typing, no `Any` in public API |
| 04 | [Variables & Declarations](./04-variables-and-declarations.md) | Mutable default args, `Final`, `ClassVar`, walrus, module-level constants |
| 05 | [Functions](./05-functions.md) | Keyword-only args, default args, decorators, `functools`, single-purpose, 50-line cap |
| 06 | [Classes & Data Modeling](./06-classes-and-data-modeling.md) | `@dataclass(frozen=True, slots=True)`, `Protocol`, ABCs sparingly, `enum`, no inheritance for reuse |
| 07 | [Pythonic Idioms](./07-pythonic-idioms.md) | Context managers, generators, comprehensions, EAFP, dunders, `pathlib`, f-strings, `match`/`case` |
| 08 | [Error Handling](./08-error-handling.md) | Custom exception hierarchies, exception chaining, no bare `except`, `contextlib.suppress`, fail fast |
| 09 | [Concurrency & Async](./09-concurrency.md) | `asyncio.TaskGroup`, `asyncio.timeout`, cancellation, `threading` only when forced, GIL realities |
| 10 | [API Design](./10-api-design.md) | `Protocol` interfaces, keyword-only public APIs, `__all__`, deprecation, semver discipline |
| 11 | [Testing](./11-testing.md) | pytest, fixtures, `@pytest.mark.parametrize`, hypothesis, no shared state, `pytest-asyncio` |
| 12 | [Package Organization](./12-package-organization.md) | `src/` layout, `pyproject.toml`, `__init__.py`, no cyclic imports |
| 13 | [Resource Management](./13-resource-management.md) | `with`/`async with`, `contextlib`, `asyncio` cancellation, timeouts, `secrets`/`os.urandom` |
| 14 | [Documentation](./14-documentation.md) | Google-style docstrings, examples, type hints document what docstrings don't |
| 15 | [Performance](./15-performance.md) | Profile first, generators, `__slots__`, `functools.lru_cache`, asyncio cost, GIL, no premature opt |

---

## Cross-Cutting Concerns

Security, performance, and git practices are covered in the [root-level code style guide](../README.md). The cross-cutting docs are language-agnostic; this guide adapts them to Python.

## Design Goals

**Correctness > performance > developer experience.** When they conflict, this ordering decides.

## Philosophy

Python's design ("There should be one — and preferably only one — obvious way to do it") is closer to our principles than most languages. The Zen of Python *is* most of this guide. The rest is discipline overlays that the language doesn't enforce but a serious codebase needs.

1. **Lean on dataclasses + Protocols. Avoid inheritance for code reuse.** Python uses classes well — and we use them. `@dataclass(frozen=True, slots=True)` for state, `Protocol` for behavior contracts, plain functions and methods for transformations. Inheritance is reserved for shared *interface* (and even there, `Protocol` usually wins), not for sharing implementation. Composition + duck typing + Protocols cover the rest.
2. **Types are part of the contract.** Type-hint every public signature. Run mypy in strict mode. No `Any` in public API. Use `Protocol` for structural typing — Python's duck typing made type-safe.
3. **Immutable by default.** `frozen=True` on dataclasses. `tuple` over `list` when the contents don't change. No mutable default arguments. Mutability is an explicit choice you have to type — that's the right way around.
4. **Explicit over implicit.** No global state. No module-level side effects in imports. No `from foo import *` outside of explicit re-exports. Every dependency in the function signature or constructor. Magic is for libraries; applications are explicit.
5. **Exceptions are the mechanism — handle every one explicitly.** Python's idiom is exceptions, and we use them. Custom hierarchies for the domain. No bare `except`. No silent `pass`. Wrap with `raise NewError(...) from cause` so debugging context survives the boundary.
6. **Async is structured.** `asyncio.TaskGroup` for parallel work, `asyncio.timeout` for bounds. Never bare `asyncio.gather` without explicit error semantics. Never `asyncio.create_task` and drop the reference.
7. **Small functions, breathing room.** Hard limit: 50 lines. Aim 10–25. Python lacks braces and types in function bodies, so vertical density tends to be high — even short functions carry a lot of logic. Cap aggressively to force decomposition; lean on private helpers and comprehensions.
8. **Validate at every public boundary; assert internal invariants.** `if not ...: raise ValueError(...)` for input validation — this runs in production. `assert` for internal invariants, never for security or contract checks (`assert` is stripped under `python -O`). Aim for two checks per function on average. This density is a project discipline overlay, not native Pythonic practice — Python culture is "validate at the boundary, trust internally"; we tighten that to "validate at every public boundary, split compound checks, fail fast with a clear message."
9. **Bound everything.** All loops, retries, queues, timeouts, async tasks. No unbounded `while True`. No recursion in library code where iteration works. Use `itertools.islice` to bound lazy iterators when feeding them into APIs.
10. **Use `match`/`case` for sealed sets; tag with a discriminator field.** Exhaustive `match` over a tagged union (sum types modeled with `Literal` discriminators or sealed-class-style hierarchies) makes refactors visible — mypy flags missing cases.
11. **Embrace Pythonic idioms — but deliberately.** Context managers, generators, comprehensions, EAFP, decorators, dunders. Each has a *right* use. Reaching for the clever one when the boring one fits is anti-Zen.
12. **`Protocol` over ABC. Composition over inheritance.** A `Protocol` documents required behavior without forcing inheritance. An ABC forces a base class. Pick Protocol unless you genuinely need the base class for shared implementation.
13. **Performance from the outset, but pay for what you use.** `__slots__` and `frozen=True` on hot dataclasses. Generators for large/streaming data. `functools.lru_cache` for pure-function memoization. Don't pre-optimize without a profile.
14. **Zero technical debt.** Public API is a contract — Python's lack of enforcement is not a license to break it. `__all__` declares the surface. Semver is a promise.

## Deviations from Upstream

This guide takes PEP 8 + PEP 20 as canonical, with PEP 484/604/695 governing types. The first entry below is a genuine softening — a place this guide deliberately relaxes the root canon to stay true to Python culture; the rest are additions the PEPs do not address (the function-size cap, and authorities layered on top of the base PEP set). Each is recorded so it can be revisited surgically.

| Rule | Upstream position | Our position | Why |
|---|---|---|---|
| Assertion density | Root canon: "assert aggressively," 2+ per function, no caveat | Same target, but reframed as a project discipline overlay — not native Pythonic practice | Python culture is "validate at the boundary, trust internally"; an aggressive-assertion mandate reads as un-Pythonic if presented as native. We keep the density but say plainly it is an overlay the language doesn't ask for: validate at every public boundary, split compound checks, fail fast. See rule 8 above and [08-error-handling.md](./08-error-handling.md). |
| Function size | No upstream cap (PEP 8 is silent) | 50-line hard cap, aim 10–25 | Owner decision; Tiger Style discipline, the tightest of the spine languages because Python bodies lack braces and type annotations, so vertical density runs high. See [05-functions.md](./05-functions.md) and [01-formatting-and-tooling.md](./01-formatting-and-tooling.md). |
| Added authorities | Root table names PEP 8 + PEP 20 + PEP 484/604 | Adds the Google Python Style Guide (docstrings, module structure) and PEP 695 (modern generics) on top | The base PEPs are silent on docstring shape and predate `type`-statement generics; the additions supply taste and modern syntax the PEPs leave open, and never override them. See [14-documentation.md](./14-documentation.md) and [03-type-hints.md](./03-type-hints.md). |

## Influences

- **[PEP 8](https://peps.python.org/pep-0008/), [PEP 20](https://peps.python.org/pep-0020/)** — canonical Python.
- **[Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)** — Google's adaptation; useful supplements on docstrings and module structure.
- **[Effective Python (Brett Slatkin)](https://effectivepython.com/)** — community canon for idiomatic patterns.
- **[Hypermodern Python (Claudio Jolowicz)](https://cjolowicz.github.io/posts/hypermodern-python-01-setup/)** — modern tooling baseline (Ruff, mypy, pytest, pre-commit, pyproject.toml).
- **[Trio's structured concurrency](https://trio.readthedocs.io/en/stable/reference-core.html)** — informs `asyncio.TaskGroup` patterns even though we use stdlib asyncio.
- **[Azure SDK for Python Design Guidelines](https://azure.github.io/azure-sdk/python_design.html)** — prescriptive, battle-tested guidance on building Python SDKs: client class shape, constructor signature, method verb taxonomy (`get_*`/`list_*`/`create_*`/`begin_*`), sync+async separation via `.aio` submodule, pageable iterators, long-running-operation pollers, conditional-request kwargs, retries/transport/credential injection. Adopted into chapters 02, 06, 08, 09, 10.
- **TigerBeetle Tiger Style** — assertion density, 50-line function limit, limits on everything, no recursion, zero technical debt.

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **module / package level or larger** — never mix two styles within the same module. A half-migrated module is more confusing than either end state.

Perfection over technical debt — debt never gets paid
