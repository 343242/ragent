# Ragent AI 基础设施层 (infra-ai) 研究报告

## 一、整体架构概览

infra-ai 模块是 Ragent 的 AI 基础设施层，负责屏蔽不同模型供应商的差异，提供统一的 AI 能力访问接口。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            infra-ai 模块                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Service 层 (路由)                            │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │    │
│  │  │ RoutingLLMService│  │RoutingEmbedding │  │RoutingRerankSvc │     │    │
│  │  │  (聊天路由)       │  │  (向量路由)      │  │  (重排路由)      │     │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         Model 层 (路由核心)                          │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │    │
│  │  │  ModelSelector   │  │ModelHealthStore │  │ModelRoutingExec │     │    │
│  │  │  (候选选择)       │  │  (健康状态)      │  │  (Fallback执行) │     │    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       Client 层 (供应商实现)                         │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │              AbstractOpenAIStyle (模板方法基类)                │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  │        │              │              │              │               │    │
│  │        ▼              ▼              ▼              ▼               │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│  │  │ BaiLian  │  │ Ollama   │  │ Silicon  │  │AIHubMix  │           │    │
│  │  │ ChatClient│  │ChatClient│  │FlowClient│  │ChatClient│           │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       支撑层 (HTTP/工具/枚举)                        │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│  │  │OkHttpClient│ │SSE Parser│  │URL Resolver│ │TokenCounter│          │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、核心职责

| 职责 | 说明 |
|------|------|
| **统一访问接口** | 屏蔽不同供应商差异，提供一致的 Chat/Embedding/Rerank API |
| **多供应商支持** | 百炼、Ollama、SiliconFlow、AIHubMix 等 |
| **智能路由** | 按优先级选择模型，支持深度思考模式 |
| **健康检查** | 三态熔断器保护，自动故障转移 |
| **流式支持** | SSE 解析、首包探测、异步执行 |
| **Token 统计** | 轻量级 Token 估算 |

---

## 三、设计模式详解

### 3.1 策略模式 (Strategy Pattern)

**应用场景**：ChatClient、EmbeddingClient、RerankClient 接口

```
┌─────────────────────────────────────────────────────────────────┐
│                      策略模式应用                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                 实现类                           │
│   ┌─────────────┐                                              │
│   │ ChatClient   │◄─────────┬── BaiLianChatClient               │
│   │ + provider() │          ├── OllamaChatClient                │
│   │ + chat()     │          ├── SiliconFlowChatClient           │
│   │ + streamChat()│         └── AIHubMixChatClient              │
│   └─────────────┘                                              │
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────┐                                              │
│   │EmbeddingClient│◄────────┬── OllamaEmbeddingClient           │
│   │ + provider() │          ├── AIHubMixEmbeddingClient         │
│   │ + embed()    │          └── SiliconFlowEmbeddingClient      │
│   │ + embedBatch()│                                             │
│   └─────────────┘                                              │
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────┐                                              │
│   │ RerankClient │◄────────┬── BaiLianRerankClient              │
│   │ + provider() │          └── NoopRerankClient                │
│   │ + rerank()   │                                              │
│   └─────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 接口定义
public interface ChatClient {
    String provider();
    String chat(ChatRequest request, ModelTarget target);
    StreamCancellationHandle streamChat(ChatRequest request, StreamCallback callback, ModelTarget target);
}

// 实现类只需提供 provider 和调用基类模板方法
@Service
public class BaiLianChatClient extends AbstractOpenAIStyleChatClient {
    @Override
    public String provider() {
        return ModelProvider.BAI_LIAN.getId();
    }

    @Override
    public String chat(ChatRequest request, ModelTarget target) {
        return doChat(request, target);  // 调用模板方法
    }
}
```

**优势**：
- 新增供应商只需实现接口，无需修改现有代码
- 运行时可动态切换供应商
- 符合开闭原则 (OCP)

---

### 3.2 模板方法模式 (Template Method Pattern)

