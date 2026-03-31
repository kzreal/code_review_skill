# Java / Spring Boot 代码审查指南

## 目录
1. [Java 通用](#java-通用)
2. [Spring Boot 特有](#spring-boot-特有)
3. [JPA / 数据库](#jpa--数据库)
4. [并发](#并发)
5. [错误处理](#错误处理)
6. [安全](#安全)
7. [测试](#测试)
8. [现代 Java 模式（17/21）](#现代-java-模式)

---

## Java 通用

### Null 安全
- 方法可能找不到结果时，优先使用 `Optional` 作为返回类型，但不要用于字段或方法参数
- 使用 `Objects.requireNonNull()` 校验构造函数/方法参数
- 检查并遵守 `@Nullable` / `@NonNull` 注解
- 注意自动拆箱 NPE：`Integer` → `int` 时值为 null
- 在 Optional 上调用 `.get()` 但未先检查 `.isPresent()` 是 Bug

### 资源管理
- `InputStream`、`OutputStream`、`Connection`、`Statement`、`ResultSet` 始终使用 try-with-resources
- 注意循环中的资源泄漏——每次迭代创建资源但异常时未关闭
- `Stream` 和 `CompletableFuture` 如果未正确链式处理可能导致资源泄漏

### 集合
- 返回空集合而非 null：`Collections.emptyList()`、`List.of()`
- 常量使用不可变集合：`List.of()`、`Map.of()`、`Set.of()`
- 注意 `ConcurrentModificationException`——迭代时不要修改集合（使用 `removeIf` 或迭代器）
- `ArrayList.subList()` 返回视图而非副本——修改会影响父列表

### equals / hashCode / toString
- 始终同时覆写 `equals` 和 `hashCode`，或都不覆写
- 使用 `Objects.equals()` 和 `Objects.hash()` 避免 null 问题
- JPA 实体基于业务键定义 `equals`/`hashCode`，而非自动生成的 ID
- 避免在 `hashCode` 中使用可变字段（会导致 HashSet/HashMap 问题）

### 字符串处理
- 循环中的字符串拼接使用 `StringBuilder`
- 多行字符串使用 `String.format()` 或文本块（Java 15+）
- 注意 String 引用的 `==`——始终使用 `.equals()`
- `String.intern()` 可能导致 permgen/metaspace 问题——除非必要否则避免使用

---

## Spring Boot 特有

### 依赖注入
- 优先使用构造函数注入而非 `@Autowired` 字段注入
- 必需依赖使用 `@RequiredArgsConstructor`（Lombok）或手动构造函数
- Spring Boot 3 中构造函数上的 `@Autowired` 可省略（单构造函数自动装配）
- 对字段上的 `@Autowired` 保持警惕——它隐藏了依赖复杂度，使测试更困难
- 注意循环依赖：A → B → A。最后手段才用 `@Lazy`，首选重新设计

### 配置
- 分组配置使用 `@ConfigurationProperties` 而非 `@Value`
- 使用 `@Validated`、`@NotNull`、`@Min`、`@Max` 校验配置
- 绝不硬编码密钥——使用环境变量或 Vault 集成
- Profile 特定的配置不应包含生产密钥

### Web 层
- Controller 应该精简——委托给 Service
- API 端点使用 `@RestController` + `ResponseEntity`
- 使用 `@Valid` / `@Validated` + Bean Validation 注解校验输入
- 不要在 Controller 中吞掉异常——让 `@ControllerAdvice` 处理
- 使用 `ProblemDetail`（Spring 6）实现标准化错误响应
- 注意 `@RequestMapping` 未限制 HTTP 方法——使用 `@GetMapping`、`@PostMapping` 等

### Service 层
- 读操作标记为 `@Transactional(readOnly = true)`——这是优化提示
- 不要在同一类内部调用 `@Transactional` 方法（代理绕过）
- 注意调用外部 API 的 `@Transactional` 方法——长事务
- 复杂场景优先使用编程式事务边界

### 异步与调度
- `@Async` 方法必须返回 `void` 或 `CompletableFuture`
- 不要在同一类内部调用 `@Async`（与 `@Transactional` 相同的代理问题）
- `@Scheduled` 任务应是幂等的——如果上次执行较慢可能重叠
- 配置线程池——生产环境不要依赖默认值

---

## JPA / 数据库

### N+1 问题
最常见的 JPA 性能问题。需要关注：
- `@OneToMany` 设置 `FetchType.EAGER` 在循环中被访问
- 事务关闭后在循环中懒加载实体（触发 `LazyInitializationException`）
- JPQL 查询中需要关联实体时缺少 `JOIN FETCH`

解决方案：
- `@EntityGraph` 实现可预测的加载
- 不可避免的懒加载使用 `@BatchSize`
- 只读查询使用 DTO 投影（JPQL/Criteria API）

### 实体设计
- 不要在 API 响应中直接暴露 JPA 实体——使用 DTO
- 实体上使用 `@Data`（Lombok）会导致 `equals`/`hashCode`/`toString` 与懒加载冲突
- 使用 `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` 并包含业务键字段
- `toString()` 不应触碰懒加载的关联关系

### 查询模式
- 简单场景优先使用 Spring Data JPA 派生查询
- 复杂查询使用 `@Query`——避免过于复杂的方法名如 `findByUserAndStatusAndCreatedAtBetween`
- 注意大结果集缺少分页
- 原生查询绕过 JPA 缓存和实体映射——需要文档说明为什么使用

### 事务
- 读操作：`@Transactional(readOnly = true)`
- 写操作：默认 `@Transactional` 即可，但如果隔离级别重要需明确指定
- 不要在事务内执行外部 API 调用——它们会持有数据库连接
- 注意同一 Bean 内事务方法的自调用

---

## 并发

### 线程安全
- `SimpleDateFormat` 非线程安全——使用 `DateTimeFormatter`（Java 8+）或 `ThreadLocal`
- `HashMap` 非线程安全——共享状态使用 `ConcurrentHashMap`
- `ArrayList` 非线程安全——使用 `CopyOnWriteArrayList` 或 `Collections.synchronizedList`
- 注意 check-then-act 竞态条件：`if (!map.containsKey(k)) { map.put(k, v); }`
- Spring Bean 上的 `synchronized` 影响所有请求——考虑 `ConcurrentHashMap.computeIfAbsent()`

### 虚拟线程（Java 21）
- 适合 I/O 密集型任务（HTTP 调用、数据库查询、文件操作）
- 不要固定虚拟线程：避免在 I/O 操作中使用 `synchronized`——改用 `ReentrantLock`
- 不要池化虚拟线程——它们的创建成本很低
- 注意虚拟线程中 `ThreadLocal` 的膨胀——仍然消耗内存

### CompletableFuture
- 始终处理异常：`.exceptionally()` 或 `.handle()`
- 注意异步链中被吞掉的异常
- 不要无超时地阻塞在 `.get()` 上——使用 `.get(timeout, unit)` 或带错误处理的 `.join()`

---

## 错误处理

### 异常设计
- 业务逻辑错误使用非受检异常，可恢复条件使用受检异常
- 创建异常层次：`BaseException` → `NotFoundException`、`ValidationException` 等
- 异常消息中包含上下文：实体 ID、操作、输入值
- 不要宽泛地捕获 `Exception`——捕获特定类型或重新抛出

### 全局错误处理
- 使用 `@ControllerAdvice` + `@ExceptionHandler` 实现一致的错误响应
- 在适当的级别记录带上下文（请求 ID、用户、操作）的异常
- 不要在 API 错误响应中暴露堆栈跟踪或内部细节
- 一致地将异常映射到 HTTP 状态码

### 反模式
- 吞掉异常：`catch (Exception e) { /* 什么也不做 */ }` 或仅 `log.error()` 不重新抛出
- 将所有异常包装为通用 `RuntimeException`——丢失类型信息
- 使用异常进行流程控制（如：用抛异常来跳出循环）
- 记录日志后重新抛出：导致重复日志条目，在生产中造成混淆

---

## 安全

### 输入校验
- 在边界处（Controller 层）校验所有外部输入
- 使用 Bean Validation：`@NotNull`、`@Size`、`@Pattern`、`@Email`、`@Min`、`@Max`
- 不要信任客户端校验——那是用户体验，不是安全
- 对用于文件路径、SQL 查询、Shell 命令的输入进行清理

### 认证与授权
- Service 方法使用 `@PreAuthorize` 和 `@Secured` 实现纵深防御
- 不要没有充分理由就绕过"内部" API 的安全检查
- 注意 IDOR：通过 ID 访问资源但未校验所有权
- JWT token：每次请求都验证过期时间、签名、签发者、受众

### 常见漏洞
- SQL 注入：使用参数化查询 / JPA，绝不字符串拼接
- XSS：转义输出，API 使用 `Content-Type: application/json`
- CSRF：确保状态变更操作有 CSRF 保护（或使用无状态认证）
- 反序列化：避免对不可信数据使用 `ObjectInputStream`
- SSRF：校验并限制服务端 HTTP 请求中的 URL

---

## 测试

### 单元测试
- 测试行为而非实现——不要断言私有方法调用
- 使用 `@ExtendWith(MockitoExtension.class)` 配合 `@Mock` 和 `@InjectMocks`
- 不要 mock 被测系统——mock 它的依赖
- 注意总是通过的测试：没有断言、捕获所有异常

### 集成测试
- 谨慎使用 `@SpringBootTest`——它启动完整上下文
- Controller 测试优先使用 `@WebMvcTest`，Repository 测试使用 `@DataJpaTest`
- 数据库集成测试使用 `Testcontainers`——不要 mock Repository
- 清理测试数据：`@Transactional` + 回滚，或显式清理

### 测试质量
- 每个测试应遵循：准备 → 执行 → 断言
- 测试名称应描述场景：`shouldReturn404WhenUserNotFound()`
- 覆盖边界情况：null、空、边界值、并发访问
- 不要无理由地用 `@Disabled` 跳过测试，必须有关联的工单引用

---

## 现代 Java 模式

### Java 17+
- **Sealed classes**：建模封闭层次——`sealed interface Shape permits Circle, Rectangle`
- **switch 模式匹配**：`switch (obj) { case String s -> ...; case Integer i -> ...; }`
- **文本块**：`"""..."""` 用于多行字符串（SQL、JSON 等）
- **Records**：`record Point(int x, int y) {}` 用于不可变数据载体——DTO 使用它

### Java 21
- **虚拟线程**：参见上方并发章节
- **Record 模式**：`if (obj instanceof Point(int x, int y)) { ... }`
- **有序集合**：`SequencedCollection`、`getFirst()`、`getLast()`

### 何时推荐现代模式
- DTO 优先使用 Records 而非 Lombok `@Data`
- 模式匹配优于 `instanceof` + 类型转换
- 多行字符串使用文本块而非拼接
- switch 表达式优于 switch 语句
