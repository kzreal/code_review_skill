# TypeScript Code Review Guide

## Table of Contents
1. [Type System](#type-system)
2. [Strict Mode](#strict-mode)
3. [Generics](#generics)
4. [Common Type Mistakes](#common-type-mistakes)
5. [React + TypeScript](#react--typescript)
6. [Vue + TypeScript](#vue--typescript)
7. [API Design](#api-design)
8. [Tooling](#tooling)

---

## Type System

### Prefer Interfaces for Objects, Types for Unions
```typescript
// Interface: extensible, can be merged
interface User {
  id: string;
  name: string;
}

// Type: unions, intersections, mapped types
type Status = "active" | "inactive" | "suspended";
type Result<T> = { success: true; data: T } | { success: false; error: string };
```

### Avoid `any` — Use Safer Alternatives
```typescript
// BAD: disables all type checking
function process(data: any) { ... }

// BETTER: unknown with narrowing
function process(data: unknown) {
  if (typeof data === "string") { ... }
}

// OR: specific type
function process(data: Record<string, unknown>) { ... }
```

`any` is acceptable only at boundaries with untyped third-party code, and should be isolated and documented.

### Type Narrowing
- Use discriminated unions: `{ type: "success"; data: T } | { type: "error"; error: E }`
- Use type predicates: `function isString(val: unknown): val is string`
- Use `in` operator: `"name" in obj` to narrow
- Avoid type assertions (`as`) — use type guards instead

### Literal Types & Enums
- Prefer union of literal strings over enums: `type Direction = "up" | "down" | "left" | "right"`
- Const enums have pitfalls (tree-shaking issues, cannot be used across packages)
- `as const` for literal type inference: `const ROLES = ["admin", "user"] as const`

---

## Strict Mode

### Must-Have `tsconfig.json` Settings
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### `noUncheckedIndexedAccess`
Array and object index access returns `T | undefined`. This catches a common class of bugs:
```typescript
const users: User[] = get Users();
const user = users[0]; // Type: User | undefined, not User
user.name; // Error: Object is possibly undefined
```

---

## Generics

### Generic Constraints
```typescript
// BAD: no constraint — T can be anything
function getProperty<T, K>(obj: T, key: K) { ... }

// GOOD: key must be a key of T
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### Don't Over-Genericize
- If a generic is only used in one place, a concrete type is clearer
- If the generic type parameter appears only once in the signature, it might not need to be generic
- Prefer `unknown` over unnecessary generics for "accept anything" scenarios

### Conditional & Mapped Types
- `Partial<T>`, `Required<T>`, `Pick<T, K>`, `Omit<T, K>` — use built-in utility types
- `Record<K, V>` for typed dictionaries
- `ReturnType<T>` and `Parameters<T>` for inferring from functions
- Don't create custom utility types that duplicate built-in ones

---

## Common Type Mistakes

### Type Assertions
```typescript
// BAD: lying to the type system
const user = data as User;

// BETTER: runtime validation
const user = UserSchema.parse(data); // zod, valibot, etc.

// ACCEPTABLE: at module boundaries with runtime check
const user = data as User;
if (!user.id || !user.name) throw new Error("Invalid user");
```

### Non-null Assertion
```typescript
// BAD: doesn't actually check
const el = document.getElementById("app")!;

// BETTER: explicit check
const el = document.getElementById("app");
if (!el) throw new Error("Element not found");
```

### Enum Pitfalls
- Numeric enums are bidirectional (can assign any number) — prefer string enums or union types
- `const enum` inlining breaks across package boundaries
- Union types are generally preferred over enums in modern TypeScript

### Declaration Files
- Don't use `@ts-ignore` — use `@ts-expect-error` (it errors if the next line doesn't have a type error)
- Custom `.d.ts` files for untyped packages — contribute to DefinitelyTyped instead if possible
- `declare module "untyped-lib"` should only be a temporary measure

---

## React + TypeScript

### Component Typing
```typescript
// Preferred: inline props type
interface UserCardProps {
  user: User;
  onEdit: (id: string) => void;
  isLoading?: boolean;
}

function UserCard({ user, onEdit, isLoading = false }: UserCardProps) { ... }
```

### Hook Typing
```typescript
// useState
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// useRef
const inputRef = useRef<HTMLInputElement>(null); // null for DOM refs
const timerRef = useRef<ReturnType<typeof setTimeout>>();

// useReducer
type Action = { type: "increment" } | { type: "decrement" };
function reducer(state: number, action: Action): number { ... }
```

### Event Typing
```typescript
function handleSubmit(e: React.FormEvent<HTMLFormElement>) { ... }
function handleClick(e: React.MouseEvent<HTMLButtonElement>) { ... }
function handleChange(e: React.ChangeEvent<HTMLInputElement>) { ... }
```

### Generic Components
```typescript
// Generic list component
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) { ... }
```

---

## Vue + TypeScript

### Component Typing (Vue 3.5+)
```vue
<script setup lang="ts">
// Props with defaults
interface Props {
  title: string;
  count?: number;
}
const { title, count = 0 } = defineProps<Props>();

// Emits
const emit = defineEmits<{
  update: [value: string];
  delete: [id: number];
}>();
</script>
```

### Ref Typing
```typescript
const count = ref<number>(0);
const user = ref<User | null>(null);
const items = ref<string[]>([]);

// Template ref
const inputRef = useTemplateRef<HTMLInputElement>("input");
```

### Composable Typing
```typescript
// Return type should be inferred, not explicitly typed
function useUser(id: Ref<string>) {
  const user = ref<User | null>(null);
  const isLoading = ref(false);

  async function fetchUser() {
    isLoading.value = true;
    user.value = await api.getUser(id.value);
    isLoading.value = false;
  }

  return { user, isLoading, fetchUser };
}
```

### Provide/Inject Typing
```typescript
// Provider
provide<UserService>(UserServiceKey, service);

// Consumer
const service = inject<UserService>(UserServiceKey);
if (!service) throw new Error("UserService not provided");
```

---

## API Design

### Runtime Validation
Use Zod, Valibot, or similar for API boundaries:
```typescript
import { z } from "zod";

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

type User = z.infer<typeof UserSchema>;
```

Always validate at system boundaries: API responses, form inputs, URL params, localStorage reads.

### API Response Typing
```typescript
// Typed fetch wrapper
async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json() as Promise<T>;
}
```

### Error Typing
```typescript
class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public statusCode: number,
    public cause?: Error
  ) {
    super(message);
  }
}
```

---

## Tooling

### ESLint Configuration
- `@typescript-eslint/recommended` as baseline
- `@typescript-eslint/strict-type-checked` for stricter rules
- Key rules to enforce:
  - `no-explicit-any` — flag or ban `any`
  - `no-unsafe-assignment` — no implicit any from any
  - `consistent-type-imports` — `import type { X }` for type-only imports
  - `no-floating-promises` — every promise must be handled

### Project Structure
- Co-locate types with the code they describe
- Shared types in `types/` or `@types/` directory
- Don't create a single `types.ts` mega-file
- Use `index.ts` barrel exports sparingly — they can hurt tree-shaking
