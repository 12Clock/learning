# AI Agents

> 涵盖：AI Agent 架构设计、Memory 系统、奖励机制、Skill 设计等。

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

## Q: Claude Code 这个 Agent 项目是怎么设计的？包含什么模块？

Claude Code 作为一个 CLI agent 系统，包含以下核心模块：

**1. 推理核心**
- 底层 LLM（Claude 系列模型）作为推理引擎，支持多模型切换（Opus/Sonnet/Haiku）。

**2. 工具系统（Tool System）**
- **文件操作**：Read、Write、Edit、Glob、Grep —— 专用工具优先于通用 Bash。
- **Bash**：执行任意 shell 命令，作为兜底工具。
- **Agent**：子 agent 派生，支持并行工作（Explore、Plan、general-purpose 等专用类型）。
- **Notebook**：Jupyter notebook 编辑能力。

**3. 权限系统（Permission System）**
- 三层控制：自动允许的操作 / 需用户审批的操作 / 拒绝的操作。
- Allowlist / denylist 配置（`settings.json`）。
- 高风险操作（git push、删除文件等）默认需要确认。

**4. 记忆系统（Memory System）**
- **CLAUDE.md**：人工编写的项目/用户级指令，层级加载（managed → user → project → local）。
- **Auto Memory**：agent 自动学习并写入的经验（`~/.claude/projects/<project>/memory/`），启动时加载前 200 行。
- **Session Memory**：会话内的对话摘要，会话间重置。

**5. 上下文管理（Context Management）**
- 接近上下文窗口限制时自动压缩历史消息。
- `/compact` 命令手动触发压缩。
- 支持 `@path/to/file` 语法引入外部文件（最多 4 层跳转）。

**6. MCP（Model Context Protocol）**
- 可扩展的工具服务器协议，接入 GitHub、数据库等外部服务。
- 工具以 `mcp__<server>__<tool>` 命名，运行时动态加载。

**7. Hooks 系统**
- Shell 命令在特定事件时执行（tool call 前后、通知等）。
- 用户可配置自动化行为（linting、格式化等）。

**8. Skill 系统**
- Markdown 文件定义的专用工作流，通过 `/skill-name` 或自动匹配触发。

**9. Rules 系统**
- `.claude/rules/` 目录下的规则文件，支持 `paths:` frontmatter 按文件路径条件加载。

**整体架构模式**：单线程主循环（Gather Context → Take Action → Verify Results），LLM 作为中央调度器，通过 tool calling 与外部世界交互。上下文管理通过五层管线控制：Budget reduction → Snip → Microcompact → Context collapse → Auto-compact。权限系统采用 deny-first 7 层防御模型（工具预过滤 → 规则评估 → 模式约束 → ML 安全分类 → Shell 沙箱 → 会话隔离 → Hook 拦截），将权限提示减少了 84%。子 agent 拥有独立的上下文窗口，不能再嵌套派生子 agent。

- 标签: `claude-code`, `architecture`, `agent-design`
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
