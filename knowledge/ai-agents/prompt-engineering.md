# Prompt 工程与上下文

> 涵盖：提示词模板设计、上下文工程实践、查询改写、Prompt 调优与回归控制。

## Q: 怎么构建提示词模板？上下文工程有哪些实践经验？TODO List 优化为什么能让模型更聚焦？

### 提示词模板构建

一个生产级 prompt 模板的基本结构：

```
┌─────────────────────────────────┐
│ 1. Role（角色定义）               │  你是谁、你的能力边界
│ 2. Task（任务描述）               │  要做什么、核心目标
│ 3. Context（上下文）              │  背景信息、用户画像、历史
│ 4. Constraints（约束）            │  不做什么、格式要求、安全边界
│ 5. Output Format（输出格式）      │  JSON Schema / Markdown / 结构化模板
│ 6. Examples（示例）               │  Few-shot 示例（1-3 个，覆盖典型和边界情况）
│ 7. Chain-of-Thought（可选）       │  思考步骤引导
└─────────────────────────────────┘
```

**实践原则**：
- **具体 > 抽象**："列出 3 个关键差异，每个用一句话概括" 优于 "分析一下它们的区别"
- **约束前置**：把最重要的约束放在 prompt 开头而非结尾，LLM 对开头内容注意力更高
- **负面约束用正面表达**："只使用提供的资料回答" 优于 "不要编造信息"
- **输出格式显式声明**：用 JSON Schema 或 Markdown 模板约束输出，比自然语言描述格式更可靠

### 上下文工程实践

上下文工程的核心问题：**在有限的上下文窗口内，放入最有价值的信息**。

**1. 上下文分层**
```
System Prompt（固定）: 角色、全局约束、输出格式    ~2K tokens
Dynamic Context（动态）: 检索结果、用户画像、历史    ~4-8K tokens
User Query（当前）: 用户输入 + 即时上下文            ~0.5-1K tokens
```

**2. 信息优先级排序**
- 高优先级：与当前查询直接相关的检索结果、最近的对话历史
- 中优先级：用户画像、任务约束
- 低优先级：背景知识、通用指令
- 窗口不够时，从低优先级开始裁剪

**3. 上下文注入时序**
- **Lazy Loading**：不在 system prompt 中预加载所有信息，而是在需要时动态注入
- **总结替代原文**：长文档先生成摘要，只在需要细节时加载原文
- **上下文窗口监控**：追踪 token 使用量，接近阈值时自动压缩或截断

### TODO List 优化原理

**做法**：在 system prompt 或中间推理步骤中，让 Agent 维护一个结构化的 TODO List：

```markdown
## 当前任务清单
- [x] 识别用户意图：代码优化
- [x] 定位目标文件：src/utils.py
- [ ] 分析性能瓶颈
- [ ] 提出优化方案
- [ ] 实现并验证
```

**为什么能让模型更聚焦**：

1. **外部化工作记忆**：LLM 没有持久的工作记忆，TODO List 把"还要做什么"显式写在上下文中，替代了模型需要隐式记住的内容。这就像人类做复杂任务时写清单——减少认知负荷。

2. **约束生成空间**：当模型看到"当前该做：分析性能瓶颈"时，输出空间被大幅收窄，不会跑去做无关的事情。没有 TODO List 时，模型每步都要从整个任务描述中重新推理"下一步该做什么"。

3. **进度可追踪**：`[x]` 标记让模型清楚知道哪些已完成，避免重复工作（这是长链任务中常见的退化模式——模型忘记自己做过什么，重复执行相同步骤）。

4. **自然的 CoT 引导**：TODO List 本质上是一种结构化的 Chain-of-Thought，但比自由形式的 CoT 更可控——步骤是预定义的，模型只需要执行而非规划。

5. **错误定位**：当输出出错时，可以精确定位到 TODO List 中的哪一步出了问题，便于调试。

**实际效果**：在长链任务（>5 步）中，加入 TODO List 后任务完成率通常提升 15-30%，主要收益来自减少遗忘和重复。

