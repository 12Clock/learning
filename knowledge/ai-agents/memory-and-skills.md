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

## Q: 短期记忆深入：实现细节、触发时机、Prompt 注入位置与信息保留策略

### 短期记忆如何做的？

短期记忆的核心是**在有限上下文窗口内维持对话连贯性**。工程上有三种递进方案：

**方案一：全量保留（Conversation Buffer）**
- 直接把所有历史消息拼接进 prompt，最简单但最浪费 token
- 适用场景：短对话（<10 轮）、上下文窗口足够大（128K+）

**方案二：滑动窗口 + 摘要（Summary Buffer）**
- 保留最近 K 轮原文，更早的对话压缩成摘要
- 实现关键：摘要是**增量式**的——每次不是对全部历史重新摘要，而是把「旧摘要 + 新滑出的消息」合并更新

```python
class SummaryBufferMemory:
    def compress(self):
        while self.token_count() > self.limit:
            oldest = self.recent.pop(0)
            self.summary = llm(
                f"将以下新消息合并到已有摘要中，保留关键信息：\n"
                f"已有摘要：{self.summary}\n"
                f"新消息：{oldest}"
            )
```

**方案三：结构化状态机（Structured Working Memory）**
- 不保留对话原文，而是维护一个**结构化状态对象**
- 每轮对话后，从用户输入中提取信息更新到对应字段

```python
working_memory = {
    "current_intent": "订机票",
    "slots": {"出发地": "北京", "目的地": "上海", "日期": None},
    "confirmed": False,
    "tool_results": {"flight_search": [...]},
    "pending_clarification": ["出发日期"]
}
```

生产系统通常**混合使用方案二和三**：结构化状态跟踪任务进度，摘要缓冲保留对话语气和未结构化的细节。

### 什么时候触发短期记忆提取？

触发时机分为三类：

| 触发类型 | 条件 | 做什么 |
|---|---|---|
| **Token 阈值触发** | `token_count > max_limit * 0.8` | 压缩最早的消息为摘要 |
| **轮次触发** | 每 N 轮（通常 3-5 轮） | 增量更新摘要 |
| **事件触发** | 意图切换、任务完成、工具调用完成 | 归档当前任务上下文，重置 working memory |

实际设计中最常用的是 **Token 阈值触发**，因为不同对话的每轮 token 量差异很大（一轮可能 100 token 也可能 5000 token），按轮次触发不够精确。

```python
def should_compress(self) -> bool:
    usage_ratio = self.token_count() / self.max_tokens
    if usage_ratio > 0.8:
        return True
    if self.intent_changed():
        return True
    return False
```

Claude Code 的做法是**被动触发**：当上下文接近窗口上限时，系统自动对历史消息做压缩（context compaction），压缩后的摘要 + 剩余未压缩的消息作为下一个窗口的输入。

### 短期记忆拼接在 Prompt 的哪里？

不同类型的短期记忆注入位置不同：

```
┌──────────────────────────────────────────────┐
│  System Prompt                                │
│  ├── 角色定义 & 全局约束                        │
│  ├── 【用户画像 / 长期记忆检索结果】 ← 长期记忆   │
│  └── 【当前任务状态】              ← 结构化状态   │
├──────────────────────────────────────────────┤
│  【对话摘要】                      ← 压缩后的历史 │
├──────────────────────────────────────────────┤
│  Message History（最近 K 轮原文）    ← 滑动窗口   │
│  ├── user: ...                                │
│  ├── assistant: ...                           │
│  ├── user: ...                                │
│  └── assistant: ...                           │
├──────────────────────────────────────────────┤
│  Current User Message               ← 当前输入  │
└──────────────────────────────────────────────┘
```

