# 05 — Serialization on JVM

Serialization is where types meet the network. Pick libraries that respect Kotlin's type system; configure them to be strict.

## Rules

### 5.1 — `kotlinx.serialization` preferred for internal-controlled JSON. Jackson at Spring boundaries.

**Reasoning, step by step:**
1. `kotlinx.serialization` is compile-time, multiplatform, and respects Kotlin's nullability and default-value semantics natively.
2. Jackson is runtime, JVM-only, and uses reflection — but it's the default for Spring Boot, and the integration is mature.
3. **Rule:**
   - For internal services or modules where you control both ends: `kotlinx.serialization`. Fewer surprises, faster, multiplatform-ready.
   - For Spring Boot HTTP boundaries (controllers, REST clients) and JVM ecosystem integration (Kafka, MongoDB): Jackson with `jackson-module-kotlin`.
4. Don't fight the framework. If Spring uses Jackson and your code is server-side, accept Jackson at the boundary.

### 5.2 — `jackson-module-kotlin` is mandatory if Jackson is on the classpath.

**Reasoning, step by step:**
1. Without `jackson-module-kotlin`, Jackson can't handle Kotlin features: data classes, default arguments, nullability, value classes.
2. Symptoms include: silent null injection into non-null fields, `MissingKotlinParameterException` on missing JSON keys, default values ignored.
3. Register on Spring Boot 3+: it's automatic if the module is on the classpath. Otherwise: `ObjectMapper().registerKotlinModule()`.
4. Verify by writing a test that deserializes JSON missing optional fields. Defaults must fire.

### 5.3 — Time types: `java.time.*`, not `java.util.Date`.

**Reasoning, step by step:**
1. `java.util.Date` is mutable, timezone-confused, and conflates instant-in-time with calendar-date semantics.
2. Use:
   - `Instant` — point in time (UTC). Most server logic.
   - `OffsetDateTime` / `ZonedDateTime` — when timezone is part of the semantic (user-facing schedules).
   - `LocalDate` / `LocalTime` — when the value has no timezone (birthdays, business hours).
3. Configure Jackson: `jackson-datatype-jsr310` + `WRITE_DATES_AS_TIMESTAMPS = false`. Emit ISO-8601 strings, not numbers.
4. For `kotlinx.serialization`: import `kotlinx-serialization-json` and use the `Instant` serializer from `kotlinx-datetime` (Kotlin Multiplatform-friendly) or write a custom serializer for `java.time.Instant`.

### 5.4 — Domain DTOs are `data class`. Never expose entities or domain objects directly.

**Reasoning, step by step:**
1. Entities (JPA, MongoDB documents) are persistence concerns. Domain objects are business logic. Neither is the *wire format*.
2. DTOs (`UserResponse`, `CreateOrderRequest`) are dedicated `data class`es designed for the wire shape.
3. **Decoupling pays off:** schema changes don't break clients, API versions don't leak into the domain, the wire shape can evolve independently.
4. Yes, it's more mapping code. Write the mapping explicitly. (Or use a library — but be aware that mappers like MapStruct add code generation complexity.)

### 5.5 — `null` vs absent: pick a semantics per field and document it.

**Reasoning, step by step:**
1. JSON has both `null` and "missing key." They mean different things; libraries default to different choices.
2. Three semantics:
   - **Required:** must be present. Missing = parse error.
   - **Optional with default:** missing or `null` falls back to the default value.
   - **Optional, nullable:** absent and present-but-null are distinct. Sometimes needed for PATCH semantics (`null` means "set to null"; absent means "don't change").
3. Express in Kotlin:
   - Required: `val name: String` (no default).
   - Optional with default: `val active: Boolean = true`.
   - Optional, nullable, distinguishing absent: model with a wrapper, e.g., `JsonValue<T>(val present: Boolean, val value: T?)`, or use Jackson's `JsonNullable<T>` from OpenAPI extensions.
4. Configure Jackson: `FAIL_ON_UNKNOWN_PROPERTIES = false` for forward-compat reads; `FAIL_ON_NULL_FOR_PRIMITIVES = true` so silent zero-injection doesn't bite.

### 5.6 — `kotlinx.serialization`: `@Serializable` on the data class, `Json` instance per use case.

**Reasoning, step by step:**
1. `@Serializable data class User(val id: UserId, val email: Email)` opts the class into kotlinx serialization.
2. Configure a `Json` instance per use case: `val pretty = Json { prettyPrint = true; encodeDefaults = false }`. Don't share `Json.Default` across modules with different needs.
3. Strict by default: `ignoreUnknownKeys = false` (catches typos), `coerceInputValues = false` (don't silently coerce types).
4. Custom serializers via `@Serializer` for types you don't own (e.g., `java.time.Instant`).

### 5.7 — Polymorphic deserialization: sealed hierarchies, discriminator field.

**Reasoning, step by step:**
1. JSON doesn't carry runtime types. Polymorphic deserialization needs a *discriminator* — usually `type: "Approved"` / `type: "Declined"`.
2. **kotlinx.serialization:** sealed-class polymorphism is first-class. The discriminator is `"type"` by default; configure with `classDiscriminator`.
3. **Jackson:** `@JsonTypeInfo` + `@JsonSubTypes` on the sealed parent. Verbose, but works.
4. Don't roll your own discriminator-handling code. Use the library's machinery; configure it explicitly.

### 5.8 — Validation at the deserialization boundary.

**Reasoning, step by step:**
1. Parsed JSON is *structurally* valid. It may still be *semantically* invalid (empty strings where business rules forbid, IDs that don't match a regex, dates in the past).
2. Validate in the deserializer or right after: Jakarta Bean Validation (`@NotBlank`, `@Pattern`, etc.) on DTOs; Spring's `@Valid` to trigger validation on controller inputs.
3. Validation failures translate to 400 responses via `@ControllerAdvice`.
4. Don't sprinkle validation throughout the domain. Validate at the boundary; downstream code trusts the type.

### 5.9 — Binary formats (Protobuf, Avro, CBOR) at performance-critical or schema-strict boundaries.

**Reasoning, step by step:**
1. JSON is good for human-readable APIs and ad-hoc integration. It's wasteful for high-volume internal traffic and offers no schema.
2. **Protobuf** — schema-first, code-generated, compact wire format. Use for high-volume RPC (gRPC), Kafka topics with strict schemas.
3. **Avro** — schema evolution-friendly. Common in Kafka with Confluent Schema Registry.
4. **CBOR / MessagePack** — binary JSON-shaped formats. Pick if you want JSON semantics with smaller wire size.
5. For Kotlin: `kotlinx.serialization` supports CBOR and Protobuf natively. For Avro: `kotlinx-serialization-avro` (third-party) or Java's Avro libraries.

### 5.10 — Streaming for large payloads. Don't load 10MB into memory.

**Reasoning, step by step:**
1. `objectMapper.readValue(inputStream, ...)` reads the whole stream into memory. For large payloads, this is an allocation hazard and a latency tax.
2. Jackson: `JsonParser` for streaming reads. Process token-by-token.
3. Producing large output: `JsonGenerator` writes incrementally. Spring's `ResponseEntity<StreamingResponseBody>` for streaming HTTP responses.
4. For files: `Flow<T>` of records, decoded one at a time, written to the response one at a time. End-to-end bounded memory.

## Cross-references

- DTOs vs entities: [ch. 04](./04-persistence.md).
- Validation as a boundary concern: [generic guide ch. 08](../kotlin/08-error-handling.md).
- Time types and clocks: [generic guide ch. 13](../kotlin/13-resource-management.md), §13.5.
