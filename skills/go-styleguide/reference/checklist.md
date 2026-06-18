# Go styleguide — full audit checklist

One heading per chapter. Walk every bullet against the code under review.

### 01 — Formatting & Tooling

- Format with `gofmt`; never override its output.
- Lint with `golangci-lint` using the curated linter set; run `go vet` separately in CI.
- Order imports in three groups with `gci` (stdlib, external, local).
- Cap functions at 70 lines; each does one thing.
- Separate logical blocks with whitespace; group related declarations, separate unrelated groups.
- No fixed line length — refactor instead of splitting; don't line-break conditionals or loops, extract operands.
- Reduce nesting with guard clauses and early returns; eliminate unnecessary `else` with defaults.
- Run the full CI gate on every project, race detector included.

### 02 — Naming Conventions

- Size every name to its scope; short for narrow scope, descriptive for wide.
- Encode visibility with case, not keywords; MixedCaps, never underscores or SCREAMING_CASE.
- Keep package names short, lowercase, shadow-proof; receivers short, consistent, derived from the type.
- Name constants by role, not value; don't repeat package, type, receiver, or params in a name.
- Never shadow package, import, or builtin names; never name identifiers after predeclared builtins.
- Name interfaces `-er` for one method, a noun for many; drop the `Get` prefix on getters.
- Nouns for value functions, verbs for action functions; keep initialisms in one consistent case.
- Name sentinel errors `Err…` and error types `…Error`; prefix constructors with `New`, signal fallibility in the return.
- Add units to names, qualifier last; end format-string functions with `f`.
- Name tests `TestFunc_Scenario` or use subtests; name test doubles by behavior, not "mock".
- Avoid import aliases unless they resolve a conflict; govern abbreviations by scope, keep only idiomatic shorthands.

### 03 — Error Handling

- Handle every error; never silently discard.
- Wrap every returned error with context; choose `%w` vs `%v` as an API decision.
- Place `%w` at the end, except for sentinel categorization; one wrap level.
- Never match error text — branch on structure with `errors.Is`/`errors.As`, never assertions or equality.
- Don't add redundant annotations; signal failure out-of-band, never through sentinel values.
- Return `error`, not concrete error types; keep error strings lowercase and unpunctuated.
- Use package-level sentinel errors for expected conditions; custom error types for structured data, implement `Unwrap`.
- Build error hierarchies through wrapping, not inheritance.
- Don't panic on runtime conditions; never let panics escape a package; use `recover()` only at boundaries.
- Restrict `Must` functions to startup, init, and test helpers.
- Validate inputs at public entry; check errors inline immediately after the call; handle each error exactly once.
- Maintain assertion discipline; handle close errors with the deferred close pattern.

### 04 — Concurrency

- Share memory by communicating, not the reverse.
- Make `context.Context` the first parameter and thread it through; never store it in a struct.
- Manage goroutine groups with `errgroup`; fan-out without errors uses `sync.WaitGroup`, always bound concurrency.
- Protect shared state with `sync.Mutex`, kept small; `sync.Once` for lazy init.
- Give every long-running loop a `ctx.Done()` case; never use `time.Sleep` in production code.
- Parallelize across cores only for CPU-bound work; declare channel direction in every signature.
- Keep channel buffer size at one or none; never start goroutines in `init()`.
- Prefer synchronous functions; give every goroutine a clear, documented lifetime.
- Test for goroutine leaks with `goleak`; manage worker lifecycle with context plus a done channel.
- Avoid the known concurrency anti-patterns.

### 05 — API Design

- Accept interfaces, return concrete types; keep interfaces small and consumer-driven.
- Use functional options for configurable constructors; rely on documented defaults.
- Name and validate constructors; keep method receivers consistent.
- Express middleware as a handler-wrapping function; pass values, not pointers, when only reading.
- Use `any`, not `interface{}`; reserve type aliases for migration only; reach for generics only on clear need.
- Never pass a pointer to an interface; know the method-set rules for value vs pointer receivers.
- Bound recursion in library code; put limits on everything (loops, queues, retries, buffers, timeouts).
- Design types so the zero value is useful or obviously invalid.
- Always use the comma-ok form for type assertions; type switches carry a default case.
- Compose interfaces by embedding smaller ones; export the interface, not a type that only implements it.
- Use `new()` and `make()` for their distinct purposes; assert interface compliance at compile time.
- Avoid embedding types in public structs; start enums at one.
- Copy slices and maps at API boundaries; avoid mutable globals, inject dependencies; exit only in `main`.

### 06 — Testing