**应用场景**：AbstractOpenAIStyleChatClient、AbstractOpenAIStyleEmbeddingClient

```
┌─────────────────────────────────────────────────────────────────┐
│                    模板方法模式                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   AbstractOpenAIStyleChatClient (抽象基类)                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  模板方法 (final)                                        │   │
│   │  + doChat(request, target)    ──── 同步调用流程           │   │
│   │  + doStreamChat(request, ...) ──── 流式调用流程           │   │
│   │                                                         │   │
│   │  钩子方法 (可覆写)                                        │   │
│   │  # requiresApiKey()           ──── 是否需要 API Key      │   │
│   │  # customizeRequestBody()     ──── 自定义请求体           │   │
│   │  # isReasoningEnabledForStream() ── 是否启用推理          │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│   BaiLianChatClient (具体实现)                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + provider() = "bailian"                                │   │
│   │  + chat() → doChat()                                     │   │
│   │  + streamChat() → doStreamChat()                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   OllamaChatClient (具体实现)                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + provider() = "ollama"                                 │   │
│   │  # requiresApiKey() = false  ←── 覆写钩子                │   │
│   │  + chat() → doChat()                                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 抽象基类 - 定义算法骨架
public abstract class AbstractOpenAIStyleChatClient implements ChatClient {

    // 模板方法：同步调用
    protected String doChat(ChatRequest request, ModelTarget target) {
        // 1. 验证配置
        ProviderConfig provider = requireProvider(target, provider());
        if (requiresApiKey()) requireApiKey(provider, provider());

        // 2. 构建请求体 (可被子类定制)
        JsonObject reqBody = buildRequestBody(request, target, false);

        // 3. 发送 HTTP 请求
        Request httpReq = newAuthorizedRequest(provider, target)
            .post(RequestBody.create(reqBody.toString(), JSON))
            .build();

        // 4. 解析响应
        JsonObject respJson = executeRequest(httpReq);
        return extractChatContent(respJson);
    }

    // 钩子方法：子类可覆写
    protected boolean requiresApiKey() { return true; }
    protected void customizeRequestBody(JsonObject body, ChatRequest request) { ... }
}

// 具体实现 - 只需提供差异部分
@Service
public class OllamaChatClient extends AbstractOpenAIStyleChatClient {
    @Override
    protected boolean requiresApiKey() {
        return false;  // Ollama 本地部署不需要 API Key
    }
}
```

**优势**：
- 消除代码重复，公共逻辑集中在基类
- 子类只需关注差异点
- 算法骨架稳定，扩展点明确

---

### 3.3 装饰器模式 (Decorator Pattern)

**应用场景**：ForwardingStreamCallback、StreamSpanCallback、ProbeStreamBridge

