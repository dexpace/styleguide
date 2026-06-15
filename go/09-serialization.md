# 09 — Serialization

Bytes crossing the process boundary are untyped and untrusted. This chapter governs how Go turns structs into JSON and back: struct tags for field mapping, `omitempty` only where absence is legal, custom marshalers for types with special wire forms, `time.Time`/`time.Duration` so units never get guessed, and strings for money so IEEE 754 never rounds a price. Every decoded value is validated at the boundary before the domain sees it. It also covers the construction idioms — field-named literals, sized maps and slices, composite literals over `new()` — that keep the in-memory values these formats carry honest.

## What good looks like

```go
// Hotel is the wire DTO. Every field is tagged explicitly; optional fields
// carry omitempty, required fields never do.
type Hotel struct {
    ID        string    `json:"id"`                  // required -- no omitempty
    Name      string    `json:"name"`                // required -- no omitempty
    Rating    float64   `json:"rating"`              // required -- no omitempty
    SortBy    string    `json:"sort_by,omitempty"`   // optional
    CreatedAt time.Time `json:"created_at"`          // RFC 3339 on the wire
}

func DecodeHotel(r io.Reader) (Hotel, error) {
    var hotel Hotel
    decoder := json.NewDecoder(r)
    decoder.DisallowUnknownFields()
    if err := decoder.Decode(&hotel); err != nil {
        return Hotel{}, fmt.Errorf("decode hotel: %w", err)
    }
    // The decoder does not know the business rules. Validate at the boundary.
    if hotel.ID == "" {
        return Hotel{}, fmt.Errorf("hotel ID is required")
    }
    if hotel.Rating < 0 || hotel.Rating > 5 {
        return Hotel{}, fmt.Errorf("hotel rating must be 0-5, got %f", hotel.Rating)
    }
    return hotel, nil
}
```

This DTO maps every field with a struct tag and keeps `omitempty` off required fields (9.1, 9.2); `CreatedAt` is a `time.Time` serialized as RFC 3339, not a raw integer (9.4); the decoder streams from an `io.Reader` with `DisallowUnknownFields()` (9.8) and wraps its failure with `%w`; the boundary validates shape and ranges before returning, trusting nothing from outside the process (9.6, 9.7). A `Money` field on this struct would be a string, never a `float64` (9.5).

## Rules

### 9.1 — Map JSON fields with explicit struct tags.

**Reasoning, step by step:**
1. Use `encoding/json` from the standard library — it is the canonical Go serializer and needs no third-party dependency.
2. Map every field to its wire name with a struct tag rather than relying on Go's default field-name casing. The tag is the contract between the struct and the JSON; making it explicit means a field rename in Go never silently changes the wire format.

```go
type Hotel struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Rating    float64   `json:"rating"`
    CreatedAt time.Time `json:"created_at"`
}
```

**Enforcement:** review; `encoding/json` with explicit `json:` tags on serialized structs.

### 9.2 — Use `omitempty` only on optional fields, never required ones.

**Reasoning, step by step:**
1. Use `omitempty` for optional fields so a zero value drops out of the output cleanly.
2. Never put `omitempty` on a required field — it hides bugs. If a required field marshals to its zero value, the absence from the output is the signal that something is wrong; `omitempty` silences that signal.

```go
type SearchParams struct {
    City     string  `json:"city"`                // required -- no omitempty
    Adults   int     `json:"adults,omitempty"`     // optional
    SortBy   string  `json:"sort_by,omitempty"`   // optional
}
```

**Enforcement:** review; `omitempty` on optional fields only, never on required fields.

### 9.3 — Implement `json.Marshaler`/`json.Unmarshaler` for types with special serialization.

**Reasoning, step by step:**
1. When a type's wire form differs from its in-memory layout, implement `json.Marshaler`/`json.Unmarshaler` so the special handling lives with the type instead of leaking into every call site.
2. Marshal through an explicit anonymous struct with its own tags, so the serialized shape is visible and controlled at the type, not inferred from the in-memory fields.

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

