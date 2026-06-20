# RAG 与 GraphRAG

> 涵盖：检索增强生成（RAG）、GraphRAG、知识图谱驱动的检索等。

## Q: GraphRAG 和传统 RAG 有什么区别？

### 传统 RAG

**流程**：文档 → 分块 → Embedding → 存入向量库 → 用户查询 → Embedding → 相似度检索 Top-K → 拼接上下文 → LLM 生成答案。

**优势**：
- 实现简单，技术栈成熟（LangChain、LlamaIndex 等）
- 对「单跳事实查询」效果好（"X 是什么？"、"Y 怎么配置？"）
- 索引成本低，chunk 直接 embedding 即可

**局限**：
- **无法多跳推理**：如果答案散落在多个不相关的 chunk 里，相似度检索很难同时召回
- **无法全局概括**：问"这批文档的主要主题是什么？"时，Top-K 检索只能返回局部片段，无法给出全局视图
- **语义孤岛**：chunk 之间没有关联关系，丢失了文档间的结构信息

### GraphRAG

**流程**：文档 → 分块 → LLM 抽取实体和关系 → 构建知识图谱 → 社区检测（Leiden 算法）→ 生成社区摘要 → 查询时在图上检索 → LLM 生成答案。

**优势**：
- **多跳推理**：通过图的边（关系）可以从实体 A 跳到 B 再到 C，即使它们在原文中距离很远
- **全局概括**：社区摘要天然覆盖全局主题，能回答"有哪些主要趋势"类问题
- **结构化知识**：实体-关系-实体三元组保留了文档的语义结构

**局限**：
- **索引成本高**：每个 chunk 都需要 LLM 调用做实体/关系抽取，token 消耗是传统 RAG 的数十倍
- **延迟更高**：图遍历 + 社区摘要检索比向量相似度搜索慢
- **质量依赖抽取**：如果实体/关系抽取不准确，整个图的质量都会下降

### 对比总结

| 维度 | 传统 RAG | GraphRAG |
|---|---|---|
| 检索方式 | 向量相似度（语义匹配） | 图遍历 + 社区摘要 |
| 适合的问题类型 | 单跳事实查询 | 多跳推理、全局概括 |
| 索引成本 | 低（只需 embedding） | 高（大量 LLM 调用） |
| 数据结构 | 扁平的 chunk 列表 | 实体-关系知识图谱 |
| 文档间关联 | 无 | 通过共享实体和关系连接 |
| 实现复杂度 | 低 | 高 |

**基准数据**（来自 Microsoft Research 和第三方评测）：
- GraphRAG 在复杂查询的综合性测试中对标准 RAG 达到 **96% 胜率**
- 图结构集成可提升答案精确度 **~35%**
- 多跳推理深度提升 **4.5%**
- 但延迟约为传统 RAG 的 **2.3 倍**（传统 RAG <1s，GraphRAG 5-10s）
- 索引成本 **5-10 倍** LLM 调用量
- 当查询涉及实体数 >5 时，传统 RAG 准确率急剧下降趋近零，Graph 检索保持稳定

**选择建议**：如果场景以精确查找为主且预算有限，传统 RAG 足够；如果需要跨文档推理、全局分析或数据量大且实体关系密集，GraphRAG 值得投入。两者也可以混合使用——向量检索做初筛，图检索做深度推理。

- 标签: `rag`, `graphrag`, `comparison`, `retrieval`
- 记录于: 2026-06-18

## Q: GraphRAG 的技术细节是什么？

### Microsoft GraphRAG 的完整管线

GraphRAG 由微软研究院提出（2024），核心思路是**先把文档转化为知识图谱，再在图上做检索**。

**索引阶段（离线）**：

```
文档 → ① 分块 → ② 实体/关系抽取 → ③ 图构建 → ④ 社区检测 → ⑤ 社区摘要
```

1. **分块（Chunking）**：将文档切分为固定大小的文本块（通常 300-600 tokens），与传统 RAG 一致。

2. **实体与关系抽取（Entity & Relationship Extraction）**：对每个 chunk 调用 LLM，提取：
   - 实体（人名、组织、概念、技术术语等）及其描述
   - 实体间的关系及关系描述
   - 输出格式为三元组：`(实体A, 关系, 实体B)`
   - 这是成本最高的步骤——每个 chunk 都需要一次 LLM 调用。

