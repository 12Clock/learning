# 实战场景

> 涵盖：客服、个人助理、代码生成等具体场景中的意图识别实践。

## Q: 客服 Agent 中，如何区分"投诉"和"咨询"这两种相近意图？

### 为什么难区分

```
"这个手机的电池怎么这么差"   → 投诉？还是在咨询电池信息？
"你们的退货政策是什么"       → 咨询？还是在为投诉做准备？
"我买了三天就坏了"           → 投诉？还是在描述需要售后的情况？
```

两者的语义重叠区域很大——用户在"描述问题"时可能是咨询也可能是投诉。

### 区分维度

| 维度 | 咨询信号 | 投诉信号 |
|---|---|---|
| **情感词** | 中性（"想了解"、"请问"） | 负面（"太差了"、"受不了"、"垃圾"） |
| **动作诉求** | 获取信息（"是什么"、"怎么用"） | 要求补偿/解决（"要退款"、"给个说法"） |
| **句式** | 疑问句为主 | 陈述句/感叹句为主 |
| **时态** | 未来/一般（"如果...会..."） | 过去（"已经...了"、"买了...就..."） |
| **隐含期望** | 答案/信息 | 行动/补偿 |

### 实现方法

**方法一：多维度 Prompt**

```markdown
区分"咨询"和"投诉"时，考虑以下信号：
1. 情感极性：用户是否表达不满？
2. 是否有具体的负面经历描述？
3. 用户是否要求补偿或解决方案？

如果仅在询问信息（即使涉及问题描述），分类为 consult。
如果表达不满且期望行动/补偿，分类为 complaint。
```

**方法二：情感分析辅助**

```python
sentiment = analyze_sentiment(query)
intent = classify_intent(query)

# 情感强负面 + 意图模糊 → 倾向投诉
if sentiment.score < -0.5 and intent.confidence < 0.7:
    intent = "complaint"
```

**方法三：对比 Few-shot**

专门针对容易混淆的 pair 给出对比示例：

```
# 这是咨询（在问信息）
用户: "你们的退货政策是什么？多少天内可以退？"
输出: {"intent": "consult"}

# 这是投诉（表达不满 + 要求行动）
用户: "你们的退货政策也太坑了吧，买了三天就不让退"
输出: {"intent": "complaint"}
```

### 实践建议

- 如果业务上对"投诉"有特殊处理流程（如转人工、升级优先级），宁可多识别一些投诉（召回优先）
- 如果只是分流（不同回答模板），则精确率和召回率均衡即可

- 标签: `customer-service`, `complaint-vs-consult`, `sentiment`, `intent-disambiguation`
- 记录于: 2026-06-19

## Q: 个人助理 Agent 中，如何识别"设置提醒"与"查询天气"这类跨域意图？

### 跨域意图的挑战

个人助理面对开放域场景——用户可能问天气、设提醒、发消息、查日历、播音乐... 意图跨越多个完全不相关的领域。

**核心困难**：
- 意图数量可能 >30 个，远多于垂直场景
- 不同领域的表达方式差异大
- 新领域随时可能出现

### 解决方案：两级路由

```
用户输入 → [领域分类器] → 领域标签 → [领域内意图分类器] → 具体意图
            (5-8 个领域)              (每个领域 3-8 个意图)
```

```python
DOMAIN_INTENTS = {
    "calendar": ["set_reminder", "query_schedule", "create_event", "cancel_event"],
    "weather": ["query_weather", "weather_alert"],
    "messaging": ["send_message", "read_message"],
    "music": ["play_music", "pause_music", "search_song"],
    "general": ["chitchat", "unknown"],
}

domain = classify_domain(query)         # 第一级：5-8 选 1
intent = classify_intent(query, domain) # 第二级：3-8 选 1
```

### 关键词+语义混合

对于"设置提醒" vs "查询天气"这种差异明显的意图，关键词匹配已经足够：

```python
KEYWORD_HINTS = {
    "set_reminder": ["提醒", "闹钟", "别忘了", "记得", "到时候"],
    "query_weather": ["天气", "下雨", "温度", "穿什么"],
}

def quick_intent(query):
    for intent, keywords in KEYWORD_HINTS.items():
        if any(kw in query for kw in keywords):
            return intent, 0.85
    return None, 0.0  # 无关键词命中 → 走 LLM 分类
```

关键词匹配处理 60-70% 的明确请求（<1ms），剩余走 LLM（200ms+）。

### 新领域扩展

个人助理的意图体系会持续扩展。用 LLM 的 zero-shot 能力做兜底：

```markdown
如果用户请求不属于已知的任何领域，输出:
{"domain": "unknown", "intent": "unknown", "raw_request": "用户原始描述"}
```

线上收集 unknown 样本后，聚类分析 → 发现新领域 → 定义新意图 → 补充示例。

