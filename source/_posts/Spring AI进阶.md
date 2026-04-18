---
title: Spring AI进阶开发
date: 2025-06-20
updated : 2025-06-20
categories:
- Spring AI
tags: 
  - Spring AI
  - 实战
description: Spring AI 高级特性：Function Calling、RAG、Advisor 链与多模型编排。
series: Spring AI 实战
series_order: 2
---

## Function Calling

Function Calling（函数调用）是 Spring AI 中连接 LLM 与外部世界的关键能力。通过注册 Java 方法作为"工具"，LLM 在对话过程中可以自主决定是否调用这些工具来获取实时数据或执行操作。

### 自定义工具函数

#### 1. 定义工具函数

使用 `@Bean` + `@Description` 注解注册工具函数：

```java
@Configuration
public class ToolConfig {

    @Bean
    @Description("查询指定城市的天气信息")
    public Function<WeatherRequest, WeatherResponse> weatherFunction() {
        return request -> weatherService.getWeather(request.city());
    }

    public record WeatherRequest(String city, String unit) {}
    public record WeatherResponse(
        double temperature,
        String unit,
        String description,
        String city
    ) {}
}
```

#### 2. 在 ChatClient 中引用工具

```java
@RestController
@RequestMapping("/api/chat")
public class ToolCallingController {

    private final ChatClient chatClient;

    public ToolCallingController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .functions("weatherFunction")  // 引用注册的 Bean 名称
            .call()
            .content();
    }
}
```

当用户输入"北京今天天气怎么样？"时，LLM 会自动识别需要调用天气查询函数，提取参数 `city=北京`，执行函数后结合结果生成自然语言回答。

#### 3. 工具函数的完整执行流程

```
用户请求 → ChatClient → LLM 判断需要调用工具 → 返回工具调用请求
    ↓
Spring AI 自动执行工具函数 → 将结果附加到对话上下文
    ↓
LLM 结合工具结果生成最终回答 → 返回给用户
```

### 多工具编排

一个对话中可以注册多个工具，LLM 会根据需要选择调用：

```java
@Bean
@Description("查询股票实时价格")
public Function<StockRequest, StockResponse> stockPriceFunction() {
    return request -> stockService.getPrice(request.symbol());
}

@Bean
@Description("查询天气预报")
public Function<WeatherRequest, WeatherResponse> weatherFunction() {
    return request -> weatherService.getWeather(request.city());
}

// 使用时指定多个工具
String response = chatClient.prompt()
    .user("苹果公司的股价和库比蒂诺今天的天气")
    .functions("stockPriceFunction", "weatherFunction")
    .call()
    .content();
```

### 工具调用回调

可以通过 Advisor 监控工具调用的过程：

```java
chatClient.prompt()
    .user("查询北京的天气")
    .functions("weatherFunction")
    .advisors(advisor -> advisor
        .observeToolCall((toolName, args, result) -> {
            log.info("工具调用: {} 参数: {} 结果: {}",
                toolName, args, result);
        }))
    .call()
    .content();
```

## RAG 集成

RAG（Retrieval-Augmented Generation，检索增强生成）是解决 LLM 知识时效性和专业领域覆盖不足的核心方案。Spring AI 提供了完整的 RAG 管道支持。

### RAG 的核心流程

```
用户提问 → 文本 Embedding → 向量检索 → 获取相关文档
    ↓
将文档作为上下文注入 Prompt → LLM 基于上下文生成回答
```

### DocumentReader：文档加载

DocumentReader 负责从不同数据源读取原始文档：

```java
// 读取 PDF 文档
PagePdfDocumentReader pdfReader = new PagePdfDocumentReader(
    new FileSystemResource("knowledge-base.pdf"),
    PdfDocumentReaderConfig.builder()
        .withPageExtractedTextFormatter(
            new ExtractedTextFormatter.Builder().build())
        .withPagesPerDocument(1)
        .build()
);

List<Document> documents = pdfReader.get();

// 读取文本文件
TextReader textReader = new TextReader(
    new FileSystemResource("knowledge-base.txt")
);
```

