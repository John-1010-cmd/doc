---
title: Spring AI Q&A
date: 2025-07-20
updated : 2025-07-20
categories:
- Spring AI
tags: 
  - Spring AI
description: Spring AI 高频面试题整理，覆盖核心 API、RAG 集成、Function Calling 等关键问题。
series: Spring AI 实战
series_order: 4
---

## Spring AI 的核心架构是什么？

1. **分层架构设计**：Spring AI 采用经典的三层架构——Model 层、API 层和应用层。Model 层对接具体的 AI 提供商（OpenAI、Ollama 等），API 层提供统一的抽象接口（ChatModel、EmbeddingModel 等），应用层通过 ChatClient 向开发者暴露简洁的链式 API。

2. **核心抽象接口**：框架围绕几个关键接口构建——`ChatModel` 负责对话生成，`EmbeddingModel` 负责文本向量化，`ImageModel` 负责图像生成，`AudioModel` 负责语音处理。这些接口屏蔽了不同模型提供商的 API 差异。

3. **可移植性设计**：通过统一的接口抽象，开发者只需更换 starter 依赖和配置即可在不同模型之间切换，无需修改业务代码。这类似于 JDBC 对数据库的抽象。

4. **Advisor 机制**：借鉴 Spring Interceptor 的设计思想，Advisor 链提供了请求/响应的拦截增强能力，用于实现对话记忆、RAG 检索、日志记录等横切关注点。

5. **Spring 生态整合**：深度集成 Spring Boot 自动配置、Micrometer 可观测性、Spring Cloud 配置中心等基础设施，对 Spring 开发者几乎没有额外学习成本。

## ChatClient 与 ChatModel 的区别？

1. **定位不同**：`ChatModel` 是底层模型接口，直接与 AI 提供商交互；`ChatClient` 是高层门面 API，内部组合了 ChatModel、Advisor、工具函数等多种能力。

2. **使用方式不同**：ChatModel 需要手动构建 Prompt 对象并处理 ChatResponse；ChatClient 提供链式 API，通过 `.prompt().user().call().content()` 即可完成一次对话。

3. **功能范围不同**：ChatClient 在 ChatModel 之上增加了 Advisor 链管理、Function Calling 集成、对话记忆、Prompt 模板、结构化输出映射等高级能力。ChatModel 只负责纯粹的模型调用。

4. **推荐使用场景**：日常开发应优先使用 ChatClient，它封装了大部分常用能力。只有在需要精细控制模型参数、直接访问原始响应元数据等场景下才需要直接使用 ChatModel。

5. **关系类比**：ChatClient 与 ChatModel 的关系类似于 JdbcTemplate 与 DataSource 的关系——一个是高层封装，一个是底层组件。ChatClient 通过 `ChatClient.builder(chatModel)` 构造，内部委托 ChatModel 执行实际的模型调用。

## Spring AI 如何实现 Function Calling？

1. **函数注册**：通过 `@Bean` + `@Description` 注解将普通的 Java `Function<I, O>` 注册为工具函数。`@Description` 提供函数的功能描述，帮助 LLM 理解何时调用该函数。

2. **触发方式**：在 ChatClient 调用时通过 `.functions("beanName")` 指定可用的工具函数。LLM 在分析用户输入后，如果判断需要外部数据，会返回工具调用请求而非直接回答。

3. **自动执行流程**：Spring AI 接收到 LLM 的工具调用请求后，自动执行对应的 Java 函数，将执行结果作为工具消息附加到对话上下文中，然后再次调用 LLM 生成最终回答。

4. **递归调用**：如果 LLM 在一次请求中需要调用多个工具，或者工具调用后仍需进一步调用其他工具，Spring AI 支持递归处理，直到 LLM 认为无需再调用工具为止。

5. **参数提取**：LLM 自动从用户输入中提取函数所需的参数（如城市名、日期等），以 JSON 格式传递给函数。Spring AI 负责将 JSON 反序列化为函数的输入类型。

6. **流式支持**：在流式模式下，Spring AI 通过 `Flux.publish()` 实现双分支处理——一个分支实时向用户推送内容，另一个分支在流结束后检测并处理工具调用。

## Spring AI 的 RAG 流程是怎样的？

1. **文档加载（DocumentReader）**：使用 PagePdfDocumentReader、TextReader 等组件从 PDF、文本、HTML 等数据源读取原始文档，将其转换为统一的 Document 对象列表。

2. **文档处理（DocumentTransformer）**：通过 TokenTextSplitter 等分块器将长文档切分为适当大小的片段（通常 500-1000 Token），并添加元数据。分块大小需要平衡检索精度和上下文完整性。

