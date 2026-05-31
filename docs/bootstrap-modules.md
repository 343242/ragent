# Ragent Bootstrap 业务层研究报告

## 一、整体架构概览

Bootstrap 模块是 Ragent 的业务逻辑层，包含 4 个核心子模块：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Bootstrap 模块                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         core (核心能力)                               │    │
│  │  ┌─────────────────┐  ┌─────────────────┐                           │    │
│  │  │    parser        │  │     chunk       │                           │    │
│  │  │  文档解析         │  │   文本分块       │                           │    │
│  │  └─────────────────┘  └─────────────────┘                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       ingestion (文档入库)                            │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │    │
│  │  │  engine   │ │   node   │ │ strategy │ │  domain  │ │ service  │  │    │
│  │  │ 执行引擎  │ │ 节点实现  │ │ 策略实现  │ │ 领域模型  │ │ 服务层   │  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       knowledge (知识库管理)                          │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │    │
│  │  │ controller│ │ service  │ │   dao    │ │ schedule │ │    mq    │  │    │
│  │  │ 控制器    │ │ 服务层   │ │ 数据访问  │ │ 定时任务  │ │ 消息队列 │  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                          rag (RAG 核心)                              │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │    │
│  │  │   core   │ │controller│ │ service  │ │   dao    │ │  config  │  │    │
│  │  │核心引擎  │ │ 控制器    │ │ 服务层   │ │ 数据访问  │ │ 配置     │  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、core 模块 - 核心能力

### 2.1 文档解析 (parser)

#### 2.1.1 设计模式：策略模式 + 注册表模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    文档解析策略模式                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │ DocumentParser   │                                          │
│   │ + getParserType()│                                          │
│   │ + parse()        │                                          │
│   │ + extractText()  │                                          │
│   │ + supports()     │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┬───────────────┐                            │
│   │               │               │                            │
│   ▼               ▼               ▼                            │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│ │  Tika    │  │ Markdown │  │  (扩展)   │                      │
│ │ Parser   │  │ Parser   │  │          │                      │
│ └──────────┘  └──────────┘  └──────────┘                      │
│                                                                 │
│   DocumentParserSelector (选择器/注册表)                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Map<String, DocumentParser> strategyMap                │   │
│   │  + select(parserType) → 按类型选择                       │   │
│   │  + selectByMimeType(mimeType) → 按 MIME 类型自动选择     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 策略接口
public interface DocumentParser {
    String getParserType();
    ParseResult parse(byte[] content, String mimeType, Map<String, Object> options);
    default boolean supports(String mimeType) { return true; }
}

// 策略选择器 (注册表)
@Component
public class DocumentParserSelector {
    private final Map<String, DocumentParser> strategyMap;

    public DocumentParserSelector(List<DocumentParser> parsers) {
        // Spring 自动注入所有 DocumentParser 实现
        this.strategyMap = parsers.stream()
            .collect(Collectors.toMap(DocumentParser::getParserType, Function.identity()));
    }

    // 按类型选择
    public DocumentParser select(String parserType) {
        return strategyMap.get(parserType);
    }