3. **图构建（Graph Construction）**：将所有三元组合并为一张知识图谱。同名实体合并（实体消歧），关系权重按出现频次累加。

4. **社区检测（Community Detection）**：使用 **Leiden 算法**对图做层次化社区划分。Leiden 是 Louvain 的改进版，能发现图中紧密连接的子群（社区）。支持多层级——大社区包含小社区，形成树状结构。

5. **社区摘要（Community Summarization）**：对每个社区（每个层级），用 LLM 生成一段文本摘要，描述该社区包含的实体、关系和核心主题。这些摘要是后续全局查询的基础。

**查询阶段（在线）**：

GraphRAG 提供两种查询模式：

**Local Search（局部搜索）**：
- 从查询中识别相关实体 → 在图中找到这些实体 → 扩展到邻居节点 → 收集相关的实体描述、关系、原始文本块 → 拼接为上下文 → LLM 生成答案。
- 适合：具体的、有明确实体的问题（"张三和李四是什么关系？"）
- 类似传统 RAG，但检索维度从"语义相似的 chunk"变为"图上相关的实体子图"。

**Global Search（全局搜索）**：
- 收集所有社区的摘要（通常从最高层级开始）→ Map 阶段：每个社区摘要独立回答查询并给出相关性评分 → Reduce 阶段：将所有有相关性的社区回答合并 → LLM 生成最终答案。
- 适合：宏观的、无特定实体的问题（"这些文档的主要主题是什么？"、"有哪些共同趋势？"）
- 这是 GraphRAG 相比传统 RAG 最大的差异化能力。

**成本分析**：
- 索引成本：每 1M token 文档约需要 3-10× 的 LLM token 消耗（抽取 + 摘要）
- 查询成本：Local Search 与传统 RAG 相当；Global Search 更贵（需要遍历所有社区摘要）
- 适合数据不频繁更新、但查询模式复杂的场景

**改进：动态社区选择（Dynamic Community Selection）**：
原版 Global Search 的问题是对所有社区摘要做 map-reduce，很多不相关的社区浪费 token。改进方案：
1. 从根社区开始，用 LLM（便宜模型如 GPT-4o-mini）评估每个社区与查询的相关性
2. 不相关的社区及其整棵子树直接剪枝
3. 相关的社区向下递归展开
4. 最终只对筛选后的社区做 map-reduce

效果：Level 1 平均成本降低 **77%**，质量持平。Level 3 深度搜索在综合性和多样性上获得显著质量提升，仅增加 34% 成本。

**DRIFT Search（Local + Global 混合）**：
三阶段：① Primer——用 HyDE（假设文档嵌入）对 Top-K 语义相关的社区报告做轻量全局搜索，产生初步答案 + 后续问题。② Follow-Up——对每个后续问题执行 Local Search（默认迭代 2 次）。③ Output——按相关性排序合并结果。测试显示 DRIFT 在综合性上 **78%** 胜过纯 Local Search，多样性上 **81%** 胜出。

**与传统 RAG 的混合方案**：实践中常将两者结合——向量检索快速召回候选 chunk，再用图结构做二次排序和多跳扩展，平衡成本与效果。

- 标签: `graphrag`, `microsoft`, `knowledge-graph`, `community-detection`, `leiden`, `drift-search`
- 记录于: 2026-06-18

## Q: RAG 流程中为什么要引入父子索引？BM25 和向量检索如何融合？

### 父子索引（Parent-Child Indexing）

#### 核心矛盾

RAG 中的文档分块存在**检索粒度与上下文完整性的矛盾**：

```
小 chunk（200 token）：
  ✅ 检索精准——embedding 向量语义集中，相似度计算准确
  ❌ 上下文不足——送给 LLM 的内容太碎片，LLM 无法理解完整语境

大 chunk（2000 token）：
  ✅ 上下文完整——LLM 能看到足够的背景信息
  ❌ 检索模糊——embedding 向量混合了多个语义，相似度计算被稀释
```

**父子索引的解决思路：用小块检索、返回大块。**

#### 工作原理

