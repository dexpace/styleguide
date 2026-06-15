# 12 — Variables & Declarations

How a value comes into existence is the first thing a reader sees. Go gives you two declaration forms, the zero value as a working default, and composite literals that say exactly what a value holds. This chapter fixes which form to reach for: `:=` carries an explicit value, `var` declares an intentional zero, struct literals name their fields across package lines, and every variable lives in the narrowest scope that still reads well. The aim is declarations that announce intent without redundant types, surplus allocation, or ambiguous call sites.

## What good looks like

```go
// var for intentional zero values, := for explicit ones.
var (
    count int          // zero is the starting point
    buf   bytes.Buffer // ready to use at its zero value
    mu    sync.Mutex   // ready to lock at its zero value
)

timeout := 30 * time.Second
results := make([]string, 0, 10)

// Top-level: omit the type when the expression already produces it.
var logger = slog.Default()
var _ http.Handler = (*Server)(nil) // compile-time interface check keeps its type

// nil is a valid empty slice; named struct fields cross the package line.
func recentNames(ctx context.Context, store otherpkg.Store) []string {
    var names []string // nil, allocated on first append
    if rows, err := store.Query(ctx, otherpkg.Filter{Limit: 10}); err == nil {
        for _, r := range rows {
            names = append(names, r.Name)
        }
    }
    return names // nil if nothing matched -- this is fine
}
```

This block uses `var` for zero values and `:=` for explicit ones (12.1), omits the redundant type on a top-level declaration while keeping it on the interface check (12.2), names struct fields for an out-of-package type (12.3), starts from a `nil` slice rather than an empty literal (12.7), and scopes `rows`/`err` to the `if` that needs them (12.9).

## Rules

### 12.1 — Use `:=` for explicit values and `var` for intentional zero values.

**Reasoning, step by step:**
1. Short variable declarations (`:=`) read best when you assign an explicit, non-zero value; the form is concise and the value is right there. Reserve `var` for the case where the zero value is the intended starting state.

```go
// Good -- explicit value, use :=
timeout := 30 * time.Second
name := config.Name()
results := make([]string, 0, 10)

// Good -- zero value is intentional, use var
var count int
var buf bytes.Buffer
var mu sync.Mutex
```
2. Never invert the two. `var x = value` with an explicit value adds a keyword that buys nothing over `:=`, and `:=` for a zero value hides that the zero is deliberate behind a redundant literal.

```go
// Bad -- var with explicit value
var timeout = 30 * time.Second

// Bad -- := for zero value
count := 0
buf := bytes.Buffer{}
```
3. Stated the other way, for non-zero initialization prefer `:=` over `var i = 42`, and for zero-value declarations prefer `var count int` over `count := 0`. The two forms split cleanly along the explicit-versus-zero line; keep them on their own sides.

**Enforcement:** golangci-lint; review.

### 12.2 — Omit redundant types on top-level declarations; keep them when the type differs.

**Reasoning, step by step:**
1. For top-level variables, use `var` without a type when the right-hand expression already produces the type you want — the annotation just repeats what the expression states.

```go
// Good -- type matches expression, omit redundant type
var logger = slog.Default()
var defaultTransport = http.DefaultTransport

// Bad -- redundant type annotation
var logger *slog.Logger = slog.Default()
```
2. Specify the type explicitly when the desired type differs from the expression's type — the annotation now carries real information, widening a concrete return to the interface you mean to hold.

```go
// Good -- desired type differs from expression type
var handler http.Handler = newRouter()  // newRouter returns *Router, want http.Handler
```
3. For compile-time interface checks, always specify the type — the whole point of the declaration is to assert that the concrete type satisfies the interface.

```go
var _ http.Handler = (*Server)(nil)
```

**Enforcement:** golangci-lint; review.

### 12.3 — Name struct-literal fields across package boundaries.

**Reasoning, step by step:**
1. Struct literals must specify field names for types defined outside the current package. Positional literals bind silently to declaration order, so a reordered or inserted field in the other package miscompiles or, worse, mis-assigns without a word.

```go
// Good
hotel := otherpkg.Hotel{
    ID:     "123",
    Name:   "Grand",
    Rating: 4.5,
}

// Bad
hotel := otherpkg.Hotel{"123", "Grand", 4.5}
```
2. For package-local types, field names are optional but should be used when they improve clarity — especially when the struct has many fields and position alone stops being legible.
3. Omit zero-value fields when clarity is not harmed. This draws the reader's attention to the fields that matter. Include zero values only when the zero is itself significant — for example, a table-driven test case where `Retries: 0` is the scenario under test.

**Enforcement:** golangci-lint; review.

### 12.4 — Use composite literals for known values and zero-value declarations for empty ones.

**Reasoning, step by step:**
1. Use composite literals when you know the initial elements, and a zero-value declaration when the value starts empty. A `[]string{}` written only to be appended to is a redundant empty literal where `var names []string` says the same with less.

```go
// Good -- known values
sizes := []int{1, 2, 3}

// Good -- zero-value slice, ready for append
var names []string

// Bad -- redundant empty literal
names := []string{}
```
2. Maps must be explicitly initialized before writes — use `make(map[K]V)` or a map literal. Reading from a nil map is safe; writing to one panics, so a map is the one container the zero value does not make ready.
3. Protobuf messages should be declared as pointers (`*pb.Message`) — the pointer form satisfies `proto.Message`.
4. For pointers to zero values, both `new(T)` and `&T{}` are fine. Prefer `new(bytes.Buffer)` when the reader should be reminded that a non-zero value requires a constructor.

**Enforcement:** golangci-lint; review.

### 12.5 — Give `make` a size hint only when the final size is known.