**关键设计原则**：
1. **结构化状态放 System Prompt 末尾**——确保模型每次都能"看到"任务进度，不会被对话历史淹没
2. **对话摘要放在原文之前**——摘要提供背景，原文提供细节，阅读顺序自然
3. **最近 K 轮保留原文**——最近的对话对当前决策最关键，不能被摘要损失细节
4. **长期记忆放 System Prompt**——它是"预加载"的背景知识，不是对话流的一部分

### 每轮都做摘要会不会丢失信息？

**会，一定会丢失信息。** 关键是控制丢什么、保留什么。

**信息丢失的类型与应对**：

| 丢失类型 | 风险 | 应对策略 |
|---|---|---|
| **语气/情感** | 摘要会丢失用户的情绪表达 | 摘要 prompt 中要求保留情感标记 |
| **否定信息** | "用户说不要 X" 容易被摘要忽略 | 显式要求提取"用户明确拒绝的内容" |
| **具体数值** | "预算 5000 元" 可能被泛化为"有限预算" | 摘要 prompt 要求保留所有具体数值和实体 |
| **上下文关联** | "前面那个方案" 的指代关系丢失 | 摘要时解析指代，替换为具体内容 |

**降低信息丢失的工程手段**：

1. **不要每轮都摘要**——只在 token 阈值超过 80% 时触发，尽量保留原文
2. **摘要 prompt 中指定保留项**：
   ```
   请将以下对话压缩为摘要，必须保留：
   - 所有具体的数值、日期、名称
   - 用户明确表达的偏好和拒绝
   - 未完成的待办事项
   - 关键决策及其原因
   可以省略：寒暄、重复确认、无关闲聊
   ```
3. **关键信息提取到结构化字段**——不依赖摘要保留，而是实时写入 working memory 的 slots
4. **双轨保留**：摘要只是降级方案，关键信息同时写入结构化状态（slots/facts），即使摘要丢失了某条信息，结构化字段里还有

**核心认知**：摘要不是"无损压缩"，而是"有策略的有损压缩"。设计好"保留策略"比追求"不丢信息"更现实。

- 标签: `short-term-memory`, `summary-buffer`, `prompt-injection`, `context-compression`, `information-retention`
- 记录于: 2026-06-20

## Q: 短期记忆的设计流程、历史对话存储与长短期记忆边界划分

### 短期记忆的设计流程

```
Step 1: 确定容量约束
  ├── 模型上下文窗口大小（8K / 32K / 128K / 200K）
  ├── 分配给短期记忆的 token 预算（通常占总窗口的 40-60%）
  └── 剩余预算留给 system prompt、长期记忆检索、当前输入

Step 2: 选择存储策略
  ├── 纯对话缓冲（短对话 + 大窗口）
  ├── 滑动窗口 + 摘要（多轮对话 + 中等窗口）
  └── 结构化状态 + 摘要（复杂任务 + 任意窗口）

Step 3: 设计压缩触发机制
  ├── Token 阈值（推荐 80%）
  ├── 轮次阈值（备选）
  └── 事件触发（意图切换 / 任务完成）

Step 4: 设计摘要 prompt
  ├── 保留项清单（数值、偏好、待办、决策）
  ├── 可丢弃项（寒暄、重复确认）
  └── 输出格式（结构化 vs 自然语言）

Step 5: 设计信息提取管线
  ├── 每轮对话后：提取 slots / facts → 更新 working memory
  ├── 压缩触发时：生成增量摘要 → 替换旧消息
  └── 会话结束时：提炼关键信息 → 写入长期记忆

Step 6: 测试与调优
  ├── 多轮对话回归测试（第 20 轮还能引用第 3 轮的信息吗？）
  ├── 摘要质量评估（关键信息召回率）
  └── Token 使用监控（是否在预算内）
```

### 历史对话记录存在哪里？

分三层存储，各有取舍：

