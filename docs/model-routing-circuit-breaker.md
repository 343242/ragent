# Ragent 模型路由与熔断器实现研究报告

## 一、整体架构概览

模型路由和熔断器系统由以下核心组件构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                      RoutingLLMService                          │
│  (路由入口 - 协调所有组件)                                        │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  ModelSelector   │  │ ModelHealthStore│  │ModelRoutingExec │
│  (候选选择)       │  │  (健康状态)      │  │  (Fallback执行) │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│  ModelTarget     │  │  State Machine  │
│  (模型目标配置)   │  │  CLOSED/OPEN/   │
└─────────────────┘  │  HALF_OPEN      │
                     └─────────────────┘
```

## 二、3-State 熔断器实现 (ModelHealthStore)

### 2.1 状态枚举

```java
private enum State {
    CLOSED,     // 正常状态，允许调用
    OPEN,       // 熔断状态，拒绝调用
    HALF_OPEN   // 半开状态，允许探测请求
}
```

### 2.2 健康状态数据结构

```java
private static class ModelHealth {
    private int consecutiveFailures;    // 连续失败次数
    private long openUntil;             // 熔断结束时间戳
    private boolean halfOpenInFlight;   // 半开状态是否有探测请求在飞
    private State state;                // 当前状态
}
```

### 2.3 状态转换逻辑

```
                    失败次数 >= threshold
    CLOSED ─────────────────────────────────────► OPEN
       ▲                                            │
       │                                            │ openDuration 超时
       │                                            ▼
       │                                       HALF_OPEN
       │                                            │
       └──────────────── 成功 ◄─────────────────────┘
                              │
                              │ 失败
                              ▼
                            OPEN (重新计时)
```

### 2.4 核心方法解析

#### `allowCall(id)` - 请求准入控制

```java
public boolean allowCall(String id) {
    // 使用 compute 原子操作更新状态
    healthById.compute(id, (k, v) -> {
        if (v.state == State.OPEN) {
            if (v.openUntil > now) {
                return v;  // 仍在熔断期，拒绝
            }
            // 熔断期结束，转为半开
            v.state = State.HALF_OPEN;
            v.halfOpenInFlight = true;
            allowed.set(true);  // 允许探测请求
            return v;
        }
        if (v.state == State.HALF_OPEN) {
            if (v.halfOpenInFlight) {
                return v;  // 已有探测在飞，拒绝
            }
            v.halfOpenInFlight = true;
            allowed.set(true);  // 允许探测
            return v;
        }
        // CLOSED 状态，直接允许
        allowed.set(true);
        return v;
    });
}
```

#### `markSuccess(id)` - 成功标记

```java
public void markSuccess(String id) {
    healthById.compute(id, (k, v) -> {
        v.state = State.CLOSED;           // 恢复正常
        v.consecutiveFailures = 0;        // 重置失败计数
        v.openUntil = 0L;                 // 清除熔断时间
        v.halfOpenInFlight = false;       // 清除探测标记
        return v;
    });
}
```

#### `markFailure(id)` - 失败标记

```java
public void markFailure(String id) {
    healthById.compute(id, (k, v) -> {
        if (v.state == State.HALF_OPEN) {
            // 半开状态失败 -> 重新熔断
            v.state = State.OPEN;
            v.openUntil = now + openDurationMs;
            v.halfOpenInFlight = false;
            return v;
        }
        // CLOSED 状态，累加失败
        v.consecutiveFailures++;
        if (v.consecutiveFailures >= failureThreshold) {
            // 达到阈值，触发熔断
            v.state = State.OPEN;
            v.openUntil = now + openDurationMs;
            v.consecutiveFailures = 0;
        }
        return v;
    });
}
```

### 2.5 配置参数

```yaml
ai:
  selection:
    failure-threshold: 2      # 连续失败2次触发熔断
    open-duration-ms: 30000   # 熔断持续30秒
```

## 三、模型路由实现

### 3.1 候选模型配置 (AIModelProperties)

```java
@Data
public static class ModelCandidate {
    private String id;              // 模型唯一标识
    private String provider;        // 提供商名称
    private String model;           // 模型名称
    private String url;             // 访问URL
    private Integer priority = 100; // 优先级(数值越小越高)
    private Boolean enabled = true; // 是否启用
    private Boolean supportsThinking = false; // 是否支持深度思考
}
```

配置示例：
```yaml
ai:
  chat:
    default-model: qwen3-max
    deep-thinking-model: qwen3-max
    candidates:
      - id: qwen-plus
        provider: bailian
        model: qwen-plus-latest
        priority: 1                    # 最高优先级
      - id: qwen3-local
        provider: ollama
        model: qwen3:8b-fp16
        priority: 2
      - id: qwen3-max
        provider: bailian
        model: qwen-max-latest
        priority: 3
