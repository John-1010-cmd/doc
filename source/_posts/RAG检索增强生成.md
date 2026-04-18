---
title: RAG检索增强生成
date: 2025-06-10
updated : 2025-06-10
categories:
- AI
tags: 
  - AI
  - 实战
description: RAG 检索增强生成技术全解，从 Embedding 到向量检索到知识库构建。
series: AI 技术实践
series_order: 3
---

## RAG 的动机与核心架构

### 为什么需要 RAG

大语言模型虽然能力强大，但存在几个根本性的局限：

- **知识截止**：训练数据有截止日期，无法获取最新信息
- **幻觉问题**：模型会编造看似合理但事实错误的内容
- **领域缺失**：企业私有数据不在训练集中，模型无法回答
- **不可溯源**：模型输出无法追溯到具体来源

**RAG（Retrieval-Augmented Generation，检索增强生成）** 通过将外部知识库的检索结果注入模型的输入上下文，有效解决了以上问题。

### 核心架构

RAG 系统的基本流程分为三个阶段：

```
索引阶段：                    查询阶段：
┌──────────────┐             ┌──────────────┐
│  原始文档     │             │  用户查询     │
└──────┬───────┘             └──────┬───────┘
       │                            │
       ▼                            ▼
┌──────────────┐             ┌──────────────┐
│  文本切分     │             │  Query 编码   │
└──────┬───────┘             └──────┬───────┘
       │                            │
       ▼                            ▼
┌──────────────┐             ┌──────────────┐
│  Embedding    │             │  向量检索     │◄──────┐
└──────┬───────┘             └──────┬───────┘       │
       │                            │               │
       ▼                            ▼               │
┌──────────────┐             ┌──────────────┐       │
│  向量数据库   │◄────────────│  相似文档     │       │
└──────────────┘             └──────┬───────┘       │
                                    │               │
                                    ▼               │
                             ┌──────────────┐       │
                             │  LLM 生成     │───────┘
                             │  (带检索上下文) │
                             └──────────────┘
```

## 文本切分策略

文本切分（Chunking）是 RAG 的第一步，切分质量直接影响检索效果。

### 固定长度切分

最简单的策略，按固定字符数切分：

```python
def fixed_chunk(text, chunk_size=500, overlap=50):
    """固定长度切分，带重叠"""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap  # 重叠部分
    return chunks

# 示例
text = "这是一段很长的文档..." * 100
chunks = fixed_chunk(text, chunk_size=500, overlap=50)
print(f"切分结果：{len(chunks)} 个 chunk，每个约 500 字符")
```

优点：实现简单，chunk 大小一致。缺点：可能在句子或段落中间切断，破坏语义完整性。

### 语义切分

基于语义边界进行切分，保持语义的完整性：

```python
import re

def semantic_chunk(text, max_chunk_size=500):
    """基于语义边界的切分"""
    # 第一层：按段落分割
    paragraphs = text.split('\n\n')

    chunks = []
    current_chunk = ""

    for para in paragraphs:
        # 如果单个段落就超过最大长度，按句子再分
        if len(para) > max_chunk_size:
            sentences = re.split(r'[。！？\.\!\?]', para)
            for sent in sentences:
                if len(current_chunk) + len(sent) > max_chunk_size:
                    if current_chunk:
                        chunks.append(current_chunk.strip())
                    current_chunk = sent
                else:
                    current_chunk += sent
        else:
            if len(current_chunk) + len(para) > max_chunk_size:
                chunks.append(current_chunk.strip())
                current_chunk = para
            else:
                current_chunk += "\n\n" + para

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

### 递归字符切分

LangChain 提供的递归切分器是最常用的方案：

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", "。", "！", "？", ".", " ", ""],
    chunk_size=500,
    chunk_overlap=50,
    length_function=len,
)

chunks = splitter.split_text(document_text)
```

递归切分的工作方式：依次尝试不同的分隔符，从大到小（段落 > 行 > 句子 > 空格），确保在达到大小限制时使用最大的语义边界进行切分。

### 切分策略选择