3. **向量存储（VectorStore）**：调用 EmbeddingModel 将文档片段转换为向量，存入向量数据库（PgVector、Milvus、Chroma 等）。Spring AI 的 VectorStore 接口统一了不同向量数据库的操作。

4. **检索增强（QuestionAnswerAdvisor）**：用户提问时，Advisor 自动将问题向量化，在 VectorStore 中进行相似度检索，获取最相关的文档片段作为上下文注入 Prompt。

5. **回答生成**：LLM 基于注入的上下文文档生成回答，确保回答基于事实数据而非模型自身的训练知识。可以通过自定义 Prompt 模板控制回答策略（如"只基于上下文回答"）。

6. **质量评估**：使用 RelevancyEvaluator 等评估器量化评估回答与上下文的相关性，确保 RAG 系统的输出质量。

## 如何实现对话记忆（ChatMemory）？

1. **ChatMemory 接口**：Spring AI 定义了 `ChatMemory` 接口，核心方法包括 `add(conversationId, messages)` 添加消息、`get(conversationId, lastN)` 获取最近 N 条消息、`clear(conversationId)` 清除历史。

2. **内置实现**：`InMemoryChatMemory` 是基于内存的实现，适合开发和测试；`VectorStoreChatMemory` 基于向量存储，支持语义检索历史对话；还有基于 Cassandra 等持久化存储的实现。

3. **Advisor 集成**：通过 `MessageChatMemoryAdvisor` 将 ChatMemory 与 ChatClient 集成。Advisor 在每次请求前自动加载历史消息附加到 Prompt，请求后自动保存新的对话消息。

4. **会话隔离**：通过 `conversationId` 实现多用户、多会话的隔离。每个用户或每次对话使用不同的 ID，确保上下文不会串扰。

5. **窗口策略**：通过 `maxSize` 参数控制保留的历史消息数量，避免 Token 数超出模型限制。一般保留最近 10-20 条消息（5-10 轮对话）即可满足大多数场景。

6. **使用方式**：在构建 ChatClient 时注册 Advisor，调用时传入 conversationId 即可。代码示例：`chatClient.prompt().user(message).advisors(a -> a.param(ChatMemory.CONVERSATION_ID_KEY, sessionId)).call().content()`。

## Spring AI 支持哪些向量数据库？

1. **PostgreSQL（PgVectorStore）**：基于 pgvector 扩展，适合已有 PostgreSQL 基础设施的项目。支持 HNSW 和 IVFFlat 索引，通过 JdbcTemplate 操作，无需额外部署服务。

2. **Milvus（MilvusVectorStore）**：专业的分布式向量数据库，支持海量数据和高并发查询。适合对检索性能有高要求的生产环境，支持多种索引类型（IVF_FLAT、IVF_SQ8、HNSW 等）。

3. **Chroma（ChromaVectorStore）**：轻量级开源向量数据库，适合中小规模应用和开发测试。部署简单，API 友好，内置文档管理和查询能力。

4. **SimpleVectorStore**：Spring AI 内置的内存向量存储，零外部依赖。支持将数据序列化到本地 JSON 文件，适合原型验证和小规模场景。

5. **Redis（RedisVectorStore）**：基于 Redis 的向量搜索功能，适合已有 Redis 基础设施的项目。支持 HNSW 索引和混合查询（向量 + 关键词）。

6. **Pinecone（PineconeVectorStore）**：全托管的向量数据库服务，无需运维。适合云原生应用和需要快速上线的项目。

7. **Weaviate、Qdrant 等**：Spring AI 还支持 Weaviate、Qdrant 等向量数据库，所有实现都遵循统一的 VectorStore 接口，切换成本低。

## Advisor 链的工作原理？

1. **责任链模式**：Advisor 链采用责任链（Chain of Responsibility）模式，多个 Advisor 按注册顺序组成处理链。请求依次通过每个 Advisor 的前置处理，到达 LLM 后响应再依次通过每个 Advisor 的后置处理。

2. **接口定义**：`CallAdvisor` 接口定义了 `adviseCall(ChatClientRequest, CallAdvisorChain)` 方法用于同步调用，`StreamAdvisor` 接口定义了 `adviseStream(ChatClientRequest, StreamAdvisorChain)` 方法用于流式调用。

3. **执行流程**：请求方向依次执行 Advisor1.before → Advisor2.before → LLM 调用，响应方向依次执行 Advisor2.after → Advisor1.after。每个 Advisor 可以修改请求参数或响应内容。

4. **内置 Advisor**：`MessageChatMemoryAdvisor` 自动管理对话历史，`QuestionAnswerAdvisor` 自动实现 RAG 检索增强。这些 Advisor 可以自由组合。