### DocumentTransformer：文档处理

文档通常需要分块（Chunking）后才能有效进行向量化检索：

```java
// 按 Token 数量分块
TokenTextSplitter splitter = new TokenTextSplitter(
    800,   // 每块目标大小
    200,   // 重叠大小
    5,     // 最大块数
    10000, // 最小块大小
    true   // 保留分隔符
);

List<Document> chunks = splitter.apply(documents);

// 添加元数据
DocumentTransformer metadataEnricher = new ContentFormatTransformer();
List<Document> enriched = metadataEnricher.apply(chunks);
```

### VectorStore：向量存储与检索

VectorStore 是 RAG 的核心组件，负责文档的向量化和相似度检索：

```java
// 存入文档
vectorStore.add(chunks);

// 相似度检索
List<Document> results = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("Spring AI 如何实现 RAG")
        .topK(5)
        .similarityThreshold(0.7)
        .build()
);
```

### QuestionAnswerAdvisor：开箱即用的 RAG

Spring AI 提供了 QuestionAnswerAdvisor，一行代码即可实现 RAG：

```java
// 首先添加 Maven 依赖
// spring-ai-advisors-vector-store

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder()
                .topK(5)
                .similarityThreshold(0.7)
                .build())
            .build()
    )
    .build();

// 自动检索相关文档并注入上下文
String answer = chatClient.prompt()
    .user("公司的退款政策是什么？")
    .call()
    .content();
```

#### 自定义 RAG Prompt 模板

```java
PromptTemplate customTemplate = new PromptTemplate("""
    基于以下参考资料回答用户的问题。
    如果参考资料中没有相关信息，请回答"我无法找到相关信息"。

    参考资料：
    {question_answer_context}

    用户问题：{query}
    """);

QuestionAnswerAdvisor ragAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .promptTemplate(customTemplate)
    .searchRequest(SearchRequest.builder().topK(3).build())
    .build();
```

#### 动态过滤条件

根据用户上下文动态构建检索过滤条件：

```java
String answer = chatClient.prompt()
    .user("企业版的价格是多少？")
    .advisors(advisor -> advisor
        .param(QuestionAnswerAdvisor.FILTER_EXPRESSION,
            "category == 'pricing'"))
    .call()
    .content();
```

#### 获取检索到的原始文档

```java
ChatClientResponse response = chatClient.prompt()
    .user("请解释退款政策")
    .call()
    .chatClientResponse();

List<Document> retrievedDocs = (List<Document>) response
    .chatResponse()
    .getMetadata()
    .get(QuestionAnswerAdvisor.RETRIEVED_DOCUMENTS);

retrievedDocs.forEach(doc ->
    System.out.println("来源: " + doc.getMetadata().get("source")));
```

## 向量存储

Spring AI 支持多种向量数据库，通过统一的 VectorStore 接口抽象。

### PgVectorStore

基于 PostgreSQL + pgvector 扩展，适合已有 PostgreSQL 基础设施的场景：

```java
@Bean
public VectorStore pgVectorStore(
        JdbcTemplate jdbcTemplate, EmbeddingModel embeddingModel) {
    return PgVectorStore.builder(jdbcTemplate, embeddingModel)
        .dimensions(1536)
        .distanceType(DistanceType.COSINE)
        .removeExistingVectorStoreTable(false)
        .initializeSchema(true)
        .build();
}
```

配置：

```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536
        distance-type: cosine_distance
        index-type: hnsw
```

### MilvusVectorStore

Milvus 是专业的向量数据库，适合大规模生产环境：

```java
@Bean
public VectorStore milvusVectorStore(MilvusServiceClient milvusClient,
        EmbeddingModel embeddingModel) {
    return MilvusVectorStore.builder(milvusClient, embeddingModel)
        .collectionName("spring_ai_docs")
        .databaseName("default")
        .embeddingDimension(1536)
        .metricType(MetricType.COSINE)
        .indexType(IndexType.IVF_FLAT)
        .initializeSchema(true)
        .build();
}
```

