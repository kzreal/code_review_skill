# Python Code Review Guide

## Table of Contents
1. [Language Fundamentals](#language-fundamentals)
2. [Type Hints & Static Analysis](#type-hints--static-analysis)
3. [Error Handling](#error-handling)
4. [Concurrency & Async](#concurrency--async)
5. [Security](#security)
6. [Performance](#performance)
7. [Testing](#testing)
8. [Common Frameworks](#common-frameworks)
9. [Modern Python (3.10+)](#modern-python)

---

## Language Fundamentals

### Mutable Default Arguments
The #1 Python gotcha. Default arguments are evaluated once at function definition time.

```python
# BUG: shared across all calls
def append_to(item, target=[]):
    target.append(item)
    return target

# CORRECT:
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

### Scope & Closures
- `for` loop variables are captured by reference in closures — late binding issue
- Use default argument trick or `functools.partial` to capture current value
- `global` and `nonlocal` are code smells in production code — prefer explicit state

### Object Identity vs Equality
- `is` checks identity (same object), `==` checks equality (same value)
- Use `is` only for `None`, `True`, `False`, sentinel values
- Don't use `is` for comparing integers, strings — CPython caches small ints, but this is implementation detail

### Iteration Patterns
- Don't modify a collection while iterating over it — collect changes and apply after
- Use `enumerate()` instead of manual counter variables
- Use `zip()` for parallel iteration, `itertools.zip_longest()` if different lengths
- Generator expressions for memory-efficient processing of large datasets

### String Formatting
- f-strings are preferred (Python 3.6+): `f"Hello {name}"`
- For logging, use `%s` formatting (lazy evaluation): `logger.info("User %s logged in", user_id)`
- Don't use f-strings in log messages — they're evaluated even if log level is disabled

---

## Type Hints & Static Analysis

### Type Annotation Quality
- Use specific types: `list[User]` not `list`, `str | None` not `Optional[str]` (Python 3.10+)
- Use `TypedDict` for dict shapes instead of `dict[str, Any]`
- Use `Literal` for enum-like strings: `def set_mode(mode: Literal["read", "write"])`
- Use `Protocol` for structural typing instead of `Any` for duck-typed interfaces
- Avoid `Any` — if you must use it, document why

### Generic Patterns
- Use `TypeVar` for generic functions: `T = TypeVar("T")`
- Use `@overload` for functions with multiple valid signatures
- `Callable[[int, str], bool]` for function type hints
- `ClassVar` for class-level attributes (not instance attributes)

### Static Analysis Setup
- Recommend `mypy --strict` or at least `mypy` with `disallow_untyped_defs = true`
- `pyright` is faster and catches different issues — consider both
- Add `type: ignore` comments with explanation: `type: ignore[arg-type]  # third-party API mismatch`

---

## Error Handling

### Exception Design
- Create custom exception hierarchies: `AppError` → `NotFoundError`, `ValidationError`
- Include context: `raise NotFoundError(f"User {user_id} not found")`
- Use exception groups (Python 3.11+) for multiple related errors

### Anti-Patterns
```python
# BAD: bare except catches SystemExit, KeyboardInterrupt
except:
    pass

# BAD: too broad
except Exception:
    pass

# BAD: losing stack trace
except ValueError as e:
    raise RuntimeError(str(e))  # loses original traceback

# CORRECT: preserve chain
except ValueError as e:
    raise RuntimeError(f"Processing failed: {e}") from e
```

### Error Handling Patterns
- Use `try/except/else/finally` idiom: try the risky operation, except handles errors, else runs on success, finally always runs
- Context managers (`with` statement) for resource cleanup
- `contextlib.suppress()` for intentionally ignored exceptions

---

## Concurrency & Async

### async/await
- Don't call async functions without `await` — creates a coroutine that's never executed
- Don't use blocking calls in async context: `time.sleep()`, `requests.get()`, synchronous DB drivers
- Use `asyncio.to_thread()` for unavoidable blocking I/O
- `asyncio.gather()` for concurrent execution, but handle exceptions with `return_exceptions=True`

### Threading
- GIL prevents true parallelism for CPU-bound work — use `multiprocessing` instead
- `threading` is fine for I/O-bound work
- Always use locks for shared mutable state
- `queue.Queue` is thread-safe, `collections.deque` is not (despite some atomic operations)

### Common Pitfalls
- Race conditions with check-then-act: `if not os.path.exists(path): os.makedirs(path)` → use `exist_ok=True`
- Shared mutable state between threads without synchronization
- Forgetting to await coroutines — they silently do nothing
- `asyncio.run()` cannot be called from an existing event loop

---

## Security

### Injection
- **SQL injection**: Always use parameterized queries. Never f-string or `%` format SQL
  ```python
  # BAD
  cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
  # CORRECT
  cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
  ```
- **Command injection**: Use `subprocess.run()` with list args, never shell=True with string interpolation
- **Template injection**: Don't use `eval()`, `exec()`, or `__import__()` on user input
- **Path traversal**: Use `pathlib.Path.resolve()` and validate the path stays within expected directory

### Deserialization
- `pickle.loads()` on untrusted data is arbitrary code execution — use `json` or `msgpack`
- `yaml.load()` without `Loader=yaml.SafeLoader` is unsafe — use `yaml.safe_load()`
- Watch for `__reduce__` methods in objects that could be exploited

### Secrets & Credentials
- Never hardcode secrets — use environment variables or a secrets manager
- Don't log sensitive data: passwords, tokens, API keys, PII
- `.env` files should be in `.gitignore` — provide `.env.example` instead
- Watch for secrets in exception messages and debug output

### Dependencies
- Pin dependency versions: `fastapi==0.104.1` not `fastapi`
- Use `pip-audit` or `safety` to check for known vulnerabilities
- Review `requirements.txt` / `pyproject.toml` for suspicious or unnecessary dependencies

---

## Performance

### Common Pitfalls
- String concatenation in loops: use `"".join(list_of_strings)`
- `in` operator on lists: O(n) — convert to sets for O(1) lookup
- Repeated function calls in loops: cache or pre-compute outside
- Unnecessary list materialization: use generators for large data

### Data Structures
- `collections.defaultdict` instead of `dict.setdefault()` in loops
- `collections.Counter` for counting occurrences
- `collections.OrderedDict` only when order matters (regular dicts preserve insertion order since 3.7)
- `heapq` for top-N problems instead of sorting entire list

### Memory
- Use generators (`yield`) instead of building full lists for large datasets
- `sys.getsizeof()` only measures the container, not contents — for nested structures use `pympler`
- `__slots__` on classes with many instances to save memory
- Watch for circular references preventing garbage collection

### Profiling Signals
- N+1 database queries (same as Java — the universal performance bug)
- Synchronous I/O in request handlers
- Missing caching for frequently accessed, rarely changing data
- Loading entire datasets into memory instead of streaming

---

## Testing

### pytest Best Practices
- Fixtures for test setup, not for test logic
- Use `@pytest.fixture(scope="module")` or `scope="session"` for expensive setup
- Parametrize tests: `@pytest.mark.parametrize("input,expected", [(1, 2), (3, 6)])`
- Test files named `test_*.py`, classes `Test*`, functions `test_*`

### Test Quality
- Each test should be independent — no ordering dependencies
- Don't mock what you don't own — mock external APIs, not your own business logic
- Use `responses` or `respx` for HTTP mocking, `freezegun` for time-dependent tests
- Test error paths, not just happy paths

### Integration Testing
- Use test databases, not mocks, for data-layer tests
- `factory_boy` for test data generation
- Clean up state between tests: truncation or transaction rollback

---

## Common Frameworks

### FastAPI
- Use dependency injection for DB sessions, auth, config
- Pydantic models for request/response validation — don't accept raw dicts
- `BackgroundTasks` for fire-and-forget, Celery for durable task queues
- Use `APIRouter` for grouping related endpoints

### Django
- Don't bypass the ORM with raw SQL unless necessary (and document why)
- Use `select_related()` and `prefetch_related()` to avoid N+1
- Django ORM `bulk_create()` / `bulk_update()` for batch operations
- Middleware for cross-cutting concerns, not views

### Flask
- Use application factories for testing and multiple configurations
- Blueprints for organizing routes
- Don't store state in global variables — use extensions or app context

---

## Modern Python

### Python 3.10+
- **Structural pattern matching**: `match` / `case` — cleaner than long `if/elif` chains
- **Union types**: `X | Y` instead of `Union[X, Y]`
- **Type guards**: `TypeGuard` for narrowing types in `isinstance` checks

### Python 3.11+
- **Exception groups**: `except*` for handling multiple exceptions
- **`ExceptionGroup`**: Raise and catch multiple related exceptions
- **`tomllib`**: Built-in TOML parsing (no more third-party dependency)

### Python 3.12+
- **Type parameter syntax**: `def func[T](x: T) -> T` instead of `TypeVar`
- **`type` statement**: `type Point = tuple[float, float]` for type aliases
- **F-string improvements**: Can use same quote types, nested expressions
