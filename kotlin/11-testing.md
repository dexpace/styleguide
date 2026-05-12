# 11 — Testing

Tests are code that runs on every push. They earn their keep when they fail meaningfully on real regressions and pass quietly otherwise.

## Rules

### 11.1 — Tests describe behavior, not implementation. Names are sentences.

**Reasoning, step by step:**
1. A test name is the failure message you'll read at 2 am. `testFoo123` is unhelpful; `\`returns 404 when user does not exist\`` is.
2. Backticked test names are encouraged — this is the one place backticks belong (see chapter 02).
3. Recommended shape: `\`<action> <expected outcome> when <condition>\``. Variations are fine; reading aloud as English isn't optional.
4. Test names appear in CI output, in IDE green/red trees, and in flakiness dashboards. Treat them as public API of the test suite.

### 11.2 — Arrange / act / assert. One concern per test.

**Reasoning, step by step:**
1. Each test has three sections: set up the world, run the operation, check the outcome. Whitespace separates them.
2. One *concern* per test. If a test asserts five unrelated facts, splitting it into five tests improves failure messages — each tells you precisely what broke.
3. Acceptable: multiple assertions that all verify the *same* concern. E.g., asserting both `result.status == OK` and `result.body.user == expectedUser` is the same concern ("the request succeeded with the expected user").
4. **Anti-pattern:** "god tests" that exercise an entire feature in one method. They fail in unhelpful ways and resist refactoring.

### 11.3 — Parameterized tests for table-driven cases.

**Reasoning, step by step:**
1. When the same logic runs against several inputs, parameterize. Don't copy-paste tests.
2. JUnit 5: `@ParameterizedTest` + `@MethodSource`/`@CsvSource`. Kotest: `withData(...)` blocks.
3. Each row of the table generates a separate failure context — far better than one test with 12 assertions where the first failure hides the rest.
4. **Make the table data-class shaped.** A `data class Case(val name: String, val input: X, val expected: Y)` is readable; a `Triple<X, Y, Z>` is not.
5. Use `@DisplayName` or Kotest's named cases so the failure shows the *case name*, not a generic index.

### 11.4 — Use real implementations where possible. Mock only at the genuine seam.

**Reasoning, step by step:**
1. The best test uses the real code with a real input. If the real code does I/O, the *test* should fake the I/O — not mock the layer above.
2. Mocking everything makes the test re-state the implementation. The test passes when the implementation is wrong in the way you mocked.
3. The genuine seam is the *external boundary*: a network call, a database connection, a clock. Mock that, not the service layer.
4. **Mocking framework preference:** MockK for Kotlin (on JVM: see [JVM guide ch. 11](../kotlin-jvm/) for setup). For pure-Kotlin domain types, often a hand-written fake (`FakeUserRepository`) reads cleaner than a mock.

### 11.5 — Test doubles named for what they are.

**Reasoning, step by step:**
1. Naming taxonomy:
   - **Stub** — returns canned data, no logic.
   - **Fake** — working in-memory implementation (a Map-backed `UserRepository`).
   - **Spy** — records calls, delegates to real.
   - **Mock** — framework-generated, has expectations.
2. Name reflects type: `StubClock`, `FakeUserRepository`, `SpyAuditor`, `MockGateway`.
3. **Anti-pattern:** "MockClock" when it's actually a hand-rolled fake. Renames over reality lie to the next reader.

### 11.6 — Property-based tests where the property is naturally invariant.

**Reasoning, step by step:**
1. Property-based testing (Kotest `forAll`, jqwik) generates inputs and checks invariants. It finds edge cases your manual tests miss.
2. Good properties: round-trip (`decode(encode(x)) == x`), idempotence (`f(f(x)) == f(x)`), monotonicity (`a <= b → f(a) <= f(b)`).
3. Use it where you genuinely have invariants. Don't use it as a clever way to run the same test 100 times against random nonsense.
4. **Shrinking** is the killer feature: when a property fails, the framework shrinks the input to a minimal failing case. Use frameworks that shrink well.