| 策略 | 适用场景 | 优势 | 劣势 |
|------|---------|------|------|
| 固定长度 | 日志、表格等格式化数据 | 简单可控 | 语义可能断裂 |
| 语义切分 | 文章、报告等叙述性文本 | 语义完整 | chunk 大小不均 |
| 递归切分 | 通用场景 | 平衡语义和大小 | 需要调参 |
| 按文档结构 | Markdown、HTML 等结构化文档 | 保持层级 | 依赖文档格式 |

## Embedding 模型选择

Embedding 模型将文本转换为高维向量，是语义检索的基础。

### OpenAI Embedding

```python
import openai

def get_embedding(text, model="text-embedding-3-small"):
    """使用 OpenAI Embedding 模型"""
    response = openai.embeddings.create(
        input=text,
        model=model,
    )
    return response.data[0].embedding

# text-embedding-3-small: 1536 维，性价比高
# text-embedding-3-large: 3072 维，精度更高
```

### BGE 系列

北京智源研究院推出的 BGE 系列是优秀的开源 Embedding 模型：

```python
from sentence_transformers import SentenceTransformer

# BGE-large-zh-v1.5：中文场景首选
model = SentenceTransformer('BAAI/bge-large-zh-v1.5')
embeddings = model.encode(["查询文本", "候选文档1", "候选文档2"])

# BGE-M3：多语言、多功能
model_m3 = SentenceTransformer('BAAI/bge-m3')
```

### M3E 系列

M3E（Moka Massive Mixed Embedding）是中文场景的另一个优秀选择：

- 专为中文优化，在中文文本检索任务上表现优秀
- 支持 M3E-small / M3E-base / M3E-large 多种规模
- 开源可私有化部署，适合数据敏感场景

### 模型对比

| 模型 | 维度 | 中文能力 | 英文能力 | 是否开源 | 延迟 |
|------|------|---------|---------|---------|------|
| OpenAI text-embedding-3-small | 1536 | 良好 | 优秀 | 否 | 低 |
| BGE-large-zh | 1024 | 优秀 | 良好 | 是 | 中 |
| BGE-M3 | 1024 | 优秀 | 优秀 | 是 | 中 |
| M3E-large | 1024 | 优秀 | 一般 | 是 | 中 |

## 向量数据库对比

向量数据库是存储和检索 Embedding 向量的专用数据库。

### Milvus

Milvus 是高性能的分布式向量数据库：

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

# 连接 Milvus
connections.connect(host="localhost", port="19530")

# 创建 Collection
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
    FieldSchema(name="text", dtype=DataType.VARCHAR, max_length=65535),
    FieldSchema(name="metadata", dtype=DataType.JSON),
]
schema = CollectionSchema(fields, description="知识库文档")
collection = Collection("knowledge_base", schema)

# 创建索引
index_params = {
    "metric_type": "COSINE",
    "index_type": "IVF_FLAT",
    "params": {"nlist": 1024},
}
collection.create_index(field_name="embedding", index_params=index_params)
```

- 优势：高性能、分布式、丰富的索引类型、云原生
- 适用场景：大规模生产环境、亿级向量检索

### Pinecone

Pinecone 是全托管的向量数据库服务：

```python
from pinecone import Pinecone

pc = Pinecone(api_key="your-api-key")
index = pc.Index("knowledge-base")

# 插入向量
index.upsert(vectors=[
    {"id": "doc1", "values": embedding, "metadata": {"source": "wiki"}},
])

# 查询
results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True,
)
```

- 优势：零运维、自动扩缩容、实时更新
- 适用场景：快速上线、不想管理基础设施

### Weaviate

Weaviate 支持向量搜索和关键词搜索的混合模式：

```python
import weaviate

client = weaviate.connect_to_local()

# 混合搜索：向量 + 关键词
collection = client.collections.get("Documents")
response = collection.query.hybrid(
    query="机器学习入门",
    vector=query_embedding,
    alpha=0.7,  # 向量搜索权重，1-向量权重
    limit=5,
)
```

- 优势：内置混合搜索、支持多模态、GraphQL API
- 适用场景：需要混合检索的场景

### Chroma

Chroma 是轻量级的嵌入式向量数据库：

```python
import chromadb

client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("docs")

