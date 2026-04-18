---
title: LangChain框架实战
date: 2025-06-20
updated : 2025-06-20
categories:
- AI
tags: 
  - AI
  - 实战
description: LangChain 框架核心概念与实战开发，掌握 AI 应用开发的主流工具链。
series: AI 技术实践
series_order: 5
---

## LangChain 核心架构

LangChain 是目前最流行的 AI 应用开发框架之一，提供了一套完整的工具链来构建基于 LLM 的应用。

### 模块概览

LangChain 的核心架构由以下模块组成：

```
┌─────────────────────────────────────────────┐
│               LangChain 核心架构              │
├──────────┬──────────┬───────────┬────────────┤
│ Model I/O │  Chains  │  Memory   │  Agents    │
│          │          │           │            │
│ - Prompt │ - LCEL   │ - Buffer  │ - Tools    │
│ - LLM    │ - Sequen.│ - Summary │ - Executor │
│ - Parser │ - Router │ - KG      │ - Planner  │
├──────────┴──────────┴───────────┴────────────┤
│              Retrieval (RAG)                  │
│  Document Loader → Splitter → Vector Store   │
├──────────────────────────────────────────────┤
│            Callbacks & Tracing               │
│         LangSmith / LangServe                │
└──────────────────────────────────────────────┘
```

| 模块 | 职责 | 核心组件 |
|------|------|---------|
| Model I/O | 与 LLM 交互 | Prompt Template、LLM/ChatModel、Output Parser |
| Chains | 编排调用链 | LCEL、Sequential Chain、Router Chain |
| Memory | 状态管理 | Buffer Memory、Summary Memory |
| Agents | 自主决策 | Tool、Agent Executor |
| Retrieval | 知识检索 | Document Loader、Text Splitter、Vector Store |

### 安装

```bash
#​ 核心包
pip install langchain langchain-core

#​ 社区集成（各种第三方服务连接器）
pip install langchain-community

#​ OpenAI 集成
pip install langchain-openai

#​ Chroma 向量数据库
pip install chromadb

#​ 全家桶（按需安装）
pip install langchain[all]
```

## LCEL（LangChain Expression Language）管道语法

LCEL 是 LangChain 的核心创新，使用管道操作符 `|` 将组件串联起来。

### 基本语法

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

#​ 组件定义
prompt = ChatPromptTemplate.from_template("讲一个关于{topic}的笑话")
model = ChatOpenAI(model="gpt-4")
parser = StrOutputParser()

#​ LCEL 管道：prompt → model → parser
chain = prompt | model | parser

#​ 执行
result = chain.invoke({"topic": "程序员"})
print(result)
```

LCEL 的核心优势：

- **声明式**：用管道语法清晰表达数据流向
- **统一接口**：所有组件都实现 `Runnable` 接口，可自由组合
- **流式支持**：自动支持流式输出
- **并行执行**：支持并行运行多个链
- **批处理**：内置批处理支持

### Runnable 接口

所有 LCEL 组件都实现 `Runnable` 接口，提供统一的调用方式：

```python
#​ invoke: 单次调用
result = chain.invoke({"topic": "AI"})

#​ batch: 批量调用
results = chain.batch([{"topic": "AI"}, {"topic": "数学"}])

#​ stream: 流式调用
for chunk in chain.stream({"topic": "AI"}):
    print(chunk, end="", flush=True)

#​ ainvoke: 异步单次调用
result = await chain.ainvoke({"topic": "AI"})

#​ abatch: 异步批量调用
results = await chain.abatch([{"topic": "AI"}, {"topic": "数学"}])

#​ astream: 异步流式调用
async for chunk in chain.astream({"topic": "AI"}):
    print(chunk, end="", flush=True)
```

### 组合操作

LCEL 提供多种组合操作符：

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

#​ RunnablePassthrough: 直接传递输入
#​ RunnableParallel: 并行执行多个链

chain = (
    RunnableParallel({
        # 并行：原始查询 + 检索结果
        "question": RunnablePassthrough(),
        "context": retriever,
    })
    | prompt
    | model
    | parser
)

#​ 等价于
chain = (
    {"question": RunnablePassthrough(), "context": retriever}
    | prompt
    | model
    | parser
)
```

