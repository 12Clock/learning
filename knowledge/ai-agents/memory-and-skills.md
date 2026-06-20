# Memory 与 Skills

> 涵盖：Memory 系统设计、奖励机制、Skill 体系设计与实现。

## Q: Memory Agent 的奖励设计有哪些主流方案？

不同框架对 Memory Agent 的"奖励"设计差异很大，大致分为三类范式：

**1. 启发式评分（无显式 RL）**

- **Generative Agents（Stanford, 2023）**：对每条记忆打「重要性分数」（1-10，LLM 自评），检索时按 `score = α_recency · recency + α_relevance · relevance + α_importance · importance` 加权排序（所有 α=1，min-max 归一化到 [0,1]）。Recency 用指数衰减（decay=0.995），Relevance 用 embedding 余弦相似度。反思机制：当近期事件重要性分数之和超过阈值（150），触发生成高层反思并存回 memory stream，形成记忆树结构。
- **MemGPT / Letta（2023）**：完全不用奖励信号。把 memory 管理建模为 tool calling——agent 通过 `core_memory_append`、`archival_memory_search` 等函数主动管理三层存储（main context / recall / archival）。记忆决策是推理的一部分，不是奖励驱动的。

**2. 环境反馈 + 自我反思**

- **Reflexion（2023）**：用「语言化的自我反思」充当奖励。任务失败后 agent 生成一段反思文本（"我应该先检查边界条件"），存入 episodic memory buffer 供下次尝试参考。奖励信号 = 环境的二元成功/失败 + LLM 自评。
- **Voyager（2023）**：技能执行的成功/失败作为隐式奖励。成功验证的技能（代码）被存入 skill library，失败的被丢弃或重写。奖励 = 代码能否在 Minecraft 里跑通。

**3. 强化学习驱动的记忆管理（2025-2026 前沿）**

这一领域在 2025-2026 年进展迅速，以下是关键工作：

| 系统 | 奖励类型 | 核心信号 | RL 算法 |
|---|---|---|---|
| **Memory-R1**（2025.08） | 纯结果驱动 | QA F1/精确匹配 | GRPO / PPO |
| **Mem-alpha**（2025.09） | 四项复合奖励 | 准确率 + 格式 + 压缩率 + 内容质量 | GRPO |
| **MemRL**（2026.01） | 基于价值的 | 蒙特卡洛 Q-value 评估记忆效用 | Q-learning |
| **EMPO2**（ICLR 2026） | 混合 on/off-policy | GRPO + 记忆 buffer 中的 tips 复用 | 扩展 GRPO |
| **AgeMem**（2025） | 分步 RL | 三阶段：监督预热 → 任务级 RL → 步级 GRPO | PPO + GRPO |
| **Meta-Cognitive MPO**（2026.05） | 自监督代理奖励 | **Belief Entropy**（信念熵） | Policy Optimization |
| **DeltaMem**（2026.04） | 基于距离 | Memory-based Levenshtein Distance | 自定义 RL |

关键细节：
- **Memory-R1**：两个专用 agent——Memory Manager（执行 ADD/UPDATE/DELETE/NOOP）+ Answer Agent（从 60 条 RAG 候选中预筛选）。仅用 152 条 QA 训练，跨三个 benchmark 超越强基线。
- **Mem-alpha**：四项奖励中 r4（内容质量）最关键。在 30K token 序列上训练，泛化到 400K+ token（13 倍泛化）。
- **MemRL**：冻结 LLM 权重，仅训练记忆的 Q-value。两阶段检索：先语义过滤，再按学到的效用分排序。
- **AgeMem**：将 store/retrieve/update/summarize/discard 五种记忆操作建模为 tool call，端到端 RL 优化。学到非直觉策略：主动在上下文填满前做摘要。
- **Meta-Cognitive MPO**：认为基于结果的 RL 无法定位长程任务中记忆质量下降的位置，提出 **Belief Entropy** 作为中间代理奖励——优化记忆诱导的信念清晰度而非最终轨迹成功。

