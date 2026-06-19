# 意图识别

> 涵盖：意图识别基础概念、基于 LLM 的意图分类、Prompt 设计、多轮/多意图处理、常见问题与对策。

## Q: 在 Agent 架构中，意图识别的核心目标是什么？

意图识别（Intent Recognition）的核心目标是**将用户的自然语言输入映射到系统预定义的动作类别**，从而驱动后续链路的路由决策。

它在 Agent 系统中扮演"调度器"角色：

```
用户输入 → [意图识别] → 意图类别 → 路由到对应处理链路
                                    ├─ 问答 → RAG Pipeline
                                    ├─ 任务执行 → Tool Agent
                                    ├─ 闲聊 → Chitchat Agent
                                    └─ 不明确 → 追问/澄清
```

**具体目标拆解**：

1. **分类（Classification）**：判断用户"要做什么"——查询信息、执行操作、闲聊、投诉、下单等。
2. **路由（Routing）**：基于意图将请求分发到正确的处理模块，避免"万能 Agent"什么都做但什么都做不好。
3. **过滤（Filtering）**：识别超出系统能力范围的请求，及早拒绝或引导，而非让后续模块白跑。
4. **优先级判断**：部分场景需要判断紧急程度（如客服系统区分一般咨询 vs 紧急投诉）。

**为什么它是核心**：意图识别处于链路最前端，错了后面全错。一个 95% 准确率的意图识别器对系统整体效果的影响，远大于一个 99% 准确率的生成器——因为意图错了根本不会走到生成。

- 标签: `intent-recognition`, `routing`, `agent-architecture`
- 记录于: 2026-06-19

## Q: 意图识别与实体识别有什么区别和联系？

### 区别

| 维度 | 意图识别（Intent Recognition） | 实体识别（Entity Extraction / NER） |
|---|---|---|
| 回答的问题 | 用户**要做什么**？ | 用户说的**关键参数**是什么？ |
| 输出粒度 | 整句级别的分类标签 | Token 级别的标注 |
| 输出示例 | `intent: book_flight` | `departure: 北京`, `destination: 上海`, `date: 6月20日` |
| 类比 | 函数名 | 函数参数 |

### 联系

意图和实体是**互补**的——意图决定"调哪个函数"，实体决定"传什么参数"：

```python
# 用户说："帮我订明天从北京到上海的机票"
intent = "book_flight"                    # 意图识别的输出
entities = {                              # 实体识别的输出
    "departure": "北京",
    "destination": "上海",
    "date": "明天"
}

# 组合后驱动执行
if intent == "book_flight":
    book_flight(**entities)
```

**实体可以辅助意图消歧**：当意图分类器对"查一下苹果"犹豫不决时（水果？公司？），如果实体识别提取到"股价"，就能确认意图是"查询股票信息"。

**在 LLM 时代的变化**：传统 NLU 中意图识别和实体识别是两个独立模型（分类器 + 序列标注）。现在用 LLM 可以在一次调用中同时完成两者——Prompt 中要求模型输出结构化 JSON，包含 intent 和 entities 字段。

- 标签: `intent-recognition`, `entity-extraction`, `ner`, `comparison`
- 记录于: 2026-06-19

## Q: 什么是单轮意图识别？什么是多轮意图识别？

### 单轮意图识别

**定义**：仅基于用户当前这一句话判断意图，不考虑对话历史。

```
输入: "北京明天天气怎么样"
输出: intent=weather_query
```

**特点**：
- 每次请求独立处理
- 实现简单，一次分类调用即可
- 适用于无上下文的场景（搜索引擎、一次性命令）

**局限**：无法处理指代和省略——"那后天呢？"单独看无法理解。

### 多轮意图识别

**定义**：结合对话历史判断当前意图，需要处理指代消解、话题延续/切换、渐进式槽位填充。

```
Turn 1: "北京明天天气怎么样"  → intent=weather_query, city=北京, date=明天
Turn 2: "那后天呢？"          → intent=weather_query, city=北京, date=后天
         ↑ 指代消解："那"→天气查询; "后天"更新了date槽位; city继承
Turn 3: "帮我订个机票吧"      → intent=book_flight（话题切换，新意图）
```

