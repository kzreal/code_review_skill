# 常见 Bug 检查清单

各语言常见的 Bug 模式，作为代码审查时的快速参考。

---

## Java

### NullPointerException
- 自动拆箱：方法返回 `Integer`，调用方赋值给 `int`
- `Map.get()` 对不存在的 key 返回 null——使用 `getOrDefault()` 或 `computeIfAbsent()`
- `Optional.get()` 未先检查 `isPresent()`
- 链式方法调用缺少 null 检查：`user.getAddress().getCity()`
- `@Autowired` 字段在注入前被访问（构造函数或生命周期方法中）

### 并发 Bug
- `SimpleDateFormat` 在多线程间共享——使用 `DateTimeFormatter`
- `HashMap` 并发修改——使用 `ConcurrentHashMap`
- 无同步的 check-then-act：`if (!cache.containsKey(k)) cache.put(k, v)`
- `@Async` / `@Transactional` 自调用（代理绕过）
- 缺少 `volatile` 的双重检查锁定

### Spring Boot 特有
- `@Transactional` 标注在 private 方法上（无效——Spring 使用代理）
- Controller 直接返回实体（事务外触发懒加载异常）
- `@Value` 缺少属性 → 运行时抛出 `IllegalArgumentException`
- 循环依赖仅在启动时发现 → 使用构造函数注入以尽早发现
- Filter/Interceptor 未指定顺序 → 执行顺序不符预期

### 资源泄漏
- `InputStream` / `OutputStream` 未在 finally 块中关闭（使用 try-with-resources）
- `Stream` 包装文件资源时未关闭
- `HttpClient` 按请求创建而非共享
- 异常路径上数据库连接未归还连接池

### 逻辑错误
- 枚举使用 `==`：可以工作但容易混淆，使用 `.equals()` 或 switch
- `Date` / `Calendar` 月份从 0 开始（一月 = 0）
- `List.remove(int)` 与 `List.remove(Object)`——行为不同
- `String.substring()` 在 Java 7+ 中创建新 String（不再共享底层数组）
- switch 缺少 `break` → 穿透执行

---

## Python

### 可变默认参数
```python
# BUG
def process(items, cache={}):
    cache[key] = value  # 所有调用共享同一个 dict

# FIX
def process(items, cache=None):
    if cache is None:
        cache = {}
```

### 闭包的延迟绑定
```python
# BUG：所有 lambda 捕获的是同一个 'i'
funcs = [lambda: i for i in range(5)]
# 全部返回 4

# FIX：默认参数捕获当前值
funcs = [lambda i=i: i for i in range(5)]
```

### 迭代中修改字典
```python
# BUG：RuntimeError
for key in d:
    if should_remove(key):
        del d[key]

# FIX
for key in list(d.keys()):
    if should_remove(key):
        del d[key]
```

### 整数缓存的陷阱
- `a is b` 对小整数（-5 到 256）有效是因为 CPython 缓存——但这是实现细节
- 值比较始终使用 `==`，不要用 `is`

### 常见差一错误
- `range(n)` 从 0 到 n-1
- 切片 `[start:end]` 包含 start，不包含 end
- `list.pop()` 移除并返回最后一个元素
- 负索引：`lst[-1]` 是最后一个元素

### 类型混淆
- `True == 1` 且 `False == 0`——布尔值是 int 的子类
- `bool("False")` 为 `True`——任何非空字符串都是真值
- `0 == False` 为 `True`——可能导致比较中的 Bug

---

## JavaScript / TypeScript

### 异步 Bug
```javascript
// BUG：本可并行却顺序执行
const a = await fetchA();
const b = await fetchB(); // 不依赖 a

// FIX
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

### 过期闭包
```javascript
// BUG：setTimeout 中的 count 始终为 0
const [count, setCount] = useState(0);
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // 始终为 1，因为 count 在挂载时被捕获
  }, 1000);
  return () => clearInterval(id);
}, []);

// FIX：函数式更新
setCount(prev => prev + 1);
```

### 数组排序 Bug
```javascript
// BUG：按字符串比较排序
[10, 9, 8, 80].sort() // → [10, 8, 80, 9]

// FIX：数值比较器
[10, 9, 8, 80].sort((a, b) => a - b) // → [8, 9, 10, 80]
```

### React Key 反模式
```jsx
// BUG：使用 index 作为 key 在重排序时导致状态错乱
items.map((item, index) => <Item key={index} {...item} />)

// FIX：使用稳定的唯一 id
items.map(item => <Item key={item.id} {...item} />)
```

### 真值/假值陷阱
- `if (count)` 在 count 为 0 时为 false——使用 `if (count > 0)` 或 `if (count !== undefined)`
- `if (items.length)` 没问题，但 `if (items)` 对数组始终为 true
- `NaN !== NaN`——使用 `Number.isNaN()`
- `typeof null === "object"`——用 `=== null` 检查

### 对象引用 Bug
```javascript
// BUG：两者共享同一个数组
const defaults = { items: [] };
const config = { ...defaults };
config.items.push("a"); // defaults.items 也被修改了！

// FIX：深拷贝或分离默认值
const config = { items: [...defaults.items] };
```

### TypeScript 特有
- 类型断言（`as`）不改变运行时行为——数据可能与断言的类型不匹配
- 非空断言（`!`）不会在运行时检查 null
- TypeScript 的 `enum` 会编译为运行时 JavaScript——行为可能不符合预期
- `keyof typeof obj` 用于从 const 对象获取键类型

---

## 跨语言模式

### 日期/时间 Bug
- 时区：存储始终使用 UTC，显示时再转换
- 夏令时切换：1 小时出现/消失——使用日期库，不要手动计算
- 闰秒/闰年：测试 2 月 29 日的边界情况
- JavaScript 的 `Date` 是可变的——小心共享实例

### 字符串编码
- 文本处理始终使用 UTF-8
- 注意系统边界处的编码不匹配（文件 I/O、HTTP、数据库）
- URL 编码 vs HTML 编码：根据上下文使用正确的编码
- Base64：不是加密，不是压缩

### 数值精度
- 浮点数：`0.1 + 0.2 !== 0.3`（JavaScript 和 Java 中均如此）——金额使用 `BigDecimal` 或 `Decimal`
- 整数溢出：Java `int` 最大约 21 亿——ID 和时间戳使用 `long`
- Python：整数精度无限制，但 float 与其他语言有相同限制

### 资源清理
- 始终关闭：数据库连接、文件句柄、HTTP 连接、流
- 使用语言适当的模式：try-with-resources、上下文管理器、useEffect 清理、onUnmounted
- 按获取的相反顺序清理
- 注意错误路径上的资源泄漏