**Enforcement:** review; `json.Marshaler`/`json.Unmarshaler` for types whose wire form differs from their layout.

### 9.4 — Carry time as `time.Time`/`time.Duration`, serialize as RFC 3339.

**Reasoning, step by step:**
1. Use `time.Time` for instants and `time.Duration` for elapsed time. Never use raw integers for time values — they force every reader to guess the unit.

```go
// Good -- type carries the unit
func NewPoller(interval time.Duration) *Poller { ... }

poller := NewPoller(10 * time.Second)

// Bad -- is this seconds? milliseconds? nanoseconds?
func NewPoller(interval int) *Poller { ... }

poller := NewPoller(10)
```

2. Compare times with `time.Time` methods, never `==`. The `==` operator does not account for the monotonic clock.

```go
// Good
if deadline.Before(time.Now()) { ... }
if t1.Equal(t2) { ... }

// Bad -- does not account for monotonic clock
if deadline == time.Now() { ... }
```

3. Distinguish calendar operations from absolute durations. `AddDate` walks the calendar and handles DST changes; `Add` advances by a fixed duration.

```go
// "Same time tomorrow" (calendar) -- handles DST changes
newDay := now.AddDate(0, 0, 1)

// "Exactly 24 hours from now" (absolute)
next := now.Add(24 * time.Hour)
```

4. When interfacing with systems that don't support `time.Time` or `time.Duration`, include the unit in the field name so the ambiguity dies at the boundary.

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

5. Use RFC 3339 (ISO 8601) format for timestamp strings in APIs — it is the standard Go serialization format and is widely supported. For date-only fields, always keep the value as a `time.Time` behind a custom type that serializes the date portion.

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

**Enforcement:** review; `time.Time`/`time.Duration` for time values, RFC 3339 on the wire, units in field names at non-`time` boundaries.

### 9.5 — Never use `float64` for money.

**Reasoning, step by step:**
1. Never use `float64` for money. IEEE 754 floating-point cannot represent most decimal fractions exactly, so arithmetic silently accumulates rounding error.
2. Use a string representation or `shopspring/decimal`. Parse from strings and serialize to strings, so the decimal value crosses every boundary intact.

**Enforcement:** review; string or `shopspring/decimal` for monetary values, never `float64`.

### 9.6 — Validate after every `json.Unmarshal`.

**Reasoning, step by step:**
1. Always validate after `json.Unmarshal`. The decoder enforces the JSON shape but knows nothing about your business rules — a syntactically valid payload can still be semantically wrong.
2. Check the fields the domain depends on: required identifiers present, numeric values within range. Wrap the decode error with `%w` so the chain survives.

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

**Enforcement:** review; validation of required fields and ranges after every unmarshal.

### 9.7 — Never trust external data.

**Reasoning, step by step:**
1. Every byte from outside the process is suspect. Assume the sender is hostile.
2. After deserialization, validate shape, types, ranges, and lengths before the value reaches domain logic.

**Enforcement:** review; shape, type, range, and length validation on all externally sourced data.

### 9.8 — Stream large payloads with `json.Decoder` and `DisallowUnknownFields()`.

**Reasoning, step by step:**
1. For large payloads, use `json.Decoder` rather than `json.Unmarshal`. When reading from an `io.Reader` it decodes the stream directly instead of buffering the entire payload into memory.
2. Call `DisallowUnknownFields()` so an unexpected field is an error, not silently dropped — the strictness catches client/server drift at the boundary.

```go
decoder := json.NewDecoder(r.Body)
decoder.DisallowUnknownFields()
if err := decoder.Decode(&req); err != nil {
    return fmt.Errorf("decode request: %w", err)
}
```

**Enforcement:** review; `json.Decoder` with `DisallowUnknownFields()` for reads from an `io.Reader`.

### 9.9 — Use `%q` for quoted strings.

**Reasoning, step by step:**
1. Prefer `%q` over hand-quoting with `\"...\"` or single quotes. `%q` handles empty strings, control characters, and unicode correctly, where hand-quoting silently mangles all three.

