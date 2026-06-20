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

## Q: Agentic CPT、SFT、RL 三阶段训练流程分别是什么？为什么 SFT 时要 mask observation tokens？

### 概览：三阶段训练 Agent 专用大模型

通用 LLM 并不天然擅长 Agent 任务（多步推理、工具调用、环境交互）。要训练一个"Agent 原生"的大模型，通常经过三个阶段：

```
基座模型 (Base LLM)
  ↓ 阶段 1: Agentic CPT (Continual Pre-Training)
注入 Agent 领域知识
  ↓ 阶段 2: Agentic SFT (Supervised Fine-Tuning)
学习 Agent 行为模式
  ↓ 阶段 3: Agentic RL (Reinforcement Learning)
优化决策策略
  ↓
Agent 专用模型
```

### 阶段 1：Agentic CPT（持续预训练）

**目标**：让基座模型理解 Agent 相关的概念和知识，不改变其"行为模式"，只扩充知识面。

**训练数据**：

| 数据类型 | 示例 | 作用 |
|---|---|---|
| API/工具文档 | Swagger 文档、SDK 文档 | 理解工具的功能和参数 |
| 代码仓库 | GitHub 上的 Agent 框架代码 | 理解代码执行逻辑 |
| Agent 交互日志 | ReAct 轨迹、函数调用记录 | 熟悉 Thought-Action-Observation 格式 |
| 领域知识 | 任务规划论文、操作手册 | 理解领域术语和流程 |

**训练方式**：标准语言模型目标（next token prediction），学习率比预训练低 1-2 个数量级（如 2e-5），防止遗忘通用能力。

```python
# CPT 阶段：标准语言模型训练，所有 token 都参与损失计算
loss = cross_entropy(model(input_tokens), target_tokens)
# 全量 token 都计算 loss，不做 mask
```

### 阶段 2：Agentic SFT（监督微调）

**目标**：让模型学会 Agent 的行为模式——何时思考、何时调用工具、如何解读结果。

**训练数据**：人工标注或强模型生成的高质量 Agent 轨迹。

```
一条训练样本（Agent 轨迹）:
┌─────────────────────────────────────────────────────┐
│ System: 你是一个天气助手，可用工具: get_weather(...)   │ ← 系统指令
│ User: 北京和上海明天谁更热？                          │ ← 用户输入
│ Thought: 需要分别查两个城市                           │ ← 模型应学会生成
│ Action: get_weather(city="北京", date="明天")         │ ← 模型应学会生成
│ Observation: {"temp": 35}                            │ ← 环境返回（非模型生成）
│ Thought: 北京35度，再查上海                           │ ← 模型应学会生成
│ Action: get_weather(city="上海", date="明天")         │ ← 模型应学会生成
│ Observation: {"temp": 32}                            │ ← 环境返回（非模型生成）
│ Thought: 北京35 > 上海32                             │ ← 模型应学会生成
│ Answer: 北京明天更热，35°C vs 32°C                   │ ← 模型应学会生成
└─────────────────────────────────────────────────────┘
```

#### 为什么要 mask observation tokens？

**Observation 是环境返回的真实数据，不是模型生成的。** 如果让模型在这些 token 上计算 loss，等于在教模型"记忆工具的返回值"——这既不可能（返回值是动态的），也会引入噪声。

```python
# SFT 阶段的 loss 计算（关键区别）
def compute_agent_sft_loss(trajectory):
    tokens = tokenize(trajectory)
    labels = tokens.clone()
    
    # mask 掉不应该学习的部分
    for span in trajectory.spans:
        if span.role in ("system", "user", "observation"):
            labels[span.start:span.end] = IGNORE_INDEX  # -100
    
    # 只在模型应该生成的部分计算 loss
    # 即 Thought、Action、Answer
    loss = cross_entropy(model(tokens), labels, ignore_index=IGNORE_INDEX)
    return loss
```

