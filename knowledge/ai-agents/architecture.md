# 架构与设计

> 涵盖：Agent 架构选型、多 Agent 分工与协作模式、链路路由、意图识别。

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

## Q: Agent 项目用什么架构？LangGraph 还是自研？Master+Sub Agent 还是 Workflow？

### 常见架构选型

**框架选择：LangGraph vs 自研**

| 维度 | LangGraph | 自研框架 |
|---|---|---|
| 优势 | 图抽象成熟、内置持久化/检查点/时间旅行、社区生态、与 LangChain 无缝集成 | 完全可控、无抽象泄漏、可针对业务深度定制、无框架升级兼容问题 |
| 劣势 | 框架概念学习成本高、抽象黑箱增加调试难度、性能开销（序列化/反序列化） | 需自建持久化/重试/监控等基础设施、开发周期长 |
| 适合场景 | 复杂多 Agent 编排、需要检查点和 human-in-the-loop、快速原型 | 业务逻辑高度定制、对延迟敏感、团队有足够工程能力 |

**实践建议**：如果团队规模 ≤5 人且 Agent 逻辑相对确定，自研基于 Python async + 状态机的轻量框架通常更高效。LangGraph 的价值在复杂度高、需要频繁迭代编排逻辑时才显现。

### 编排模式：Master+Sub vs Workflow

**Master+Sub Agent（Supervisor 模式）**：
- 一个 orchestrator agent 接收用户请求，动态决定调度哪些子 agent、以何种顺序执行
- 路由决策由 LLM 做，灵活但不确定性高
- 适合：用户意图多变、任务路径无法预定义的场景（如通用对话 agent、开放域问答）
- 风险：supervisor 自身可能做出错误路由决策，需要做 fallback

**Workflow 模式（DAG/Pipeline）**：
- 预定义的有向无环图，每个节点是固定步骤，边是确定性条件分支
- 路由由代码逻辑（if/else、状态判断）驱动，可预测、可测试
- 适合：流程明确、步骤固定的业务场景（如表单填写、标准化报告生成）
- 风险：灵活性差，新增路径需要改代码

**实际选型：混合模式最常见**：
- 整体用 **Workflow** 定义主链路（意图识别 → 检索 → 生成 → 后处理），保证基线稳定
- 在关键决策点用 **LLM 路由**（如意图分类、是否需要追问、选择哪个检索策略）
- Sub Agent 用于处理特定领域子任务（如代码生成、数据分析），各自有独立上下文和 prompt
- Orchestrator 不做具体任务，只负责状态管理、错误处理和流程推进

```
用户输入 → [意图识别(LLM)] → 路由决策
  ├─ 简单问答 → RAG Pipeline (Workflow)
  ├─ 多轮对话 → Dialogue Agent (Sub Agent)
  ├─ 复杂任务 → Task Planner → [Sub Agent 1, Sub Agent 2, ...] → 结果合并
  └─ 兜底 → 通用生成
```

**选型核心原则**：不要为了用框架而用框架。先跑通单 Agent + 简单 pipeline，验证业务可行后再按需拆分 Agent 和引入编排框架。过早引入多 Agent 架构是最常见的过度工程。

- 标签: `architecture`, `langgraph`, `workflow`, `supervisor`, `design-decision`
- 记录于: 2026-06-19

## Q: 项目中是单 Agent 还是多 Agent？各子 Agent 的核心任务和分工是什么？

### 何时需要多 Agent

**单 Agent 适用**：任务链路短（≤3 步）、不需要领域隔离、上下文窗口足够。大部分场景下，单 Agent + 良好的 prompt + 工具集 已经足够。

**多 Agent 必要的信号**：
1. **上下文冲突**：不同子任务需要截然不同的 system prompt（如代码生成 vs 自然语言对话），放在一个 prompt 里互相干扰
2. **能力隔离**：某些子任务需要不同的模型（如用大模型做推理、小模型做分类）
3. **并行需求**：多个独立子任务可以同时执行以降低延迟
4. **错误隔离**：一个子任务失败不应阻塞其他子任务

### 典型多 Agent 分工