    // 按 MIME 类型自动选择
    public DocumentParser selectByMimeType(String mimeType) {
        return strategies.stream()
            .filter(parser -> parser.supports(mimeType))
            .findFirst()
            .orElseGet(() -> select(ParserType.TIKA.getType()));
    }
}
```

**优势**：
- 新增解析器只需实现接口 + `@Component`，自动注册
- 支持按类型或 MIME 类型动态选择
- 符合开闭原则

---

### 2.2 文本分块 (chunk)

#### 2.2.1 设计模式：策略模式 + 工厂模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    文本分块策略模式                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │ChunkingStrategy  │                                          │
│   │ + getType()      │                                          │
│   │ + chunk()        │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┬───────────────┐                            │
│   │               │               │                            │
│   ▼               ▼               ▼                            │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│ │FixedSize │  │Structure │  │  (扩展)   │                      │
│ │Chunker   │  │Aware     │  │          │                      │
│ │(固定大小) │  │(结构感知) │  │          │                      │
│ └──────────┘  └──────────┘  └──────────┘                      │
│                                                                 │
│   ChunkingStrategyFactory (工厂)                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Map<ChunkingMode, ChunkingStrategy> strategies         │   │
│   │  @PostConstruct init() → 自动注册                        │   │
│   │  + findStrategy(type) → 查找策略                         │   │
│   │  + requireStrategy(type) → 获取策略 (不存在则抛异常)      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 策略接口
public interface ChunkingStrategy {
    ChunkingMode getType();
    List<VectorChunk> chunk(String text, ChunkingOptions config);
}

// 策略工厂
@Component
public class ChunkingStrategyFactory {
    private final List<ChunkingStrategy> chunkingStrategies;
    private volatile Map<ChunkingMode, ChunkingStrategy> strategies = Map.of();

    @PostConstruct
    public void init() {
        Map<ChunkingMode, ChunkingStrategy> map = new EnumMap<>(ChunkingMode.class);
        chunkingStrategies.forEach(s -> {
            ChunkingStrategy old = map.put(s.getType(), s);
            if (old != null) {
                throw new IllegalStateException("Duplicate ChunkingStrategy for type: " + s.getType());
            }
        });
        this.strategies = Map.copyOf(map);
    }

    public ChunkingStrategy requireStrategy(ChunkingMode type) {
        return findStrategy(type)
            .orElseThrow(() -> new IllegalArgumentException("Unknown strategy: " + type));
    }
}
```

**分块策略**：

| 策略 | 类型 | 说明 |
|------|------|------|
| `FixedSizeTextChunker` | FIXED_SIZE | 按固定字符数切分，支持重叠 |
| `StructureAwareTextChunker` | STRUCTURE_AWARE | 按文档结构（标题、段落）切分 |

---

## 三、ingestion 模块 - 文档入库

### 3.1 核心设计：Pipeline + 节点编排

#### 3.1.1 设计模式：模板方法 + 策略 + 责任链

```
┌─────────────────────────────────────────────────────────────────┐
│                    文档入库 Pipeline                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   IngestionEngine (执行引擎)                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  1. 验证 Pipeline 配置 (环检测)                          │   │
│   │  2. 找到起始节点                                         │   │
│   │  3. 链式执行节点                                         │   │
│   │  4. 条件评估 + 日志记录                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │ IngestionNode    │                                          │
│   │ + getNodeType()  │                                          │
│   │ + execute()      │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────┴───────┬───────────────┬───────────────┐            │
│   │               │               │               │            │
│   ▼               ▼               ▼               ▼            │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│ │FetcherNode│  │ParserNode│  │ChunkerNode│  │IndexerNode│       │
│ │ (获取)    │  │ (解析)    │  │ (分块)    │  │ (索引)    │       │
│ └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│                                                                 │
│   Pipeline 配置 (数据库存储)                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  IngestionPipelineDO                                    │   │
│   │  ├── IngestionPipelineNodeDO[] (节点列表)                │   │
│   │  │   ├── nodeId: "fetcher-1"                            │   │
│   │  │   ├── nodeType: "FETCHER"                            │   │
│   │  │   ├── nextNodeId: "parser-1"                         │   │
│   │  │   ├── condition: {...}                               │   │
│   │  │   └── settings: {...}                                │   │
│   │  └── ...                                                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 节点类型

| 节点类型 | 职责 | 使用的策略/组件 |
|---------|------|----------------|
| `FetcherNode` | 从不同源获取文档 | `DocumentFetcher` 策略 |
| `ParserNode` | 解析文档为文本 | `DocumentParserSelector` |
| `ChunkerNode` | 文本分块 + 向量化 | `ChunkingStrategyFactory` |
| `EnhancerNode` | LLM 增强文本 | LLM 服务 |
| `EnricherNode` | 元数据丰富 | LLM 服务 |
| `IndexerNode` | 写入向量数据库 | 向量存储服务 |

### 3.3 文档获取策略 (FetcherNode 内部)

```
┌─────────────────────────────────────────────────────────────────┐
│                    文档获取策略模式                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │DocumentFetcher   │                                          │
│   │ + supportedType()│                                          │
│   │ + fetch()        │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────┴───────┬───────────────┬───────────────┐            │
│   │               │               │               │            │
│   ▼               ▼               ▼               ▼            │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│ │LocalFile │  │  S3      │  │ HttpUrl  │  │ Feishu   │        │
│ │Fetcher   │  │ Fetcher  │  │ Fetcher  │  │ Fetcher  │        │
│ │(本地文件) │  │(对象存储) │  │(HTTP URL)│  │(飞书文档) │        │
│ └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│                                                                 │
│   FetcherNode (注册表)                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Map<SourceType, DocumentFetcher> fetchers              │   │
│   │  // Spring 自动注入，按 SourceType 注册                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 策略接口
public interface DocumentFetcher {
    SourceType supportedType();
    FetchResult fetch(DocumentSource source);
}

