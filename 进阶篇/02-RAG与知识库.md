# 02-RAG与知识库

> 读完本文你将了解：什么是 RAG（Retrieval-Augmented Generation）、为什么需要 RAG（它解决了什么问题）、RAG 的基本流程（文档切分→向量化→检索→生成）以及一个完整可运行的代码示例
> 注：本文代码示例基于 OpenAI Embeddings API + ChromaDB，你可以在本地环境直接运行。

---

## 你可能遇到过的问题

你试过把产品文档、公司内部知识丢给 AI，尝试搭建一个「企业知识库问答系统」，但遇到了这些问题：

- **AI 回答的内容听起来很有道理，但细节全是错的**——它把 2023 年的产品定价说成了 2025 年的版本，你没法信任它给出的任何具体信息
- **你尝试把知识库内容塞进 prompt**——结果 token 限制不够用，几千字的文档根本装不下，更不用说几万页的企业知识库
- **你试了 fine-tuning，发现效果不理想**——模型确实学到了知识的「风格」，但问到具体细节还是瞎编，而且每次新增知识都要重新训练
- **用户问了一个和内部文档高度相关的问题，AI 给出了一篇通用但答非所问的回复**——它对你们公司特有的产品术语、内部缩写完全没概念

这些问题的共同答案是一个已经很成熟的方案：**RAG（Retrieval-Augmented Generation，检索增强生成）**。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| RAG 就是把文档塞进 prompt 里 | RAG 的核心是「检索相关片段，而不是塞入全部文档」。它通过向量相似度找到最相关的几个段落，大幅降低 token 消耗 |
| RAG 需要复杂的基础设施才能用 | 2026 年用 ChromaDB、Pinecone 这类向量数据库，几十行代码就能跑通一个最小原型 |
| RAG 能完全消除 AI 幻觉 | RAG 显著减少了幻觉，但不能完全消除。如果检索到的片段不相关，或者用户问题超出知识库范围，AI 还是会编 |
| Fine-tuning 和 RAG 二选一 | 两者是互补关系。RAG 管「知识获取」，fine-tuning 管「行为风格和格式偏好」，可以组合使用 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| 想让 AI 基于公司内部文档回答问题 | 用 ChromaDB 或 Pinecone 搭建 RAG 流程，把文档切分后向量化存储，查询时检索最相关片段 |
| 文档太长放不进 prompt | RAG 会自动选择最相关的 3~5 个片段，而不是全量塞入，token 消耗可控 |
| 文档更新后 AI 还在用旧知识 | 只需要更新向量数据库中的文档片段，不需要重新训练模型 |
| 不知道从哪入手实现 RAG | 从「文档切分→embeddings→向量存储→检索→生成」这个五步流程开始，下面有完整代码示例 |

---

## 核心概念：RAG 的基本流程

RAG 的流程可以分为清晰的 5 个环节。理解这个 pipeline 是搭建任何 RAG 系统的基础。

```
原始文档 → [文档切分] → 文本块(Chunks)
每个文本块 → [Embedding 模型] → 向量(高维浮点数数组)
所有向量 → [向量数据库] → 可检索的索引
用户提问 → [Embedding 同一模型] → 问题向量 → [相似度检索] → 最相关的 Top-K 块
相关片段 + 原始问题 → [LLM] → 最终答案
```

### 步骤一：文档切分（Chunking）

原始文档不能整篇塞进去。你需要按一定策略切成小块。

```python
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """将长文本切分成重叠的片段。"""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

# 示例
document = "这是一篇很长的产品文档..." * 100
chunks = chunk_text(document)
print(f"切分成 {len(chunks)} 个片段，每个约 500 字符")
```

切分策略会影响检索质量：片段太短（<100 字符）丢失上下文；太长（>1000 字符）语义混杂，检索精度下降。overlap 保证切分边界处不会丢失关键信息。

### 步骤二：向量化（Embedding）

把文本块转换成向量。语义相似的文本在向量空间中的距离更近。

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-你的API Key",
    base_url="https://api.siliconflow.cn/v1"  # 国内直接用，无需特殊网络
)

def embed_texts(texts: list[str]) -> list[list[float]]:
    """用 Embedding 模型将文本转为向量。"""
    response = client.embeddings.create(
        model="BAAI/bge-large-zh-v1.5",  # 国内可用: SiliconFlow 提供的 Embedding 模型，1024 维
        input=texts
    )
    return [data.embedding for data in response.data]
```

### 步骤三：存入向量数据库

这里用 ChromaDB，一个轻量级纯 Python 向量数据库。

```python
import chromadb

# 初始化 ChromaDB（ChromaDB 0.5+ 新 API）
chroma_client = chromadb.PersistentClient(path="./knowledge_base")

# 创建或获取集合
collection = chroma_client.get_or_create_collection(
    name="product_docs",
    metadata={"hnsw:space": "cosine"}  # 使用余弦相似度
)