**mask 的具体原因**：

| 原因 | 详细说明 |
|---|---|
| **不可预测性** | Observation 内容由外部环境决定（API 返回值、数据库查询结果），模型不应试图"预测"这些值 |
| **防止幻觉** | 如果模型学习了特定的 observation pattern，推理时可能在没有真正调用工具的情况下"编造"返回值 |
| **梯度噪声** | Observation 的分布与模型生成的文本分布不同（JSON、数据表等），在其上计算 loss 会引入梯度噪声，干扰对 Thought/Action 的学习 |
| **因果错误** | 模型应学习"根据 observation 做判断"，而非"预测 observation 内容"——mask 确保因果方向正确 |

```
训练时的 loss mask 示意:
  [System...]  [User...]  [Thought...]  [Action...]  [Observation...]  [Thought...]  [Answer...]
  ████████████ ██████████ ▓▓▓▓▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓▓▓▓ ████████████████ ▓▓▓▓▓▓▓▓▓▓▓▓ ▓▓▓▓▓▓▓▓▓▓
  █ = masked (不计算 loss)   ▓ = 计算 loss (模型应学会生成的部分)
```

### 阶段 3：Agentic RL（强化学习）

**目标**：通过试错优化 Agent 的**决策策略**——选择哪个工具、何时停止、如何规划多步。

**为什么 SFT 不够**：SFT 只能模仿示范轨迹中的行为，不能发现更优的策略。RL 通过奖励信号让模型探索并强化更好的决策路径。

**训练流程**：

```
模型在环境中交互 → 产生轨迹 → 评估轨迹质量（奖励） → 更新策略
```

**奖励设计**：

```python
def compute_reward(trajectory):
    reward = 0.0
    
    # 1. 任务完成度（核心奖励）
    if task_completed_correctly(trajectory):
        reward += 1.0
    
    # 2. 效率惩罚（鼓励用更少步骤完成）
    reward -= 0.05 * len(trajectory.steps)
    
    # 3. 工具使用质量
    for step in trajectory.steps:
        if step.action == "unnecessary_tool_call":
            reward -= 0.1  # 惩罚无意义的工具调用
        if step.action == "correct_tool_selection":
            reward += 0.1  # 奖励选对工具
    
    # 4. 安全约束
    if violates_safety_rules(trajectory):
        reward = -1.0  # 强惩罚违规行为
    
    return reward
```

**常用 RL 算法**：

| 算法 | 特点 | 适用场景 |
|---|---|---|
| **PPO** | 稳定、广泛使用 | 通用 Agent 策略优化 |
| **GRPO** | Group Relative Policy Optimization，DeepSeek 提出 | 数学/代码推理 |
| **DPO** | 直接偏好优化，无需显式奖励模型 | 有人类偏好数据时 |
| **ReST** | 自博弈 + 过滤，迭代生成→筛选→训练 | 可自动验证结果的任务 |

### 三阶段协同关系

```
CPT:  知识注入  → 模型"知道"工具和 Agent 概念
SFT:  行为对齐  → 模型"会做"工具调用和推理
RL:   策略优化  → 模型"做得好"，选择最优行动序列

类比：
CPT = 读教材（获取知识）
SFT = 看范例 + 做习题（模仿行为）
RL  = 实战 + 复盘（优化策略）
```

**不是每个阶段都必须**：很多实际方案跳过 CPT 直接做 SFT（如 Toolformer），或跳过 RL 只做 SFT（如大部分开源 Agent 模型）。但最强的 Agent 模型（如 Claude、GPT-4）通常三阶段都做。

- 标签: `agentic-training`, `cpt`, `sft`, `rl`, `observation-masking`, `reward-design`
- 记录于: 2026-06-20

## Q: Tree of Thoughts 在线上系统中能用吗？如何平衡成本和效果？

### Tree of Thoughts（ToT）原理