**关键趋势**：
1. 结果驱动 RL 仍是主流，但正在向更精细的信号演进（信念熵、编辑距离）
2. 冻结 LLM + 可学习记忆策略成为共识，避免灾难性遗忘
3. 记忆操作越来越多被建模为 tool call，通过 RL 优化调用策略

**核心权衡**：启发式方案简单但不自适应；自我反思方案灵活但依赖 LLM 自评质量；RL 方案自适应但训练成本高。工业界仍以启发式 + 自我反思为主流，但 RL 方案在 2025-2026 快速成熟，已在多个 benchmark 上显著超越传统方案。

- 标签: `memory-agent`, `reward-design`, `RL`, `LLM-agent`
- 记录于: 2026-06-18

## Q: Claude Code 的 Skill 是怎么设计的？为什么不用 Skill Creator？

**设计方式**：Claude Code 的 skill 是**纯 Markdown 文件 + YAML frontmatter**。一个 skill 就是一个 `.md` 文件，frontmatter 中声明 `name` 和 `description`（触发条件），正文写明工作流程和约束。skill 可以通过相对路径引用其他 `.md` 文件实现渐进式加载。

```yaml
---
name: my-skill
description: 触发场景描述...
---
# Skill 正文
工作流步骤...
```

**为什么不用 Skill Creator（自动生成工具）**：

1. **透明可控**：纯 Markdown 人类可直接阅读、编辑、review，不存在黑箱生成的逻辑。
2. **版本管理友好**：`.md` 文件天然适合 git 追踪，diff 可读，可 code review。
3. **零依赖**：不需要额外工具链，任何文本编辑器都能创建 skill。
4. **格式即约束**：Markdown 的结构（标题、列表、代码块）本身就是对 LLM 足够好的指令格式，不需要更复杂的 DSL。
5. **可组合**：通过文件引用（`references/xxx.md`）实现模块化，比自动生成工具更灵活。
6. **Skill Creator 的问题**：自动生成的 skill 质量取决于生成 prompt 的质量，容易产生冗余/不精确的指令，反而不如人工编写的简洁明确。

**渐进式加载架构（Progressive Disclosure）**——这是选择 Markdown 而非自动生成的关键技术原因：
- **Level 1（元数据）**：YAML frontmatter（~100 tokens/skill），启动时加载到 system prompt
- **Level 2（指令）**：SKILL.md 正文（~5K tokens），仅在触发时加载
- **Level 3（资源）**：引用文件、脚本等，按需加载，未使用时零 token 消耗

100 个 skill 的元数据只占 ~10K tokens；如果把全部内容内联注入则会导致 context 爆炸。文件系统模型让脚本可以通过 Bash 执行而不加载代码——只有输出消耗 token。

本质上 skill 就是"给 agent 的 SOP 文档"，Markdown 已经是写 SOP 最成熟的格式。

- 标签: `claude-code`, `skill`, `design-decision`
- 记录于: 2026-06-18

## Q: Claude Code 的 Memory 和 OpenClaw 的 Memory 分别是怎么做的？

### Claude Code 的 Memory

三层架构，**纯文件注入，无向量数据库**：

| 层 | 谁写的 | 存什么 | 持久性 |
|---|---|---|---|
| **CLAUDE.md** | 人工编写 | 项目规则、编码规范、架构约定 | 每次会话加载 |
| **Auto Memory** | Agent 自动写 | 学到的模式、调试经验、偏好 | 跨会话持久，机器本地 |
| **Session Memory** | Agent 自动写 | 会话内对话摘要 | 会话内有效，结束即重置 |

- **加载策略**：CLAUDE.md 全量加载；Auto Memory 加载 `MEMORY.md` 前 200 行（≤25KB），topic 文件按需加载；Session Memory 会话内追踪。
- **层级优先级**：managed policy → user → project → local，越靠近工作目录优先级越高。
- **无检索模型**：不做 embedding 或语义搜索，直接把文件内容注入 system prompt / context window。简单但消耗 token。

