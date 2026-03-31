# Python 代码审查指南

## 目录
1. [语言基础](#语言基础)
2. [类型提示与静态分析](#类型提示与静态分析)
3. [错误处理](#错误处理)
4. [并发与异步](#并发与异步)
5. [安全](#安全)
6. [性能](#性能)
7. [测试](#测试)
8. [常见框架](#常见框架)
9. [现代 Python（3.10+）](#现代-python)

---

## 语言基础

### 可变默认参数
Python 排第一的陷阱。默认参数在函数定义时求值一次。

```python
# BUG：所有调用共享同一个列表
def append_to(item, target=[]):
    target.append(item)
    return target

# CORRECT：
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

### 作用域与闭包
- `for` 循环变量在闭包中按引用捕获——延迟绑定问题
- 使用默认参数技巧或 `functools.partial` 捕获当前值
- 生产代码中 `global` 和 `nonlocal` 是代码异味——优先使用显式状态

### 对象同一性 vs 相等性
- `is` 检查同一性（同一对象），`==` 检查相等性（相同值）
- `is` 仅用于 `None`、`True`、`False`、哨兵值
- 不要用 `is` 比较整数、字符串——CPython 缓存小整数，但这是实现细节

### 迭代模式
- 不要在迭代时修改集合——先收集变更，迭代后应用
- 使用 `enumerate()` 代替手动计数器变量
- 并行迭代使用 `zip()`，不同长度使用 `itertools.zip_longest()`
- 大数据集使用生成器表达式实现内存高效处理

### 字符串格式化
- f-string 是首选（Python 3.6+）：`f"Hello {name}"`
- 日志使用 `%s` 格式化（惰性求值）：`logger.info("User %s logged in", user_id)`
- 不要在日志消息中使用 f-string——即使日志级别被禁用也会被求值

---

## 类型提示与静态分析

### 类型标注质量
- 使用具体类型：`list[User]` 而非 `list`，`str | None` 而非 `Optional[str]`（Python 3.10+）
- 字典形状使用 `TypedDict` 而非 `dict[str, Any]`
- 类枚举字符串使用 `Literal`：`def set_mode(mode: Literal["read", "write"])`
- 鸭子类型接口使用 `Protocol` 而非 `Any`
- 避免 `Any`——如果必须使用，文档说明原因

### 泛型模式
- 泛型函数使用 `TypeVar`：`T = TypeVar("T")`
- 多种有效签名的函数使用 `@overload`
- 函数类型提示使用 `Callable[[int, str], bool]`
- 类级别属性（非实例属性）使用 `ClassVar`

### 静态分析配置
- 推荐 `mypy --strict` 或至少 `mypy` 配合 `disallow_untyped_defs = true`
- `pyright` 更快且捕获不同的问题——考虑同时使用
- 添加 `type: ignore` 注释并说明原因：`type: ignore[arg-type]  # 第三方 API 不匹配`

---

## 错误处理

### 异常设计
- 创建自定义异常层次：`AppError` → `NotFoundError`、`ValidationError`
- 包含上下文：`raise NotFoundError(f"User {user_id} not found")`
- 多个相关错误使用异常组（Python 3.11+）

### 反模式
```python
# BAD：裸 except 会捕获 SystemExit、KeyboardInterrupt
except:
    pass

# BAD：范围太宽
except Exception:
    pass

# BAD：丢失堆栈跟踪
except ValueError as e:
    raise RuntimeError(str(e))  # 丢失原始 traceback

# CORRECT：保留异常链
except ValueError as e:
    raise RuntimeError(f"Processing failed: {e}") from e
```

### 错误处理模式
- 使用 `try/except/else/finally` 惯用法：try 执行风险操作，except 处理错误，else 成功时运行，finally 始终运行
- 资源清理使用上下文管理器（`with` 语句）
- 故意忽略的异常使用 `contextlib.suppress()`

---

## 并发与异步

### async/await
- 不要不 `await` 就调用 async 函数——创建了永远不会执行的协程
- 不要在 async 上下文中使用阻塞调用：`time.sleep()`、`requests.get()`、同步数据库驱动
- 不可避免的阻塞 I/O 使用 `asyncio.to_thread()`
- 并发执行使用 `asyncio.gather()`，但用 `return_exceptions=True` 处理异常

### 线程
- GIL 阻止 CPU 密集型工作的真正并行——使用 `multiprocessing`
- `threading` 适合 I/O 密集型工作
- 共享可变状态始终使用锁
- `queue.Queue` 是线程安全的，`collections.deque` 不是（尽管有些原子操作）

### 常见陷阱
- check-then-act 竞态条件：`if not os.path.exists(path): os.makedirs(path)` → 使用 `exist_ok=True`
- 线程间共享可变状态没有同步
- 忘记 await 协程——它们静默地什么都不做
- `asyncio.run()` 不能在已有事件循环中调用

---

## 安全

### 注入
- **SQL 注入**：始终使用参数化查询。绝不用 f-string 或 `%` 格式化 SQL
  ```python
  # BAD
  cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
  # CORRECT
  cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
  ```
- **命令注入**：使用 `subprocess.run()` 的列表参数，绝不使用 shell=True 加字符串插值
- **模板注入**：不要对用户输入使用 `eval()`、`exec()` 或 `__import__()`
- **路径穿越**：使用 `pathlib.Path.resolve()` 并验证路径在预期目录内

### 反序列化
- 对不可信数据使用 `pickle.loads()` 等同于任意代码执行——使用 `json` 或 `msgpack`
- `yaml.load()` 不指定 `Loader=yaml.SafeLoader` 是不安全的——使用 `yaml.safe_load()`
- 注意对象中可能被利用的 `__reduce__` 方法

### 密钥与凭证
- 绝不硬编码密钥——使用环境变量或密钥管理器
- 不要记录敏感数据：密码、token、API Key、PII
- `.env` 文件应加入 `.gitignore`——改为提供 `.env.example`
- 注意异常消息和调试输出中的密钥

### 依赖
- 锁定依赖版本：`fastapi==0.104.1` 而非 `fastapi`
- 使用 `pip-audit` 或 `safety` 检查已知漏洞
- 审查 `requirements.txt` / `pyproject.toml` 中的可疑或不必要依赖

---

## 性能

### 常见陷阱
- 循环中的字符串拼接：使用 `"".join(list_of_strings)`
- 列表上的 `in` 运算符：O(n)——转为 set 实现 O(1) 查找
- 循环中重复的函数调用：缓存或提前计算
- 不必要的列表物化：大数据使用生成器

### 数据结构
- 循环中使用 `collections.defaultdict` 替代 `dict.setdefault()`
- 计数使用 `collections.Counter`
- 仅在顺序重要时使用 `collections.OrderedDict`（3.7 起普通字典保留插入顺序）
- Top-N 问题使用 `heapq` 而非排序整个列表

### 内存
- 大数据集使用生成器（`yield`）而非构建完整列表
- `sys.getsizeof()` 仅测量容器，不包含内容——嵌套结构使用 `pympler`
- 大量实例的类使用 `__slots__` 节省内存
- 注意阻止垃圾回收的循环引用

### 性能信号
- N+1 数据库查询（与 Java 相同——通用性能 Bug）
- 请求处理器中的同步 I/O
- 频繁访问但很少变化的数据缺少缓存
- 将整个数据集加载到内存而非流式处理

---

## 测试

### pytest 最佳实践
- Fixture 用于测试准备，而非测试逻辑
- 昂贵的准备工作使用 `@pytest.fixture(scope="module")` 或 `scope="session"`
- 参数化测试：`@pytest.mark.parametrize("input,expected", [(1, 2), (3, 6)])`
- 测试文件命名 `test_*.py`，类命名 `Test*`，函数命名 `test_*`

### 测试质量
- 每个测试应该独立——无顺序依赖
- 不要 mock 你不拥有的东西——mock 外部 API，而非自己的业务逻辑
- HTTP mock 使用 `responses` 或 `respx`，时间相关测试使用 `freezegun`
- 测试错误路径，不仅仅是正常路径

### 集成测试
- 数据层测试使用测试数据库，而非 mock
- `factory_boy` 用于生成测试数据
- 测试间清理状态：截断或事务回滚

---

## 常见框架

### FastAPI
- 数据库会话、认证、配置使用依赖注入
- 请求/响应校验使用 Pydantic 模型——不要接受原始 dict
- `BackgroundTasks` 用于即发即弃，Celery 用于持久化任务队列
- 使用 `APIRouter` 分组相关端点

### Django
- 除非必要不要绕过 ORM 使用原始 SQL（并文档说明原因）
- 使用 `select_related()` 和 `prefetch_related()` 避免 N+1
- 批量操作使用 Django ORM 的 `bulk_create()` / `bulk_update()`
- 横切关注点使用中间件，而非视图

### Flask
- 使用应用工厂模式支持测试和多配置
- 使用蓝图组织路由
- 不要在全局变量中存储状态——使用扩展或应用上下文

---

## 现代 Python

### Python 3.10+
- **结构化模式匹配**：`match` / `case`——比冗长的 `if/elif` 链更清晰
- **联合类型**：`X | Y` 替代 `Union[X, Y]`
- **类型守卫**：`TypeGuard` 用于 `isinstance` 检查中的类型收窄

### Python 3.11+
- **异常组**：`except*` 用于处理多个异常
- **`ExceptionGroup`**：抛出和捕获多个相关异常
- **`tomllib`**：内置 TOML 解析（不再需要第三方依赖）

### Python 3.12+
- **类型参数语法**：`def func[T](x: T) -> T` 替代 `TypeVar`
- **`type` 语句**：`type Point = tuple[float, float]` 用于类型别名
- **F-string 改进**：可以使用相同引号类型、嵌套表达式