### SimpleVectorStore

基于内存的简单实现，适合开发和测试：

```java
@Bean
public VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return SimpleVectorStore.builder(embeddingModel).build();
}
```

SimpleVectorStore 支持将数据持久化到本地文件：

```java
SimpleVectorStore store = SimpleVectorStore.builder(embeddingModel).build();
store.add(documents);
store.save(new File("vector-store.json"));

// 加载已有数据
store.load(new File("vector-store.json"));
```

### ChromaVectorStore

Chroma 是轻量级的开源向量数据库：

```java
@Bean
public VectorStore chromaVectorStore(EmbeddingModel embeddingModel,
        ChromaApi chromaApi) {
    return ChromaVectorStore.builder(chromaApi, embeddingModel)
        .collectionName("my_collection")
        .initializeSchema(true)
        .build();
}
```

## Advisor 链

Advisor 是 Spring AI 的请求/响应拦截机制，类似于 Servlet Filter 或 Spring Interceptor，可以在不修改业务代码的情况下增强 ChatClient 的能力。

### Advisor 的工作原理

```
用户请求 → Advisor1.before → Advisor2.before → LLM 调用
    ↓
LLM 响应 → Advisor2.after → Advisor1.after → 返回结果
```

### 内置 Advisor

#### MessageChatMemoryAdvisor

自动管理对话历史消息：

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory)
            .maxSize(20)
            .build()
    )
    .build();
```

#### QuestionAnswerAdvisor

自动实现 RAG 检索增强：

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder().topK(5).build())
            .build()
    )
    .build();
```

### 自定义 Advisor

#### 请求日志 Advisor

记录每次请求和响应的详细信息：

```java
public class LoggingAdvisor implements CallAdvisor {

    private static final Logger log = LoggerFactory.getLogger(LoggingAdvisor.class);

    @Override
    public String getName() {
        return "LoggingAdvisor";
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
            CallAdvisorChain chain) {
        log.info("请求内容: {}", request.text());
        ChatClientResponse response = chain.nextCall(request);
        log.info("响应内容: {}", response.chatResponse()
            .getResult().getOutput().getText());
        return response;
    }
}
```

#### 内容安全过滤 Advisor

对输入和输出进行安全检查：

```java
public class ContentSafetyAdvisor implements CallAdvisor {

    private final List<String> blockedWords;

    public ContentSafetyAdvisor(List<String> blockedWords) {
        this.blockedWords = blockedWords;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
            CallAdvisorChain chain) {
        // 检查用户输入
        String userInput = request.text();
        for (String word : blockedWords) {
            if (userInput.contains(word)) {
                throw new ContentSafetyException(
                    "输入包含不允许的内容");
            }
        }

        ChatClientResponse response = chain.nextCall(request);

        // 检查模型输出
        String output = response.chatResponse()
            .getResult().getOutput().getText();
        for (String word : blockedWords) {
            if (output.contains(word)) {
                return sanitizeResponse(response, word);
            }
        }
        return response;
    }
}
```

### Advisor 链组合

多个 Advisor 按顺序组成处理链：

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        // 顺序执行：日志 → 安全检查 → 记忆管理 → RAG
        loggingAdvisor,
        contentSafetyAdvisor,
        MessageChatMemoryAdvisor.builder(chatMemory).build(),
        QuestionAnswerAdvisor.builder(vectorStore).build()
    )
    .build();
```

## 结构化输出

除了基础的 BeanOutputConverter，Spring AI 还提供了更灵活的结构化输出方案。

### MapOutputConverter

将 LLM 输出解析为 Map 结构：

```java
MapOutputConverter converter = new MapOutputConverter();

String response = chatClient.prompt()
    .user(u -> u.text("""
        分析以下技术栈的优缺点：
        {techStack}
        {format}
        """)
        .param("techStack", "Spring Boot + MyBatis + MySQL")
        .param("format", converter.getFormat()))
    .call()
    .content();