### coerce_to_runnable

普通函数也可以包装为 Runnable：

```python
from langchain_core.runnables import RunnableLambda

#​ 使用 RunnableLambda 包装普通函数
def parse_length(text):
    return len(text)

chain = prompt | model | parser | RunnableLambda(parse_length)
result = chain.invoke({"topic": "AI"})
print(f"回答长度: {result}")
```

## Model I/O

Model I/O 模块处理与 LLM 交互的三个核心环节：Prompt 构造、模型调用、输出解析。

### ChatModel 与 LLM

LangChain 区分两种模型接口：

```python
from langchain_openai import ChatOpenAI, OpenAI

#​ ChatModel：聊天模型（推荐）
#​ 输入：消息列表（System/Human/AI Message）
#​ 输出：AIMessage
chat_model = ChatOpenAI(model="gpt-4", temperature=0.7)

#​ LLM：文本补全模型（旧接口）
#​ 输入：字符串
#​ 输出：字符串
llm = OpenAI(model="gpt-3.5-turbo-instruct")
```

ChatModel 是主流接口，支持多角色消息：

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage(content="你是一位 Python 专家。"),
    HumanMessage(content="解释装饰器的工作原理。"),
]

response = chat_model.invoke(messages)
print(response.content)
```

### Prompt Template

Prompt Template 将变量注入到模板中：

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    MessagesPlaceholder,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)

#​ 方式 1：from_template（简单场景）
prompt = ChatPromptTemplate.from_template(
    "将以下文本翻译为{language}：\n{text}"
)

#​ 方式 2：从消息模板构建（复杂场景）
system_template = SystemMessagePromptTemplate.from_template(
    "你是一位专业的{role}，擅长{expertise}。"
)
human_template = HumanMessagePromptTemplate.from_template(
    "{question}"
)

prompt = ChatPromptTemplate.from_messages([
    system_template,
    MessagesPlaceholder(variable_name="history"),  # 动态消息列表
    human_template,
])

#​ 使用
formatted = prompt.invoke({
    "role": "数据科学家",
    "expertise": "机器学习和统计分析",
    "history": [],  # 对话历史
    "question": "如何处理不平衡数据集？",
})
```

### 输出解析器

输出解析器将 LLM 的原始输出转换为结构化数据：

```python
from langchain_core.output_parsers import (
    StrOutputParser,
    JsonOutputParser,
    CommaSeparatedListOutputParser,
)
from langchain_core.pydantic_v1 import BaseModel, Field


#​ 1. 字符串解析器（最简单，直接获取文本）
str_parser = StrOutputParser()

#​ 2. JSON 解析器
json_parser = JsonOutputParser()

#​ 3. Pydantic 模型解析器（推荐，带 schema 约束）
class MovieReview(BaseModel):
    title: str = Field(description="电影名称")
    rating: float = Field(description="评分 1-10")
    summary: str = Field(description="一句话评价")
    pros: list[str] = Field(description="优点列表")
    cons: list[str] = Field(description="缺点列表")

from langchain_core.output_parsers import PydanticOutputParser
pydantic_parser = PydanticOutputParser(pydantic_object=MovieReview)

#​ 将格式说明注入 Prompt
prompt = ChatPromptTemplate.from_template(
    """分析以下电影评论，提取结构化信息。

{format_instructions}

评论内容：
{review}"""
)

chain = prompt | chat_model | pydantic_parser

result = chain.invoke({
    "review": "这部电影视觉效果震撼，但剧情有些老套...",
    "format_instructions": pydantic_parser.get_format_instructions(),
})
#​ result 是 MovieReview 实例
print(result.title, result.rating)
```

## RAG 链

LangChain 提供了完整的 RAG 管道组件。

### 文档加载器