**核心挑战**：

| 挑战 | 说明 | 示例 |
|---|---|---|
| **指代消解** | "它"、"这个"指什么？ | "这个怎么用？" → 需要从历史推断"这个"是什么 |
| **话题延续 vs 切换** | 当前话是在追问还是换话题？ | "那价格呢"（延续）vs "换个话题"（切换） |
| **槽位继承与更新** | 哪些参数沿用、哪些被覆盖？ | "改成上海出发"——只更新 departure，其他不变 |
| **意图修正** | 用户纠正之前的意图 | "不是订机票，我要订酒店" |

**实现要点**：
- 将最近 N 轮对话（或摘要）拼入 prompt，让 LLM 在上下文中做意图判断
- 维护对话状态对象（当前意图 + 已填槽位 + 确认状态）
- 显式检测话题切换信号——新出现的无关实体、否定词、转折词

- 标签: `intent-recognition`, `multi-turn`, `single-turn`, `dialogue-state`
- 记录于: 2026-06-19

## Q: 基于 LLM 的意图识别与传统 NLU 方法（如 BERT 分类）有什么不同？

### 对比

| 维度 | 传统 NLU（BERT 分类器） | LLM-based（GPT/Claude） |
|---|---|---|
| **训练方式** | 有监督微调，需标注数据集 | Zero-shot / Few-shot，不需要（或极少）标注 |
| **意图体系** | 固定的，新增意图需重新训练 | 灵活的，Prompt 中直接描述新意图即可 |
| **推理速度** | 快（~5-20ms，本地 GPU） | 慢（~200-2000ms，API 调用） |
| **推理成本** | 低（自部署模型，边际成本趋近零） | 高（按 token 计费） |
| **准确率（域内）** | 高（专门训练过，>95%） | 中高（依赖 Prompt 质量，90-95%） |
| **泛化能力** | 差（分布外数据性能骤降） | 强（能处理未见过的表达方式） |
| **多意图 / 复杂意图** | 需要专门建模（多标签分类） | 自然支持（LLM 推理能力） |
| **可解释性** | 弱（分类概率） | 强（可要求 LLM 给出推理过程） |

### 各自适用场景

**选传统 NLU 的场景**：
- 意图体系固定、变化少
- 延迟要求极高（<50ms）
- 请求量极大、成本敏感
- 有充足的标注数据（>5000 条/意图）
- 典型：智能客服的一级意图路由

**选 LLM 的场景**：
- 意图体系频繁变更或尚未确定
- 需要处理复杂/开放域意图
- 冷启动阶段，没有标注数据
- 对延迟容忍度高（>500ms 可接受）
- 典型：通用 Agent 助手、复杂任务分解

### 实践中的混合方案

```
用户输入 → [轻量分类器(BERT)] → 高置信度(>0.9) → 直接路由
                               → 低置信度(<0.9) → [LLM 精细判断] → 路由
```

轻量分类器处理 80% 的明确意图（快、便宜），LLM 处理 20% 的模糊/复杂意图（准、贵）。兼顾速度和覆盖度。

- 标签: `intent-recognition`, `llm`, `bert`, `nlu`, `comparison`
- 记录于: 2026-06-19

## Q: Agent 中为什么不能把用户所有输入都直接丢给大模型处理？

### 五个核心原因

**1. 成本不可控**

```
假设：日均 100 万次请求，平均 input 500 tokens + output 200 tokens
GPT-4o: ~$0.005/请求 → $5,000/天 → $150,000/月
轻量分类器: ~$0.00001/请求 → $10/天

80% 的请求是简单意图，用分类器处理就够了。
```

没有意图分层，所有请求都走大模型，成本白白放大 5-10 倍。

**2. 延迟叠加**

大模型调用延迟 200-2000ms，而很多请求根本不需要这么强的推理能力。用户问"现在几点"不需要 Claude Opus 思考 2 秒——查时间 API 即可。意图识别先分流，简单请求走快速通道。

