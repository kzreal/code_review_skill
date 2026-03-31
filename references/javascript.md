# JavaScript Code Review Guide

## Table of Contents
1. [Language Fundamentals](#language-fundamentals)
2. [Async Patterns](#async-patterns)
3. [Error Handling](#error-handling)
4. [React Patterns](#react-patterns)
5. [Vue Patterns](#vue-patterns)
6. [Security](#security)
7. [Performance](#performance)
8. [Testing](#testing)
9. [Modern JS (ES2022+)](#modern-js)

---

## Language Fundamentals

### Variable Declarations
- `const` by default, `let` when reassignment is needed, never `var`
- `const` objects/arrays can still be mutated — use `Object.freeze()` or `readOnly` wrappers when immutability matters
- Avoid declaring variables in global scope in browser environments

### Equality & Coercion
- Always use `===` and `!==` — never `==` / `!=`
- Watch for truthy/falsy traps: `0`, `""`, `null`, `undefined`, `NaN`, `false` are all falsy
- `typeof null === "object"` — historical bug, check with `=== null`
- `NaN !== NaN` — use `Number.isNaN()` to check

### Object & Array Pitfalls
- `Object.assign()` and spread (`{...obj}`) are shallow copies — nested objects are shared
- `JSON.parse(JSON.stringify(obj))` deep clone fails on: `Date`, `RegExp`, `Map`, `Set`, `undefined`, functions, circular refs
- Array methods: `map`, `filter`, `reduce` return new arrays; `sort`, `splice` mutate
- `Array.prototype.sort()` sorts by string comparison by default: `[10, 9, 8].sort()` → `[10, 8, 9]`

### Closures & Scope
- `for (var i = ...)` captures variable by reference — all closures share same `i`
- Block scoping with `let`/`const` fixes this
- Stale closures in React: event handlers capturing old state — use functional updates or refs

### `this` Binding
- Arrow functions don't bind `this` — they capture from enclosing scope
- Regular functions get `this` from call site
- React class components: must bind methods or use arrow function class fields
- In Vue 3 Composition API, `this` is not used — arrow functions are fine

---

## Async Patterns

### Promises
- Always handle rejections — unhandled promise rejections can crash Node.js
- `Promise.all()` fails fast: one rejection rejects the whole thing
- `Promise.allSettled()` when you need all results regardless of individual failures
- Don't create new Promise when you can return/chain existing one (Promise constructor anti-pattern)

### async/await
- Every `await` should be in a `try/catch` or the function should propagate errors
- Sequential `await` when operations are independent is a performance bug — use `Promise.all()`
- Top-level await in ES modules is fine, but blocks module execution

```javascript
// BAD: sequential when parallel is fine
const user = await getUser(id);
const orders = await getOrders(id); // doesn't depend on user

// GOOD: parallel
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

### Error Handling in Async
- `async` functions always return a promise — errors become rejections
- Don't mix `callback` and `promise` patterns
- Use `AbortController` for cancellable async operations (fetch, etc.)

---

## Error Handling

### Anti-Patterns
```javascript
// BAD: empty catch
try { something(); } catch (e) {}

// BAD: losing error info
catch (e) { throw new Error('Failed'); }

// CORRECT: preserve chain
catch (e) { throw new Error(`Failed: ${e.message}`, { cause: e }); }
```

### Patterns
- Custom error classes extending `Error` for domain-specific errors
- `error.cause` (ES2022) for error chaining
- Error boundaries in React, `errorCaptured` in Vue for UI error handling
- Centralized error handler for API errors (interceptors, etc.)

---

## React Patterns

### Hooks Rules
- Only call hooks at the top level — not in loops, conditions, or nested functions
- Only call hooks from React function components or custom hooks
- Dependencies in `useEffect` / `useMemo` / `useCallback` must be complete and correct
- Stale closure is the #1 bug source: state accessed in timeout/interval/subscription without proper ref

### useEffect Pitfalls
- Missing dependencies → stale data, incorrect behavior
- Object/array dependencies cause infinite loops (new reference each render) → memoize or use primitive deps
- Fetching in useEffect without cleanup → race conditions on rapid re-renders → use AbortController
- `useEffect` runs after paint — don't use it for layout measurements (use `useLayoutEffect`)

### State Management
- Don't derive state that can be computed: `const fullName = `${firstName} ${lastName}`` — no need for state
- Lifting state up: when siblings share state, move it to their common parent
- Avoid prop drilling beyond 2-3 levels — use Context, Zustand, or other state management
- Use `useReducer` for complex state logic with multiple related values

### Component Design
- One component, one responsibility — split when a component does too many things
- Extract reusable logic into custom hooks
- Use `key` prop correctly: stable, unique identifiers, not array indices (causes bugs with reordering)
- Avoid inline object/array in JSX: `style={{ color: 'red' }}` creates new object every render

### React 18/19 Specifics
- `useTransition()` for marking non-urgent state updates
- `useDeferredValue()` for deferring expensive renders
- `useId()` for generating stable unique IDs (not for keys)
- React 19: `useActionState()` for form state, `useOptimistic()` for optimistic UI
- React 19: Server Components — don't use hooks or browser APIs in server components
- React 19: `use()` hook for consuming promises and contexts in render

### Performance
- `React.memo()` for components that re-render often with same props
- `useMemo()` for expensive computations, not for every value
- `useCallback()` only when passing callbacks to memoized children
- Virtualize long lists: `react-window` or `react-virtuoso`
- Code splitting: `React.lazy()` + `Suspense` for route-level splitting

---

## Vue Patterns

### Composition API (Vue 3.5+)
- Use `<script setup>` syntax — it's cleaner and more performant
- `ref()` for primitives, `reactive()` for objects — be consistent within a project
- `toRefs()` / `toRef()` to destructure reactive objects without losing reactivity
- `computed()` for derived state — don't compute in `watch` or manual assignments

### Reactivity Gotchas
- Destructuring `reactive()` objects loses reactivity — use `toRefs()`
- `ref()` needs `.value` access in `<script>` (auto-unwrapped in template)
- Don't mutate props — emit events to parent for changes
- `watchEffect()` auto-tracks dependencies, `watch()` is explicit — choose intentionally

### Component Communication
- Props down, events up — standard unidirectional data flow
- `provide` / `inject` for deep prop passing (theme, config, etc.)
- `defineEmits` for declaring component events
- `defineExpose` for exposing specific methods/properties to parent via ref

### Vue 3.5+ Features
- `useId()` for SSR-safe unique IDs
- `useTemplateRef()` for typed template refs
- Reactive props destructure: `const { count, name } = defineProps({ ... })` — reactive without `toRefs`
- `useModel()` for two-way binding with `defineModel()`

### Performance
- `v-once` for static content that never changes
- `v-memo` for conditional re-rendering optimization
- `shallowRef()` / `shallowReactive()` when deep reactivity is unnecessary
- Lazy-load routes with `defineAsyncComponent()`
- Keep components small — Vue's fine-grained reactivity means small components update efficiently

---

## Security

### XSS
- React's JSX auto-escapes, but `dangerouslySetInnerHTML` bypasses it
- Vue's `{{ }}` auto-escapes, but `v-html` bypasses it
- Sanitize any HTML rendered from user input (DOMPurify)
- Don't construct URLs from user input without validation

### CSRF
- Use CSRF tokens for state-changing requests with cookie-based auth
- `SameSite` cookie attribute helps but isn't sufficient alone
- JWT in Authorization header is CSRF-resistant (but has other trade-offs)

### Supply Chain
- Audit dependencies regularly: `npm audit`
- Pin versions in production: lockfiles (`package-lock.json`, `yarn.lock`)
- Review `postinstall` scripts — they can execute arbitrary code
- Use `npm ci` in CI (installs from lockfile, faster and more reliable)

### Client-Side
- Don't store sensitive data in `localStorage` — accessible to any script, including XSS
- Validate and sanitize input server-side — client validation is UX only
- Content Security Policy (CSP) headers to restrict resource loading
- Use `HttpOnly`, `Secure`, `SameSite` flags for cookies

---

## Performance

### Bundle Size
- Tree-shaking: use named imports, avoid side effects
- Dynamic imports for code splitting: `import('./module').then(m => ...)`
- Analyze bundle: `webpack-bundle-analyzer`, `vite-plugin-visualizer`
- Don't import entire libraries: `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`

### Runtime Performance
- Debounce/throttle event handlers: scroll, resize, input
- Virtual scrolling for lists > 100 items
- `requestAnimationFrame` for visual updates
- Web Workers for CPU-intensive computation
- `IntersectionObserver` for lazy loading images and infinite scroll

### Memory
- Clean up event listeners and subscriptions on unmount (React `useEffect` cleanup, Vue `onUnmounted`)
- Avoid closures that retain large objects unnecessarily
- WeakMap/WeakSet for caches that shouldn't prevent garbage collection
- Disconnect observers (IntersectionObserver, ResizeObserver, MutationObserver) on unmount

---

## Testing

### Unit Testing (Vitest / Jest)
- Test behavior, not implementation details
- `@testing-library/react` for React: test what users see, not component internals
- `@vue/test-utils` for Vue: shallow mount for unit tests, full mount for integration
- Mock external APIs, not internal modules

### Integration Testing
- MSW (Mock Service Worker) for API mocking — works in both unit and E2E
- Test user flows: form submission, navigation, error states
- Test loading states and error boundaries

### E2E Testing
- Playwright preferred over Cypress for reliability and cross-browser support
- Test critical user journeys, not every possible path
- Use data-testid attributes for stable selectors

---

## Modern JS (ES2022+)

- **Top-level await**: In ES modules, no wrapper async function needed
- **Array.at()**: `arr.at(-1)` for last element (no more `arr[arr.length - 1]`)
- **Object.hasOwn()**: Safer than `Object.prototype.hasOwnProperty.call()`
- **Structured clone**: `structuredClone(obj)` for deep cloning (handles Date, Map, Set, RegExp, etc.)
- **Error cause**: `new Error('message', { cause: originalError })` for error chains
