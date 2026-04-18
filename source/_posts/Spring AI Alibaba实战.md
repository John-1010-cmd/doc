---
title: Spring AI Alibaba实战
date: 2025-07-05
updated : 2025-07-05
categories:
- Spring AI
tags: 
  - Spring AI
  - 实战
description: Spring AI Alibaba 实战开发，使用通义千问等国产模型构建 AI 应用。
series: Spring AI 实战
series_order: 3
---

## Spring AI Alibaba 简介

Spring AI Alibaba 是基于 Spring AI 构建的开源扩展项目，专门为阿里云 AI 生态提供 Spring 风格的集成方案。它将通义千问（Qwen）、通义万相（Wanx）、Paraformer、CosyVoice 等国产模型通过 DashScope API 统一接入 Spring 应用，为 Java 开发者提供了一条使用国产大模型的低门槛路径。

### 核心定位

- **Spring AI 的阿里云实现**：完全遵循 Spring AI 的标准接口，无缝切换
- **DashScope API 封装**：统一对接阿里云 DashScope 平台的全部模型能力
- **云原生集成**：与 Spring Cloud Alibaba 深度整合，支持微服务架构
- **多模态覆盖**：文本、图像、音频全覆盖的 AI 能力矩阵

### 架构概览

```
Spring AI 标准 API（ChatModel / EmbeddingModel / ImageModel / AudioModel）
        ↑
Spring AI Alibaba（DashScope 实现）
        ↑
DashScope API（阿里云模型服务统一接入点）
        ↑
通义千问 | 通义万相 | Paraformer | CosyVoice ...
```

## 通义千问（Qwen）模型接入配置

### 添加依赖

在 `pom.xml` 中添加 Spring AI Alibaba 的 DashScope starter：

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
    <version>1.1.2.1</version>
</dependency>
```

### API Key 配置

在阿里云控制台的 DashScope 服务中创建 API Key，然后在配置文件中设置：

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-max
          temperature: 0.7
```

也可以通过环境变量设置：

```bash
export DASHSCOPE_API_KEY=sk-your-api-key-here
```

### 验证连接

创建一个简单的测试类验证模型连接是否正常：

```java
@SpringBootApplication
public class DashScopeDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DashScopeDemoApplication.class, args);
    }

    @Bean
    CommandLineRunner runner(ChatClient.Builder chatClientBuilder) {
        return args -> {
            ChatClient chatClient = chatClientBuilder.build();

            String response = chatClient.prompt()
                .user("你好，请用一句话介绍你自己")
                .call()
                .content();

            System.out.println("通义千问回复: " + response);
        };
    }
}
```

## DashScope API 集成

DashScope 是阿里云的大模型服务平台，Spring AI Alibaba 通过封装 DashScope API 实现对各种模型的统一访问。

### API 特点

- **统一的接入点**：所有模型通过同一个 API 地址访问
- **OpenAI 兼容模式**：DashScope API 支持 OpenAI 格式的请求和响应
- **流式响应**：支持 SSE 流式输出
- **异步调用**：支持异步非阻塞调用模式

### 配置项说明

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
      chat:
        enabled: true
        options:
          model: qwen-max
          temperature: 0.7
          top-p: 0.8
          max-tokens: 2048
      embedding:
        options:
          model: text-embedding-v2
```

## 对话补全

Spring AI Alibaba 支持通义千问全系列模型，不同模型适用于不同场景。

### 模型选择指南

| 模型 | 适用场景 | 特点 |
|------|----------|------|
| qwen-max | 复杂推理、长文本理解 | 能力最强，成本最高 |
| qwen-plus | 日常对话、内容生成 | 能力与成本均衡 |
| qwen-turbo | 简单问答、快速响应 | 速度最快，成本最低 |
| qwen-long | 超长文本处理 | 支持 1M Token 上下文 |

### 基本对话

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String chat(String message) {
        return chatClient.prompt()
            .system("你是一个有帮助的AI助手，请用中文回答")
            .user(message)
            .call()
            .content();
    }
}
```

