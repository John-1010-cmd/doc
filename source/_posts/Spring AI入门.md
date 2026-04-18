---
title: Spring AI入门
date: 2025-06-05
updated : 2025-06-05
categories:
- Spring AI
tags: 
  - Spring AI
  - 实战
description: Spring AI 框架入门指南，快速掌握 Java 应用集成大模型的核心 API。
series: Spring AI 实战
series_order: 1
---

## Spring AI 简介

Spring AI 是 Spring 生态系统中专注于 AI 集成的基础框架，旨在为 Java 开发者提供一套统一的、Spring 风格的 API 抽象层，简化大语言模型（LLM）在企业应用中的集成工作。它的核心设计理念是**可移植性**——通过统一的接口屏蔽不同 AI 提供商的差异，让开发者像切换数据库驱动一样切换底层模型。

Spring AI 的主要特性包括：

- **统一的 ChatModel 接口**：一套 API 对接 OpenAI、Ollama、ZhipuAI、通义千问等多种模型
- **Prompt 模板引擎**：参数化 Prompt 管理，支持模板复用
- **结构化输出**：自动将 LLM 的文本输出转换为 Java 对象
- **Function Calling**：让 LLM 调用 Java 方法，连接外部数据和服务
- **RAG 支持**：内置向量存储、文档处理、检索增强生成的完整链路
- **Advisor 机制**：类似 Servlet Filter 的请求/响应拦截增强链
- **可观测性**：与 Micrometer 深度集成，提供指标监控和链路追踪

## 核心概念

理解 Spring AI 需要掌握以下几个核心抽象：

### Model

Model 是 Spring AI 对 AI 模型的统一抽象，主要包括：

- **ChatModel**：对话模型，接收 Prompt 返回 ChatResponse，是最常用的接口
- **EmbeddingModel**：文本嵌入模型，将文本转换为向量表示，用于语义搜索和 RAG
- **ImageModel**：图像生成模型
- **AudioModel**：语音识别与合成模型

### Prompt

Prompt 是发送给模型的输入封装，包含消息列表和模型参数：

```java
Prompt prompt = new Prompt(
    List.of(
        new SystemMessage("你是一个 Java 编程助手"),
        new UserMessage("请解释 Spring IoC 的原理")
    ),
    ChatOptionsBuilder.builder()
        .withModel("gpt-4o")
        .withTemperature(0.7)
        .build()
);
```

### ChatResponse

ChatResponse 封装了模型的完整响应，包含生成内容、元数据、Token 用量等信息：

```java
ChatResponse response = chatModel.call(prompt);

// 获取生成的文本内容
String content = response.getResult().getOutput().getText();

// 获取 Token 使用量
Usage usage = response.getMetadata().getUsage();
System.out.println("Prompt Tokens: " + usage.getPromptTokens());
System.out.println("Completion Tokens: " + usage.getCompletionTokens());
```

### Embedding

Embedding 将文本转换为高维向量，用于语义搜索、相似度计算等场景：

```java
EmbeddingModel embeddingModel = // ... 获取 EmbeddingModel
float[] vector = embeddingModel.embed("Spring AI 是一个优秀的框架");
// vector 是一个 float 数组，表示文本的语义向量
```

## 快速开始

### 环境要求

- JDK 17 及以上
- Spring Boot 3.2 及以上
- Maven 或 Gradle 构建工具

### 添加依赖

在 `pom.xml` 中添加 Spring AI 的 BOM 和所需模型的 starter：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- OpenAI 支持 -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>

    <!-- Web 支持（用于 SSE 流式响应） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 配置文件

在 `application.yml` 中配置模型连接信息：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

如果使用 Ollama 本地模型：

```yaml
spring:
  ai:
    openai:
      chat:
        base-url: http://localhost:11434
        options:
          model: mistral
```

### 第一个对话程序

```java
@SpringBootApplication
public class SpringAiDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }

    @Bean
    CommandLineRunner runner(ChatClient.Builder chatClientBuilder) {
        return args -> {
            ChatClient chatClient = chatClientBuilder.build();

            String response = chatClient.prompt()
                .user("请用一句话介绍 Spring AI")
                .call()
                .content();

            System.out.println(response);
        };
    }
}
```

启动应用后，你将看到模型生成的回答。这就是一个最简单的 Spring AI 应用。

## ChatClient API

ChatClient 是 Spring AI 提供的高层 API，支持链式调用，是日常开发中最常用的入口。

### 同步对话

最基本的同步调用方式：

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

### 流式对话

