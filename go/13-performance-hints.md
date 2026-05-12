# 13 - Performance Hints

Performance is not an afterthought -- it is part of correctness for production services. These guidelines address common performance pitfalls that are easy to avoid and measurable in benchmarks.

## Prefer strconv over fmt

`strconv` is significantly faster than `fmt` for primitive-to-string and string-to-primitive conversions. Avoid `fmt.Sprintf` when a type-specific `strconv` function exists.

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

The performance difference compounds in hot paths -- request handlers, serialization loops, metric formatting. Use `fmt.Sprintf` only when you need composite formatting with multiple verbs.

## Avoid Repeated String-to-Byte Conversions

Each `[]byte("...")` conversion allocates. Perform the conversion once and reuse the result:

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

This matters in tight loops. For one-off conversions outside hot paths, inline `[]byte("...")` is fine.

## Prefer Stack over Heap

Go's escape analysis determines whether a value lives on the stack (cheap) or the heap (requires GC). Help the compiler keep values on the stack:

- Return values instead of pointers when the value is small.
- Use fixed-size arrays instead of slices when the size is known at compile time.
- Avoid capturing variables in closures that escape the function.

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

Use `go build -gcflags='-m'` to inspect escape analysis decisions when optimizing hot paths. Don't micro-optimize cold paths.

## Size Hints on Collections

Provide capacity hints to `make` when the size is known or estimable. This eliminates growthcopy allocations:

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

Map capacity hints are approximate -- the runtime may allocate more. Overestimating is better than no hint at all.

## Avoid Unnecessary fmt in Hot Paths

`fmt.Errorf` uses reflection internally. In extremely hot error paths, consider `errors.New` with string concatenation:

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

This is a micro-optimization. Use `fmt.Errorf` by default and only switch to `errors.New` when benchmarks show it matters.

## strings.Builder for Concatenation

Use `strings.Builder` for building strings in a loop. It minimizes allocations compared to `+` or `fmt.Sprintf`:

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

When the total size is known, call `builder.Grow(n)` upfront to avoid intermediate allocations entirely.