一个完整的对话式 Agent 系统通常包含以下角色：

| Agent | 核心任务 | 模型选择 | 关键特点 |
|---|---|---|---|
| **意图识别 Agent** | 分类用户意图、提取关键实体、判断是否需要澄清 | 轻量模型（速度优先） | 延迟敏感，需要 <200ms |
| **对话管理 Agent** | 维护对话状态、管理多轮上下文、决定下一步动作 | 中等模型 | 状态机 + LLM 混合驱动 |
| **检索 Agent** | 查询改写、多路检索（向量/关键词/图谱）、结果排序 | 可用小模型做 reranking | 与外部数据源交互 |
| **生成 Agent** | 基于检索结果和上下文生成最终回答 | 大模型（质量优先） | 消耗 token 最多的环节 |
| **质量检查 Agent** | 检查幻觉、格式合规、安全过滤 | 中等模型 | 可选，但对 toB 场景重要 |
| **工具执行 Agent** | 执行代码、调用 API、操作数据库 | 代码模型 | 需要沙箱隔离 |

### 分工设计原则

1. **单一职责**：每个 Agent 只做一件事，prompt 短且聚焦。一个 Agent 的 system prompt 超过 2000 tokens 就该考虑拆分。
2. **接口清晰**：Agent 间通过结构化数据（JSON schema）通信，不传递自然语言。减少歧义，方便调试。
3. **可独立评测**：每个 Agent 有自己的评测集，能独立迭代而不影响其他 Agent。
4. **成本分层**：高频低价值任务用小模型，低频高价值任务用大模型。意图识别每次请求都跑，用 Haiku 级别；最终生成可能只在特定路径触发，用 Opus 级别。

- 标签: `multi-agent`, `architecture`, `agent-division`, `design-pattern`
- 记录于: 2026-06-19

## Q: 首次生成和多轮补充的链路路由是怎么区分和实现的？

### 核心区别

**首次生成（Cold Start）**：用户发起新话题，系统无历史上下文。需要完整的意图识别 → 槽位提取 → 路由决策 → 检索 → 生成全链路。

**多轮补充（Follow-up）**：用户在已有对话基础上追问、补充或修正。需要判断是"延续当前话题"还是"切换新话题"，并正确继承上下文。

### 路由判断机制

**Step 1：对话状态判断**

```python
# 核心信号
signals = {
    "has_history": len(conversation_history) > 0,
    "coreference": detect_coreference(query),      # "它"、"这个"、"上面说的"
    "topic_shift": compute_topic_similarity(query, last_topic),
    "explicit_new": detect_new_topic_markers(query), # "换个话题"、"另外问一下"
}
```

**判定规则**：
- 无历史 → 首次生成
- 有指代词且话题相似度 > 阈值 → 多轮补充
- 有显式切换信号 → 当作新的首次生成
- 话题相似度 < 阈值且无指代 → 新话题，走首次链路

**Step 2：上下文继承**

首次生成：
```
用户 Query → 意图识别 → 槽位提取 → 路由 → 检索(全量) → 生成
```

多轮补充：
```
用户 Query + 历史上下文 → 指代消解/Query 补全 → 增量槽位更新 → 路由(可复用) → 检索(增量) → 生成
```

关键差异：
- **指代消解**：多轮时需要把"它怎么用？"解析为"LangGraph 的 checkpointer 怎么用？"
- **槽位继承**：首次提取了"查询对象=LangGraph"，补充问"性能怎么样"时自动继承
- **增量检索**：不重新全量搜索，而是基于追问内容做增量补充

**Step 3：路由复用与更新**

多轮场景下，路由决策有三种情况：
1. **路径不变**：追问同一主题的细节 → 复用原路由，只追加检索
2. **子路径切换**：从"介绍 X"变为"X 的代码示例" → 切换到代码生成子路径
3. **完全切换**：话题跳转 → 重置状态，走首次生成链路

### 实现关键

- **对话状态管理**：用状态机 + 栈结构管理多轮上下文。每个话题是一个状态节点，追问压栈，切换出栈。
- **LLM 辅助判断**：话题切换/延续的判断用 LLM 分类器（accuracy 通常 >95%），比规则判断更鲁棒。
- **超时重置**：用户长时间未发言（如 >30min），自动视为新话题。
- **显式控制**：提供"新对话"按钮或指令，让用户主动重置上下文。