```go
// Good
return fmt.Errorf("parse %q: %w", raw, err)

// Bad
return fmt.Errorf("parse \"%s\": %w", raw, err)
```

2. This matters most for human-facing output where the input may be empty, contain whitespace, or contain control characters.

**Enforcement:** review; `%q` for quoting string values in format strings.

### 9.10 — Implement `fmt.Stringer` for human-readable types.

**Reasoning, step by step:**
1. Implement `fmt.Stringer` on types that benefit from a human-readable representation. `%s` and `%v` use it, so logging and debugging output stays legible.

```go
type Money struct {
    Amount   string
    Currency string
}

func (m Money) String() string {
    return m.Amount + " " + m.Currency
}
```

2. Choose the formatting verb deliberately: `%v` for default formatting, `%+v` for struct field names, `%#v` for Go syntax, and `%T` for the type name. These are valuable in logging and debugging.

**Enforcement:** review; `fmt.Stringer` on types with a meaningful human-readable form.

### 9.11 — Initialize structs with field names.

**Reasoning, step by step:**
1. Always use field names in struct literals. Positional initialization breaks silently when fields are added or reordered.

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

2. Omit zero-value fields unless they add meaningful context. In test tables, explicit zero values can improve readability.

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

3. For empty struct declarations, use `var` instead of an empty literal — the empty value-type literal `T{}` adds no information.

```go
// Good
var user User

// Bad -- empty literal adds no information
user := User{}
```

**Enforcement:** review; field-named struct literals, `var` over empty `T{}` literals.

### 9.12 — Initialize maps by purpose: `make()` to populate, literals for fixed sets.

**Reasoning, step by step:**
1. Use `make()` for maps that will be populated programmatically, and map literals for maps with a fixed set of elements. An empty literal `map[K]V{}` for a programmatic map is the wrong tool — say `make()`.

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

2. When the map size is known or estimable, provide a capacity hint to reduce allocations.

```go
// Good -- capacity hint reduces allocations
lookup := make(map[string]*Hotel, len(hotels))
for _, h := range hotels {
    lookup[h.ID] = h
}
```

3. Declare nil maps for maps that are only read (e.g., used in a lookup). Nil maps behave identically to empty maps for reads but panic on writes — this catches accidental mutation.

**Enforcement:** review; `make()` for programmatic maps, literals for fixed sets, capacity hints where the size is known.

### 9.13 — Prefer composite literals over `new()`, with size hints for slices and maps.

**Reasoning, step by step:**
1. Prefer composite literals over `new()` followed by field assignment. The literal states the whole value in one expression instead of mutating it field by field.

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

2. For slices and maps, provide size hints when the size is known or estimable, so the value does not grow through multiple reallocations.

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

**Enforcement:** review; composite literals over `new()`, size hints for slices and maps with known length.

### 9.14 — Declare format strings used outside `fmt.*` as `const`.

**Reasoning, step by step:**
1. Declare format strings as `const` when they are used outside of a direct `fmt.*` call. This lets `go vet` perform static analysis on the format string; a `var` format string is opaque to the analyzer.

```go
// Good -- go vet can analyze the format
const userFormat = "user %s (ID: %d)"

log.Printf(userFormat, name, id)

// Bad -- format string in a variable, go vet cannot check
var userFormat = "user %s (ID: %d)"

log.Printf(userFormat, name, id)
```

2. This matters most for logging and error formatting, where a mismatched format verb is a runtime bug, not a compile-time error.

**Enforcement:** `go vet`; `const` format strings used outside direct `fmt.*` calls.

## Cross-references

- Zero values via pointer composite literals (`&T{}`), struct construction, and declaration idioms: [12 - Variables and Declarations](./12-variables-and-declarations.md).
- Error wrapping with `%w` and `fmt.Errorf`: [03 - Error Handling](./03-error-handling.md).
- API boundaries and DTO design: [05 - API Design](./05-api-design.md).