// 注册表 + 策略选择
@Component
public class FetcherNode implements IngestionNode {
    private final Map<SourceType, DocumentFetcher> fetchers;

    public FetcherNode(List<DocumentFetcher> fetchers) {
        // Spring 自动注入，按 SourceType 注册
        this.fetchers = fetchers.stream()
            .collect(Collectors.toMap(DocumentFetcher::supportedType, Function.identity()));
    }

    @Override
    public NodeResult execute(IngestionContext context, NodeConfig config) {
        SourceType sourceType = context.getSource().getType();
        DocumentFetcher fetcher = fetchers.get(sourceType);  // 策略选择
        FetchResult result = fetcher.fetch(context.getSource());  // 执行策略
        // ...
    }
}
```

### 3.4 Pipeline 执行引擎

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pipeline 执行流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   IngestionEngine.execute()                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │  1. validatePipeline() ──── 环检测                      │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  2. findStartNode() ───── 找起始节点                    │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  3. executeChain() ────── 链式执行                      │   │
│   │      │                                                  │   │
│   │      ├──► executeNode(FetcherNode)                      │   │
│   │      │      │                                           │   │
│   │      │      ▼                                           │   │
│   │      ├──► executeNode(ParserNode)                       │   │
│   │      │      │                                           │   │
│   │      │      ▼                                           │   │
│   │      ├──► executeNode(ChunkerNode)                      │   │
│   │      │      │                                           │   │
│   │      │      ▼                                           │   │
│   │      └──► executeNode(IndexerNode)                      │   │
│   │                                                         │   │
│   │  每个节点执行:                                            │   │
│   │  - 条件评估 (ConditionEvaluator)                         │   │
│   │  - 执行节点逻辑                                          │   │
│   │  - 记录日志 (NodeLog)                                    │   │
│   │  - 判断是否继续                                          │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、knowledge 模块 - 知识库管理

### 4.1 模块职责

| 子模块 | 职责 |
|--------|------|
| `controller` | REST API 接口 |
| `service` | 业务逻辑 (知识库、文档、分块、调度) |
| `dao` | 数据访问 (MyBatis-Plus) |
| `schedule` | 定时任务 (文档状态扫描) |
| `mq` | 消息队列 (分块任务) |
| `filter` | 限流过滤器 |
| `config` | 配置 (信号量、调度) |

### 4.2 核心实体

```
┌─────────────────────────────────────────────────────────────────┐
│                    知识库实体关系                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   KnowledgeBaseDO (知识库)                                      │
│       │                                                        │
│       │ 1:N                                                    │
│       ▼                                                        │
│   KnowledgeDocumentDO (文档)                                    │
│       │                                                        │
│       │ 1:N                                                    │
│       ▼                                                        │
│   KnowledgeChunkDO (分块)                                       │
│                                                                 │
│   KnowledgeDocumentScheduleDO (调度任务)                        │
│       │                                                        │
│       │ 1:N                                                    │
│       ▼                                                        │
│   KnowledgeDocumentScheduleExecDO (调度执行记录)                │
│                                                                 │
│   KnowledgeDocumentChunkLogDO (分块日志)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 定时任务调度

