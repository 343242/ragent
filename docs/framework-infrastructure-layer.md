# Ragent 基础设施层 (Framework) 研究报告

## 一、整体架构概览

Framework 模块是 Ragent 的基础设施层，提供与业务无关的通用能力，覆盖 10 个横切关注点：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Framework 模块                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  exception   │  │  errorcode  │  │   web       │  │  convention │        │
│  │  异常体系     │  │  错误码      │  │  Web组件    │  │  统一规约    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  idempotent │  │   trace     │  │distributedid│  │  context    │        │
│  │  幂等控制    │  │  链路追踪    │  │  分布式ID   │  │  上下文传递  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
│  │  database   │  │    cache    │  │     mq      │                         │
│  │  数据库      │  │  缓存       │  │  消息队列    │                         │
│  └─────────────┘  └─────────────┘  └─────────────┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、三级异常体系

### 2.1 异常层级结构

```
RuntimeException
    │
    └── AbstractException (抽象基类)
            │
            ├── ClientException     (A类: 客户端错误)
            │
            ├── ServiceException    (B类: 服务端错误)
            │
            └── RemoteException     (C类: 远程服务错误)
```

### 2.2 抽象异常基类

```java
@Getter
public abstract class AbstractException extends RuntimeException {
    public final String errorCode;
    public final String errorMessage;

    public AbstractException(String message, Throwable throwable, IErrorCode errorCode) {
        super(message, throwable);
        this.errorCode = errorCode.code();
        this.errorMessage = Optional.ofNullable(
            StringUtils.hasLength(message) ? message : null
        ).orElse(errorCode.message());
    }
}
```

**设计亮点**：
- 统一携带 `errorCode` 和 `errorMessage`
- 支持自定义消息覆盖默认错误消息
- 支持异常链 (cause)

### 2.3 三类异常使用场景

| 异常类型 | 错误码前缀 | 使用场景 | 示例 |
|---------|-----------|---------|------|
| `ClientException` | A | 用户提交参数错误、权限不足、重复提交 | 参数校验失败、未登录 |
| `ServiceException` | B | 业务逻辑不符合预期、系统内部错误 | 数据不存在、状态异常 |
| `RemoteException` | C | 调用第三方服务失败 | 模型调用超时、向量数据库连接失败 |

---

## 三、统一错误码规范

### 3.1 错误码接口

```java
public interface IErrorCode {
    String code();     // 错误码
    String message();  // 错误信息
}
```

### 3.2 错误码编码规则 (阿里巴巴规范)

```
┌─────────────────────────────────────────────────────────────┐
│                    错误码结构: X000YYY                        │
├─────────────────────────────────────────────────────────────┤
│  X: 宏观分类                                                │
│     A = 客户端错误 (Client Error)                            │
│     B = 系统执行错误 (Service Error)                         │
│     C = 第三方服务错误 (Remote Error)                        │
│                                                             │
│  000: 一级宏观错误码                                          │
│  YYY: 二级/三级错误码                                         │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 已定义错误码

```java
public enum BaseErrorCode implements IErrorCode {
    // ========== A 类错误：用户端错误 ==========
    CLIENT_ERROR("A000001", "用户端错误"),
    USER_REGISTER_ERROR("A000100", "用户注册错误"),
    USER_NAME_EXIST_ERROR("A000111", "用户名已存在"),
    PASSWORD_VERIFY_ERROR("A000120", "密码校验失败"),
    IDEMPOTENT_TOKEN_NULL_ERROR("A000200", "幂等Token为空"),
    IDEMPOTENT_TOKEN_DELETE_ERROR("A000201", "幂等Token已被使用或失效"),
    SEARCH_AMOUNT_EXCEEDS_LIMIT("A000300", "查询数据量超过最大限制"),

    // ========== B 类错误：系统执行错误 ==========
    SERVICE_ERROR("B000001", "系统执行出错"),
    SERVICE_TIMEOUT_ERROR("B000100", "系统执行超时"),