ToT 是 Chain-of-Thought（CoT）的扩展——CoT 是线性推理链，ToT 是**树形推理**：

```
CoT（单路径）:
  问题 → 步骤1 → 步骤2 → 步骤3 → 答案

ToT（多路径探索）:
  问题 → 步骤1a ─→ 步骤2a ─→ 步骤3a → 答案A ← 评估：好
       → 步骤1b ─→ 步骤2b ─→ ✗ (剪枝)
       → 步骤1c ─→ 步骤2c ─→ 步骤3c → 答案C ← 评估：差
                                              ↓
                                          选择答案 A
```

**核心组件**：
1. **思维生成（Thought Generation）**：在每个节点生成多个候选下一步（通常 3-5 个）
2. **状态评估（State Evaluation）**：用 LLM 评估每个候选的前景
3. **搜索策略（Search Strategy）**：BFS 或 DFS 遍历思维树
4. **剪枝（Pruning）**：丢弃评估得分低的分支

```python
class TreeOfThoughts:
    def __init__(self, llm, branching_factor=3, max_depth=3):
        self.llm = llm
        self.b = branching_factor  # 每步生成几个候选
        self.max_depth = max_depth
    
    def solve(self, problem: str) -> str:
        root = ThoughtNode(state=problem, depth=0)
        
        # BFS 搜索
        frontier = [root]
        for depth in range(self.max_depth):
            candidates = []
            for node in frontier:
                # 生成 b 个候选下一步
                thoughts = self.generate_thoughts(node.state, k=self.b)
                for thought in thoughts:
                    child = ThoughtNode(
                        state=node.state + "\n" + thought,
                        depth=depth + 1,
                        parent=node,
                    )
                    # 评估这个思路的前景
                    child.score = self.evaluate(child.state)
                    candidates.append(child)
            
            # 只保留得分最高的 b 个节点继续探索
            frontier = sorted(candidates, key=lambda x: x.score, reverse=True)[:self.b]
        
        return frontier[0].state  # 返回最优路径
    
    def generate_thoughts(self, state, k):
        return self.llm.generate(
            f"给定当前推理状态:\n{state}\n\n"
            f"请生成 {k} 种不同的下一步推理方向。",
            n=k  # 生成 k 个候选
        )
    
    def evaluate(self, state):
        score = self.llm.generate(
            f"评估以下推理过程的质量（1-10 分）:\n{state}\n"
            f"考虑：逻辑是否正确？是否朝正确方向前进？"
        )
        return float(score)
```

### 成本分析

**ToT 的成本是 CoT 的 N 倍**：

```
CoT: 1 次 LLM 调用
ToT: 每层 b 个候选 × 每个候选 1 次生成 + 1 次评估
     = 每层 2b 次调用
     × depth 层
     = 2 × b × depth 次调用

例: b=3, depth=3 → 18 次 LLM 调用（vs CoT 的 1 次）
成本: ~18 倍
延迟: ~6 倍（3 层串行，每层内可并行）
```

### 线上系统能用吗？

**直接用 ToT 论文的实现，大多数线上系统用不了。** 原因：

| 限制 | 详情 |
|---|---|
| **延迟** | 18 次 LLM 调用，即使并行也需 3 轮串行（每轮 ~500ms），总延迟 1.5s+ |
| **成本** | 单次请求成本 ×18，日均 100 万请求 → 成本从 $5K 变 $90K |
| **复杂度** | 需要实现搜索树管理、并行调用、结果聚合 |

**但 ToT 的思想可以以轻量形式落地**：

### 轻量化方案

**方案一：Sample + Vote（最简单的"ToT"）**

```python
def sample_and_vote(query, n=3):
    # 生成 n 个独立回答（并行，可用低温度增加多样性）
    responses = [
        llm.generate(query, temperature=0.7)
        for _ in range(n)
    ]
    
    # 让 LLM 投票选最佳
    best = llm.generate(
        f"以下是对同一问题的 {n} 个回答:\n"
        + "\n".join(f"方案{i+1}: {r}" for i, r in enumerate(responses))
        + f"\n\n请选择最佳方案并说明理由。"
    )
    return best
```