- 标签: `prompt-engineering`, `context-engineering`, `todo-list`, `template-design`
- 记录于: 2026-06-19

## Q: 查询改写是什么？多维度查询改写具体怎么做？需要用户补充时怎么设计交互？

### 什么是查询改写

查询改写（Query Rewriting）是指将用户原始查询转化为更适合检索系统处理的形式。用户的自然语言表达往往含糊、不完整或不匹配索引的术语，改写的目标是弥合用户意图和检索能力之间的鸿沟。

### 多维度改写

**维度一：意图澄清（Intent Clarification）**
```
原始: "Python 怎么加速"
改写: "Python 程序性能优化方法，包括 CPU 密集型和 I/O 密集型场景"
```
把模糊的意图展开为明确的检索方向。

**维度二：实体扩展（Entity Expansion）**
```
原始: "Transformer 的注意力"
改写: "Transformer 架构中的 Self-Attention 自注意力机制"
```
补充同义词、全称、相关术语，提高召回率。

**维度三：时间限定（Temporal Resolution）**
```
原始: "最新的 RAG 技术"
改写: "2025-2026 年 RAG 检索增强生成最新进展"
```
将模糊的时间表达转化为具体范围。

**维度四：子查询分解（Sub-query Decomposition）**
```
原始: "GraphRAG 和 RAG 的区别以及怎么选"
改写为两个子查询:
  1. "GraphRAG 与传统 RAG 的核心架构和能力差异"
  2. "GraphRAG vs RAG 选型建议和适用场景"
```
复合查询拆成独立子查询，分别检索后合并结果。

**维度五：HyDE（假设文档嵌入）**
```
原始: "怎么处理长上下文"
改写为假设的答案段落: "处理长上下文的常见方法包括滑动窗口、分层摘要、
RAG 检索、KV Cache 优化等..."
```
用 LLM 生成一段假设的答案，用这段文本（而非原始 query）做 embedding 检索。原理：假设答案在语义空间中更接近真实文档。

### 实现架构

```python
async def multi_dim_rewrite(query: str, context: dict) -> list[str]:
    rewrites = await asyncio.gather(
        intent_clarify(query),            # 意图澄清
        entity_expand(query),             # 实体扩展
        temporal_resolve(query, context),  # 时间限定
        sub_query_decompose(query),        # 子查询分解
    )
    # 去重 + 合并
    return deduplicate(flatten(rewrites))
```

每个改写维度用独立的 LLM 调用（通常用轻量模型如 Haiku），并行执行以控制延迟。

### 需要用户补充信息时的交互设计

**触发条件**：
- 意图分类置信度 < 阈值（如 0.6）
- 关键槽位缺失（如"帮我查航班"缺少出发地/目的地/日期）
- 查询存在歧义（如"苹果价格"——水果还是公司？）

**交互设计原则**：

1. **最小化追问**：一次最多问 1-2 个问题，不要一次性列出所有缺失信息。
2. **提供选项而非开放问题**："你是想了解 [A. 代码优化] 还是 [B. 算法优化]？" 优于 "你想了解什么方面的优化？"
3. **渐进式澄清**：先问最关键的缺失信息，拿到后判断是否还需要追问。
4. **带默认值的追问**："你指的时间范围是最近一年吗？（还是其他时段？）"——给出合理默认值，用户确认比从零填写更快。

**技术实现**：

```python
class ClarificationManager:
    def check_completeness(self, intent, slots) -> ClarificationRequest | None:
        missing = self.get_missing_required_slots(intent, slots)
        if not missing:
            return None
        # 优先级排序：必填 > 有助于精确检索 > 可选
        top_missing = sorted(missing, key=lambda s: s.priority)[:2]
        return ClarificationRequest(
            message=self.generate_question(top_missing),
            options=self.generate_options(top_missing),
            slot_names=[s.name for s in top_missing],
        )

    def merge_response(self, original_slots, user_response):
        # 将用户回答解析后合并回槽位
        new_slots = self.parse_response(user_response)
        return {**original_slots, **new_slots}
```

