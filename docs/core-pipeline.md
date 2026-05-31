# Ragent 核心流水线组织研究报告

## 一、整体架构概览

Ragent 的核心流水线分为两条主线：

1. **文档入库流水线** (Ingestion Pipeline) - 文档从上传到可检索的完整链路
2. **查询处理流水线** (RAG Pipeline) - 用户问题从输入到回答的完整链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Ragent 核心流水线                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    文档入库流水线 (Ingestion)                         │    │
│  │  Fetcher ──► Parser ──► Chunker ──► Enhancer ──► Enricher ──► Indexer│    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    查询处理流水线 (RAG)                               │    │
│  │  Question ──► Rewrite ──► Intent ──► Retrieve ──► Prompt ──► LLM    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、文档入库流水线 (Ingestion Pipeline)

### 2.1 流水线架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Ingestion Pipeline                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   IngestionEngine (执行引擎)                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  1. 验证 Pipeline 配置 (环检测)                                      │   │
│   │  2. 找到起始节点 (无入边的节点)                                       │   │
│   │  3. 链式执行节点                                                     │   │
│   │  4. 条件评估 + 日志记录                                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   节点执行链:                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │   │
│   │  │ Fetcher  │───►│  Parser  │───►│ Chunker  │───►│ Indexer  │      │   │
│   │  │  Node    │    │   Node   │    │   Node   │    │   Node   │      │   │
│   │  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │   │
│   │       │               │               │               │             │   │
│   │       ▼               ▼               ▼               ▼             │   │
│   │  DocumentParser  ChunkingStrategy  EmbeddingService  VectorStore    │   │
│   │  Selector        Factory                                              │   │
│   │                                                                     │   │
│   │  可选节点:                                                           │   │
│   │  ┌──────────┐    ┌──────────┐                                       │   │
│   │  │ Enhancer │    │ Enricher │                                       │   │
│   │  │   Node   │    │   Node   │                                       │   │
│   │  └──────────┘    └──────────┘                                       │   │
│   │  (LLM文本增强)    (元数据丰富)                                        │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 节点职责详解

| 节点 | 职责 | 输入 | 输出 | 使用的策略/组件 |
|------|------|------|------|----------------|
| **FetcherNode** | 从不同源获取文档 | `DocumentSource` | `rawBytes` | `DocumentFetcher` (策略模式) |
| **ParserNode** | 解析文档为文本 | `rawBytes` | `rawText`, `StructuredDocument` | `DocumentParserSelector` (注册表) |
| **EnhancerNode** | LLM 增强文本 | `rawText` | `enhancedText` | LLM 服务 |
| **EnricherNode** | 元数据丰富 | `rawText` | `metadata` | LLM 服务 |
| **ChunkerNode** | 文本分块 + 向量化 | `rawText`/`enhancedText` | `List<VectorChunk>` | `ChunkingStrategyFactory` (工厂) |
| **IndexerNode** | 写入向量数据库 | `List<VectorChunk>` | 索引完成 | 向量存储服务 |

### 2.3 Pipeline 配置结构

```java
// Pipeline 定义 (数据库存储)
public class PipelineDefinition {
    private String pipelineId;
    private List<NodeConfig> nodes;  // 节点列表
}

// 节点配置
public class NodeConfig {
    private String nodeId;        // 节点 ID
    private String nodeType;      // 节点类型 (FETCHER, PARSER, CHUNKER, etc.)
    private String nextNodeId;    // 下一个节点 ID (链式传递)
    private JsonNode condition;   // 执行条件 (SpEL 表达式)
    private JsonNode settings;    // 节点配置 (分块大小、解析规则等)
}
```

### 2.4 执行引擎核心逻辑

