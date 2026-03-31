# Java / Spring Boot Code Review Guide

## Table of Contents
1. [General Java](#general-java)
2. [Spring Boot Specific](#spring-boot-specific)
3. [JPA / Database](#jpa--database)
4. [Concurrency](#concurrency)
5. [Error Handling](#error-handling)
6. [Security](#security)
7. [Testing](#testing)
8. [Modern Java Patterns (17/21)](#modern-java-patterns)

---

## General Java

### Null Safety
- Prefer `Optional` as return type for methods that may not find a result, but don't use it for fields or method parameters
- Use `Objects.requireNonNull()` for constructor/method parameter validation
- Check for `@Nullable` / `@NonNull` annotations and respect them
- Watch for auto-unboxing NPEs: `Integer` → `int` when value is null
- Map `.get()` on Optional without `.isPresent()` check is a bug

### Resource Management
- Always use try-with-resources for `InputStream`, `OutputStream`, `Connection`, `Statement`, `ResultSet`
- Watch for resource leaks in loops — resource created per iteration but not closed on exception
- `Stream` and `CompletableFuture` can leak resources if not properly chained

### Collections
- Return empty collections instead of null: `Collections.emptyList()`, `List.of()`
- Use immutable collections for constants: `List.of()`, `Map.of()`, `Set.of()`
- Watch for `ConcurrentModificationException` — don't modify collection while iterating (use `removeIf` or iterator)
- `ArrayList.subList()` returns a view, not a copy — mutations affect the parent

### Equals / HashCode / ToString
- Always override both `equals` and `hashCode` together, or neither
- Use `Objects.equals()` and `Objects.hash()` to avoid null issues
- For JPA entities, base `equals`/`hashCode` on business key, not auto-generated ID
- Avoid using mutable fields in `hashCode` (causes issues with HashSet/HashMap)

### String Handling
- Use `StringBuilder` for string concatenation in loops
- `String.format()` or text blocks (Java 15+) for multi-line strings
- Watch for `==` on String references — always use `.equals()`
- `String.intern()` can cause permgen/metaspace issues — avoid unless necessary

---

## Spring Boot Specific

### Dependency Injection
- Prefer constructor injection over `@Autowired` field injection
- Use `@RequiredArgsConstructor` (Lombok) or manual constructors for required dependencies
- `@Autowired` on constructors is optional in Spring Boot 3 (single constructor auto-wires)
- Be suspicious of `@Autowired` on fields — it hides dependency complexity and makes testing harder
- Watch for circular dependencies: A → B → A. Use `@Lazy` as a last resort, redesign as first

### Configuration
- Use `@ConfigurationProperties` instead of `@Value` for grouped config
- Validate config with `@Validated`, `@NotNull`, `@Min`, `@Max`
- Never hardcode secrets — use environment variables or vault integration
- Profile-specific configs should not contain production secrets

### Web Layer
- Controllers should be thin — delegate to services
- Use `@RestController` + `ResponseEntity` for API endpoints
- Validate input with `@Valid` / `@Validated` + Bean Validation annotations
- Don't swallow exceptions in controllers — let `@ControllerAdvice` handle them
- Use `ProblemDetail` (Spring 6) for standardized error responses
- Watch for `@RequestMapping` without method restriction — use `@GetMapping`, `@PostMapping`, etc.

### Service Layer
- Mark as `@Transactional(readOnly = true)` for read operations — it's a hint for optimization
- Don't call `@Transactional` methods from within the same class (proxy bypass)
- Watch for `@Transactional` on methods that call external APIs — long-running transactions
- Prefer programmatic transaction boundaries for complex scenarios

### Async & Scheduling
- `@Async` methods must return `void` or `CompletableFuture`
- Don't call `@Async` from within the same class (same proxy issue as `@Transactional`)
- `@Scheduled` tasks should be idempotent — they may overlap if previous execution is slow
- Configure thread pools — don't rely on defaults for production

---

## JPA / Database

### N+1 Problem
The most common JPA performance issue. Watch for:
- `@OneToMany` with `FetchType.EAGER` accessed in loops
- Lazy-loading entities inside loops after transaction closes (causes `LazyInitializationException`)
- Missing `JOIN FETCH` in JPQL queries when related entities are needed

Solutions:
- `@EntityGraph` for predictable fetching
- `@BatchSize` for unavoidable lazy access
- DTO projections with JPQL/Criteria API for read-only queries

### Entity Design
- Don't expose JPA entities directly in API responses — use DTOs
- `@Data` (Lombok) on entities causes `equals`/`hashCode`/`toString` issues with lazy loading
- Use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` and include business key fields
- `toString()` should not touch lazy-loaded associations

### Query Patterns
- Prefer Spring Data JPA derived queries for simple cases
- Use `@Query` for complex queries — avoid overly complex method names like `findByUserAndStatusAndCreatedAtBetween`
- Watch for missing pagination on large result sets
- Native queries bypass JPA cache and entity mapping — document why they're needed

### Transactions
- Read operations: `@Transactional(readOnly = true)`
- Write operations: default `@Transactional` is fine, but be explicit about isolation if it matters
- Don't perform external API calls inside transactions — they hold DB connections
- Watch for self-invocation of transactional methods within the same bean

---

## Concurrency

### Thread Safety
- `SimpleDateFormat` is not thread-safe — use `DateTimeFormatter` (Java 8+) or `ThreadLocal`
- `HashMap` is not thread-safe — use `ConcurrentHashMap` for shared state
- `ArrayList` is not thread-safe — use `CopyOnWriteArrayList` or `Collections.synchronizedList`
- Watch for check-then-act race conditions: `if (!map.containsKey(k)) { map.put(k, v); }`
- `synchronized` on Spring beans affects all requests — consider `ConcurrentHashMap.computeIfAbsent()`

### Virtual Threads (Java 21)
- Good for I/O-bound tasks (HTTP calls, DB queries, file operations)
- Don't pin virtual threads: avoid `synchronized` blocks with I/O — use `ReentrantLock` instead
- Don't pool virtual threads — they're cheap to create
- Watch for `ThreadLocal` proliferation with virtual threads — they still consume memory

### CompletableFuture
- Always handle exceptions: `.exceptionally()` or `.handle()`
- Watch for swallowed exceptions in async chains
- Don't block on `.get()` without timeout — use `.get(timeout, unit)` or `.join()` with error handling

---

## Error Handling

### Exception Design
- Use unchecked exceptions for business logic errors, checked for recoverable conditions
- Create a hierarchy: `BaseException` → `NotFoundException`, `ValidationException`, etc.
- Include context in exception messages: entity ID, operation, input values
- Don't catch `Exception` broadly — catch specific types or rethrow

### Global Error Handling
- Use `@ControllerAdvice` with `@ExceptionHandler` for consistent error responses
- Log exceptions with context (request ID, user, operation) at the appropriate level
- Don't expose stack traces or internal details in API error responses
- Map exceptions to HTTP status codes consistently

### Anti-Patterns
- Swallowing exceptions: `catch (Exception e) { /* nothing */ }` or just `log.error()` without rethrowing
- Wrapping everything in a generic `RuntimeException` — loses type information
- Using exceptions for control flow (e.g., throwing to break a loop)
- Logging and rethrowing: causes duplicate log entries, confusing in production

---

## Security

### Input Validation
- Validate all external input at the boundary (controller layer)
- Use Bean Validation: `@NotNull`, `@Size`, `@Pattern`, `@Email`, `@Min`, `@Max`
- Don't trust client-side validation — it's for UX, not security
- Sanitize input used in file paths, SQL queries, shell commands

### Authentication & Authorization
- `@PreAuthorize` and `@Secured` on service methods for defense in depth
- Don't bypass security checks for "internal" APIs without good reason
- Watch for IDOR: accessing resources by ID without ownership check
- JWT tokens: validate expiry, signature, issuer, audience on every request

### Common Vulnerabilities
- SQL injection: use parameterized queries / JPA, never string concatenation
- XSS: escape output, use `Content-Type: application/json` for APIs
- CSRF: ensure CSRF protection for state-changing operations (or use stateless auth)
- Deserialization: avoid `ObjectInputStream` with untrusted data
- SSRF: validate and restrict URLs in server-side HTTP requests

---

## Testing

### Unit Tests
- Test behavior, not implementation — don't assert private method calls
- Use `@ExtendWith(MockitoExtension.class)` with `@Mock` and `@InjectMocks`
- Don't mock the system under test — mock its dependencies
- Watch for tests that always pass: no assertions, catching all exceptions

### Integration Tests
- Use `@SpringBootTest` sparingly — it starts the full context
- Prefer `@WebMvcTest` for controllers, `@DataJpaTest` for repositories
- Use `Testcontainers` for database integration tests — don't mock repositories
- Clean up test data: `@Transactional` + rollback, or explicit cleanup

### Test Quality
- Every test should have: Arrange → Act → Assert
- Test names should describe the scenario: `shouldReturn404WhenUserNotFound()`
- Cover edge cases: null, empty, boundary values, concurrent access
- Don't ignore tests with `@Disabled` without a reason and a ticket reference

---

## Modern Java Patterns

### Java 17+
- **Sealed classes**: Model closed hierarchies — `sealed interface Shape permits Circle, Rectangle`
- **Pattern matching for switch**: `switch (obj) { case String s -> ...; case Integer i -> ...; }`
- **Text blocks**: `"""..."""` for multi-line strings (SQL, JSON, etc.)
- **Records**: `record Point(int x, int y) {}` for immutable data carriers — use for DTOs

### Java 21
- **Virtual threads**: See Concurrency section above
- **Record patterns**: `if (obj instanceof Point(int x, int y)) { ... }`
- **Sequenced collections**: `SequencedCollection`, `getFirst()`, `getLast()`

### When to Recommend Modern Patterns
- Records over Lombok `@Data` for DTOs
- Pattern matching over `instanceof` + casts
- Text blocks over string concatenation for multi-line
- Switch expressions over statement switches