**关键点**：改写和追问是互补的——改写解决"用户说了但说得不好"的问题，追问解决"用户根本没说"的问题。生产中通常先尝试改写，改写后检索结果质量仍低时再触发追问。

- 标签: `query-rewriting`, `hyde`, `clarification`, `retrieval`, `interaction-design`
- 记录于: 2026-06-19

## Q: Prompt 调优经常"修好一类、坏了另一类"，怎么解决？

### 问题本质

Prompt 是一个整体的指令集合，LLM 对 prompt 的理解是全局的——修改任何一部分都可能影响模型对其他部分的权重分配。这类似于调整神经网络的超参数：改变一个维度往往引发连锁反应。

### 系统化解决方案

**1. 建立回归测试集（Regression Test Suite）**

这是最核心的解决方案。在修改 prompt 之前，先有一套自动化评测：

```
test_suite/
  intent_classification/      # 意图识别测试集
    test_cases.jsonl           # {"query": "...", "expected_intent": "...", "tags": [...]}
  generation_quality/          # 生成质量测试集
    test_cases.jsonl
  edge_cases/                  # 边界情况
    test_cases.jsonl
```

每次 prompt 改动后，自动跑全量测试集：
- 新增 case 的通过率（改好了多少）
- 原有 case 的通过率（有没有退化）
- **只有新增通过且原有不退化时才合入**

**2. Prompt 模块化**

把单一大 prompt 拆分为独立模块，每个模块控制一类行为：

```markdown
## 角色定义
{role_definition}

## 输出约束
{output_constraints}

## 领域 A 规则
{domain_a_rules}

## 领域 B 规则
{domain_b_rules}
```

修改"领域 A 规则"时，其他模块不变。模块化降低了修改的耦合度，但无法完全消除——LLM 仍然是整体理解全部内容的。

**3. 分支策略（Prompt Branching）**

当两类任务的 prompt 要求根本矛盾时，不要硬塞在一个 prompt 里，而是拆分为不同的 prompt 分支：

```python
if intent == "creative_writing":
    prompt = creative_prompt     # 鼓励发散、创造
elif intent == "fact_qa":
    prompt = factual_prompt      # 强调准确、严谨
```

这样修改 creative_prompt 不会影响 fact_qa 的效果。

**4. 版本管理 + Diff 分析**

```
prompts/
  v1.0.md
  v1.1.md    # 修改了格式约束
  v1.2.md    # 回滚了 v1.1 的部分改动，保留有效部分
  CHANGELOG.md
```

每个版本附带评测结果，可以精确对比哪个版本在哪类 case 上表现最好。

**5. 增量修改原则**

- **每次只改一个方面**：不要同时修改角色定义和输出格式。
- **先加后减**：先用增量的方式加规则，观察效果；如果有冲突再考虑减掉旧规则。
- **用示例代替规则**：当文字规则难以精确表达时，用 few-shot 示例更稳定——模型对示例的理解比对规则描述更一致。

**6. A/B 测试**

生产环境中对 prompt 改动做流量分桶：
- 10% 流量跑新 prompt
- 90% 流量跑旧 prompt
- 对比关键指标（满意度、任务完成率、幻觉率）
- 显著优于旧版时全量上线

### 核心认知

prompt 调优本质上是一个**多目标优化问题**，不存在"改好 A 且不影响 B"的银弹。正确的应对不是追求完美的单一 prompt，而是通过测试体系和工程实践最小化回归风险。

- 标签: `prompt-tuning`, `regression-testing`, `prompt-management`, `a-b-testing`
- 记录于: 2026-06-19

## Q: Prompt 调到极限后，优先选择换模型、调参数还是上 SFT？为什么？

### 决策阶梯

```
成本低 ←────────────────────────────→ 成本高
效果快 ←────────────────────────────→ 效果慢

Prompt 优化 → 参数调整 → 换模型 → SFT → 全量微调/预训练
```

### 各方案分析

**1. 调参数（Temperature / Top-P / Max Tokens）**