**成本**：n+1 次调用（而非 2×b×depth），n=3 时只有 4 次，延迟 2 轮（生成并行 + 评估）。

**方案二：Best-of-N（纯采样）**

```python
def best_of_n(query, n=5, evaluator=None):
    responses = parallel_generate(query, n=n, temperature=0.8)
    
    if evaluator:
        # 外部评估器（如规则或小模型）
        scores = [evaluator.score(r) for r in responses]
    else:
        # 自评估：让 LLM 给每个回答打分
        scores = [
            llm.score(f"评估回答质量(1-10): {r}")
            for r in responses
        ]
    
    return responses[scores.index(max(scores))]
```

**方案三：条件触发的深度推理**

```python
def adaptive_reasoning(query, complexity_threshold=0.7):
    # 快速估计问题复杂度
    complexity = estimate_complexity(query)
    
    if complexity < 0.3:
        return llm.generate(query)  # 简单问题：直接回答
    elif complexity < complexity_threshold:
        return llm.generate(query, system="请一步步思考")  # 中等：CoT
    else:
        return sample_and_vote(query, n=3)  # 复杂：轻量 ToT
```

**效果**：只对 5-10% 的高复杂度请求使用多路推理，整体成本增加 <20%，但这些难题的准确率提升 15-30%。

### 成本效果平衡策略

| 策略 | 成本倍数 | 延迟增加 | 效果提升 | 适用场景 |
|---|---|---|---|---|
| **直接回答** | 1x | 0 | 基准 | 简单事实查询 |
| **CoT** | 1x-1.5x | ~0 | +10-15% | 需要推理的问题 |
| **Sample+Vote** | 4x | 2 轮 | +15-20% | 高价值决策 |
| **Best-of-N** | Nx | 1 轮 | +10-15% | 有自动评估器时 |
| **完整 ToT** | 18x+ | 3+ 轮 | +20-30% | 离线/研究/极高价值 |

### 实际落地建议

1. **CoT 已经足够好**：对大多数线上场景，在 prompt 中加"请逐步思考"就能获得 80% 的收益
2. **Sample+Vote 是最佳性价比**：当 CoT 不够时，生成 3 个候选 + 1 次评估，成本可控
3. **完整 ToT 留给离线**：离线数据分析、难题求解、生成训练数据等不要求实时响应的场景
4. **用便宜模型做探索**：生成候选用 Haiku/GPT-4o-mini，评估用 Sonnet/GPT-4o，降低成本
5. **缓存复用**：相似问题的搜索树可以缓存和复用中间节点

- 标签: `tree-of-thoughts`, `reasoning`, `cost-optimization`, `sample-and-vote`, `chain-of-thought`
- 记录于: 2026-06-20

## Q: 如何量化评估一个上线的 Agent 好坏？

### 评估维度：四个核心层面

Agent 的评估不能只看"回答对不对"——它涉及多步决策、工具调用、用户交互等多个环节，需要分层评估。

```
┌────────────────────────────────────────────┐
│  Layer 4: 业务指标（最终价值）               │
│  用户满意度、转化率、人工接管率               │
├────────────────────────────────────────────┤
│  Layer 3: 端到端任务指标                     │
│  任务完成率、平均轮次、首次解决率             │
├────────────────────────────────────────────┤
│  Layer 2: 模块级指标                         │
│  意图准确率、工具选择准确率、生成质量         │
├────────────────────────────────────────────┤
│  Layer 1: 系统级指标                         │
│  延迟、吞吐、成本、可用性                    │
└────────────────────────────────────────────┘
```

### Layer 1：系统级指标（基础设施）

