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

## Q: Agent 的容错机制有哪些？

Agent 系统的容错可以分为六个层次，从底层基础设施到高层策略依次递进：

### 一、重试与降级（最基础的一层）

**重试模式**：
- **指数退避（Exponential Backoff）**：API 调用失败后按 2s → 4s → 8s → 16s 递增等待重试，避免雪崩。几乎所有框架（LangChain、OpenAI SDK、Claude Code）都内置此机制。
- **Circuit Breaker（熔断器）**：连续 N 次失败后直接停止调用，经过冷却期再恢复。防止对已宕机的服务持续无效请求。
- **幂等性保证**：对有副作用的 tool call（如写文件、发消息），确保重试不会导致重复执行。通常通过请求 ID 去重或先检查状态再操作。

**降级模式**：
- **模型降级链**：主模型不可用时自动切换——如 Opus → Sonnet → Haiku。保证可用性的同时牺牲一定质量。
- **工具降级**：首选工具失败时回退到替代方案。例如专用搜索工具不可用时退化为 Web 搜索，或结构化 API 不可用时退化为 Bash 命令。
- **功能降级**：非核心功能出错时跳过而非阻塞整体流程。例如格式化失败不阻止代码提交。

### 二、自我纠错（Agent 特有的核心能力）

**执行层纠错**：
- **ReAct Error Loop**：Agent 执行 tool call 后观察返回结果，如果报错则分析错误原因、调整参数或换用其他工具重试。这是最基本的 agent 纠错——观察 → 推理 → 重试。
- **输出验证**：执行后主动验证结果是否符合预期。例如写完代码后运行测试、编辑文件后检查语法、执行命令后检查退出码。
- **多步骤回溯**：多步任务中某一步失败时，不只重试当前步骤，而是回退到更早的分支点尝试不同路径。

**推理层纠错**：
- **Reflexion 式自我反思**：任务失败后 agent 生成自然语言反思（"我错在哪里？应该怎么做？"），存入记忆供下次尝试参考。不改模型权重，通过 in-context learning 纠正行为。
- **LLM-as-Judge**：用 LLM 评估自身输出质量，不合格则重新生成。可以是同一模型自评，也可以是用更强模型（如 Opus）审核较弱模型（如 Haiku）的输出。
- **幻觉检测**：检查 agent 的回答是否有事实依据——对比检索到的原始资料与生成内容，标记无法溯源的陈述。

### 三、状态管理与回滚（确保可恢复）

**检查点（Checkpoint）**：
- **LangGraph**：每个图节点执行后自动保存状态快照到持久化存储（内存、SQLite、PostgreSQL）。支持"时间旅行"——回到任意历史节点重新执行。这使得长链任务中途失败时不需要从头开始。
- **Claude Code**：每次文件编辑前自动创建文件快照，按 Esc×2 可回退到上一状态。会话以 JSONL 格式保存在本地。
- **通用模式**：在每个有副作用的步骤前记录"做了什么"和"之前状态是什么"，失败时按逆序回滚。

**事务性操作**：
- 将多步操作打包为逻辑事务，要么全部成功，要么全部回滚。例如"创建分支 → 修改代码 → 运行测试 → 提交"——测试失败则丢弃所有改动。
- Git 天然提供了这种能力：agent 可以在临时分支操作，成功后合并，失败后删除分支。

### 四、上下文溢出处理

上下文窗口填满是 agent 独有的"故障"模式，处理策略：

- **自动压缩（Auto-compact）**：接近上下文上限时自动摘要老对话，保留关键信息。Claude Code 在 ~95% 容量时触发，优先清除旧 tool 输出。
- **子 agent 委托**：把大文件读取、大量搜索等高 token 消耗的工作委托给子 agent，子 agent 在独立上下文窗口中执行，只返回精简的结论。
- **渐进式加载**：不一次性加载所有资料，而是按需读取（如本仓库 skill 的设计）。相比预加载，大幅减少 token 浪费。
- **分治策略**：把大任务拆成多个子任务，每个子任务在独立会话/agent 中执行，最后汇总结果。

### 五、多 Agent 容错

**冗余模式**：
- **投票/共识**：同一问题交给多个 agent 独立回答，取多数一致的答案。适合高风险决策——降低单个 agent 幻觉的影响。
- **冗余执行**：关键步骤由多个 agent 并行执行，比较结果。如果结果一致则采纳，不一致则交给 supervisor 或人类仲裁。

**监督者模式（Supervisor Pattern）**：
- 一个 orchestrator agent 负责任务分配和状态监控。工作 agent 执行任务，orchestrator 检测超时、异常、低质量输出。
- 检测到故障时，orchestrator 可以：重试同一 agent、重新分配给另一个 agent、降级任务要求、升级给人类处理。
- LangGraph、CrewAI、AutoGen 都支持这种层级化的 agent 编排。

**Human-in-the-Loop**：
- 当 agent 置信度低或操作风险高时，主动暂停并请求人类确认。这是容错的最终兜底——不确定就问人。
- Claude Code 的权限系统本质上就是 human-in-the-loop：高风险操作（git push、删除文件）默认需要人类批准。

### 六、防护栏与安全网（预防性容错）