```
┌─────────────────────────────────────────────────────────────────┐
│                    装饰器模式                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │ StreamCallback   │                                          │
│   │ + onContent()    │                                          │
│   │ + onThinking()   │                                          │
│   │ + onComplete()   │                                          │
│   │ + onError()      │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌─────────────────┐      装饰器链:                           │
│   │ForwardingCallback│      ┌─────────────────────────────┐    │
│   │ (抽象装饰器)      │      │ ProbeStreamBridge           │    │
│   │ - delegate       │      │   └─► StreamSpanCallback    │    │
│   │ + onContent()    │      │       └─► ForwardingCallback│    │
│   │ + onFinish()     │      │           └─► 实际 Callback  │    │
│   └─────────────────┘      └─────────────────────────────┘    │
│           ▲                                                    │
│           │ extends                                            │
│           │                                                    │
│   ┌─────────────────┐  ┌─────────────────┐                    │
│   │StreamSpanCallback│  │ProbeStreamBridge│                    │
│   │ (Trace 收尾)     │  │ (首包探测缓冲)   │                    │
│   └─────────────────┘  └─────────────────┘                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 抽象装饰器基类
public abstract class ForwardingStreamCallback implements StreamCallback {
    private final StreamCallback delegate;
    private final AtomicBoolean finished = new AtomicBoolean(false);
    private final AtomicBoolean firstContentSeen = new AtomicBoolean(false);

    @Override
    public final void onContent(String content) {
        // 首包钩子
        if (firstContentSeen.compareAndSet(false, true)) {
            onFirstContent();
        }
        delegate.onContent(content);  // 透传
    }

    @Override
    public final void onComplete() {
        try {
            delegate.onComplete();
        } finally {
            finishOnce(true, null);  // CAS 保证只触发一次
        }
    }

    protected abstract void onFinish(boolean success, Throwable error);
}

// 具体装饰器：Trace 收尾
public final class StreamSpanCallback extends ForwardingStreamCallback {
    private final StreamSpan span;

    @Override
    protected void onFinish(boolean success, Throwable error) {
        if (success) span.finishSuccess();
        else span.finishError(error);
    }
}

// 具体装饰器：首包探测
public final class ProbeStreamBridge implements StreamCallback {
    private final CompletableFuture<ProbeResult> probe = new CompletableFuture<>();
    private final List<Runnable> buffer = new ArrayList<>();
    private volatile boolean committed;

    @Override
    public void onContent(String content) {
        probe.complete(ProbeResult.success());  // 通知探测成功
        bufferOrDispatch(() -> downstream.onContent(content));  // 缓冲或转发
    }

    ProbeResult awaitFirstPacket(long timeout, TimeUnit unit) {
        ProbeResult result = probe.get(timeout, unit);
        if (result.isSuccess()) commit();  // 成功时提交缓冲
        return result;
    }
}
```

**优势**：
- 灵活组合功能，如 Trace + 首包探测 + 实际回调
- 单一职责，每个装饰器只做一件事
- 运行时动态添加功能

---

### 3.4 外观模式 (Facade Pattern)

**应用场景**：RoutingLLMService、RoutingEmbeddingService、RoutingRerankService

```
┌─────────────────────────────────────────────────────────────────┐
│                    外观模式 (Facade)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   业务层                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  调用方 (Bootstrap 层)                                   │   │
│   │  llmService.streamChat(request, callback)               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│   外观层                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  RoutingLLMService (外观)                                │   │
│   │  ┌───────────────────────────────────────────────────┐   │   │
│   │  │ 1. ModelSelector.selectChatCandidates()           │   │   │
│   │  │ 2. 遍历候选列表                                    │   │   │
│   │  │ 3. ModelHealthStore.allowCall()                   │   │   │
│   │  │ 4. ChatClient.streamChat()                        │   │   │
│   │  │ 5. LlmFirstPacketProbe.awaitFirstPacket()         │   │   │
│   │  │ 6. ModelHealthStore.markSuccess/Failure()         │   │   │
│   │  └───────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│   子系统层                                                      │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│   │ModelSelector│  │HealthStore  │  │ ChatClient  │           │
│   └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
@Service
@Primary
public class RoutingLLMService implements LLMService {

    private final ModelSelector selector;
    private final ModelHealthStore healthStore;
    private final ModelRoutingExecutor executor;
    private final LlmFirstPacketProbe firstPacketProbe;
    private final Map<String, ChatClient> clientsByProvider;

    @Override
    public StreamCancellationHandle streamChat(ChatRequest request, StreamCallback callback) {
        // 1. 选择候选模型
        List<ModelTarget> targets = selector.selectChatCandidates(
            Boolean.TRUE.equals(request.getThinking()));

        // 2. 遍历候选，尝试调用
        for (ModelTarget target : targets) {
            ChatClient client = clientsByProvider.get(target.candidate().getProvider());

            // 3. 健康检查
            if (!healthStore.allowCall(target.id())) continue;

            // 4. 首包探测
            ProbeStreamBridge bridge = new ProbeStreamBridge(callback);
            StreamCancellationHandle handle = client.streamChat(request, bridge, target);

            // 5. 等待首包
            ProbeResult result = firstPacketProbe.awaitFirstPacket(bridge, 60, SECONDS);

            if (result.isSuccess()) {
                healthStore.markSuccess(target.id());
                return handle;
            }

            // 6. 失败处理
            healthStore.markFailure(target.id());
            handle.cancel();
        }

        throw notifyAllFailed(callback, lastError);
    }
}
```

