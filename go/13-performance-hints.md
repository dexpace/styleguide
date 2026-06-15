# 13 — Performance Hints

Performance is part of correctness for production services, not an afterthought bolted on later. This chapter targets the allocation and conversion pitfalls that are cheap to avoid and visible in benchmarks: prefer `strconv` over `fmt`, convert strings to bytes once, keep small values on the stack, hint collection capacity, and build strings with `strings.Builder`. Measure before you optimize, and spend the effort on hot paths only.

## What good looks like

```go
// formatHotelNames serializes hotel names into a single comma-separated line.
// Hot path: called per request in the listing handler.
func formatHotelNames(hotels []Hotel) string {
    // Capacity hint sized from the input -- no grow-and-copy cycles (13.4).
    names := make([]string, 0, len(hotels))
    for _, h := range hotels {
        names = append(names, h.Name)
    }

    var builder strings.Builder // zero value is ready to use
    for i, name := range names {
        if i > 0 {
            builder.WriteByte(',') // strconv/Builder, no fmt in the loop (13.1, 13.6)
        }
        builder.WriteString(name)
    }
    return builder.String()
}

// NewPoint returns a value, letting escape analysis keep it on the stack (13.3).
func NewPoint(x, y int) Point {
    return Point{X: x, Y: y}
}
```

This exemplar preallocates the slice with a size hint so `append` never reallocates (13.4), builds the result with `strings.Builder` instead of `+` in a loop (13.6), keeps formatting out of the loop in favor of `WriteByte`/`WriteString` rather than `fmt` (13.1), and returns the small `Point` by value so it stays on the stack (13.3). The hot-path comment records *why* the optimization is justified -- the rules below all defer to measurement before applying them to cold paths.

## Rules

### 13.1 — Prefer `strconv` over `fmt` for primitive conversions.

**Reasoning, step by step:**

1. `strconv` is significantly faster than `fmt` for primitive-to-string and string-to-primitive conversions. Avoid `fmt.Sprintf` when a type-specific `strconv` function exists.

```go
// Good -- fast (~64ns/op)
s := strconv.Itoa(42)
s := strconv.FormatFloat(3.14, 'f', -1, 64)
s := strconv.FormatBool(true)

n, err := strconv.Atoi("42")
f, err := strconv.ParseFloat("3.14", 64)

// Bad -- slow (~143ns/op), allocates more
s := fmt.Sprintf("%d", 42)
s := fmt.Sprintf("%f", 3.14)
s := fmt.Sprintf("%t", true)
```

2. The performance difference compounds in hot paths -- request handlers, serialization loops, metric formatting.
3. Use `fmt.Sprintf` only when you need composite formatting with multiple verbs.

**Enforcement:** `go test -bench`; review.

### 13.2 — Convert strings to bytes once, then reuse.

**Reasoning, step by step:**

1. Each `[]byte("...")` conversion allocates. Perform the conversion once and reuse the result.

```go
// Good -- convert once, reuse
var separator = []byte(",")

func writeItems(w io.Writer, items []string) error {
    for i, item := range items {
        if i > 0 {
            if _, err := w.Write(separator); err != nil {
                return err
            }
        }
        if _, err := w.Write([]byte(item)); err != nil {
            return err
        }
    }
    return nil
}

// Bad -- converts on every iteration
func writeItems(w io.Writer, items []string) error {
    for i, item := range items {
        if i > 0 {
            if _, err := w.Write([]byte(",")); err != nil {
                return err
            }
        }
        if _, err := w.Write([]byte(item)); err != nil {
            return err
        }
    }
    return nil
}
```

2. This matters in tight loops.
3. For one-off conversions outside hot paths, inline `[]byte("...")` is fine.

**Enforcement:** `pprof`; `go test -bench`; review.

### 13.3 — Keep small values on the stack, not the heap.

**Reasoning, step by step:**

1. Go's escape analysis determines whether a value lives on the stack (cheap) or the heap (requires GC). Help the compiler keep values on the stack by:
   - Returning values instead of pointers when the value is small.
   - Using fixed-size arrays instead of slices when the size is known at compile time.
   - Avoiding capturing variables in closures that escape the function.

```go
// Good -- Point stays on the stack
func NewPoint(x, y int) Point {
    return Point{X: x, Y: y}
}

// Potentially worse -- forces heap allocation
func NewPoint(x, y int) *Point {
    return &Point{X: x, Y: y}
}
```

2. Use `go build -gcflags='-m'` to inspect escape analysis decisions when optimizing hot paths.
3. Don't micro-optimize cold paths.

**Enforcement:** `go build -gcflags='-m'`; `pprof`; review.

### 13.4 — Give `make` a capacity hint when the size is known.

**Reasoning, step by step:**

1. Provide capacity hints to `make` when the size is known or estimable. This eliminates grow-and-copy allocations.

```go
// Good -- no reallocation needed
names := make([]string, 0, len(hotels))
for _, h := range hotels {
    names = append(names, h.Name)
}

// Good -- map hint reduces rehashing
lookup := make(map[string]*Hotel, len(hotels))

// Bad -- multiple grow-and-copy cycles
var names []string
for _, h := range hotels {
    names = append(names, h.Name)
}
```

2. Map capacity hints are approximate -- the runtime may allocate more.
3. Overestimating is better than no hint at all.

**Enforcement:** `go test -bench`; `pprof`; review.

### 13.5 — Skip `fmt.Errorf` in extremely hot error paths.

**Reasoning, step by step:**

1. `fmt.Errorf` uses reflection internally. In extremely hot error paths, consider `errors.New` with string concatenation.

```go
// Hot path -- avoid fmt overhead
if id == "" {
    return errors.New("id is required")
}

// Normal path -- fmt.Errorf is fine, readability wins
if err != nil {
    return fmt.Errorf("fetch hotel %s: %w", id, err)
}
```

2. This is a micro-optimization. Use `fmt.Errorf` by default and only switch to `errors.New` when benchmarks show it matters.

**Enforcement:** `go test -bench`; review.

### 13.6 — Build strings with `strings.Builder`, not repeated `+`.

**Reasoning, step by step:**

1. Use `strings.Builder` for building strings in a loop. It minimizes allocations compared to `+` or `fmt.Sprintf`.

```go
// Good -- O(n) allocations
var builder strings.Builder
for _, item := range items {
    builder.WriteString(item)
    builder.WriteByte(',')
}
result := builder.String()

// Bad -- O(n^2) allocations, each + creates a new string
var result string
for _, item := range items {
    result += item + ","
}
```

2. When the total size is known, call `builder.Grow(n)` upfront to avoid intermediate allocations entirely.

**Enforcement:** `go test -bench`; `golangci-lint`; review.

## Cross-references

- Error construction with `errors.New`, `fmt.Errorf`, and `%w` wrapping: [error handling](./03-error-handling.md).
- Benchmarking discipline and `go test -bench`: [testing](./06-testing.md).