**3. 行为不可控**

不加意图约束的大模型可能：
- 对应该拒绝的请求给出回答（如超出业务范围的问题）
- 随意切换话题或角色
- 执行不该执行的操作

意图识别相当于"门禁"——只让合法的请求进入后续处理，不属于系统能力范围的及早拦截。

**4. 安全风险**

所有输入直接给大模型 = 放大 prompt injection 攻击面。恶意用户可以通过精心构造的输入让模型执行非预期操作。意图识别层可以作为第一道过滤：

```
用户输入 → [意图识别 + 安全检查] → 合法意图 → 大模型处理
                                 → 疑似注入 → 拒绝/脱敏
```

**5. 可观测性和可调试性差**

所有请求走同一个大模型黑箱，出了问题很难定位：是意图理解错了？检索没找到？生成质量差？分层后每个环节有独立的输入输出日志，可以精确定位问题在哪一步。

### 正确做法

分层处理，按需调用：

```
用户输入
  │
  ├─ 意图识别（轻量模型/规则） → 简单意图 → 直接响应（不用大模型）
  │                            → 复杂意图 → 大模型处理
  │
  ├─ 安全过滤（规则 + 小模型） → 拦截恶意请求
  │
  └─ 格式校验（代码逻辑） → 拦截格式错误
```

**大模型应该只处理真正需要其推理能力的请求**——开放域问答、复杂任务分解、创意生成。简单的查询、格式化的操作、固定流程的任务，用更轻、更快、更便宜的方式处理。

- 标签: `intent-recognition`, `cost-optimization`, `architecture`, `safety`
- 记录于: 2026-06-19

## Q: 如何用 Few-shot Prompt 让 LLM 做意图分类？

### 基本结构

```
System Prompt:
  角色定义 + 意图列表 + 输出格式约束

User Prompt:
  Few-shot 示例（每个意图 1-2 个） + 当前用户输入
```

### 完整示例

```markdown
## System Prompt

你是一个意图分类器。将用户输入分类到以下意图之一：

- `weather_query`: 查询天气信息
- `book_flight`: 预订机票
- `chitchat`: 闲聊、问候
- `unknown`: 无法归入以上类别

仅输出 JSON，不要解释。

## Few-shot 示例

用户: 明天北京会下雨吗
输出: {"intent": "weather_query", "confidence": 0.95}

用户: 帮我订一张去上海的机票
输出: {"intent": "book_flight", "confidence": 0.98}

用户: 你好呀，最近怎么样
输出: {"intent": "chitchat", "confidence": 0.92}

用户: 量子计算的原理是什么
输出: {"intent": "unknown", "confidence": 0.88}

## 当前输入

用户: {user_query}
输出:
```

### Few-shot 设计要点

**1. 示例数量**：每个意图 1-3 个示例，总共不超过 10-15 个。太多浪费 token 且收益递减。

**2. 示例选择**：
- 覆盖典型表达（"订机票" "买张飞机票" "我要坐飞机"）
- 包含边界 case（容易混淆的意图各给一个对比示例）
- 包含一个 `unknown` 示例，让模型知道可以拒绝分类

**3. 示例顺序**：将与当前查询最可能相关的示例放最后——LLM 对末尾内容的注意力更高（recency bias）。可以用 embedding 相似度动态选择最相关的 few-shot 示例（Dynamic Few-shot）。

**4. 动态 Few-shot**：

```python
def get_dynamic_examples(query: str, example_pool: list, k: int = 5):
    query_embedding = embed(query)
    similarities = [cosine_sim(query_embedding, embed(ex.text)) for ex in example_pool]
    top_k = sorted(range(len(similarities)), key=lambda i: similarities[i], reverse=True)[:k]
    return [example_pool[i] for i in top_k]
```

从大例子池中动态选择与当前 query 最相似的 K 个示例，比固定示例更精准。