**优势**：
- 简化复杂子系统的调用
- 业务层无需了解路由、健康检查、首包探测等细节
- 集中管理横切关注点

---

### 3.5 注册表模式 (Registry Pattern)

**应用场景**：Spring 自动发现 + Map 注册

```
┌─────────────────────────────────────────────────────────────────┐
│                    注册表模式                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Spring 容器                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  @Service BaiLianChatClient                             │   │
│   │  @Service OllamaChatClient                              │   │
│   │  @Service SiliconFlowChatClient                         │   │
│   │  @Service AIHubMixChatClient                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼  自动注入                           │
│   注册表                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  RoutingLLMService 构造函数                              │   │
│   │  ┌───────────────────────────────────────────────────┐   │   │
│   │  │  this.clientsByProvider = clients.stream()        │   │   │
│   │  │      .collect(Collectors.toMap(                   │   │   │
│   │  │          ChatClient::provider,                    │   │   │
│   │  │          Function.identity()                      │   │   │
│   │  │      ));                                          │   │   │
│   │  └───────────────────────────────────────────────────┘   │   │
│   │                                                         │   │
│   │  Map<String, ChatClient>                                │   │
│   │  ┌───────────────────────────────────────────────────┐   │   │
│   │  │  "bailian"    → BaiLianChatClient                 │   │   │
│   │  │  "ollama"     → OllamaChatClient                  │   │   │
│   │  │  "siliconflow"→ SiliconFlowChatClient             │   │   │
│   │  │  "aihubmix"   → AIHubMixChatClient                │   │   │
│   │  └───────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 自动注册
@Service
public class BaiLianChatClient extends AbstractOpenAIStyleChatClient {
    @Override
    public String provider() {
        return ModelProvider.BAI_LIAN.getId();  // "bailian"
    }
}

// 自动发现与注册
@Service
public class RoutingLLMService implements LLMService {

    private final Map<String, ChatClient> clientsByProvider;

    public RoutingLLMService(List<ChatClient> clients) {
        // Spring 自动注入所有 ChatClient 实现
        // 按 provider 建立索引
        this.clientsByProvider = clients.stream()
            .collect(Collectors.toMap(
                ChatClient::provider,
                Function.identity()
            ));
    }
}
```

**优势**：
- 新增供应商只需加 `@Service` 注解，自动被发现
- 无需维护硬编码的供应商列表
- 运行时动态查找，零配置扩展

---

### 3.6 适配器模式 (Adapter Pattern)

**应用场景**：适配不同供应商的 API 差异

