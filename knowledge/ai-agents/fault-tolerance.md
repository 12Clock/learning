# 容错机制

> 涵盖：重试降级、自我纠错、状态回滚、多 Agent 容错、防护栏与安全网。

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

## Q: Agent 出现死循环怎么办？异常处理机制如何设计？

### 死循环的三种类型

**1. 推理循环（Reasoning Loop）**

LLM 反复生成相同或相似的 Thought，不产生有效 Action：

```
Turn 1: Thought: 我需要查询用户信息 → Action: query_user(id=123)
Turn 2: Observation: 用户不存在
Turn 3: Thought: 我需要查询用户信息 → Action: query_user(id=123)  ← 重复
Turn 4: Observation: 用户不存在
Turn 5: Thought: 我需要查询用户信息 → Action: query_user(id=123)  ← 又重复
...
```

**2. 工具循环（Tool Loop）**

Agent 在多个工具之间反复切换，形成环：

```
Action: search("天气") → 结果不满意
Action: web_browse(url) → 页面加载失败
Action: search("天气预报") → 结果不满意
Action: web_browse(url2) → 又失败
...  ← search 和 browse 交替循环
```

**3. 修复循环（Fix Loop）**

Agent 尝试修复一个问题但每次引入新问题：

```
修复 Bug A → 引入 Bug B → 修复 Bug B → 引入 Bug C → 修复 Bug C → 引入 Bug A
```

### 防死循环机制

**层级一：硬性限制（最基本的安全网）**

```python
class AgentLoop:
    def __init__(self, max_iterations=15, max_time_seconds=300):
        self.max_iterations = max_iterations
        self.max_time_seconds = max_time_seconds
    
    def run(self, query: str) -> str:
        start_time = time.time()
        
        for i in range(self.max_iterations):
            # 超时检查
            if time.time() - start_time > self.max_time_seconds:
                return self._graceful_exit("timeout", partial_results)
            
            result = self.step(query)
            
            if result.is_final:
                return result.answer
        
        # 达到最大迭代次数
        return self._graceful_exit("max_iterations", partial_results)
    
    def _graceful_exit(self, reason, partial_results):
        # 不是简单报错，而是用已有信息给出最佳回答
        return self.llm.generate(
            f"你已经尝试了多次但未能完成任务。原因: {reason}\n"
            f"已收集的部分信息: {partial_results}\n"
            f"请基于已有信息给出最佳回答，并告知用户哪些部分未能完成。"
        )
```

**层级二：重复检测（识别循环模式）**

```python
class LoopDetector:
    def __init__(self, window=5, similarity_threshold=0.9):
        self.action_history = []
        self.window = window
        self.threshold = similarity_threshold
    
    def check(self, action: str) -> bool:
        """返回 True 表示检测到循环"""
        self.action_history.append(action)
        
        if len(self.action_history) < self.window:
            return False
        
        recent = self.action_history[-self.window:]
        
        # 检测1：完全相同的动作重复
        if len(set(recent)) == 1:
            return True
        
        # 检测2：周期性模式（A→B→A→B→A）
        for period in range(1, len(recent) // 2 + 1):
            pattern = recent[:period]
            is_periodic = all(
                recent[i] == pattern[i % period]
                for i in range(len(recent))
            )
            if is_periodic:
                return True
        
        # 检测3：语义相似度（用 embedding 检测变体重复）
        if self._semantic_similarity(recent[-1], recent[-2]) > self.threshold:
            self.repeat_count += 1
            return self.repeat_count >= 3
        
        return False
```

**层级三：策略切换（检测到循环后的应对）**

```python
def handle_detected_loop(agent, loop_type, context):
    if loop_type == "same_action_repeat":
        # 策略1：强制换一种方法
        return agent.step_with_constraint(
            f"你之前的方法已经尝试了 3 次但没有效果。"
            f"请尝试完全不同的方法来解决这个问题。"
            f"禁止再使用: {context.last_action}"
        )
    
    elif loop_type == "tool_oscillation":
        # 策略2：暂停并反思
        reflection = agent.reflect(
            f"你在以下工具之间循环: {context.cycle_tools}\n"
            f"请分析为什么会循环，然后选择一个最可能成功的方案。"
        )
        return agent.step_with_plan(reflection)
    
    elif loop_type == "fix_regression":
        # 策略3：回滚到已知良好状态
        agent.rollback_to_checkpoint(context.last_good_state)
        return agent.step_with_constraint(
            "之前的修复引入了新问题。请重新分析原始问题，"
            "制定一个不会引入副作用的修复方案。"
        )
```

### 完整异常处理框架