```
┌─────────────────────────────────────────────────────────────────┐
│                    定时任务调度架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   KnowledgeDocumentScheduleJob                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  @Scheduled 定时执行                                     │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  ScheduleLockManager ──── 分布式锁 (防重复执行)           │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  DocumentStatusHelper ─── 状态机 (文档状态流转)           │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  ScheduleStateManager ─── 调度状态管理                   │   │
│   │      │                                                  │   │
│   │      ▼                                                  │   │
│   │  触发 IngestionPipeline 执行                             │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 设计模式

| 设计模式 | 应用场景 |
|---------|---------|
| **状态机** | 文档状态流转 (PENDING → PROCESSING → COMPLETED / FAILED) |
| **分布式锁** | ScheduleLockManager 防止重复调度 |
| **观察者** | MQ 消费者触发分块任务 |

---

## 五、rag 模块 - RAG 核心

### 5.1 模块架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         rag 模块架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      core (核心引擎)                      │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │ retrieve │ │  intent  │ │   mcp    │ │  memory  │   │    │
│  │  │ 多路检索  │ │ 意图识别  │ │ MCP工具  │ │ 对话记忆  │   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                │    │
│  │  │  prompt  │ │  rewrite │ │          │                │    │
│  │  │Prompt编排│ │ 问题重写  │ │          │                │    │
│  │  └──────────┘ └──────────┘ └──────────┘                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      外围支撑                             │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │    │
│  │  │controller│ │ service  │ │   dao    │ │  config  │   │    │
│  │  │ REST API │ │ 服务层   │ │ 数据访问  │ │ 配置     │   │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 多路检索引擎

#### 5.2.1 设计模式：策略模式 + 责任链 + 并行执行

```
┌─────────────────────────────────────────────────────────────────┐
│                    多路检索架构                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MultiChannelRetrievalEngine                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │  并行执行多个 SearchChannel:                              │   │
│   │  ┌─────────────────────────────────────────────────┐     │   │
│   │  │  CompletableFuture<ChannelA> ─┐                  │     │   │
│   │  │  CompletableFuture<ChannelB> ─┼─► 合并结果       │     │   │
│   │  │  CompletableFuture<ChannelC> ─┘                  │     │   │
│   │  └─────────────────────────────────────────────────┘     │   │
│   │                                                         │   │
│   │  串行执行 PostProcessor 链:                               │   │
│   │  ┌─────────────────────────────────────────────────┐     │   │
│   │  │  Deduplication ──► Rerank ──► ...                │     │   │
│   │  └─────────────────────────────────────────────────┘     │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   <<interface>>                 实现类                           │
│   ┌─────────────┐                                              │
│   │SearchChannel │◄────────┬── VectorGlobalSearchChannel       │
│   │ + getName()  │          └── IntentDirectedSearchChannel     │
│   │ + search()   │                                              │
│   │ + isEnabled()│                                              │
│   └─────────────┘                                              │
│                                                                 │
│   <<interface>>                 实现类                           │
│   ┌─────────────────────┐                                      │
│   │SearchResultPostProc  │◄────────┬── DeduplicationPostProc    │
│   │ + getName()          │          └── RerankPostProcessor     │
│   │ + process()          │                                      │
│   │ + getOrder()         │                                      │
│   └─────────────────────┘                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**核心代码**：

```java
// 检索通道接口 (策略)
public interface SearchChannel {
    String getName();
    int getPriority();
    boolean isEnabled(SearchContext context);
    SearchChannelResult search(SearchContext context);
    SearchChannelType getType();
}

// 后处理器接口 (责任链)
public interface SearchResultPostProcessor {
    String getName();
    int getOrder();  // 执行顺序
    boolean isEnabled(SearchContext context);
    List<RetrievedChunk> process(List<RetrievedChunk> chunks,
                                  List<SearchChannelResult> results,
                                  SearchContext context);
}
```

### 5.3 意图识别系统

#### 5.3.1 设计模式：策略模式 + 注册表模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    意图识别系统                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │IntentClassifier  │                                          │
│   │ + classifyTargets│                                          │
│   │ + topKAboveThreshold│                                       │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┐                                            │
│   │               │                                            │
│   ▼               ▼                                            │
│ ┌──────────┐  ┌──────────┐                                     │
│ │ Default  │  │ (并行)    │                                     │
│ │ Intent   │  │ Intent   │                                     │
│ │Classifier│  │Classifier│                                     │
│ │ (串行)   │  │          │                                     │
│ └──────────┘  └──────────┘                                     │
│                                                                 │
│   IntentNodeRegistry (注册表)                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Map<String, IntentNode> nodes                          │   │
│   │  // 从数据库加载意图树，按 ID 注册                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   IntentTreeFactory (工厂)                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + buildTree() → 构建意图树                              │   │
│   │  // 从数据库加载，构建树形结构                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   IntentResolver (解析器)                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + resolve(question) → 解析意图                          │   │
│   │  // 协调 Classifier 和 Registry                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**意图节点结构**：

