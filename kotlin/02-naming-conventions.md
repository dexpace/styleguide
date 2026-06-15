# 02 — Naming Conventions

A good name is the cheapest documentation in the codebase. We pick conventions that the compiler and IDE can enforce, and we keep the surface area small.

## What good looks like

```kotlin
package com.acme.checkout

const val MAX_RETRIES = 3

/** A request is a noun describing what it is, not what it does. */
data class PaymentRequest(val userId: String, val amountCents: Long)

sealed interface PaymentResult {
    data class Approved(val receipt: Receipt) : PaymentResult
    data class Declined(val reason: String) : PaymentResult
}

class PaymentGateway(private val httpClient: HttpClient) {
    val isReady: Boolean get() = httpClient.isConnected

    fun submitPayment(request: PaymentRequest): PaymentResult {
        var attempt = 0
        while (attempt < MAX_RETRIES) {
            val response = httpClient.post(request)
            if (response.shouldRetry) { attempt++; continue }
            return if (response.approved) PaymentResult.Approved(response.receipt)
            else PaymentResult.Declined(response.reason)
        }
        return PaymentResult.Declined("exhausted $MAX_RETRIES retries")
    }
}
```

`camelCase` names locals and members, `PascalCase` the types, `SCREAMING_SNAKE_CASE` the compile-time constant (2.1); `httpClient` treats the acronym as a word rather than `HTTPClient` (2.1); the short-lived `attempt` counter earns a short name while the long-lived `httpClient` does not (2.2); `isReady` is a noun-shaped boolean property, not `getReady()` (2.3, 2.6); `PaymentRequest` and `PaymentGateway` name what they are while `submitPayment` names what it does (2.7); `isReady`/`shouldRetry` take the positive form (2.8); the parameter `request` reads cleanly at a named-argument call site (2.9); the sealed parent is the noun `PaymentResult` with specific-noun variants `Approved`/`Declined` (2.7).

## Rules

### 2.1 — `camelCase` for everything that isn't a type or a constant.

**Reasoning, step by step:**
1. Functions, properties, parameters, local `val`/`var`: `camelCase`. Types and class-like constructs: `PascalCase`. Compile-time constants (`const val`) and top-level `val` of an immutable primitive at file scope: `SCREAMING_SNAKE_CASE`.
2. Kotlin official conventions enforce this and `ktlint` will reject deviations.
3. Acronyms are treated as words: `httpClient`, `xmlParser`, `userId`. Not `HTTPClient`, not `XMLParser`. Reasoning: `userIDFromHTTPRequest` is unreadable; `userIdFromHttpRequest` is not.
4. Two-letter acronyms in type names *may* stay uppercase (`IOException`) because we inherit them from the JDK; new types use `Io*` style.

**Enforcement:** ktlint `standard:property-naming`, `standard:function-naming`, and `standard:class-naming` at error in CI.

### 2.2 — Name scope-proportional.

**Reasoning, step by step:**
1. A loop counter inside a 5-line block can be `i`. A `i` at module scope is a crime.
2. Short scope → short name; long scope → descriptive name. The variable's lifetime determines how much context the reader has when they encounter it.
3. **Heuristic:** if a `val` lives within 10 lines of its declaration, 1–3 characters is fine. If it lives across an entire function (>20 lines) or across a class, write a real word.
4. Exception: domain conventions (`r` for radius in a geometry library) are fine if they're documented at the type level.

**Enforcement:** review; scope-proportionality is a judgment call no linter encodes.

### 2.3 — Properties replace getters and setters. Name them as nouns.

**Reasoning, step by step:**
1. Kotlin has properties — use them. `val name: String` not `fun getName(): String`.
2. Boolean properties: `is*` / `has*` / `should*`. `user.isActive`, `payment.hasRefunds`, `request.shouldRetry`.
3. **Anti-pattern:** writing `fun getName()` because the Java idiom is muscle memory. If Java callers need bean naming, see the [JVM guide](../kotlin-jvm/) — the language solves this with `@JvmName` or by writing it as a property and letting the compiler generate `getName()` for you.
4. Functions describe *actions* and start with verbs: `parseJson`, `loadUser`, `closeStream`. Properties describe *state* and start with nouns or `is/has`.

**Enforcement:** review; a `getX()`/`setX()` function in Kotlin sources is flagged in code review.

### 2.4 — Backticks only in test names.

**Reasoning, step by step:**
1. `fun \`returns 404 when user not found\`()` reads like a sentence — perfect for test names.
2. Outside tests, backticked identifiers are a sign you're working around a poor name or a Java keyword collision. Fix the name instead.
3. Never use backticks for production identifiers, even when they "look nicer." The compiler accepts them; humans do not.