```python
class AgentExceptionHandler:
    def execute_with_protection(self, agent, task):
        try:
            return agent.run(task)
        
        except ContextOverflowError:
            # 上下文溢出：压缩历史后重试
            agent.compress_context()
            return agent.run(task)
        
        except ToolExecutionError as e:
            if e.is_retryable:
                return self.retry_with_backoff(agent, task, max_retries=3)
            else:
                # 不可重试的错误：告知 LLM 工具不可用
                agent.disable_tool(e.tool_name)
                return agent.run(task)  # 让 LLM 用其他方式完成
        
        except LoopDetectedError as e:
            return handle_detected_loop(agent, e.loop_type, e.context)
        
        except RateLimitError:
            # API 限流：等待后重试
            time.sleep(e.retry_after)
            return agent.run(task)
        
        except TokenBudgetExceeded:
            # 成本超限：生成部分结果
            return agent.generate_partial_answer()
        
        except Exception as e:
            # 未知异常：记录日志 + 安全降级
            logger.error(f"Unexpected error: {e}", exc_info=True)
            return "抱歉，处理过程中遇到了意外问题。请稍后重试或联系支持。"
```

### 关键设计原则

1. **分级超时**：总超时（5min）+ 单步超时（30s）+ 单工具超时（10s），任何一级超时都触发对应的降级策略
2. **成本预算**：设置 token/金额上限，防止无限循环产生天价账单
3. **Graceful degradation**：即使无法完成全部任务，也要基于已收集的部分信息给出有用的回答
4. **可观测性**：每步记录 action、观察、耗时、token 数，死循环发生时能复现和分析

- 标签: `dead-loop`, `exception-handling`, `loop-detection`, `graceful-degradation`, `agent-reliability`
- 记录于: 2026-06-20

## Q: Agent 执行过程中遇到工具调用失败（如支付接口超时）如何处理？

### 工具调用失败的分类

首先要区分失败的类型——不同类型的处理策略完全不同：

| 失败类型 | 示例 | 是否可重试 | 处理策略 |
|---|---|---|---|
| **暂时性故障** | 网络超时、503、限流 | ✅ 可重试 | 指数退避重试 |
| **输入错误** | 参数格式错误、缺必填字段 | ❌ 重试无用 | 让 LLM 修正参数后重试 |
| **业务拒绝** | 余额不足、权限不够、商品已下架 | ❌ 重试无用 | 告知用户，让 LLM 决策 |
| **永久性故障** | 服务下线、API 废弃 | ❌ 不可恢复 | 降级到替代工具或告知用户 |

### 处理流程

```python
class ToolExecutor:
    def execute(self, tool_name: str, params: dict) -> ToolResult:
        error_context = {
            "tool": tool_name,
            "params": params,
            "attempts": [],
        }
        
        for attempt in range(self.max_retries + 1):
            try:
                result = self._call_tool(tool_name, params, timeout=10)
                return ToolResult(success=True, data=result)
            
            except TimeoutError:
                error_context["attempts"].append({
                    "type": "timeout", "attempt": attempt
                })
                if attempt < self.max_retries:
                    time.sleep(2 ** attempt)  # 指数退避
                    continue
                return ToolResult(
                    success=False,
                    error_type="timeout",
                    message=f"{tool_name} 连续 {attempt+1} 次超时",
                    context=error_context,
                )
            
            except InvalidParamsError as e:
                # 参数错误不重试，直接返回给 LLM 修正
                return ToolResult(
                    success=False,
                    error_type="invalid_params",
                    message=str(e),
                    suggestion="请检查参数格式后重试",
                )
            
            except BusinessError as e:
                # 业务拒绝不重试
                return ToolResult(
                    success=False,
                    error_type="business_rejection",
                    message=str(e),  # "余额不足" / "权限不够"
                )
```

### 错误信息传回 LLM

**关键：把结构化的错误信息传给 LLM，让它自主决策。**

```python
def handle_tool_failure(self, tool_result: ToolResult, messages: list):
    # 将错误作为 tool 角色消息传回（不是简单的错误字符串）
    error_message = {
        "role": "tool",
        "tool_call_id": tool_result.call_id,
        "content": json.dumps({
            "status": "error",
            "error_type": tool_result.error_type,
            "message": tool_result.message,
            "retryable": tool_result.error_type == "timeout",
            "suggestion": tool_result.suggestion,
            "alternatives": self.get_alternative_tools(tool_result.tool),
        }, ensure_ascii=False)
    }
    messages.append(error_message)
    
    # LLM 看到错误后会自主决策：
    # - 修正参数后重试
    # - 换用替代工具
    # - 告知用户无法完成
    # - 跳过该步骤继续其他任务
```

### 支付接口超时的具体案例

