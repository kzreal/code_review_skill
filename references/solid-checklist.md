# SOLID & Architecture Review Checklist

## Single Responsibility Principle (SRP)

**A module should have one reason to change.**

### Smells
- Class/function that handles both business logic and infrastructure (DB, HTTP, file I/O)
- Service class with "and" in its conceptual name: `UserAndOrderService`
- Functions longer than ~50 lines doing multiple distinct operations
- Module importing from many unrelated packages
- `utils.ts` / `helpers.py` catch-all files that grow without bound

### Review Prompts
- "What single responsibility does this class/module have?"
- "If business rule X changes, which files need to be modified?" (Should be 1-2, not 5+)
- "Does this function do one thing, or does it do the thing AND handle errors AND format output AND log?"

### Refactoring Approach
- Extract infrastructure concerns (DB, HTTP, logging) from business logic
- Split "god classes" along responsibility boundaries
- Use the Strategy pattern when a class has multiple algorithms

---

## Open/Closed Principle (OCP)

**Software entities should be open for extension, closed for modification.**

### Smells
- Adding a new type requires modifying a large switch/if-else chain
- Adding a new feature requires changes in 5+ existing files
- Feature flags scattered across the codebase
- Type-checking pattern: `if (type === "A") { ... } else if (type === "B") { ... }`

### Review Prompts
- "How many files need to change to add a new payment method / notification channel / data source?"
- "Is this switch statement closed for modification, or does every new case require editing it?"
- "Could this be expressed as a registry or plugin pattern instead?"

### Refactoring Approach
- Registry pattern: each type registers itself, the core doesn't change
- Strategy pattern: behavior injected rather than hard-coded
- Template method pattern for algorithm skeletons with varying steps
- In TypeScript/Python: discriminated unions or protocols for extensibility

---

## Liskov Substitution Principle (LSP)

**Subtypes must be substitutable for their base types.**

### Smells
- Subclass that throws `UnsupportedOperationException` for inherited methods
- `instanceof` checks to handle specific subclass behavior
- Subclass that strengthens preconditions or weakens postconditions
- Empty method overrides that do nothing (violating parent contract)
- Subclass that changes the meaning of inherited methods

### Review Prompts
- "Can I use any subclass of this interface interchangeably without knowing which one?"
- "Does this override respect the parent's contract, or does it change expectations?"
- "Why is there an `instanceof` check here? Does the hierarchy need redesigning?"

### Refactoring Approach
- If subclasses can't fulfill the contract, the hierarchy is wrong
- Prefer composition over inheritance when behavior differs significantly
- Split the interface: narrower interfaces that each subclass can fully implement
- Use `final` / `sealed` when a class is not designed for extension

---

## Interface Segregation Principle (ISP)

**Clients should not be forced to depend on interfaces they do not use.**

### Smells
- Interface with 10+ methods where most implementers only care about 2-3
- "Fat interface" that combines unrelated capabilities
- Implementers throwing `NotImplementedException` for methods they don't need
- Changes to one method force recompilation/redeployment of unrelated consumers

### Review Prompts
- "Does every consumer of this interface use all its methods?"
- "Could this be split into 2-3 focused interfaces?"
- "Why does this class implement methods it doesn't use?"

### Refactoring Approach
- Split fat interfaces into focused ones (e.g., `Readable` + `Writable` instead of `IO`)
- Role interfaces: one interface per consumer role
- In dynamic languages (Python, JS): Protocol / duck typing naturally supports ISP

---

## Dependency Inversion Principle (DIP)

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

### Smells
- Business logic directly importing/using specific database client, HTTP library, or file system
- Hard-coded external service URLs or connection strings in business logic
- Tests that require real database or external API to run
- `new` keyword for infrastructure objects inside business logic
- Business logic that knows about framework-specific types

### Review Prompts
- "Can I test this business logic without a database?"
- "If we switch from MySQL to PostgreSQL, does business logic need to change?"
- "Does this module depend on an abstraction or a specific implementation?"

### Refactoring Approach
- Define interfaces/protocols for external dependencies
- Dependency injection: receive dependencies, don't create them
- Repository pattern for data access abstraction
- Anti-corruption layer for third-party API integration

---

## Cross-Cutting Architecture Concerns

### Coupling
- **Afferent coupling** (who depends on this): High for stable modules, low for volatile ones
- **Efferent coupling** (what does this depend on): Should be minimal for core business logic
- Avoid circular dependencies between modules
- Use events/messages for cross-module communication when appropriate

### Cohesion
- All elements of a module should work together to accomplish a single purpose
- Low cohesion indicators: many private methods that don't share data, many parameters passed between methods
- High cohesion indicators: methods work on the same data, meaningful module name

### Layering
- Clear separation: presentation → application → domain → infrastructure
- Dependency direction: outer layers depend on inner layers, never the reverse
- No business logic in controllers/handlers
- No presentation concerns (HTTP status codes, HTML) in services/domain

### API Design
- Consistent naming and error handling across endpoints
- Version strategy for breaking changes
- Pagination for list endpoints
- Request/response DTOs separate from domain entities

### Error Architecture
- Define error hierarchy per domain
- Errors should carry context (what operation, what entity, what went wrong)
- Don't expose internal error details to API consumers
- Distinguish between client errors (4xx) and server errors (5xx)
