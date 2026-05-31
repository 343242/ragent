# Ragent 流式输出实现研究报告

## 一、整体架构概览

Ragent 的流式输出基于 **SSE (Server-Sent Events) + 观察者模式 + 装饰器模式**，实现了从 LLM 到前端的实时流式推送。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          流式输出架构                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Frontend (SSE Client)                                                     │
│       │                                                                     │
│       │ SSE Connection (text/event-stream)                                  │
│       ▼                                                                     │
│   SseEmitterSender (线程安全封装)                                            │
│       │                                                                     │
│       │ sendEvent(eventName, data)                                          │
│       ▼                                                                     │
│   StreamCallback (观察者接口)                                                │
│       │                                                                     │
│       ├──► onContent(content)   ──── 增量内容                               │
│       ├──► onThinking(content)  ──── 思考过程 (深度思考模式)                 │
│       ├──► onComplete()         ──── 完成                                   │
│       └──► onError(error)       ──── 错误                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、核心组件

### 2.1 StreamCallback 接口 (观察者模式)

```java
public interface StreamCallback {
    /**
     * 接收增量内容
     * 模型推送的每一段内容都会通过该方法回调
     */
    void onContent(String content);

    /**
     * 接收思考过程增量内容 (深度思考模式)
     * 默认空实现，未支持思考的场景可以忽略
     */
    default void onThinking(String content) {}

    /**
     * 整个推理流程结束
     */
    void onComplete();

    /**
     * 流式推送过程中出现异常
     */
    void onError(Throwable error);
}
```

### 2.2 SseEmitterSender (线程安全 SSE 封装)

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

    /**
     * 发送 SSE 事件 (线程安全)
     */
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

    /**
     * 正常完成 (CAS 保证幂等)
     */
    public void complete() {
        if (closed.compareAndSet(false, true)) {
            emitter.complete();
        }
    }

    /**
     * 异常结束 (CAS 保证幂等)
     */
    public void fail(Throwable throwable) {
        if (closed.compareAndSet(false, true)) {
            emitter.completeWithError(throwable);
        }
    }
}
```

**设计亮点**：
- **线程安全**：`AtomicBoolean` CAS 操作
- **幂等关闭**：`compareAndSet` 保证只关闭一次
- **快速失败**：连接已关闭时直接返回
- **生命周期感知**：`onCompletion/onTimeout/onError` 自动同步状态

### 2.3 ForwardingStreamCallback (装饰器模式)

```java
public abstract class ForwardingStreamCallback implements StreamCallback {
    private final StreamCallback delegate;
    private final AtomicBoolean finished = new AtomicBoolean(false);
    private final AtomicBoolean firstContentSeen = new AtomicBoolean(false);