```java
@Component
public class IngestionEngine {

    private final Map<String, IngestionNode> nodeMap;  // 节点注册表

    public IngestionContext execute(PipelineDefinition pipeline, IngestionContext context) {
        // 1. 验证 Pipeline (环检测)
        validatePipeline(nodeConfigMap);

        // 2. 找到起始节点 (无入边的节点)
        String startNodeId = findStartNode(nodeConfigMap);

        // 3. 链式执行
        executeChain(startNodeId, nodeConfigMap, context);

        return context;
    }

    private void executeChain(String nodeId, Map<String, NodeConfig> nodeConfigMap,
                              IngestionContext context) {
        String currentNodeId = nodeId;
        while (currentNodeId != null) {
            NodeConfig config = nodeConfigMap.get(currentNodeId);
            IngestionNode node = nodeMap.get(config.getNodeType());

            // 条件评估
            if (config.getCondition() != null) {
                if (!conditionEvaluator.evaluate(context, config.getCondition())) {
                    // 跳过该节点
                    currentNodeId = config.getNextNodeId();
                    continue;
                }
            }

            // 执行节点
            NodeResult result = node.execute(context, config);

            // 记录日志
            context.getLogs().add(NodeLog.builder()
                .nodeId(currentNodeId)
                .nodeType(config.getNodeType())
                .durationMs(duration)
                .success(result.isSuccess())
                .build());

            // 判断是否继续
            if (!result.isSuccess() || !result.isShouldContinue()) {
                break;
            }

            // 移动到下一个节点
            currentNodeId = config.getNextNodeId();
        }
    }
}
```

### 2.5 设计模式

| 设计模式 | 应用 | 说明 |
|---------|------|------|
| **模板方法** | `IngestionNode` 接口 | 统一节点执行流程，子类只关注核心逻辑 |
| **策略模式** | `DocumentFetcher`, `ChunkingStrategy` | 可插拔的文档获取和分块策略 |
| **注册表模式** | `DocumentParserSelector`, `FetcherNode` | 自动发现并注册组件 |
| **责任链** | `IngestionEngine` 链式执行 | 节点按配置顺序串行执行 |
| **工厂模式** | `ChunkingStrategyFactory` | 根据配置创建分块策略 |

---

## 三、查询处理流水线 (RAG Pipeline)

### 3.1 流水线架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          RAG Query Pipeline                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   用户问题                                                                  │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 1: 问题预处理                                                  │   │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │   │
│   │  │ QueryRewrite │  │ MultiQuestion│  │ QueryTerm    │               │   │
│   │  │ Service      │  │ RewriteService│  │ MappingService│              │   │
│   │  │ (问题重写)    │  │ (问题拆分)    │  │ (术语映射)    │               │   │
│   │  └──────────────┘  └──────────────┘  └──────────────┘               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 2: 意图识别                                                    │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  IntentClassifier.classifyTargets(question)                 │    │   │
│   │  │      │                                                      │    │   │
│   │  │      ├──► KB 意图 ──── 知识库检索                            │    │   │
│   │  │      │                                                      │    │   │
│   │  │      └──► MCP 意图 ──── 工具调用                             │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 3: 并行检索                                                    │   │
│   │  ┌───────────────────────┐     ┌───────────────────────┐            │   │
│   │  │   KB 检索 (并行)       │     │   MCP 调用 (并行)      │            │   │
│   │  │  ┌─────────────────┐  │     │  ┌─────────────────┐  │            │   │
│   │  │  │ VectorGlobal    │  │     │  │ McpToolExecutor │  │            │   │
│   │  │  │ SearchChannel   │  │     │  │ .execute()      │  │            │   │
│   │  │  └─────────────────┘  │     │  └─────────────────┘  │            │   │
│   │  │  ┌─────────────────┐  │     │                       │            │   │
│   │  │  │ IntentDirected  │  │     │                       │            │   │
│   │  │  │ SearchChannel   │  │     │                       │            │   │
│   │  │  └─────────────────┘  │     │                       │            │   │
│   │  └───────────────────────┘     └───────────────────────┘            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 4: 后处理                                                      │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  PostProcessor 链 (串行)                                     │    │   │
│   │  │  DeduplicationPostProcessor ──► RerankPostProcessor         │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 5: Prompt 编排                                                 │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  RAGPromptService.buildStructuredMessages()                 │    │   │
│   │  │      │                                                      │    │   │
│   │  │      ├──► system prompt (系统提示词)                         │    │   │
│   │  │      ├──► conversation history (对话历史)                    │    │   │
│   │  │      ├──► evidence (KB + MCP 上下文)                         │    │   │
│   │  │      └──► user question (用户问题)                           │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  阶段 6: LLM 流式调用                                                │   │
│   │  ┌─────────────────────────────────────────────────────────────┐    │   │
│   │  │  RoutingLLMService.streamChat()                             │    │   │
│   │  │      │                                                      │    │   │
│   │  │      ├──► ModelSelector ──── 选择候选模型                    │    │   │
│   │  │      ├──► ModelHealthStore ── 健康检查 (熔断器)               │    │   │
│   │  │      ├──► ChatClient.streamChat() ── 首包探测               │    │   │
│   │  │      └──► StreamCallback ──── SSE 推送到前端                 │    │   │
│   │  └─────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 阶段详解

