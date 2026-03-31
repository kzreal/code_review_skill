# Common Bugs Checklist

Language-specific common bug patterns. Use as a quick reference during code review.

---

## Java

### NullPointerException
- Auto-unboxing: method returns `Integer`, caller assigns to `int`
- `Map.get()` returns null for missing keys — use `getOrDefault()` or `computeIfAbsent()`
- `Optional.get()` without `isPresent()` check
- Chained method calls without null checks: `user.getAddress().getCity()`
- `@Autowired` field accessed before injection (constructor or lifecycle method)

### Concurrency Bugs
- `SimpleDateFormat` shared across threads — use `DateTimeFormatter`
- `HashMap` concurrent modification — use `ConcurrentHashMap`
- Check-then-act without synchronization: `if (!cache.containsKey(k)) cache.put(k, v)`
- `@Async` / `@Transactional` self-invocation (proxy bypass)
- Double-checked locking without `volatile`

### Spring Boot Specific
- `@Transactional` on private methods (doesn't work — Spring uses proxies)
- Controller returning entity directly (lazy loading exception outside transaction)
- `@Value` with missing property → `IllegalArgumentException` at runtime
- Circular dependency detected only at startup → use constructor injection to catch early
- Filter/Interceptor order not specified → unexpected execution order

### Resource Leaks
- `InputStream` / `OutputStream` not closed in finally block (use try-with-resources)
- `Stream` not closed when it wraps a file resource
- `HttpClient` created per request instead of shared
- Database connection not returned to pool on exception

### Logic Errors
- `==` on enums: works but confusing, use `.equals()` or switch
- `Date` / `Calendar` months are 0-indexed (January = 0)
- `List.remove(int)` vs `List.remove(Object)` — different behavior
- `String.substring()` in Java 7+ creates new String (no longer shares backing array)
- Switch without `break` → fall-through

---

## Python

### Mutable Default Arguments
```python
# BUG
def process(items, cache={}):
    cache[key] = value  # shared across ALL calls

# FIX
def process(items, cache=None):
    if cache is None:
        cache = {}
```

### Late Binding in Closures
```python
# BUG: all lambdas capture the same 'i'
funcs = [lambda: i for i in range(5)]
# All return 4

# FIX: default argument captures value
funcs = [lambda i=i: i for i in range(5)]
```

### Dictionary Modification During Iteration
```python
# BUG: RuntimeError
for key in d:
    if should_remove(key):
        del d[key]

# FIX
for key in list(d.keys()):
    if should_remove(key):
        del d[key]
```

### Integer Caching Surprise
- `a is b` works for small integers (-5 to 256) due to CPython caching — but it's implementation detail
- Always use `==` for value comparison, never `is`

### Common Off-by-One
- `range(n)` goes from 0 to n-1
- Slice `[start:end]` includes start, excludes end
- `list.pop()` removes and returns the last element
- Negative indexing: `lst[-1]` is the last element

### Type Confusion
- `True == 1` and `False == 0` — boolean is a subclass of int
- `bool("False")` is `True` — any non-empty string is truthy
- `0 == False` is `True` — can cause bugs in comparisons

---

## JavaScript / TypeScript

### Async Bugs
```javascript
// BUG: sequential when parallel is correct
const a = await fetchA();
const b = await fetchB(); // doesn't depend on a

// FIX
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

### Stale Closures
```javascript
// BUG: count inside setTimeout is always 0
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // always 1, because count is captured at mount
  }, 1000);
  return () => clearInterval(id);
}, []);

// FIX: functional update
setCount(prev => prev + 1);
```

### Array Sort Bug
```javascript
// BUG: sorts by string comparison
[10, 9, 8, 80].sort() // → [10, 8, 80, 9]

// FIX: numeric comparator
[10, 9, 8, 80].sort((a, b) => a - b) // → [8, 9, 10, 80]
```

### React Key Anti-Patterns
```jsx
// BUG: using index as key causes state mismatch on reorder
items.map((item, index) => <Item key={index} {...item} />)

// FIX: use stable unique id
items.map(item => <Item key={item.id} {...item} />)
```

### Truthy/Falsy Trbugs
- `if (count)` is false when count is 0 — use `if (count > 0)` or `if (count !== undefined)`
- `if (items.length)` is fine, but `if (items)` is always true for arrays
- `NaN !== NaN` — use `Number.isNaN()`
- `typeof null === "object"` — check with `=== null`

### Object Reference Bugs
```javascript
// BUG: both share the same array
const defaults = { items: [] };
const config = { ...defaults };
config.items.push("a"); // mutates defaults.items too!

// FIX: deep clone or separate defaults
const config = { items: [...defaults.items] };
```

### TypeScript Specific
- Type assertions (`as`) don't change runtime behavior — data might not match the asserted type
- Non-null assertion (`!`) doesn't check for null at runtime
- `enum` in TypeScript compiles to runtime JavaScript — may not behave as expected
- `keyof typeof obj` for getting keys from a const object

---

## Cross-Language Patterns

### Date/Time Bugs
- Timezone: always use UTC for storage, convert for display
- DST transitions: 1 hour appears/disappears — use date library, not manual math
- Leap seconds/years: test February 29 edge cases
- `Date` in JavaScript is mutable — be careful with shared instances

### String Encoding
- Always use UTF-8 for text processing
- Watch for encoding mismatches at system boundaries (file I/O, HTTP, database)
- URL encoding vs HTML encoding: use the right one for the context
- Base64: not encryption, not compression

### Number Precision
- Floating point: `0.1 + 0.2 !== 0.3` in JavaScript and Java (use `BigDecimal` or `Decimal` for money)
- Integer overflow: Java `int` max is ~2.1 billion — use `long` for IDs, timestamps
- Python: arbitrary precision ints, but float has same limitations as other languages

### Resource Cleanup
- Always close: database connections, file handles, HTTP connections, streams
- Use language-appropriate patterns: try-with-resources, context managers, useEffect cleanup, onUnmounted
- Clean up in reverse order of acquisition
- Watch for resource leaks in error paths