- 标签: `personal-assistant`, `cross-domain`, `keyword-matching`, `hierarchical-intent`
- 记录于: 2026-06-19

## Q: 代码生成 Agent 中，如何识别用户是想"写代码"还是"解释代码"？

### 关键区分信号

| 信号 | 写代码 | 解释代码 |
|---|---|---|
| **动作动词** | "写"、"实现"、"创建"、"生成" | "解释"、"什么意思"、"为什么"、"怎么理解" |
| **是否附带代码** | 通常不附带（或附带部分作为参考） | 通常附带需要解释的代码 |
| **期望输出** | 代码块 | 自然语言解释 |
| **上下文** | 描述需求/功能 | 引用已有代码 |

### 实现

**Prompt 设计**：

```markdown
将用户请求分类为以下意图之一：
- `code_generate`: 用户需要你编写/生成代码
- `code_explain`: 用户需要你解释已有代码的含义/逻辑
- `code_fix`: 用户需要你修复/调试代码中的问题
- `code_refactor`: 用户需要你重构/优化已有代码
- `code_review`: 用户需要你审查代码质量

判断依据：
1. 如果用户提供了代码并问"这是什么/为什么"→ code_explain
2. 如果用户描述功能需求但没有提供代码 → code_generate
3. 如果用户提供了代码并说"有bug/不工作"→ code_fix
4. 如果用户提供了代码并说"优化/改进"→ code_refactor
```

### 特殊 case

```
"用 Python 写一个快排"            → code_generate（明确"写"）
"这段代码是什么意思？[代码块]"     → code_explain（明确"什么意思" + 附代码）
"帮我看看这段代码"                 → 模糊！可能是 explain / review / fix
```

对于模糊 case，用代码上下文辅助判断：
- 代码有语法错误 → 倾向 code_fix
- 代码正常但用户表述简短 → 倾向 code_review
- 无代码附带 → 倾向 code_generate

- 标签: `code-agent`, `code-generation`, `code-explanation`, `intent-disambiguation`
- 记录于: 2026-06-19

## Q: 多轮对话中，如何判断用户是在开启新意图还是延续上一轮意图？

### 判断信号

| 信号 | 延续（Continue） | 新意图（Switch） |
|---|---|---|
| **指代词** | 有（"它"、"这个"、"上面说的"） | 无 |
| **话题词重叠** | 高（使用相同实体/术语） | 低（出现全新实体） |
| **转折标记** | 无 | 有（"另外"、"换个话题"、"对了"） |
| **语义相似度** | 与上轮相似度 >0.6 | 与上轮相似度 <0.4 |
| **时间间隔** | 短（<30s） | 长（>5min） |

### 实现方案

**方案一：LLM 直接判断**

```markdown
给定对话历史和用户最新输入，判断用户是：
- CONTINUE: 在延续/追问上一个话题
- SWITCH: 在开启一个新话题
- CORRECT: 在纠正/修改上一轮的意图

对话历史:
{last_3_turns}

用户最新输入: {current_query}

判断:
```

**方案二：规则 + 模型混合**

```python
def detect_topic_transition(current_query, history):
    # 规则层（快速判断明确 case）
    if has_explicit_switch_marker(current_query):
        return "SWITCH"   # "换个话题"、"另外问一下"
    
    if has_coreference(current_query):
        return "CONTINUE" # "它"、"这个"、"那个呢"
    
    # 模型层（处理模糊 case）
    similarity = compute_semantic_similarity(
        current_query, history[-1].query
    )
    
    if similarity > 0.6:
        return "CONTINUE"
    elif similarity < 0.3:
        return "SWITCH"
    else:
        # 不确定时让 LLM 判断
        return llm_judge_transition(current_query, history)
```

**方案三：embedding 相似度**

计算当前 query 与上一轮 query 的 embedding 余弦相似度。阈值通常在 0.4-0.6 之间：

```python
sim = cosine_similarity(embed(current), embed(last_query))
if sim > 0.55:
    return "CONTINUE"
else:
    return "SWITCH"
```

简单但对表面形式敏感（"查天气"和"天气怎么样"相似度高，但"查天气"和"明天呢？"相似度低，即使后者是延续）。需要配合指代检测使用。

### 状态转换后的处理

```python
if transition == "CONTINUE":
    # 继承上一轮的意图和槽位，只做增量更新
    state.update_slots(new_entities)
    
elif transition == "SWITCH":
    # 归档当前意图，开启新的意图识别
    archive_intent(state.current_intent)
    state = fresh_state()
    state.current_intent = classify_intent(current_query)
    
elif transition == "CORRECT":
    # 用户在纠正，替换当前意图
    state.current_intent = classify_intent(current_query)
    state.clear_slots()  # 槽位可能也需要重新提取
```

- 标签: `topic-transition`, `multi-turn`, `coreference`, `semantic-similarity`, `dialogue-management`
- 记录于: 2026-06-19
