# 11 — Testing

Tests are code that runs on every push. They earn their keep when they fail meaningfully on real regressions and pass quietly otherwise.

## Rules

### 11.1 — `pytest`, not `unittest`. Always.

**Reasoning, step by step:**
1. `pytest` is the de facto standard. It supports plain `assert`, fixtures, parameterization, plugins, and minimal ceremony.
2. `unittest`-style `TestCase` classes work in pytest but lose half the benefits — fixtures, parametrize, plain assertions.
3. Write tests as top-level functions. Group with `class Test*:` only when fixtures are class-scoped and shared.
4. Migrate existing `unittest` code when you touch it.

### 11.2 — Test names are sentences in `snake_case`.

**Reasoning, step by step:**
1. `def test_returns_404_when_user_does_not_exist(): ...` reads as the failure message you'll see at 2 am.
2. `def test_user_404():` reads as a code.
3. Pattern: `test_<action>_<expected outcome>_when_<condition>`. Variations are fine; reading aloud as English is not optional.
4. Test files: `tests/test_*.py`. pytest discovers automatically.

### 11.3 — Arrange / act / assert. One concern per test.

**Reasoning, step by step:**
1. Three sections: set up the world, run the operation, check the outcome. Blank lines separate them.
2. One *concern* per test. If a test asserts five unrelated facts, splitting improves failure messages.
3. Acceptable: multiple assertions that all verify the same concern.
4. **Anti-pattern:** god-tests that exercise an entire feature in one function. They fail in unhelpful ways and resist refactoring.

### 11.4 — `@pytest.mark.parametrize` for table-driven cases.

**Reasoning, step by step:**
1. Don't copy-paste tests with different inputs. Parametrize.
2. **Pattern:**
   ```python
   @pytest.mark.parametrize(
       "raw, expected",
       [
           ("2025-01-01", date(2025, 1, 1)),
           ("2025-12-31", date(2025, 12, 31)),
       ],
       ids=["new-year", "year-end"],
   )
   def test_parse_iso_date(raw: str, expected: date) -> None:
       assert parse_iso_date(raw) == expected
   ```
3. Each row is a separate test with its own failure context. The `ids=` parameter labels failures readably.
4. For complex inputs, use a `@dataclass` parameter value rather than a tuple. Readable failure output.

### 11.5 — Fixtures over `setUp`/`tearDown`. Scope them deliberately.

**Reasoning, step by step:**
1. `setUp`/`tearDown` run for every test, even when the setup is shared. Wasteful and noisy.
2. pytest fixtures: scope per-function (default), per-class, per-module, per-session. Pick by what's safe to share.
3. **Pattern:**
   ```python
   @pytest.fixture
   def user() -> User:
       return User(id=UserId("u-1"), email="t@example.com", active=True)
   ```
4. Yield-fixtures for cleanup:
   ```python
   @pytest.fixture
   def temp_dir() -> Iterator[Path]:
       d = Path(tempfile.mkdtemp())
       yield d
       shutil.rmtree(d)
   ```
5. Share fixtures across files via `conftest.py` at the test directory's root.

### 11.6 — Tests are independent. No order, no shared state.

**Reasoning, step by step:**
1. Tests must be runnable in any order, alone or in groups, on any machine, in parallel.
2. Shared mutable state (module-level `_cache: dict = {}`) creates cross-test coupling. One test's failure masks another's success.
3. Each test gets fresh fixtures. Session-scoped fixtures are for *immutable* shared data only (a parsed schema, a fixed clock).
4. **Parallel-by-default:** assume `pytest -n auto` (with `pytest-xdist`) will run. Anything assuming serial order is a future flake.

### 11.7 — Use real implementations where possible. Mock at the genuine seam.

**Reasoning, step by step:**
1. The best test uses the real code with a real input. If real code does I/O, fake the I/O — don't mock the layer above.
2. Mocking everything makes the test re-state the implementation. The test passes when the implementation is wrong in the way you mocked.
3. The genuine seam is the *external boundary*: HTTP, database, clock, filesystem.
4. **Tools:** `unittest.mock.Mock`/`MagicMock` from stdlib; `pytest-mock` for the `mocker` fixture; hand-rolled Protocol-conforming fakes are often clearest.