### OpenClaw 的 Memory

三层架构，**文件 + 向量数据库混合**：

| 层 | 内容 | 持久性 |
|---|---|---|
| **MEMORY.md** | 关于用户的长期事实（偏好、重要信息） | 跨会话持久 |
| **Daily Notes** | 按日期存储的会话记录（`memory/YYYY-MM-DD.md`） | 跨会话持久 |
| **SOUL.md** | Agent 人格、名称、沟通风格 | 永久 |

- **向量检索**：Memory 文件和会话记录被 embedding 后存入向量数据库（基于 `sqlite-vec`），检索时做语义搜索，支持 GPU 加速。
- **上下文压缩（Context Compaction）**：上下文窗口将满时，自动摘要老对话，保留语义同时减少 token。
- **社区扩展**：存在 12 层 memory 架构方案，引入知识图谱（3000+ 事实节点）、activation/decay 衰减机制、领域专用 RAG。

### 核心差异

| 维度 | Claude Code | OpenClaw |
|---|---|---|
| **检索方式** | 文件直接注入 context | 向量 embedding + 语义搜索 |
| **存储格式** | 纯 Markdown 文件 | Markdown + SQLite 向量库 |
| **记忆筛选** | 靠文件大小限制（200 行）+ 按需加载 | embedding 相似度排序 |
| **人格系统** | 无（通过 CLAUDE.md 间接实现） | 专用 SOUL.md |
| **扩展性** | MCP 协议扩展工具 | 社区插件 + 知识图谱方案 |
| **设计哲学** | 简单、透明、无魔法 | 功能丰富、自动化程度高 |

Claude Code 选择"文件即记忆"的极简方案，牺牲检索精度换取透明度和可调试性。OpenClaw 走向量检索路线，记忆容量更大但系统更复杂。

- 标签: `claude-code`, `openclaw`, `memory-system`, `comparison`
- 记录于: 2026-06-18

## Q: Agent 的 Skills 功能原理是什么？怎么设计和实现 Skills 体系？

### Skills 的本质

Skill 本质上是**预定义的、可复用的任务执行模板**——告诉 Agent "遇到某类任务时，按这套流程走"。可以理解为 Agent 的 SOP（标准操作流程）。

### 核心原理

**触发机制**：
- **显式触发**：用户输入 `/skill-name` 直接调用
- **隐式触发**：Agent 根据用户意图自动匹配。匹配依据是 skill 的 `description` 字段与用户输入的语义相似度。
- **条件触发**：根据上下文条件（如当前文件类型、项目结构）自动激活

**执行流程**：
```
Skill 被触发 → 加载 Skill 定义 → 注入上下文 → Agent 按指令执行 → 返回结果
```

### 设计方法论

**1. 定义层级（Progressive Disclosure）**

```
Level 0: 注册表         skill 名称 + 描述（~100 tokens/skill）→ 常驻 system prompt
Level 1: 入口文件       SKILL.md 核心流程（~1-5K tokens）→ 触发时加载
Level 2: 引用资源       详细规则、模板、示例（按需加载）→ 执行时按需读取
```

这种分层的关键在于：**未触发的 skill 几乎零 token 开销**。

**2. Skill 定义结构**

一个好的 skill 定义应包含：
- **名称与描述**：用于匹配和发现
- **前置条件**：在什么场景下使用（可选）
- **执行步骤**：明确的 step-by-step 流程
- **约束与边界**：不做什么、在哪个范围内操作
- **引用资源**：详细规则文件的路径（按需加载）

**3. Skill 发现与匹配**

两种常见策略：
- **基于描述匹配**：启动时把所有 skill 的描述注入 system prompt，LLM 自行判断是否触发。简单但描述质量决定匹配效果。
- **基于 embedding 检索**：把 skill 描述向量化，用户输入与描述做相似度匹配，Top-K 候选注入上下文。适合 skill 数量 >50 的场景。