**优先度**：最先尝试（成本最低）

- `temperature`：降低到 0-0.3 可以减少随机性，提升一致性；升高到 0.7-1.0 增加创造性
- `top_p`：配合 temperature 使用，限制采样范围
- `max_tokens`：输出截断问题时首先检查这个
- `stop_sequences`：控制输出终止条件，避免过度生成

**适用场景**：输出不稳定、格式不一致、生成过长/过短。

**局限**：参数调整无法提升模型的能力上限，只能调节输出的分布特征。

**2. 换模型**

**优先度**：参数调完后

**何时换模型**：
- **推理能力不足**：当前模型无法完成多步逻辑推理 → 换推理模型（如 o1/o3 系列）
- **领域知识缺失**：当前模型对特定领域（如医疗、法律）的知识不够 → 换该领域预训练更充分的模型
- **多语言能力**：中文场景下某些模型明显优于其他模型
- **成本考量**：当前模型价格过高但任务不需要那么强的能力 → 降级到性价比更高的模型

**权衡**：换模型意味着所有 prompt 可能需要重新调优，因为不同模型对 prompt 的敏感度不同。

**3. SFT（Supervised Fine-Tuning）**

**优先度**：最后手段

**何时上 SFT**：
- **一致的失败模式**：prompt 无论怎么调都无法解决某类特定问题，且这类问题有清晰的模式
- **输出格式强约束**：需要严格遵循复杂的输出格式（如特定 DSL、行业标准格式）
- **领域适配**：模型需要理解大量领域专用术语和逻辑
- **已有足够标注数据**：至少 500-1000 条高质量标注样本（质量 > 数量）
- **Prompt 已经很长**：大量 few-shot 示例吃掉了上下文窗口空间

**SFT 的成本**：
- 数据标注（最大成本）：1000 条高质量标注可能需要 2-4 周
- 训练成本：取决于模型大小和数据量
- 维护成本：基座模型升级时需要重新微调
- 评估成本：需要维护独立的测试集防止过拟合

**SFT 的风险**：
- 灾难性遗忘：微调后模型在通用能力上可能退化
- 过拟合：数据量不够时容易过拟合训练集分布
- 基座依赖：基座模型更新时，微调需要重做

### 决策框架

```
问题诊断
  ├─ 输出不稳定/格式不对 → 调参数
  ├─ 能力不够（推理/知识） → 换模型
  ├─ Prompt 太长/示例太多 → 考虑 SFT 替代 few-shot
  ├─ 特定模式持续失败 + 有标注数据 → SFT
  └─ 以上都不是 → 检查是不是任务本身定义有问题
```

**核心原则**：穷尽 prompt 和参数优化后再考虑 SFT。Prompt 的改动可以在分钟级别验证，SFT 的改动需要天级别。如果 prompt 能解决 80% 的问题，只对剩余 20% 的硬骨头考虑 SFT。

- 标签: `prompt-optimization`, `sft`, `model-selection`, `decision-framework`
- 记录于: 2026-06-19

## Q: 上下文超出限制时如何处理？滑动窗口和动态摘要的区别？

### 问题场景

LLM 的上下文窗口有限（4K-200K tokens）。长对话或大量工具调用结果会导致上下文溢出：

```
System Prompt: ~500 tokens
长期记忆注入:  ~500 tokens
工具定义:      ~1000 tokens
对话历史:      ~???? tokens  ← 这里不断增长
当前用户输入:  ~200 tokens
─────────────────────────
总计不能超过:  128K tokens（以 GPT-4o 为例）
```

### 方案一：滑动窗口（Sliding Window）

**思路**：只保留最近 N 轮对话，丢弃更早的。

```python
class SlidingWindowMemory:
    def __init__(self, max_turns=10):
        self.messages = []
        self.max_turns = max_turns
    
    def add(self, message):
        self.messages.append(message)
    
    def get_context(self):
        # 保留 system prompt + 最近 N 轮
        return self.system_prompt + self.messages[-self.max_turns * 2:]
```