```java
public class IntentNode {
    private String id;
    private String parentId;
    private String name;
    private String description;
    private List<String> sampleQuestions;
    private String promptTemplate;
    private String mcpToolId;        // MCP 工具 ID
    private Integer topK;            // 检索 TopK
    private Double minScore;         // 最低分数阈值
    private List<IntentNode> children;  // 子节点
}
```

### 5.4 MCP 工具集成

#### 5.4.1 设计模式：策略模式 + 注册表模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 工具集成                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │McpToolExecutor   │                                          │
│   │ + getToolDefinition()│                                      │
│   │ + execute()      │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┐                                            │
│   │               │                                            │
│   ▼               ▼                                            │
│ ┌──────────┐  ┌──────────┐                                     │
│ │McpClient │  │ (自定义)  │                                     │
│ │ToolExec  │  │ Executor │                                     │
│ │(MCP协议) │  │          │                                     │
│ └──────────┘  └──────────┘                                     │
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────┐                                          │
│   │McpToolRegistry   │                                          │
│   │ + register()     │                                          │
│   │ + unregister()   │                                          │
│   │ + getExecutor()  │                                          │
│   │ + listAllTools() │                                          │
│   └─────────────────┘                                          │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┐                                            │
│   │ DefaultMcpTool │                                            │
│   │ Registry       │                                            │
│   │ (自动发现)      │                                            │
│   └───────────────┘                                            │
│                                                                 │
│   McpParameterExtractor (参数提取)                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + extractParameters(question, tool, prompt)             │   │
│   │  // LLM 从用户问题中提取工具参数                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.5 对话记忆管理

#### 5.5.1 设计模式：策略模式

```
┌─────────────────────────────────────────────────────────────────┐
│                    对话记忆管理                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────────┐                                      │
│   │ConversationMemory    │                                      │
│   │Service               │                                      │
│   │ + load()             │                                      │
│   │ + append()           │                                      │
│   │ + loadAndAppend()    │                                      │
│   └─────────────────────┘                                      │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┐                                            │
│   │               │                                            │
│   ▼               ▼                                            │
│ ┌──────────┐  ┌──────────┐                                     │
│ │ Default  │  │  (扩展)   │                                     │
│ │Conversa- │  │          │                                     │
│ │tionMemory│  │          │                                     │
│ │Service   │  │          │                                     │
│ └──────────┘  └──────────┘                                     │
│                                                                 │
│   <<interface>>                                                 │
│   ┌─────────────────────┐                                      │
│   │ConversationMemory    │                                      │
│   │Store                 │                                      │
│   │ + save()             │                                      │
│   │ + load()             │                                      │
│   └─────────────────────┘                                      │
│           ▲                                                    │
│           │ implements                                         │
│           │                                                    │
│   ┌───────────────┐                                            │
│   │               │                                            │
│   ▼               ▼                                            │
│ ┌──────────┐  ┌──────────┐                                     │
│ │   JDBC   │  │  (Redis) │                                     │
│ │ Memory   │  │ Memory   │                                     │
│ │ Store    │  │ Store    │                                     │
│ └──────────┘  └──────────┘                                     │
│                                                                 │
│   ConversationMemorySummaryService                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + summarize() → 对话摘要 (压缩长对话)                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.6 Prompt 编排

#### 5.6.1 设计模式：策略模式 + 模板方法

```
┌─────────────────────────────────────────────────────────────────┐
│                    Prompt 编排                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   RAGPromptService                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + buildSystemPrompt(context) → 系统提示词               │   │
│   │  + buildStructuredMessages(...) → 完整消息列表           │   │
│   │                                                         │   │
│   │  场景选择:                                                │   │
│   │  ├── KB_ONLY → RAG_ENTERPRISE_PROMPT_PATH               │   │
│   │  ├── MCP_ONLY → MCP_ONLY_PROMPT_PATH                    │   │
│   │  └── MIXED → MCP_KB_MIXED_PROMPT_PATH                   │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   PromptTemplateLoader                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + load(path) → 加载模板                                 │   │
│   │  + renderSection(path, section, slots) → 渲染片段        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ContextFormatter                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + formatKbContext(...) → 格式化 KB 上下文                │   │
│   │  + formatMcpContext(...) → 格式化 MCP 上下文              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.7 问题重写