5. **自定义 Advisor**：开发者可以实现 Advisor 接口创建自定义拦截器，用于日志记录、内容安全过滤、请求限流、响应缓存等横切关注点。

6. **动态参数**：Advisor 支持通过 `.advisors(a -> a.param(key, value))` 在运行时传入动态参数，如 conversationId、过滤条件等，实现灵活的请求级配置。

## Spring AI Alibaba 与 Spring AI 的关系？

1. **扩展而非替代**：Spring AI Alibaba 是 Spring AI 的扩展实现，而非独立框架。它在 Spring AI 标准接口的基础上，添加了对阿里云 DashScope 平台和通义系列模型的支持。

2. **接口兼容**：Spring AI Alibaba 实现了 Spring AI 的 ChatModel、EmbeddingModel、ImageModel、AudioModel 等标准接口。使用 Spring AI Alibaba 的应用可以无缝切换到其他 Spring AI 实现的模型。

3. **依赖关系**：Spring AI Alibaba 依赖 Spring AI 核心库，通过 `spring-ai-alibaba-starter-dashscope` starter 自动配置通义千问等模型的接入。

4. **额外能力**：除了标准接口实现，Spring AI Alibaba 还提供了阿里云生态的深度集成，如与 Spring Cloud Alibaba（Nacos、Sentinel）的配合、阿里云 OSS 文档读取、阿里云 RDS 数据库查询等。

5. **模型覆盖**：Spring AI Alibaba 通过 DashScope API 统一接入通义千问（qwen-max/plus/turbo）、通义万相（图像生成）、Paraformer（语音识别）、CosyVoice（语音合成）等多模态模型。

6. **版本对应**：Spring AI Alibaba 的版本号与 Spring AI 保持兼容关系。选择 Spring AI Alibaba 版本时需要注意其对 Spring AI 核心版本的依赖要求。

## 如何实现流式响应？

1. **stream() 方法**：ChatClient 提供 `.stream()` 方法替代 `.call()`，返回 `Flux<String>`（文本流）或 `Flux<ChatResponse>`（完整响应流）。

2. **SSE 协议**：在 Spring MVC 控制器中，通过 `produces = MediaType.TEXT_EVENT_STREAM_VALUE` 声明 SSE 端点。前端使用 EventSource API 建立连接，实时接收逐步生成的文本。

3. **两种内容模式**：`.stream().content()` 返回纯文本片段流，适合直接展示；`.stream().chatResponse()` 返回包含元数据的完整 ChatResponse 流，适合需要 Token 用量、工具调用信息等场景。

4. **前端对接**：JavaScript 端通过 `new EventSource(url)` 建立连接，在 `onmessage` 回调中逐步拼接文本内容。连接断开后可通过 `onerror` 回调处理重连逻辑。

5. **流式工具调用**：流式模式下 Spring AI 使用 `Flux.publish()` 创建双分支——一个分支实时推送内容，另一个分支在流结束后聚合完整响应并检测工具调用，确保流式体验与 Function Calling 兼容。

6. **WebFlux 支持**：在 WebFlux 环境中，流式响应可以直接返回 `Flux<ServerSentEvent<String>>`，与响应式编程模型天然契合，避免线程阻塞。

## Spring AI 的可观测性如何实现？

1. **Micrometer 集成**：Spring AI 自动将模型调用的关键指标注册到 Micrometer，包括调用次数（`spring.ai.chat.model.calls`）、Token 使用量（`spring.ai.chat.model.tokens`）和响应延迟（`spring.ai.chat.model.latency`）。

2. **Actuator 暴露**：添加 `spring-boot-starter-actuator` 依赖后，通过 `/actuator/metrics` 端点即可查看 AI 模型调用的各项指标数据。

3. **Prometheus 集成**：配置 Micrometer 的 Prometheus Registry 后，Spring AI 的指标会自动以 Prometheus 格式暴露，可在 Grafana 中创建仪表盘进行可视化监控。

4. **分布式追踪**：结合 Micrometer Tracing 或 Spring Cloud Sleuth，Spring AI 的模型调用会自动生成 Span，记录调用的完整链路信息，包括模型名称、输入输出 Token 数、响应时间等。

5. **自定义指标**：开发者可以通过 MeterRegistry 注册自定义指标，如按业务场景分类的调用量、特定模型的错误率、不同用户的 Token 消耗量等。

6. **日志与告警**：通过自定义 Advisor 记录每次模型调用的详细日志，结合 Prometheus Alertmanager 或其他告警系统，可以实现 Token 用量超限告警、模型调用失败率告警等运维能力。
