# 14 — Documentation

KDoc is part of the public API. Treat it like code.

## What good looks like

```kotlin
/**
 * Returns a stable hash of [key] for assigning it to one of [shardCount] shards.
 *
 * Stable across processes and releases, so the same key always lands on the same
 * shard — callers persist the result. NOT cryptographically secure: the output is
 * trivial to forge, so never use it for tokens or signatures.
 *
 * @param key the value to shard; must be non-blank.
 * @param shardCount the number of shards; must be positive.
 * @return the shard index, in `[0, shardCount)`.
 * @throws IllegalArgumentException if `key` is blank or `shardCount` is not positive.
 * @sample com.acme.shard.ShardSamples.assignOrder
 * @see ShardRouter for the routing layer built on this.
 */
public fun shardFor(key: String, shardCount: Int): Int {
    require(key.isNotBlank()) { "key must be non-blank" }
    require(shardCount > 0) { "shardCount must be positive, got $shardCount" }
    // FNV-1a: cheap, well-distributed, and not the JDK hashCode (which varies by release).
    val hash = key.fold(FNV_OFFSET) { acc, c -> (acc xor c.code.toLong()) * FNV_PRIME }
    return (hash and Long.MAX_VALUE).mod(shardCount.toLong()).toInt()
}
```

The summary leads in imperative voice and the body explains *why* and the security caveat the signature cannot (14.2, 14.3); `@param` carries the constraints, not the obvious types (14.4); `@return` states the range and `@throws` names the offending input most-likely-first (14.5, 14.6); `@sample` points at a compiled function instead of an inline block, and `[ShardRouter]` links rather than restates (14.7, 14.9); the inline comment explains the *why* of the algorithm choice, not the *what* (14.12).

## Rules

### 14.1 — Every `public` symbol has KDoc.

**Reasoning, step by step:**
1. A `public` function or class is a contract. Without KDoc, the contract lives in the head of whoever wrote it.
2. KDoc generates IDE tooltips, Dokka output, and is the first thing every caller reads.
3. `internal` and `private` symbols are documented when the name doesn't carry the meaning. Don't paste KDoc on `private fun add(a: Int, b: Int): Int = a + b` — the name already says it.

**Enforcement:** detekt's `UndocumentedPublicFunction` / `UndocumentedPublicClass` set to error in CI.

### 14.2 — KDoc structure: summary line, blank line, body, blank line, tags.

**Reasoning, step by step:**
1. The first line is a one-sentence summary in imperative voice: "Parses the input as a JSON value." Not "This function will parse..." (verbose) and not a fragment.
2. A blank line separates the summary from the body.
3. The body explains *why* and *when to use*, plus any non-obvious behavior. Keep it tight.
4. Tags last: `@param`, `@return`, `@throws`, `@sample`, `@see`.
5. The summary is what shows up in completion menus and Dokka indexes. Spend time on it.

**Enforcement:** review; Dokka render check that summaries are single sentences and tags trail the body.

### 14.3 — Explain *why*, not *what* the code does.

**Reasoning, step by step:**
1. The signature documents the *what*. KDoc should add the context the signature can't.
2. `/** Adds two integers. */ fun add(a: Int, b: Int): Int = a + b` — useless. The function name says exactly that.
3. `/** Returns a stable hash suitable for sharding. NOT cryptographically secure. */` — useful. The signature can't tell you that.
4. **Heuristic:** if you can delete the KDoc without losing information the caller needs to use the function correctly, delete it.

**Enforcement:** review against the delete-test heuristic; reject KDoc that paraphrases the signature.

### 14.4 — `@param` for every parameter the signature doesn't fully document.

**Reasoning, step by step:**
1. `@param email the user's email address` adds nothing if the parameter is named `email: Email`.
2. `@param threshold the minimum match score, in `[0.0, 1.0]`; values outside this range throw IAE` — adds the range and the failure mode.
3. Use `@param` to add constraints, value ranges, and references to other parameters. Skip it when the type and name carry the meaning.

**Enforcement:** review; `@param` carries a constraint or range, never a restatement of type and name.

### 14.5 — `@return` for non-trivial return values.

