# TypeScript 代码审查指南

## 目录
1. [类型系统](#类型系统)
2. [严格模式](#严格模式)
3. [泛型](#泛型)
4. [常见类型错误](#常见类型错误)
5. [React + TypeScript](#react--typescript)
6. [Vue + TypeScript](#vue--typescript)
7. [API 设计](#api-设计)
8. [工具链](#工具链)

---

## 类型系统

### 对象优先使用 Interface，联合类型使用 Type
```typescript
// Interface：可扩展，可合并
interface User {
  id: string;
  name: string;
}

// Type：联合类型、交叉类型、映射类型
type Status = "active" | "inactive" | "suspended";
type Result<T> = { success: true; data: T } | { success: false; error: string };
```

### 避免 `any`——使用更安全的替代方案
```typescript
// BAD：禁用所有类型检查
function process(data: any) { ... }

// BETTER：unknown 配合类型收窄
function process(data: unknown) {
  if (typeof data === "string") { ... }
}

// OR：具体类型
function process(data: Record<string, unknown>) { ... }
```

`any` 仅在与无类型的第三方代码交互的边界处可接受，且应隔离并文档说明。

### 类型收窄
- 使用可辨识联合：`{ type: "success"; data: T } | { type: "error"; error: E }`
- 使用类型谓词：`function isString(val: unknown): val is string`
- 使用 `in` 运算符：`"name" in obj` 进行收窄
- 避免类型断言（`as`）——改用类型守卫

### 字面量类型与枚举
- 字符串字面量联合优于枚举：`type Direction = "up" | "down" | "left" | "right"`
- Const enum 有陷阱（tree-shaking 问题，不能跨包使用）
- `as const` 用于字面量类型推断：`const ROLES = ["admin", "user"] as const`

---

## 严格模式

### `tsconfig.json` 必要设置
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
数组和对象索引访问返回 `T | undefined`。这捕获了一类常见的 Bug：
```typescript
const users: User[] = getUsers();
const user = users[0]; // 类型：User | undefined，不是 User
user.name; // 报错：Object is possibly undefined
```

---

## 泛型

### 泛型约束
```typescript
// BAD：无约束——T 可以是任何东西
function getProperty<T, K>(obj: T, key: K) { ... }

// GOOD：key 必须是 T 的键
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### 不要过度泛化
- 如果泛型只在一处使用，具体类型更清晰
- 如果泛型类型参数在签名中只出现一次，可能不需要泛型
- "接受任何类型"的场景优先使用 `unknown` 而非不必要的泛型

### 条件类型与映射类型
- `Partial<T>`、`Required<T>`、`Pick<T, K>`、`Omit<T, K>`——使用内置工具类型
- `Record<K, V>` 用于类型化字典
- `ReturnType<T>` 和 `Parameters<T>` 用于从函数推断
- 不要创建与内置重复的自定义工具类型

---

## 常见类型错误

### 类型断言
```typescript
// BAD：对类型系统撒谎
const user = data as User;

// BETTER：运行时校验
const user = UserSchema.parse(data); // zod、valibot 等

// ACCEPTABLE：模块边界处有运行时检查
const user = data as User;
if (!user.id || !user.name) throw new Error("Invalid user");
```

### 非空断言
```typescript
// BAD：实际上没有检查
const el = document.getElementById("app")!;

// BETTER：显式检查
const el = document.getElementById("app");
if (!el) throw new Error("Element not found");
```

### 枚举陷阱
- 数字枚举是双向的（可以赋任意数字）——优先使用字符串枚举或联合类型
- `const enum` 的内联在跨包边界时会出问题
- 现代 TypeScript 中通常推荐使用联合类型而非枚举

### 声明文件
- 不要使用 `@ts-ignore`——使用 `@ts-expect-error`（下一行没有类型错误时会报错）
- 无类型包使用自定义 `.d.ts` 文件——可能的话贡献到 DefinitelyTyped
- `declare module "untyped-lib"` 应只作为临时措施

---

## React + TypeScript

### 组件类型定义
```typescript
// 推荐方式：内联 props 类型
interface UserCardProps {
  user: User;
  onEdit: (id: string) => void;
  isLoading?: boolean;
}

function UserCard({ user, onEdit, isLoading = false }: UserCardProps) { ... }
```

### Hook 类型定义
```typescript
// useState
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);

// useRef
const inputRef = useRef<HTMLInputElement>(null); // DOM ref 用 null
const timerRef = useRef<ReturnType<typeof setTimeout>>();

// useReducer
type Action = { type: "increment" } | { type: "decrement" };
function reducer(state: number, action: Action): number { ... }
```

### 事件类型定义
```typescript
function handleSubmit(e: React.FormEvent<HTMLFormElement>) { ... }
function handleClick(e: React.MouseEvent<HTMLButtonElement>) { ... }
function handleChange(e: React.ChangeEvent<HTMLInputElement>) { ... }
```

### 泛型组件
```typescript
// 泛型列表组件
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) { ... }
```

---

## Vue + TypeScript

### 组件类型定义（Vue 3.5+）
```vue
<script setup lang="ts">
// 带默认值的 Props
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

### Ref 类型定义
```typescript
const count = ref<number>(0);
const user = ref<User | null>(null);
const items = ref<string[]>([]);

// 模板 ref
const inputRef = useTemplateRef<HTMLInputElement>("input");
```

### Composable 类型定义
```typescript
// 返回类型应该被推断，而非显式标注
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

### Provide/Inject 类型定义
```typescript
// Provider
provide<UserService>(UserServiceKey, service);

// Consumer
const service = inject<UserService>(UserServiceKey);
if (!service) throw new Error("UserService not provided");
```

---

## API 设计

### 运行时校验
API 边界使用 Zod、Valibot 或类似库：
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

始终在系统边界处校验：API 响应、表单输入、URL 参数、localStorage 读取。

### API 响应类型定义
```typescript
// 类型化的 fetch 封装
async function fetchJson<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json() as Promise<T>;
}
```

### 错误类型定义
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

## 工具链

### ESLint 配置
- `@typescript-eslint/recommended` 作为基线
- `@typescript-eslint/strict-type-checked` 用于更严格的规则
- 关键规则：
  - `no-explicit-any`——标记或禁止 `any`
  - `no-unsafe-assignment`——禁止从 any 隐式获取 any
  - `consistent-type-imports`——仅类型导入使用 `import type { X }`
  - `no-floating-promises`——每个 Promise 都必须被处理

### 项目结构
- 类型定义与它们描述的代码放在一起
- 共享类型放在 `types/` 或 `@types/` 目录
- 不要创建单个 `types.ts` 巨型文件
- 谨慎使用 `index.ts` barrel 导出——它们可能影响 tree-shaking