```
┌─────────────────────────────────────────────────────────────────┐
│                    适配器模式                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   统一接口                                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  ChatClient.chat(request, target) → String              │   │
│   │  ChatClient.streamChat(request, callback, target)       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│   适配器层                                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  AbstractOpenAIStyleChatClient                          │   │
│   │  ┌───────────────────────────────────────────────────┐   │   │
│   │  │  // 统一 OpenAI 协议格式                           │   │   │
│   │  │  buildRequestBody() → {"model":"...","messages":[]}│   │   │
│   │  │  extractChatContent() → choices[0].message.content │   │   │
│   │  └───────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│   供应商 API                                                    │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│   │ 百炼 API     │  │ Ollama API  │  │ OpenAI API  │           │
│   │ (兼容OpenAI) │  │ (兼容OpenAI) │  │ (标准)      │           │
│   └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**适配内容**：

| 差异点 | 适配方式 |
|--------|---------|
| API Key | `requiresApiKey()` 钩子 |
| 请求体字段 | `customizeRequestBody()` 钩子 |
| URL 路径 | `ModelUrlResolver` 统一解析 |
| 响应格式 | `extractChatContent()` 统一解析 |
| 推理内容 | `isReasoningEnabledForStream()` 钩子 |

---

### 3.7 责任链模式 (Chain of Responsibility)

**应用场景**：模型 Fallback 链

```
┌─────────────────────────────────────────────────────────────────┐
│                    责任链模式 (Fallback)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ModelRoutingExecutor.executeWithFallback()                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │  targets = [ModelA, ModelB, ModelC]  // 按优先级排序     │   │
│   │                                                         │   │
│   │  for (target : targets) {                               │   │
│   │      ┌─────────────────────────────────────────────┐    │   │
│   │      │ 1. healthStore.allowCall(target.id())       │    │   │
│   │      │    └─► 熔断? → skip                          │    │   │
│   │      │                                             │    │   │
│   │      │ 2. client.chat(request, target)              │    │   │
│   │      │    ├─► 成功 → return                         │    │   │
│   │      │    └─► 失败 → markFailure, continue          │    │   │
│   │      └─────────────────────────────────────────────┘    │   │
│   │  }                                                      │   │
│   │                                                         │   │
│   │  throw "All models failed"                              │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   流式调用链:                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  ModelA (priority=1)                                    │   │
│   │      │                                                  │   │
│   │      ├─► 首包成功 → return handle                       │   │
│   │      │                                                  │   │
│   │      └─► 首包失败/超时                                  │   │
│   │              │                                          │   │
│   │              ▼                                          │   │
│   │  ModelB (priority=2)                                    │   │
│   │      │                                                  │   │
│   │      ├─► 首包成功 → return handle                       │   │
│   │      │                                                  │   │
│   │      └─► 首包失败/超时                                  │   │
│   │              │                                          │   │
│   │              ▼                                          │   │
│   │  ModelC (priority=3)                                    │   │
│   │      │                                                  │   │
│   │      └─► 首包失败 → throw "All models failed"           │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.8 观察者模式 (Observer Pattern)

**应用场景**：StreamCallback 流式事件通知