- 标签: `few-shot`, `intent-classification`, `prompt-design`, `dynamic-few-shot`
- 记录于: 2026-06-19

## Q: Chain-of-Thought 对复杂意图识别有帮助吗？为什么？

### 结论：有帮助，但仅限于复杂/模糊意图

**有效场景**：

1. **复合意图拆分**：一句话包含多个意图
```
用户: "查一下明天北京天气，顺便帮我订张去上海的机票"

不用 CoT: {"intent": "book_flight"}  ← 只识别了一个
用 CoT:
  思考：用户提了两个请求。第一个是查天气（"查一下明天北京天气"），
  第二个是订机票（"帮我订张去上海的机票"）。这是两个独立意图。
  输出: {"intents": ["weather_query", "book_flight"]}
```

2. **需要推理的意图**：意图不直接表达，需要从上下文推断
```
用户: "我后天要去上海出差，但还没准备好"

不用 CoT: {"intent": "chitchat"}  ← 误判为闲聊
用 CoT:
  思考：用户说"后天要去上海出差"暗示有出行需求，"还没准备好"
  暗示需要帮助安排。最可能的意图是寻求出行准备帮助（订票/订酒店）。
  输出: {"intent": "travel_planning", "confidence": 0.75}
```

3. **消歧**：同一表述可映射多个意图
```
用户: "苹果多少钱"

CoT: 考虑上下文——如果之前在讨论水果，意图是 fruit_price；
如果在讨论电子产品，意图是 product_price。无上下文时倾向于 product_price
（更常见的查询场景）。
```

### 为什么有帮助

1. **强制模型分解问题**：CoT 让模型不是直接从输入跳到标签，而是先分析输入的各个要素——提到了哪些实体？有几个动作？语气是什么？这种中间推理减少了跳跃式判断的错误。
2. **隐式上下文利用**：CoT 过程中模型会自然地引入对话历史、常识推理，比直接分类更能利用上下文信息。
3. **可解释性**：推理过程本身就是"为什么这么分类"的解释，便于 Badcase 分析。

### 何时不用 CoT

- **意图明确的简单查询**："今天天气"——分类器直接秒判，CoT 是浪费。
- **极端延迟敏感**：CoT 增加 output tokens（~50-100 tokens），延迟增加 100-300ms。
- **意图数量少且边界清晰**：5 个不重叠的意图，Few-shot 直接分类的准确率已经 >98%。

**实践建议**：默认不用 CoT（快且省 token）；只在 Few-shot 直接分类的准确率 <90% 的复杂场景启用。可以两阶段——先不带 CoT 快速分类，置信度低时再带 CoT 重试。

- 标签: `chain-of-thought`, `intent-recognition`, `reasoning`, `disambiguation`
- 记录于: 2026-06-19

## Q: 如何设计 System Prompt 来约束 Agent 的意图识别范围？

### 设计原则

**明确的"只做这些"比"别做那些"更有效**。

### 模板结构

```markdown
你是 [产品名] 的意图分类模块。你的唯一职责是将用户输入分类到以下意图之一。

## 支持的意图

| 意图标签 | 含义 | 典型表达 |
|---|---|---|
| `order_query` | 查询订单状态 | "我的快递到哪了"、"订单号 xxx 查一下" |
| `refund_request` | 申请退款 | "我要退货"、"怎么申请退款" |
| `product_consult` | 商品咨询 | "这个手机支持 5G 吗" |
| `complaint` | 投诉 | "你们服务太差了"、"我要投诉" |
| `out_of_scope` | 不属于以上任何类别 | "今天天气怎么样"、"帮我写代码" |

## 输出规则

1. 必须且只能输出上述意图之一
2. 如果无法确定，输出 `out_of_scope`
3. 不要回答用户的问题，只做分类
4. 输出格式：`{"intent": "xxx", "confidence": 0.xx}`
```

### 关键约束技巧

**1. 封闭式意图列表**

显式列出所有合法意图，并加一个兜底类（`out_of_scope` / `unknown`）。不要用"等"、"类似"这种开放式表述——模型会自行发挥。