### 切换模型

通过配置或代码动态切换模型：

```java
// 方式1：配置文件指定模型
// spring.ai.dashscope.chat.options.model=qwen-plus

// 方式2：代码中动态指定
String response = chatClient.prompt()
    .user("快速回答：Java 是什么？")
    .options(DashScopeChatOptions.builder()
        .withModel("qwen-turbo")
        .build())
    .call()
    .content();
```

### 流式对话

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

### 多轮对话

结合 ChatMemory 实现多轮对话：

```java
@Service
public class MultiTurnChatService {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public MultiTurnChatService(ChatClient.Builder builder) {
        this.chatMemory = new InMemoryChatMemory();
        this.chatClient = builder
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .maxSize(20)
                    .build()
            )
            .build();
    }

    public String chat(String sessionId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID_KEY, sessionId))
            .call()
            .content();
    }
}
```

## 文本 Embedding

Spring AI Alibaba 支持通义千问的文本 Embedding 模型，用于语义搜索和 RAG 场景。

### text-embedding-v2 接入

```yaml
spring:
  ai:
    dashscope:
      embedding:
        options:
          model: text-embedding-v2
```

### 基本使用

```java
@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    public float[] getEmbedding(String text) {
        return embeddingModel.embed(text);
    }

    public double calculateSimilarity(String text1, String text2) {
        float[] vector1 = embeddingModel.embed(text1);
        float[] vector2 = embeddingModel.embed(text2);
        return cosineSimilarity(vector1, vector2);
    }

    private double cosineSimilarity(float[] a, float[] b) {
        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;
        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### 结合向量存储

将 DashScope Embedding 与 VectorStore 结合构建 RAG 系统：

```java
@Configuration
public class RagConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}

@Service
public class DocumentIndexService {

    private final VectorStore vectorStore;

    public DocumentIndexService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void indexDocument(String content, Map<String, Object> metadata) {
        Document doc = new Document(content, metadata);
        vectorStore.add(List.of(doc));
    }

    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(topK)
                .similarityThreshold(0.7)
                .build()
        );
    }
}
```

## 图像生成

Spring AI Alibaba 支持通义万相（Wanx）图像生成模型，可以在应用中集成 AI 绘画能力。

### 配置

```yaml
spring:
  ai:
    dashscope:
      image:
        options:
          model: wanx-v1
          size: 1024x1024
```

### 图像生成

```java
@Service
public class ImageGenerationService {

    private final ImageModel imageModel;

    public ImageGenerationService(ImageModel imageModel) {
        this.imageModel = imageModel;
    }

    public String generateImage(String prompt) {
        ImagePrompt imagePrompt = new ImagePrompt(prompt);
        ImageResponse response = imageModel.call(imagePrompt);

        return response.getResult().getOutput().getUrl();
    }

    public byte[] generateImageBytes(String prompt) {
        ImagePrompt imagePrompt = new ImagePrompt(prompt);
        ImageResponse response = imageModel.call(imagePrompt);

        String imageUrl = response.getResult().getOutput().getUrl();
        // 下载图片字节数据
        return downloadImage(imageUrl);
    }

    private byte[] downloadImage(String url) {
        try (var inputStream = new URL(url).openStream()) {
            return inputStream.readAllBytes();
        } catch (IOException e) {
            throw new RuntimeException("下载图片失败", e);
        }
    }
}
```

### 图像生成 API

```java
@RestController
@RequestMapping("/api/image")
public class ImageController {

    private final ImageGenerationService imageService;

    public ImageController(ImageGenerationService imageService) {
        this.imageService = imageService;
    }

    @PostMapping("/generate")
    public Map<String, String> generate(@RequestBody ImageRequest request) {
        String imageUrl = imageService.generateImage(request.prompt());
        return Map.of("url", imageUrl);
    }