- 标签: `routing`, `multi-turn`, `dialogue-management`, `context-inheritance`
- 记录于: 2026-06-19

## Q: 并行化意图识别是什么？为什么要做并行化？如何实现？

### 什么是并行化意图识别

传统意图识别是串行的：先判断一级意图 → 再判断二级意图 → 再提取实体。每一步依赖上一步的结果，延迟叠加。

**并行化意图识别**是将多个识别任务同时执行：
- 多个维度的分类器并行运行（意图类型、情感、紧急度、领域）
- 意图识别与实体提取并行
- 多个候选意图同时验证

### 为什么要做并行化

1. **延迟是核心痛点**：意图识别处于链路最前端，每增加 100ms 都会被用户直接感知。串行的三级分类（一级→二级→实体）可能需要 3 次 LLM 调用，延迟 1-2s。
2. **多维度信息互不依赖**：意图类型、情感倾向、紧急程度是正交维度，没有先后依赖关系。
3. **提升召回率**：多个分类器独立判断，可以通过融合策略发现单一分类器遗漏的意图。
4. **支持复合意图**：用户一句话可能包含多个意图（"帮我查天气，顺便设个闹钟"），并行识别每个意图比串行解析更自然。

### 实现方式

**方案一：多分类器并行 Fan-out**

```python
import asyncio

async def parallel_intent(query: str):
    # 多个识别任务并行执行
    results = await asyncio.gather(
        classify_intent_type(query),      # 意图类型（问答/任务/闲聊）
        extract_entities(query),           # 实体提取（人名/地点/时间）
        detect_sentiment(query),           # 情感分析
        classify_domain(query),            # 领域分类（技术/生活/工作）
        detect_urgency(query),             # 紧急程度
    )
    return merge_results(results)
```

**方案二：多模型/多 Prompt 投票**

```python
async def voting_intent(query: str):
    # 同一任务用不同策略并行，取共识
    results = await asyncio.gather(
        llm_classifier(query, prompt_v1),   # LLM 分类（Prompt A）
        llm_classifier(query, prompt_v2),   # LLM 分类（Prompt B）
        rule_based_classifier(query),       # 规则分类（快但覆盖窄）
        embedding_classifier(query),        # Embedding 相似度分类
    )
    return majority_vote(results)
```

**方案三：层级并行**

```
            ┌─ 一级意图分类 ─┐
用户Query ─→├─ 实体提取     ├─→ 路由决策
            └─ 情感分析     ─┘
                 ↓（一级结果出来后）
            ┌─ 二级意图细分 ─┐
            └─ 槽位填充     ─┘─→ 执行
```

第一层完全并行；第二层等第一层的意图结果后再并行细分。相比全串行，延迟从 `T1+T2+T3+T4+T5` 降低到 `max(T1,T2,T3) + max(T4,T5)`。

### 关键设计考量

- **超时控制**：每个并行任务设独立超时（如 500ms），超时的分类器结果直接丢弃，用其他分类器的结果兜底。
- **结果融合**：用加权投票（基于各分类器历史准确率）或优先级规则合并多个分类结果。
- **成本控制**：并行调用 N 个 LLM 会使 token 消耗翻 N 倍。可以用轻量模型做大部分分类、大模型只在低置信度时介入。
- **取消传播**：如果某个快速分类器已经高置信度（>0.95）确定了意图，可以取消其他还在运行的分类器以节省成本。

- 标签: `intent-recognition`, `parallel`, `latency-optimization`, `fan-out`
- 记录于: 2026-06-19

## Q: ReAct 框架的工程实现细节？消息格式如何设计？tool_response 用什么角色传回？为什么？

### ReAct 核心循环

ReAct（Reasoning + Acting）让 LLM 在**思考（Thought）→ 行动（Action）→ 观察（Observation）**的循环中解决问题：