```python
# 场景：用户说"帮我支付订单 #12345"

# 第一次调用
Action: pay_order(order_id="12345", amount=299.00)
Observation: {"status": "error", "error_type": "timeout", 
              "message": "支付接口响应超时(10s)", "retryable": true}

# LLM 的推理
Thought: 支付接口超时了，这是暂时性故障，但支付是敏感操作，
         不能贸然重试（可能已经扣款但响应丢失）。
         我应该先查询支付状态，确认是否已扣款。

# 第二次调用（LLM 自主决策查询状态而非直接重试）
Action: query_payment_status(order_id="12345")
Observation: {"status": "success", "payment_status": "unpaid"}

# 确认未扣款后才重试
Thought: 确认未扣款，可以安全重试支付。
Action: pay_order(order_id="12345", amount=299.00)
Observation: {"status": "success", "transaction_id": "TXN_789"}
```

### 工具降级链

```python
TOOL_FALLBACK_CHAIN = {
    "pay_order": [
        "pay_order",           # 首选：直接支付
        "create_payment_link", # 降级1：生成支付链接让用户手动支付
        "notify_support",      # 降级2：通知人工客服处理
    ],
    "search_flights": [
        "search_flights",      # 首选：航班搜索 API
        "web_search_flights",  # 降级1：通过网页搜索
        "suggest_manual",      # 降级2：建议用户自行搜索
    ],
}

def execute_with_fallback(tool_name, params):
    chain = TOOL_FALLBACK_CHAIN.get(tool_name, [tool_name])
    
    for fallback_tool in chain:
        result = execute(fallback_tool, params)
        if result.success:
            return result
        if result.error_type == "business_rejection":
            return result  # 业务拒绝不走降级链
    
    return ToolResult(success=False, message="所有备选方案均失败")
```

### 幂等性保证（防止重复执行）

```python
class IdempotentToolExecutor:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def execute(self, tool_name, params, idempotency_key: str):
        # 检查是否已经成功执行过
        cache_key = f"tool_result:{idempotency_key}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)  # 直接返回之前的结果
        
        # 加分布式锁防止并发重复执行
        lock = self.redis.lock(f"tool_lock:{idempotency_key}", timeout=30)
        if lock.acquire(blocking_timeout=5):
            try:
                result = self._call_tool(tool_name, params)
                # 缓存成功结果
                self.redis.setex(cache_key, 3600, json.dumps(result))
                return result
            finally:
                lock.release()
```

- 标签: `tool-failure`, `retry`, `fallback`, `idempotency`, `error-handling`, `payment`
- 记录于: 2026-06-20

## Q: Agent 决策出错导致数据误删，系统设计上如何防范？

### 防范体系：五层防线

```
用户指令 → [意图确认] → [权限控制] → [沙箱隔离] → [软删除] → [审计回滚]
           第 1 层        第 2 层       第 3 层       第 4 层      第 5 层
           事前防范        事前防范      执行隔离      减轻影响     事后恢复
```

### 第 1 层：高危操作确认（Human-in-the-Loop）

```python
DANGEROUS_OPERATIONS = {
    "delete": {"confirm": True, "describe": "删除数据"},
    "drop_table": {"confirm": True, "describe": "删除数据表"},
    "truncate": {"confirm": True, "describe": "清空数据表"},
    "rm_rf": {"confirm": True, "describe": "递归删除文件"},
    "force_push": {"confirm": True, "describe": "强制推送覆盖远程分支"},
    "revoke_access": {"confirm": True, "describe": "撤销用户权限"},
}

class SafetyGate:
    def check(self, action: str, params: dict) -> bool:
        if action in DANGEROUS_OPERATIONS:
            op = DANGEROUS_OPERATIONS[action]
            # 暂停执行，向用户确认
            confirmed = self.ask_user(
                f"⚠️ Agent 准备执行高危操作: {op['describe']}\n"
                f"操作详情: {action}({params})\n"
                f"确认执行？(yes/no)"
            )
            return confirmed
        return True  # 非高危操作直接放行
```

### 第 2 层：最小权限原则

```python
# Agent 使用受限的数据库账号
AGENT_DB_PERMISSIONS = {
    "read": ["SELECT"],
    "write": ["SELECT", "INSERT", "UPDATE"],
    "admin": ["SELECT", "INSERT", "UPDATE", "DELETE", "DROP"],
}

# 默认只给 write 权限，不给 DELETE 和 DROP
agent_db_user = create_db_user(
    username="agent_worker",
    permissions=AGENT_DB_PERMISSIONS["write"],  # 无 DELETE
)

# 需要删除时，走专用的审批流程
class DeleteProxy:
    def delete(self, table, condition):
        # 记录删除请求
        request_id = self.log_delete_request(table, condition)
        # 发送审批通知
        self.notify_admin(request_id)
        # 等待审批（或超时拒绝）
        approval = self.wait_for_approval(request_id, timeout=300)
        if approval:
            self.execute_delete(table, condition)
```

### 第 3 层：沙箱与隔离