### 11.7 — Tests are independent. No order, no shared state.

**Reasoning, step by step:**
1. Tests must be runnable in any order, alone or in groups, on any machine, in parallel.
2. Shared mutable state (`companion object var counter = 0`) creates cross-test coupling. One test's failure masks another's success.
3. Each test sets up its own fixtures. `@BeforeEach` per test, not `@BeforeAll`. `@BeforeAll` is acceptable only for truly immutable shared setup (a parsed schema, a fixed clock).
4. **Parallel-by-default:** assume the test runner will execute tests concurrently. Anything assuming serial order is a future flake.

### 11.8 — Fixtures via builders or fixture functions, not test-class mutable state.

**Reasoning, step by step:**
1. A `data class UserFixture { fun build(): User = ... }` or a top-level `fun anyUser(...): User` lets each test ask for exactly the user it wants.
2. The builder defines sensible defaults; the test overrides only what matters.
3. Don't share `private lateinit var user: User` across tests — that's state coupling masquerading as DRY.
4. **Pattern:**
   ```kotlin
   fun anyUser(
       id: UserId = UserId("u-${nextId()}"),
       email: Email = Email("test@example.com"),
       isActive: Boolean = true,
   ): User = User(id, email, isActive)
   ```
   Test overrides only what it cares about: `val u = anyUser(isActive = false)`.

### 11.9 — Time, randomness, and IDs: injected, not pulled from the wild.

**Reasoning, step by step:**
1. `Clock.systemUTC().now()` inside production code is untestable. Inject a `Clock`.
2. Same for `Random`, `UUID.randomUUID()`, any source of non-determinism.
3. Tests use `Clock.fixed(...)`, deterministic `Random(seed)`, and ID generators they control.
4. The injection cost is one constructor parameter. The testability payoff is permanent.

### 11.10 — Useful failure messages. The framework helps; you help more.

**Reasoning, step by step:**
1. AssertJ-style fluent assertions (`assertThat(x).isEqualTo(y)`) produce diff-shaped failure messages — better than `assertEquals(x, y)`.
2. Kotest's matcher syntax (`x shouldBe y`, `coll shouldContain v`) reads naturally.
3. Add context with `.withFailMessage("when processing $request")` or Kotest's `assertSoftly { ... }`.
4. **Rule:** when a test fails, the message should let a teammate debug *without* re-running the test. Include relevant inputs, intermediate values, and what was expected.

### 11.11 — No `Thread.sleep` in tests. Use polling, time-virtualization, or coroutines.

**Reasoning, step by step:**
1. `Thread.sleep(1000)` is a flake waiting to happen — too short on slow CI, too long for every other test.
2. For async work: use the coroutine test machinery (`runTest`, `TestCoroutineScheduler.advanceTimeBy`) to virtualize time.
3. For genuinely external timing: poll with a timeout. `awaitility` or hand-rolled `while (System.currentTimeMillis() < deadline)` patterns.
4. A test that needs to sleep "to give the thread time to start" is a test that has the wrong synchronization model.

### 11.12 — Assertion density mirrors production code: 2+ per test on average.

**Reasoning, step by step:**
1. A test with one assertion checks one fact. Often that's right.
2. Tests verifying complex outcomes should assert *both* the positive (what happened) and the negative (what didn't). Example: "user was created" implies `user.id != null` *and* `repository.count == 1` *and* `audit.log was called exactly once`.
3. Pair-assertion: verify the same property two ways. After `sort(xs)`, assert (a) `xs.size unchanged` (no loss) and (b) ordering invariant holds.
4. Don't pile up unrelated assertions just to hit a number. Two related assertions > one bare one > five unrelated ones.

## Cross-references

- Coroutine testing helpers: chapter 09 and [JVM guide ch. 02](../kotlin-jvm/02-jvm-concurrency.md).
- Mocking framework setup (MockK, etc.) on JVM: [JVM guide](../kotlin-jvm/).
- Naming conventions (incl. backticked test names): chapter 02.
