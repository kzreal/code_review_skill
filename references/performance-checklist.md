# Performance Review Checklist

## Database

### N+1 Queries
The #1 performance issue in almost every codebase.

**Patterns to look for:**
- Loop that queries the database on each iteration
- ORM lazy-loading relationships inside a loop
- `@OneToMany` with EAGER fetch accessed in batch operations
- Service method that queries one entity, then loops over its children and queries each

**Solutions:**
- JOIN FETCH / Entity Graph / select_related / prefetch_related
- Batch queries with IN clause
- DataLoader pattern (GraphQL, or any batch-loading scenario)
- Denormalize for read-heavy access patterns

### Missing Indexes
- WHERE clause columns without indexes
- JOIN columns without indexes
- ORDER BY columns without indexes (causes filesort)
- Composite indexes for multi-column WHERE conditions

### Query Efficiency
- SELECT * when only specific columns are needed
- Missing pagination on large result sets
- Unnecessary DISTINCT / GROUP BY
- Subqueries that could be JOINs

---

## Caching

### When to Cache
- Frequently read, rarely written data (reference data, config, user profiles)
- Computed results that are expensive to regenerate
- External API responses that don't change often

### Cache Pitfalls
- No TTL / expiration → stale data forever
- Cache invalidation not triggered on data changes
- Cache key doesn't include relevant parameters (language, user context, etc.)
- Hot keys: single cache entry accessed by all requests
- Cache stampede: many requests miss cache simultaneously and all compute

### Cache Strategy
- Cache-aside: application manages cache explicitly (most common)
- Write-through: update cache on write
- Consider cache penetration protection for non-existent keys

---

## Memory

### Java
- Large object allocations in hot paths
- Memory leaks through static collections, unclosed resources, listener registration
- String concatenation in loops → `StringBuilder`
- Unnecessary boxing: `Integer` vs `int` in collections
- Stream operations that materialize large collections

### Python
- Loading entire datasets into memory instead of streaming
- Circular references preventing garbage collection
- `__slots__` for classes with many instances
- Generator expressions vs list comprehensions for large data

### JavaScript
- Memory leaks: event listeners not removed, closures holding references, timers not cleared
- Large arrays in memory when streaming would work
- `Map`/`Set` that grow unbounded (needs eviction strategy)
- DOM references in detached components

---

## Concurrency & Parallelism

### Thread Pool Configuration
- Default thread pool sizes may not be appropriate for production load
- I/O-bound: more threads than CPU cores (or virtual threads in Java 21)
- CPU-bound: roughly equal to CPU cores
- Watch for thread pool exhaustion from blocking operations

### Async Patterns
- Sequential awaits when operations are independent → `Promise.all()`, `asyncio.gather()`
- Blocking the event loop with CPU-intensive synchronous work
- Not using connection pooling for database / HTTP clients

### Lock Contention
- `synchronized` methods in Java that are called frequently
- Global locks when finer-grained locks would work
- Lock ordering issues that could cause deadlocks

---

## Frontend Performance

### Bundle Size
- Tree-shaking: verify named imports, avoid side effects
- Code splitting: lazy-load routes and heavy components
- Large dependencies that could be replaced with smaller alternatives
- Duplicate dependencies in bundle

### Rendering Performance
- Unnecessary re-renders: missing `React.memo`, `useMemo`, `useCallback`
- Expensive computations on every render
- Large lists without virtualization
- Layout thrashing: reading DOM layout properties in a loop

### Network
- Missing compression (gzip/brotli)
- No caching headers for static assets
- Waterfall requests: sequential API calls that could be parallel
- Missing image optimization (WebP, lazy loading, responsive sizes)

### Core Web Vitals
- **LCP** (Largest Contentful Paint): < 2.5s — affected by server response time, resource load, render-blocking
- **INP** (Interaction to Next Paint): < 200ms — affected by main thread blocking, event handler performance
- **CLS** (Cumulative Layout Shift): < 0.1 — affected by image/video dimensions, dynamic content insertion, web fonts

---

## API Performance

### Request/Response
- Over-fetching: returning more data than the client needs
- Under-fetching: requiring multiple requests for related data
- No compression for large JSON responses
- Missing pagination for list endpoints
- Synchronous processing for operations that could be async

### Backend Patterns
- Connection pooling for database and HTTP clients
- Request batching for external API calls
- Background processing for non-critical operations (email, logging, analytics)
- Circuit breakers for external service calls

---

## Common Anti-Patterns

| Anti-Pattern | Symptom | Fix |
|-------------|---------|-----|
| N+1 queries | Linear increase in DB calls with data size | Batch queries, JOIN FETCH |
| Synchronous in hot path | Request latency proportional to external calls | Async, parallel, cache |
| Loading everything | High memory, slow startup | Pagination, lazy loading, streaming |
| No caching for reads | Repeated identical queries/computations | Cache with TTL and invalidation |
| Premature optimization | Complex code for hypothetical performance | Measure first, optimize the bottleneck |
| Missing pagination | OOM or timeout on large datasets | Cursor or offset pagination |