#### 阶段 1: 问题预处理

```java
// 问题重写 (补全上下文)
QueryRewriteService.rewrite(question, history)
    → 补全代词、补全省略信息

// 复杂问题拆分
MultiQuestionRewriteService.rewriteToSubQuestions(question)
    → 将复杂问题拆分为多个子问题

// 术语映射
QueryTermMappingService.getMappings(query)
    → 专业术语 → 通俗解释
```

#### 阶段 2: 意图识别

```java
// 意图分类
List<NodeScore> scores = IntentClassifier.classifyTargets(question);

// 意图分离
List<NodeScore> kbIntents = NodeScoreFilters.kb(scores);   // KB 意图
List<NodeScore> mcpIntents = NodeScoreFilters.mcp(scores);  // MCP 意图
```

**意图节点结构**：
```java
public class IntentNode {
    private String id;
    private String name;
    private String description;
    private List<String> sampleQuestions;
    private String promptTemplate;    // 意图专属 Prompt
    private String mcpToolId;         // MCP 工具 ID
    private Integer topK;             // 检索 TopK
    private Double minScore;          // 最低分数阈值
}
```

#### 阶段 3: 并行检索

```java
// KB 检索 (多通道并行)
List<RetrievedChunk> chunks = MultiChannelRetrievalEngine.retrieveKnowledgeChannels(subIntents, topK);
    ├── VectorGlobalSearchChannel.search()      // 向量全局检索 (并行)
    └── IntentDirectedSearchChannel.search()    // 意图定向检索 (并行)

// MCP 调用 (并行)
Map<String, CallToolResult> mcpResults = executeMcpTools(question, mcpIntents);
    └── CompletableFuture.runAsync(() -> McpToolExecutor.execute(), mcpBatchExecutor)
```

#### 阶段 4: 后处理

```java
// PostProcessor 链 (串行执行)
List<RetrievedChunk> processed = postProcessorChain.process(chunks, results, context);
    ├── DeduplicationPostProcessor.process()  // 去重
    └── RerankPostProcessor.process()         // 重排序
```

#### 阶段 5: Prompt 编排

```java
// 构建结构化消息
List<ChatMessage> messages = RAGPromptService.buildStructuredMessages(context, history, question, subQuestions);
    ├── ChatMessage.system(systemPrompt)      // 系统提示词
    ├── history                               // 对话历史 (含摘要)
    └── ChatMessage.user(evidence + question) // 证据 + 问题
```

**Prompt 场景选择**：
```java
switch (scene) {
    case KB_ONLY → RAG_ENTERPRISE_PROMPT_PATH;      // 仅 KB
    case MCP_ONLY → MCP_ONLY_PROMPT_PATH;           // 仅 MCP
    case MIXED → MCP_KB_MIXED_PROMPT_PATH;          // 混合
}
```

#### 阶段 6: LLM 流式调用

