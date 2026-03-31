# C# / .NET Code Review Guide

## Table of Contents
1. [General C#](#general-c)
2. [ASP.NET Core](#aspnet-core)
3. [Entity Framework Core](#entity-framework-core)
4. [Concurrency & Async](#concurrency--async)
5. [Error Handling](#error-handling)
6. [Security](#security)
7. [Testing](#testing)
8. [Modern C# (10/12/13)](#modern-c)
9. [WPF / Desktop](#wpf--desktop)

---

## General C#

### Null Safety
- Enable `<Nullable>enable</Nullable>` in `.csproj` — treat null warnings as errors in new projects
- Use `?` for nullable reference types: `string? name` not `string name` when null is possible
- Use null-forgiving operator `!` only when you can prove non-null — document why
- `NullReferenceException` in production means nullable annotations are wrong or missing
- Prefer `pattern matching` for null checks: `if (user is null)` or `if (user is { })`

### Resource Management
- Always use `using` declarations or `using` blocks for `IDisposable`: `SqlConnection`, `FileStream`, `HttpClient`, `StreamReader`
- `using var` declaration (C# 8+) for scope-based cleanup — cleaner than block form
- Watch for `IDisposable` stored in fields without `Dispose` pattern implementation
- `Task` and `IAsyncEnumerable` should be awaited or disposed — never fire-and-forget

### Collections
- Use collection expressions (C# 12): `List<int> list = [1, 2, 3];`
- Return `IEnumerable<T>` for read-only, `IReadOnlyList<T>` for indexed read-only
- Use `FrozenDictionary` / `FrozenSet` ( .NET 8+) for immutable lookup collections
- Don't expose `List<T>` as public property — use `IReadOnlyList<T>` or `ImmutableArray<T>`
- `Span<T>` and `Memory<T>` for zero-allocation slicing in hot paths

### Equality & Hashing
- Override `Equals` and `GetHashCode` together, or use `record` types (compiler generates them)
- Implement `IEquatable<T>` for value comparisons in collections
- For EF entities, don't use auto-generated ID in `Equals` — transient entities won't match
- Use `EqualityComparer<T>.Default` for generic comparisons

### String Handling
- Use string interpolation: `$"Hello {name}"` — performant and readable
- `StringBuilder` for concatenation in loops
- `ReadOnlySpan<char>` for zero-allocation string parsing (avoid `Substring` in hot paths)
- Use `string.Create()` for efficient string building with patterns
- Watch for culture-sensitive comparisons: use `StringComparison.Ordinal` for programmatic comparisons

---

## ASP.NET Core

### Dependency Injection
- Prefer constructor injection — avoid `[FromServices]` on every action parameter
- Use `IServiceCollection` extension methods to organize registrations
- Scoped services for per-request state (DbContext, unit of work)
- Transient for stateless services; Singleton for thread-safe shared state
- **Captive dependency bug**: Singleton holding Scoped or Transient service — lifetime mismatch

### Middleware & Filters
- Exception handling middleware should be registered early in pipeline
- Use `IAsyncExceptionFilter` or exception handling middleware, not try/catch in every controller
- Authorization filters execute before action filters — don't duplicate auth logic
- Use `IResultFilter` for response wrapping, not middleware (more targeted)

### Controllers & Minimal APIs
- Controllers should be thin — delegate to services
- Minimal APIs (.NET 7+): use `Results.Ok()`, `Results.NotFound()`, etc. for typed responses
- Use `[ApiController]` for automatic 400 validation responses
- Validate with `DataAnnotations` or `FluentValidation` — don't roll your own
- Don't return entities directly — use DTOs or records to control response shape
- Tag endpoints with `[Authorize]` by default, `[AllowAnonymous]` to opt out

### Configuration
- Use `IOptions<T>`, `IOptionsMonitor<T>`, or `IOptionsSnapshot<T>` pattern
- `appsettings.json` for non-sensitive config; environment variables or Key Vault for secrets
- Never store connection strings with passwords in `appsettings.json` — use User Secrets locally, Key Vault in production
- `IValidateOptions<T>` for configuration validation at startup

---

## Entity Framework Core

### N+1 Problem
The #1 EF performance issue.

**Patterns to look for:**
- Lazy loading triggered inside loops (navigation property access after query)
- `Include()` missing for eagerly needed relationships
- `AsNoTracking()` missing for read-only queries (default tracking adds overhead)

**Solutions:**
- `Include()` / `ThenInclude()` for eager loading
- `AsNoTracking()` for read-only queries — significant performance improvement
- `AsSplitQuery()` for queries with multiple includes (avoids Cartesian explosion)
- Projection to DTO: `Select(x => new Dto { ... })` — only loads needed columns
- `TagWith()` for query identification in logs

### Query Patterns
- Use raw SQL only when EF can't express the query — and document why
- `FromSqlRaw()` with parameterized queries, never string interpolation
- Watch for client-side evaluation: `.Where(x => SomeLocalFunction(x))` pulls entire table into memory
- `AsAsyncEnumerable()` for streaming large results — don't `.ToListAsync()` everything
- Pagination: use `Skip().Take()` or keyset pagination for large datasets

### Migrations & Schema
- Always review generated migrations before applying
- Check for missing indexes on frequently queried columns
- Watch for destructive migrations in production: dropping columns, changing types
- Use `HasConversion()` for value objects, don't store as JSON blobs unless needed

### Concurrency
- Use `RowVersion` / `[Timestamp]` for optimistic concurrency
- Handle `DbUpdateConcurrencyException` — retry or inform user
- Don't hold DbContext across multiple requests — it's not thread-safe

---

## Concurrency & Async

### async/await
- Don't use `async void` — use `async Task` except for event handlers (the only valid case)
- Every `await` should be in a method returning `Task` or `ValueTask`
- Configure `ConfigureAwait(false)` in library code — not needed in ASP.NET Core (no SynchronizationContext)
- Don't sync-over-async: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` — causes deadlocks

```csharp
// BAD: sync over async, potential deadlock
var result = httpClient.GetStringAsync(url).Result;

// GOOD: async all the way
var result = await httpClient.GetStringAsync(url);
```

### Cancellation
- Accept `CancellationToken` in async methods and pass it through
- Check `token.IsCancellationRequested` in CPU-bound loops
- Use `token.ThrowIfCancellationRequested()` periodically in long operations
- Register cancellation with `HttpClient` requests and database queries

### Parallelism
- `Parallel.ForEach` for CPU-bound work, not I/O-bound
- `Task.WhenAll` for concurrent I/O operations
- `Parallel.ForEachAsync` (.NET 6+) combines both patterns
- Watch for shared mutable state in parallel loops — use `ConcurrentDictionary`, `Interlocked`, or locks

### Common Pitfalls
- `async void` event handlers: exceptions crash the process — wrap in try/catch
- Forgetting to await a `Task` — silent failure, unobserved task exception
- `ValueTask` used incorrectly (awaited twice or after caching) — use `Task` unless profiling shows allocation pressure
- `SemaphoreSlim` for async locking — don't use `lock` statement with async code

---

## Error Handling

### Exception Design
- Use specific exception types: `NotFoundException`, `ValidationException`, not generic `Exception`
- Create custom exceptions inheriting from a base app exception
- Include context in exception messages and `Data` dictionary
- Don't throw exceptions for control flow — use `Result<T>` or `OneOf` pattern for expected failures

### Global Error Handling
- Use exception handling middleware in ASP.NET Core pipeline
- `IExceptionHandler` (.NET 8) for minimal API error handling
- Map exceptions to `ProblemDetails` (RFC 7807) for consistent API errors
- Don't expose stack traces or internal details in production error responses

### Anti-Patterns
```csharp
// BAD: swallowing exceptions
catch (Exception) { }

// BAD: catching too broadly and losing info
catch (Exception ex) { throw ex; } // destroys stack trace

// CORRECT: rethrow preserving stack trace
catch (Exception ex) { throw; }

// CORRECT: exception chaining
catch (SqlException ex) { throw new DataAccessException("Query failed", ex); }
```

### Result Pattern
For expected failure cases (validation, not-found), prefer over exceptions:
```csharp
// Using OneOf, FluentResults, or custom Result<T>
public Result<User> FindUser(Guid id) { ... }

// Over exception-based control flow
public User FindUser(Guid id) {
    throw new NotFoundException(); // avoid for expected cases
}
```

---

## Security

### Authentication & Authorization
- Use `[Authorize]` attribute — don't roll your own auth checks
- `[Authorize(Roles = "Admin")]` for role-based; `[Authorize(Policy = "CanEdit")]` for claim-based
- Resource-based authorization: verify ownership, not just authentication
- Don't store sensitive data in `HttpContext.Items` without proper authorization
- JWT validation: check issuer, audience, expiry, signing key on every request

### Input Validation
- Validate all input at the boundary (controller / minimal API)
- DataAnnotations: `[Required]`, `[StringLength]`, `[Range]`, `[RegularExpression]`
- FluentValidation for complex rules
- Don't trust client-side validation — server must re-validate
- Sanitize input used in file paths, SQL, URLs, shell arguments

### Common Vulnerabilities
- **SQL injection**: Use EF Core parameterized queries, never raw string interpolation in `FromSqlRaw()`
- **XSS**: ASP.NET Core auto-encodes Razor output, but watch for `@Html.Raw()`
- **CSRF**: Use `[ValidateAntiForgeryToken]` for cookie-based auth; JWT is CSRF-resistant
- **Open redirect**: Validate redirect URLs against allowlist
- **Deserialization**: Avoid `BinaryFormatter` (deprecated, dangerous); use `System.Text.Json`
- **SSRF**: Validate and restrict URLs in `HttpClient` calls from server-side code

### Secrets Management
- Use .NET Secret Manager for development (`dotnet user-secrets`)
- Azure Key Vault, AWS Secrets Manager, or HashiCorp Vault for production
- Never commit connection strings with passwords to source control
- Don't log secrets: mask in logging configuration

---

## Testing

### Unit Testing
- xUnit preferred: `[Fact]`, `[Theory]`, `[InlineData]`
- `Moq` or `NSubstitute` for mocking
- `FluentAssertions` for readable assertions: `result.Should().Be(expected)`
- Test naming: `MethodName_Scenario_ExpectedResult`
- Don't mock the system under test — mock its dependencies

### Integration Testing
- `WebApplicationFactory<T>` for ASP.NET Core integration tests
- Use `Testcontainers` for real database instances — don't mock DbContext
- Reset database state between tests: transaction rollback or recreation
- Test middleware pipeline, auth filters, and error handling end-to-end

### Test Quality
- Cover happy path AND error paths
- Test boundary conditions: null, empty, max values, concurrent access
- Don't use `[Ignore]` / `[Skip]` without a linked issue/ticket
- Keep tests independent — no ordering dependencies

---

## Modern C#

### C# 10 / .NET 6
- **Global usings**: `global using System.Collections.Generic;` — reduce boilerplate
- **File-scoped namespaces**: `namespace MyApp.Services;` — less nesting
- **Record structs**: `record struct Point(int X, int Y);` for value-type records
- **Constant interpolated strings**: `const string msg = $"Hello {name}";` when `name` is const

### C# 11 / .NET 7
- **Raw string literals**: `"""..."""` for multi-line strings, regex, JSON, SQL
- **List patterns**: `if (arr is [1, 2, ..])` for array matching
- **Required members**: `required string Name { init; }` enforces initialization
- **`Span<T>` pattern matching**: more efficient buffer handling

### C# 12 / .NET 8
- **Primary constructors**: `class UserService(ILogger logger)` — simpler DI
- **Collection expressions**: `int[] arr = [1, 2, 3];` unified syntax
- **Inline arrays**: Performance-critical fixed-size buffers
- **`FrozenDictionary`/`FrozenSet`**: Optimized for read-heavy immutable lookups

### C# 13 / .NET 9
- **`params` collections**: `params ReadOnlySpan<int>` — fewer allocations
- **Partial properties**: Better source generator support
- **`lock` object**: `lock` statement now uses `System.Threading.Lock` for better performance
- **`allows ref struct`**: Generic constraints for ref struct types

### When to Recommend Modern Patterns
- Records over classes for immutable data carriers (DTOs, value objects)
- Primary constructors over boilerplate constructor + field assignments
- Collection expressions over `new List<T>` initialization
- File-scoped namespaces over block-scoped
- Pattern matching over `if`/`switch` with casts and `is` checks
- `FrozenDictionary` for configuration/lookup data initialized once

---

## WPF / Desktop

### MVVM Pattern
- ViewModels should not reference UI elements — use data binding
- `INotifyPropertyChanged` for property change notification — use `ObservableObject` (MVVM Toolkit)
- `ICommand` / `RelayCommand` for button actions — don't use code-behind event handlers
- Use `CommunityToolkit.Mvvm` for source-generated MVVM boilerplate

### Threading
- UI updates must be on UI thread — use `Dispatcher.Invoke()` or `TaskScheduler.FromCurrentSynchronizationContext()`
- Don't block UI thread with synchronous I/O — use `async/await`
- `IProgress<T>` for reporting progress from background threads
- Watch for race conditions between UI events and async operations

### Resource Management
- Don't create `HttpClient` per request — use static or `IHttpClientFactory`
- Dispose `BitmapImage`, `FileStream`, database connections properly
- Watch for memory leaks: event handler subscriptions not unsubscribed, static collections growing