```python
from langchain_community.document_loaders import (
    TextLoader,
    PyPDFLoader,
    CSVLoader,
    DirectoryLoader,
    UnstructuredMarkdownLoader,
)

#​ 加载文本文件
loader = TextLoader("document.txt", encoding="utf-8")
docs = loader.load()

#​ 加载 PDF
pdf_loader = PyPDFLoader("report.pdf")
pdf_docs = pdf_loader.load()  # 每页一个 Document

#​ 加载 CSV（每行一个 Document）
csv_loader = CSVLoader("data.csv")
csv_docs = csv_loader.load()

#​ 批量加载目录下的所有文件
dir_loader = DirectoryLoader(
    "./documents",
    glob="**/*.md",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"},
)
all_docs = dir_loader.load()

#​ 每个 Document 包含：
#​ - page_content: 文本内容
#​ - metadata: 元数据（来源、页码等）
print(pdf_docs[0].page_content[:200])
print(pdf_docs[0].metadata)  # {'source': 'report.pdf', 'page': 0}
```

### 文本切分器

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,
    CharacterTextSplitter,
    MarkdownHeaderTextSplitter,
)

#​ 递归字符切分器（推荐）
splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", "。", ".", " ", ""],
    chunk_size=500,
    chunk_overlap=50,
    length_function=len,
)
chunks = splitter.split_documents(docs)

#​ Markdown 按标题切分
md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]
)
md_chunks = md_splitter.split_text(md_content)
```

### 向量存储与检索器

```python
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

#​ 创建 Embedding 模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

#​ 从文档创建向量存储
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db",
)

#​ 持久化存储
vectorstore.persist()

#​ 加载已有向量存储
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings,
)

#​ 相似度检索
results = vectorstore.similarity_search("机器学习入门", k=5)
for doc in results:
    print(doc.page_content[:100])
    print(doc.metadata)

#​ 带分数的检索
results_with_scores = vectorstore.similarity_search_with_score(
    "机器学习入门", k=5
)
for doc, score in results_with_scores:
    print(f"相似度: {score:.4f} - {doc.page_content[:50]}")

#​ 转换为检索器（供 LCEL 使用）
retriever = vectorstore.as_retriever(
    search_type="mmr",       # 最大边际相关性（平衡相关性和多样性）
    search_kwargs={"k": 5, "fetch_k": 20},
)
```

### 完整 RAG 链

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

#​ 构建 RAG 链
template = """基于以下检索到的文档内容回答问题。如果文档中没有相关信息，说明无法回答。

检索到的文档：
{context}

问题：{question}

回答："""

prompt = ChatPromptTemplate.from_template(template)
model = ChatOpenAI(model="gpt-4", temperature=0)

#​ 方式 1：基础 RAG 链
rag_chain = (
    # 并行获取问题和检索结果
    RunnableParallel({
        "context": retriever | format_docs,  # 检索并格式化
        "question": RunnablePassthrough(),    # 原样传递问题
    })
    | prompt
    | model
    | StrOutputParser()
)

#​ 执行
answer = rag_chain.invoke("什么是机器学习？")


#​ 方式 2：带来源追踪的 RAG 链
def format_docs_with_sources(docs):
    """格式化文档并保留来源信息"""
    formatted = []
    for i, doc in enumerate(docs):
        source = doc.metadata.get("source", "未知来源")
        formatted.append(f"[文档{i+1}]（来源：{source}）\n{doc.page_content}")
    return "\n\n".join(formatted)

rag_chain_with_sources = RunnableParallel({
    "context": retriever | format_docs_with_sources,
    "question": RunnablePassthrough(),
    "sources": retriever | (lambda docs: [
        {"source": d.metadata.get("source"), "content": d.page_content[:100]}
        for d in docs
    ]),
}) | {
    "answer": prompt | model | StrOutputParser(),
    "sources": lambda x: x["sources"],
}

result = rag_chain_with_sources.invoke("什么是机器学习？")
print(result["answer"])
print(f"引用来源: {result['sources']}")
```

## Agent 构建

