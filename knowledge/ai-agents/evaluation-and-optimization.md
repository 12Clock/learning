# 评估与优化

> 涵盖：Agent 效果评估、Badcase 定位、SFT 决策、LLM 推理优化。

## Q: Agent 系统的整体效果怎么评估？没有用户反馈时如何抽检？

### 评估体系设计

**离线评估（Offline Evaluation）**

构建 Golden Dataset——覆盖主要场景和边界情况的标注测试集：

```
evaluation/
  golden_set.jsonl              # 核心评测集
  ├─ 意图识别: 200+ cases       # 输入 → 期望意图
  ├─ 检索质量: 100+ cases       # Query → 期望召回文档
  ├─ 生成质量: 150+ cases       # 输入 → 参考答案
  └─ 端到端: 100+ cases         # 用户问题 → 完整期望输出
```

**核心指标**：

| 维度 | 指标 | 衡量什么 |
|---|---|---|
| 意图识别 | Precision / Recall / F1 | 意图分类准确性 |
| 检索质量 | Recall@K, MRR, NDCG | 检索是否找到正确文档 |
| 生成质量 | BLEU / ROUGE（参考） | 与参考答案的文本相似度 |
| 忠实度 | Faithfulness Score | 回答是否基于检索到的资料 |
| 幻觉率 | Hallucination Rate | 回答中无法溯源的陈述比例 |
| 完整性 | Completeness Score | 是否覆盖了问题的所有要点 |
| 任务完成 | Task Success Rate | 端到端任务是否成功 |
| 延迟 | P50 / P95 / P99 Latency | 响应速度 |

### 无用户反馈的抽检方案

**1. LLM-as-Judge（模型自动评估）**

用一个独立的强模型（如 GPT-4 / Claude Opus）评估目标模型的输出：

```python
judge_prompt = """
评估以下回答的质量，从 1-5 打分：
- 准确性：回答是否正确？
- 完整性：是否覆盖了问题的关键点？
- 相关性：是否切题？
- 幻觉：是否有编造的内容？

用户问题：{query}
检索资料：{retrieved_context}
模型回答：{answer}
"""
```

**注意**：LLM-as-Judge 自身也有偏见（偏好长回答、偏好自己的风格），需要定期用人工评估校准。

**2. 分层抽检策略**

不做均匀随机抽样，而是**按风险分层**：

- **高风险层（全量人工审核）**：涉及金融、医疗、法律的回答；用户投诉关联的 case
- **中风险层（20% 抽检）**：LLM-as-Judge 评分 <3 的 case；检索结果为空但仍生成了回答的 case
- **低风险层（5% 抽检）**：高分、高频的常见问题

```python
def sampling_strategy(case):
    if case.domain in HIGH_RISK_DOMAINS:
        return "full_review"
    if case.judge_score < 3 or case.retrieval_empty:
        return "20_percent_sample"
    return "5_percent_sample"
```

**3. 对比评估（Comparative Evaluation）**

不问"这个回答好不好"（绝对评价难），而是问"A 和 B 哪个更好"（相对评价容易）：
- 同一问题用不同 prompt/模型版本生成两个回答
- 让 LLM-as-Judge 或人工做 side-by-side 对比
- 统计 Win/Loss/Tie 比率

**4. 自动化 Canary 检测**

在日常流量中插入已知答案的"金丝雀"问题：
- 定期自动注入 canary query
- 检查回答是否符合预期
- 如果 canary 的准确率下降，说明系统整体质量可能在退化（模型更新、数据污染等）

### 评估的核心难题

生成式任务没有标准答案——同一个问题有多种正确的回答方式。解决方法是**评估维度分离**：不评"整体好不好"，而是分别评准确性、完整性、格式、安全性，每个维度用最合适的评估方法。

- 标签: `evaluation`, `llm-as-judge`, `quality-assurance`, `testing`
- 记录于: 2026-06-19

## Q: Badcase 出现时，怎么快速定位到哪个 Agent 环节出了问题？如何判断应该对哪个 Agent 做 SFT？