```
用户提问
  ↓
┌──────────────────────────────────┐
│ Thought: 我需要查询天气 API       │ ← LLM 生成推理过程
│ Action: get_weather(city="北京")  │ ← LLM 输出工具调用
└──────────────────────────────────┘
  ↓ 系统执行工具
┌──────────────────────────────────┐
│ Observation: {"temp": 28, ...}   │ ← 工具返回结果
└──────────────────────────────────┘
  ↓ 结果回传给 LLM
┌──────────────────────────────────┐
│ Thought: 已拿到结果，可以回答了   │
│ Answer: 北京今天 28°C...          │ ← LLM 生成最终回答
└──────────────────────────────────┘
```

### 消息格式设计

现代 API（OpenAI / Anthropic）的消息格式直接支持 ReAct 模式。关键是**四种角色**的配合：

```python
messages = [
    # 1. system：定义 Agent 身份和可用工具
    {"role": "system", "content": "你是一个天气助手..."},
    
    # 2. user：用户输入
    {"role": "user", "content": "北京明天天气怎么样？"},
    
    # 3. assistant：LLM 的思考 + 工具调用请求
    {"role": "assistant", "content": None, "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "get_weather",
                "arguments": '{"city": "北京", "date": "tomorrow"}'
            }
        }
    ]},
    
    # 4. tool：工具执行结果回传
    {"role": "tool", "tool_call_id": "call_abc123",
     "content": '{"temp": 28, "condition": "晴", "humidity": 45}'},
    
    # 5. assistant：LLM 基于观察生成最终回答
    {"role": "assistant", "content": "北京明天晴，气温 28°C，湿度 45%。"}
]
```

### tool_response 应该用 `tool` 角色传回——为什么？

**用 `tool` 角色（而非 `user` 或 `system`）的三个核心原因**：

**1. 语义区分——LLM 需要知道这不是人说的话**

```python
# ❌ 错误：用 user 角色传工具结果
{"role": "user", "content": '{"temp": 28}'}
# LLM 会困惑：这是用户在说 JSON？还是用户在提问？

# ✅ 正确：用 tool 角色
{"role": "tool", "tool_call_id": "call_abc123", "content": '{"temp": 28}'}
# LLM 明确知道：这是我调用的工具返回的结果
```

用 `user` 角色会导致 LLM 将工具结果误解为用户的新输入，触发新的对话轮次而非继续推理。用 `system` 角色会与系统级指令混淆，且部分模型对 system 消息有特殊处理（如置顶注意力）。

**2. 调用链追踪——`tool_call_id` 实现一一对应**

```python
# LLM 同时调用多个工具（并行 tool call）
assistant_msg = {"role": "assistant", "tool_calls": [
    {"id": "call_001", "function": {"name": "get_weather", "arguments": "..."}},
    {"id": "call_002", "function": {"name": "get_news", "arguments": "..."}},
]}

# 每个 tool 结果通过 tool_call_id 精确匹配
tool_msg_1 = {"role": "tool", "tool_call_id": "call_001", "content": "天气数据..."}
tool_msg_2 = {"role": "tool", "tool_call_id": "call_002", "content": "新闻数据..."}
```

`tool` 角色携带 `tool_call_id`，让 LLM 知道哪个结果对应哪个调用。如果用 `user` 角色，多个工具结果会混在一起无法区分。

**3. 训练信号——模型被训练为期望这种格式**

现代 LLM（GPT-4、Claude）在 RLHF/SFT 阶段使用 `tool` 角色的对话数据训练。使用正确角色能触发模型训练中学到的"工具使用"行为模式，产生更准确的后续推理。

### 完整工程实现