    @GetMapping(value = "/generate", produces = MediaType.IMAGE_PNG_VALUE)
    public byte[] generateAndReturn(@RequestParam String prompt) {
        return imageService.generateImageBytes(prompt);
    }
}

public record ImageRequest(String prompt) {}
```

## 音频识别与合成

### Paraformer 语音识别

Paraformer 是阿里达摩院开源的语音识别模型，支持中文语音转文字：

```java
@Service
public class SpeechRecognitionService {

    private final AudioModel audioModel;

    public SpeechRecognitionService(AudioModel audioModel) {
        this.audioModel = audioModel;
    }

    public String transcribe(Resource audioFile) {
        AudioPrompt audioPrompt = new AudioPrompt(audioFile);
        AudioResponse response = audioModel.call(audioPrompt);

        return response.getResult().getOutput().getText();
    }
}
```

### CosyVoice 语音合成

CosyVoice 是通义实验室的语音合成模型，支持自然语音生成：

```java
@Service
public class SpeechSynthesisService {

    private final AudioModel audioModel;

    public SpeechSynthesisService(AudioModel audioModel) {
        this.audioModel = audioModel;
    }

    public byte[] synthesize(String text) {
        // 调用语音合成模型
        AudioPrompt prompt = new AudioPrompt(text);
        AudioResponse response = audioModel.call(prompt);

        return response.getResult().getOutput().getData();
    }
}
```

### 音频处理 API

```java
@RestController
@RequestMapping("/api/audio")
public class AudioController {

    private final SpeechRecognitionService recognitionService;
    private final SpeechSynthesisService synthesisService;

    public AudioController(SpeechRecognitionService recognitionService,
            SpeechSynthesisService synthesisService) {
        this.recognitionService = recognitionService;
        this.synthesisService = synthesisService;
    }

    @PostMapping(value = "/transcribe", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public Map<String, String> transcribe(
            @RequestParam("file") MultipartFile audioFile) {
        Resource resource = audioFile.getResource();
        String text = recognitionService.transcribe(resource);
        return Map.of("text", text);
    }

    @PostMapping(value = "/synthesize", produces = MediaType.APPLICATION_OCTET_STREAM_VALUE)
    public byte[] synthesize(@RequestBody Map<String, String> request) {
        return synthesisService.synthesize(request.get("text"));
    }
}
```

## Function Calling 与工具集成

Spring AI Alibaba 完全支持 Spring AI 的 Function Calling 机制，并针对阿里云服务提供了开箱即用的工具。

### 自定义工具函数

```java
@Configuration
public class DashScopeToolConfig {

    @Bean
    @Description("查询指定城市的实时天气信息")
    public Function<WeatherRequest, WeatherResponse> weatherTool() {
        return request -> {
            // 调用阿里云天气 API 或其他天气服务
            return weatherApiClient.getWeather(request.city());
        };
    }

    @Bean
    @Description("查询阿里云 RDS 数据库的运行状态")
    public Function<DatabaseRequest, DatabaseResponse> databaseStatusTool() {
        return request -> {
            // 调用阿里云 RDS API
            return aliyunRdsService.getStatus(request.instanceId());
        };
    }

    public record WeatherRequest(String city) {}
    public record WeatherResponse(
        double temperature, String description, int humidity) {}
    public record DatabaseRequest(String instanceId) {}
    public record DatabaseResponse(
        String status, double cpuUsage, double memoryUsage) {}
}
```

### 在对话中使用工具

```java
String response = chatClient.prompt()
    .user("帮我查一下杭州的天气和数据库实例prod-rds-01的状态")
    .functions("weatherTool", "databaseStatusTool")
    .call()
    .content();
```

通义千问模型会自动识别用户意图，分别调用两个工具获取数据，然后综合生成自然语言回答。

## 与 Spring Cloud 微服务架构集成

Spring AI Alibaba 可以与 Spring Cloud Alibaba 无缝集成，构建 AI 微服务架构。

### 依赖配置

```xml
<dependencies>
    <!-- Spring AI Alibaba -->
    <dependency>
        <groupId>com.alibaba.cloud.ai</groupId>
        <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
    </dependency>