| 指标 | 计算方式 | 基准参考 |
|---|---|---|
| **P50/P95/P99 延迟** | 从用户发送到 Agent 回复的耗时 | P50 <2s, P95 <5s |
| **吞吐量（QPS）** | 单位时间处理的请求数 | 取决于业务规模 |
| **单次请求成本** | Token 消耗 × 单价 + 工具调用成本 | <$0.05/请求（一般场景） |
| **可用性** | 成功响应数 / 总请求数 | >99.5% |
| **错误率** | 异常/超时/空回复的比例 | <1% |

```python
# 系统指标采集
metrics = {
    "latency_p50": percentile(response_times, 50),
    "latency_p95": percentile(response_times, 95),
    "avg_tokens_per_request": sum(token_counts) / len(requests),
    "avg_cost_per_request": sum(costs) / len(requests),
    "error_rate": error_count / total_count,
    "availability": success_count / total_count,
}
```

### Layer 2：模块级指标（各环节质量）

| 模块 | 指标 | 计算方式 |
|---|---|---|
| **意图识别** | Accuracy / F1 | 定期采样人工标注对比 |
| **工具选择** | Tool Selection Accuracy | 选对工具的比例 |
| **参数提取** | Slot Filling Rate | 必填参数正确提取率 |
| **回答生成** | LLM-as-Judge 评分 | 用强模型打分（1-5） |
| **安全** | 拒绝率 / 越权率 | 应拒绝的拒了 / 不应执行的执行了 |

### Layer 3：端到端任务指标（核心）

```python
class TaskMetrics:
    def compute(self, sessions: list[Session]) -> dict:
        return {
            # 1. 任务完成率（最重要的单一指标）
            "task_completion_rate": self._completion_rate(sessions),
            
            # 2. 首次解决率（FCR）——一次交互就解决问题的比例
            "first_contact_resolution": self._fcr(sessions),
            
            # 3. 平均解决轮次——完成任务需要多少轮对话
            "avg_turns_to_resolve": self._avg_turns(sessions),
            
            # 4. 人工接管率——Agent 无法解决、转人工的比例
            "human_handoff_rate": self._handoff_rate(sessions),
            
            # 5. 任务放弃率——用户中途放弃的比例
            "abandonment_rate": self._abandonment_rate(sessions),
        }
    
    def _completion_rate(self, sessions):
        completed = sum(1 for s in sessions if s.task_completed)
        return completed / len(sessions)
    
    def _fcr(self, sessions):
        one_turn_success = sum(
            1 for s in sessions 
            if s.task_completed and s.turn_count <= 2
        )
        return one_turn_success / len(sessions)
```

### Layer 4：业务指标（最终价值）

| 指标 | 含义 | 如何衡量 |
|---|---|---|
| **用户满意度（CSAT）** | 用户对 Agent 服务的评分 | 对话结束后弹出评分（1-5） |
| **Net Promoter Score** | 用户推荐意愿 | 定期问卷 |
| **人力成本节约** | Agent 替代了多少人工 | 对比部署前后人工工单量 |
| **转化率** | 营销/推荐场景的转化 | 追踪 Agent 推荐后的成交 |
| **留存率** | 用户是否继续使用 Agent | 7 日 / 30 日活跃率 |

### 评估体系的实施

**离线评估（上线前）**：

```python
# 用标注好的评测集评估
def offline_evaluation(agent, test_set):
    results = []
    for case in test_set:
        response = agent.run(case.query, context=case.context)
        results.append({
            "query": case.query,
            "expected": case.expected_output,
            "actual": response,
            "intent_correct": response.intent == case.expected_intent,
            "tool_correct": response.tool == case.expected_tool,
            # LLM-as-Judge 评分
            "quality_score": judge_llm.evaluate(
                query=case.query,
                expected=case.expected_output,
                actual=response.text,
                rubric="准确性(0-2) + 完整性(0-2) + 流畅性(0-1)"
            ),
        })
    
    return aggregate_metrics(results)
```