# 把文档块和对应的向量存入
chunks = chunk_text(document)
embeddings = embed_texts(chunks)
ids = [f"chunk_{i}" for i in range(len(chunks))]

collection.add(
    embeddings=embeddings,
    documents=chunks,
    ids=ids
)
```

### 步骤四：检索（Retrieval）

用户提问时，把问题转成向量，在数据库中找最相似的 Top-K 片段。

```python
def retrieve(query: str, k: int = 3) -> list[str]:
    """检索与查询最相关的 k 个文档片段。"""
    query_embedding = embed_texts([query])[0]
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=k
    )
    return results["documents"][0]

# 示例
question = "我们产品的定价策略是什么？"
relevant_chunks = retrieve(question, k=3)
```

### 步骤五：生成（Generation）

把检索到的片段和原始问题拼在一起，让 LLM 基于上下文生成答案。

```python
def generate_answer(question: str, context_chunks: list[str]) -> str:
    """基于检索到的文档片段生成回答。"""
    context = "\n\n".join(context_chunks)
    prompt = f"""基于以下文档内容回答问题。如果你在文档中找不到相关信息，请明确说明不知道，不要编造答案。

文档内容：
{context}

问题：{question}

回答："""
    
    response = client.chat.completions.create(
        model="deepseek-ai/DeepSeek-V4-Flash",  # 可换成任意 SiliconFlow 上的模型
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3
    )
    return response.choices[0].message.content

# 完整 RAG 调用
answer = generate_answer(question, relevant_chunks)
print(answer)
```

---

## 把流程串起来：完整的 RAG 类

```python
class SimpleRAG:
    def __init__(self, collection_name: str = "default"):
        self.client = OpenAI(
            api_key="sk-你的API Key",
            base_url="https://api.siliconflow.cn/v1"  # 国内直接用，无需特殊网络
        )
        self.chroma = chromadb.PersistentClient(path="./vector_db")
        self.collection = self.chroma.get_or_create_collection(
            name=collection_name
        )
    
    def add_documents(self, documents: list[str]):
        chunks = []
        for doc in documents:
            chunks.extend(self._chunk_text(doc))
        embeddings = self._embed(chunks)
        ids = [f"doc_{i}" for i in range(len(chunks))]
        self.collection.add(embeddings=embeddings, documents=chunks, ids=ids)
    
    def query(self, question: str, k: int = 3) -> str:
        relevant = self._retrieve(question, k)
        return self._generate(question, relevant)
    
    def _chunk_text(self, text: str, size=500, overlap=50):
        chunks = []
        start = 0
        while start < len(text):
            chunks.append(text[start:start+size])
            start += size - overlap
        return chunks
    
    def _embed(self, texts: list[str]):
        # 国内可用: SiliconFlow 的 BAAI/bge-large-zh-v1.5 等
        resp = self.client.embeddings.create(
            model="BAAI/bge-large-zh-v1.5", input=texts
        )
        return [d.embedding for d in resp.data]
    
    def _retrieve(self, query: str, k: int):
        q_emb = self._embed([query])[0]
        results = self.collection.query(query_embeddings=[q_emb], n_results=k)
        return results["documents"][0]
    
    def _generate(self, question: str, context: list[str]):
        ctx = "\n\n".join(context)
        prompt = f"基于以下内容回答问题。不知道就说不知道：\n{ctx}\n\n问题：{question}"
        resp = self.client.chat.completions.create(
            model="deepseek-ai/DeepSeek-V4-Flash",  # 可换成任意 SiliconFlow 上的模型
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )
        return resp.choices[0].message.content

# 使用
rag = SimpleRAG()
rag.add_documents(["<你的产品文档或者知识库>"])
answer = rag.query("我们支持哪些支付方式？")
print(answer)
```

---

## 进阶方向

RAG 做好了只是第一步。生产级的 RAG 还需要考虑：

- **Hybrid Search**：结合向量相似度和关键词匹配（BM25），兼顾语义和精确匹配
- **Reranking**：检索出 Top-20 后用交叉编码器重排序，选出最精准的 Top-3
- **Query Rewriting**：用户问题模糊时，先让 LLM 改写成更清晰的检索语句
- **Multi-Hop RAG**：一个问题可能需要在知识库中跳转多次，比如「A 产品的作者写了哪些书？」
- **分块策略优化**：根据文档结构（标题、段落、表格）做语义切分，而不是固定 token 数

---

## 要点总结

- RAG 通过「检索+生成」的方式，让 LLM 能基于外部知识回答问题，解决了知识时效性和幻觉问题
- 核心流程：文档切分 → Embedding 向量化 → 存入向量数据库 → 检索相似片段 → LLM 生成
- ChromaDB + OpenAI Embeddings 是最快上手的组合，50 行代码就能跑通
- 切分策略（chunk size + overlap）直接影响检索质量，需要根据文档类型调优
- RAG 和 Fine-tuning 是互补关系，不是替代关系
- 生产级 RAG 需要额外处理：混合检索、重排序、Query Rewriting 等问题