### 11.8 — Hand-rolled fakes for domain types. Mocks for external IO.

**Reasoning, step by step:**
1. A `FakeUserRepository` that stores users in a dict is clearer than a `MagicMock` configured to return canned values.
2. Mocks shine when the dependency is hard to fake (an HTTP client, a database connection) — let the mocking framework do the call-recording.
3. Domain-shaped fakes can grow to be tiny implementations: in-memory, deterministic, fully behavioral.
4. **Pattern:**
   ```python
   class FakeUserRepository:
       def __init__(self) -> None:
           self._by_id: dict[UserId, User] = {}

       def save(self, user: User) -> None:
           self._by_id[user.id] = user

       def find(self, user_id: UserId) -> User | None:
           return self._by_id.get(user_id)
   ```

### 11.9 — Time, randomness, IDs: injected, not pulled from the wild.

**Reasoning, step by step:**
1. `datetime.now()` inside production code is untestable. Inject a `Clock` Protocol.
2. Same for `random.random()`, `uuid.uuid4()`, any source of non-determinism.
3. Tests use a deterministic clock and seeded random. The fixture provides them.
4. The injection cost is one constructor parameter. The testability payoff is permanent.

### 11.10 — Property-based tests where invariants are natural: `hypothesis`.

**Reasoning, step by step:**
1. `hypothesis` generates inputs and checks invariants. It finds edge cases your manual tests miss.
2. Good properties: round-trip (`decode(encode(x)) == x`), idempotence (`f(f(x)) == f(x)`), monotonicity (`a <= b ⇒ f(a) <= f(b)`).
3. Don't use it as a clever way to run the same test against random nonsense.
4. **Shrinking** is the killer feature: failed tests shrink to a minimal failing case. `hypothesis` shrinks well.

### 11.11 — Async tests: `pytest-asyncio` with `@pytest.mark.asyncio`.

**Reasoning, step by step:**
1. `pytest-asyncio` lets you write `async def test_*`. Configure once in `pyproject.toml`:
   ```toml
   [tool.pytest.ini_options]
   asyncio_mode = "auto"
   ```
2. Alternative: `anyio`'s test plugin if you target multiple async libraries.
3. For time-based async tests, use `freezegun` or a fake clock — don't `await asyncio.sleep(1)` in tests.

### 11.12 — No `time.sleep` in tests. Use polling, fake clocks, or async test machinery.

**Reasoning, step by step:**
1. `time.sleep(1)` is a flake waiting to happen — too short on slow CI, too long for fast tests.
2. For async timing: fake the clock or use `anyio.move_on_after`.
3. For waiting on a condition: poll with a timeout.
4. A test that needs to sleep "to give the thread time to start" has the wrong synchronization model.

### 11.13 — Assertion density mirrors production: 2+ per test on average.

**Reasoning, step by step:**
1. A test with one assertion checks one fact. Often that's right.
2. Complex outcomes assert both the positive (what happened) and the negative (what didn't). "User was created" implies `user.id is not None` *and* `repo.count == 1` *and* `audit.called_once`.
3. Pair-assertion: verify the same property two ways. After `sort(xs)`, assert (a) `len(xs)` unchanged and (b) ordering invariant holds.
4. Don't pile up unrelated assertions to hit a number.

### 11.14 — Coverage is a *floor*, not a ceiling. 100% coverage with 0% meaning is worse than 70% with deliberate tests.

**Reasoning, step by step:**
1. Code coverage measures lines executed. It doesn't measure assertion quality.
2. Set a minimum (often 80%) and fail builds below. Don't gloat about 100%.
3. Focus tests on: behavior at boundaries, edge cases (empty, max, off-by-one), error paths, regressions.
4. Mutation testing (`mutmut`, `cosmic-ray`) is the next level — but expensive. Use selectively.

## Cross-references

- Protocols for testable seams: chapter 03, chapter 06.
- Async testing primitives: chapter 09.
- Test fixtures and the testability of constructor injection: chapter 10.