**4. Skill 组合**

复杂任务可以编排多个 skill：
- **顺序组合**：skill A 的输出作为 skill B 的输入
- **条件组合**：根据中间结果决定下一步调用哪个 skill
- **嵌套调用**：一个 skill 内部调用另一个 skill

### 实现示例：本仓库的 knowledge-collection skill

```
skills/knowledge-collection/
  SKILL.md                    # 入口：定义核心工作流
  references/
    categorization.md         # 引用：分类规则（按需加载）
    format.md                 # 引用：格式规范（按需加载）
```

工作流：读 INDEX → 定位分类 → 写 Q&A → 检查是否需要细分 → 更新 INDEX

这个设计体现了几个关键原则：
- 入口 SKILL.md 精简（只有核心步骤），详细规则在 references/ 中按需加载
- Skill 绑定特定目录（`knowledge/`），明确操作边界
- 支持随内容增长自动细分类别，体现了 skill 的自适应能力

### 设计陷阱

1. **Skill 过多导致 token 膨胀**：50 个 skill 的描述就是 5K tokens。需要做好层级化加载。
2. **描述歧义导致误触发**：skill 的描述需要精准，避免过于宽泛。
3. **Skill 间冲突**：多个 skill 都能匹配同一意图时，需要优先级或置信度排序。
4. **过度 Skill 化**：不是所有行为都需要封装为 skill。只有**重复出现的、步骤可标准化的任务**才值得做 skill。

- 标签: `skill-design`, `agent-skill`, `progressive-disclosure`, `architecture`
- 记录于: 2026-06-19

## Q: Skill 过多会导致检索精确度降低，如何解决？

### 问题本质

Skill 数量增长带来三层递进的问题：

1. **Token 膨胀**：每个 skill 的描述（name + description）需要注入 system prompt 以供 LLM 匹配。100 个 skill ≈ 10K tokens，500 个 skill ≈ 50K tokens——大量不相关的 skill 描述挤占上下文窗口，干扰模型对当前任务的注意力。
2. **语义混淆**：skill 数量越多，描述之间的语义重叠越多。"代码审查" vs "代码质量检查" vs "代码安全扫描"——LLM 难以区分细微差异，导致误触发。
3. **长尾失效**：常用 skill 被频繁匹配（模型对它们的描述"记住了"），低频 skill 在大量描述中被淹没，几乎永远不会被触发。

### 解决方案体系

**方案一：分层检索（Hierarchical Retrieval）**

将 skill 组织为树状结构，分两级匹配：

```
Level 0: 领域分类（5-10 个）          ← 全部注入 system prompt（~1K tokens）
Level 1: 具体 skill（每个领域 10-30 个） ← 只加载匹配领域的 skill

用户输入 → 匹配领域 → 只加载该领域的 skill 列表 → 匹配具体 skill
```

```yaml
domains:
  - name: code-quality
    description: 代码审查、测试、覆盖率相关
    skills: [code-review, security-review, test-coverage, lint-fix]
  - name: deployment
    description: 部署、发布、CI/CD 相关
    skills: [deploy-staging, deploy-prod, rollback, canary-release]
```

效果：搜索空间从 N 降到 ~N/K（K 为领域数），token 消耗从全量 skill 降到一个领域的 skill。

**方案二：Embedding 检索 + 重排序（Retrieve & Rerank）**

当 skill 数量 >50 时，不再把所有描述塞进 prompt，而是用向量检索做预筛选：