    // ========== C 类错误：第三方服务错误 ==========
    REMOTE_ERROR("C000001", "调用第三方服务出错");
}
```

---

## 四、统一响应体

### 4.1 Result 结构

```java
@Data
@Accessors(chain = true)
public class Result<T> implements Serializable {
    public static final String SUCCESS_CODE = "0";

    private String code;       // 状态码: "0"=成功, 其他=错误
    private String message;    // 响应消息
    private T data;            // 响应数据
    private String requestId;  // 请求追踪ID

    public boolean isSuccess() {
        return SUCCESS_CODE.equals(code);
    }
}
```

### 4.2 Results 构建器

```java
public final class Results {
    // 成功响应
    public static Result<Void> success() { ... }
    public static <T> Result<T> success(T data) { ... }

    // 失败响应
    public static Result<Void> failure() { ... }
    static Result<Void> failure(AbstractException ex) { ... }
    static Result<Void> failure(String errorCode, String errorMessage) { ... }
}
```

### 4.3 响应示例

```json
// 成功响应
{
    "code": "0",
    "message": null,
    "data": { ... },
    "requestId": "trace-abc123"
}

// 失败响应
{
    "code": "A000001",
    "message": "用户名已存在",
    "data": null,
    "requestId": "trace-abc123"
}
```

---

## 五、全局异常处理器

### 5.1 拦截策略

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 1. 参数验证异常 (MethodArgumentNotValidException)
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public Result<Void> validExceptionHandler(...) { ... }

    // 2. 应用内抛出的异常 (AbstractException 及其子类)
    @ExceptionHandler(value = {AbstractException.class})
    public Result<Void> abstractException(...) { ... }

    // 3. 未登录异常 (Sa-Token)
    @ExceptionHandler(value = NotLoginException.class)
    public Result<Void> notLoginException(...) { ... }

    // 4. 无角色权限异常 (Sa-Token)
    @ExceptionHandler(value = NotRoleException.class)
    public Result<Void> notRoleException(...) { ... }

    // 5. 文件上传大小超限异常
    @ExceptionHandler(value = MaxUploadSizeExceededException.class)
    public Result<Void> maxUploadSizeExceededException(...) { ... }

    // 6. 兜底: 未捕获异常
    @ExceptionHandler(value = Throwable.class)
    public Result<Void> defaultErrorHandler(...) { ... }
}
```

### 5.2 日志记录策略

| 异常类型 | 日志级别 | 记录内容 |
|---------|---------|---------|
| 参数验证异常 | ERROR | 请求方法、URL、验证错误信息 |
| AbstractException (有cause) | ERROR | 请求方法、URL、异常栈 |
| AbstractException (无cause) | ERROR | 请求方法、URL、前5行堆栈 |
| 未登录/无权限 | WARN | 请求方法、URL、异常消息 |
| 文件超限 | WARN | 请求方法、URL、限制说明 |
| 未知异常 | ERROR | 完整异常栈 |

---

## 六、SSE 线程安全封装

### 6.1 SseEmitterSender 设计

```java
public class SseEmitterSender {
    private final SseEmitter emitter;
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public SseEmitterSender(SseEmitter emitter) {
        this.emitter = emitter;
        // 注册生命周期回调
        emitter.onCompletion(() -> closed.set(true));
        emitter.onTimeout(() -> closed.set(true));
        emitter.onError(e -> closed.set(true));
    }
}
```

### 6.2 核心方法

```java
// 发送事件 (线程安全)
public void sendEvent(String eventName, Object data) {
    if (closed.get()) return;  // 快速失败
    try {
        if (eventName == null) {
            emitter.send(data);
        } else {
            emitter.send(SseEmitter.event().name(eventName).data(data));
        }
    } catch (Exception e) {
        fail(e);
    }
}

// 正常完成 (CAS 保证幂等)
public void complete() {
    if (closed.compareAndSet(false, true)) {
        emitter.complete();
    }
}

// 异常结束 (CAS 保证幂等)
public void fail(Throwable throwable) {
    if (closed.compareAndSet(false, true)) {
        emitter.completeWithError(throwable);
    }
}
```

