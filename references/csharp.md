# C# / .NET 代码审查指南

## 目录
1. [C# 通用](#c-通用)
2. [ASP.NET Core](#aspnet-core)
3. [Entity Framework Core](#entity-framework-core)
4. [并发与异步](#并发与异步)
5. [错误处理](#错误处理)
6. [安全](#安全)
7. [测试](#测试)
8. [现代 C#（10/12/13）](#现代-c)
9. [WPF / 桌面应用](#wpf--桌面应用)

---

## C# 通用

### Null 安全
- 在 `.csproj` 中启用 `<Nullable>enable</Nullable>`——新项目将 null 警告视为错误
- 可空引用类型使用 `?`：`string? name` 而非 `string name`（当 null 是可能值时）
- 仅在你能证明非 null 时使用 null 宽恕运算符 `!`——并说明原因
- 生产环境中出现 `NullReferenceException` 意味着可空标注有误或缺失
- null 检查优先使用模式匹配：`if (user is null)` 或 `if (user is { })`

### 资源管理
- `IDisposable` 始终使用 `using` 声明或 `using` 块：`SqlConnection`、`FileStream`、`HttpClient`、`StreamReader`
- `using var` 声明（C# 8+）用于基于作用域的清理——比块形式更简洁
- 注意存储在字段中的 `IDisposable` 未实现 `Dispose` 模式
- `Task` 和 `IAsyncEnumerable` 应被 await 或 dispose——绝不能 fire-and-forget

### 集合
- 使用集合表达式（C# 12）：`List<int> list = [1, 2, 3];`
- 只读返回 `IEnumerable<T>`，需要索引的只读返回 `IReadOnlyList<T>`
- 不可变查找集合使用 `FrozenDictionary` / `FrozenSet`（.NET 8+）
- 不要将 `List<T>` 作为公共属性暴露——使用 `IReadOnlyList<T>` 或 `ImmutableArray<T>`
- 热路径中的零分配切片使用 `Span<T>` 和 `Memory<T>`

### 相等性与哈希
- 同时覆写 `Equals` 和 `GetHashCode`，或使用 `record` 类型（编译器自动生成）
- 集合中的值比较实现 `IEquatable<T>`
- EF 实体不要在 `Equals` 中使用自动生成的 ID——瞬态实体无法匹配
- 泛型比较使用 `EqualityComparer<T>.Default`

### 字符串处理
- 使用字符串插值：`$"Hello {name}"`——高性能且可读
- 循环中的拼接使用 `StringBuilder`
- 零分配字符串解析使用 `ReadOnlySpan<char>`（热路径中避免 `Substring`）
- 使用 `string.Create()` 高效构建模式字符串
- 注意区域性敏感的比较：编程比较使用 `StringComparison.Ordinal`

---

## ASP.NET Core

### 依赖注入
- 优先使用构造函数注入——避免在每个 action 参数上使用 `[FromServices]`
- 使用 `IServiceCollection` 扩展方法组织注册
- Scoped 服务用于每个请求的状态（DbContext、工作单元）
- Transient 用于无状态服务；Singleton 用于线程安全的共享状态
- **Captive dependency Bug**：Singleton 持有 Scoped 或 Transient 服务——生命周期不匹配

### 中间件与过滤器
- 异常处理中间件应在管道中尽早注册
- 使用 `IAsyncExceptionFilter` 或异常处理中间件，而非在每个 Controller 中 try/catch
- 授权过滤器在 action 过滤器之前执行——不要重复认证逻辑
- 响应包装使用 `IResultFilter` 而非中间件（更精准）

### Controller 与 Minimal API
- Controller 应该精简——委托给 Service
- Minimal API（.NET 7+）：使用 `Results.Ok()`、`Results.NotFound()` 等实现类型化响应
- 使用 `[ApiController]` 自动生成 400 校验响应
- 使用 `DataAnnotations` 或 `FluentValidation` 校验——不要自己手写
- 不要直接返回实体——使用 DTO 或 record 控制响应形状
- 端点默认标记 `[Authorize]`，用 `[AllowAnonymous]` 退出

### 配置
- 使用 `IOptions<T>`、`IOptionsMonitor<T>` 或 `IOptionsSnapshot<T>` 模式
- `appsettings.json` 用于非敏感配置；环境变量或 Key Vault 用于密钥
- 绝不在 `appsettings.json` 中存储包含密码的连接字符串——本地使用 User Secrets，生产使用 Key Vault
- 使用 `IValidateOptions<T>` 在启动时校验配置

---

## Entity Framework Core

### N+1 问题
EF 排第一的性能问题。

**需要关注的模式：**
- 循环中触发懒加载（查询后访问导航属性）
- 需要立即加载的关联缺少 `Include()`
- 只读查询缺少 `AsNoTracking()`（默认跟踪增加开销）

**解决方案：**
- `Include()` / `ThenInclude()` 用于立即加载
- 只读查询使用 `AsNoTracking()`——显著提升性能
- 多个 Include 的查询使用 `AsSplitQuery()`（避免笛卡尔爆炸）
- 投影到 DTO：`Select(x => new Dto { ... })`——只加载需要的列
- `TagWith()` 用于日志中的查询标识

### 查询模式
- 仅当 EF 无法表达查询时使用原生 SQL——并说明原因
- `FromSqlRaw()` 使用参数化查询，绝不使用字符串插值
- 注意客户端评估：`.Where(x => SomeLocalFunction(x))` 会将整个表拉入内存
- 大结果集流式处理使用 `AsAsyncEnumerable()`——不要所有都 `.ToListAsync()`
- 分页：使用 `Skip().Take()` 或大数据集使用 keyset 分页

### 迁移与 Schema
- 应用前始终审查生成的迁移
- 检查频繁查询列上的缺失索引
- 注意生产中的破坏性迁移：删除列、修改类型
- 值对象使用 `HasConversion()`，除非必要不要存为 JSON blob

### 并发
- 使用 `RowVersion` / `[Timestamp]` 实现乐观并发
- 处理 `DbUpdateConcurrencyException`——重试或通知用户
- 不要跨多个请求持有 DbContext——它不是线程安全的

---

## 并发与异步

### async/await
- 不要使用 `async void`——使用 `async Task`（事件处理器是唯一合理场景）
- 每个 `await` 都应在返回 `Task` 或 `ValueTask` 的方法中
- 库代码配置 `ConfigureAwait(false)`——ASP.NET Core 中不需要（无 SynchronizationContext）
- 不要 sync-over-async：`.Result`、`.Wait()`、`.GetAwaiter().GetResult()`——会导致死锁

```csharp
// BAD：sync over async，可能导致死锁
var result = httpClient.GetStringAsync(url).Result;

// GOOD：全程异步
var result = await httpClient.GetStringAsync(url);
```

### 取消
- 异步方法接收 `CancellationToken` 并传递下去
- CPU 密集型循环中检查 `token.IsCancellationRequested`
- 长时间操作中定期使用 `token.ThrowIfCancellationRequested()`
- `HttpClient` 请求和数据库查询注册取消

### 并行
- CPU 密集型工作使用 `Parallel.ForEach`，而非 I/O 密集型
- 并发 I/O 操作使用 `Task.WhenAll`
- `Parallel.ForEachAsync`（.NET 6+）结合两种模式
- 注意并行循环中的共享可变状态——使用 `ConcurrentDictionary`、`Interlocked` 或锁

### 常见陷阱
- `async void` 事件处理器：异常会崩溃进程——用 try/catch 包裹
- 忘记 await 一个 `Task`——静默失败、未被观察的任务异常
- `ValueTask` 使用不当（被 await 两次或缓存后使用）——除非性能分析显示分配压力，否则使用 `Task`
- 异步代码使用 `SemaphoreSlim` 进行异步锁定——不要在异步代码中使用 `lock` 语句

---

## 错误处理

### 异常设计
- 使用具体的异常类型：`NotFoundException`、`ValidationException`，而非通用 `Exception`
- 创建继承自应用基础异常的自定义异常
- 在异常消息和 `Data` 字典中包含上下文
- 不要用异常进行流程控制——预期失败使用 `Result<T>` 或 `OneOf` 模式

### 全局错误处理
- 在 ASP.NET Core 管道中使用异常处理中间件
- Minimal API 使用 `IExceptionHandler`（.NET 8）处理错误
- 将异常映射为 `ProblemDetails`（RFC 7807）实现一致的 API 错误
- 生产错误响应中不暴露堆栈跟踪或内部细节

### 反模式
```csharp
// BAD：吞掉异常
catch (Exception) { }

// BAD：捕获范围太宽且丢失信息
catch (Exception ex) { throw ex; } // 破坏堆栈跟踪

// CORRECT：重新抛出保留堆栈跟踪
catch (Exception ex) { throw; }

// CORRECT：异常链
catch (SqlException ex) { throw new DataAccessException("Query failed", ex); }
```

### Result 模式
对于预期失败场景（校验、未找到），优先于异常使用：
```csharp
// 使用 OneOf、FluentResults 或自定义 Result<T>
public Result<User> FindUser(Guid id) { ... }

// 而非基于异常的流程控制
public User FindUser(Guid id) {
    throw new NotFoundException(); // 预期场景应避免
}
```

---

## 安全

### 认证与授权
- 使用 `[Authorize]` 特性——不要自己写认证检查
- `[Authorize(Roles = "Admin")]` 用于基于角色；`[Authorize(Policy = "CanEdit")]` 用于基于声明
- 基于资源的授权：验证所有权，不仅仅是认证
- 未适当授权不要在 `HttpContext.Items` 中存储敏感数据
- JWT 校验：每次请求检查签发者、受众、过期时间、签名密钥

### 输入校验
- 在边界处（Controller / Minimal API）校验所有输入
- DataAnnotations：`[Required]`、`[StringLength]`、`[Range]`、`[RegularExpression]`
- 复杂规则使用 FluentValidation
- 不要信任客户端校验——服务端必须重新校验
- 对用于文件路径、SQL、URL、Shell 参数的输入进行清理

### 常见漏洞
- **SQL 注入**：使用 EF Core 参数化查询，`FromSqlRaw()` 中绝不用原始字符串插值
- **XSS**：ASP.NET Core 自动编码 Razor 输出，但注意 `@Html.Raw()`
- **CSRF**：Cookie 认证使用 `[ValidateAntiForgeryToken]`；JWT 天然防 CSRF
- **开放重定向**：重定向 URL 按白名单校验
- **反序列化**：避免 `BinaryFormatter`（已弃用、危险）；使用 `System.Text.Json`
- **SSRF**：校验并限制服务端代码中 `HttpClient` 调用的 URL

### 密钥管理
- 开发环境使用 .NET Secret Manager（`dotnet user-secrets`）
- 生产环境使用 Azure Key Vault、AWS Secrets Manager 或 HashiCorp Vault
- 绝不将包含密码的连接字符串提交到源代码管理
- 不要记录密钥：在日志配置中做脱敏

---

## 测试

### 单元测试
- 推荐使用 xUnit：`[Fact]`、`[Theory]`、`[InlineData]`
- `Moq` 或 `NSubstitute` 用于 mock
- `FluentAssertions` 用于可读的断言：`result.Should().Be(expected)`
- 测试命名：`MethodName_Scenario_ExpectedResult`
- 不要 mock 被测系统——mock 它的依赖

### 集成测试
- `WebApplicationFactory<T>` 用于 ASP.NET Core 集成测试
- 使用 `Testcontainers` 获取真实数据库实例——不要 mock DbContext
- 测试间重置数据库状态：事务回滚或重建
- 端到端测试中间件管道、认证过滤器和错误处理

### 测试质量
- 覆盖正常路径和错误路径
- 测试边界条件：null、空、最大值、并发访问
- 不要无关联工单/issue 地使用 `[Ignore]` / `[Skip]`
- 保持测试独立——无顺序依赖

---

## 现代 C#

### C# 10 / .NET 6
- **Global usings**：`global using System.Collections.Generic;`——减少样板代码
- **文件范围命名空间**：`namespace MyApp.Services;`——减少嵌套
- **Record struct**：`record struct Point(int X, int Y);` 用于值类型 record
- **常量插值字符串**：`const string msg = $"Hello {name}";`（当 `name` 为 const 时）

### C# 11 / .NET 7
- **原始字符串字面量**：`"""..."""` 用于多行字符串、正则表达式、JSON、SQL
- **列表模式**：`if (arr is [1, 2, ..])` 用于数组匹配
- **Required 成员**：`required string Name { init; }` 强制初始化
- **`Span<T>` 模式匹配**：更高效的缓冲区处理

### C# 12 / .NET 8
- **主构造函数**：`class UserService(ILogger logger)`——更简洁的 DI
- **集合表达式**：`int[] arr = [1, 2, 3];` 统一语法
- **内联数组**：性能关键的固定大小缓冲区
- **`FrozenDictionary`/`FrozenSet`**：针对读多不变的查找优化

### C# 13 / .NET 9
- **`params` 集合**：`params ReadOnlySpan<int>`——更少分配
- **Partial 属性**：更好的源生成器支持
- **`lock` 对象**：`lock` 语句现在使用 `System.Threading.Lock`，性能更好
- **`allows ref struct`**：ref struct 类型的泛型约束

### 何时推荐现代模式
- 不可变数据载体（DTO、值对象）使用 record 而非 class
- 主构造函数优于样板化的构造函数 + 字段赋值
- 集合表达式优于 `new List<T>` 初始化
- 文件范围命名空间优于块范围
- 模式匹配优于带类型转换和 `is` 检查的 `if`/`switch`
- 一次性初始化的配置/查找数据使用 `FrozenDictionary`

---

## WPF / 桌面应用

### MVVM 模式
- ViewModel 不应引用 UI 元素——使用数据绑定
- `INotifyPropertyChanged` 用于属性变更通知——使用 `ObservableObject`（MVVM Toolkit）
- `ICommand` / `RelayCommand` 用于按钮操作——不要使用 code-behind 事件处理器
- 使用 `CommunityToolkit.Mvvm` 实现源生成的 MVVM 样板代码

### 线程
- UI 更新必须在 UI 线程——使用 `Dispatcher.Invoke()` 或 `TaskScheduler.FromCurrentSynchronizationContext()`
- 不要用同步 I/O 阻塞 UI 线程——使用 `async/await`
- `IProgress<T>` 用于从后台线程报告进度
- 注意 UI 事件和异步操作间的竞态条件

### 资源管理
- 不要每次请求创建 `HttpClient`——使用 static 或 `IHttpClientFactory`
- 正确 Dispose `BitmapImage`、`FileStream`、数据库连接
- 注意内存泄漏：事件处理器订阅未取消、静态集合无限增长