**Reasoning, step by step:**
1. `@return the parsed user` is redundant for `fun parseUser(raw: String): User`.
2. `@return the parsed user, or `null` if the input is empty or whitespace-only` adds the contract for the `User?` case.
3. For sealed-`Result<T, E>` returns, `@return` explains which `E` cases callers should expect.

**Enforcement:** review; nullable and `Result` returns document their cases, trivial returns omit `@return`.

### 14.6 — `@throws` for every exception the function deliberately throws or propagates.

**Reasoning, step by step:**
1. Exception throwing isn't in the signature; KDoc is where it lives.
2. List exceptions that the caller might reasonably catch. Don't list every `RuntimeException` that could conceivably escape.
3. Order: most-likely first.
4. `@throws IllegalArgumentException if `email` is malformed` — concrete, actionable, mentions the offending input.

**Enforcement:** review; `@throws` on every deliberate throw, checked against the implementation's `require`/`throw` sites.

### 14.7 — `@sample` over copy-pasted examples.

**Reasoning, step by step:**
1. `@sample com.acme.docs.OrderClientSample.example` references a real, compiled, tested function — the example can't rot.
2. Inline code blocks in KDoc rot the moment the API changes. Avoid them for non-trivial examples.
3. Sample functions live in a `*-samples` source set, never the main source.

**Enforcement:** Dokka `samples` source set compiled in CI; review forbids inline code blocks for non-trivial examples.

### 14.8 — Package documentation: one `package.md` per public package.

**Reasoning, step by step:**
1. A package is a *thing*, not just a folder. Document its purpose, its entry points, and the rules for what belongs in it.
2. Dokka picks up `package.md` (or `package-info` equivalents) and renders package-level docs.
3. Cover: purpose of the package, the main types, when to reach for this package vs. a sibling.

**Enforcement:** CI check that every public package directory has a `package.md` registered with Dokka.

### 14.9 — Links over restatement. `[OtherClass]`, `[#otherFunction]`.

**Reasoning, step by step:**
1. KDoc supports markdown-style links to other symbols: `[User]`, `[User.email]`, `[parseUser]`.
2. Linking creates navigable docs and stays current under rename refactors (the IDE updates the link).
3. Restating "see also User" without a link is dead text.

**Enforcement:** Dokka fails on unresolved `[symbol]` links; review rejects bare "see also" prose.

### 14.10 — Don't document obvious overrides.

**Reasoning, step by step:**
1. `override fun toString(): String = ...` doesn't need KDoc — it inherits from `Any.toString`.
2. Override with surprising behavior *does* need KDoc explaining the surprise.
3. Implementing an interface method: KDoc only if the implementation's *contract* is narrower than the interface's. The IDE renders the interface's KDoc otherwise.

**Enforcement:** review; KDoc on an override only when it narrows or surprises against the inherited contract.

### 14.11 — `@Deprecated` with `ReplaceWith` and a removal date.

**Reasoning, step by step:**
1. `@Deprecated("Use `newName` instead. Removed in 3.0.", ReplaceWith("newName(arg)"))` gives callers an automated migration via IDE quick-fix.
2. Always state *when* the symbol will be removed. "Deprecated forever" is a lie callers learn not to trust.
3. The replacement should compile and behave identically. If it can't, the deprecation needs a longer KDoc explaining the migration.

**Enforcement:** review; every `@Deprecated` carries `ReplaceWith` and a removal version, checked at API-review time.

### 14.12 — Comments inside function bodies: only where the *why* isn't in the code.

**Reasoning, step by step:**
1. A comment that restates the next line is noise: `// increment counter\ncounter++`.
2. A comment that explains a non-obvious *why* earns its line: `// Apple's smart-quote interferes with our regex; normalize.`.
3. Use TODO comments sparingly and with an owner + date: `// TODO(omar 2026-08-01): remove after API v3 cutover`.
4. Don't leave commented-out code. Delete it. Git remembers.

**Enforcement:** review; detekt flags commented-out code and unowned `TODO` markers.

## Cross-references

- Naming as the first line of documentation: chapter 02.
- API design (what to expose, what to hide before documenting): chapter 10.
- Generating documentation on JVM (Dokka): [JVM guide ch. 08](../kotlin-jvm/08-build-and-distribution.md).