### 6.3 设计亮点

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| 线程安全 | `AtomicBoolean` CAS | 多线程并发发送不冲突 |
| 幂等关闭 | `compareAndSet` | 避免重复关闭导致异常 |
| 快速失败 | `closed.get()` 检查 | 连接已关闭时直接返回 |
| 生命周期感知 | `onCompletion/onTimeout/onError` | 自动同步状态 |

---

## 七、双维度幂等控制

### 7.1 提交幂等 (@IdempotentSubmit)

**场景**：防止用户重复提交表单

```java
@IdempotentSubmit(message = "您操作太快，请稍后再试")
@PostMapping("/api/submit")
public Result<Void> submit(@RequestBody SubmitRequest request) { ... }
```

**实现机制**：

```java
@Around("@annotation(IdempotentSubmit)")
public Object idempotentSubmit(ProceedingJoinPoint joinPoint) throws Throwable {
    // 1. 构建锁 Key
    String lockKey = buildLockKey(joinPoint, idempotentSubmit);
    // 格式: idempotent-submit:path:{servletPath}:currentUserId:{userId}:md5:{argsMd5}
    // 或自定义 SpEL: idempotent-submit:key:{spelValue}

    // 2. 尝试获取分布式锁
    RLock lock = redissonClient.getLock(lockKey);
    if (!lock.tryLock()) {
        throw new ClientException(idempotentSubmit.message());
    }

    // 3. 执行业务逻辑
    try {
        return joinPoint.proceed();
    } finally {
        lock.unlock();
    }
}
```

**锁 Key 生成策略**：
- **默认**: `{path}:{userId}:{argsMD5}` (同一用户、同一接口、相同参数)
- **自定义**: 通过 SpEL 表达式生成

### 7.2 消费幂等 (@IdempotentConsume)

**场景**：防止消息队列消费者重复消费

```java
@IdempotentConsume(keyPrefix = "mq:doc:parse:", key = "#msg.taskId")
public void handleDocumentParseMessage(DocumentParseMessage msg) { ... }
```

**实现机制**：

```java
@Around("@annotation(IdempotentConsume)")
public Object idempotentConsume(ProceedingJoinPoint joinPoint) throws Throwable {
    String uniqueKey = keyPrefix + SpELUtil.parseKey(key, method, args);

    // 1. Lua 脚本原子操作: SET key value NX PX expire
    String absentAndGet = stringRedisTemplate.execute(LUA_SCRIPT,
        List.of(uniqueKey),
        CONSUMING.getCode(),
        String.valueOf(keyTimeoutMs)
    );

    // 2. 状态判断
    if (isError(absentAndGet)) {
        throw new ServiceException("消息消费者幂等异常");  // 消费中，延迟重试
    }
    if (CONSUMED.getCode().equals(absentAndGet)) {
        return null;  // 已消费，直接跳过
    }

    // 3. 执行业务逻辑
    try {
        Object result = joinPoint.proceed();
        // 标记消费完成
        stringRedisTemplate.opsForValue().set(uniqueKey, CONSUMED.getCode(), keyTimeout, SECONDS);
        return result;
    } catch (Throwable ex) {
        stringRedisTemplate.delete(uniqueKey);  // 失败时删除，允许重试
        throw ex;
    }
}
```

**状态机**：

```
┌─────────────────────────────────────────────────────────────┐
│                    消费幂等状态机                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   不存在 ──► CONSUMING ──► CONSUMED                         │
│      │           │                                          │
│      │           └──► 删除 (失败时)                          │
│      │                                                      │
│   CONSUMING: 消费中，重复消息延迟重试                          │
│   CONSUMED:  已消费，重复消息直接跳过                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 八、分布式 ID 生成 (Snowflake)

### 8.1 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                    Snowflake ID 生成                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Redis (Lua脚本)                                           │
│       │                                                     │
│       ▼                                                     │
│   workerId + datacenterId                                   │
│       │                                                     │
│       ▼                                                     │
│   Hutool Snowflake                                          │
│       │                                                     │
│       ▼                                                     │
│   MyBatis-Plus CustomIdentifierGenerator                    │
│       │                                                     │
│       ▼                                                     │
│   实体ID自动生成                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 初始化流程

```java
@Component
public class SnowflakeIdInitializer {