**2. "不要做 X"的正面表达**

```
差: "不要回答用户的问题"
好: "你的唯一输出是意图分类的 JSON，不包含任何其他文字"
```

正面约束比负面约束更稳定——模型对"只做 A"的遵循度 > 对"不要做 B"的遵循度。

**3. 边界 case 显式处理**

```markdown
## 边界处理

- 用户同时表达多个意图时，输出最主要的一个（优先级：complaint > refund > order > product > out_of_scope）
- 用户意图模糊时，优先分类为 `out_of_scope` 而非猜测
- 即使用户用命令式语气（"告诉我 xxx"），仍然只做分类不回答
```

**4. 角色隔离**

```markdown
你不是对话 Agent，不是助手，不是客服。你是一个分类函数。
输入：一段文本。输出：一个 JSON。没有其他交互。
```

用极端的角色限定减少模型"想帮忙回答问题"的倾向。

- 标签: `system-prompt`, `intent-recognition`, `constraint-design`, `prompt-engineering`
- 记录于: 2026-06-19

## Q: 用户意图模糊（如"帮我弄一下"）时，Prompt 中如何设计追问策略？

### 问题本质

模糊意图的核心特征：**动词宽泛 + 缺少关键实体**。"帮我弄一下"——"弄"可以是查询、修改、删除、创建中的任何一个；"一下"没有指向任何具体对象。

### Prompt 中的追问设计

```markdown
## 模糊意图处理规则

当用户输入无法明确分类到任何具体意图时：

1. 不要猜测意图
2. 生成一个澄清问题，遵循以下原则：
   - 提供 2-3 个最可能的选项供用户选择
   - 选项基于上下文推断（如果有对话历史）
   - 问题简短，不超过一句话

输出格式：
{
  "intent": "need_clarification",
  "clarification": {
    "question": "你想要我帮你做哪个？",
    "options": ["查询订单状态", "修改收货地址", "申请退款"]
  }
}
```

### 追问策略的层次

**Level 1：选项式追问（优先）**

```
用户: "帮我弄一下"
Agent: "你是想：A. 查询订单状态  B. 修改订单信息  C. 其他？"
```

选项从上下文和高频意图推断。用户选择比开放式回答更快，且结果结构化。

**Level 2：引导式追问（次选）**

```
用户: "帮我处理一下那个问题"
Agent: "你说的是哪个问题？能具体描述一下吗？"
```

当连选项都无法推断时，要求用户补充具体信息。

**Level 3：确认式追问（有一定把握时）**

```
用户: "弄一下上次说的"
Agent: "你是说上次提到的退款申请吗？"（基于对话历史推断）
```

有 >70% 的把握时，直接给出推测并确认，比开放式追问更高效。

### 实现要点

```python
def handle_vague_intent(query, history, intent_result):
    if intent_result.confidence > 0.7:
        # 有一定把握 → 确认式
        return confirm_intent(intent_result.top_intent, query)
    
    if history:
        # 有对话历史 → 基于历史推断选项
        options = infer_options_from_history(history, top_k=3)
        return ask_with_options(options)
    
    # 无任何线索 → 引导式
    return ask_open_ended("能具体描述一下你想做什么吗？")
```

**关键原则**：
- **最多追问 2 次**，超过 2 次用户会烦。第 2 次仍不清楚就兜底到通用 Agent。
- **追问要带上上次的回答**，不要让用户重复说过的信息。
- **提供跳出选项**：始终包含"其他"选项，不要把用户框死在有限选项里。

- 标签: `clarification`, `vague-intent`, `interaction-design`, `prompt-design`
- 记录于: 2026-06-19

## Q: 意图识别结果以 JSON 格式输出时，Prompt 怎么写更稳定？

### 稳定 JSON 输出的五个关键技巧

**1. 提供 JSON Schema 定义**

```markdown
按以下 JSON Schema 输出：
{
  "intent": string,       // 必填，值必须是: "weather", "booking", "chitchat", "unknown"
  "confidence": number,   // 必填，0.0-1.0
  "entities": {           // 可选
    "city": string,
    "date": string
  }
}
```