```
原始文档
  ↓ 分块
父 chunk（大，800-2000 token）
  ├─ 子 chunk 1（小，200 token）→ 生成 embedding → 存入向量库
  ├─ 子 chunk 2（小，200 token）→ 生成 embedding → 存入向量库
  └─ 子 chunk 3（小，200 token）→ 生成 embedding → 存入向量库

检索时:
  Query → 向量检索 → 命中子 chunk 2
                      ↓ 通过 parent_id 回溯
                    返回父 chunk（包含子 1+2+3 的完整上下文）
```

```python
from langchain.retrievers import ParentDocumentRetriever

# 两种分块器
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,          # 子 chunk 的 embedding 存这里
    docstore=InMemoryStore(),         # 父 chunk 存这里（原文）
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 检索：用子 chunk embedding 匹配，返回父 chunk 原文
results = retriever.get_relevant_documents("什么是注意力机制？")
# 返回的是完整的父 chunk，而非碎片化的子 chunk
```

#### 变体

| 变体 | 思路 | 适用场景 |
|---|---|---|
| **Sentence Window** | 检索单句，返回前后 N 句窗口 | 内容结构松散 |
| **父子 chunk** | 检索小 chunk，返回父 chunk | 内容有层次结构 |
| **摘要索引** | 对大 chunk 生成摘要做 embedding，命中后返回原文 | 文档很长时 |

### BM25 与向量检索的融合

#### 为什么要融合？

两种检索方法各有盲区：

| 维度 | BM25（词频统计） | 向量检索（语义匹配） |
|---|---|---|
| **擅长** | 精确关键词匹配、专有名词、编号 | 语义相似、同义词、跨语言 |
| **短板** | 无法理解同义词和语义 | 对精确关键词不敏感 |
| **典型失败** | "LLM 推理优化" 查不到 "大模型加速" | "订单号 ABC123" 查不到精确匹配 |
| **速度** | 极快（倒排索引） | 较快（ANN 检索） |

```
用户查询: "Transformer attention 机制的 PyTorch 实现"

BM25 会命中:   包含 "Transformer"、"attention"、"PyTorch" 这些词的文档
向量检索会命中: 语义相关的文档（可能标题是"自注意力的代码实现"，不含原关键词）
融合后:        两种结果互补，覆盖率更高
```

#### 融合方法一：RRF（Reciprocal Rank Fusion）

最常用、最简单的融合方法。**不需要归一化分数**，只使用排名：

```python
def reciprocal_rank_fusion(result_lists: list[list], k=60) -> list:
    """
    result_lists: 多个检索器返回的排序结果列表
    k: 常数，控制高排名文档的权重（通常 60）
    """
    scores = {}
    for result_list in result_lists:
        for rank, doc in enumerate(result_list):
            if doc.id not in scores:
                scores[doc.id] = 0
            scores[doc.id] += 1.0 / (k + rank + 1)
    
    # 按融合分数降序排列
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)

# 使用
bm25_results = bm25_search(query, top_k=20)
vector_results = vector_search(query, top_k=20)
fused = reciprocal_rank_fusion([bm25_results, vector_results])
```

**RRF 的数学直觉**：排名第 1 的文档得分 `1/(60+1) ≈ 0.016`，排名第 10 的得分 `1/(60+10) ≈ 0.014`。如果一个文档在两个列表中都排前几名，它的融合分数会远高于只在一个列表中排名靠前的文档。

#### 融合方法二：加权分数融合

当两个检索器的分数可比（归一化后）时，直接加权：

```python
def weighted_fusion(bm25_results, vector_results, alpha=0.5):
    """alpha: 向量检索的权重"""
    # 分数归一化到 [0, 1]
    bm25_scores = normalize(bm25_results)
    vector_scores = normalize(vector_results)
    
    all_docs = set(bm25_scores.keys()) | set(vector_scores.keys())
    fused = {}
    for doc_id in all_docs:
        bm25_s = bm25_scores.get(doc_id, 0)
        vector_s = vector_scores.get(doc_id, 0)
        fused[doc_id] = (1 - alpha) * bm25_s + alpha * vector_s
    
    return sorted(fused.items(), key=lambda x: x[1], reverse=True)
```

**alpha 的调参经验**：
- `alpha=0.7`（偏向量）：通用问答、语义搜索
- `alpha=0.3`（偏 BM25）：技术文档、含大量专有名词
- `alpha=0.5`：默认起点