```python
class SkillRouter:
    def __init__(self, skills: list[Skill]):
        # 离线：将所有 skill 描述 embedding 化
        self.index = build_vector_index([s.description for s in skills])
        self.skills = skills
    
    def match(self, user_query: str, top_k: int = 5) -> list[Skill]:
        # Step 1: 向量检索 Top-K 候选
        candidates = self.index.search(embed(user_query), top_k=top_k)
        
        # Step 2: 将 Top-K 候选的完整描述注入 LLM，让 LLM 做精确选择
        selected = llm_select(user_query, [self.skills[i] for i in candidates])
        return selected
```

**关键设计**：
- 向量检索做**召回**（高召回、可接受误差），LLM 做**精排**（高精度）
- 只有 Top-K（通常 3-5 个）的 skill 描述进入 context，而非全量
- 向量索引几乎零延迟（<10ms），不增加用户可感知的延迟

**方案三：Skill 作为 Tool 定义（Structured Discovery）**

将 skill 注册为 LLM 的 tool（function calling），利用模型原生的 tool 选择能力：

```json
{
  "type": "function",
  "function": {
    "name": "code_review",
    "description": "Review code changes for correctness bugs and style issues",
    "parameters": {
      "effort": {"type": "string", "enum": ["low", "medium", "high"]}
    }
  }
}
```

**优势**：
- Tool 定义有结构化的 name + description + parameters，比自然语言描述更精确
- LLM 对 tool calling 的训练做了专门优化，匹配精度高于从自由文本中解析意图
- 支持参数约束（enum、required），进一步减少歧义

**劣势**：
- Tool 定义也消耗 token（每个约 100-200 tokens）
- 数量上限受模型支持的 tool 数量限制（通常 64-128 个）

**方案四：延迟加载 / Deferred Tools（Claude Code 的做法）**

Claude Code 在 skill/tool 数量多时的策略：

```
启动时：只加载 skill 名称列表（零参数定义，~10 tokens/skill）
触发时：通过 ToolSearch 按需加载完整 schema

即：名称常驻 → 定义延迟加载
```

这样 500 个 skill 只消耗 ~5K tokens（仅名称），而非 ~100K tokens（完整定义）。用户输入匹配到名称后，再动态加载该 skill 的完整定义和参数。

**方案五：描述优化 + 消歧规则**

在 skill 数量有限（<50）但存在语义重叠时，优化描述本身：

**精确化描述**：
```yaml
# 差：过于宽泛
- name: review
  description: 审查代码

# 好：明确边界和使用场景
- name: code-review
  description: 审查当前分支 diff 中的正确性 bug 和代码质量问题。不做安全审查（用 security-review）。
```

**添加反向约束（Negative Examples）**：
```yaml
- name: code-review
  description: >
    审查代码变更的正确性和质量。
    NOT for: 安全漏洞扫描（用 security-review）、
    性能优化建议（用 perf-review）、
    文档检查（用 doc-review）。
```

**添加触发示例**：
```yaml
- name: code-review
  description: 审查代码变更
  trigger_examples:
    - "帮我 review 一下这个 PR"
    - "检查一下代码有没有 bug"
  non_trigger_examples:
    - "这个接口有没有安全风险"     # → security-review
    - "帮我写个单元测试"           # → test-gen
```

**方案六：热度感知 + 动态排序**

```python
class AdaptiveSkillRouter:
    def __init__(self, skills):
        self.skills = skills
        self.usage_count = defaultdict(int)      # 使用频次
        self.last_used = {}                       # 最近使用时间
    
    def get_prompt_skills(self, max_in_prompt: int = 20):
        # 高频 skill 常驻 prompt，低频 skill 走检索
        sorted_skills = sorted(
            self.skills,
            key=lambda s: self.usage_count[s.name],
            reverse=True
        )
        hot_skills = sorted_skills[:max_in_prompt]    # Top-20 常驻
        cold_skills = sorted_skills[max_in_prompt:]   # 其余走向量检索
        return hot_skills, cold_skills
```

高频 skill 注入 prompt（保证常用场景的匹配速度和精度），低频 skill 通过向量检索按需加载（节省 token，牺牲少量延迟）。

### 方案选择指南

