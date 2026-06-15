# 12 — Package Organization

How code is grouped, named, and exposed across boundaries. Python's import system is flexible — and the rules below are what keep that flexibility from becoming chaos.

## What good looks like

```text
project/
├── pyproject.toml          # single source of config: build, deps, tools (12.2)
├── .python-version         # pinned interpreter, not system Python (12.12)
├── uv.lock                 # application locks; libraries declare bounds (12.2)
├── src/
│   └── shop/
│       ├── __init__.py     # re-exports the public surface via __all__ (12.3)
│       ├── checkout/       # grouped by feature, two levels deep (12.4, 12.5)
│       │   ├── __init__.py
│       │   ├── endpoints.py
│       │   └── orders.py
│       ├── _internal/      # private; named so it shows in the tree (12.7)
│       │   └── tokens.py
│       └── _generated/     # protobuf/OpenAPI output, never hand-edited (12.9)
│           └── api_pb2.py
└── tests/
    ├── conftest.py         # shared fixtures (12.8)
    └── checkout/
        └── test_orders.py  # mirrors src/shop/checkout/orders.py (12.8)
```

The `src/` layout keeps the in-tree package off the import path so tests run against the installed copy (12.1), with `pyproject.toml` and `.python-version` holding all config (12.2, 12.12). Code groups by feature and stays flat — `checkout/` owns its endpoints and orders rather than scattering across layer packages (12.4, 12.5). `_internal/` and `_generated/` name their privacy in the tree (12.7, 12.9), the package's `__init__.py` re-exports the public surface through `__all__` (12.3), and `tests/` mirrors `src/` module for module (12.8).

## Rules

### 12.1 — `src/` layout for every project that publishes a package.

**Reasoning, step by step:**
1. The `src/` layout puts your package under `src/<package_name>/` instead of at the repo root. The layout prevents accidental imports of the in-source package from the project directory.
2. Without `src/`, running tests from the repo root imports your local code (un-built). With `src/`, tests run against the installed package — exactly what users get.
3. **Pattern:**
   ```
   project/
   ├── pyproject.toml
   ├── src/
   │   └── mypackage/
   │       ├── __init__.py
   │       └── ...
   └── tests/
       └── test_*.py
   ```
4. Pure applications (not published as packages) can skip `src/`, but the discipline still pays off when extracting modules later.

**Enforcement:** review; package directory lives under `src/`, and CI runs tests against the installed package, not the repo root.

### 12.2 — `pyproject.toml` is the single source of project configuration.

**Reasoning, step by step:**
1. PEP 518 + PEP 621: `pyproject.toml` holds build system, project metadata, dependencies, and tool config.
2. Don't scatter `setup.cfg`, `setup.py`, `tox.ini`, `.flake8`, `.mypy.ini`. Consolidate.
3. Build backend: `hatchling` (modern, minimal), `setuptools` (legacy default, fine), `poetry-core` (if you use Poetry), `pdm-backend` (PDM). Pick one and stick with it.
4. Dependencies declared with version constraints (`requests>=2.31,<3`). Lock for applications (`uv.lock`, `poetry.lock`, `pip-compile` output). Libraries do not lock — they declare bounds.

**Enforcement:** review; no `setup.py`/`setup.cfg`/`tox.ini`/`.flake8` present, config consolidated in `pyproject.toml`, lockfile committed for applications.

### 12.3 — `__init__.py` is the package's public API.

**Reasoning, step by step:**
1. Re-export from `__init__.py` only the symbols you want as the package's public surface. Use `__all__`.
2. **Pattern:**
   ```python
   # src/mypackage/__init__.py
   from mypackage.user import User, UserId, UserNotFound
   from mypackage.payments import Payment, charge

   __all__ = ["User", "UserId", "UserNotFound", "Payment", "charge"]
   ```
3. Without re-export, callers import internal paths (`from mypackage.user.model import User`) — which then become breaking changes when you reorganize internally.
4. Empty `__init__.py` files create implicit "all submodules are equal" behavior — fine for tests and small packages, not for libraries.

**Enforcement:** review; library `__init__.py` declares `__all__`, and callers import the re-exported surface rather than internal module paths.

### 12.4 — Group by feature, not by technical layer.

**Reasoning, step by step:**
1. `mypackage/checkout/` (api, domain, storage) keeps a feature's code together.
2. `mypackage/controllers/`, `mypackage/services/`, `mypackage/repositories/` scatters each feature across three packages.
3. Feature-shaped layout localizes change: a new checkout requirement modifies the `checkout` package and nothing else.
4. Cross-feature shared code lives in a sibling `common`/`shared` module. Keep it minimal — every entry pulls every feature toward it.

**Enforcement:** review; `import-linter` layer rules forbid feature-spanning `controllers`/`services`/`repositories` packages.

### 12.5 — Modules are flat where possible. Deep nesting is a smell.