    <!-- Spring Cloud Alibaba Nacos -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Spring Cloud OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

### AI 服务注册与发现

将 AI 服务注册到 Nacos，其他微服务通过 Feign 调用：

```java
// AI 服务提供方
@SpringBootApplication
@EnableDiscoveryClient
@RestController
@RequestMapping("/api/ai")
public class AiServiceProvider {

    private final ChatClient chatClient;

    public AiServiceProvider(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @PostMapping("/chat")
    public String chat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
            .system(request.systemPrompt())
            .user(request.userMessage())
            .call()
            .content();
    }
}

// AI 服务消费方
@FeignClient(name = "ai-service")
public interface AiServiceClient {

    @PostMapping("/api/ai/chat")
    String chat(@RequestBody ChatRequest request);
}
```

### 配置中心管理 Prompt

将 Prompt 模板存储在 Nacos 配置中心，实现动态更新：

```yaml
# Nacos 配置 dataId: ai-prompts.yaml
prompts:
  code-review: |
    你是一位资深代码审查专家，请审查以下 {language} 代码：
    {code}
    从代码质量、安全性、性能三个维度给出评审意见。
  customer-service: |
    你是 {company} 的客服代表，请以专业友好的语气回答用户问题。
    公司名称：{company}
    业务范围：{business}
```

在代码中动态加载：

```java
@Configuration
@RefreshScope
@ConfigurationProperties(prefix = "prompts")
public class PromptConfig {

    private Map<String, String> prompts = new HashMap<>();

    public Map<String, String> getPrompts() {
        return prompts;
    }

    public void setPrompts(Map<String, String> prompts) {
        this.prompts = prompts;
    }

    public String getPrompt(String name) {
        return prompts.getOrDefault(name, "");
    }
}
```

### AI 网关集成

通过 Spring Cloud Gateway 统一管理 AI 服务的访问：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ai-chat
          uri: lb://ai-service
          predicates:
            - Path=/api/ai/**
          filters:
            - name: RateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

## 完整实战：构建企业智能客服系统

下面通过一个完整的企业智能客服系统案例，综合运用前面介绍的各项能力。

### 系统架构

```
用户 → API 网关 → 客服服务 → ChatClient（通义千问）
                              ├── ChatMemory（对话记忆）
                              ├── QuestionAnswerAdvisor（RAG）
                              ├── Function Calling（订单查询、工单创建）
                              └── 内容安全 Advisor
```

### 1. 依赖配置

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud.ai</groupId>
        <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-advisors-vector-store</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 2. 核心配置

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
      embedding:
        options:
          model: text-embedding-v2
```

### 3. 知识库构建

```java
@Configuration
public class KnowledgeBaseConfig {

    @Bean
    public VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
}

@Service
public class KnowledgeBaseService {

    private final VectorStore vectorStore;

    public KnowledgeBaseService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    @PostConstruct
    public void init() {
        // 加载客服知识库文档
        List<Document> documents = loadKnowledgeDocuments();
        vectorStore.add(documents);
    }

    private List<Document> loadKnowledgeDocuments() {
        // 从文件或数据库加载 FAQ、产品说明等文档
        TextReader reader = new TextReader(
            new ClassPathResource("knowledge/faq.txt")
        );
        List<Document> docs = reader.get();

        TokenTextSplitter splitter = new TokenTextSplitter(500, 100, 5, 10000, true);
        return splitter.apply(docs);
    }
}
```

### 4. 工具函数定义

```java
@Configuration
public class CustomerServiceTools {

    @Bean
    @Description("根据订单号查询订单状态和物流信息")
    public Function<OrderQueryRequest, OrderQueryResponse> orderQueryTool() {
        return request -> orderService.queryOrder(request.orderNo());
    }

    @Bean
    @Description("创建客服工单，记录用户反馈的问题")
    public Function<TicketCreateRequest, TicketCreateResponse> ticketCreateTool() {
        return request -> ticketService.createTicket(
            request.userId(), request.title(), request.description());
    }

    public record OrderQueryRequest(String orderNo) {}
    public record OrderQueryResponse(
        String orderNo, String status, String logistics, String estimatedArrival) {}
    public record TicketCreateRequest(
        String userId, String title, String description) {}
    public record TicketCreateResponse(String ticketId, String status) {}
}
```

### 5. 客服服务核心

```java
@Service
public class CustomerServiceBot {

    private final ChatClient chatClient;

    public CustomerServiceBot(ChatClient.Builder builder,
            ChatMemory chatMemory, VectorStore vectorStore) {
        this.chatClient = builder
            .defaultSystem("""
                你是{company}的智能客服代表。
                请以专业友好的语气回答用户问题。
                回答时遵循以下规则：
                1. 优先使用知识库中的信息回答
                2. 涉及订单查询时使用订单查询工具
                3. 无法解决的问题引导创建工单
                4. 不要编造不确定的信息
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .maxSize(20)
                    .build(),
                QuestionAnswerAdvisor.builder(vectorStore)
                    .searchRequest(SearchRequest.builder()
                        .topK(3)
                        .similarityThreshold(0.7)
                        .build())
                    .build()
            )
            .defaultFunctions("orderQueryTool", "ticketCreateTool")
            .build();
    }

    public String chat(String sessionId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID_KEY, sessionId))
            .call()
            .content();
    }

    public Flux<String> chatStream(String sessionId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID_KEY, sessionId))
            .stream()
            .content();
    }
}
```

### 6. API 接口

```java
@RestController
@RequestMapping("/api/customer-service")
public class CustomerServiceController {

    private final CustomerServiceBot bot;

    public CustomerServiceController(CustomerServiceBot bot) {
        this.bot = bot;
    }

    @PostMapping("/chat")
    public Map<String, String> chat(@RequestBody ChatRequest request) {
        String reply = bot.chat(request.sessionId(), request.message());
        return Map.of("reply", reply);
    }

    @GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamChat(
            @RequestParam String sessionId,
            @RequestParam String message) {
        return bot.chatStream(sessionId, message)
            .map(chunk -> ServerSentEvent.<String>builder()
                .data(chunk)
                .build());
    }
}

public record ChatRequest(String sessionId, String message) {}
```

### 7. 对话效果示例

```
用户：我昨天下的订单到哪了？订单号是 ORD20250705001
AI：我来帮您查询订单 ORD20250705001 的状态。
    [调用 orderQueryTool]
    您的订单目前正在运输中，预计明天（7月7日）到达。
    物流公司：顺丰速运，运单号：SF1234567890。

用户：太慢了，我要投诉
AI：非常抱歉给您带来了不好的体验。我帮您创建一个工单，
    我们的客服团队会尽快处理。
    [调用 ticketCreateTool]
    已为您创建工单 TK-20250705-001，
    我们的专属客服会在2小时内联系您处理。

用户：你们公司的退货政策是什么？
AI：[通过 RAG 检索知识库]
    根据我们的退货政策，自收到商品之日起7天内可以申请退货...
```

## 小结

本文通过 Spring AI Alibaba 实战，完整展示了国产大模型的集成方案：

1. **Spring AI Alibaba** 是 Spring AI 在阿里云生态的扩展实现
2. **通义千问** 提供 qwen-max/plus/turbo 多档位模型选择
3. **DashScope API** 统一接入阿里云全部 AI 模型能力
4. **多模态能力**：文本对话、Embedding、图像生成、语音识别与合成
5. **Function Calling**：与 Spring AI 标准完全兼容
6. **Spring Cloud 集成**：与 Nacos、Gateway 等组件无缝配合
7. **企业实战**：完整的智能客服系统案例，综合运用 RAG、记忆、工具调用

下一篇文章将整理 Spring AI 的高频面试题，帮助巩固核心知识。