```
┌─────────────────────────────────────────────────────────────────┐
│                    观察者模式 (Observer)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Subject (主题)                                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  AbstractOpenAIStyleChatClient.doStream()               │   │
│   │  ┌───────────────────────────────────────────────────┐   │   │
│   │  │  while (source.readLine()) {                      │   │   │
│   │  │      parsed = SSE_PARSER.parse(line)              │   │   │
│   │  │                                                  │   │   │
│   │  │      if (parsed.hasContent())                     │   │   │
│   │  │          callback.onContent(parsed.content())     │   │   │
│   │  │                                                  │   │   │
│   │  │      if (parsed.hasReasoning())                   │   │   │
│   │  │          callback.onThinking(parsed.reasoning())  │   │   │
│   │  │                                                  │   │   │
│   │  │      if (parsed.completed())                      │   │   │
│   │  │          callback.onComplete()                    │   │   │
│   │  │  }                                               │   │   │
│   │  └───────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼ 通知                               │
│   Observers (观察者)                                            │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│   │ProbeBridge  │  │StreamSpan   │  │业务Callback  │           │
│   │(首包探测)    │  │(Trace)      │  │(SSE推送)     │           │
│   └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、核心组件详解

### 4.1 三大能力接口

| 接口 | 能力 | 方法 |
|------|------|------|
| `ChatClient` | 聊天对话 | `chat()`, `streamChat()` |
| `EmbeddingClient` | 文本向量化 | `embed()`, `embedBatch()` |
| `RerankClient` | 结果重排 | `rerank()` |

### 4.2 路由核心组件

| 组件 | 职责 |
|------|------|
| `ModelSelector` | 候选模型选择与排序 |
| `ModelHealthStore` | 三态熔断器 (CLOSED/OPEN/HALF_OPEN) |
| `ModelRoutingExecutor` | Fallback 执行引擎 |
| `ModelTarget` | 模型目标配置记录 |

### 4.3 流式支持组件

| 组件 | 职责 |
|------|------|
| `StreamCallback` | 流式回调接口 |
| `ForwardingStreamCallback` | 装饰器基类 |
| `ProbeStreamBridge` | 首包探测桥接器 |
| `StreamSpanCallback` | Trace 收尾装饰器 |
| `StreamAsyncExecutor` | 异步执行器 |
| `OpenAIStyleSseParser` | SSE 解析器 |

### 4.4 HTTP 支撑组件

| 组件 | 职责 |
|------|------|
| `ModelUrlResolver` | URL 解析 (候选URL > 提供商URL + 端点) |
| `HttpResponseHelper` | 响应解析工具 |
| `ModelClientException` | 模型客户端异常 |
| `ModelClientErrorType` | 错误类型枚举 |

### 4.5 枚举定义

| 枚举 | 值 |
|------|-----|
| `ModelProvider` | OLLAMA, BAI_LIAN, SILICON_FLOW, AI_HUB_MIX, NOOP |
| `ModelCapability` | CHAT, EMBEDDING, RERANK |

---

## 五、供应商实现对比

### 5.1 ChatClient 实现

| 供应商 | 基类 | API Key | 特殊处理 |
|--------|------|---------|---------|
| BaiLian | AbstractOpenAIStyle | 需要 | 无 |
| Ollama | AbstractOpenAIStyle | **不需要** | `requiresApiKey()=false` |
| SiliconFlow | AbstractOpenAIStyle | 需要 | 无 |
| AIHubMix | AbstractOpenAIStyle | 需要 | 无 |

### 5.2 EmbeddingClient 实现

| 供应商 | 基类 | 特殊处理 |
|--------|------|---------|
| Ollama | AbstractOpenAIStyle | 不需要 API Key |
| AIHubMix | AbstractOpenAIStyle | 标准实现 |
| SiliconFlow | AbstractOpenAIStyle | 标准实现 |

### 5.3 RerankClient 实现

| 供应商 | 实现方式 | 说明 |
|--------|---------|------|
| BaiLian | 独立实现 | 百炼特有 API 格式 |
| Noop | 空实现 | 直接返回前 N 个，用于测试 |

---

## 六、配置结构

```yaml
ai:
  # 提供商配置
  providers:
    ollama:
      url: http://localhost:11434
      endpoints:
        chat: /v1/chat/completions
        embedding: /v1/embeddings
    bailian:
      url: https://dashscope.aliyuncs.com
      api-key: ${BAILIAN_API_KEY}
      endpoints:
        chat: /compatible-mode/v1/chat/completions
        rerank: /api/v1/services/rerank/text-rerank/text-rerank

  # 熔断配置
  selection:
    failure-threshold: 2
    open-duration-ms: 30000

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

  # Embedding 模型组
  embedding:
    default-model: text-embedding-v3
    candidates:
      - id: text-embedding-v3
        provider: bailian
        model: text-embedding-v3
        dimension: 1536

  # Rerank 模型组
  rerank:
    default-model: gte-rerank
    candidates:
      - id: gte-rerank
        provider: bailian
        model: gte-rerank
