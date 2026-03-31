# JavaScript 代码审查指南

## 目录
1. [语言基础](#语言基础)
2. [异步模式](#异步模式)
3. [错误处理](#错误处理)
4. [React 模式](#react-模式)
5. [Vue 模式](#vue-模式)
6. [安全](#安全)
7. [性能](#性能)
8. [测试](#测试)
9. [现代 JS（ES2022+）](#现代-js)

---

## 语言基础

### 变量声明
- 默认使用 `const`，需要重新赋值时用 `let`，绝不用 `var`
- `const` 对象/数组仍可被修改——需要不可变性时使用 `Object.freeze()` 或 `readOnly` 包装器
- 浏览器环境中避免在全局作用域声明变量

### 相等与类型转换
- 始终使用 `===` 和 `!==`——绝不使用 `==` / `!=`
- 注意真值/假值陷阱：`0`、`""`、`null`、`undefined`、`NaN`、`false` 都是假值
- `typeof null === "object"`——历史遗留 Bug，用 `=== null` 检查
- `NaN !== NaN`——使用 `Number.isNaN()` 检查

### 对象与数组陷阱
- `Object.assign()` 和展开运算符（`{...obj}`）是浅拷贝——嵌套对象是共享的
- `JSON.parse(JSON.stringify(obj))` 深拷贝在以下类型上会失败：`Date`、`RegExp`、`Map`、`Set`、`undefined`、函数、循环引用
- 数组方法：`map`、`filter`、`reduce` 返回新数组；`sort`、`splice` 会修改原数组
- `Array.prototype.sort()` 默认按字符串比较排序：`[10, 9, 8].sort()` → `[10, 8, 9]`

### 闭包与作用域
- `for (var i = ...)` 按引用捕获变量——所有闭包共享同一个 `i`
- `let`/`const` 的块作用域修复了这个问题
- React 中的过期闭包：事件处理器捕获了旧状态——使用函数式更新或 ref

### `this` 绑定
- 箭头函数不绑定 `this`——从外层作用域捕获
- 普通函数的 `this` 取决于调用位置
- React class 组件：必须绑定方法或使用箭头函数类字段
- Vue 3 Composition API 中不使用 `this`——箭头函数完全没问题

---

## 异步模式

### Promise
- 始终处理 rejection——未处理的 Promise rejection 可能导致 Node.js 崩溃
- `Promise.all()` 快速失败：一个 rejection 就拒绝整个结果
- 需要所有结果（不论个别是否失败）时使用 `Promise.allSettled()`
- 可以返回/链式已有 Promise 时不要创建新 Promise（Promise 构造函数反模式）

### async/await
- 每个 `await` 都应在 `try/catch` 中，或函数应向上传播错误
- 互不依赖的操作使用顺序 `await` 是性能 Bug——使用 `Promise.all()`
- ES 模块中的顶层 await 没问题，但会阻塞模块执行

```javascript
// BAD：可以并行却顺序执行
const user = await getUser(id);
const orders = await getOrders(id); // 不依赖 user

// GOOD：并行执行
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

### 异步中的错误处理
- `async` 函数始终返回 Promise——错误变为 rejection
- 不要混用 `callback` 和 `promise` 模式
- 可取消的异步操作使用 `AbortController`（fetch 等）

---

## 错误处理

### 反模式
```javascript
// BAD：空 catch
try { something(); } catch (e) {}

// BAD：丢失错误信息
catch (e) { throw new Error('Failed'); }

// CORRECT：保留错误链
catch (e) { throw new Error(`Failed: ${e.message}`, { cause: e }); }
```

### 推荐模式
- 领域特定错误使用继承 `Error` 的自定义错误类
- `error.cause`（ES2022）用于错误链
- React 中使用 Error Boundaries，Vue 中使用 `errorCaptured` 处理 UI 错误
- API 错误使用集中式错误处理器（拦截器等）

---

## React 模式

### Hooks 规则
- 只在顶层调用 Hooks——不要在循环、条件或嵌套函数中调用
- 只在 React 函数组件或自定义 Hooks 中调用 Hooks
- `useEffect` / `useMemo` / `useCallback` 的依赖必须完整且正确
- 过期闭包是第一大的 Bug 来源：在 timeout/interval/subscription 中访问状态但没有正确的 ref

### useEffect 陷阱
- 缺少依赖 → 过期数据、行为不正确
- 对象/数组作为依赖导致无限循环（每次渲染新引用）→ 使用 memo 或基本类型依赖
- useEffect 中 fetch 无清理 → 快速重渲染时的竞态条件 → 使用 AbortController
- `useEffect` 在绘制后执行——布局测量不要用它（使用 `useLayoutEffect`）

### 状态管理
- 不要用 state 存储可计算得出的值：`const fullName = `${firstName} ${lastName}`——不需要 state
- 状态提升：兄弟组件共享状态时，移到它们的共同父组件
- 避免超过 2-3 层的 prop drilling——使用 Context、Zustand 或其他状态管理
- 多个相关值的复杂状态逻辑使用 `useReducer`

### 组件设计
- 一个组件一个职责——组件做了太多事时拆分
- 可复用逻辑提取为自定义 Hooks
- 正确使用 `key` prop：稳定、唯一的标识符，不要用数组索引（重排序时会导致 Bug）
- 避免 JSX 中的内联对象/数组：`style={{ color: 'red' }}` 每次渲染创建新对象

### React 18/19 特性
- `useTransition()` 用于标记非紧急的状态更新
- `useDeferredValue()` 用于延迟昂贵的渲染
- `useId()` 用于生成稳定的唯一 ID（不用于 key）
- React 19：`useActionState()` 用于表单状态，`useOptimistic()` 用于乐观 UI
- React 19：Server Components——不要在服务端组件中使用 Hooks 或浏览器 API
- React 19：`use()` hook 用于在渲染中消费 Promise 和 Context

### 性能
- `React.memo()` 用于频繁重渲染但 props 相同的组件
- `useMemo()` 用于昂贵计算，不要每个值都用
- `useCallback()` 仅在传递回调给 memo 化的子组件时使用
- 长列表虚拟化：`react-window` 或 `react-virtuoso`
- 代码分割：`React.lazy()` + `Suspense` 用于路由级分割

---

## Vue 模式

### Composition API（Vue 3.5+）
- 使用 `<script setup>` 语法——更简洁、性能更好
- 基本类型使用 `ref()`，对象使用 `reactive()`——项目中保持一致
- `toRefs()` / `toRef()` 解构 reactive 对象而不丢失响应性
- 派生状态使用 `computed()`——不要在 `watch` 或手动赋值中计算

### 响应性陷阱
- 解构 `reactive()` 对象会丢失响应性——使用 `toRefs()`
- `ref()` 在 `<script>` 中需要 `.value` 访问（模板中自动解包）
- 不要直接修改 props——通过 emit 事件通知父组件
- `watchEffect()` 自动追踪依赖，`watch()` 是显式的——有意识地选择

### 组件通信
- Props 向下，Events 向上——标准的单向数据流
- `provide` / `inject` 用于深层 prop 传递（主题、配置等）
- `defineEmits` 用于声明组件事件
- `defineExpose` 用于通过 ref 向父组件暴露特定方法/属性

### Vue 3.5+ 特性
- `useId()` 用于 SSR 安全的唯一 ID
- `useTemplateRef()` 用于类型化的模板 ref
- 响应式 props 解构：`const { count, name } = defineProps({ ... })`——无需 `toRefs` 即可响应
- `useModel()` 配合 `defineModel()` 实现双向绑定

### 性能
- `v-once` 用于永不变化的静态内容
- `v-memo` 用于条件重渲染优化
- `shallowRef()` / `shallowReactive()` 当深层响应性不必要时
- 使用 `defineAsyncComponent()` 懒加载路由
- 保持组件小巧——Vue 的细粒度响应性使小组件更新更高效

---

## 安全

### XSS
- React 的 JSX 自动转义，但 `dangerouslySetInnerHTML` 会绕过
- Vue 的 `{{ }}` 自动转义，但 `v-html` 会绕过
- 渲染来自用户输入的 HTML 时必须消毒（DOMPurify）
- 不要用未经校验的用户输入构造 URL

### CSRF
- Cookie 认证的状态变更请求使用 CSRF token
- `SameSite` Cookie 属性有帮助但不足以单独依赖
- Authorization header 中的 JWT 天然防 CSRF（但有其他权衡）

### 供应链
- 定期审计依赖：`npm audit`
- 生产环境锁定版本：lockfile（`package-lock.json`、`yarn.lock`）
- 审查 `postinstall` 脚本——它们可以执行任意代码
- CI 中使用 `npm ci`（从 lockfile 安装，更快更可靠）

### 客户端
- 不要在 `localStorage` 中存储敏感数据——任何脚本（包括 XSS）都可以访问
- 服务端校验和清理输入——客户端校验仅用于用户体验
- Content Security Policy（CSP）头限制资源加载
- Cookie 使用 `HttpOnly`、`Secure`、`SameSite` 标志

---

## 性能

### 包体积
- Tree-shaking：使用具名导入，避免副作用
- 动态导入用于代码分割：`import('./module').then(m => ...)`
- 分析包：`webpack-bundle-analyzer`、`vite-plugin-visualizer`
- 不要导入整个库：`import { debounce } from 'lodash-es'` 而非 `import _ from 'lodash'`

### 运行时性能
- 事件处理器使用 debounce/throttle：scroll、resize、input
- 超过 100 项的列表使用虚拟滚动
- 视觉更新使用 `requestAnimationFrame`
- CPU 密集型计算使用 Web Workers
- 图片懒加载和无限滚动使用 `IntersectionObserver`

### 内存
- 组件卸载时清理事件监听器和订阅（React `useEffect` 清理函数，Vue `onUnmounted`）
- 避免闭包不必要地持有大对象引用
- 不应阻止垃圾回收的缓存使用 WeakMap/WeakSet
- 组件卸载时断开 Observer（IntersectionObserver、ResizeObserver、MutationObserver）

---

## 测试

### 单元测试（Vitest / Jest）
- 测试行为，而非实现细节
- React 使用 `@testing-library/react`：测试用户看到的内容，而非组件内部
- Vue 使用 `@vue/test-utils`：单元测试用浅挂载，集成测试用完整挂载
- Mock 外部 API，而非内部模块

### 集成测试
- MSW（Mock Service Worker）用于 API mock——在单元和 E2E 测试中都可使用
- 测试用户流程：表单提交、导航、错误状态
- 测试加载状态和错误边界

### E2E 测试
- Playwright 优于 Cypress（可靠性和跨浏览器支持更好）
- 测试关键用户旅程，而非所有可能路径
- 使用 data-testid 属性作为稳定的选择器

---

## 现代 JS（ES2022+）

- **顶层 await**：ES 模块中不需要包装的 async 函数
- **Array.at()**：`arr.at(-1)` 获取最后一个元素（不再需要 `arr[arr.length - 1]`）
- **Object.hasOwn()**：比 `Object.prototype.hasOwnProperty.call()` 更安全
- **Structured clone**：`structuredClone(obj)` 深拷贝（处理 Date、Map、Set、RegExp 等）
- **Error cause**：`new Error('message', { cause: originalError })` 用于错误链