### 快速定位方法

**1. 全链路 Trace（最关键的基础设施）**

每次请求的完整执行链路都要有 trace log：

```
Request ID: req_abc123
├─ [00ms] 意图识别 → intent=code_qa, confidence=0.82
├─ [120ms] 查询改写 → rewritten="Python GIL 多线程限制"
├─ [350ms] 检索 → 3 docs retrieved, top_score=0.91
├─ [400ms] 生成 → response_length=256 tokens
└─ [450ms] 质量检查 → passed
```

**Badcase 发生时，按链路逐步排查**：
1. 意图识别对不对？→ 如果错了，问题在意图分类
2. 改写后的 query 合理吗？→ 如果跑偏了，问题在改写
3. 检索到的文档相关吗？→ 如果不相关，问题在检索（索引或 query）
4. 检索结果好但回答差？→ 问题在生成

**2. 逐层断点验证**

拿 Badcase 的输入，手动喂给每一层，用"已知正确的中间结果"替换上一层的输出：

```python
# 用 ground truth 意图替换意图识别结果，看后续是否正确
test_with_gt_intent = pipeline.run(
    query=badcase.query,
    override_intent="correct_intent",  # 跳过意图识别
)
```

如果替换后结果正确，说明问题在被替换的那一层。

**3. 分类统计**

积累 Badcase 后做分类统计，找出系统性问题：

| 故障类型 | 比例 | 对应环节 |
|---|---|---|
| 意图分错 | 25% | 意图识别 Agent |
| 检索未命中 | 30% | 检索 Agent / 索引质量 |
| 检索命中但答非所问 | 20% | 生成 Agent |
| 格式/安全问题 | 10% | 后处理 |
| 多轮上下文丢失 | 15% | 对话管理 |

### 如何判断该对哪个 Agent 做 SFT

**SFT 决策三问**：

1. **是系统性问题还是偶发问题？**
   - 同一类 Badcase 反复出现 → 系统性，值得 SFT
   - 偶发的随机错误 → 不值得 SFT，调参数或加 retry

2. **Prompt 优化是否已到极限？**
   - 先尝试 prompt 修改能否解决，如果修好了就不需要 SFT
   - 如果 prompt 改了一周还是不行 → 考虑 SFT

3. **有没有足够的标注数据？**
   - SFT 至少需要 500+ 高质量标注样本
   - 数据不够时，优先标数据而非硬上 SFT

**SFT 优先级排序**：

```
优先度 = 故障影响面 × 故障频率 × (1 - Prompt 可修复度)
```

- 高优先级：意图识别 Agent（处于链路头部，错了后面全错；且训练数据相对容易标注）
- 中优先级：生成 Agent（影响面大，但需要更多标注且效果不如意图识别确定）
- 低优先级：检索 Agent（通常通过改索引/改 embedding 模型就能解决，不需要 SFT 整个 Agent）

**实操建议**：优先 SFT 最前端的 Agent（如意图识别），因为前端错误会导致链路级联失败。一个 95% 准确率的意图识别 Agent 比一个 99% 准确率的生成 Agent 对系统整体效果的影响更大。

- 标签: `badcase`, `debugging`, `sft`, `trace`, `root-cause-analysis`
- 记录于: 2026-06-19

## Q: LLM 推理优化做了哪些工作？Continuous Batching、KV Cache、vLLM 等技术？

### 核心优化技术

**1. KV Cache（键值缓存）**

**原理**：Transformer 的自注意力机制中，生成第 N 个 token 时需要与前 N-1 个 token 做注意力计算。KV Cache 将已计算过的 Key 和 Value 矩阵缓存起来，新 token 只需计算自己的 Q，并与缓存的 K、V 做注意力，避免重复计算。

**效果**：将生成阶段的计算复杂度从 O(n²) 降到 O(n)（每步只算一个新 token 与所有历史的注意力）。