```java
// 路由式 LLM 调用
StreamCancellationHandle handle = RoutingLLMService.streamChat(request, callback);
    ├── ModelSelector.selectChatCandidates()      // 选择候选模型
    ├── ModelHealthStore.allowCall()               // 健康检查
    ├── ProbeStreamBridge (首包探测)               // 首包超时切换
    └── StreamCallback (SSE 推送)                  // 流式输出
```

### 3.3 设计模式

| 阶段 | 设计模式 | 核心组件 |
|------|---------|---------|
| 问题预处理 | **策略模式** | `QueryRewriteService`, `MultiQuestionRewriteService` |
| 意图识别 | **策略 + 注册表 + 工厂** | `IntentClassifier`, `IntentNodeRegistry`, `IntentTreeFactory` |
| 多路检索 | **策略 + 责任链 + 并行** | `SearchChannel`, `SearchResultPostProcessor` |
| MCP 工具 | **策略 + 注册表** | `McpToolExecutor`, `McpToolRegistry` |
| Prompt 编排 | **策略 + 模板方法** | `RAGPromptService`, `PromptTemplateLoader` |
| 模型路由 | **策略 + 外观 + 责任链** | `RoutingLLMService`, `ModelRoutingExecutor` |

---

## 四、并行执行点

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          并行执行架构                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   线程池配置 (ThreadPoolExecutorConfig)                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  ragContextExecutor ──── RAG 上下文组装                              │   │
│   │  mcpBatchExecutor ────── MCP 批量调用                                │   │
│   │  modelStreamExecutor ─── 模型流式输出                                │   │
│   │  multiSearchExecutor ─── 多路检索                                    │   │
│   │  intentClassifyExecutor ── 意图分类                                  │   │
│   │  memorySummaryExecutor ── 记忆摘要                                   │   │
│   │  chatEntryExecutor ────── 对话入口                                   │   │
│   │  ...                                                                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   并行执行点:                                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  1. 多路检索 (SearchChannel) ──── multiSearchExecutor 并行          │   │
│   │     ├── VectorGlobalSearchChannel                                   │   │
│   │     └── IntentDirectedSearchChannel                                 │   │
│   │                                                                     │   │
│   │  2. MCP 工具调用 ──── mcpBatchExecutor 并行                         │   │
│   │     ├── McpToolExecutor.execute()                                   │   │
│   │     └── McpToolExecutor.execute()                                   │   │
│   │                                                                     │   │
│   │  3. 子问题处理 ──── ragContextExecutor 并行                          │   │
│   │     ├── buildSubQuestionContext(subQuestion1)                       │   │
│   │     ├── buildSubQuestionContext(subQuestion2)                       │   │
│   │     └── buildSubQuestionContext(subQuestion3)                       │   │
│   │                                                                     │   │
│   │  4. 流式输出 ──── modelStreamExecutor 异步                          │   │
│   │     └── StreamAsyncExecutor.submit()                                │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   所有线程池使用 TTL 包装:                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  TtlExecutors.getTtlExecutor(executor)                              │   │
│   │  // 确保 UserContext 和 RagTraceContext 跨线程透传                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键入口类

| 入口类 | 模块 | 职责 |
|--------|------|------|
| `RAGChatController` | rag | 用户聊天入口 |
| `IngestionPipelineController` | ingestion | 文档入库入口 |
| `IngestionEngine` | ingestion | 入库 Pipeline 执行引擎 |
| `RetrievalEngine` | rag | 检索引擎 (协调多路检索 + MCP) |
| `MultiChannelRetrievalEngine` | rag | 多通道检索引擎 |
| `RAGPromptService` | rag | Prompt 编排服务 |
| `RoutingLLMService` | infra-ai | 模型路由服务 |
| `IntentClassifier` | rag | 意图分类器 |
| `ConversationMemoryService` | rag | 对话记忆服务 |

---