显式定义字段名、类型、取值范围。比"输出 JSON 格式"这种笼统描述稳定得多。

**2. Few-shot 示例中保持格式一致**

所有示例的 JSON 结构必须完全一致——字段顺序、嵌套层级、引号风格。模型会模仿示例的格式：

```
# 好：所有示例格式统一
输入: "明天天气"
输出: {"intent": "weather", "confidence": 0.95, "entities": {"date": "明天"}}

输入: "你好"
输出: {"intent": "chitchat", "confidence": 0.90, "entities": {}}

# 差：格式不一致
输入: "明天天气"
输出: {"intent": "weather", "confidence": 0.95, "entities": {"date": "明天"}}

输入: "你好"
输出: {intent: "chitchat"}  ← 缺少 confidence、entities，引号风格不同
```

**3. 强约束 Prompt 尾部**

```markdown
严格输出 JSON，不要输出任何解释、前缀或后缀。
输出以 `{` 开头，以 `}` 结尾。
```

防止模型在 JSON 前后加"好的，以下是分类结果："这类废话（导致 JSON 解析失败）。

**4. 使用 Function Calling / Structured Output（最稳定方案）**

如果模型支持 tool use / function calling，将意图识别定义为一个 function：

```python
tools = [{
    "type": "function",
    "function": {
        "name": "classify_intent",
        "parameters": {
            "type": "object",
            "properties": {
                "intent": {"type": "string", "enum": ["weather", "booking", "chitchat", "unknown"]},
                "confidence": {"type": "number", "minimum": 0, "maximum": 1},
            },
            "required": ["intent", "confidence"]
        }
    }
}]
```

Function calling 的输出由模型底层的结构化生成机制保证格式正确，比靠 Prompt 约束稳定得多。

**5. 防御性解析**

即使 Prompt 写得再好，也要在代码层做容错解析：

```python
import json, re

def safe_parse_intent(raw_output: str) -> dict:
    # 尝试直接解析
    try:
        return json.loads(raw_output)
    except json.JSONDecodeError:
        pass
    
    # 尝试提取 JSON 块（模型可能在 JSON 前后加了文字）
    match = re.search(r'\{.*\}', raw_output, re.DOTALL)
    if match:
        try:
            return json.loads(match.group())
        except json.JSONDecodeError:
            pass
    
    # 兜底
    return {"intent": "unknown", "confidence": 0.0, "parse_error": True}
```

**稳定性排序**：Function Calling > JSON Schema + 统一示例 > 纯文字 Prompt 约束

- 标签: `json-output`, `structured-output`, `function-calling`, `prompt-stability`
- 记录于: 2026-06-19

## Q: 用户一句话中包含多个意图，识别难点在哪里？

### 多意图的类型

```
并列意图: "查一下天气，再帮我订个机票"     ← 两个独立意图
条件意图: "如果明天下雨就帮我取消行程"      ← 意图 B 依赖条件 A
嵌套意图: "帮我订机票，要最便宜的那种"      ← 主意图 + 约束子意图
隐含意图: "明天出差去上海"                  ← 表面无意图，但隐含订票/订酒店需求
```

### 四大难点

**1. 意图分割——在哪里切分？**

传统分类器输出单一标签，天然不支持多意图。即使用 LLM，也需要正确识别意图的边界：

```
"帮我查一下昨天的订单然后把地址改成北京"
 └──── 意图1: order_query ────┘└── 意图2: update_address ──┘
```

自然语言没有显式的意图分隔符，模型需要从语义上判断哪些词属于哪个意图。

**2. 优先级和执行顺序**

多个意图可能有依赖关系：
- "查一下订单，如果到了就确认收货"——意图 2 依赖意图 1 的结果
- "取消订单，然后退款"——必须先取消再退款，顺序不能反

系统需要推断出正确的执行顺序，而非简单并行。

**3. 实体归属——参数属于哪个意图？**