#### 工程实现：Elasticsearch + 向量库

```python
# 方案 A：Elasticsearch 8.x 原生混合搜索
response = es.search(
    index="documents",
    query={
        "bool": {
            "should": [
                {"match": {"content": query}},  # BM25
                {"knn": {                        # 向量检索
                    "field": "embedding",
                    "query_vector": embed(query),
                    "k": 20,
                }},
            ]
        }
    },
    rank={"rrf": {"window_size": 100, "rank_constant": 60}}  # RRF 融合
)

# 方案 B：分开检索 + 自行融合
bm25_hits = es.search(query={"match": {"content": query}}, size=20)
vector_hits = milvus.search(embed(query), top_k=20)
final = reciprocal_rank_fusion([bm25_hits, vector_hits])
```

### 父子索引 + 混合检索的组合

```
Query → BM25 检索子 chunk → Top-K₁
      → 向量检索子 chunk   → Top-K₂
               ↓
        RRF 融合 → Top-K 子 chunk
               ↓
        通过 parent_id 回溯 → 去重 → 父 chunk 列表
               ↓
        送入 LLM 生成回答
```

这是当前生产级 RAG 系统中**最常见的完整检索架构**。

- 标签: `rag`, `parent-child-index`, `bm25`, `hybrid-search`, `rrf`, `retrieval`
- 记录于: 2026-06-20

## Q: Elasticsearch 在 RAG 系统中的作用？如何优化检索性能？

### ES 在 RAG 中的三个角色

```
用户 Query
  ↓
┌──────────────────────────────────────────┐
│            Elasticsearch                  │
│                                           │
│  角色 1: BM25 关键词检索                   │
│  角色 2: 向量检索 (dense_vector)           │
│  角色 3: 混合检索 (BM25 + kNN + RRF)      │
│                                           │
└──────────────────────────────────────────┘
  ↓ Top-K 文档
  ↓ 送入 LLM 生成回答
```

### 角色 1：BM25 关键词检索

ES 的核心能力——基于倒排索引的全文搜索：

```json
// 索引文档
PUT /rag_documents/_doc/1
{
  "content": "Redis 支持五种数据结构：String、Hash、List、Set、Sorted Set",
  "title": "Redis 数据结构",
  "source": "redis-docs",
  "chunk_id": "chunk_001",
  "parent_id": "doc_001"
}
```

```json
// BM25 检索
GET /rag_documents/_search
{
  "query": {
    "match": {
      "content": {
        "query": "Redis 数据结构有哪些",
        "analyzer": "ik_smart"  // 中文分词器
      }
    }
  },
  "size": 10
}
```

**BM25 擅长**：精确关键词匹配、专有名词、编号/代码。
**BM25 不擅长**：同义词（"LLM" vs "大模型"）、语义相似但无共同词。

### 角色 2：向量检索（ES 8.x+）

ES 8.x 原生支持 dense_vector 字段和 kNN 检索：

```json
// 索引时存储 embedding
PUT /rag_documents/_doc/1
{
  "content": "Redis 支持五种数据结构...",
  "embedding": [0.12, -0.34, 0.56, ...]  // 1536 维向量
}
```

```json
// kNN 向量检索
GET /rag_documents/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.15, -0.30, 0.58, ...],  // query 的 embedding
    "k": 10,
    "num_candidates": 100  // HNSW 搜索范围
  }
}
```

### 角色 3：混合检索（最佳实践）

ES 8.x 原生支持 BM25 + kNN + RRF 融合：

```json
GET /rag_documents/_search
{
  "query": {
    "match": {
      "content": "Redis 缓存策略"
    }
  },
  "knn": {
    "field": "embedding",
    "query_vector": [0.15, -0.30, ...],
    "k": 20,
    "num_candidates": 100
  },
  "rank": {
    "rrf": {
      "window_size": 100,
      "rank_constant": 60
    }
  },
  "size": 10
}
```

一次请求同时执行 BM25 和向量检索，用 RRF 融合结果——这是 RAG 中最推荐的检索方式。

### 检索性能优化

#### 1. 索引设计优化