**线上评估（持续监控）**：

```python
# 建立监控 Dashboard
ONLINE_METRICS = {
    # 实时指标（秒级）
    "realtime": ["latency_p95", "error_rate", "qps"],
    
    # 小时级指标
    "hourly": [
        "task_completion_rate",
        "human_handoff_rate",
        "avg_cost_per_request",
    ],
    
    # 日级指标
    "daily": [
        "csat_score",
        "abandonment_rate",
        "low_confidence_ratio",  # 低置信度请求占比
    ],
    
    # 周级指标
    "weekly": [
        "intent_accuracy",       # 人工抽样标注
        "badcase_count",         # 发现的 Badcase 数
        "new_intent_coverage",   # 新意图的覆盖率
    ],
}
```

**A/B 测试**：

```python
# 对比新旧 Agent 版本
def ab_test(agent_a, agent_b, traffic_split=0.1):
    """将 10% 流量导向新版本 B"""
    if random.random() < traffic_split:
        response = agent_b.run(query)
        log_metric("agent_b", response)
    else:
        response = agent_a.run(query)
        log_metric("agent_a", response)
    return response

# 对比维度
comparison = {
    "task_completion": {"A": 0.82, "B": 0.87},  # B 版本提升 5%
    "avg_latency": {"A": 1.8, "B": 2.1},        # B 版本慢了 0.3s
    "avg_cost": {"A": 0.03, "B": 0.04},          # B 版本贵了 33%
    # → 权衡：完成率提升是否值得多花的成本和延迟？
}
```

### 关键指标的优先级

```
上线初期:  任务完成率 > 错误率 > 延迟    （先保证能用）
稳定期:    CSAT > 人工接管率 > 成本       （优化体验和效率）
规模化:    成本 > 吞吐 > 延迟            （控制边际成本）
```

**一个实用的"Agent 健康度"综合评分**：

```python
def agent_health_score(metrics):
    return (
        0.30 * metrics["task_completion_rate"] +
        0.20 * (1 - metrics["human_handoff_rate"]) +
        0.15 * metrics["csat_normalized"] +
        0.15 * (1 - metrics["error_rate"]) +
        0.10 * (1 - metrics["latency_p95"] / MAX_ACCEPTABLE_LATENCY) +
        0.10 * (1 - metrics["avg_cost"] / MAX_ACCEPTABLE_COST)
    )
```

- 标签: `evaluation`, `online-metrics`, `task-completion`, `ab-testing`, `csat`, `agent-quality`
- 记录于: 2026-06-20

## Q: 当前阻碍 Agent 大规模落地的最大挑战是什么？如何解决可控性和能力的平衡问题？

### 五大核心挑战

#### 1. 可靠性不足（最致命）

**现状**：Agent 的准确率在持续提升，但**可靠性（一致性）几乎没有改善**。

```
同一个任务执行 10 次:
  传统软件: 10 次结果完全一致
  Agent:    7 次正确，2 次部分正确，1 次完全错误

准确率 70% 看似不错，但对企业来说意味着：
  日均 10000 次请求 → 3000 次出错 → 不可接受
```

**根本原因**：LLM 是概率模型，inherently non-deterministic。即使 temperature=0，不同批次的推理结果仍可能不同（浮点精度、KV Cache 策略等）。

**解决方向**：
- 关键路径用确定性逻辑（规则/代码），只在需要推理的环节用 LLM
- 多路验证（Sample+Vote）降低单次失误的影响
- 完善的回退和人工兜底机制

#### 2. 成本与延迟

```
单次 Agent 交互的成本结构:
  意图识别:      ~$0.001  (Haiku / 规则)
  上下文组装:     ~$0.005  (检索 + embedding)
  LLM 推理:      ~$0.02-0.10  (主模型，多轮调用)
  工具调用:       ~$0.001-0.01  (API 费用)
  ────────────
  总计:          ~$0.03-0.12 / 次

  日均 100 万次 → $30K-120K / 天

对比:
  人工客服:  ~$5-10 / 次（但人力有限）
  传统 NLU:  ~$0.001 / 次（但能力有限）
```