**Enforcement:** detekt `BacktickIdentifiers` (or equivalent) at error outside test source sets.

### 2.5 — Packages: all lowercase, no underscores, structured by feature.

**Reasoning, step by step:**
1. Kotlin official convention: lowercase, no underscores. `com.acme.checkout` not `com.acme.check_out`.
2. Multi-word segments are concatenated: `com.acme.userprofile`. Awkward, but consistent with the JDK and the rest of the Kotlin ecosystem.
3. Structure packages by *feature*, not by *technical layer*. `com.acme.checkout` (all checkout code) beats `com.acme.controllers` + `com.acme.services` + `com.acme.repositories` (each layer scattered across features).
4. Mirror the package structure in your directory layout — even though Kotlin doesn't require it, every tool assumes it.

**Enforcement:** ktlint `standard:package-name` at error; directory-to-package alignment checked in review.

### 2.6 — Avoid Hungarian notation and type prefixes.

**Reasoning, step by step:**
1. `strName`, `bIsActive`, `iCount` are 1990s Windows-era patterns. The type system tells the reader the type.
2. `IUser` (Java/C# "interface" prefix) is also banned. Kotlin doesn't use it; the official conventions explicitly recommend against it.
3. Exception: well-established prefixes like `is`/`has` for booleans, `on*` for callbacks (`onSuccess`, `onError`).

**Enforcement:** review; Hungarian prefixes and `I`-prefixed interfaces are rejected in code review.

### 2.7 — Class names describe *what they are*, function names describe *what they do*.

**Reasoning, step by step:**
1. Classes are nouns: `UserRepository`, `PaymentRequest`, `JsonParser`.
2. Functions are verbs: `loadUser`, `submitPayment`, `parse`. Top-level functions even more so.
3. Beware of `*Manager`, `*Helper`, `*Util`, `*Service` — these often signal that the class has no single responsibility and could be a top-level function or split into smaller types.
4. Sealed hierarchies: parent is the noun (`PaymentResult`), variants are *specific* nouns (`PaymentResult.Approved`, `PaymentResult.Declined.InsufficientFunds`). Variants nested inside the parent — see chapter 06.

**Enforcement:** review; `*Manager`/`*Helper`/`*Util` names trigger a single-responsibility challenge.

### 2.8 — Function names: positive form for booleans; verb-first for actions.

**Reasoning, step by step:**
1. `isReady()` is clearer than `notReady()`. Negative names compound badly under negation (`!notReady()` is a head-scratcher).
2. `loadUser(id)` not `userLoad(id)` and not `getUser(id)` if it does I/O. `get*` implies a cheap accessor; `load*`/`fetch*` implies I/O.
3. Builders / mutators are verb-first: `withRetries(...)`, `andHeaders(...)`. Reads like a sentence in a chain.

**Enforcement:** review; negative boolean names and `get*` on I/O functions are flagged in code review.

### 2.9 — Parameter names live forever in named-argument call sites.

**Reasoning, step by step:**
1. Kotlin supports named arguments; callers can write `connect(host = "localhost", port = 8080)`. The parameter name is part of your public API.
2. Renaming a parameter is a *binary-incompatible change* for any caller using named args. Treat parameter names like method names.
3. Therefore: choose parameter names carefully on `public` and `internal` APIs. `value: T` is fine on a one-line extension; `s: String` on a public function is not.
4. Tools (`binary-compatibility-validator` on JVM, equivalent linting elsewhere) will flag accidental parameter renames in public API.

**Enforcement:** `binary-compatibility-validator` fails CI on a public-API parameter rename.

### 2.10 — Test doubles: name by what they are, not what they pretend to be.

**Reasoning, step by step:**
1. `FakeUserRepository` is honest. `MockUserRepository` is honest only if it's actually a mocking-framework mock. Don't lie.
2. Naming taxonomy: `Stub*` (returns canned data), `Fake*` (working in-memory implementation), `Spy*` (records calls, delegates to real), `Mock*` (framework-generated, has expectations).
3. Test fixtures and builders end in `Fixture` or `Builder`: `UserFixture`, `OrderRequestBuilder`.

**Enforcement:** review; `Mock*` naming on a hand-written fake is rejected in code review.

## Cross-references

- Visibility modifiers (`internal`, `private`, `public`): chapter 10 (API Design).
- Test naming and structure: chapter 11 (Testing).
- KDoc on public symbols (which the name doesn't fully document): chapter 14.