### 工具绑定

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

#​ 方式 1：使用 @tool 装饰器
@tool
def search_database(query: str, table: str = "users") -> str:
    """在数据库中搜索信息。

    Args:
        query: 搜索关键词
        table: 数据库表名
    """
    # 实际的数据库查询逻辑
    return f"在 {table} 表中搜索 '{query}' 的结果"

@tool
def calculate(expression: str) -> str:
    """计算数学表达式的值。

    Args:
        expression: 数学表达式
    """
    try:
        result = eval(expression)  # 注意：生产环境需安全处理
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"

#​ 方式 2：使用 StructuredTool
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class EmailInput(BaseModel):
    to: str = Field(description="收件人邮箱")
    subject: str = Field(description="邮件主题")
    body: str = Field(description="邮件正文")

send_email = StructuredTool.from_function(
    func=lambda to, subject, body: f"邮件已发送至 {to}",
    name="send_email",
    description="发送邮件",
    args_schema=EmailInput,
)

#​ 绑定工具到模型
tools = [search_database, calculate, send_email]
model = ChatOpenAI(model="gpt-4")
model_with_tools = model.bind_tools(tools)
```

### Agent 执行循环

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

#​ 创建 Prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个智能助手，可以使用工具来帮助用户。"),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),  # Agent 的思考过程
])

#​ 创建 Agent
agent = create_tool_calling_agent(
    llm=model,
    tools=tools,
    prompt=prompt,
)

#​ 创建 Agent Executor
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,          # 打印执行过程
    max_iterations=10,     # 最大迭代次数
    handle_parsing_errors=True,  # 处理解析错误
)

#​ 执行
result = agent_executor.invoke({"input": "查询数据库中名为张三的用户信息"})
print(result["output"])
```

### 自定义 Agent 循环

对于更精细的控制，可以实现自定义的 Agent 循环：

```python
import json
from langchain_core.messages import AIMessage, HumanMessage, ToolMessage


def custom_agent_loop(model_with_tools, tools_map, user_input, max_steps=5):
    """自定义 Agent 执行循环"""
    messages = [HumanMessage(content=user_input)]

    for step in range(max_steps):
        # 1. 调用模型
        response = model_with_tools.invoke(messages)
        messages.append(response)

        # 2. 检查是否有工具调用
        if not response.tool_calls:
            return response.content

        # 3. 执行工具调用
        for tool_call in response.tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]

            print(f"  [步骤 {step+1}] 调用工具: {tool_name}({tool_args})")

            if tool_name in tools_map:
                try:
                    result = tools_map[tool_name].invoke(tool_args)
                    tool_message = ToolMessage(
                        content=str(result),
                        tool_call_id=tool_call["id"],
                    )
                except Exception as e:
                    tool_message = ToolMessage(
                        content=f"工具执行错误: {e}",
                        tool_call_id=tool_call["id"],
                    )
            else:
                tool_message = ToolMessage(
                    content=f"未知工具: {tool_name}",
                    tool_call_id=tool_call["id"],
                )

            messages.append(tool_message)

    return "达到最大步骤数，任务未完成。"


#​ 使用
tools_map = {
    "search_database": search_database,
    "calculate": calculate,
}
result = custom_agent_loop(model_with_tools, tools_map, "计算 (15 + 27) * 3 的结果")
```

## Memory 管理

### 对话缓冲记忆

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables import RunnableWithMessageHistory

#​ 创建会话存储
session_store = {}

def get_session_history(session_id: str) -> InMemoryChatMessageHistory:
    if session_id not in session_store:
        session_store[session_id] = InMemoryChatMessageHistory()
    return session_store[session_id]

#​ 基础对话链（带历史）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])

chain = prompt | model | StrOutputParser()

chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

#​ 多轮对话
response1 = chain_with_history.invoke(
    {"input": "我叫张三"},
    config={"configurable": {"session_id": "user-001"}},
)