**解决方向**：分层处理（80% 简单请求走轻量模型/规则）、语义缓存（相似请求复用回答）、批处理（非实时请求打包处理）。

#### 3. 评估困难

```
传统软件: 输入→输出确定 → 单元测试覆盖
Agent:    输入→推理链→多步工具调用→输出
          中间任何一步都可能变化
          "正确答案"本身可能不唯一
```

**解决方向**：LLM-as-Judge 自动评估、离线评测集 + 线上 Badcase 监控双轨制、关注端到端指标而非中间步骤。

#### 4. 安全与合规

```
风险矩阵:
  Prompt Injection  → Agent 被劫持执行恶意操作
  数据泄露          → Agent 将敏感信息暴露给用户
  幻觉              → Agent 给出错误信息导致决策失误
  越权操作          → Agent 执行了超出授权的操作
  审计缺失          → 无法追溯 Agent 的决策过程
```

**解决方向**：分层防护（输入过滤→意图校验→权限控制→输出审查）、审计日志、人工审批关键操作。

#### 5. 工程复杂度

```
开发一个 Agent vs 开发一个传统 API:
  传统 API:  定义接口 → 实现逻辑 → 测试 → 上线
  Agent:     定义意图 → 设计 Prompt → 选择模型 → 配置工具 →
             处理多轮 → 管理上下文 → 设计容错 → 评估质量 →
             监控漂移 → 迭代优化
```

### 可控性与能力的核心矛盾

这是 Agent 设计中最根本的 tension：

```
可控性高 ←──────────────────────────→ 能力强
  │                                      │
  │  规则系统                             │  完全自主 Agent
  │  传统 NLU + 固定流程                  │  自由推理 + 任意工具
  │  准确率 99%                           │  准确率 70%
  │  只能处理预定义场景                   │  能处理开放域问题
  │  无惊喜也无惊吓                       │  有惊喜也有惊吓
  │                                      │
  决定论                                  概率论
```

**越给 Agent 自由度，它的能力越强，但出错的可能性也越大。** 这个矛盾无法消除，只能管理。

### 平衡策略

#### 策略一：分级自主权（Graduated Autonomy）

```python
AUTONOMY_LEVELS = {
    "level_0": {
        "description": "纯规则执行",
        "example": "查询天气 → 直接调 API，无需 LLM",
        "controllability": "★★★★★",
        "capability": "★",
    },
    "level_1": {
        "description": "LLM 辅助的结构化流程",
        "example": "客服对话 → 固定流程，LLM 填充自然语言",
        "controllability": "★★★★",
        "capability": "★★★",
    },
    "level_2": {
        "description": "LLM 决策 + 工具约束",
        "example": "Agent 自主选工具，但工具集受限 + 需确认",
        "controllability": "★★★",
        "capability": "★★★★",
    },
    "level_3": {
        "description": "完全自主 + 事后审计",
        "example": "Agent 自主完成复杂任务，系统记录全链路",
        "controllability": "★★",
        "capability": "★★★★★",
    },
}

def select_autonomy_level(task):
    if task.risk_level == "high":    # 支付、删除
        return "level_1"            # 固定流程 + LLM 填充
    elif task.risk_level == "medium": # 修改设置、发消息
        return "level_2"            # LLM 决策 + 人工确认
    else:                           # 查询、闲聊
        return "level_3"            # 完全自主
```

**核心思想**：不是所有任务都需要同等自主权。高风险任务用高可控方案，低风险任务给自由度。

#### 策略二：护栏内自由（Freedom within Guardrails）