| 存储层 | 位置 | 内容 | 持久性 | 用途 |
|---|---|---|---|---|
| **热存储** | LLM 上下文窗口（内存） | 最近 K 轮原文 + 摘要 | 会话内有效 | 当前对话连贯性 |
| **温存储** | Redis / 内存数据库 | 当前会话完整对话日志 | 会话内有效，可设 TTL | 会话内回溯、调试 |
| **冷存储** | PostgreSQL / S3 / 日志系统 | 全部历史对话（结构化存储） | 永久 | 审计、分析、训练数据 |

```python
class ConversationStore:
    def __init__(self):
        self.hot = []                        # 当前窗口内的消息
        self.warm = redis.Redis()            # 当前会话完整记录
        self.cold = PostgresConversationLog() # 永久归档
    
    def add_message(self, session_id, message):
        self.hot.append(message)
        self.warm.rpush(f"session:{session_id}", json.dumps(message))
        self.warm.expire(f"session:{session_id}", 86400)  # 24h TTL
        self.cold.insert(session_id, message)
    
    def get_hot(self) -> list:
        return self.hot[-K:]  # 最近 K 轮
    
    def get_session_full(self, session_id) -> list:
        return [json.loads(m) for m in self.warm.lrange(f"session:{session_id}", 0, -1)]
```

**关键区分**：LLM 上下文窗口里的"对话历史"只是**工作副本**，不是原始记录。原始记录必须独立持久化存储——这既是为了审计需要，也是因为摘要压缩后原文就从窗口中丢失了。

### 长期记忆存在哪里？

| 存储方案 | 适用场景 | 优势 | 劣势 |
|---|---|---|---|
| **向量数据库**（Milvus/Pinecone/Chroma） | 语义检索为主 | 相似度搜索快，支持模糊匹配 | 不支持精确查询、更新不便 |
| **关系型数据库**（PostgreSQL + pgvector） | 结构化 + 语义混合 | SQL 精确查询 + 向量检索，事务安全 | 向量检索性能弱于专用向量库 |
| **文件系统**（Markdown / JSON） | 小规模、透明可控 | 人类可读可编辑，版本管理友好 | 不支持语义检索，规模有上限 |
| **知识图谱**（Neo4j / 自建） | 关系推理为主 | 支持多跳推理、关系查询 | 构建和维护成本高 |

**生产推荐架构**：PostgreSQL + pgvector 作为主存储（结构化数据 + 向量索引统一管理），Redis 做热缓存，S3 做冷归档。

```sql
CREATE TABLE memories (
    id UUID PRIMARY KEY,
    user_id VARCHAR NOT NULL,
    content TEXT NOT NULL,
    memory_type VARCHAR NOT NULL,  -- 'preference' | 'fact' | 'episode' | 'skill'
    embedding VECTOR(1536),
    importance FLOAT DEFAULT 0.5,
    access_count INT DEFAULT 0,
    last_accessed TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB
);
CREATE INDEX ON memories USING ivfflat (embedding vector_cosine_ops);
```

### 长期记忆和短期记忆的边界怎么划分？

边界不是"时间长短"，而是**信息的生命周期和用途**：

| 维度 | 短期记忆 | 长期记忆 |
|---|---|---|
| **生命周期** | 单次会话 | 跨会话持久 |
| **内容类型** | 当前任务上下文、对话历史、中间状态 | 用户偏好、历史事实、学到的知识 |
| **存储位置** | LLM 上下文窗口 + Redis | 数据库 + 向量库 |
| **读取方式** | 直接在 prompt 中 | 按需检索注入 prompt |
| **写入时机** | 每轮对话实时更新 | 会话结束时提炼 / 关键事件触发 |
| **淘汰机制** | 滑动窗口 / Token 阈值压缩 | 重要性衰减 / 访问频率淘汰 |

**边界的划分规则**：

```
一条信息应该存入长期记忆，当且仅当它满足以下任一条件：
1. 跨会话有效：下次对话还可能用到（用户偏好、重要事实）
2. 不可重新获取：用户不太可能再次提供（个人信息、历史决策）
3. 高复用价值：多个任务可能需要（技能、模式）

反之，以下信息只需要短期记忆：
1. 仅当前任务有效：槽位填充状态、工具调用结果
2. 可重新获取：搜索结果、计算中间值
3. 一次性使用：确认消息、临时澄清
```