## 六、数据流转图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          数据流转图                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  文档入库数据流                                                      │   │
│   │                                                                     │   │
│   │  DocumentSource ──► rawBytes ──► rawText ──► enhancedText           │   │
│   │       │                                   │                         │   │
│   │       │                                   ▼                         │   │
│   │       │                             List<VectorChunk>               │   │
│   │       │                                   │                         │   │
│   │       │                                   ▼                         │   │
│   │       │                             VectorStore (Milvus/PG)         │   │
│   │       │                                                             │   │
│   │       └──► metadata ──► KnowledgeDocumentDO (PostgreSQL)            │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  查询处理数据流                                                      │   │
│   │                                                                     │   │
│   │  userQuestion ──► rewrittenQuestion ──► List<SubQuestionIntent>     │   │
│   │                                              │                      │   │
│   │                      ┌───────────────────────┼───────────────────┐  │   │
│   │                      ▼                       ▼                   │  │   │
│   │              List<RetrievedChunk>    Map<toolId, CallToolResult>  │  │   │
│   │                      │                       │                   │  │   │
│   │                      └───────────────────────┼───────────────────┘  │   │
│   │                                              ▼                      │   │
│   │                                      RetrievalContext               │   │
│   │                                     (kbContext + mcpContext)         │   │
│   │                                              │                      │   │
│   │                                              ▼                      │   │
│   │                                      List<ChatMessage>              │   │
│   │                                     (system + history + evidence)   │   │
│   │                                              │                      │   │
│   │                                              ▼                      │   │
│   │                                      LLM.streamChat()               │   │
│   │                                              │                      │   │
│   │                                              ▼                      │   │
│   │                                      SSE Stream → Frontend          │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 七、设计模式汇总

| 设计模式 | 应用场景 | 核心组件 |
|---------|---------|---------|
| **策略模式** | 文档解析、分块、检索通道、意图分类、MCP工具、对话记忆、问题重写 | `DocumentParser`, `ChunkingStrategy`, `SearchChannel`, `IntentClassifier`, `McpToolExecutor`, `ConversationMemoryService` |
| **注册表模式** | 自动发现并注册组件 | `DocumentParserSelector`, `ChunkingStrategyFactory`, `FetcherNode`, `IntentNodeRegistry`, `McpToolRegistry` |
| **工厂模式** | 创建策略实例 | `ChunkingStrategyFactory`, `IntentTreeFactory` |
| **模板方法** | 统一执行流程 | `IngestionNode`, `AbstractOpenAIStyle*` |
| **责任链** | Pipeline 执行、后处理链 | `IngestionEngine`, `SearchResultPostProcessor` |
| **外观模式** | 简化复杂子系统调用 | `RoutingLLMService`, `RetrievalEngine` |
| **状态机** | 文档状态流转 | `DocumentStatusHelper` |
| **观察者** | 流式事件通知 | `StreamCallback` |
| **装饰器** | 流式回调增强 | `ForwardingStreamCallback`, `ProbeStreamBridge` |
| **并行执行** | 多通道检索、MCP调用 | `CompletableFuture`, 线程池 |

---

## 八、扩展点

| 扩展点 | 接口 | 新增方式 |
|--------|------|---------|
| 文档解析器 | `DocumentParser` | 实现接口 + `@Component` |
| 分块策略 | `ChunkingStrategy` | 实现接口 + `@Component` |
| 文档获取器 | `DocumentFetcher` | 实现接口 + `@Component` |
| 检索通道 | `SearchChannel` | 实现接口 + `@Component` |
| 后处理器 | `SearchResultPostProcessor` | 实现接口 + `@Component` |
| 意图分类器 | `IntentClassifier` | 实现接口 + `@Component` |
| MCP 工具 | `McpToolExecutor` | 实现接口 + `@Component` |
| 对话记忆存储 | `ConversationMemoryStore` | 实现接口 + `@Component` |
| 入库节点 | `IngestionNode` | 实现接口 + `@Component` |
| 模型供应商 | `ChatClient` | 实现接口 + `@Component` |

**统一模式**：实现接口 + `@Component` 注解，Spring 自动发现并注册。