```
完整对话: [T1] [T2] [T3] [T4] [T5] [T6] [T7] [T8] [T9] [T10]
窗口=6:                              [T5] [T6] [T7] [T8] [T9] [T10]
                                      ↑ 窗口起点            ↑ 当前
         T1-T4 的信息完全丢失
```

**优点**：
- 实现极简，零额外 LLM 调用
- 延迟零增加
- 成本零增加

**缺点**：
- 早期信息**完全丢失**——如果用户在 T2 说了关键偏好，到 T8 就忘了
- 窗口边界处可能切断语义连贯的对话

### 方案二：动态摘要（Dynamic Summary）

**思路**：将超出窗口的对话压缩为摘要，保留关键信息。

```python
class DynamicSummaryMemory:
    def __init__(self, llm, max_tokens=4000):
        self.llm = llm
        self.summary = ""
        self.recent = []
        self.max_tokens = max_tokens
    
    def add(self, message):
        self.recent.append(message)
        
        if self._total_tokens() > self.max_tokens:
            # 将最早的几条消息压缩进摘要
            to_compress = self.recent[:4]
            self.recent = self.recent[4:]
            
            self.summary = self.llm.generate(
                f"将以下对话摘要与新内容合并为一份简洁摘要，"
                f"保留关键事实、用户偏好和未完成的任务：\n\n"
                f"已有摘要:\n{self.summary}\n\n"
                f"新对话:\n{format_messages(to_compress)}"
            )
    
    def get_context(self):
        return f"[对话摘要] {self.summary}\n\n[最近对话]\n{self.recent}"
```

```
完整对话:    [T1] [T2] [T3] [T4] [T5] [T6] [T7] [T8] [T9] [T10]
动态摘要:    [T1-T6 的摘要: "用户想订北京到上海的机票，偏好经济舱..."]
             + [T7] [T8] [T9] [T10]
             ↑ 信息有损但保留了关键内容
```

**优点**：
- 早期信息**保留核心**——关键事实和用户偏好不会丢失
- 信息密度高（摘要的信噪比远高于原始对话）

**缺点**：
- 需要额外的 LLM 调用（增加延迟和成本）
- 摘要本身是有损的——细节和精确措辞会丢失
- 摘要质量依赖 LLM 能力（可能遗漏关键信息或引入错误）

### 核心对比

| 维度 | 滑动窗口 | 动态摘要 |
|---|---|---|
| **信息保留** | 窗口外完全丢失 | 有损保留核心信息 |
| **额外成本** | 零 | 每次压缩需调用 LLM |
| **额外延迟** | 零 | 摘要生成 200-500ms |
| **实现复杂度** | 极低 | 中等 |
| **适用场景** | 短对话、信息时效性强 | 长对话、需要长期上下文 |
| **失败模式** | 忘记早期关键信息 | 摘要遗漏或歪曲信息 |

### 方案三：混合策略（实践中最常用）

```python
class HybridContextManager:
    def __init__(self, llm, max_context_tokens=8000):
        self.llm = llm
        self.max_context = max_context_tokens
        self.summary = ""
        self.pinned = []     # 关键信息（永不丢弃）
        self.recent = []     # 最近对话（滑动窗口）
    
    def add(self, message):
        # 检测关键信息并固定
        if self._is_critical(message):
            self.pinned.append(message)
        
        self.recent.append(message)
        
        # 超限时：滑动窗口 + 对溢出部分做摘要
        if self._total_tokens() > self.max_context:
            overflow = self.recent[:len(self.recent)//2]
            self.recent = self.recent[len(self.recent)//2:]
            self.summary = self.llm.summarize(self.summary, overflow)
    
    def get_context(self):
        # 高注意力区域放关键信息
        return [
            {"role": "system", "content": self.system_prompt},
            {"role": "system", "content": f"对话摘要: {self.summary}"},
            {"role": "system", "content": f"关键信息: {self.pinned}"},
            *self.recent,  # 最近对话放最后（recency bias 友好）
        ]
    
    def _is_critical(self, message):
        # 用户明确表达偏好、确认关键决策、提供个人信息
        critical_patterns = ["记住", "以后都", "我的地址是", "密码是"]
        return any(p in message["content"] for p in critical_patterns)
```