    @Override
    public final void onContent(String content) {
        // 首包钩子 (仅触发一次)
        if (firstContentSeen.compareAndSet(false, true)) {
            try {
                onFirstContent();
            } catch (Throwable ex) {
                // 钩子异常不能影响正常推流
            }
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

    @Override
    public final void onError(Throwable error) {
        try {
            delegate.onError(error);
        } finally {
            finishOnce(false, error);
        }
    }

    /**
     * 外部路径触发收尾 (如 cancel)
     */
    protected final void finishExternally(boolean success, Throwable error) {
        finishOnce(success, error);
    }

    private void finishOnce(boolean success, Throwable error) {
        if (!finished.compareAndSet(false, true)) {
            return;
        }
        onFinish(success, error);
    }

    /**
     * 流式终态收尾 (仅触发一次)
     */
    protected abstract void onFinish(boolean success, Throwable error);
}
```

---

## 三、流式调用完整流程

### 3.1 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          流式调用完整流程                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. 前端发起 SSE 连接                                                      │
│      └── EventSource("/api/ragent/chat/stream")                             │
│                                                                             │
│   2. Controller 创建 SseEmitter                                             │
│      └── SseEmitter emitter = new SseEmitter(300_000)  // 5分钟超时         │
│                                                                             │
│   3. 创建 StreamCallback 实现                                               │
│      └── 将 SseEmitterSender 包装为 StreamCallback                          │
│                                                                             │
│   4. 调用 RoutingLLMService.streamChat(request, callback)                   │
│      │                                                                      │
│      ├──► ModelSelector.selectChatCandidates(deepThinking)                  │
│      │    // 选择候选模型 (按优先级排序)                                     │
│      │                                                                      │
│      ├──► 遍历候选模型 for (ModelTarget target : targets)                   │
│      │    │                                                                 │
│      │    ├──► ModelHealthStore.allowCall(target.id())                      │
│      │    │    // 健康检查 (熔断器)                                          │
│      │    │                                                                 │
│      │    ├──► ProbeStreamBridge bridge = new ProbeStreamBridge(callback)   │
│      │    │    // 创建首包探测桥接器                                         │
│      │    │                                                                 │
│      │    ├──► ChatClient.streamChat(request, bridge, target)               │
│      │    │    // 发起流式请求 (异步)                                        │
│      │    │                                                                 │
│      │    ├──► LlmFirstPacketProbe.awaitFirstPacket(bridge, 60, SECONDS)    │
│      │    │    // 等待首包 (60秒超时)                                        │
│      │    │                                                                 │
│      │    ├──► if (result.isSuccess())                                      │
│      │    │    ├── healthStore.markSuccess(target.id())                     │
│      │    │    └── return handle  // 成功，返回句柄                          │
│      │    │                                                                 │
│      │    └──► if (result.isFail())                                         │
│      │         ├── healthStore.markFailure(target.id())                     │
│      │         ├── handle.cancel()  // 取消当前请求                          │
│      │         └── continue  // 尝试下一个模型                               │
│      │                                                                      │
│   5. 流式输出 (异步线程)                                                    │
│      │                                                                      │
│      └──► StreamAsyncExecutor.submit(modelStreamExecutor, ...)              │
│           │                                                                 │
│           └──► CompletableFuture.runAsync(() -> {                           │
│                  Response response = call.execute();                         │
│                  BufferedSource source = response.body().source();           │
│                  │                                                          │
│                  while (!cancelled.get()) {                                  │
│                      String line = source.readUtf8Line();                    │
│                      │                                                      │
│                      ParsedEvent event = OpenAIStyleSseParser.parseLine()    │
│                      │                                                      │
│                      ├──► if (event.hasContent())                            │
│                      │       callback.onContent(event.content())             │
│                      │                                                      │
│                      ├──► if (event.hasReasoning())                          │
│                      │       callback.onThinking(event.reasoning())          │
│                      │                                                      │
│                      └──► if (event.completed())                             │
│                              callback.onComplete()                           │
│                              break                                           │
│                  }                                                          │
│                }, modelStreamExecutor)                                       │
│                                                                             │
│   6. 前端接收 SSE 事件                                                      │
│      └── eventSource.onmessage = (event) => {                               │
│              const data = JSON.parse(event.data);                            │
│              // 渲染增量内容                                                 │
│          }                                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心代码

```java
// RoutingLLMService.streamChat()
@Override
public StreamCancellationHandle streamChat(ChatRequest request, StreamCallback callback) {
    // 1. 选择候选模型
    List<ModelTarget> targets = selector.selectChatCandidates(
        Boolean.TRUE.equals(request.getThinking()));

    // 2. 遍历候选模型
    for (ModelTarget target : targets) {
        ChatClient client = resolveClient(target, label);

        // 3. 健康检查
        if (!healthStore.allowCall(target.id())) {
            continue;
        }

        // 4. 创建首包探测桥接器
        ProbeStreamBridge bridge = new ProbeStreamBridge(callback);

        // 5. 发起流式请求
        StreamCancellationHandle handle;
        try {
            handle = client.streamChat(request, bridge, target);
        } catch (Exception e) {
            healthStore.markFailure(target.id());
            continue;
        }

        // 6. 等待首包
        ProbeStreamBridge.ProbeResult result = awaitFirstPacket(bridge, handle, callback);

        // 7. 判断结果
        if (result.isSuccess()) {
            healthStore.markSuccess(target.id());
            return handle;  // 成功，返回句柄
        }

        // 8. 失败处理
        healthStore.markFailure(target.id());
        handle.cancel();
    }

    // 9. 所有模型都失败
    throw notifyAllFailed(callback, lastError);
}
```

---

## 四、首包探测机制 (ProbeStreamBridge)

### 4.1 设计目的

- **问题**：流式请求启动后，如果模型响应慢或失败，用户需要等待很长时间
- **方案**：首包探测机制，在收到第一个有效内容前，可以快速切换模型
- **效果**：用户无感知的模型切换，提升体验

### 4.2 实现原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          首包探测机制                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ProbeStreamBridge (装饰器 + CompletableFuture)                            │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  CompletableFuture<ProbeResult> probe = new CompletableFuture<>()   │   │
│   │  List<Runnable> buffer = new ArrayList<>()  // 事件缓冲区           │   │
│   │  volatile boolean committed = false  // 是否已提交                   │   │
│   │                                                                     │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  onContent(content) {                                       │    │   │
│   │  │      probe.complete(ProbeResult.success());                 │    │   │
│   │  │      bufferOrDispatch(() -> downstream.onContent(content)); │    │   │
│   │  │  }                                                          │    │   │
│   │  │                                                             │    │   │
│   │  │  onComplete() {                                             │    │   │
│   │  │      probe.complete(ProbeResult.noContent());               │    │   │
│   │  │      bufferOrDispatch(downstream::onComplete);              │    │   │
│   │  │  }                                                          │    │   │
│   │  │                                                             │    │   │
│   │  │  onError(Throwable t) {                                     │    │   │
│   │  │      probe.complete(ProbeResult.error(t));                  │    │   │
│   │  │      bufferOrDispatch(() -> downstream.onError(t));         │    │   │
│   │  │  }                                                          │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   │                                                                     │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  awaitFirstPacket(timeout, unit) {                          │    │   │
│   │  │      ProbeResult result = probe.get(timeout, unit);         │    │   │
│   │  │      if (result.isSuccess()) {                              │    │   │
│   │  │          commit();  // 成功时提交缓冲                        │    │   │
│   │  │      }                                                      │    │   │
│   │  │      return result;                                         │    │   │
│   │  │  }                                                          │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   │                                                                     │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  commit() {                                                 │    │   │
│   │  │      synchronized (lock) {                                  │    │   │
│   │  │          if (committed) return;                             │    │   │
│   │  │          committed = true;                                  │    │   │
│   │  │          buffer.forEach(Runnable::run);  // 提交所有缓冲     │    │   │
│   │  │      }                                                      │    │   │
│   │  │  }                                                          │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   │                                                                     │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  bufferOrDispatch(Runnable action) {                        │    │   │
│   │  │      boolean dispatchNow;                                   │    │   │
│   │  │      synchronized (lock) {                                  │    │   │
│   │  │          dispatchNow = committed;                           │    │   │
│   │  │          if (!dispatchNow) {                                │    │   │
│   │  │              buffer.add(action);  // 未提交，先缓冲          │    │   │
│   │  │          }                                                  │    │   │
│   │  │      }                                                      │    │   │
│   │  │      if (dispatchNow) {                                     │    │   │
│   │  │          action.run();  // 已提交，直接执行                  │    │   │
│   │  │      }                                                      │    │   │
│   │  │  }                                                          │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   探测结果类型:                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  enum Type { SUCCESS, ERROR, TIMEOUT, NO_CONTENT }                  │   │
│   │                                                                     │   │
│   │  SUCCESS ──── 收到内容 (onContent 或 onThinking)                    │   │
│   │  ERROR ────── 发生错误 (onError)                                    │   │
│   │  TIMEOUT ──── 超时 (60秒未收到首包)                                  │   │
│   │  NO_CONTENT ── 完成但无内容 (onComplete 但无 onContent)              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 设计亮点

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| **首包探测** | `CompletableFuture.get(timeout)` | 超时自动切换模型 |
| **事件缓冲** | `List<Runnable> buffer` | 首包到达前缓冲所有事件 |
| **原子提交** | `synchronized` + `committed` 标记 | 线程安全，只提交一次 |
| **延迟派发** | `bufferOrDispatch()` | 未提交时缓冲，已提交时直接派发 |

---

## 五、SSE 解析器 (OpenAIStyleSseParser)

### 5.1 解析流程

```java
public final class OpenAIStyleSseParser {

    static ParsedEvent parseLine(String line, Gson gson, boolean reasoningEnabled) {
        // 1. 提取 data: 前缀
        String payload = line.trim();
        if (payload.startsWith("data:")) {
            payload = payload.substring(5).trim();
        }

        // 2. 检查结束标记
        if ("[DONE]".equalsIgnoreCase(payload)) {
            return ParsedEvent.done();
        }

        // 3. 解析 JSON
        JsonObject obj = gson.fromJson(payload, JsonObject.class);
        JsonArray choices = obj.getAsJsonArray("choices");
        if (choices == null || choices.isEmpty()) {
            return ParsedEvent.empty();
        }

        // 4. 提取内容
        JsonObject choice0 = choices.get(0).getAsJsonObject();
        String content = extractText(choice0, "content");
        String reasoning = reasoningEnabled ? extractText(choice0, "reasoning_content") : null;
        boolean completed = hasFinishReason(choice0);

        return new ParsedEvent(content, reasoning, completed);
    }

    private static String extractText(JsonObject choice, String fieldName) {
        // 优先从 delta 提取 (流式)
        if (choice.has("delta")) {
            JsonObject delta = choice.getAsJsonObject("delta");
            if (delta.has(fieldName)) {
                JsonElement value = delta.get(fieldName);
                if (value != null && !value.isJsonNull()) {
                    return value.getAsString();
                }
            }
        }
        // 回退到 message 提取 (非流式)
        if (choice.has("message")) {
            JsonObject message = choice.getAsJsonObject("message");
            if (message.has(fieldName)) {
                JsonElement value = message.get(fieldName);
                if (value != null && !value.isJsonNull()) {
                    return value.getAsString();
                }
            }
        }
        return null;
    }
}
```

### 5.2 SSE 数据格式

```
data: {"id":"chatcmpl-xxx","choices":[{"delta":{"content":"你"},"index":0}]}

data: {"id":"chatcmpl-xxx","choices":[{"delta":{"content":"好"},"index":0}]}

data: {"id":"chatcmpl-xxx","choices":[{"delta":{},"finish_reason":"stop","index":0}]}

data: [DONE]
```

---

## 六、装饰器组合链

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          装饰器组合链                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   业务层 Callback (最终消费者)                                               │
│       │                                                                     │
│       │ 包装                                                               │
│       ▼                                                                     │
│   StreamSpanCallback (Trace 收尾装饰器)                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  extends ForwardingStreamCallback                                   │   │
│   │                                                                     │   │
│   │  onFinish(success, error) {                                         │   │
│   │      if (success) span.finishSuccess();                             │   │
│   │      else span.finishError(error);                                  │   │
│   │  }                                                                  │   │
│   │                                                                     │   │
│   │  onCancel() {                                                       │   │
│   │      span.finishCancelledIfRunning();                               │   │
│   │      finishExternally(false, null);                                 │   │
│   │  }                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       │ 包装                                                               │
│       ▼                                                                     │
│   ProbeStreamBridge (首包探测装饰器)                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  implements StreamCallback                                          │   │
│   │                                                                     │   │
│   │  - 首包探测 + 事件缓冲                                               │   │
│   │  - awaitFirstPacket() 阻塞等待                                      │   │
│   │  - commit() 提交缓冲                                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       │ 传递                                                               │
│       ▼                                                                     │
│   ChatClient.streamChat() (底层 HTTP 流式调用)                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  - OkHttp 异步请求                                                  │   │
│   │  - SSE 流式读取                                                     │   │
│   │  - OpenAIStyleSseParser 解析                                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 七、异步执行 (StreamAsyncExecutor)

```java
public final class StreamAsyncExecutor {

    static StreamCancellationHandle submit(Executor executor,
                                           Call call,
                                           StreamCallback callback,
                                           Consumer<AtomicBoolean> streamTask) {
        AtomicBoolean cancelled = new AtomicBoolean(false);
        try {
            // 提交到线程池异步执行
            CompletableFuture.runAsync(() -> streamTask.accept(cancelled), executor);
        } catch (RejectedExecutionException ex) {
            // 线程池拒绝时，取消请求并通知回调
            call.cancel();
            callback.onError(new ModelClientException("流式线程池繁忙", ...));
            return StreamCancellationHandles.noop();
        }
        // 返回取消句柄
        return StreamCancellationHandles.fromOkHttp(call, cancelled);
    }
}
```

**设计要点**：
- 使用专用线程池 `modelStreamExecutor` 执行流式任务
- 返回 `StreamCancellationHandle` 用于取消
- 线程池满时优雅降级

---

## 八、取消机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          取消机制                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   StreamCancellationHandle (取消句柄接口)                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  void cancel();                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   实现:                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  StreamCancellationHandles.fromOkHttp(Call call, AtomicBoolean cancelled) │
│   │                                                                     │   │
│   │  cancel() {                                                        │   │
│   │      cancelled.set(true);  // 设置取消标志                          │   │
│   │      call.cancel();        // 取消 OkHttp 请求                      │   │
│   │  }                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   取消流程:                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  1. 前端断开连接 或 调用 cancel()                                    │   │
│   │  2. cancelled.set(true)                                             │   │
│   │  3. call.cancel() → OkHttp 取消底层连接                             │   │
│   │  4. 流式读取循环检测 cancelled.get() → 退出                         │   │
│   │  5. StreamSpanCallback.onCancel() → Trace 收尾                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键设计点总结

| 设计点 | 实现方式 | 优势 |
|--------|---------|------|
| **线程安全** | `AtomicBoolean` CAS | 多线程并发发送不冲突 |
| **幂等关闭** | `compareAndSet` | 避免重复关闭导致异常 |
| **首包探测** | `CompletableFuture` + 缓冲 | 用户无感知的模型切换 |
| **装饰器链** | `ForwardingStreamCallback` | 灵活组合功能 (Trace + 探测 + 业务) |
| **异步执行** | `StreamAsyncExecutor` | 流式读取不阻塞主线程 |
| **超时控制** | `probe.get(timeout)` | 首包超时自动切换模型 |
| **取消机制** | `AtomicBoolean` + `Call.cancel()` | 优雅取消，资源释放 |
| **SSE 解析** | `OpenAIStyleSseParser` | 兼容 OpenAI 协议，支持 reasoning_content |

---

## 十、配置参数

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 100MB

rag:
  default:
    sse-timeout-ms: 300000  # SSE 全局超时 5 分钟

ai:
  stream:
    message-chunk-size: 1   # 消息分块大小
```

---

## 十一、源文件索引

| 文件 | 模块 | 职责 |
|------|------|------|
| `StreamCallback.java` | infra-ai | 流式回调接口 (观察者) |
| `ForwardingStreamCallback.java` | infra-ai | 透传装饰器基类 (装饰器) |
| `StreamSpanCallback.java` | infra-ai | Trace 收尾装饰器 |
| `ProbeStreamBridge.java` | infra-ai | 首包探测桥接器 |
| `LlmFirstPacketProbe.java` | infra-ai | 首包探测 AOP 切面 |
| `StreamAsyncExecutor.java` | infra-ai | 异步执行器 |
| `StreamCancellationHandle.java` | infra-ai | 取消句柄接口 |
| `StreamCancellationHandles.java` | infra-ai | 取消句柄工厂 |
| `OpenAIStyleSseParser.java` | infra-ai | SSE 解析器 |
| `SseEmitterSender.java` | framework | SSE 线程安全封装 |
| `RoutingLLMService.java` | infra-ai | 路由式 LLM 服务 (协调者) |
| `AbstractOpenAIStyleChatClient.java` | infra-ai | OpenAI 协议基类 (模板方法) |

---

## 十二、设计模式汇总

| 设计模式 | 应用场景 | 核心组件 |
|---------|---------|---------|
| **观察者模式** | 流式事件通知 | `StreamCallback` |
| **装饰器模式** | 流式回调增强 | `ForwardingStreamCallback`, `ProbeStreamBridge`, `StreamSpanCallback` |
| **模板方法** | 流式调用流程 | `AbstractOpenAIStyleChatClient.doStreamChat()` |
| **外观模式** | 简化流式调用 | `RoutingLLMService.streamChat()` |
| **责任链** | 模型 Fallback | `ModelRoutingExecutor` |
| **工厂模式** | 取消句柄创建 | `StreamCancellationHandles` |