**转化机制——短期 → 长期**：

```python
class MemoryPromoter:
    def on_session_end(self, working_memory, conversation):
        candidates = self.llm.extract(
            "从以下对话中提取需要跨会话记住的信息：\n"
            "1. 用户偏好（喜好、习惯、风格要求）\n"
            "2. 重要事实（个人信息、业务背景）\n"
            "3. 待办事项（未完成的任务）\n"
            "4. 学到的教训（什么方案有效/无效）\n"
            f"\n对话内容：{conversation}"
        )
        
        for item in candidates:
            existing = self.long_term.search(item.content, threshold=0.9)
            if existing:
                self.long_term.update(existing.id, item)  # 去重：更新而非新增
            else:
                self.long_term.add(item)
```

- 标签: `memory-design`, `conversation-store`, `hot-warm-cold`, `memory-boundary`, `memory-promotion`
- 记录于: 2026-06-20

## Q: 知识卡片抽取的 Prompt 设计：结构、原理与好坏示例方法论

### 知识卡片抽取的 Prompt 怎么写？

知识卡片抽取的目标是：从非结构化对话中提取结构化的记忆单元（Knowledge Card）。

```python
EXTRACTION_PROMPT = """
你是一个信息提取专家。从以下对话中提取需要长期记忆的知识卡片。

## 提取规则
1. 每张卡片只包含一个独立的信息点
2. 用陈述句表达，不要用疑问句
3. 包含足够的上下文使卡片脱离原对话仍可理解
4. 标注信息类型和置信度

## 输出格式
[
  {
    "content": "用户偏好使用 Python，主要做后端开发",
    "type": "preference",      // preference | fact | skill | episode
    "confidence": 0.9,         // 0.0-1.0
    "source": "用户在讨论技术栈时主动提及",
    "tags": ["python", "backend"]
  }
]

## 提取标准
- preference：用户明确表达的喜好和习惯（"我喜欢..."、"以后别..."、"我一般..."）
- fact：关于用户或其环境的事实（工作、项目、团队信息）
- skill：用户展示的技能或知识水平
- episode：重要事件的记录（做了什么决策、遇到什么问题）

## 不要提取
- 临时性的任务指令（"帮我查一下 X"）
- 通用常识（"Python 是解释型语言"）
- 已经在历史记忆中存在的重复信息

## 对话内容
{conversation}

## 已有记忆（避免重复）
{existing_memories}
"""
```

### 为什么要设计这样的 Prompt 结构？

每个组成部分都有明确的工程目的：

| Prompt 组件 | 作用 | 如果去掉会怎样 |
|---|---|---|
| **角色定义**（"你是信息提取专家"） | 激活模型的信息提取能力，抑制闲聊倾向 | 模型可能生成解释性文字而非结构化输出 |
| **提取规则** | 约束输出粒度（一张卡片一个信息点） | 模型会把多个信息混在一张卡片里，后续检索不精确 |
| **输出格式（JSON Schema）** | 强制结构化输出，便于程序解析 | 输出格式不稳定，解析失败率高 |
| **类型枚举（type enum）** | 限定分类空间，避免自由发挥 | 模型会创造各种类型名，下游无法统一处理 |
| **置信度字段** | 区分明确陈述 vs 模型推断 | 推测性信息和确定性信息混在一起，后续使用时无法判断可靠性 |
| **source 字段** | 记录信息来源，支持溯源审计 | 无法追溯某条记忆是从哪次对话提取的 |
| **提取标准（正面）** | 用触发词示例告诉模型什么该提取 | 模型对提取边界把握不准，漏提或过提 |
| **不要提取（负面）** | 明确排除项，减少噪音 | 会提取大量无价值信息（临时指令、常识），污染记忆库 |
| **已有记忆注入** | 去重依据 | 同一信息被反复提取存储，记忆库膨胀且检索时出现大量重复 |