```json
// 针对 RAG 场景优化的 mapping
PUT /rag_documents
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "rag_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",   // 索引时细粒度分词
          "filter": ["lowercase", "synonym_filter"]
        },
        "rag_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart",       // 搜索时粗粒度分词
          "filter": ["lowercase", "synonym_filter"]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "LLM,大语言模型,大模型",
            "RAG,检索增强生成"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "rag_analyzer",
        "search_analyzer": "rag_search_analyzer"
      },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine"  // 或 dot_product
      },
      "source": {"type": "keyword"},   // 精确匹配字段用 keyword
      "parent_id": {"type": "keyword"},
      "created_at": {"type": "date"}
    }
  }
}
```

**关键优化点**：
- 索引时用细粒度分词（`ik_max_word`），搜索时用粗粒度分词（`ik_smart`），提高召回率
- 同义词过滤器解决"LLM"搜不到"大模型"的问题
- `keyword` 类型用于过滤字段（不做全文搜索的字段不要用 `text`）

#### 2. 查询优化

```json
// 优化前：全量字段搜索
{"query": {"match": {"content": "Redis 缓存"}}}

// 优化后：多字段加权 + 过滤 + 分页
{
  "query": {
    "bool": {
      "must": [
        {"multi_match": {
          "query": "Redis 缓存",
          "fields": ["title^3", "content"],  // title 权重 3 倍
          "type": "best_fields"
        }}
      ],
      "filter": [
        {"term": {"source": "official_docs"}},  // filter 不计算相关度，更快
        {"range": {"created_at": {"gte": "2025-01-01"}}}
      ]
    }
  },
  "_source": ["content", "title", "parent_id"],  // 只返回需要的字段
  "size": 10
}
```

**性能优化清单**：

| 优化 | 效果 | 原因 |
|---|---|---|
| **filter 代替 must** | 2-5x 快 | filter 不计算分数，可缓存 |
| **_source 过滤** | 减少网络传输 | 不返回 embedding 等大字段 |
| **keyword 代替 text** | 精确匹配更快 | 无需分词和相关度计算 |
| **routing** | 减少扫描分片数 | 同一数据源的文档路由到同一分片 |
| **预热缓存** | 首次查询更快 | `_search` 前先 `_warmers` |

#### 3. 向量检索优化

```json
// HNSW 参数调优
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "hnsw",
          "m": 16,              // 每个节点的邻居数（越大越准但越慢）
          "ef_construction": 200 // 构建时搜索范围（越大索引越慢但质量越高）
        }
      }
    }
  }
}
```

```json
// 查询时调 num_candidates
{
  "knn": {
    "field": "embedding",
    "query_vector": [...],
    "k": 10,
    "num_candidates": 200  // 越大越准但越慢，通常设为 k 的 5-20 倍
  }
}
```

| 参数 | 增大效果 | 减小效果 |
|---|---|---|
| **m** | 更准确，索引更大 | 更快，准确度下降 |
| **ef_construction** | 索引质量更高，构建更慢 | 构建更快 |
| **num_candidates** | 查询更准确，更慢 | 查询更快，可能漏召回 |

#### 4. 分片和硬件优化

```
经验法则:
- 单分片大小控制在 10-50 GB
- 分片数 = 数据总量 / 30GB（粗略估算）
- 向量检索对内存要求高：1M 条 1536 维向量 ≈ 6GB 内存
- 使用 SSD，向量检索的随机读很多
```

### ES vs 专用向量数据库

| 维度 | Elasticsearch | Milvus / Pinecone |
|---|---|---|
| **BM25** | 原生支持（核心能力） | 不支持 |
| **向量检索** | 8.x 支持（非核心，但够用） | 原生优化（更快更准） |
| **混合检索** | 原生 RRF | 需外部融合 |
| **运维成本** | 成熟生态，团队通常已有经验 | 新组件，需额外学习 |
| **适用场景** | 已有 ES 基础设施 + 混合检索需求 | 纯向量检索 + 超大规模 |

**实践建议**：如果团队已有 ES，直接用 ES 做混合检索是最快的 RAG 落地方案。向量检索性能略逊于 Milvus，但 BM25 + 向量 + RRF 的一站式体验是专用向量库做不到的。

- 标签: `elasticsearch`, `rag`, `bm25`, `vector-search`, `hybrid-search`, `performance-tuning`
- 记录于: 2026-06-20