- Default to table-driven tests; keep them flat and declarative.
- Call `t.Helper()` in every helper; keep assertions inline, no `*testing.T` helpers that fail directly.
- `testify/require` for preconditions, `testify/assert` for values; keep test state self-contained.
- Name tests `TestFunctionName_Scenario`; use `httptest.NewServer` for HTTP-level integration tests.
- Mark independent tests parallel; use `t.Cleanup()` instead of `defer`; keep test logic in the test function.
- Prefer real dependencies over mocks; mock with interfaces and structs, not frameworks.
- Prefer `t.Error` over `t.Fatal`; never call `t.Fatal` from a goroutine.
- Write failures that diagnose themselves; compare with `cmp`, never `reflect.DeepEqual`; compare whole structures.
- Compare only stable results; store complex expected output in golden files.
- Organize test files next to code, prefer external test packages; provide acceptance tests for contract APIs.
- Never start background goroutines in helpers without stopping them; name test doubles by behavior.

### 07 — Package Organization

- One package per directory, one directory per package; name packages as short, lowercase, composable nouns.
- Use `internal/` aggressively for implementation-detail packages.
- Reserve `cmd/` for binaries, `pkg/` for genuine external consumption.
- Keep packages small and single-purpose; never create `utils/`, `helpers/`, or `common/`.
- Keep dependencies pointing inward toward a dependency-free domain.
- One file per major type; put package documentation in `doc.go`.
- Group imports in standard order, separated by blank lines; rename imports only when required.
- Never use dot imports; restrict blank imports to `main`, tests, and `//go:embed`.

### 08 — Logging

- Log through `log/slog`, not a third-party logger; inject the logger, never the global default in library code.
- Log key-value attributes, never `fmt.Sprintf`; choose the level by meaning.
- Mask secrets and PII with `slog.LogValuer`; redact at the type, never log secrets or PII.
- Attach request context once with `slog.With`; pass the error as a structured attribute.
- Guard expensive log arguments behind a level check.
- Don't log and return; log once at the boundary.
- Define no command-line flags in library code; logging must have no side effects on control flow.

### 09 — Serialization

- Map JSON fields with explicit struct tags; use `omitempty` only on optional fields, never required ones.
- Implement `json.Marshaler`/`json.Unmarshaler` for types with special serialization.
- Carry time as `time.Time`/`time.Duration`, serialize as RFC 3339; never use `float64` for money.
- Validate after every `json.Unmarshal`; never trust external data.
- Stream large payloads with `json.Decoder` and `DisallowUnknownFields()`.
- Use `%q` for quoted strings; implement `fmt.Stringer` for human-readable types.
- Initialize structs with field names; init maps by purpose (`make()` to populate, literals for fixed sets).
- Prefer composite literals over `new()`, with size hints; declare format strings used outside `fmt.*` as `const`.

### 10 — Resource Management

- Pair every acquisition with an immediate `defer` close; acquire in dependency order, LIFO release.
- Always close HTTP response bodies; thread `context.Context` first in long-running operations.
- Bound every wait with a timeout and defer the cancel.
- Configure connection pools explicitly; never ship `http.DefaultClient`.
- Shut down gracefully on `os.Signal`; bound `io.ReadAll` on untrusted input with a named limit.
- Reserve `sync.Pool` for hot-path allocations only; never rely on finalizers for cleanup.
- Use `crypto/rand`, never `math/rand`, for unpredictable values.
- Treat slices as headers — copy explicitly when you need independence; use comma-ok when the zero value is ambiguous.
- Give `make` a capacity hint whenever the size is known.

### 11 — Documentation

- Put a GoDoc comment on every exported symbol, starting with its name; write complete, punctuated sentences.
- Give every package a package comment, flush against the clause.
- Name result parameters only when they make the signature clearer.
- Write testable Example functions for public APIs; signal-boost code that subtly deviates from a pattern.
- Explain why in inline comments, not what; give every TODO an owner and a tracking reference.
- Put a license header on every source file, enforced in CI; delete dead code, never comment it out.
- Link to other symbols with GoDoc `[Type]` syntax; document the concurrency safety of every exported type.
- Document cleanup requirements on the constructor and the errors a function may return.
- Write error messages as lowercase, unpunctuated fragments.

### 12 — Variables & Declarations

- Use `:=` for explicit values, `var` for intentional zero values.
- Omit redundant types on top-level declarations; keep them when the type differs.
- Name struct-literal fields across package boundaries.
- Use composite literals for known values, zero-value declarations for empty ones.
- Give `make` a size hint only when the final size is known; specify channel direction on declarations.
- Treat `nil` as a valid empty slice; do not shadow standard-library package names.
- Declare variables in the narrowest scope that stays readable; clarify naked boolean and numeric parameters.
- Use raw string literals to avoid hand-escaping.

### 13 — Performance Hints

- Prefer `strconv` over `fmt` for primitive conversions.
- Convert strings to bytes once, then reuse.
- Keep small values on the stack, not the heap.
- Give `make` a capacity hint when the size is known.
- Skip `fmt.Errorf` in extremely hot error paths.
- Build strings with `strings.Builder`, not repeated `+`.