**Reasoning, step by step:**
1. `mypackage.checkout.api.endpoints.user_lookup` is hard to import, hard to remember, and signals over-organization.
2. Aim: two levels under the package root (`mypackage.checkout.endpoints`).
3. Splits happen when (a) a module exceeds ~500 lines, (b) it has two unrelated responsibilities, (c) circular import pressure forces extraction.
4. Flat structure with `__all__` discipline beats deep structure with no surface contract.

**Enforcement:** review; nesting beyond two levels under the package root is flagged, and ~500-line modules trigger a split.

### 12.6 — No cyclic imports. Ever.

**Reasoning, step by step:**
1. Module A imports from B. B must not import from A — directly or transitively.
2. Cycles produce `ImportError` at startup, or work-by-accident due to import-time-only evaluation, or break under test reordering.
3. **Resolve cycles by:** extracting the shared abstraction into a third module both depend on; moving the import to inside a function (last resort, document why); reworking the design.
4. Tooling: `import-linter` enforces architectural constraints. Define "layer" rules and fail builds on violation.

**Enforcement:** `import-linter` independence/layer contracts fail the build on any cycle.

### 12.7 — Public vs private: underscore convention + `__all__`.

**Reasoning, step by step:**
1. Leading underscore on a name = "internal, don't import this." Not enforced — convention.
2. `__all__` defines what `from module import *` exposes and documents the public surface.
3. Together: anything not in `__all__` and not exposed via `__init__.py` re-export is private to the package, regardless of underscore.
4. Internal modules can also be named `_internal.py` or grouped under `_internal/` — visible in the file tree.

**Enforcement:** review; `__all__` present on public modules, and underscore-prefixed names stay out of `__init__.py` re-exports.

### 12.8 — Tests live in `tests/`, mirror production structure.

**Reasoning, step by step:**
1. `src/mypackage/checkout/orders.py` is tested by `tests/checkout/test_orders.py`.
2. Same structure makes the test for a given module obvious.
3. `tests/conftest.py` for shared fixtures (and per-subdirectory `conftest.py` for scoped fixtures).
4. Integration tests in `tests/integration/`, separated by pytest marker (`@pytest.mark.integration`). Keep them out of the default test run unless they're fast.

**Enforcement:** review; `tests/` mirrors `src/` path for path, and the default `pytest` run deselects the `integration` marker.

### 12.9 — Generated code lives in its own module path and isn't edited.

**Reasoning, step by step:**
1. Generated code (Protobuf, OpenAPI, GraphQL clients) goes in a clearly-named generated module: `mypackage._generated/`.
2. Never hand-edit. Change the generator config; regenerate.
3. Check generated output in or not in source control depending on generation cost. If it's slow, check it in. If it's fast, generate on every build.

**Enforcement:** review; generated output isolated under a `_generated` path, regenerated from config rather than hand-edited.

### 12.10 — Namespace packages (PEP 420) only for plugin/extension architectures.

**Reasoning, step by step:**
1. PEP 420 lets multiple distributions contribute submodules under the same top-level name (`mypackage.plugin_foo`, `mypackage.plugin_bar`, each from a different package).
2. Useful for: plugin architectures where third parties extend your namespace.
3. **Not useful for:** normal packages. Stick with regular packages (`__init__.py` present) — fewer surprises.

**Enforcement:** review; namespace packages permitted only for declared plugin namespaces, otherwise `__init__.py` is present.

### 12.11 — Module documentation: README per package, docstrings per module.

**Reasoning, step by step:**
1. Package root: a README explaining purpose, entry points, and audience.
2. Each module starts with a docstring describing its responsibility:
   ```python
   """Payment processing: card tokenization, gateway dispatch, receipt construction."""
   ```
3. Don't repeat the module name in the docstring. Describe what it *does*.
4. KDoc-equivalent for Python is the module + function + class docstring. See chapter 14.

**Enforcement:** `ruff` D100 (module docstring) plus review for a per-package README.

### 12.12 — Dependency management: `uv` or `poetry` or `pdm`. Pick one per project.

**Reasoning, step by step:**
1. `uv` (Astral, Rust-based) — modern, fast, replaces `pip`/`venv`/`pip-tools` for many workflows. Recommended for new projects.
2. `poetry` — established, opinionated, has its own dependency resolver.
3. `pdm` — PEP-compliant, supports PEP 582 (`__pypackages__`).
4. Pick one per project. Mixing breaks lockfile and venv conventions.
5. Pin Python version with `.python-version` (works with `pyenv`, `uv`, `pdm`). Don't depend on the developer's system Python.

**Enforcement:** review; one tool's lockfile and one `.python-version` committed per project, no mixed lockfiles.

## Cross-references

- `__all__` and visibility: chapter 01, chapter 10.
- Tests and fixtures location: chapter 11.
- Generated code (e.g., from OpenAPI): chapter 06 (data modeling) for the dataclass/TypedDict split at JSON boundaries.