# 添加文档
collection.add(
    documents=["文档内容1", "文档内容2"],
    metadatas=[{"source": "file1"}, {"source": "file2"}],
    ids=["id1", "id2"],
)

# 查询
results = collection.query(
    query_texts=["搜索查询"],
    n_results=5,
)
```

- 优势：轻量、易集成、支持内存模式
- 适用场景：原型开发、小规模应用、单机部署

### Qdrant

Qdrant 是 Rust 实现的高性能向量数据库：

- 优势：高性能（Rust 实现）、丰富的过滤条件、支持量化
- 适用场景：需要高性能且对延迟敏感的场景

### 数据库选型建议

| 数据库 | 规模 | 部署复杂度 | 特色 |
|--------|------|-----------|------|
| Chroma | < 100 万 | 极低 | 快速原型 |
| Qdrant | < 1000 万 | 中 | 高性能 |
| Weaviate | < 1000 万 | 中 | 混合搜索 |
| Milvus | 亿级 | 高 | 分布式 |
| Pinecone | 任意 | 极低 | 全托管 |

## 检索策略

### 相似度搜索

最基本的检索策略，计算查询向量与文档向量的相似度：

```python
def similarity_search(query_embedding, doc_embeddings, top_k=5):
    """余弦相似度搜索"""
    import numpy as np

    # 归一化
    query_norm = query_embedding / np.linalg.norm(query_embedding)
    doc_norms = doc_embeddings / np.linalg.norm(doc_embeddings, axis=1, keepdims=True)

    # 计算余弦相似度
    similarities = np.dot(doc_norms, query_norm)

    # 返回 top_k 结果
    top_indices = np.argsort(similarities)[-top_k:][::-1]
    return [(i, similarities[i]) for i in top_indices]
```

常用的相似度度量：

- **余弦相似度**：衡量方向相似性，不受向量大小影响（最常用）
- **欧氏距离**：衡量绝对距离
- **内积（IP）**：适用于已归一化的向量

### 混合检索

混合检索结合向量搜索和关键词搜索，兼顾语义理解和精确匹配：

```python
from rank_bm25 import BM25Okapi
import jieba

class HybridRetriever:
    """混合检索器：向量 + BM25"""

    def __init__(self, vector_store, documents, embedding_model):
        self.vector_store = vector_store
        self.embedding_model = embedding_model

        # 构建 BM25 索引
        tokenized_docs = [list(jieba.cut(doc)) for doc in documents]
        self.bm25 = BM25Okapi(tokenized_docs)
        self.documents = documents

    def search(self, query, top_k=5, alpha=0.7):
        """
        alpha: 向量搜索权重 (0-1)
        1-alpha: BM25 搜索权重
        """
        # 向量搜索
        query_embedding = self.embedding_model.encode([query])
        vector_results = self.vector_store.search(query_embedding, top_k=top_k*2)
        vector_scores = {r['id']: r['score'] for r in vector_results}

        # BM25 搜索
        tokenized_query = list(jieba.cut(query))
        bm25_scores = self.bm25.get_scores(tokenized_query)
        bm25_results = {i: score for i, score in enumerate(bm25_scores)}

        # 分数归一化与融合
        all_ids = set(vector_scores.keys()) | set(bm25_results.keys())
        combined_scores = {}
        for doc_id in all_ids:
            v_score = normalize(vector_scores.get(doc_id, 0))
            b_score = normalize(bm25_results.get(doc_id, 0))
            combined_scores[doc_id] = alpha * v_score + (1 - alpha) * b_score

        # 排序返回
        sorted_results = sorted(combined_scores.items(), key=lambda x: x[1], reverse=True)
        return sorted_results[:top_k]
```

### 重排序（Reranker）

先通过粗检索获取候选集，再用精细模型重排序：

```python
from sentence_transformers import CrossEncoder

class Reranker:
    """使用 Cross-Encoder 进行重排序"""

    def __init__(self, model_name="BAAI/bge-reranker-large"):
        self.model = CrossEncoder(model_name)

    def rerank(self, query, documents, top_k=5):
        """对候选文档重排序"""
        # 构建查询-文档对
        pairs = [[query, doc] for doc in documents]

        # Cross-Encoder 打分（同时考虑 query 和 document）
        scores = self.model.predict(pairs)

        # 按分数排序
        ranked_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)
        return [(documents[i], scores[i]) for i in ranked_indices[:top_k]]