**布局策略**：利用 LLM 的 U 形注意力分布——摘要和关键信息放在前面（system 区域，高注意力），最近对话放在后面（高注意力），中间区域放次要信息。

### 其他补充方案

| 方案 | 思路 | 适用场景 |
|---|---|---|
| **RAG 式记忆检索** | 将对话历史存入向量库，按需检索相关片段 | 超长对话（100+ 轮） |
| **分层压缩** | 不同时间段用不同压缩率（越早压越狠） | 长时间跨度的交互 |
| **Token 级修剪** | 用注意力分数修剪低注意力 token | 研究阶段，工程落地少 |

- 标签: `context-management`, `sliding-window`, `dynamic-summary`, `context-overflow`, `memory`
- 记录于: 2026-06-20

## Q: 长上下文对话中如何让 Agent 不忘记关键信息？除了向量检索还有什么方法？

### 为什么会"忘记"

LLM 在长上下文中遗失信息的三个根本原因（详见 intent-recognition/challenges.md 中的分析）：
1. **U 形注意力分布**：开头和结尾注意力高，中间低
2. **信息稀释**：50 token 的关键信息淹没在 5000+ token 的后续对话中
3. **Recency bias**：模型过度依赖最近几轮

### 方法一：关键信息锚定（Information Pinning）

**核心思想**：将关键信息从"中间"移到"高注意力区域"（开头或结尾）。

```python
class InformationPinner:
    def __init__(self):
        self.pinned_facts = []  # 关键事实
    
    def pin(self, fact: str, source_turn: int):
        self.pinned_facts.append({
            "content": fact,
            "source": f"Turn {source_turn}",
            "pinned_at": datetime.now(),
        })
    
    def build_messages(self, system_prompt, conversation):
        # 关键信息注入到 system prompt 末尾（高注意力区域）
        pinned_section = "\n".join(
            f"- {f['content']}（来自{f['source']}）"
            for f in self.pinned_facts
        )
        
        enhanced_system = (
            f"{system_prompt}\n\n"
            f"## 当前对话的关键信息（务必牢记）\n"
            f"{pinned_section}"
        )
        
        return [
            {"role": "system", "content": enhanced_system},
            *conversation,
            # 也在最后一条 user 消息前再提醒一次（利用 recency bias）
            {"role": "system", "content": f"提醒：{pinned_section}"},
        ]
```

**何时触发 pin**：
- 用户明确说了偏好或约束（"我要经济舱"、"预算不超过 5000"）
- Agent 确认了关键决策（"已确认退款金额为 299 元"）
- 多步任务中的中间结果（"第一步查询结果：航班 CA1234"）

### 方法二：结构化状态摘要（Structured State Summary）

不是保留原始对话，而是维护一个**结构化的状态对象**，每轮更新：

```python
class ConversationState:
    def __init__(self):
        self.task_goal = ""            # 用户的核心目标
        self.confirmed_facts = {}      # 已确认的事实
        self.pending_questions = []    # 待解决的问题
        self.decisions_made = []       # 已做出的决策
        self.constraints = []          # 用户的约束条件
    
    def update(self, llm, latest_turn: str):
        # 用 LLM 从最新对话中提取结构化信息更新状态
        update = llm.extract(
            f"当前状态:\n{self.to_json()}\n\n"
            f"最新对话:\n{latest_turn}\n\n"
            f"请更新状态中的各字段（只更新有变化的）。"
        )
        self.merge(update)
    
    def to_prompt(self) -> str:
        return f"""## 当前任务状态
目标: {self.task_goal}
已确认: {json.dumps(self.confirmed_facts, ensure_ascii=False)}
待解决: {self.pending_questions}
已决策: {self.decisions_made}
约束: {self.constraints}"""
```