对于长文本生成场景，流式响应可以显著改善用户体验：

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .stream()
        .content();
}
```

流式调用返回 `Flux<String>`，客户端通过 SSE（Server-Sent Events）实时接收逐步生成的文本。

### System Message 引导

通过 System Message 设定 AI 的角色和行为边界：

```java
String response = chatClient.prompt()
    .system("你是一个专业的 Java 架构师，回答需要简洁、准确、有深度。")
    .user("如何设计一个高可用的微服务架构？")
    .call()
    .content();
```

## 多模型支持

Spring AI 的核心优势在于统一的 API 接口，切换模型只需更换依赖和配置。

### OpenAI

最主流的商业模型接入：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
```

### Ollama

本地部署的开源模型，适合开发测试和隐私敏感场景：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      chat:
        options:
          model: llama3
```

### ZhipuAI（智谱 AI）

国产大模型 GLM 系列：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-zhipuai-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    zhipu:
      api-key: ${ZHIPU_API_KEY}
      chat:
        options:
          model: glm-4
```

### 通义千问（Qwen）

通过 Spring AI Alibaba 接入：

```xml
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    dashscope:
      api-key: ${DASHSCOPE_API_KEY}
      chat:
        options:
          model: qwen-max
```

### 多模型配置

在同一应用中使用多个模型，通过 `@Qualifier` 区分：

```java
@Configuration
public class ModelConfig {

    @Bean
    @Primary
    public ChatClient primaryChatClient(
            @Qualifier("openAiChatModel") ChatModel openAiChatModel) {
        return ChatClient.builder(openAiChatModel).build();
    }

    @Bean
    public ChatClient localChatClient(
            @Qualifier("ollamaChatModel") ChatModel ollamaChatModel) {
        return ChatClient.builder(ollamaChatModel).build();
    }
}
```

## Prompt 模板

Spring AI 提供了 PromptTemplate 机制，支持参数化 Prompt 管理，避免字符串拼接。

### 基本用法

```java
PromptTemplate template = new PromptTemplate("""
    请为以下技术主题编写一份学习路线图：
    主题：{topic}
    难度级别：{level}
    目标学习者：{audience}
    """);

Prompt prompt = template.create(
    Map.of(
        "topic", "Spring AI",
        "level", "中级",
        "audience", "有 Spring Boot 基础的 Java 开发者"
    )
);

ChatResponse response = chatModel.call(prompt);
```

### 外部模板文件

将 Prompt 模板放在资源文件中，便于管理和迭代：

在 `src/main/resources/prompts/code-review.st` 中定义模板：

```
你是一位资深的代码审查专家，请审查以下代码：

语言：{language}
代码：
```
{code}
```

请从以下维度给出评审意见：
1. 代码质量与可读性
2. 潜在的 Bug 和安全问题
3. 性能优化建议
4. 最佳实践建议
```

在代码中加载并使用：

```java
@Value("classpath:/prompts/code-review.st")
private Resource codeReviewTemplate;

public String reviewCode(String language, String code) {
    PromptTemplate template = new PromptTemplate(codeReviewTemplate);
    Prompt prompt = template.create(Map.of(
        "language", language,
        "code", code
    ));
    return chatModel.call(prompt).getResult().getOutput().getText();
}
```

### 在 ChatClient 中使用模板

```java
String response = chatClient.prompt()
    .user(u -> u.text("请将以下{sourceLang}文本翻译为{targetLang}：{text}")
        .param("sourceLang", "英文")
        .param("targetLang", "中文")
        .param("text", "Spring AI is a framework for AI engineering"))
    .call()
    .content();
```

## 输出解析

LLM 的输出本质上是字符串，Spring AI 提供了 OutputConverter 自动将其转换为结构化数据。

### BeanOutputConverter

将 LLM 输出自动映射为 Java 对象：

```java
// 定义目标数据结构
public record BookRecommendation(
    String title,
    String author,
    String reason,
    int rating
) {}

// 使用 BeanOutputConverter
BeanOutputConverter<BookRecommendation> converter =
    new BeanOutputConverter<>(BookRecommendation.class);

String response = chatClient.prompt()
    .user(u -> u.text("""
        请推荐一本关于 {topic} 的好书
        {format}
        """)
        .param("topic", "Spring 框架")
        .param("format", converter.getFormat()))
    .call()
    .content();

BookRecommendation recommendation = converter.convert(response);
System.out.println(recommendation.title());
System.out.println(recommendation.author());
```

### ListOutputConverter

将 LLM 输出转换为列表：