```python
import json

class ReActAgent:
    def __init__(self, llm_client, tools: dict):
        self.llm = llm_client
        self.tools = tools  # {"tool_name": callable}
        self.max_iterations = 5
    
    def run(self, user_query: str) -> str:
        messages = [
            {"role": "system", "content": self._build_system_prompt()},
            {"role": "user", "content": user_query},
        ]
        
        for i in range(self.max_iterations):
            response = self.llm.chat(
                messages=messages,
                tools=self._tool_definitions(),
            )
            
            assistant_msg = response.message
            messages.append(assistant_msg)
            
            # 没有工具调用 → LLM 认为可以直接回答了
            if not assistant_msg.get("tool_calls"):
                return assistant_msg["content"]
            
            # 执行每个工具调用，结果以 tool 角色回传
            for tool_call in assistant_msg["tool_calls"]:
                func_name = tool_call["function"]["name"]
                func_args = json.loads(tool_call["function"]["arguments"])
                
                try:
                    result = self.tools[func_name](**func_args)
                    tool_content = json.dumps(result, ensure_ascii=False)
                except Exception as e:
                    tool_content = json.dumps({"error": str(e)})
                
                # 关键：用 tool 角色 + tool_call_id 回传
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": tool_content,
                })
        
        return "达到最大迭代次数，无法完成任务。"
```

### Anthropic (Claude) 的差异

Claude 的消息格式略有不同——工具结果放在 `user` 消息的 `tool_result` content block 中：

```python
# Claude 的格式
messages = [
    {"role": "user", "content": "北京天气？"},
    {"role": "assistant", "content": [
        {"type": "tool_use", "id": "toolu_001",
         "name": "get_weather", "input": {"city": "北京"}}
    ]},
    # 工具结果嵌入在 user 角色中，但通过 type 和 tool_use_id 区分
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_001",
         "content": '{"temp": 28}'}
    ]},
]
```

虽然外层是 `user` 角色，但 `type: "tool_result"` + `tool_use_id` 在语义上等价于 OpenAI 的 `tool` 角色——模型内部能正确区分这是工具返回而非用户输入。

### 消息链路中的常见坑

| 问题 | 表现 | 解决 |
|---|---|---|
| tool_call_id 不匹配 | API 报错或 LLM 忽略结果 | 严格使用 LLM 返回的 id |
| 工具结果太长 | 上下文溢出 | 截断/摘要后再传回（如只传前 2000 字符） |
| 工具报错未处理 | LLM 反复重试同一调用 | 将错误信息作为 tool content 传回，让 LLM 决策 |
| 缺少停止条件 | 无限循环 | 设 max_iterations + 超时 |
| 并行调用结果顺序 | 结果乱序 | tool_call_id 保证匹配，顺序无关 |

- 标签: `react`, `tool-calling`, `message-format`, `agent-loop`, `function-calling`
- 记录于: 2026-06-20

## Q: 工具库有上百个工具时，如何让模型快速准确地选择工具？

### 问题本质

LLM 的 function calling 在工具数量少（<20）时表现良好，但工具数增多后面临两个核心问题：

1. **Token 成本爆炸**：100 个工具的定义 ≈ 20K-50K tokens，每次请求都要传入
2. **选择准确率下降**：搜索空间太大，LLM 在 100 个工具中选对的概率远低于 10 个

### 解决方案一：分层路由（Hierarchical Routing）

```
用户输入 → [领域分类器] → 领域标签 → [只加载该领域的工具] → LLM 选择
           (轻量模型)     "支付"      (5-10 个工具)
```

```python
DOMAIN_TOOLS = {
    "payment": ["create_payment", "query_payment", "refund", "cancel_payment"],
    "order": ["create_order", "query_order", "modify_order", "cancel_order"],
    "user": ["get_profile", "update_profile", "reset_password"],
    "logistics": ["track_package", "estimate_delivery", "change_address"],
}

def route_and_select(query: str):
    # 第一步：轻量分类器选领域（<5ms）
    domain = classify_domain(query)  # BERT 或规则
    
    # 第二步：只传入该领域的工具定义给 LLM
    relevant_tools = DOMAIN_TOOLS[domain]  # 5-10 个，而非 100 个
    
    result = llm.chat(
        messages=[{"role": "user", "content": query}],
        tools=get_tool_definitions(relevant_tools),  # 小搜索空间
    )
    return result
```

**效果**：搜索空间从 100 → 5-10，token 消耗降低 80%+，选择准确率显著提升。

### 解决方案二：Embedding 检索 + 重排序