    @PostConstruct
    public void init() {
        // 1. 加载 Lua 脚本
        DefaultRedisScript<List> script = new DefaultRedisScript<>();
        script.setScriptSource(new ResourceScriptSource(
            new ClassPathResource("lua/snowflake_init.lua")));

        // 2. 从 Redis 获取 workerId 和 datacenterId
        List<Long> result = stringRedisTemplate.execute(script, Collections.emptyList());

        // 3. 注册到 Hutool 单例
        Snowflake snowflake = new Snowflake(workerId, datacenterId);
        Singleton.put(snowflake);
    }
}
```

### 8.3 MyBatis-Plus 集成

```java
@Component
public class CustomIdentifierGenerator implements IdentifierGenerator {

    @Override
    public Number nextId(Object entity) {
        return IdUtil.getSnowflakeNextId();  // 使用 Hutool Snowflake
    }

    @Override
    public String nextUUID(Object entity) {
        return IdUtil.getSnowflakeNextIdStr();
    }
}
```

**优势**：
- 分布式环境自动生成唯一 ID
- 无需手动设置实体 ID
- 与 MyBatis-Plus 无缝集成

---

## 九、上下文透传

### 9.1 用户上下文 (UserContext)

```java
public final class UserContext {
    // 基于 TTL 实现跨线程传递
    private static final TransmittableThreadLocal<LoginUser> CONTEXT =
        new TransmittableThreadLocal<>();

    public static void set(LoginUser user) { CONTEXT.set(user); }
    public static LoginUser get() { return CONTEXT.get(); }
    public static LoginUser requireUser() {
        LoginUser user = CONTEXT.get();
        if (user == null) throw new ClientException("未获取到当前登录用户");
        return user;
    }
    public static String getUserId() { ... }
    public static String getUsername() { ... }
    public static String getRole() { ... }
    public static void clear() { CONTEXT.remove(); }
}
```

### 9.2 Trace 上下文 (RagTraceContext)

```java
public final class RagTraceContext {
    private static final TransmittableThreadLocal<String> TRACE_ID = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<String> TASK_ID = new TransmittableThreadLocal<>();
    // 节点栈 - 深拷贝避免父子线程串扰
    private static final TransmittableThreadLocal<Deque<String>> NODE_STACK =
        new TransmittableThreadLocal<>() {
            @Override
            public Deque<String> copy(Deque<String> parentValue) {
                return parentValue == null ? null : new ArrayDeque<>(parentValue);
            }
        };

    public static void pushNode(String nodeId) { ... }
    public static void popNode() { ... }
    public static String currentNodeId() { ... }
}
```

### 9.3 LoginUser 数据结构

```java
@Data
@Builder
public class LoginUser {
    private String userId;
    private String username;
    private String role;     // admin / user
    private String avatar;
}
```

### 9.4 TTL 传递机制

```
┌─────────────────────────────────────────────────────────────┐
│              TransmittableThreadLocal 传递                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   主线程                                                     │
│   ├── UserContext.set(loginUser)                            │
│   ├── RagTraceContext.setTraceId(traceId)                   │
│   │                                                        │
│   ├── submit(task) ──────────► 线程池                       │
│   │                              │                         │
│   │                              ▼                         │
│   │                           worker线程                    │
│   │                           ├── UserContext.get() ✓       │
│   │                           └── RagTraceContext.getTraceId() ✓  │
│   │                                                        │
│   └── UserContext.clear()                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 十、全链路追踪 (AOP)

### 10.1 注解定义

```java
// 根节点 (一次完整请求)
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RagTraceRoot {
    String name() default "";
    String conversationIdArg() default "conversationId";
    String taskIdArg() default "taskId";
}

// 普通节点
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RagTraceNode {
    String name() default "";
    String type() default "METHOD";
}
```