**代价**：显存消耗大——每个请求的 KV Cache 大小 ≈ `2 × num_layers × hidden_size × seq_len × dtype_bytes`。对于 70B 模型、4K context，单个请求的 KV Cache 约 2-4GB。

**2. Continuous Batching（连续批处理）**

**传统 Static Batching 的问题**：
- 一个 batch 中所有请求必须等最长的那个完成才能释放资源
- 短请求完成后 GPU 空转，利用率低（通常 <30%）

**Continuous Batching 原理**：
- 请求完成后立即从 batch 中移除，新请求立即加入
- 无需等待整个 batch 完成
- GPU 利用率从 <30% 提升到 >80%

```
Static:    [A████████] [B██] [C████] → 所有请求等 A 结束
Continuous: [A██][+B][A██B█][+C][A██B█C█][B done→+D][A██C█D█]...
```

**效果**：吞吐量提升 2-5 倍（同等硬件），延迟显著降低。

**3. PagedAttention（vLLM 核心创新）**

**问题**：KV Cache 需要连续显存，导致显存碎片化严重。预分配最大长度的 KV Cache 会浪费 60-80% 的显存。

**PagedAttention 原理**：借鉴操作系统虚拟内存的分页思想：
- KV Cache 不需要连续显存，而是分成固定大小的"页"（page）
- 用页表（page table）管理逻辑地址到物理地址的映射
- 按需分配页——只在实际产生 token 时才分配新页
- 显存利用率从 20-40% 提升到 >95%

**vLLM 的完整优化栈**：
- PagedAttention（核心）
- Continuous Batching
- 前缀缓存（Prefix Caching）——共享 system prompt 的 KV Cache
- 投机解码（Speculative Decoding）——小模型预测 + 大模型验证
- 量化支持（INT8/INT4/GPTQ/AWQ）

**4. 量化（Quantization）**

将模型权重从 FP16 压缩到更低精度：

| 量化方式 | 精度 | 显存节省 | 精度损失 | 适用场景 |
|---|---|---|---|---|
| FP16 | 基线 | 0% | 无 | 最高质量 |
| INT8 (W8A8) | 8-bit | ~50% | <1% | 生产推荐 |
| INT4 (GPTQ/AWQ) | 4-bit | ~75% | 2-5% | 资源受限 |
| GGUF (llama.cpp) | 混合 | 可变 | 可变 | CPU/边缘设备 |

**5. 投机解码（Speculative Decoding）**

- 用小模型（draft model）快速生成 K 个候选 token
- 大模型（target model）一次性验证这 K 个 token
- 验证通过的直接采纳，不通过的从不通过位置重新生成
- 效果：生成速度提升 2-3 倍，输出质量与大模型完全一致（数学证明）

### 线上部署关键指标

**吞吐量参考值**（单卡 A100 80GB）：

| 模型规模 | 量化 | 吞吐量（tokens/s） | 并发请求数 |
|---|---|---|---|
| 7B | FP16 | 2000-3000 | 64-128 |
| 13B | INT8 | 1000-1500 | 32-64 |
| 70B | INT4 | 300-500 | 8-16 |

**高峰吞吐**的关键不是单卡性能，而是：
- **水平扩缩容**：根据请求量动态增减 GPU 实例
- **请求队列管理**：优先级队列 + 速率限制
- **多级缓存**：语义缓存（相似 query 命中历史回答） + 前缀缓存（共享 system prompt）

### 部署框架选型

| 框架 | 核心特点 | 适用场景 |
|---|---|---|
| **vLLM** | PagedAttention、生态最好、支持最广 | 通用推理服务 |
| **TensorRT-LLM** | NVIDIA 深度优化、极致性能 | NVIDIA GPU 场景 |
| **SGLang** | 编译优化、RadixAttention | 复杂 pipeline |
| **llama.cpp** | CPU 推理、GGUF 量化 | 边缘/本地部署 |

- 标签: `llm-inference`, `kv-cache`, `continuous-batching`, `vllm`, `quantization`, `optimization`
- 记录于: 2026-06-19