| Skill 数量 | 推荐方案 | 理由 |
|---|---|---|
| <20 | 描述优化 + 消歧规则 | 数量少，全量注入 prompt 可接受 |
| 20-50 | Tool 定义 + 描述优化 | 利用 LLM 原生 tool calling 能力 |
| 50-200 | 分层检索 或 Embedding 检索 | 全量注入 token 成本过高 |
| 200+ | Embedding 检索 + 延迟加载 + 热度排序 | 多种策略组合 |

### 核心原则

1. **精度问题先从描述质量入手**——很多"检索不准"实际上是描述写得模糊，而非检索机制的问题
2. **减少候选比优化排序更有效**——从 200 个 skill 中选 1 个很难，从 5 个候选中选 1 个很容易
3. **分层是最通用的思路**——先粗筛再精排，每一层的搜索空间都可控
4. **监控触发准确率**——定期统计 skill 的触发准确率，针对低准确率的 skill 优化描述或合并

- 标签: `skill-retrieval`, `skill-routing`, `embedding-retrieval`, `scalability`, `deferred-loading`
- 记录于: 2026-06-19

## Q: Agent 的记忆机制怎么设计？短期记忆和长期记忆分别如何实现？

### 记忆分类

Agent 的记忆体系参照认知科学，通常分为三层：

```
┌─────────────────────────────────────────────┐
│  感知记忆（Sensory Buffer）                   │ ← 当前输入（用户最新消息）
│  ~当前 turn 的 raw input                     │
├─────────────────────────────────────────────┤
│  短期/工作记忆（Working Memory）              │ ← 当前任务上下文
│  ~对话历史 + 当前意图 + 槽位状态               │
│  容量有限，随任务结束释放                      │
├─────────────────────────────────────────────┤
│  长期记忆（Long-term Memory）                 │ ← 跨会话持久化
│  ~用户偏好 + 历史交互摘要 + 学到的知识          │
│  持久存储，按需检索                            │
└─────────────────────────────────────────────┘
```

### 短期记忆（Working Memory）

**职责**：维持当前对话/任务的上下文，让 Agent 在多轮交互中保持连贯。

**实现方式一：对话缓冲区（Conversation Buffer）**

```python
class ConversationBufferMemory:
    def __init__(self, max_turns=20):
        self.messages = []
        self.max_turns = max_turns
    
    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        # 超出限制时丢弃最早的消息
        if len(self.messages) > self.max_turns * 2:
            self.messages = self.messages[-self.max_turns * 2:]
    
    def get_context(self) -> list:
        return self.messages
```

简单直接，但随对话增长会超出 LLM 上下文窗口。

**实现方式二：对话摘要缓冲区（Summary Buffer）**

```python
class ConversationSummaryBufferMemory:
    def __init__(self, llm, max_token_limit=2000):
        self.llm = llm
        self.summary = ""          # 早期对话的摘要
        self.recent_messages = []  # 最近几轮的原文
        self.max_token_limit = max_token_limit
    
    def add(self, role: str, content: str):
        self.recent_messages.append({"role": role, "content": content})
        
        # 超出 token 限制时，将最早的消息压缩进摘要
        while self._count_tokens() > self.max_token_limit:
            oldest = self.recent_messages.pop(0)
            self.summary = self.llm.summarize(
                f"已有摘要:\n{self.summary}\n\n新消息:\n{oldest}"
            )
    
    def get_context(self) -> str:
        return f"对话摘要:\n{self.summary}\n\n最近对话:\n{self.recent_messages}"
```

**实现方式三：任务状态对象（Structured State）**

```python
class AgentWorkingMemory:
    current_intent: str = None
    slots: dict = {}              # 已填充的参数
    pending_actions: list = []    # 待执行的操作
    tool_results: dict = {}       # 工具调用结果
    clarification_needed: list = [] # 需要追问的信息
    
    def to_prompt_context(self) -> str:
        return f"""当前意图: {self.current_intent}
已知参数: {json.dumps(self.slots, ensure_ascii=False)}
待执行: {self.pending_actions}
工具结果: {json.dumps(self.tool_results, ensure_ascii=False)}"""
```