```python
class GuardedAgent:
    def execute(self, task):
        # 护栏 1：输入过滤
        sanitized = self.input_guard.check(task.input)
        
        # 自由区：LLM 自主推理和决策
        result = self.llm_agent.run(sanitized)
        
        # 护栏 2：输出校验
        validated = self.output_guard.check(result)
        
        # 护栏 3：操作审批
        if validated.has_side_effects:
            if validated.risk > THRESHOLD:
                return self.request_human_approval(validated)
        
        return validated

# 类比：高速公路
# 护栏 = 道路边界、限速、收费站
# 自由 = 在道路内可以自由变道、选择路线
# 不需要每一步都人工干预，但出界时有防护
```

#### 策略三：渐进式放权（Progressive Trust）

```
新 Agent 上线:
  Week 1:  Level 1 — 所有决策需人工确认      → 收集数据
  Week 2:  Level 1.5 — 低风险决策自动执行     → 监控准确率
  Week 4:  Level 2 — 中风险决策自动执行       → 持续监控
  Week 8:  Level 2.5 — 大部分决策自动执行     → 只审批高风险
  ...
  
  准确率达标 → 提升自主权
  准确率下降 → 降低自主权（自动降级）
```

```python
class ProgressiveTrust:
    def __init__(self, agent_id):
        self.trust_score = 0.5  # 初始信任度
        self.history = []
    
    def update_trust(self, task_result):
        if task_result.correct:
            self.trust_score = min(1.0, self.trust_score + 0.01)
        else:
            self.trust_score = max(0.0, self.trust_score - 0.05)
            # 错误的惩罚是正确的 5 倍——信任难建立，易摧毁
    
    def should_auto_execute(self, task) -> bool:
        required_trust = {
            "low_risk": 0.3,
            "medium_risk": 0.7,
            "high_risk": 0.95,
        }
        return self.trust_score >= required_trust[task.risk_level]
```

#### 策略四：确定性骨架 + LLM 填充

```
传统做法（全 LLM）:
  用户输入 → LLM 决定流程 → LLM 选工具 → LLM 判断结果 → LLM 生成回答
  （每一步都是概率性的 → 不确定性叠加）

改进做法（确定性骨架）:
  用户输入 → LLM 意图识别 → 确定性流程路由 → 确定性工具选择 →
  确定性结果解析 → LLM 生成自然语言回答
  （只在必要环节用 LLM → 最小化不确定性）
```

```python
# 确定性骨架示例
def handle_refund_request(user_input):
    # Step 1: LLM 提取关键信息（需要推理能力）
    entities = llm.extract(user_input, schema=RefundSchema)
    
    # Step 2: 确定性流程（不需要 LLM）
    order = db.get_order(entities.order_id)
    if not order:
        return template_response("order_not_found")
    
    if order.days_since_purchase > 30:
        return template_response("refund_expired")
    
    # Step 3: 确定性操作（不需要 LLM）
    refund_result = payment_api.refund(order.id, order.amount)
    
    # Step 4: LLM 生成自然语言回答（需要自然语言能力）
    return llm.generate_response(
        template="refund_success",
        data=refund_result,
    )
```

### 总结

```
最大挑战排序:
  1. 可靠性 — 概率性推理无法保证一致输出
  2. 成本    — 多步 LLM 调用的边际成本高
  3. 评估    — 没有好的自动化评估手段
  4. 安全    — 攻击面远大于传统系统
  5. 工程    — 开发和维护复杂度远高于传统系统

可控性 vs 能力的平衡:
  ✗ 不是选一个点，而是设计一个谱
  ✓ 不同任务/风险等级选不同的自主权
  ✓ 用确定性逻辑包裹不确定性推理
  ✓ 渐进式放权，数据驱动地调整信任边界
  ✓ 始终保留人工兜底的逃生通道
```

- 标签: `agent-challenges`, `controllability`, `reliability`, `graduated-autonomy`, `guardrails`, `progressive-trust`
- 记录于: 2026-06-20
