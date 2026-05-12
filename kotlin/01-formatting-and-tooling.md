# 01 — Formatting & Tooling

Formatting is **not** a matter of taste. We delegate to tools and disagree elsewhere.

## Rules

### 1.1 — `ktlint` is the formatter. `detekt` is the linter. CI enforces both.

**Reasoning, step by step:**
1. Argument over whitespace is a tax on every PR.
2. `ktlint` implements the official Kotlin style; deviating from it requires a justification longer than `// format-disable`.
3. `detekt` catches structural smells (complexity, naming, dead code) that `ktlint` deliberately doesn't.
4. Running them only locally means broken main branches. They run in CI on every PR and must pass before merge.
5. Format-disable comments are tracked: each one needs a TODO with an owner and a date.

**Configuration:** keep `.editorconfig` at the repo root. Pin tool versions in your build (Gradle version catalog or equivalent).

### 1.2 — Line length is 120 columns. Hard limit.

**Reasoning, step by step:**
1. The default Kotlin convention is 100; we extend to 120 because Kotlin signatures (named args, generic bounds, lambda types) genuinely run wider than Java.
2. 120 fits side-by-side diffs at modern resolutions. Wider lines break review.
3. Wrap *at semantic boundaries*: between arguments, after `=`, between chained `.` calls — not mid-expression.
4. If a line still needs to wrap after argument-wrapping, the function is doing too much.

### 1.3 — Trailing commas everywhere they're allowed.

**Reasoning, step by step:**
1. Trailing commas reduce diff noise: adding a new argument touches one line, not two.
2. They make swapping argument order a one-line change.
3. Kotlin 1.4+ allows them in argument lists, parameter lists, `when` entries, destructuring, and collection literals. Use them in all of these.
4. `ktlint` enforces this when configured; configure it.

### 1.4 — Expression bodies for single-expression functions.

**Reasoning, step by step:**
1. `fun double(x: Int): Int = x * 2` is shorter and clearer than the same function with `return`.
2. Reserved for *single expressions* — if the body needs a local `val`, two `when` arms, or a side effect, use a block body.
3. The return type stays explicit on public/visible functions; type inference is fine for `private`/`internal` if the expression is obvious.
4. **Anti-pattern:** wrapping a multi-statement body in `run { … }` to force an expression body. That's not concise; it's hostile.

```kotlin
// good
fun displayName(user: User): String = "${user.firstName} ${user.lastName}"

// good (block body, multiple statements)
fun displayName(user: User): String {
    val first = user.firstName.trim()
    val last = user.lastName.trim()
    return "$first $last"
}

// bad — `run` is hiding a block body in an expression
fun displayName(user: User) = run {
    val first = user.firstName.trim()
    val last = user.lastName.trim()
    "$first $last"
}
```

### 1.5 — Function-size cap: 60 lines, hard. Aim 15–30.

**Reasoning, step by step:**
1. A function you can't see at once on one screen costs you context every time you read it.
2. Kotlin packs more per line than Java (expression bodies, default args, scope functions), so we go tighter than Go's 70.
3. Counting rule: declaration line, blank lines inside the body, and the closing brace all count. KDoc does not.
4. If a function approaches 60, the right fix is decomposition — extract a private helper, a `when` over a sealed class, or a chained `Sequence` pipeline.
5. The 60-line cap is a *signal*, not a target. If you need 35 lines to be clear, write 35 lines. Don't compress for the sake of the number.

### 1.6 — Imports: explicit, sorted, no wildcards.

**Reasoning, step by step:**
1. Wildcard imports (`import foo.*`) break grep, hide which symbols you depend on, and create silent collisions when upstream adds a name.
2. Sort imports alphabetically; group as `ktlint` orders them. No manual grouping that fights the tool.
3. **Exception:** the official Kotlin style permits wildcards for `kotlinx.android.synthetic` and a few similar generated packages; those are deprecated anyway and don't apply here.

### 1.7 — One top-level declaration per logical unit, not per file.

**Reasoning, step by step:**
1. Kotlin allows multiple top-level declarations per file; this is a feature, not a hazard, when used with discipline.
2. Group declarations that share a *single responsibility* — a sealed hierarchy and its variants, a domain type and its companion factory, a small family of extension functions.
3. Do **not** group unrelated declarations because they happen to start with the same letter.
4. File name matches the *primary* declaration. If no declaration dominates, name the file after the topic (`Json.kt`, `TimeFormat.kt`).

### 1.8 — Blank lines as logical separators.

**Reasoning, step by step:**
1. Cramped code is unreadable code. Whitespace is free.
2. One blank line between logical sections inside a function. One blank line between top-level declarations. Two blank lines between *clusters* of declarations only if the file is large.
3. Never two consecutive blank lines inside a function.
4. No trailing whitespace; the formatter removes it.

### 1.9 — Conditionals: no Yoda, no naked `if` without braces.

**Reasoning, step by step:**
1. `if (x == 5)` reads in the order English speakers expect — keep it that way.
2. Single-line `if` without braces (`if (x) doSomething()`) creates statements that look like declarations and break diffs when expanded.
3. `if` *expression* on one line is fine: `val y = if (x) a else b`. That's an expression, not a statement.
4. `when` should be exhaustive for sealed/enum subjects (see chapter 08).

### 1.10 — Tooling drift is a bug.

**Reasoning, step by step:**
1. If the formatter and the codebase disagree, *one* of them is wrong. Fix the rule or fix the code.
2. Pin tool versions. Don't let `latest` ratchet your project quietly.
3. Format-on-save in IDE; format-on-commit in pre-commit hook; format-check in CI. Three rings of defense.

## Cross-references

- Function size and decomposition: see chapter 05 (Functions).
- Expression-oriented Kotlin: see chapter 07 (Kotlin Idioms).
- KDoc and documentation formatting: see chapter 14.