Map<String, Object> analysis = converter.convert(response);
```

### 自定义 OutputConverter

实现自定义的输出格式转换器：

```java
public class MarkdownTableConverter implements OutputConverter<List<Map<String, String>>> {

    private final String formatInstructions;

    public MarkdownTableConverter() {
        this.formatInstructions = """
            请以 Markdown 表格格式输出，列名用英文，值用中文。
            格式示例：
            | 名称 | 说明 |
            |------|------|
            | xxx  | xxx  |
            """;
    }

    @Override
    public String getFormat() {
        return formatInstructions;
    }

    @Override
    public List<Map<String, String>> convert(String text) {
        // 解析 Markdown 表格为 List<Map>
        return parseMarkdownTable(text);
    }
}
```

### 使用 entity() 的链式映射

ChatClient 的 entity() 方法支持泛型推断：

```java
// 映射到单个对象
public record CodeReview(
    String summary,
    List<String> issues,
    List<String> suggestions,
    int severityScore
) {}

CodeReview review = chatClient.prompt()
    .user("审查以下代码：\n" + code)
    .call()
    .entity(CodeReview.class);

// 映射到嵌套结构
public record ApiDocumentation(
    String endpoint,
    String method,
    List<ParamDoc> parameters,
    String responseBody
) {}

public record ParamDoc(String name, String type, String description) {}

List<ApiDocumentation> docs = chatClient.prompt()
    .user("为以下 Controller 生成 API 文档：\n" + controllerCode)
    .call()
    .entity(new ParameterizedTypeReference<List<ApiDocumentation>>() {});
```

## 多轮对话管理

### ChatMemory 架构

Spring AI 的 ChatMemory 采用分层设计：

```
ChatMemory（接口）
├── InMemoryChatMemory       — 内存存储，开发测试用
├── VectorStoreChatMemory    — 向量存储，支持语义检索
└── CassandraChatMemory      — 持久化存储，生产环境用
```

### 会话隔离

通过 conversationId 实现多用户会话隔离：

```java
@Service
public class ConversationService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public ConversationService(ChatClient.Builder builder,
            ChatMemory chatMemory) {
        this.chatMemory = chatMemory;
        this.chatClient = builder
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .maxSize(20)
                    .build()
            )
            .build();
    }

    public String chat(String userId, String message) {
        String conversationId = "user-" + userId;
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
    }

    public void clearHistory(String userId) {
        chatMemory.clear("user-" + userId);
    }
}
```

### 对话历史窗口策略

通过 maxSize 控制保留的历史消息数量，避免 Token 超限：

```java
// 保留最近 10 条消息（5 轮对话）
MessageChatMemoryAdvisor.builder(chatMemory)
    .maxSize(10)
    .build()

// 保留最近 50 条消息（25 轮对话）
MessageChatMemoryAdvisor.builder(chatMemory)
    .maxSize(50)
    .build()
```

## 模型评估

### Evaluator 接口

Spring AI 提供了模型评估框架，用于量化评估 LLM 输出的质量：

```java
public interface Evaluator {
    EvaluationResponse evaluate(EvaluationRequest request);
}
```

### RelevancyEvaluator

评估 LLM 回答与上下文的相关性：

```java
@Service
public class EvaluationService {

    private final RelevancyEvaluator relevancyEvaluator;

    public EvaluationService(ChatModel evaluationChatModel) {
        this.relevancyEvaluator = RelevancyEvaluator.builder()
            .chatModel(evaluationChatModel)
            .build();
    }