```python
# 文件系统隔离
class SandboxedAgent:
    def __init__(self, workspace_dir):
        self.allowed_paths = [workspace_dir]
    
    def execute_file_operation(self, operation, path):
        # 路径白名单检查
        resolved = os.path.realpath(path)
        if not any(resolved.startswith(p) for p in self.allowed_paths):
            raise PermissionError(f"禁止访问: {path}")
        
        # 进一步限制：即使在白名单内，也不能删除关键文件
        if operation == "delete" and self._is_critical_file(path):
            raise PermissionError(f"关键文件受保护: {path}")

# 数据库操作隔离
class DatabaseSandbox:
    def execute_query(self, sql: str):
        # SQL 注入防护
        if self._detect_injection(sql):
            raise SecurityError("检测到潜在 SQL 注入")
        
        # DDL 语句拦截（CREATE/DROP/ALTER/TRUNCATE）
        if self._is_ddl(sql):
            raise PermissionError("Agent 不允许执行 DDL 语句")
        
        # DELETE/UPDATE 必须带 WHERE 子句
        if self._is_destructive(sql) and "WHERE" not in sql.upper():
            raise PermissionError("DELETE/UPDATE 必须包含 WHERE 条件")
        
        # 影响行数限制
        affected_rows = self._estimate_affected_rows(sql)
        if affected_rows > 100:
            raise PermissionError(f"操作影响 {affected_rows} 行，超过限制(100)")
```

### 第 4 层：软删除（Soft Delete）

```sql
-- 不做物理删除，只标记
UPDATE users SET 
    is_deleted = TRUE, 
    deleted_at = NOW(),
    deleted_by = 'agent_session_abc123'  -- 记录是哪个 Agent 会话删除的
WHERE id = 12345;

-- 保留期内可恢复
UPDATE users SET 
    is_deleted = FALSE, 
    deleted_at = NULL, 
    deleted_by = NULL
WHERE id = 12345;

-- 定时任务在保留期（如 30 天）后才物理删除
DELETE FROM users 
WHERE is_deleted = TRUE AND deleted_at < NOW() - INTERVAL 30 DAY;
```

```python
# 文件系统的"软删除"：移到回收站而非直接删除
class SafeFileOperations:
    def __init__(self, trash_dir="/tmp/agent_trash"):
        self.trash_dir = trash_dir
    
    def safe_delete(self, path: str):
        # 移到回收站，保留原始路径信息
        trash_path = os.path.join(
            self.trash_dir,
            datetime.now().strftime("%Y%m%d_%H%M%S"),
            os.path.basename(path)
        )
        shutil.move(path, trash_path)
        
        # 记录删除日志
        self.log_deletion(original=path, trash=trash_path)
        
        return trash_path  # 返回回收站路径，方便恢复
```

### 第 5 层：审计日志与回滚

```python
class AuditLogger:
    def log_action(self, action: str, params: dict, result: dict,
                   agent_id: str, session_id: str):
        audit_record = {
            "timestamp": datetime.now().isoformat(),
            "agent_id": agent_id,
            "session_id": session_id,
            "action": action,
            "params": params,
            "result": result,
            # 关键：记录操作前的状态快照
            "before_state": self._capture_state(action, params),
        }
        
        # 写入不可变的审计日志（append-only）
        self.audit_store.append(audit_record)
    
    def rollback(self, audit_record_id: str):
        record = self.audit_store.get(audit_record_id)
        # 用 before_state 恢复
        self._restore_state(record["before_state"])
```

### 实际案例：Git 操作的防护

```python
class SafeGitAgent:
    BLOCKED_COMMANDS = [
        "git push --force",
        "git push -f",
        "git reset --hard",
        "git clean -fd",
        "git branch -D",
    ]
    
    def execute_git(self, command: str):
        # 黑名单拦截
        if any(blocked in command for blocked in self.BLOCKED_COMMANDS):
            return self.ask_user_confirmation(command)
        
        # 推送前创建备份 tag
        if "git push" in command:
            branch = self._current_branch()
            self._run(f"git tag backup/{branch}/{int(time.time())} {branch}")
        
        return self._run(command)
```

### 设计原则总结

| 原则 | 做法 | 类比 |
|---|---|---|
| **最小权限** | Agent 默认无删除权限 | 员工默认没有管理员账号 |
| **操作确认** | 高危操作暂停等待人工确认 | ATM 大额转账需二次确认 |
| **软删除** | 标记删除而非物理删除 | 回收站 vs 粉碎文件 |
| **审计追踪** | 每个操作记录 who/what/when/before | 银行交易流水 |
| **影响限制** | 限制单次操作的影响范围 | 数据库事务的行数限制 |
| **快速恢复** | 快照 + 回滚机制 | 数据库备份 + Point-in-time Recovery |

- 标签: `data-safety`, `soft-delete`, `audit-log`, `permission`, `sandbox`, `human-in-the-loop`
- 记录于: 2026-06-20