### 10.2 使用示例

```java
// 根节点
@RagTraceRoot(name = "rag-chat")
public StreamCancellationHandle streamChat(ChatRequest request, ...) { ... }

// 子节点
@RagTraceNode(name = "llm-chat-routing", type = "LLM_ROUTING")
public String chat(ChatRequest request) { ... }

@RagTraceNode(name = "llm-first-packet", type = "LLM_TTFT")
public ProbeResult awaitFirstPacket(...) { ... }
```

### 10.3 跨线程 Stream 支持

```java
public interface RagStreamTraceSupport {

    // 在调用线程开启 stream 节点
    StreamSpan beginStreamNode(String name, String type);

    interface StreamSpan {
        void detach();                    // 调用线程同步部分结束
        void finishSuccess();             // 异步线程: 完成
        void finishError(Throwable error); // 异步线程: 失败
        void finishCancelledIfRunning();  // 取消路径
    }

    // 空实现 (不开启 trace 时)
    StreamSpan NOOP_SPAN = new StreamSpan() { ... };
}
```

**解决的问题**：
- `@RagTraceNode` AOP 在 stream 场景下只测到任务提交 (`runAsync`)
- 同步部分在调用线程，SSE 读循环在线程池 worker 线程
- 通过 `StreamSpan` 实现跨线程节点跟踪

---

## 十一、数据库自动填充

### 11.1 MyMetaObjectHandler

```java
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        strictInsertFill(metaObject, "createTime", Date::new, Date.class);
        strictInsertFill(metaObject, "updateTime", Date::new, Date.class);
        strictInsertFill(metaObject, "deleted", () -> 0, Integer.class);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
}
```

**自动填充字段**：
- `createTime`: 插入时自动填充
- `updateTime`: 插入和更新时自动填充
- `deleted`: 插入时默认为 0 (逻辑删除标记)

---

## 十二、缓存 Key 序列化

### 12.1 RedisKeySerializer

```java
@Component
@ConditionalOnProperty(name = "framework.cache.redis.prefix")
public class RedisKeySerializer implements RedisSerializer<String> {

    @Value("${framework.cache.redis.prefix:}")
    private String keyPrefix;

    @Override
    public byte[] serialize(String key) throws SerializationException {
        return (keyPrefix + key).getBytes();
    }
}
```

**用途**：为 Redis Key 添加统一前缀，便于管理和避免冲突

---

## 十三、消息队列适配

### 13.1 生产者接口

```java
public interface MessageQueueProducer {

    // 普通消息
    SendResult send(String topic, String keys, String bizDesc, Object body);

    // 事务消息
    void sendInTransaction(String topic, String keys, String bizDesc, Object body,
                           Consumer<Object> localTransaction);
}
```

### 13.2 事务消息流程

```
┌─────────────────────────────────────────────────────────────┐
│                    事务消息流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. 发送 half 消息                                          │
│       │                                                     │
│       ▼                                                     │
│   2. 执行本地事务 (localTransaction)                         │
│       │                                                     │
│       ├── 成功 ──► commit                                   │
│       │                                                     │
│       └── 失败 ──► rollback                                 │
│                                                             │
│   3. 事务回查 (TransactionChecker)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 十四、Web 自动配置

```java
@Configuration
public class WebAutoConfiguration {

