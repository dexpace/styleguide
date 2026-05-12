# 14 — Documentation

Docstrings are part of the public API. So are type hints. So are examples. Treat them like code.

## Rules

### 14.1 — Every public function, class, and module has a docstring.

**Reasoning, step by step:**
1. A docstring is the first thing `help(thing)`, the IDE tooltip, and documentation generators show. Without one, the contract lives in source-reading.
2. Public = anything that could be imported by another module (and isn't underscored or absent from `__all__`).
3. Private functions and methods: docstring only when the name doesn't fully document.
4. **Lint:** Ruff's `D` rules (pydocstyle) enforce. Configure for the docstring style you pick (Google by default).

### 14.2 — Google-style docstrings.

**Reasoning, step by step:**
1. Google style is readable in source, parseable by Sphinx (with `napoleon`) and pdoc, well-supported by IDEs.
2. **Pattern:**
   ```python
   def charge_card(card: Card, amount: int, *, currency: Currency = Currency.USD) -> Receipt:
       """Charge the given card for the amount in the specified currency.

       The card must be active and have sufficient available credit. On success,
       returns a Receipt with the gateway-issued transaction ID.

       Args:
           card: The card to charge. Must be active.
           amount: Amount in the smallest unit of the currency (cents for USD).
           currency: Currency to charge in. Defaults to USD.

       Returns:
           A Receipt describing the successful charge.

       Raises:
           CardDeclined: If the gateway declines the charge.
           PaymentError: If the gateway is unreachable or returns an unexpected error.
       """
   ```
3. Alternatives: NumPy style (more structured, heavier), reStructuredText (Sphinx default, more markup). Google style is our default.

### 14.3 — Docstrings explain *why* and *when*, not *what*.

**Reasoning, step by step:**
1. The signature documents the *what*. The docstring adds context.
2. `"""Adds two integers."""` on `def add(a: int, b: int) -> int:` is useless.
3. `"""Returns a stable hash for sharding. NOT cryptographically secure — use hashlib for security."""` adds non-obvious information.
4. **Heuristic:** if deleting the docstring loses no information a caller needs, delete it.

### 14.4 — One-line summary first; blank line; then details.

**Reasoning, step by step:**
1. The first line is a one-sentence imperative summary. Ends in a period.
2. Blank line separates summary from details. Sphinx/pdoc both rely on this.
3. The first line appears in completion menus and indexes. Spend time on it.
4. Multi-paragraph details for non-trivial cases. Be brief.

### 14.5 — Examples in docstrings: `Example` block or doctest.

**Reasoning, step by step:**
1. Examples make docs actionable. Show how to use the function, not just what it does.
2. **Format:**
   ```python
   def parse_iso_date(s: str) -> date:
       """Parse an ISO-8601 date.

       Example:
           >>> parse_iso_date("2025-01-01")
           datetime.date(2025, 1, 1)
       """
   ```
3. doctest can execute these (`pytest --doctest-modules`). Make them runnable.
4. For complex examples, prefer a sample function in a `_samples` module — referenced from the docstring, executed by tests.

### 14.6 — Document raised exceptions.

**Reasoning, step by step:**
1. Exception raising isn't in the signature. Docstring `Raises:` is where it lives.
2. List exceptions the caller might reasonably catch. Don't list every `RuntimeError` that could theoretically escape.
3. Order: most-likely first.
4. Specific exception types with what causes them: `Raises: ValueError if amount is non-positive.`

### 14.7 — Type hints document data; docstrings document behavior.

**Reasoning, step by step:**
1. Don't restate the type in the docstring. `Args: amount (int): The amount to charge.` is noise — the signature says `amount: int`.
2. Do add information the type can't: ranges, units, formats, special values.
3. `Args: amount: Amount in the smallest currency unit (cents for USD). Must be positive.` — the type system can't say "smallest unit" or "must be positive."
4. The compiler / mypy sees types. Humans see types *and* docstrings. Make the docstring add what types can't.

### 14.8 — Module docstrings at the top of every module.

**Reasoning, step by step:**
1. The first line of a module is its docstring. Describes the module's responsibility.
2. Sphinx/pdoc use this as the module description in generated docs.
3. **Pattern:**
   ```python
   """Payment processing: card tokenization, gateway dispatch, receipt construction.

   Public entry points:
       charge_card  -- Charge a card for an amount.
       refund       -- Refund a previously-charged transaction.
   """
   ```

### 14.9 — Class docstrings: describe purpose; document `Attributes` when non-obvious.

**Reasoning, step by step:**
1. Class docstring describes the class's purpose, when to use it, and any non-obvious lifecycle.
2. For dataclasses, fields are documented in attribute annotations (and any explicit `field(metadata=...)`). Don't restate trivially.
3. Non-dataclass attributes: `Attributes:` section in the docstring.
4. Don't duplicate `__init__` parameters in the class docstring if `__init__` has its own docstring. Pick one home.

### 14.10 — Deprecation: in the docstring *and* via `warnings.warn`.

**Reasoning, step by step:**
1. `warnings.warn(...)` produces runtime signals (chapter 10 §10.7).
2. The docstring adds a `Deprecated:` note explaining what to use instead.
3. **Pattern:**
   ```python
   def old_name(x: int) -> int:
       """Compute the old thing.

       Deprecated: use ``new_name`` instead. Will be removed in 3.0.
       """
       warnings.warn("old_name is deprecated; use new_name", DeprecationWarning, stacklevel=2)
       return new_name(x)
   ```

### 14.11 — Comments inside function bodies: explain *why*, not *what*.

**Reasoning, step by step:**
1. A comment that restates the next line is noise.
2. A comment that explains a non-obvious *why* earns its line: `# Apple's smart-quote interferes with the parser; normalize.`.
3. TODO comments need owner + date: `# TODO(omar 2026-08-01): remove after API v3 cutover`.
4. Don't leave commented-out code. Delete it. Git remembers.

### 14.12 — Generated docs: `pdoc` or `mkdocs` + `mkdocstrings`. Run in CI.

**Reasoning, step by step:**
1. Docs that don't build don't get read. CI builds docs on every PR.
2. **`pdoc`** — zero-config Python doc generator. Reads docstrings, produces HTML. Best default.
3. **`mkdocs` + `mkdocstrings`** — for documentation sites with handwritten guides + auto-generated reference. Better for libraries with conceptual docs.
4. Sphinx + ReadTheDocs is the heavyweight, mature alternative. Pick if you need cross-references and complex structure.

## Cross-references

- Type hints and what they document: chapter 03.
- `__all__` as public-surface documentation: chapter 10.
- Module README per package: chapter 12.
