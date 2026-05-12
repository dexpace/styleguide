# 12 - Variables & Declarations

## Local Variable Declarations

Use short variable declarations (`:=`) when assigning an explicit value. Use `var` when the zero value is intentional:

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

Never use `var` with an explicit value or `:=` for zero values:

```go
// Bad -- var with explicit value
var timeout = 30 * time.Second

// Bad -- := for zero value
count := 0
buf := bytes.Buffer{}
```

## Top-level Variable Declarations

For top-level variables, use `var` without specifying the type when the right-hand expression already produces the correct type. Specify the type explicitly when the desired type differs from the expression's type:

```go
// Good -- type matches expression, omit redundant type
var logger = slog.Default()
var defaultTransport = http.DefaultTransport

// Good -- desired type differs from expression type
var handler http.Handler = newRouter()  // newRouter returns *Router, want http.Handler

// Bad -- redundant type annotation
var logger *slog.Logger = slog.Default()
```

For compile-time interface checks, always specify the type:

```go
var _ http.Handler = (*Server)(nil)
```

## Struct Literals: Field Names

Struct literals **must specify field names** for types defined outside the current package:

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

For package-local types, field names are optional but should be used when they improve clarity — especially when the struct has many fields.

**Omit zero-value fields** when clarity is not harmed. This draws the reader's attention to the fields that matter. Include zero values only when the zero is itself significant (e.g., a table-driven test case where `Retries: 0` is the scenario under test).

## Initialization: `:=` vs `var`

Prefer `:=` over `var = ...` when initializing with a non-zero value:

```go
// Good
i := 42
timeout := 30 * time.Second

// Bad
var i = 42
```

Use `var` for zero-value declarations:

```go
// Good
var count int
var buf bytes.Buffer

// Bad
count := 0
buf := bytes.Buffer{}
```

## Composite Literals for Known Initial Values

Use composite literals when you know initial elements. Use zero-value declarations for empty values:

```go
// Good -- known values
sizes := []int{1, 2, 3}

// Good -- zero-value slice, ready for append
var names []string

// Bad -- redundant empty literal
names := []string{}
```

**Maps must be explicitly initialized** before writes — use `make(map[K]V)` or a map literal. Reading from a nil map is safe; writing panics.

**Protobuf messages** should be declared as pointers (`*pb.Message`) — the pointer form satisfies `proto.Message`.

For pointers to zero values, both `new(T)` and `&T{}` are fine. Prefer `new(bytes.Buffer)` when the reader should be reminded that a non-zero value requires a constructor.

## Size Hints on `make`

Provide a size hint to `make` **only when** the final size is known (e.g., allocating a slice to hold N elements from a loop of known length). Excess preallocation wastes memory and can harm performance.

Most code does not need a size hint.

## Channel Direction

Specify channel direction in function parameters where possible. This prevents programming errors and conveys ownership:

```go
// Good
func produce(out chan<- Event) { ... }
func consume(in <-chan Event) { ... }

// Bad
func produce(out chan Event) { ... }
```

## Shadowing Package Names

Do not introduce local variable names that shadow standard library package names — a variable named `url` blocks access to `net/url` for the rest of the scope. Choose a different name, or rename the import.

## nil is a Valid Slice

`nil` is a valid, empty, zero-length slice. It supports `len`, `cap`, `append`, and `range` without allocation. Return `nil` instead of an allocated empty slice when there are no elements:

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

Check emptiness with `len`, never compare to `nil`:

```go
// Good
if len(items) == 0 { ... }

// Bad -- nil slice and empty slice are both "empty"
if items == nil { ... }
```

**Note**: `nil` and `[]string{}` differ in JSON serialization -- `nil` marshals to `null`, `[]string{}` marshals to `[]`. When the distinction matters for API contracts, allocate explicitly.

## Reduce Scope of Variables

Declare variables in the narrowest scope possible. Use the `if`-init form when the result is only needed inside the `if` block:

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

Keep constants local to the function that uses them:

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

Don't reduce scope if it increases nesting. Readability wins over minimal scope:

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

## Avoid Naked Parameters

Boolean and numeric parameters without names are ambiguous at the call site. Use comments or custom types to clarify intent.

For infrequent call sites, use C-style inline comments:

```go
// Good -- parameter purpose is clear
printInfo("config.yaml", true /* isLocal */, true /* verbose */)

// Bad -- what do these booleans mean?
printInfo("config.yaml", true, true)
```

For frequently used parameters, define a custom type:

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

This is especially important for adjacent parameters of the same type, where argument order is easy to swap.

## Use Raw String Literals

Use backtick strings (raw string literals) to avoid hand-escaped characters. Raw strings are easier to read and less error-prone:

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

Raw strings cannot contain backticks. When a string must include backticks, use a regular string literal with escaping or string concatenation.