```python
class ToolRetriever:
    def __init__(self, tools: list[Tool]):
        # 预计算每个工具的 embedding
        self.tool_embeddings = {
            tool.name: embed(tool.description + tool.usage_examples)
            for tool in tools
        }
        self.tools = {t.name: t for t in tools}
    
    def retrieve(self, query: str, top_k=8) -> list[Tool]:
        query_embedding = embed(query)
        
        # 语义检索 Top-K 候选工具
        scores = {
            name: cosine_similarity(query_embedding, tool_emb)
            for name, tool_emb in self.tool_embeddings.items()
        }
        
        top_candidates = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
        return [self.tools[name] for name, _ in top_candidates]

# 使用
retriever = ToolRetriever(all_100_tools)
relevant_tools = retriever.retrieve(user_query, top_k=8)
# 只把这 8 个工具传给 LLM
result = llm.chat(messages=messages, tools=relevant_tools)
```

**进阶：加重排序**

```python
def retrieve_and_rerank(query, top_k=8):
    # 粗筛：embedding 检索 top-20
    candidates = retriever.retrieve(query, top_k=20)
    
    # 精排：用 LLM 或 cross-encoder 重排序
    reranked = reranker.rerank(
        query=query,
        documents=[t.description for t in candidates],
        top_k=top_k,
    )
    
    return [candidates[i] for i in reranked.indices]
```

### 解决方案三：工具描述优化

工具选择的准确率很大程度取决于**工具描述的区分度**：

```python
# ❌ 差的描述：模糊、重叠
{"name": "search", "description": "搜索信息"}
{"name": "query", "description": "查询数据"}

# ✅ 好的描述：明确职责、区分边界、含示例
{
    "name": "search_product_catalog",
    "description": (
        "在商品目录中搜索商品。"
        "输入商品名称或类目关键词，返回匹配的商品列表。"
        "仅用于搜索商品信息，不涉及订单或用户操作。"
        "示例输入: '红色连衣裙', 'iPhone 15 手机壳'"
    ),
}
```

**描述优化清单**：
1. 说清楚工具**做什么**和**不做什么**
2. 说清楚**什么时候用**（触发条件）
3. 给出 2-3 个**输入示例**
4. 与容易混淆的工具**显式区分**（"用 X 查询商品，用 Y 查询订单"）

### 解决方案四：工具分组 + 延迟加载

```python
class LazyToolLoader:
    def __init__(self):
        # 只加载工具的元信息（名称 + 简短描述）
        self.tool_index = {
            "payment_tools": "处理支付相关操作（创建支付、退款、查询支付状态）",
            "order_tools": "处理订单相关操作（创建、修改、取消、查询订单）",
            "user_tools": "处理用户相关操作（个人信息、密码、偏好设置）",
        }
        # 完整定义按需加载
        self._loaded_groups = {}
    
    def get_tool_index_prompt(self) -> str:
        """第一次调用只传工具组的索引（几百 token）"""
        return "\n".join(
            f"- {name}: {desc}" for name, desc in self.tool_index.items()
        )
    
    def load_group(self, group_name: str) -> list:
        """LLM 选定工具组后，再加载完整定义"""
        if group_name not in self._loaded_groups:
            self._loaded_groups[group_name] = load_tool_definitions(group_name)
        return self._loaded_groups[group_name]
```

### 综合架构

```
用户输入
  ↓
[关键词匹配] → 命中高频工具 → 直接使用（<1ms）
  ↓ 未命中
[Embedding 检索] → Top-8 候选工具
  ↓
[LLM Function Calling] → 从 8 个中选 1-2 个
  ↓
执行工具
```

| 阶段 | 方法 | 延迟 | 覆盖率 |
|---|---|---|---|
| 关键词匹配 | 规则/关键词 | <1ms | ~40% 明确请求 |
| Embedding 检索 | 向量相似度 | ~5ms | ~95% |
| LLM 选择 | Function Calling | ~200ms | ~99% |

**核心原则**：不要把 100 个工具全部塞给 LLM——先缩小范围（通过路由/检索/分组），再让 LLM 在小范围内精确选择。

- 标签: `tool-selection`, `tool-routing`, `embedding-retrieval`, `function-calling`, `scalability`
- 记录于: 2026-06-20