```java
ListOutputConverter converter = new ListOutputConverter(
    new DefaultConversionService()
);

String response = chatClient.prompt()
    .user(u -> u.text("""
        请列出学习 {topic} 的 {count} 个关键步骤
        {format}
        """)
        .param("topic", "Spring AI")
        .param("count", "5")
        .param("format", converter.getFormat()))
    .call()
    .content();

List<String> steps = converter.convert(response);
steps.forEach(System.out::println);
```

### 直接使用 entity() 方法

ChatClient 提供了更简洁的实体映射方式：

```java
// 单个对象
BookRecommendation book = chatClient.prompt()
    .user("推荐一本 Spring 相关书籍")
    .call()
    .entity(BookRecommendation.class);

// 列表
List<BookRecommendation> books = chatClient.prompt()
    .user("推荐3本 Java 相关书籍")
    .call()
    .entity(new ParameterizedTypeReference<List<BookRecommendation>>() {});
```

## 对话记忆

LLM 本身是无状态的，每次请求都是独立的。对话记忆机制用于在多轮对话中维护上下文。

### ChatMemory 接口

Spring AI 提供了 ChatMemory 抽象来管理对话历史：

```java
public interface ChatMemory {
    void add(String conversationId, List<Message> messages);
    List<Message> get(String conversationId, int lastN);
    void clear(String conversationId);
}
```

### InMemoryChatMemory

最简单的内存实现，适合开发和测试：

```java
@Configuration
public class ChatMemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();
    }
}
```

### MessageChatMemoryAdvisor

通过 Advisor 机制自动管理对话历史：

```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder, ChatMemory chatMemory) {
        this.chatClient = builder
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .maxSize(20)  // 保留最近 20 条消息
                    .build()
            )
            .build();
    }

    public String chat(String conversationId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(a -> a.param(
                ChatMemory.CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
    }
}
```

每次对话时传入 `conversationId`，Advisor 会自动从 ChatMemory 加载历史消息并附加到 Prompt 中，对话结束后自动保存新的消息。

### 基于向量存储的持久化记忆

对于生产环境，可以将对话历史持久化到向量数据库：

```java
@Bean
public ChatMemory chatMemory(VectorStore vectorStore) {
    return VectorStoreChatMemory.builder(vectorStore)
        .build();
}
```

这种方式支持语义检索历史对话，适合需要长期记忆的场景。

## 配置管理

Spring AI 提供了丰富的配置项，通过 `spring.ai` 前缀统一管理。

### 通用配置结构

```yaml
spring:
  ai:
    # OpenAI 配置
    openai:
      api-key: ${OPENAI_API_KEY}
      base-url: https://api.openai.com  # 可替换为代理地址
      chat:
        enabled: true
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2048
          top-p: 1.0
      embedding:
        options:
          model: text-embedding-3-small

    # Ollama 配置
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3
          temperature: 0.7

    # 向量存储配置
    vectorstore:
      pgvector:
        dimensions: 1536
        distance-type: cosine_distance
        index-type: hnsw
```

### 运行时动态配置

除了配置文件，还可以在代码中动态指定模型参数：

```java
ChatResponse response = chatClient.prompt()
    .user("解释量子计算的基本原理")
    .options(ChatOptionsBuilder.builder()
        .withModel("gpt-4o")
        .withTemperature(0.3)  // 降低随机性，提高准确性
        .build())
    .call()
    .chatResponse();
```

### Retry 配置

Spring AI 内置了重试机制，可配置重试策略：

```yaml
spring:
  ai:
    retry:
      max-attempts: 3
      backoff:
        initial-interval: 1000
        multiplier: 2
        max-interval: 10000
      on-http-codes: 429, 500, 502, 503
```

## 小结

本文介绍了 Spring AI 的核心概念和基础用法：

1. **Spring AI** 是 Spring 生态的 AI 集成框架，提供统一的模型接入 API
2. **核心概念**：Model、Prompt、ChatResponse、Embedding 构成了框架的基础抽象
3. **ChatClient** 是日常开发的主要入口，支持同步和流式调用
4. **多模型支持**：通过更换 starter 和配置即可切换 OpenAI、Ollama、ZhipuAI 等模型
5. **Prompt 模板**：参数化管理 Prompt，支持外部模板文件
6. **输出解析**：BeanOutputConverter 和 ListOutputConverter 自动将文本转为结构化数据
7. **对话记忆**：通过 ChatMemory 和 Advisor 机制管理多轮对话上下文

掌握了这些基础能力后，下一篇文章将深入 Function Calling、RAG、Advisor 链等高级特性。