**输入防护**：
- Prompt injection 检测：识别并过滤用户输入或外部数据中的指令注入攻击
- 输入格式校验：在交给 LLM 前检查输入是否合规

**输出防护**：
- 格式校验：确保 agent 输出符合预期 schema（JSON schema 验证、类型检查）
- 安全过滤：检测输出中是否包含敏感信息（密钥、密码、PII）
- 语义检查：用独立模型检查输出是否合理，是否包含有害内容

**执行沙箱**：
- 文件系统隔离：限制 agent 只能访问工作目录（Claude Code 用 Seatbelt/bubblewrap）
- 网络隔离：限制 agent 可以访问的域名和端口
- 资源限制：超时控制（默认 2 分钟）、输出大小限制（30K 字符）、防止无限循环

### 各框架容错能力对比

| 机制 | Claude Code | LangGraph | CrewAI | AutoGen/AG2 | OpenAI Agents SDK |
|---|---|---|---|---|---|
| 重试/退避 | 内置（API + git push） | 内置 | 内置 | 内置 | 内置 |
| 模型降级 | 手动切换 | 可配置 fallback | 支持 | 支持 | 支持 |
| 自我纠错 | ReAct loop + 测试验证 | 节点重试 + 条件分支 | 任务重试 + agent 委托 | 对话式错误恢复 | tool error handling |
| 状态检查点 | 文件快照 + JSONL 会话 | 图节点级持久化 | 任务级 | 对话历史 | 会话级 |
| 回滚 | Esc×2 / git reset | 时间旅行 | 手动 | 手动 | 手动 |
| 上下文管理 | 5 层压缩管线 | 消息修剪 | 摘要 | 对话压缩 | 截断 |
| 子 agent 隔离 | 独立上下文窗口 | 子图 | 支持 | Group Chat | Handoff |
| Human-in-loop | 权限系统 + AskUser | interrupt_before | 回调 | 人类代理 | Guardrails |
| 沙箱 | Seatbelt/bubblewrap | 无内置 | 无内置 | Docker（可选） | 无内置 |

### 前沿数据与研究（2025-2026）

**故障率现状**：
- 多 Agent 系统在无专门容错设计时，生产环境故障率 **41%-86.7%**（MAST 分类法，UC Berkeley 2025，1642 条执行轨迹标注）
- 故障分类：系统设计问题 43.8%（含步骤重复 15.7%、不识别终止条件 12.4%）、Agent 间失配 32.1%、验证缺失 23.5%
- 仅通过架构调整（不换模型）就能提升 +9.4% ~ +15.6% 成功率

**自我纠错的局限**：
- **纯内省式自纠错不可靠**——生成器和评估器共享相关错误模式（Princeton 2026）。LLM 在没有外部信号时无法可靠地纠正自身推理错误
- **有效的纠错需要接地信号**：代码执行结果、工具验证、数据库查询等外部真值
- 标准 ReAct agent **90.8% 的重试是浪费的**（513 次重试中 466 次用于不可重试的错误），需要错误分类（可重试/不可重试/需人工）
- Reflexion 在 HumanEval 上从 80% → 91%，但在简单任务上无提升甚至退化

**可靠性悖论**（Princeton "Towards a Science of AI Agent Reliability", 2026）：
- 经过 18 个月模型迭代，agent 准确率大幅提升，但**可靠性（一致性）几乎没有改善**
- 推理模型比非推理模型更可靠，但可靠性提升远慢于准确率
- 小模型在一致性上经常等于甚至优于大模型
- 模型对表面层的 prompt 改写仍然脆弱，即使能处理技术/工具故障

**生产效果**：
- 应用四种容错模式（重试 + 降级 + 错误分类 + 检查点）后，不可恢复故障率从 23% 降至 <2%，单次故障成本降低 85%
- LangGraph 在 10 步管线上延迟 4.2s vs CrewAI 7.8s
- Spotify 的 judge 组件标记了 25% 的代码变更，agent 成功自纠错其中 50%

**Agent 混沌工程（2026 新兴方向）**：
- LLM API 在生产中有 1-5% 的失败率；agent 执行 10-20 次 tool call 时，故障概率显著累积
- 传统混沌工程模式（熔断器、幂等重试）对 AI agent 失效的三个原因：LLM 重试产生不同推理路径（非幂等）、agent 会即兴发挥而非优雅降级、静默失败（格式正确但语义错误）
- 需要注入测试的六类故障：LLM 层故障、工具调用故障、上下文退化、多 agent 级联故障、规格漂移、静默故障

### 设计原则

1. **快速失败，优雅恢复**：尽早检测错误（输入校验、超时），失败后有明确的恢复路径。
2. **分层防御**：不依赖单一容错机制，而是多层叠加——重试 → 自纠错 → 回滚 → 人工介入。
3. **最小爆炸半径**：通过沙箱、权限、子 agent 隔离，确保单点故障不扩散。
4. **可观测性**：记录每步操作和结果，故障时可追溯和复现。日志是容错的基础设施。
5. **人类兜底**：agent 置信度低或操作不可逆时，升级给人类。自动化要有逃生通道。

- 标签: `agent`, `fault-tolerance`, `error-handling`, `reliability`, `self-correction`
- 记录于: 2026-06-18