```

### 3.2 模型选择器 (ModelSelector)

#### 选择算法

```java
private List<ModelCandidate> filterAndSortCandidates(...) {
    return candidates.stream()
        .filter(c -> c.getEnabled())                    // 过滤禁用的
        .filter(c -> !deepThinking || c.getSupportsThinking())  // 深度过滤
        .sorted(Comparator
            .comparing(c -> !Objects.equals(id, firstChoice))  // 默认模型优先
            .thenComparing(c -> c.getPriority())                // 按优先级排序
            .thenComparing(c -> c.getId()))                     // ID作为稳定排序
        .collect(Collectors.toList());
}
```

#### 健康状态过滤

```java
private ModelTarget buildModelTarget(ModelCandidate candidate, ...) {
    String modelId = resolveId(candidate);
    
    // 过滤掉不健康的模型
    if (healthStore.isUnavailable(modelId)) {
        return null;
    }
    
    return new ModelTarget(modelId, candidate, provider);
}
```

### 3.3 路由执行器 (ModelRoutingExecutor)

#### 同步调用 Fallback 链

```java
public <C, T> T executeWithFallback(
        ModelCapability capability,
        List<ModelTarget> targets,           // 已排序的候选列表
        Function<ModelTarget, C> clientResolver,  // 解析客户端
        ModelCaller<C, T> caller) {          // 实际调用逻辑
    
    Throwable last = null;
    for (ModelTarget target : targets) {
        C client = clientResolver.apply(target);
        
        // 检查熔断器
        if (!healthStore.allowCall(target.id())) {
            continue;  // 跳过熔断的模型
        }
        
        try {
            T response = caller.call(client, target);
            healthStore.markSuccess(target.id());  // 标记成功
            return response;
        } catch (Exception e) {
            last = e;
            healthStore.markFailure(target.id());  // 标记失败
            // 继续尝试下一个模型
        }
    }
    
    throw new RemoteException("All model candidates failed");
}
```

### 3.4 流式调用与首包探测 (RoutingLLMService.streamChat)

```java
public StreamCancellationHandle streamChat(ChatRequest request, StreamCallback callback) {
    List<ModelTarget> targets = selector.selectChatCandidates(deepThinking);
    
    for (ModelTarget target : targets) {
        // 1. 检查熔断器
        if (!healthStore.allowCall(target.id())) {
            continue;
        }
        
        // 2. 创建探测桥接器
        ProbeStreamBridge bridge = new ProbeStreamBridge(callback);
        
        // 3. 发起流式请求
        StreamCancellationHandle handle = client.streamChat(request, bridge, target);
        
        // 4. 等待首包(60秒超时)
        ProbeResult result = firstPacketProbe.awaitFirstPacket(
            bridge, 60, TimeUnit.SECONDS);
        
        // 5. 判断结果
        if (result.isSuccess()) {
            healthStore.markSuccess(target.id());
            return handle;  // 成功，返回句柄
        }
        
        // 6. 失败处理
        healthStore.markFailure(target.id());
        handle.cancel();  // 取消当前请求
        // 继续尝试下一个模型
    }
    
    throw notifyAllFailed(callback, lastError);
}
```

## 四、首包探测机制 (ProbeStreamBridge)

### 4.1 设计模式：装饰器 + CompletableFuture

```java
public final class ProbeStreamBridge implements StreamCallback {
    private final StreamCallback downstream;           // 下游回调
    private final CompletableFuture<ProbeResult> probe; // 探测Future
    private final List<Runnable> buffer = new ArrayList<>(); // 缓冲区
    private volatile boolean committed;                 // 是否已提交
}
```

### 4.2 工作流程

```
┌──────────────────────────────────────────────────────────────┐
│                     ProbeStreamBridge                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   onContent() ─────► probe.complete(SUCCESS)                 │
│        │                │                                    │
│        ▼                ▼                                    │
│   bufferOrDispatch()   awaitFirstPacket()                    │
│        │                    │                                │
│        ▼                    ▼                                │
│   [缓冲/直接转发]      probe.get(timeout)                    │
│                             │                                │
│        ┌────────────────────┼────────────────────┐           │
│        ▼                    ▼                    ▼           │
│    SUCCESS              TIMEOUT               ERROR          │
│    commit()          return TIMEOUT      return ERROR        │
│    转发缓冲                                                 │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 核心方法

```java
@Override
public void onContent(String content) {
    probe.complete(ProbeResult.success());  // 通知探测成功
    bufferOrDispatch(() -> downstream.onContent(content));
}

ProbeResult awaitFirstPacket(long timeout, TimeUnit unit) {
    ProbeResult result = probe.get(timeout, unit);  // 阻塞等待
    
    if (result.isSuccess()) {
        commit();  // 成功时提交缓冲
    }
    return result;
}

private void commit() {
    synchronized (lock) {
        committed = true;
        buffer.forEach(Runnable::run);  // 执行所有缓冲的操作
    }
}
```

