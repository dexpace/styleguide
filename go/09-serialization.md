# 09 - Serialization

## JSON

Use `encoding/json` from the standard library. Use struct tags for field mapping:

```go
type Hotel struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Rating    float64   `json:"rating"`
    CreatedAt time.Time `json:"created_at"`
}
```

## omitempty Rules

Use `omitempty` for optional fields. Never use on required fields -- it hides bugs:

```go
type SearchParams struct {
    City     string  `json:"city"`                // required -- no omitempty
    Adults   int     `json:"adults,omitempty"`     // optional
    SortBy   string  `json:"sort_by,omitempty"`   // optional
}
```

If a required field marshals to its zero value, the absence from the output signals a bug. `omitempty` silences that signal.

## Custom Marshaling

Implement `json.Marshaler`/`json.Unmarshaler` for types with special serialization:

```go
type Money struct {
    Amount   string // decimal as string, never float
    Currency string
}

func (m Money) MarshalJSON() ([]byte, error) {
    return json.Marshal(struct {
        Amount   string `json:"amount"`
        Currency string `json:"currency"`
    }{Amount: m.Amount, Currency: m.Currency})
}
```

## Time Handling

### Use `time.Time` and `time.Duration`

Use `time.Time` for instants and `time.Duration` for elapsed time. Never use raw integers for time values -- they force every reader to guess the unit.

```go
// Good -- type carries the unit
func NewPoller(interval time.Duration) *Poller { ... }

poller := NewPoller(10 * time.Second)

// Bad -- is this seconds? milliseconds? nanoseconds?
func NewPoller(interval int) *Poller { ... }

poller := NewPoller(10)
```

### Comparing Time

Use `time.Time` methods for comparison, never `==`:

```go
// Good
if deadline.Before(time.Now()) { ... }
if t1.Equal(t2) { ... }

// Bad -- does not account for monotonic clock
if deadline == time.Now() { ... }
```

### Calendar Time vs Absolute Time

Distinguish between calendar operations and absolute durations:

```go
// "Same time tomorrow" (calendar) -- handles DST changes
newDay := now.AddDate(0, 0, 1)

// "Exactly 24 hours from now" (absolute)
next := now.Add(24 * time.Hour)
```

### External Systems

When interfacing with systems that don't support `time.Time` or `time.Duration`, include the unit in the field name:

```go
// Good -- unit is explicit
type ExternalConfig struct {
    IntervalMillis int    `json:"interval_millis"`
    CreatedAtUnix  int64  `json:"created_at_unix"`
}

// Bad -- ambiguous
type ExternalConfig struct {
    Interval  int   `json:"interval"`
    CreatedAt int64 `json:"created_at"`
}
```

Use RFC 3339 format for timestamp strings in APIs. It is the standard Go serialization format and widely supported.

### Date Serialization

Always use `time.Time`. Serialize as RFC 3339 (ISO 8601). For date-only fields, use a custom type:

```go
type Date struct{ time.Time }

func (d Date) MarshalJSON() ([]byte, error) {
    return json.Marshal(d.Format("2006-01-02"))
}

func (d *Date) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    t, err := time.Parse("2006-01-02", s)
    if err != nil {
        return fmt.Errorf("parse date %q: %w", s, err)
    }
    d.Time = t
    return nil
}
```

## Money and Decimals

Never use `float64` for money. Use string representation or `shopspring/decimal`. Parse from strings, serialize to strings. IEEE 754 floating-point cannot represent most decimal fractions exactly.

## Validation After Unmarshal

Always validate after `json.Unmarshal`. The decoder doesn't know your business rules:

```go
if err := json.Unmarshal(data, &hotel); err != nil {
    return fmt.Errorf("unmarshal hotel: %w", err)
}
if hotel.ID == "" {
    return fmt.Errorf("hotel ID is required")
}
if hotel.Rating < 0 || hotel.Rating > 5 {
    return fmt.Errorf("hotel rating must be 0-5, got %f", hotel.Rating)
}
```

## Never Trust External Data

Every byte from outside the process is suspect. Validate shape, types, ranges, and lengths after deserialization. Assume the sender is hostile.

## Streaming

For large payloads, use `json.Decoder` with `DisallowUnknownFields()`:

```go
decoder := json.NewDecoder(r.Body)
decoder.DisallowUnknownFields()
if err := decoder.Decode(&req); err != nil {
    return fmt.Errorf("decode request: %w", err)
}
```

Use `json.Decoder` over `json.Unmarshal` when reading from an `io.Reader` -- it avoids buffering the entire payload into memory.

## Use `%q` for Quoted Strings

Prefer `%q` over hand-quoting with `\"...\"` or single quotes. `%q` handles empty strings, control characters, and unicode correctly.

```go
// Good
return fmt.Errorf("parse %q: %w", raw, err)

// Bad
return fmt.Errorf("parse \"%s\": %w", raw, err)
```

This is especially important for human-facing output where the input may be empty, contain whitespace, or contain control characters.

## fmt.Stringer

Implement `fmt.Stringer` on types that benefit from a human-readable representation. This is used by `%s` and `%v` in format strings:

```go
type Money struct {
    Amount   string
    Currency string
}

func (m Money) String() string {
    return m.Amount + " " + m.Currency
}
```

Use `%v` for default formatting, `%+v` for struct field names, `%#v` for Go syntax, and `%T` for the type name. These are valuable in logging and debugging.

## Initializing Structs

Always use field names in struct literals. Positional initialization breaks silently when fields are added or reordered.

```go
// Good -- field names make intent explicit
hotel := Hotel{
    ID:     "123",
    Name:   "Grand Hotel",
    Rating: 4.5,
}

// Bad -- positional, breaks if a field is inserted
hotel := Hotel{"123", "Grand Hotel", 4.5}
```

Omit zero-value fields unless they add meaningful context. In test tables, explicit zero values can improve readability:

```go
// Good -- zero values omitted in production code
config := ClientConfig{
    BaseURL: "https://api.example.com",
    Timeout: 30 * time.Second,
    // Retries defaults to 0, which is correct
}

// Good -- explicit zero in test table for clarity
{
    name:    "no retries",
    config:  ClientConfig{Retries: 0},
    wantErr: true,
}
```

For empty struct declarations, use `var` instead of an empty literal:

```go
// Good
var user User

// Bad -- empty literal adds no information
user := User{}
```

## Initializing Maps

Use `make()` for maps that will be populated programmatically. Use map literals for maps with a fixed set of elements:

```go
// Good -- populated programmatically
seen := make(map[string]bool)
for _, item := range items {
    seen[item.ID] = true
}

// Good -- fixed elements
statusText := map[int]string{
    200: "OK",
    404: "Not Found",
    500: "Internal Server Error",
}

// Bad -- empty literal for a programmatic map
seen := map[string]bool{}
```

When the map size is known or estimable, provide a capacity hint:

```go
// Good -- capacity hint reduces allocations
lookup := make(map[string]*Hotel, len(hotels))
for _, h := range hotels {
    lookup[h.ID] = h
}
```

Declare nil maps for maps that are only read (e.g., used in a lookup). Nil maps behave identically to empty maps for reads but panic on writes -- this catches accidental mutation.

## Composite Literals

Prefer composite literals over `new()` + field assignment:

```go
// Good
hotel := &Hotel{
    ID:     "123",
    Name:   "Grand Hotel",
    Rating: 4.5,
}

// Bad
hotel := new(Hotel)
hotel.ID = "123"
hotel.Name = "Grand Hotel"
hotel.Rating = 4.5
```

For slices and maps, provide size hints when the size is known or estimable:

```go
// Good -- pre-allocates capacity
names := make([]string, 0, len(hotels))
for _, h := range hotels {
    names = append(names, h.Name)
}

// Bad -- grows dynamically through multiple allocations
var names []string
for _, h := range hotels {
    names = append(names, h.Name)
}
```

## Format Strings outside Printf

Declare format strings as `const` when they are used outside of a direct `fmt.*` call. This enables `go vet` to perform static analysis on the format string:

```go
// Good -- go vet can analyze the format
const userFormat = "user %s (ID: %d)"

log.Printf(userFormat, name, id)

// Bad -- format string in a variable, go vet cannot check
var userFormat = "user %s (ID: %d)"

log.Printf(userFormat, name, id)
```

This matters most for logging and error formatting where a mismatched format verb is a runtime bug, not a compile-time error.