response2 = chain_with_history.invoke(
    {"input": "我叫什么名字？"},
    config={"configurable": {"session_id": "user-001"}},
)
#​ 模型会记住"张三"
```

### 摘要记忆

当对话过长时，使用摘要压缩历史：

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import SystemMessage

#​ 摘要生成链
summary_prompt = ChatPromptTemplate.from_template(
    "请将以下对话历史压缩为一段简洁的摘要，保留关键信息：\n\n{history}"
)
summary_chain = summary_prompt | model | StrOutputParser()

class SummaryMemory:
    """摘要记忆管理"""

    def __init__(self, max_messages=10):
        self.messages = []
        self.summary = ""
        self.max_messages = max_messages

    def add_message(self, message):
        """添加消息"""
        self.messages.append(message)

        # 超过阈值时生成摘要
        if len(self.messages) > self.max_messages:
            self._compress()

    def _compress(self):
        """压缩历史消息"""
        history_text = "\n".join(
            f"{m.type}: {m.content}" for m in self.messages[:-2]
        )
        self.summary = summary_chain.invoke({"history": history_text})
        # 只保留最近的消息
        self.messages = self.messages[-2:]

    def get_context(self):
        """获取上下文（摘要 + 最近消息）"""
        context = []
        if self.summary:
            context.append(SystemMessage(content=f"对话摘要：{self.summary}"))
        context.extend(self.messages)
        return context
```

## Callback 与可观测性

### Callback 系统

LangChain 的 Callback 系统允许在链执行的各个阶段插入自定义逻辑：

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyCallbackHandler(BaseCallbackHandler):
    """自定义 Callback 处理器"""

    def on_llm_start(self, serialized, prompts, **kwargs):
        """LLM 开始调用"""
        print(f"[LLM 开始] prompts: {prompts[0][:50]}...")

    def on_llm_end(self, response, **kwargs):
        """LLM 调用结束"""
        token_usage = response.llm_output.get("token_usage", {})
        print(f"[LLM 结束] token 使用: {token_usage}")

    def on_tool_start(self, serialized, input_str, **kwargs):
        """工具开始调用"""
        print(f"[工具开始] {serialized['name']}: {input_str}")

    def on_tool_end(self, output, **kwargs):
        """工具调用结束"""
        print(f"[工具结束] 输出: {str(output)[:100]}")

    def on_chain_start(self, serialized, inputs, **kwargs):
        """链开始执行"""
        print(f"[链开始] {serialized.get('name', 'unknown')}")

    def on_chain_end(self, outputs, **kwargs):
        """链执行结束"""
        print(f"[链结束] 输出: {str(outputs)[:100]}")

    def on_chain_error(self, error, **kwargs):
        """链执行出错"""
        print(f"[链错误] {error}")


#​ 使用 Callback
callback_handler = MyCallbackHandler()

chain = prompt | model | parser
result = chain.invoke(
    {"topic": "AI"},
    config={"callbacks": [callback_handler]},
)
```

## LangSmith 调试与追踪

### 配置 LangSmith

LangSmith 是 LangChain 的官方可观测性平台：

```bash
#​ 环境变量配置
export LANGCHAIN_TRACING_V2="true"
export LANGCHAIN_API_KEY="your-langsmith-api-key"
export LANGCHAIN_PROJECT="my-project"  # 项目名称（可选）
```

配置完成后，所有 LangChain 调用会自动记录到 LangSmith，包括：

- 每次链调用的完整输入输出
- LLM 的 token 使用量
- 工具调用的参数和结果
- 执行时间和延迟
- 错误堆栈

### 在代码中使用

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"

#​ 之后的所有调用都会自动追踪
chain = prompt | model | parser
result = chain.invoke({"topic": "AI"})
#​ 在 LangSmith 控制台可以看到完整的追踪信息
```

### 程序化访问追踪数据

```python
from langsmith import Client

client = Client()

#​ 获取项目的运行记录
runs = client.list_runs(project_name="my-project")

for run in runs:
    print(f"运行 ID: {run.id}")
    print(f"名称: {run.name}")
    print(f"状态: {run.status}")
    print(f"耗时: {run.total_time}")
    print(f"Token 使用: {run.prompt_tokens}/{run.completion_tokens}")
    print("---")
```