**Reasoning, step by step:**
1. Provide a size hint to `make` only when the final size is known — for example, allocating a slice to hold N elements drawn from a loop of known length. There the hint avoids repeated regrowth.
2. Excess preallocation wastes memory and can harm performance, so a guessed or generous hint is worse than none. Most code does not need a size hint at all.

**Enforcement:** review.

### 12.6 — Specify channel direction on declarations.

**Reasoning, step by step:**
1. When a declaration involves a channel parameter, specify its direction (`chan<-`, `<-chan`) where possible — it prevents programming errors and conveys ownership, letting the type say which end of the channel this code is allowed to use. See `04-concurrency.md` for the full treatment.

**Enforcement:** go vet; review.

### 12.7 — Treat `nil` as a valid empty slice.

**Reasoning, step by step:**
1. `nil` is a valid, empty, zero-length slice. It supports `len`, `cap`, `append`, and `range` without allocation, so a function that may produce no elements should return `nil` rather than an allocated empty slice.

```go
// Good -- return nil
func filter(items []string, predicate func(string) bool) []string {
    var result []string // nil, will be allocated on first append
    for _, item := range items {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result // nil if nothing matched -- this is fine
}

// Bad -- unnecessary allocation
func filter(items []string, predicate func(string) bool) []string {
    result := []string{} // allocates even when no matches
    for _, item := range items {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}
```
2. Check emptiness with `len`, never by comparing to `nil` — a nil slice and an empty slice are both "empty", so a `nil` comparison answers the wrong question.

```go
// Good
if len(items) == 0 { ... }

// Bad -- nil slice and empty slice are both "empty"
if items == nil { ... }
```
3. The one place `nil` and `[]string{}` diverge is JSON serialization: `nil` marshals to `null`, `[]string{}` marshals to `[]`. When that distinction matters for an API contract, allocate explicitly.

**Enforcement:** review.

### 12.8 — Do not shadow standard-library package names.

**Reasoning, step by step:**
1. Do not introduce local variable names that shadow standard library package names — a variable named `url` blocks access to `net/url` for the rest of the scope. Choose a different name, or rename the import.

**Enforcement:** golangci-lint; review.

### 12.9 — Declare variables in the narrowest scope that stays readable.

**Reasoning, step by step:**
1. Declare variables in the narrowest scope possible. Use the `if`-init form when the result is only needed inside the `if` block, so the binding does not leak past the one place that reads it.

```go
// Good -- err is scoped to the if block
if err := validate(input); err != nil {
    return fmt.Errorf("validate: %w", err)
}

// Bad -- err leaks into the outer scope unnecessarily
err := validate(input)
if err != nil {
    return fmt.Errorf("validate: %w", err)
}
// err is still in scope here but serves no purpose
```
2. Keep constants local to the function that uses them rather than at package level, so a single-use constant declares itself next to its one reader.

```go
// Good -- constant scoped to the function
func classify(score int) string {
    const threshold = 80
    if score >= threshold {
        return "pass"
    }
    return "fail"
}

// Bad -- package-level constant used in one function
const threshold = 80

func classify(score int) string { ... }
```
3. Do not reduce scope if it increases nesting. Readability wins over minimal scope — an `if`/`else` that traps a value inside the `else` to keep it scoped reads worse than a slightly wider declaration with an early return.

```go
// Good -- clear despite slightly wider scope
data, err := fetch(ctx, url)
if err != nil {
    return err
}
process(data)

// Bad -- reduced scope at the cost of nesting
if data, err := fetch(ctx, url); err != nil {
    return err
} else {
    process(data) // data is trapped inside else
}
```

**Enforcement:** golangci-lint; review.

### 12.10 — Clarify naked boolean and numeric parameters.

**Reasoning, step by step:**
1. Boolean and numeric parameters without names are ambiguous at the call site — a bare `true, true` says nothing about what each argument controls. Use comments or custom types to clarify intent.
2. For infrequent call sites, use C-style inline comments so the meaning rides along with the argument.

```go
// Good -- parameter purpose is clear
printInfo("config.yaml", true /* isLocal */, true /* verbose */)

// Bad -- what do these booleans mean?
printInfo("config.yaml", true, true)
```
3. For frequently used parameters, define a custom type. It is self-documenting at every call site rather than at the ones a writer remembers to annotate.

```go
// Better -- self-documenting at every call site
type Verbosity int

const (
    VerbosityQuiet   Verbosity = iota
    VerbosityNormal
    VerbosityVerbose
)

func printInfo(path string, isLocal bool, verbosity Verbosity) { ... }

printInfo("config.yaml", true, VerbosityVerbose)
```
4. This matters most for adjacent parameters of the same type, where argument order is easy to swap and the compiler cannot catch the transposition.

**Enforcement:** review.

### 12.11 — Use raw string literals to avoid hand-escaping.

**Reasoning, step by step:**
1. Use backtick strings (raw string literals) to avoid hand-escaped characters. Raw strings are easier to read and less error-prone, since the content appears verbatim instead of behind a wall of backslashes and escaped quotes.

```go
// Good -- no escaping needed
pattern := `\d{3}-\d{2}-\d{4}`
query := `SELECT * FROM hotels WHERE name = $1 AND rating >= $2`
message := `He said "hello" and she replied "hi"`

// Bad -- escaping obscures the content
pattern := "\\d{3}-\\d{2}-\\d{4}"
query := "SELECT * FROM hotels WHERE name = $1 AND rating >= $2"
message := "He said \"hello\" and she replied \"hi\""
```
2. Raw strings cannot contain backticks. When a string must include a backtick, use a regular string literal with escaping, or concatenate the pieces.

**Enforcement:** review.

## Cross-references

- Channel direction in full, ownership of channel ends: [`04-concurrency.md`](./04-concurrency.md).
