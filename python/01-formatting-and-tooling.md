# 01 — Formatting & Tooling

Formatting is **not** a matter of taste. We delegate to tools and disagree elsewhere.

## Rules

### 1.1 — Ruff is the formatter and the linter. mypy is the type checker. CI enforces all three.

**Reasoning, step by step:**
1. Argument over whitespace is a tax on every PR. A deterministic formatter ends the discussion.
2. Ruff (modern, fast, written in Rust) replaces Black + isort + flake8 + pyupgrade + pylint-subset in one tool. It's the new default for serious Python projects.
3. mypy (or pyright) checks types — Python's optional, gradual type system is only as good as the checker that enforces it.
4. All three run in CI on every PR. They also run pre-commit on every developer machine.
5. Format-disable comments (`# fmt: off`, `# noqa`, `# type: ignore`) need a TODO with an owner and a date. Tracked.

**Minimum `pyproject.toml`:**
```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "RUF", "ANN", "ASYNC", "S"]
ignore = ["ANN101", "ANN102"]  # self/cls type hints

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_ignores = true
```

### 1.2 — Line length is 100 columns. Hard.

**Reasoning, step by step:**
1. PEP 8 says 79; Black's default is 88; Google says 80. We pick 100 because typed Python signatures, dataclass fields, and Protocol method signatures genuinely run wider than untyped Python.
2. 100 fits side-by-side diffs at modern resolutions. Wider lines break review.
3. Wrap at semantic boundaries — between arguments, after `=`, between chained method calls — not mid-expression.
4. If a line still wraps after argument-wrapping, the function is doing too much.

### 1.3 — Trailing commas in every multi-line collection, argument list, and parameter list.

**Reasoning, step by step:**
1. Trailing commas reduce diff noise — adding an element touches one line, not two.
2. They make argument-order swaps a one-line change.
3. Ruff enforces this; configure it.
4. Required especially in `__all__`, function calls with named args, and tuple literals where the comma is sometimes semantically required (`(item,)` for single-element tuple).

### 1.4 — Function-size cap: 50 lines, hard. Aim 10–25.

**Reasoning, step by step:**
1. A function you can't see at once on one screen costs you context every time you read it.
2. Python is more concise per line than Kotlin or Go (no braces, no semicolons, no types in body), so we go tighter than Kotlin's 60.
3. Counting: signature, blank lines, and closing — all count. Docstrings do not.
4. Approaching 50? Extract a private helper, lift a comprehension out, or split into a class with smaller methods.
5. The cap is a *signal*, not a target. Don't compress for the number.

### 1.5 — `__all__` declares the public surface of every module that exports anything.

**Reasoning, step by step:**
1. Python has no real visibility modifier. `__all__` is the convention: a list of names that `from module import *` will expose and that documents your public contract.
2. Without `__all__`, every top-level name is "public" by default — refactoring is then a guessing game about what callers depend on.
3. Place `__all__` at the top of the file, just below imports:
   ```python
   __all__ = ["UserId", "load_user", "User"]
   ```
4. Module-level helpers not in `__all__` are conventionally `_private` (leading underscore).

### 1.6 — Imports: explicit, sorted, grouped. Ruff handles the order.

**Reasoning, step by step:**
1. Ruff (with `isort` rules) sorts imports into three groups: stdlib, third-party, local. Within each group, alphabetical.
2. `from foo import *` is banned outside `__init__.py` re-exports and unless the module deliberately defines `__all__`.
3. Relative imports (`from .utils import x`) only within a package, only when they make the dependency clearer than absolute. In application code, prefer absolute.
4. Conditional imports (inside functions) for genuinely circular or optional-dependency cases only. Document why.

### 1.7 — Expression-oriented Python: comprehensions, generator expressions, ternaries — when they fit.

**Reasoning, step by step:**
1. `[x.id for x in items if x.active]` is clearer than a `for` loop with `append`. Use the comprehension.
2. The comprehension stops being clearer when (a) it spans more than 2 lines, (b) it has more than one `if`, (c) the inner expression is non-trivial.
3. At that point, write a `for` loop. Readability wins over Python golf.
4. Ternary `a if cond else b` is fine for one-liners. Nested ternaries are not — write an `if/elif/else`.

### 1.8 — Conditionals: no parentheses around the condition; no truthy-checks on `None` / empty sequences when explicit is clearer.

**Reasoning, step by step:**
1. `if user is None:` is unambiguous. `if not user:` is true for `None`, empty string, `0`, empty list — usually not what you meant.
2. `if items:` is fine for "the list is non-empty." `if items is not None:` is what you need if `None` is a different signal than empty.
3. PEP 8: explicit comparisons against `None` use `is`/`is not`, never `==`/`!=`.

### 1.9 — Blank lines as logical separators.

**Reasoning, step by step:**
1. Two blank lines between top-level functions and classes.
2. One blank line between methods.
3. One blank line inside a function to separate logical sections (rare — if you need many, split the function).
4. No trailing whitespace.
5. Final newline at EOF.

### 1.10 — `pre-commit` runs Ruff and mypy before every commit.

**Reasoning, step by step:**
1. `pre-commit` (https://pre-commit.com/) catches formatting and type errors before they reach the remote.
2. Minimum `.pre-commit-config.yaml`:
   ```yaml
   repos:
     - repo: https://github.com/astral-sh/ruff-pre-commit
       rev: v0.x.x
       hooks:
         - id: ruff-format
         - id: ruff
     - repo: https://github.com/pre-commit/mirrors-mypy
       rev: v1.x.x
       hooks:
         - id: mypy
   ```
3. Pin tool versions. Don't let `latest` ratchet your project quietly.

### 1.11 — `pyproject.toml` is the single source of project configuration.

**Reasoning, step by step:**
1. `pyproject.toml` (PEP 518, PEP 621) holds: build system, project metadata, dependencies, and tool config (Ruff, mypy, pytest).
2. Don't scatter `setup.cfg`, `tox.ini`, `.flake8`, `.mypy.ini`. Consolidate.
3. Exception: `pre-commit` and CI workflow files live where their tools expect them.

### 1.12 — Tooling drift is a bug.

**Reasoning, step by step:**
1. If the formatter and the codebase disagree, *one* of them is wrong. Fix the rule or fix the code.
2. Pin versions. Don't let `latest` ratchet quietly.
3. Format-on-save in IDE; format-on-commit in pre-commit; format-check in CI. Three rings of defense.

## Cross-references

- Type hints in detail: chapter 03.
- Module organization and `__init__.py`: chapter 12.
- Docstring style: chapter 14.