## 部署：LangServe 与 REST API

### 基础部署

LangServe 将 LangChain 的 Runnable 快速部署为 REST API：

```bash
pip install langserve
```

```python
from fastapi import FastAPI
from langserve import add_routes
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

app = FastAPI(title="LangChain API Server")

#​ 定义链
prompt = ChatPromptTemplate.from_template("讲一个关于{topic}的笑话")
chain = prompt | ChatOpenAI() | StrOutputParser()

#​ 添加路由
add_routes(app, chain, path="/joke")

#​ 启动服务器: uvicorn server:app --host 0.0.0.0 --port 8000
```

部署后自动获得以下端点：

```
POST /joke/invoke          - 单次调用
POST /joke/batch           - 批量调用
POST /joke/stream          - 流式调用
GET  /joke/playground      - 交互式测试页面
```

### 带输入验证的部署

```python
from pydantic import BaseModel, Field

#​ 定义输入输出 Schema
class JokeInput(BaseModel):
    topic: str = Field(description="笑话主题")

class JokeOutput(BaseModel):
    joke: str = Field(description="笑话内容")

#​ 部署带类型验证的链
add_routes(
    app,
    chain.with_types(input_type=JokeInput, output_type=JokeOutput),
    path="/joke",
    enable_playground_api=True,
)
```

### 多链部署

```python
#​ 部署多个链
add_routes(app, rag_chain, path="/rag")
add_routes(app, agent_executor, path="/agent")
add_routes(app, summary_chain, path="/summary")

#​ 客户端调用
from langserve import RemoteRunnable

remote_rag = RemoteRunnable("http://localhost:8000/rag")
result = remote_rag.invoke({"input": "什么是深度学习？"})
```

## 实战：构建文档问答 + 数据库查询 Agent

### 系统设计

构建一个综合性的 Agent，能够回答文档相关的问题并查询数据库：

```
用户输入
    │
    ▼
┌──────────┐
│ 路由器     │─── 文档问题 ──► RAG 链
│ (意图分类) │
│           ─── 数据查询 ──► SQL Agent
│           ─── 通用问题 ──► 通用对话
└──────────┘
```

### 完整实现