```
┌─────────────────────────────────────────────────────────────────┐
│                    问题重写                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   QueryRewriteService                                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + rewrite(question, history) → 重写后的问题             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   MultiQuestionRewriteService                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + rewriteToSubQuestions(question) → 子问题列表          │   │
│   │  // 复杂问题拆分为多个子问题                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   QueryTermMappingService                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  + getMappings(query) → 术语映射                         │   │
│   │  // 专业术语 → 通俗解释                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、设计模式总结

### 6.1 模式统计

| 设计模式 | 应用数量 | 主要应用 |
|---------|---------|---------|
| **策略模式** | 10+ | DocumentParser, ChunkingStrategy, SearchChannel, IntentClassifier, McpToolExecutor, DocumentFetcher, ConversationMemoryService/Store, SearchResultPostProcessor |
| **注册表模式** | 6 | DocumentParserSelector, ChunkingStrategyFactory, FetcherNode, IntentNodeRegistry, DefaultMcpToolRegistry, SearchChannel 注册 |
| **工厂模式** | 2 | ChunkingStrategyFactory, IntentTreeFactory |
| **模板方法** | 2 | IngestionNode 基类, AbstractOpenAIStyle* |
| **责任链** | 2 | IngestionEngine 链式执行, PostProcessor 链 |
| **状态机** | 1 | 文档状态流转 |
| **观察者** | 2 | MQ 消费者, StreamCallback |
| **并行执行** | 2 | MultiChannelRetrievalEngine, RetrievalEngine |

### 6.2 设计模式详细清单

| 模块 | 设计模式 | 应用场景 | 核心类 |
|------|---------|---------|--------|
| **core/parser** | 策略 + 注册表 | 文档解析器 | `DocumentParser`, `DocumentParserSelector` |
| **core/chunk** | 策略 + 工厂 | 文本分块 | `ChunkingStrategy`, `ChunkingStrategyFactory` |
| **ingestion/node** | 模板方法 + 策略 | 入库节点 | `IngestionNode`, `FetcherNode` |
| **ingestion/engine** | 责任链 | Pipeline 执行 | `IngestionEngine` |
| **ingestion/strategy** | 策略 + 注册表 | 文档获取 | `DocumentFetcher`, `FetcherNode` |
| **knowledge/schedule** | 状态机 + 分布式锁 | 定时调度 | `DocumentStatusHelper`, `ScheduleLockManager` |
| **rag/core/retrieve** | 策略 + 责任链 + 并行 | 多路检索 | `SearchChannel`, `SearchResultPostProcessor` |
| **rag/core/intent** | 策略 + 注册表 + 工厂 | 意图识别 | `IntentClassifier`, `IntentNodeRegistry`, `IntentTreeFactory` |
| **rag/core/mcp** | 策略 + 注册表 | MCP 工具 | `McpToolExecutor`, `McpToolRegistry` |
| **rag/core/memory** | 策略 | 对话记忆 | `ConversationMemoryService`, `ConversationMemoryStore` |
| **rag/core/prompt** | 策略 + 模板方法 | Prompt 编排 | `RAGPromptService`, `PromptTemplateLoader` |
| **rag/core/rewrite** | 策略 | 问题重写 | `QueryRewriteService`, `MultiQuestionRewriteService` |

---

## 七、源文件索引

### core 模块 (17 文件)

| 子模块 | 文件 | 职责 |
|--------|------|------|
| parser | `DocumentParser.java` | 文档解析器接口 |
| parser | `DocumentParserSelector.java` | 解析器选择器 (注册表) |
| parser | `TikaDocumentParser.java` | Tika 解析器实现 |
| parser | `MarkdownDocumentParser.java` | Markdown 解析器实现 |
| parser | `ParserType.java` | 解析器类型枚举 |
| parser | `ParseResult.java` | 解析结果记录 |
| parser | `TextCleanupUtil.java` | 文本清理工具 |
| chunk | `ChunkingStrategy.java` | 分块策略接口 |
| chunk | `ChunkingStrategyFactory.java` | 分块策略工厂 |
| chunk | `FixedSizeTextChunker.java` | 固定大小分块器 |
| chunk | `StructureAwareTextChunker.java` | 结构感知分块器 |
| chunk | `ChunkingMode.java` | 分块模式枚举 |
| chunk | `ChunkingOptions.java` | 分块配置 |
| chunk | `FixedSizeOptions.java` | 固定大小配置 |
| chunk | `TextBoundaryOptions.java` | 文本边界配置 |
| chunk | `VectorChunk.java` | 向量分块记录 |
| chunk | `ChunkEmbeddingService.java` | 分块嵌入服务 |

### ingestion 模块 (64 文件)

| 子模块 | 文件 | 职责 |
|--------|------|------|
| engine | `IngestionEngine.java` | Pipeline 执行引擎 |
| engine | `ConditionEvaluator.java` | 条件评估器 |
| engine | `NodeOutputExtractor.java` | 节点输出提取器 |
| node | `IngestionNode.java` | 节点接口 (模板方法) |
| node | `FetcherNode.java` | 文档获取节点 |
| node | `ParserNode.java` | 文档解析节点 |
| node | `ChunkerNode.java` | 文本分块节点 |
| node | `EnhancerNode.java` | LLM 增强节点 |
| node | `EnricherNode.java` | 元数据丰富节点 |
| node | `IndexerNode.java` | 向量索引节点 |
| strategy/fetcher | `DocumentFetcher.java` | 文档获取接口 (策略) |
| strategy/fetcher | `LocalFileFetcher.java` | 本地文件获取器 |
| strategy/fetcher | `S3Fetcher.java` | S3 获取器 |
| strategy/fetcher | `HttpUrlFetcher.java` | HTTP URL 获取器 |
| strategy/fetcher | `FeishuFetcher.java` | 飞书文档获取器 |
| domain | `IngestionContext.java` | 入库上下文 |
| domain | `PipelineDefinition.java` | Pipeline 定义 |
| domain | `NodeConfig.java` | 节点配置 |
| domain | `NodeResult.java` | 节点结果 |
| service | `IngestionPipelineService.java` | Pipeline 服务 |
| service | `IngestionTaskService.java` | 任务服务 |
| controller | `IngestionPipelineController.java` | Pipeline 控制器 |
| controller | `IngestionTaskController.java` | 任务控制器 |

### knowledge 模块 (61 文件)

| 子模块 | 文件 | 职责 |
|--------|------|------|
| service | `KnowledgeBaseService.java` | 知识库服务 |
| service | `KnowledgeDocumentService.java` | 文档服务 |
| service | `KnowledgeChunkService.java` | 分块服务 |
| service | `KnowledgeDocumentScheduleService.java` | 调度服务 |
| schedule | `KnowledgeDocumentScheduleJob.java` | 定时任务 |
| schedule | `ScheduleLockManager.java` | 分布式锁管理 |
| schedule | `DocumentStatusHelper.java` | 状态机辅助 |
| schedule | `ScheduleStateManager.java` | 调度状态管理 |
| mq | `KnowledgeDocumentChunkConsumer.java` | MQ 消费者 |
| dao | `KnowledgeBaseMapper.java` | 知识库 Mapper |
| dao | `KnowledgeDocumentMapper.java` | 文档 Mapper |
| dao | `KnowledgeChunkMapper.java` | 分块 Mapper |
| controller | `KnowledgeBaseController.java` | 知识库控制器 |
| controller | `KnowledgeDocumentController.java` | 文档控制器 |
| controller | `KnowledgeChunkController.java` | 分块控制器 |

### rag 模块 (100+ 文件)

| 子模块 | 文件 | 职责 |
|--------|------|------|
| core/retrieve | `RetrievalEngine.java` | 检索引擎 |
| core/retrieve | `MultiChannelRetrievalEngine.java` | 多通道检索引擎 |
| core/retrieve/channel | `SearchChannel.java` | 检索通道接口 (策略) |
| core/retrieve/channel | `VectorGlobalSearchChannel.java` | 向量全局检索 |
| core/retrieve/channel | `IntentDirectedSearchChannel.java` | 意图定向检索 |
| core/retrieve/postprocessor | `SearchResultPostProcessor.java` | 后处理器接口 (责任链) |
| core/retrieve/postprocessor | `DeduplicationPostProcessor.java` | 去重处理器 |
| core/retrieve/postprocessor | `RerankPostProcessor.java` | 重排处理器 |
| core/intent | `IntentClassifier.java` | 意图分类器接口 (策略) |
| core/intent | `DefaultIntentClassifier.java` | 默认意图分类器 |
| core/intent | `IntentNodeRegistry.java` | 意图节点注册表 |
| core/intent | `IntentTreeFactory.java` | 意图树工厂 |
| core/intent | `IntentResolver.java` | 意图解析器 |
| core/mcp | `McpToolExecutor.java` | MCP 工具执行器接口 (策略) |
| core/mcp | `McpToolRegistry.java` | MCP 工具注册表接口 |
| core/mcp | `DefaultMcpToolRegistry.java` | 默认 MCP 工具注册表 |
| core/mcp | `McpClientToolExecutor.java` | MCP 客户端工具执行器 |
| core/memory | `ConversationMemoryService.java` | 对话记忆服务接口 (策略) |
| core/memory | `ConversationMemoryStore.java` | 对话记忆存储接口 (策略) |
| core/memory | `DefaultConversationMemoryService.java` | 默认对话记忆服务 |
| core/memory | `JdbcConversationMemoryStore.java` | JDBC 记忆存储 |
| core/prompt | `RAGPromptService.java` | Prompt 编排服务 |
| core/prompt | `PromptTemplateLoader.java` | 模板加载器 |
| core/prompt | `ContextFormatter.java` | 上下文格式化器 |
| core/rewrite | `QueryRewriteService.java` | 问题重写服务 |
| core/rewrite | `MultiQuestionRewriteService.java` | 多问题重写服务 |
| controller | `RAGChatController.java` | RAG 聊天控制器 |
| controller | `IntentTreeController.java` | 意图树控制器 |
| controller | `ConversationController.java` | 对话控制器 |

---

## 八、关键设计亮点

### 8.1 统一的策略注册机制

所有策略类都遵循相同的模式：
1. 定义接口 (`DocumentParser`, `ChunkingStrategy`, `SearchChannel`, etc.)
2. 提供实现类 + `@Component` 注解
3. 注册表自动发现并建立索引

```java
// 统一模式
@Component
public class XxxSelector {
    private final Map<String, XxxStrategy> strategyMap;