**结构设计的核心原则**：
1. **正面约束 + 负面约束成对出现**——只有正面约束，模型倾向过度提取；只有负面约束，模型倾向欠提取
2. **Schema 即约束**——用 JSON Schema 代替自然语言描述格式，解析成功率从 ~85% 提升到 ~98%
3. **上下文注入**——把已有记忆传入，是让**模型在提取时就去重**，而不是提取后再做后处理去重。前者更精准，后者需要额外的语义相似度计算

### 如何设计好/坏示例（Few-shot Examples）？

示例设计是提取质量的决定性因素。原则：**示例不是教模型"怎么提取"，而是校准模型对"边界情况"的判断。**

**Step 1：覆盖典型 + 边界**

```json
// 好示例 1：典型的偏好提取（基线校准）
{
  "对话片段": "用户：以后回复尽量简短一些，我不喜欢长篇大论",
  "提取结果": {
    "content": "用户偏好简洁的回复风格，不喜欢冗长的解释",
    "type": "preference",
    "confidence": 0.95
  },
  "解释": "用户用'以后'和'尽量'明确表达了持久性偏好"
}

// 好示例 2：隐式偏好（边界情况——模型需要推断）
{
  "对话片段": "用户：嗯这个回答可以，之前那种带 emoji 的看着有点不正式",
  "提取结果": {
    "content": "用户不喜欢在回复中使用 emoji，认为不够正式",
    "type": "preference",
    "confidence": 0.8
  },
  "解释": "用户没有直接说'不要用emoji'，但通过对比表达了偏好，置信度适当降低"
}

// 好示例 3：不该提取的情况（负面示例）
{
  "对话片段": "用户：帮我查一下北京明天的天气",
  "提取结果": [],
  "解释": "这是一次性任务指令，不包含需要长期记忆的信息"
}
```

**Step 2：展示粒度边界**

```json
// 坏示例（粒度太粗——4 个独立信息混在一张卡片里）
{
  "content": "用户是程序员，用 Python 做后端，团队用微服务架构，最近在做 AI Agent 项目"
}

// 好示例（粒度适当，拆分为多张卡片）
[
  {"content": "用户的主要编程语言是 Python", "type": "fact"},
  {"content": "用户主要从事后端开发", "type": "fact"},
  {"content": "用户所在团队使用微服务架构", "type": "fact"},
  {"content": "用户当前在做 AI Agent 项目", "type": "episode"}
]
```

**Step 3：校准置信度**

```
高置信度 0.95：用户明确陈述
  来源："我在字节工作了三年"
  → {"content": "用户在字节跳动工作", "confidence": 0.95}

中置信度 0.7：合理推断
  来源：用户在讨论中正确使用了 Go 的 goroutine 概念
  → {"content": "用户熟悉 Go 语言", "confidence": 0.7}

不应提取：过度推断
  来源：用户只问了一个 Java 问题
  → 不能推断"用户是 Java 开发者"
```

**设计示例的系统方法**：

1. **先跑 zero-shot**——不给示例让模型提取一批，观察常见错误模式
2. **针对错误模式设计示例**——模型常犯什么错，就设计什么示例来纠正
3. **正负示例 1:1 配比**——好示例教"该提取什么"，坏示例教"不该提取什么"，同样重要
4. **示例数量 3-5 个为宜**——太少不能覆盖边界，太多消耗 token 且可能导致模型过度模仿示例格式
5. **示例附带解释**——告诉模型"为什么这个该提取 / 不该提取"，比单纯给 input-output 对效果更好（相当于 Chain-of-Thought 式的 few-shot）

- 标签: `knowledge-extraction`, `prompt-design`, `few-shot`, `example-design`, `memory-card`
- 记录于: 2026-06-20