    public boolean isRelevant(String userQuery, String context,
            String response) {
        EvaluationRequest request = new EvaluationRequest(
            userQuery,
            List.of(new Context(context)),
            response
        );
        EvaluationResponse result = relevancyEvaluator.evaluate(request);
        return result.isPass();
    }
}
```

### 批量评估

对 RAG 系统进行端到端的质量评估：

```java
@Test
void evaluateRagQuality() {
    List<TestData> testCases = loadTestCases();

    int passCount = 0;
    for (TestData test : testCases) {
        String answer = ragService.query(test.question());

        EvaluationRequest request = new EvaluationRequest(
            test.question(),
            List.of(new Context(test.expectedContext())),
            answer
        );

        EvaluationResponse result = relevancyEvaluator.evaluate(request);
        if (result.isPass()) passCount++;
    }

    double passRate = (double) passCount / testCases.size();
    assertThat(passRate).isGreaterThan(0.8);  // 期望 80% 以上通过率
}
```

## 可观测性

### Micrometer 集成

Spring AI 自动将关键指标注册到 Micrometer：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

核心指标包括：

- `spring.ai.chat.model.calls` — 模型调用次数
- `spring.ai.chat.model.tokens` — Token 使用量
- `spring.ai.chat.model.latency` — 模型响应延迟

### 自定义指标

```java
@Configuration
public class ObservabilityConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> aiMetrics() {
        return registry -> {
            Counter.builder("ai.chat.requests")
                .description("AI 对话请求总数")
                .tag("application", "my-ai-app")
                .register(registry);
        };
    }
}
```

### 链路追踪

结合 Spring Cloud Sleuth 或 Micrometer Tracing 实现分布式追踪：

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

### Prometheus 集成

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

在 Prometheus 中配置抓取任务后，即可在 Grafana 中可视化监控 AI 模型的调用情况。

## 流式响应处理

### Flux<ChatResponse> 基础

流式调用返回完整的 ChatResponse 流，包含元数据信息：

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ChatResponse> chatStream(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .chatResponse();
}
```

### SSE（Server-Sent Events）

通过 SSE 将流式响应推送到前端：

```java
@GetMapping(value = "/sse", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> sseChat(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content()
        .map(chunk -> ServerSentEvent.<String>builder()
            .data(chunk)
            .build());
}
```

前端通过 EventSource API 接收：

```javascript
const eventSource = new EventSource('/api/chat/sse?message=你好');
eventSource.onmessage = function(event) {
    document.getElementById('response').textContent += event.data;
};
eventSource.onerror = function() {
    eventSource.close();
};
```

### 流式响应中的工具调用

Spring AI 支持在流式模式下处理 Function Calling，使用 `publish()` 操作符实现并行流处理：

```java
private Flux<ChatClientResponse> streamWithToolCallResponses(
        Flux<ChatClientResponse> responseFlux) {

    return responseFlux.publish(shared -> {
        // 分支1：流式输出给用户
        Flux<ChatClientResponse> streamingBranch =
            new ChatClientMessageAggregator()
                .aggregateChatClientResponse(shared, aggregated::set);

        // 分支2：流结束后检查是否需要工具调用
        Flux<ChatClientResponse> toolCallBranch = Flux
            .defer(() -> handleToolCall(aggregated.get()));

        return streamingBranch.concatWith(toolCallBranch);
    });
}
```

这种模式确保了用户体验的实时性，同时在流结束后自动处理工具调用并递归执行。

## 小结

本文深入探讨了 Spring AI 的高级特性：

1. **Function Calling**：通过注册 Java 函数让 LLM 与外部世界交互
2. **RAG 集成**：DocumentReader → DocumentTransformer → VectorStore 完整管道
3. **向量存储**：PgVector、Milvus、SimpleVector、Chroma 等多种选择
4. **Advisor 链**：请求/响应拦截机制，支持日志、安全、记忆等横切关注点
5. **结构化输出**：MapOutputConverter 和自定义格式转换器
6. **多轮对话管理**：ChatMemory 架构和会话隔离
7. **模型评估**：RelevancyEvaluator 评估回答质量
8. **可观测性**：Micrometer 指标监控与 Prometheus 集成
9. **流式响应**：Flux<ChatResponse>、SSE 和流式工具调用处理

下一篇文章将聚焦 Spring AI Alibaba，介绍如何使用国产大模型构建 AI 应用。
