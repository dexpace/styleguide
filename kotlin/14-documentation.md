# 14 — Documentation

KDoc is part of the public API. Treat it like code.

## Rules

### 14.1 — Every `public` symbol has KDoc.

**Reasoning, step by step:**
1. A `public` function or class is a contract. Without KDoc, the contract lives in the head of whoever wrote it.
2. KDoc generates IDE tooltips, Dokka output, and is the first thing every caller reads.
3. `internal` and `private` symbols are documented when the name doesn't carry the meaning. Don't paste KDoc on `private fun add(a: Int, b: Int): Int = a + b` — the name already says it.
4. **Lint:** detekt's `UndocumentedPublicFunction` / `UndocumentedPublicClass` set to error in CI.

### 14.2 — KDoc structure: summary line, blank line, body, blank line, tags.

**Reasoning, step by step:**
1. The first line is a one-sentence summary in imperative voice: "Parses the input as a JSON value." Not "This function will parse..." (verbose) and not a fragment.
2. A blank line separates the summary from the body.
3. The body explains *why* and *when to use*, plus any non-obvious behavior. Keep it tight.
4. Tags last: `@param`, `@return`, `@throws`, `@sample`, `@see`.
5. The summary is what shows up in completion menus and Dokka indexes. Spend time on it.

### 14.3 — Explain *why*, not *what* the code does.

**Reasoning, step by step:**
1. The signature documents the *what*. KDoc should add the context the signature can't.
2. `/** Adds two integers. */ fun add(a: Int, b: Int): Int = a + b` — useless. The function name says exactly that.
3. `/** Returns a stable hash suitable for sharding. NOT cryptographically secure. */` — useful. The signature can't tell you that.
4. **Heuristic:** if you can delete the KDoc without losing information the caller needs to use the function correctly, delete it.

### 14.4 — `@param` for every parameter the signature doesn't fully document.

**Reasoning, step by step:**
1. `@param email the user's email address` adds nothing if the parameter is named `email: Email`.
2. `@param threshold the minimum match score, in `[0.0, 1.0]`; values outside this range throw IAE` — adds the range and the failure mode.
3. Use `@param` to add constraints, value ranges, and references to other parameters. Skip it when the type and name carry the meaning.

### 14.5 — `@return` for non-trivial return values.

**Reasoning, step by step:**
1. `@return the parsed user` is redundant for `fun parseUser(raw: String): User`.
2. `@return the parsed user, or `null` if the input is empty or whitespace-only` adds the contract for the `User?` case.
3. For sealed-`Result<T, E>` returns, `@return` explains which `E` cases callers should expect.

### 14.6 — `@throws` for every exception the function deliberately throws or propagates.

**Reasoning, step by step:**
1. Exception throwing isn't in the signature; KDoc is where it lives.
2. List exceptions that the caller might reasonably catch. Don't list every `RuntimeException` that could conceivably escape.
3. Order: most-likely first.
4. `@throws IllegalArgumentException if `email` is malformed` — concrete, actionable, mentions the offending input.

### 14.7 — `@sample` over copy-pasted examples.

**Reasoning, step by step:**
1. `@sample com.acme.docs.OrderClientSample.example` references a real, compiled, tested function — the example can't rot.
2. Inline code blocks in KDoc rot the moment the API changes. Avoid them for non-trivial examples.
3. Sample functions live in a `*-samples` source set, never the main source.

### 14.8 — Package documentation: one `package.md` per public package.

**Reasoning, step by step:**
1. A package is a *thing*, not just a folder. Document its purpose, its entry points, and the rules for what belongs in it.
2. Dokka picks up `package.md` (or `package-info` equivalents) and renders package-level docs.
3. Cover: purpose of the package, the main types, when to reach for this package vs. a sibling.

### 14.9 — Links over restatement. `[OtherClass]`, `[#otherFunction]`.

**Reasoning, step by step:**
1. KDoc supports markdown-style links to other symbols: `[User]`, `[User.email]`, `[parseUser]`.
2. Linking creates navigable docs and stays current under rename refactors (the IDE updates the link).
3. Restating "see also User" without a link is dead text.

### 14.10 — Don't document obvious overrides.

**Reasoning, step by step:**
1. `override fun toString(): String = ...` doesn't need KDoc — it inherits from `Any.toString`.
2. Override with surprising behavior *does* need KDoc explaining the surprise.
3. Implementing an interface method: KDoc only if the implementation's *contract* is narrower than the interface's. The IDE renders the interface's KDoc otherwise.

### 14.11 — `@Deprecated` with `ReplaceWith` and a removal date.

**Reasoning, step by step:**
1. `@Deprecated("Use `newName` instead. Removed in 3.0.", ReplaceWith("newName(arg)"))` gives callers an automated migration via IDE quick-fix.
2. Always state *when* the symbol will be removed. "Deprecated forever" is a lie callers learn not to trust.
3. The replacement should compile and behave identically. If it can't, the deprecation needs a longer KDoc explaining the migration.

### 14.12 — Comments inside function bodies: only where the *why* isn't in the code.

**Reasoning, step by step:**
1. A comment that restates the next line is noise: `// increment counter\ncounter++`.
2. A comment that explains a non-obvious *why* earns its line: `// Apple's smart-quote interferes with our regex; normalize.`.
3. Use TODO comments sparingly and with an owner + date: `// TODO(omar 2026-08-01): remove after API v3 cutover`.
4. Don't leave commented-out code. Delete it. Git remembers.

## Cross-references

- Naming as the first line of documentation: chapter 02.
- API design (what to expose, what to hide before documenting): chapter 10.
- Generating documentation on JVM (Dokka): [JVM guide ch. 08](../kotlin-jvm/08-build-and-distribution.md).