    public XxxSelector(List<XxxStrategy> strategies) {
        this.strategyMap = strategies.stream()
            .collect(Collectors.toMap(XxxStrategy::getType, Function.identity()));
    }
}
```

### 8.2 Pipeline 可编排

- 节点配置存储在数据库
- 支持条件执行 (`ConditionEvaluator`)
- 支持链式传递 (`nextNodeId`)
- 每个节点有独立日志 (`NodeLog`)

### 8.3 多路并行检索

- `SearchChannel` 并行执行
- `SearchResultPostProcessor` 串行处理
- 支持动态启用/禁用通道

### 8.4 意图驱动的检索

- 树形意图结构 (`IntentNode`)
- LLM 意图分类 (`IntentClassifier`)
- 按意图路由到不同检索通道
- 支持 MCP 工具调用

---

## 九、模块依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    模块依赖关系                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   rag ──────────────────────────────┐                           │
│     │                               │                           │
│     │ 依赖                          │ 依赖                       │
│     ▼                               ▼                           │
│   core ◄──────────────────────── ingestion                      │
│     │                               │                           │
│     │ 依赖                          │ 依赖                       │
│     ▼                               ▼                           │
│   knowledge ◄──────────────────── framework                     │
│     │                               │                           │
│     │ 依赖                          │ 依赖                       │
│     ▼                               ▼                           │
│   infra-ai ◄──────────────────── framework                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| 模块 | 依赖 |
|------|------|
| `rag` | `core`, `framework`, `infra-ai` |
| `ingestion` | `core`, `framework`, `infra-ai` |
| `knowledge` | `ingestion`, `framework` |
| `core` | `framework` |