```
"帮我订一张明天去上海的机票和一间酒店"
- "明天"属于机票还是酒店？（两个都属于）
- "上海"属于机票还是酒店？（两个都属于）
```

共享实体的归属问题在单意图模型中不存在，多意图时需要显式处理。

**4. 输出结构设计**

单意图：`{"intent": "xxx"}` 足够。
多意图需要列表 + 关系：

```json
{
  "intents": [
    {"intent": "order_query", "entities": {"order_id": "xxx"}, "priority": 1},
    {"intent": "update_address", "entities": {"address": "北京"}, "priority": 2, "depends_on": 0}
  ]
}
```

### 解决方案

**方案一：Prompt 显式要求多意图识别**

```markdown
用户可能在一句话中表达多个意图。请识别所有意图并按执行顺序排列。
如果只有一个意图，也用数组格式输出。
```

**方案二：先分句再分类**

```python
# Step 1: 用 LLM 将多意图语句拆分为单意图子句
sub_sentences = split_to_sub_intents("查天气，再订机票")
# → ["查天气", "订机票"]

# Step 2: 对每个子句独立做意图识别
intents = [classify(s) for s in sub_sentences]
```

**方案三：多标签分类**

训练时将多意图样本标注为多个标签（multi-label），模型输出每个意图的概率，取超过阈值的所有意图。适合传统 NLU 方案。

- 标签: `multi-intent`, `intent-recognition`, `priority`, `entity-attribution`
- 记录于: 2026-06-19

## Q: 什么是"意图漂移"？多轮对话中如何避免？

### 定义

**意图漂移（Intent Drift）**是指在多轮对话中，系统对用户核心意图的理解逐渐偏离用户的真实目的。

```
Turn 1: 用户: "我想退款"              → intent: refund（正确）
Turn 2: 用户: "这个商品质量太差了"     → intent: complaint（漂移了——用户在解释退款原因，不是投诉）
Turn 3: 用户: "能快点处理吗"          → intent: urgent_request（又漂移了——用户只是催促退款进度）
Turn 4: Agent 开始处理"紧急请求"，退款流程被丢了
```

每一轮单独看，意图分类都"合理"，但串起来看系统丢失了核心意图"退款"。

### 产生原因

1. **无状态分类**：每轮独立做意图识别，不考虑对话主线
2. **表面语义干扰**：用户的补充说明被识别为新意图（抱怨 → 投诉，催促 → 紧急请求）
3. **上下文窗口过长**：早期的核心意图被大量后续对话稀释，模型注意力偏向最近的内容
4. **模型的 recency bias**：LLM 天然对上下文末尾的信息给予更多权重

### 避免方案

**1. 意图状态锁定**

一旦确认核心意图，将其"锁定"在对话状态中，后续轮次只做子意图/补充识别：

```python
class DialogueState:
    primary_intent: str = None       # 主意图（一旦确认不轻易改变）
    sub_intents: list[str] = []      # 子意图/补充意图
    intent_confirmed: bool = False   # 是否已确认
    
    def update(self, new_intent, confidence):
        if not self.intent_confirmed:
            self.primary_intent = new_intent
            if confidence > 0.9:
                self.intent_confirmed = True
        else:
            # 主意图已锁定，新识别结果作为子意图
            if new_intent != self.primary_intent:
                self.sub_intents.append(new_intent)
```

**2. 显式话题切换检测**

区分"补充同一意图"和"切换新意图"：

```markdown
在 Prompt 中加入：
判断用户当前发言与主意图的关系：
- CONTINUE: 在补充/追问同一个问题 → 保持主意图
- SWITCH: 明确要换一个新话题 → 更新主意图
- DIGRESS: 短暂离题但主意图不变 → 记录但不改变主意图
```

**3. 主意图注入**

每轮对话时，将当前主意图显式注入 prompt：

```markdown
当前对话的主意图是：退款申请（已确认）
用户正在补充退款原因。请在主意图框架下理解用户的最新输入。
```

这相当于用 prompt 对抗 recency bias——强制模型记住核心意图。