# 使用流程
retriever = HybridRetriever(vector_store, documents, model)
candidates = retriever.search(query, top_k=20)  # 粗检索
reranker = Reranker()
final_results = reranker.rerank(query, candidates, top_k=5)  # 精排序
```

Cross-Encoder vs Bi-Encoder：

- **Bi-Encoder**：分别编码 query 和 document，计算向量相似度。速度快，适合粗检索
- **Cross-Encoder**：同时编码 query-document 对，精度高但速度慢。适合精排序

## Chunk 优化

### 大小选择

Chunk 大小是影响 RAG 效果的关键参数：

| Chunk 大小 | 优势 | 劣势 | 适用场景 |
|-----------|------|------|---------|
| 小（128-256 字符） | 检索精确 | 上下文不完整 | FAQ、精确匹配 |
| 中（256-512 字符） | 平衡精确和上下文 | 通用场景 | 通用文档问答 |
| 大（512-1024 字符） | 上下文丰富 | 检索噪声多 | 长文档理解 |

### 重叠设置

重叠（Overlap）确保相邻 chunk 之间有信息衔接：

```python
# 推荐的重叠设置
chunk_configs = {
    "small": {"size": 200, "overlap": 20},   # 10% 重叠
    "medium": {"size": 400, "overlap": 40},   # 10% 重叠
    "large": {"size": 800, "overlap": 100},   # 12.5% 重叠
}
```

### 元数据

为每个 chunk 附加元数据，支持过滤和溯源：

```python
def chunk_with_metadata(document, doc_metadata):
    """切分文档并附加元数据"""
    chunks = semantic_chunk(document.text)

    enriched_chunks = []
    for i, chunk in enumerate(chunks):
        enriched_chunks.append({
            "content": chunk,
            "metadata": {
                "doc_id": document.id,
                "doc_title": document.title,
                "chunk_index": i,
                "chunk_total": len(chunks),
                "source": document.source,
                "created_at": document.created_at,
                "category": document.category,
            }
        })

    return enriched_chunks
```

元数据的典型用途：
- **过滤**：只检索特定类别或时间范围的文档
- **溯源**：在回答中引用具体来源
- **去重**：避免重复索引相同内容

## 高级 RAG

### 查询改写

优化用户的原始查询以提高检索质量：

```python
def query_rewrite(original_query, llm):
    """使用 LLM 改写查询"""

    prompt = f"""
你是一个查询优化专家。将用户的口语化查询改写为更适合检索的形式。

原始查询：{original_query}

要求：
1. 提取核心关键词
2. 补充同义词和相关术语
3. 生成 2-3 个不同角度的检索查询

输出格式（JSON）：
{% raw %}
{{
  "optimized_query": "优化后的主查询",
  "alternative_queries": ["备选查询1", "备选查询2"],
  "keywords": ["关键词1", "关键词2"]
}}
{% endraw %}
"""
    return llm.invoke(prompt)