    @Bean
    public GlobalExceptionHandler globalExceptionHandler() {
        return new GlobalExceptionHandler();
    }
}
```

---

## 十五、统一规约 (Convention)

### 15.1 ChatRequest

```java
@Data
@Builder
public class ChatRequest {
    private List<ChatMessage> messages = new ArrayList<>();  // 消息列表
    private Double temperature;     // 采样温度 (0~2)
    private Double topP;            // nucleus sampling
    private Integer topK;           // top-k 采样
    private Integer maxTokens;      // 最大生成 token 数
    private Boolean thinking;       // 是否启用思考模式
    private Boolean enableTools;    // 是否启用工具调用
}
```

### 15.2 ChatMessage

```java
@Data
@Builder
public class ChatMessage {
    private String role;       // system / user / assistant
    private String content;    // 消息内容
}
```

### 15.3 RetrievedChunk

```java
@Data
@Builder
public class RetrievedChunk {
    private String chunkId;
    private String content;
    private Double score;
    private Map<String, Object> metadata;
}
```

---

## 十六、源文件索引

### 异常体系
| 文件 | 职责 |
|------|------|
| `exception/AbstractException.java` | 抽象异常基类 |
| `exception/ClientException.java` | 客户端异常 (A类) |
| `exception/ServiceException.java` | 服务端异常 (B类) |
| `exception/RemoteException.java` | 远程服务异常 (C类) |
| `exception/kb/VectorCollectionAlreadyExistsException.java` | 向量集合已存在异常 |

### 错误码
| 文件 | 职责 |
|------|------|
| `errorcode/IErrorCode.java` | 错误码接口 |
| `errorcode/BaseErrorCode.java` | 基础错误码枚举 |

### Web 组件
| 文件 | 职责 |
|------|------|
| `web/GlobalExceptionHandler.java` | 全局异常处理器 |
| `web/Results.java` | 响应构建器 |
| `web/SseEmitterSender.java` | SSE 线程安全封装 |

### 统一规约
| 文件 | 职责 |
|------|------|
| `convention/Result.java` | 统一响应体 |
| `convention/ChatRequest.java` | 聊天请求对象 |
| `convention/ChatMessage.java` | 聊天消息对象 |
| `convention/RetrievedChunk.java` | 检索结果块 |

### 幂等控制
| 文件 | 职责 |
|------|------|
| `idempotent/IdempotentSubmit.java` | 提交幂等注解 |
| `idempotent/IdempotentSubmitAspect.java` | 提交幂等切面 |
| `idempotent/IdempotentConsume.java` | 消费幂等注解 |
| `idempotent/IdempotentConsumeAspect.java` | 消费幂等切面 |
| `idempotent/IdempotentConsumeStatusEnum.java` | 消费状态枚举 |
| `idempotent/SpELUtil.java` | SpEL 表达式工具 |

### 链路追踪
| 文件 | 职责 |
|------|------|
| `trace/RagTraceRoot.java` | 根节点注解 |
| `trace/RagTraceNode.java` | 普通节点注解 |
| `trace/RagTraceContext.java` | Trace 上下文 (TTL) |
| `trace/RagStreamTraceSupport.java` | 跨线程 Stream 支持 |

### 分布式 ID
| 文件 | 职责 |
|------|------|
| `distributedid/SnowflakeIdInitializer.java` | Snowflake 初始化 |
| `distributedid/CustomIdentifierGenerator.java` | MyBatis-Plus ID 生成器 |

### 上下文
| 文件 | 职责 |
|------|------|
| `context/UserContext.java` | 用户上下文 (TTL) |
| `context/LoginUser.java` | 登录用户数据 |
| `context/ApplicationContextHolder.java` | Spring 上下文持有者 |

### 数据库/缓存/MQ
| 文件 | 职责 |
|------|------|
| `database/MyMetaObjectHandler.java` | 自动填充处理器 |
| `cache/RedisKeySerializer.java` | Redis Key 序列化 |
| `mq/producer/MessageQueueProducer.java` | MQ 生产者接口 |
| `mq/producer/RocketMQProducerAdapter.java` | RocketMQ 适配器 |
| `mq/producer/DelegatingTransactionListener.java` | 事务监听器 |
| `mq/producer/TransactionChecker.java` | 事务回查接口 |
| `mq/MessageWrapper.java` | 消息包装器 |

### 配置
| 文件 | 职责 |
|------|------|
| `config/WebAutoConfiguration.java` | Web 自动配置 |
| `config/DataBaseConfiguration.java` | 数据库配置 |
| `config/RocketMQAutoConfiguration.java` | RocketMQ 配置 |