**4. 定期确认**

对话进行到 3-5 轮时，主动确认主意图是否仍然正确：

```
Agent: "确认一下，你主要是想申请退款对吗？还是有其他需要？"
```

**5. 意图摘要**

每 N 轮生成一次对话摘要，摘要中显式标注主意图，用摘要替代冗长的对话历史注入 prompt，减少信息稀释。

- 标签: `intent-drift`, `multi-turn`, `dialogue-state`, `topic-tracking`
- 记录于: 2026-06-19

## Q: 长上下文对话中，为什么模型可能忽略早期的意图？

### 现象

在 20+ 轮对话后，用户在 Turn 3 提出的核心需求被模型"遗忘"，转而关注最近几轮的内容。即使上下文窗口足够大（128K tokens），模型仍然可能忽略早期信息。

### 三个根本原因

**1. 注意力分布的位置偏差（Positional Bias）**

Transformer 的自注意力机制在理论上对所有位置等权，但实际训练后模型呈现明显的**U 形注意力分布**——对开头（system prompt 区域）和结尾（最近几轮）的注意力最高，中间部分最低。

```
注意力权重
  ↑
  │ ██                                                    ████
  │ ████                                              ████████
  │ ██████                                        ████████████
  │ ████████████████████████████████████████████████████████████
  └────────────────────────────────────────────────────────────→
    System     早期对话        中期对话        最近对话     当前
    Prompt                                               输入
```

这意味着 Turn 3 的意图信息如果落在"注意力谷底"，即使在窗口内也可能被弱化。

**2. 信息稀释**

早期意图表达只有 1-2 句（~50 tokens），但后续 20 轮对话可能有 5000+ tokens。核心信息在大量后续内容中被"稀释"——信噪比极低。模型在做注意力加权时，50 tokens 的意图信号被 5000 tokens 的对话噪音淹没。

**3. LLM 的 Recency Bias**

LLM 在训练过程中学到了"最近的信息通常最相关"的 prior——因为在大多数训练数据中确实如此。这导致模型在做决策时过度依赖最近几轮的内容，即使早期内容更重要。

### 解决方案

**1. 对话摘要注入（最推荐）**

```python
def compress_context(history: list[Message], max_tokens: int = 2000):
    if count_tokens(history) < max_tokens:
        return history
    
    # 保留: System Prompt（全部）+ 最近 3 轮（全部）
    # 中间部分: 生成摘要
    summary = llm_summarize(history[:-3], focus="保留核心意图和关键决策")
    
    return [system_prompt, summary_message, *history[-3:]]
```

用 LLM 对中间对话生成摘要，摘要中显式保留核心意图。摘要在上下文中占据更紧凑的位置，信噪比远高于原始对话。

**2. 意图锚点（Intent Anchor）**

在每轮对话的上下文中，在最近几轮之前显式插入一行"意图锚点"：

```
[系统提醒] 当前对话的核心意图：退款申请（Turn 3 确认）。当前阶段：等待用户提供订单号。
```

放在最近对话的紧前方（高注意力区域），强制模型"看到"早期意图。

**3. 滑动窗口 + 关键信息固定**

```
上下文结构:
[System Prompt]                    ← 固定不动
[意图摘要 + 关键实体]              ← 固定不动，每轮更新
[最近 N 轮对话]                   ← 滑动窗口
[当前用户输入]                    ← 最新
```

不同信息有不同的生存策略：关键信息（意图、已确认实体）常驻；对话细节只保留最近 N 轮。

**4. 关键信息前置（Recency Hack）**

既然模型对末尾注意力最高，就把关键信息放到末尾：

```
用户最新输入: "好的"
[系统注入] 提醒：用户的核心需求是退款，已提供订单号 xxx，当前在等待退款审批。
```

这虽然是"hack"，但在实践中非常有效——利用模型的 recency bias 而非对抗它。

- 标签: `long-context`, `attention-bias`, `intent-loss`, `context-management`, `dialogue-summary`
- 记录于: 2026-06-19