```

---

## 七、源文件索引

### Chat 模块
| 文件 | 职责 | 设计模式 |
|------|------|---------|
| `chat/ChatClient.java` | 聊天客户端接口 | 策略 |
| `chat/LLMService.java` | LLM 服务接口 | - |
| `chat/RoutingLLMService.java` | 路由式 LLM 服务 | 外观 |
| `chat/AbstractOpenAIStyleChatClient.java` | OpenAI 协议基类 | 模板方法 |
| `chat/BaiLianChatClient.java` | 百炼实现 | 策略 |
| `chat/OllamaChatClient.java` | Ollama 实现 | 策略 |
| `chat/SiliconFlowChatClient.java` | SiliconFlow 实现 | 策略 |
| `chat/AIHubMixChatClient.java` | AIHubMix 实现 | 策略 |
| `chat/StreamCallback.java` | 流式回调接口 | 观察者 |
| `chat/ForwardingStreamCallback.java` | 透传装饰器基类 | 装饰器 |
| `chat/StreamSpanCallback.java` | Trace 收尾装饰器 | 装饰器 |
| `chat/ProbeStreamBridge.java` | 首包探测桥接器 | 装饰器 |
| `chat/LlmFirstPacketProbe.java` | 首包探测 AOP | - |
| `chat/StreamAsyncExecutor.java` | 异步执行器 | - |
| `chat/OpenAIStyleSseParser.java` | SSE 解析器 | - |
| `chat/StreamCancellationHandle.java` | 取消句柄接口 | - |

### Embedding 模块
| 文件 | 职责 | 设计模式 |
|------|------|---------|
| `embedding/EmbeddingClient.java` | 向量客户端接口 | 策略 |
| `embedding/EmbeddingService.java` | 向量服务接口 | - |
| `embedding/RoutingEmbeddingService.java` | 路由式向量服务 | 外观 |
| `embedding/AbstractOpenAIStyleEmbeddingClient.java` | OpenAI 协议基类 | 模板方法 |
| `embedding/OllamaEmbeddingClient.java` | Ollama 实现 | 策略 |
| `embedding/AIHubMixEmbeddingClient.java` | AIHubMix 实现 | 策略 |
| `embedding/SiliconFlowEmbeddingClient.java` | SiliconFlow 实现 | 策略 |

### Rerank 模块
| 文件 | 职责 | 设计模式 |
|------|------|---------|
| `rerank/RerankClient.java` | 重排客户端接口 | 策略 |
| `rerank/RerankService.java` | 重排服务接口 | - |
| `rerank/RoutingRerankService.java` | 路由式重排服务 | 外观 |
| `rerank/BaiLianRerankClient.java` | 百炼实现 | 策略 |
| `rerank/NoopRerankClient.java` | 空实现 (测试) | 策略 |

### Model 路由模块
| 文件 | 职责 | 设计模式 |
|------|------|---------|
| `model/ModelSelector.java` | 候选选择器 | - |
| `model/ModelHealthStore.java` | 健康状态存储 | 状态机 |
| `model/ModelRoutingExecutor.java` | 路由执行器 | 责任链 |
| `model/ModelTarget.java` | 模型目标记录 | - |
| `model/ModelCaller.java` | 调用器函数接口 | - |

### HTTP 支撑模块
| 文件 | 职责 |
|------|------|
| `http/ModelUrlResolver.java` | URL 解析器 |
| `http/HttpResponseHelper.java` | 响应解析工具 |
| `http/ModelClientException.java` | 模型客户端异常 |
| `http/ModelClientErrorType.java` | 错误类型枚举 |
| `http/HttpMediaTypes.java` | HTTP 媒体类型 |

### 其他
| 文件 | 职责 |
|------|------|
| `config/AIModelProperties.java` | 配置属性 |
| `enums/ModelProvider.java` | 提供商枚举 |
| `enums/ModelCapability.java` | 能力枚举 |
| `token/TokenCounterService.java` | Token 统计接口 |
| `token/HeuristicTokenCounterService.java` | 启发式 Token 统计 |
| `util/LLMResponseCleaner.java` | 响应清理工具 |

---

## 八、设计模式总结

| 设计模式 | 应用场景 | 解决的问题 |
|---------|---------|-----------|
| **策略模式** | ChatClient, EmbeddingClient, RerankClient | 供应商可插拔替换 |
| **模板方法** | AbstractOpenAIStyle* | 公共逻辑复用，差异点可定制 |
| **装饰器** | ForwardingStreamCallback, ProbeStreamBridge | 灵活组合流式处理功能 |
| **外观模式** | RoutingLLMService | 简化复杂子系统调用 |
| **注册表模式** | Spring 自动发现 + Map 注册 | 零配置扩展供应商 |
| **适配器模式** | AbstractOpenAIStyle* | 统一不同供应商 API 差异 |
| **责任链模式** | ModelRoutingExecutor | 按优先级尝试，自动故障转移 |
| **观察者模式** | StreamCallback | 流式事件异步通知 |