结构化状态比原始对话历史更高效——信噪比高，token 消耗少。

### 长期记忆（Long-term Memory）

**职责**：跨会话记住用户偏好、历史交互模式和学到的知识。

**实现方式一：向量数据库存储**

```python
class VectorLongTermMemory:
    def __init__(self, vectorstore, llm):
        self.vectorstore = vectorstore  # Milvus / Pinecone / Chroma
        self.llm = llm
    
    def save_memory(self, session_id: str, conversation: list):
        # 会话结束时，提取关键信息存入长期记忆
        key_info = self.llm.extract(
            f"从以下对话中提取需要长期记住的信息（用户偏好、重要事实、待办事项）:\n"
            f"{conversation}"
        )
        
        for item in key_info:
            self.vectorstore.add(
                text=item.content,
                metadata={
                    "session_id": session_id,
                    "timestamp": datetime.now().isoformat(),
                    "type": item.type,  # preference / fact / todo
                }
            )
    
    def recall(self, query: str, top_k=5) -> list:
        # 根据当前对话内容检索相关的长期记忆
        return self.vectorstore.similarity_search(query, k=top_k)
```

**实现方式二：结构化用户画像**

```python
class UserProfile:
    user_id: str
    preferences: dict = {
        "language": "zh-CN",
        "response_style": "concise",
        "timezone": "Asia/Shanghai",
    }
    facts: list = [
        {"content": "用户是 Python 开发者", "confidence": 0.9},
        {"content": "用户偏好 CLI 工具", "confidence": 0.85},
    ]
    interaction_stats: dict = {
        "total_sessions": 42,
        "most_used_tools": ["code_search", "file_edit"],
        "avg_session_length": 15,
    }
```

**实现方式三：知识图谱**

```
User_A --[prefers]--> Python
User_A --[works_at]--> Company_X
User_A --[asked_about]--> "Redis 缓存策略" (2026-06-15)
User_A --[asked_about]--> "Agent 架构" (2026-06-19)
```

适合关系复杂、需要推理的场景（"用户之前问过什么相关问题？"）。

### 记忆的读写时机

```
会话开始:
  1. 从长期记忆中检索与用户相关的背景信息
  2. 注入 system prompt（"该用户偏好简洁回答，是 Python 开发者"）

对话进行中:
  3. 短期记忆实时更新（每轮追加消息或更新状态）
  4. 触发关键事件时写入长期记忆（用户明确表达偏好："以后别用emoji"）

会话结束:
  5. 将短期记忆中的关键信息提炼后写入长期记忆
  6. 清空短期记忆
```

### 实际架构

```python
class MemoryManager:
    def __init__(self, user_id):
        self.working = AgentWorkingMemory()
        self.buffer = ConversationSummaryBufferMemory(llm, max_token_limit=3000)
        self.long_term = VectorLongTermMemory(vectorstore, llm)
        self.profile = load_user_profile(user_id)
    
    def build_context(self, current_query: str) -> str:
        # 组装完整的记忆上下文
        long_term_recall = self.long_term.recall(current_query, top_k=3)
        
        return f"""
## 用户画像
{self.profile.to_prompt()}

## 相关长期记忆
{format_memories(long_term_recall)}

## 当前任务状态
{self.working.to_prompt_context()}

## 对话历史
{self.buffer.get_context()}
"""
```

**核心原则**：短期记忆保证连贯性（会话内），长期记忆保证个性化（跨会话）。两者的分界线是**会话结束**这个事件——结束时提炼、持久化、清空。

- 标签: `memory`, `working-memory`, `long-term-memory`, `user-profile`, `vector-store`, `agent-memory`
- 记录于: 2026-06-20