```python
"""
文档问答 + 数据库查询 Agent - 完整实现
"""
from typing import Annotated
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel, RunnablePassthrough
from langchain_core.tools import tool
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_community.vectorstores import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.messages import HumanMessage


#​ ===================== 工具定义 =====================

@tool
def search_knowledge_base(query: str) -> str:
    """搜索知识库文档。当用户询问公司政策、产品信息、技术文档等问题时使用。

    Args:
        query: 搜索查询
    """
    # 实际实现中从向量数据库检索
    retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
    docs = retriever.invoke(query)
    if not docs:
        return "知识库中未找到相关信息。"
    return "\n\n".join(f"[来源: {d.metadata.get('source', '未知')}]\n{d.page_content}" for d in docs)


@tool
def query_database(sql: str) -> str:
    """执行 SQL 查询。当用户需要查询业务数据（订单、用户、产品等）时使用。
    仅支持 SELECT 查询。

    Args:
        sql: SQL 查询语句
    """
    # 安全检查：只允许 SELECT
    sql_upper = sql.strip().upper()
    if not sql_upper.startswith("SELECT"):
        return "错误：仅支持 SELECT 查询，不允许修改数据。"

    # 实际实现中连接数据库执行
    # 这里返回模拟数据
    if "users" in sql_lower(sql):
        return '[{"id": 1, "name": "张三", "email": "zhang@example.com"}]'
    return "查询结果为空"


def sql_lower(sql):
    return sql.lower()


@tool
def get_table_schema(table_name: str) -> str:
    """获取数据库表的结构信息。

    Args:
        table_name: 表名
    """
    schemas = {
        "users": "users(id INT PK, name VARCHAR, email VARCHAR, created_at DATETIME)",
        "orders": "orders(id INT PK, user_id INT FK, amount DECIMAL, status VARCHAR, created_at DATETIME)",
        "products": "products(id INT PK, name VARCHAR, price DECIMAL, stock INT)",
    }
    return schemas.get(table_name, f"未找到表 {table_name} 的结构信息")


#​ ===================== 链定义 =====================

def build_rag_chain():
    """构建 RAG 问答链"""
    prompt = ChatPromptTemplate.from_template(
        """基于以下文档内容回答问题。如果文档中没有相关信息，明确说明。

文档内容：
{context}

问题：{question}

回答："""
    )
    model = ChatOpenAI(model="gpt-4", temperature=0)

    return (
        RunnableParallel({
            "context": vectorstore.as_retriever() | (lambda docs: "\n\n".join(d.page_content for d in docs)),
            "question": RunnablePassthrough(),
        })
        | prompt
        | model
        | StrOutputParser()
    )


def build_agent():
    """构建主 Agent"""
    tools = [search_knowledge_base, query_database, get_table_schema]

    prompt = ChatPromptTemplate.from_messages([
        ("system", """你是一个企业智能助手，可以回答文档相关的问题和查询数据库。

可用工具：
- search_knowledge_base: 搜索公司知识库文档
- query_database: 执行 SQL 查询获取业务数据
- get_table_schema: 查看数据库表结构

注意：
1. 回答文档相关问题前，先搜索知识库
2. 查询数据库前，先用 get_table_schema 了解表结构
3. 只执行 SELECT 查询
4. 如果信息不足，明确告知用户"""),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ])

    model = ChatOpenAI(model="gpt-4", temperature=0)
    agent = create_tool_calling_agent(model, tools, prompt)

    return AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        max_iterations=10,
        handle_parsing_errors=True,
    )


#​ ===================== 初始化与使用 =====================

#​ 初始化向量存储（实际使用时需要先加载文档）
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(
    persist_directory="./knowledge_base",
    embedding_function=embeddings,
)


class SmartAssistant:
    """智能助手主类"""

    def __init__(self):
        self.agent = build_agent()
        self.chat_history = []

    def chat(self, user_input: str) -> str:
        """处理用户输入"""
        result = self.agent.invoke({
            "input": user_input,
            "chat_history": self.chat_history,
            "agent_scratchpad": [],
        })

        # 更新对话历史
        self.chat_history.append(HumanMessage(content=user_input))
        self.chat_history.append(HumanMessage(content=result["output"]))

        # 保持历史在合理长度
        if len(self.chat_history) > 20:
            self.chat_history = self.chat_history[-20:]

        return result["output"]


#​ 使用示例
if __name__ == "__main__":
    assistant = SmartAssistant()

    # 场景 1：文档问答
    print(assistant.chat("公司的年假政策是什么？"))

    # 场景 2：数据查询
    print(assistant.chat("帮我查一下上个月的总订单金额"))

    # 场景 3：混合查询
    print(assistant.chat("哪些用户的订单金额超过了 10000 元？给我他们的邮箱"))
```

## 总结

LangChain 通过 LCEL 提供了声明式的链编排语法，将 Model I/O、RAG、Agent、Memory 等模块统一在 Runnable 接口下。核心学习路径：

1. **掌握 LCEL**：管道语法 `|` 和 `RunnableParallel`/`RunnablePassthrough` 是基础
2. **理解 Model I/O**：Prompt Template + ChatModel + Output Parser 三件套
3. **构建 RAG 链**：Document Loader → Splitter → VectorStore → Retriever → Chain
4. **开发 Agent**：定义工具（`@tool`）、绑定模型、配置执行器
5. **生产部署**：Callback 追踪 + LangSmith 调试 + LangServe 部署

框架选择上，LangChain 的优势在于生态丰富和社区活跃，但抽象层较多。对于简单场景，直接使用 API SDK 可能更高效；对于复杂场景，LangChain 的模块化设计能有效提升开发效率。