**优势**：状态摘要的信噪比远高于原始对话。200 token 的状态摘要可以替代 2000 token 的对话历史，且关键信息不会丢失。

### 方法三：周期性重述（Periodic Recap）

每隔 N 轮（如 5 轮），在 prompt 中插入一段系统消息，总结到目前为止的关键内容：

```python
def maybe_inject_recap(messages, turn_count, llm):
    if turn_count % 5 == 0 and turn_count > 0:
        recap = llm.generate(
            f"请用 3-5 句话总结以下对话的关键信息、未完成的任务和用户的核心需求:\n"
            f"{format_recent_turns(messages, last_n=10)}"
        )
        messages.insert(-1, {
            "role": "system",
            "content": f"[对话回顾] {recap}"
        })
```

### 方法四：分层上下文管理

```
┌──────────────────────────────────────┐
│  Layer 0: System Prompt              │  ← 始终在上下文头部
│  - Agent 身份和能力                   │
│  - 工具定义                           │
├──────────────────────────────────────┤
│  Layer 1: Pinned 关键信息            │  ← 紧随 system prompt
│  - 用户偏好/约束                      │
│  - 已确认的关键事实                   │
│  - 当前任务状态                       │
├──────────────────────────────────────┤
│  Layer 2: 历史摘要（压缩）            │  ← 中间（低注意力区域放低密度信息）
│  - 早期对话的摘要                     │
├──────────────────────────────────────┤
│  Layer 3: 最近 N 轮原文              │  ← 尾部（高注意力区域放最新信息）
│  - 最近的对话详情                     │
│  - 最新工具调用结果                   │
├──────────────────────────────────────┤
│  Layer 4: 再次提醒                   │  ← 最后注入（利用 recency bias）
│  - 重复关键约束                       │
└──────────────────────────────────────┘
```

### 方法五：外部记忆存储（非向量检索的方式）

除了向量检索，还有其他外部存储方式：

| 方式 | 机制 | 优点 | 缺点 |
|---|---|---|---|
| **Key-Value 存储** | 显式 key 检索（用户名→偏好） | 精确、快速 | 需预定义 key 结构 |
| **知识图谱** | 实体-关系图上的遍历 | 支持关系推理 | 构建成本高 |
| **SQL 数据库** | 结构化查询 | 精确查询、支持聚合 | 需定义 schema |
| **对话日志检索** | BM25 关键词搜索 | 精确关键词命中 | 无语义理解 |
| **Scratchpad** | Agent 维护的文本文件笔记 | 灵活、可读 | 需 Agent 主动维护 |

```python
# Scratchpad 模式：Agent 自己维护笔记
class AgentScratchpad:
    def __init__(self):
        self.notes = ""
    
    def update_note(self, content: str):
        """Agent 主动调用此工具记录关键信息"""
        self.notes += f"\n[{datetime.now().strftime('%H:%M')}] {content}"
    
    def read_notes(self) -> str:
        return self.notes
```

让 Agent 有一个 `save_note` 工具——当它判断某个信息重要时，主动保存到笔记中。后续需要时调用 `read_notes` 检索。

### 各方法对比

| 方法 | 额外 LLM 调用 | 信息保真度 | 适用场景 |
|---|---|---|---|
| **信息锚定** | 无 | 高（原文保留） | 关键约束/偏好 |
| **结构化状态** | 每轮 1 次 | 中（提炼后） | 任务型对话 |
| **周期性重述** | 每 5 轮 1 次 | 中 | 通用长对话 |
| **分层上下文** | 压缩时 1 次 | 中高 | 所有场景 |
| **向量检索** | 检索时 0 次 | 高（原文） | 超长对话/跨会话 |
| **Scratchpad** | Agent 自主 | 取决于 Agent | 需要精确记忆的任务 |

**实践建议**：信息锚定 + 分层上下文是性价比最高的组合——零额外 LLM 调用，通过合理安排信息位置就能显著减少遗忘。

- 标签: `long-context`, `information-pinning`, `structured-state`, `context-layering`, `scratchpad`
- 记录于: 2026-06-20