### 4.4 探测结果类型

```java
enum Type {
    SUCCESS,      // 收到内容
    ERROR,        // 发生错误
    TIMEOUT,      // 超时(60秒)
    NO_CONTENT    // 完成但无内容
}
```

## 五、完整调用链路

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                 RoutingLLMService.streamChat()               │
├─────────────────────────────────────────────────────────────┤
│  1. selector.selectChatCandidates(deepThinking)             │
│     │                                                        │
│     ├─► 过滤 enabled=true                                    │
│     ├─► 过滤 supportsThinking (如果深度思考)                   │
│     ├─► 按 priority 排序                                     │
│     └─► 过滤 healthStore.isUnavailable()                     │
│                                                              │
│  2. for (ModelTarget target : targets)                       │
│     │                                                        │
│     ├─► healthStore.allowCall(target.id())                   │
│     │   ├─► CLOSED: 允许                                     │
│     │   ├─► OPEN + 未超时: 拒绝                               │
│     │   ├─► OPEN + 已超时: 转HALF_OPEN, 允许探测              │
│     │   └─► HALF_OPEN + 无在飞: 允许探测                      │
│     │                                                        │
│     ├─► ProbeStreamBridge bridge = new(callback)             │
│     │                                                        │
│     ├─► client.streamChat(request, bridge, target)           │
│     │                                                        │
│     ├─► firstPacketProbe.awaitFirstPacket(60s)               │
│     │   ├─► bridge.onContent() → SUCCESS                     │
│     │   ├─► bridge.onError() → ERROR                         │
│     │   ├─► bridge.onComplete() → NO_CONTENT                 │
│     │   └─► timeout → TIMEOUT                                │
│     │                                                        │
│     ├─► if SUCCESS:                                          │
│     │   ├─► healthStore.markSuccess() → CLOSED               │
│     │   └─► return handle (流式继续)                          │
│     │                                                        │
│     └─► if FAIL:                                             │
│         ├─► healthStore.markFailure()                        │
│         │   ├─► HALF_OPEN → OPEN (重新熔断)                   │
│         │   └─► CLOSED → ++failures → OPEN (如果>=阈值)       │
│         ├─► handle.cancel()                                  │
│         └─► continue (尝试下一个模型)                          │
│                                                              │
│  3. throw RemoteException("All models failed")               │
└─────────────────────────────────────────────────────────────┘
```

## 六、关键设计亮点

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| **原子状态更新** | `ConcurrentHashMap.compute()` | 线程安全，无锁竞争 |
| **首包探测缓冲** | `CompletableFuture` + 缓冲队列 | 用户无感知的模型切换 |
| **半开状态单飞** | `halfOpenInFlight` 标记 | 避免探测请求风暴 |
| **优先级排序** | 多级 Comparator | 灵活的候选排序策略 |
| **深度思考路由** | `supportsThinking` 过滤 | 按需选择高质量模型 |

## 七、配置参考

```yaml
ai:
  # 提供商配置
  providers:
    ollama:
      url: http://localhost:11434
      endpoints:
        chat: /v1/chat/completions
    bailian:
      url: https://dashscope.aliyuncs.com
      api-key: ${BAILIAN_API_KEY}
      endpoints:
        chat: /compatible-mode/v1/chat/completions

  # 熔断配置
  selection:
    failure-threshold: 2      # 连续失败2次触发熔断
    open-duration-ms: 30000   # 熔断持续30秒

  # 聊天模型组
  chat:
    default-model: qwen3-max
    deep-thinking-model: qwen3-max
    candidates:
      - id: qwen-plus
        provider: bailian
        model: qwen-plus-latest
        priority: 1
        enabled: true
        supports-thinking: false
      - id: qwen3-max
        provider: bailian
        model: qwen-max-latest
        priority: 2
        supports-thinking: true
```

## 八、源文件索引

| 文件 | 职责 |
|------|------|
| `infra-ai/src/main/java/.../model/ModelHealthStore.java` | 三态熔断器状态机 |
| `infra-ai/src/main/java/.../model/ModelSelector.java` | 候选模型选择与排序 |
| `infra-ai/src/main/java/.../model/ModelRoutingExecutor.java` | 同步调用 Fallback 执行 |
| `infra-ai/src/main/java/.../chat/RoutingLLMService.java` | 路由入口，协调所有组件 |
| `infra-ai/src/main/java/.../chat/ProbeStreamBridge.java` | 首包探测桥接器 |
| `infra-ai/src/main/java/.../chat/LlmFirstPacketProbe.java` | 首包探测 AOP 切面 |
| `infra-ai/src/main/java/.../config/AIModelProperties.java` | 配置属性定义 |
| `infra-ai/src/main/java/.../chat/ChatClient.java` | 模型客户端接口 |
| `infra-ai/src/main/java/.../model/ModelTarget.java` | 模型目标配置记录 |