```

### 多路召回

从多个不同维度进行检索并融合结果：

```python
class MultiRouteRetriever:
    """多路召回检索器"""

    def __init__(self, vector_retriever, keyword_retriever, knowledge_graph_retriever):
        self.retrievers = {
            "vector": vector_retriever,
            "keyword": keyword_retriever,
            "kg": knowledge_graph_retriever,
        }

    def retrieve(self, query, top_k=5):
        all_results = {}

        for name, retriever in self.retrievers.items():
            results = retriever.search(query, top_k=top_k)
            for doc_id, score in results:
                if doc_id in all_results:
                    all_results[doc_id]["scores"].append(score)
                else:
                    all_results[doc_id] = {
                        "scores": [score],
                        "sources": [name],
                    }

        # Reciprocal Rank Fusion (RRF)
        fused_scores = {}
        for doc_id, info in all_results.items():
            rrf_score = sum(1.0 / (60 + rank) for rank, _ in enumerate(info["scores"]))
            fused_scores[doc_id] = rrf_score

        return sorted(fused_scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
```

### Self-RAG

Self-RAG 让模型自主判断是否需要检索、检索结果是否有用：

```
Self-RAG 流程：

1. 给定问题，模型判断是否需要检索
   └─ 不需要 → 直接生成回答
   └─ 需要 → 生成检索查询

2. 检索相关文档

3. 对每个检索结果，模型评估：
   └─ 相关性评估（is_relevant?）
   └─ 如果相关，基于检索结果生成回答片段

4. 对生成的回答，模型自评估：
   └─ 是否得到检索支持（is_supported?）
   └─ 回答是否有效（is_useful?）

5. 如果不满意，回到步骤 1 重新检索
```

Self-RAG 的核心优势在于引入了自我反思机制，能够自适应地决定何时检索、何时生成。

## 评估指标

### 检索质量评估

| 指标 | 公式 | 含义 |
|------|------|------|
| 召回率（Recall） | 相关且检索到的 / 总相关文档 | 找到了多少相关内容 |
| 准确率（Precision） | 相关且检索到的 / 总检索文档 | 检索结果中有多少是相关的 |
| MRR | 1 / 第一个正确结果的排名 | 第一个正确结果出现的位置 |
| NDCG | 考虑排序位置的加权得分 | 排序质量 |

### 生成质量评估

- **Faithfulness（忠实度）**：回答是否忠实于检索到的文档内容，有无幻觉
- **Relevancy（相关性）**：回答是否与用户的问题相关
- **Completeness（完整性）**：回答是否覆盖了问题的所有方面
- **Conciseness（简洁性）**：回答是否简洁，没有冗余信息

```python
def evaluate_rag_system(test_cases, rag_pipeline):
    """评估 RAG 系统的整体性能"""
    results = {
        "retrieval_recall": [],
        "retrieval_precision": [],
        "faithfulness": [],
        "relevancy": [],
    }

    for case in test_cases:
        # 运行 RAG
        answer, retrieved_docs = rag_pipeline.query(case["question"])

        # 评估检索质量
        relevant_docs = set(case["relevant_doc_ids"])
        retrieved_ids = set(d["id"] for d in retrieved_docs)

        recall = len(relevant_docs & retrieved_ids) / len(relevant_docs)
        precision = len(relevant_docs & retrieved_ids) / len(retrieved_ids)

        results["retrieval_recall"].append(recall)
        results["retrieval_precision"].append(precision)

        # 使用 LLM 评估生成质量
        faith_score = evaluate_faithfulness(answer, retrieved_docs)
        relevancy_score = evaluate_relevancy(answer, case["question"])

        results["faithfulness"].append(faith_score)
        results["relevancy"].append(relevancy_score)

    # 汇总
    return {k: sum(v) / len(v) for k, v in results.items()}
```

## 完整实战：构建企业知识库问答系统

### 系统架构

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 文档管理  │───>│ 索引管道  │───>│ 向量存储  │    │  用户     │
│ (上传/解析)│    │ (切分/编码)│    │          │    │  接口     │
└──────────┘    └──────────┘    └────┬─────┘    └────┬─────┘
                                      │               │
                                      ▼               ▼
                                ┌──────────────────────┐
                                │      RAG 引擎         │
                                │  查询 → 检索 → 生成    │
                                └──────────────────────┘
```

### 完整代码实现

```python
"""
企业知识库问答系统 - 核心实现
"""
from dataclasses import dataclass
from typing import Optional
from sentence_transformers import SentenceTransformer
import chromadb


@dataclass
class Document:
    """文档数据类"""
    id: str
    title: str
    content: str
    source: str
    category: str


@dataclass
class SearchResult:
    """检索结果"""
    content: str
    score: float
    metadata: dict


class KnowledgeBase:
    """知识库管理"""

    def __init__(self, embedding_model_name="BAAI/bge-large-zh-v1.5"):
        self.embedding_model = SentenceTransformer(embedding_model_name)
        self.chroma_client = chromadb.PersistentClient(path="./kb_storage")
        self.collection = self.chroma_client.get_or_create_collection(
            name="knowledge_base",
            metadata={"hnsw:space": "cosine"},
        )

    def _chunk_document(self, doc: Document, chunk_size=400, overlap=40):
        """切分文档"""
        text = doc.content
        chunks = []
        start = 0
        idx = 0
        while start < len(text):
            end = min(start + chunk_size, len(text))
            chunks.append({
                "id": f"{doc.id}_chunk_{idx}",
                "content": text[start:end],
                "metadata": {
                    "doc_id": doc.id,
                    "title": doc.title,
                    "source": doc.source,
                    "category": doc.category,
                    "chunk_index": idx,
                },
            })
            start += chunk_size - overlap
            idx += 1
        return chunks

    def add_document(self, doc: Document):
        """添加文档到知识库"""
        chunks = self._chunk_document(doc)
        if not chunks:
            return

        # 生成 Embedding
        contents = [c["content"] for c in chunks]
        embeddings = self.embedding_model.encode(contents).tolist()

        # 存入 Chroma
        self.collection.add(
            ids=[c["id"] for c in chunks],
            embeddings=embeddings,
            documents=contents,
            metadatas=[c["metadata"] for c in chunks],
        )

    def search(self, query: str, top_k: int = 5,
               category: Optional[str] = None) -> list[SearchResult]:
        """检索相关文档"""
        query_embedding = self.embedding_model.encode([query]).tolist()

        # 构建过滤条件
        where_filter = None
        if category:
            where_filter = {"category": category}

        results = self.collection.query(
            query_embeddings=query_embedding,
            n_results=top_k,
            where=where_filter,
            include=["documents", "metadatas", "distances"],
        )

        search_results = []
        for i in range(len(results["ids"][0])):
            search_results.append(SearchResult(
                content=results["documents"][0][i],
                score=1 - results["distances"][0][i],  # 距离转相似度
                metadata=results["metadatas"][0][i],
            ))

        return search_results


class RAGEngine:
    """RAG 问答引擎"""

    def __init__(self, knowledge_base: KnowledgeBase, llm_client):
        self.kb = knowledge_base
        self.llm = llm_client

    def answer(self, question: str, top_k: int = 5) -> dict:
        """回答用户问题"""
        # 1. 检索相关文档
        search_results = self.kb.search(question, top_k=top_k)

        if not search_results:
            return {
                "answer": "抱歉，知识库中未找到相关信息。",
                "sources": [],
            }

        # 2. 构建上下文
        context_parts = []
        sources = []
        for i, result in enumerate(search_results):
            context_parts.append(f"[文档{i+1}]（来源：{result.metadata['title']}）\n{result.content}")
            sources.append({
                "title": result.metadata["title"],
                "source": result.metadata["source"],
                "score": result.score,
            })

        context = "\n\n".join(context_parts)

        # 3. 生成回答
        prompt = f"""基于以下检索到的文档内容回答用户问题。

要求：
- 仅基于提供的文档内容回答，不要编造信息
- 如果文档中没有相关信息，明确说明
- 引用具体的文档编号

检索到的文档：
{context}

用户问题：{question}

回答："""

        answer = self.llm.invoke(prompt)

        return {
            "answer": answer,
            "sources": sources,
        }


# 使用示例
if __name__ == "__main__":
    kb = KnowledgeBase()

    # 添加文档
    doc = Document(
        id="doc_001",
        title="公司年假政策",
        content="根据公司规定，入职满一年的员工享有10天年假...",
        source="hr_handbook.pdf",
        category="HR",
    )
    kb.add_document(doc)

    # 问答
    engine = RAGEngine(kb, llm_client=None)
    result = engine.answer("入职一年有多少天年假？")
    print(result["answer"])
    print(f"来源：{result['sources']}")
```

## 总结

RAG 是构建企业级 AI 应用的核心技术路线。从文本切分到向量检索到上下文生成，每个环节都有优化的空间。实践中的关键决策包括：选择合适的 Embedding 模型、确定最优的 Chunk 策略、设计检索策略（混合检索 + 重排序通常效果最好）。高级技术如查询改写和 Self-RAG 可以进一步提升系统质量，但也增加了系统复杂度。建议从简单的基线系统开始，通过评估驱动的方式逐步迭代优化。
