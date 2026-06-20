# 系统设计

> 涵盖：系统架构设计、分层设计、技术选型、MVP 方案。

## Q: 如何设计一个带 TUI 界面的交互式视频剪辑工具（MVP 版）？

### 分层架构

```
┌───────────────────────────────────────┐
│  Layer 4: TUI 层（Presentation）       │  终端界面、用户交互、快捷键
├───────────────────────────────────────┤
│  Layer 3: 命令层（Command）            │  用户操作 → 命令对象、Undo/Redo
├───────────────────────────────────────┤
│  Layer 2: 核心引擎（Core Engine）      │  时间线模型、剪辑操作、项目状态
├───────────────────────────────────────┤
│  Layer 1: 媒体后端（Media Backend）    │  FFmpeg/FFprobe 封装、编解码、预览
└───────────────────────────────────────┘
```

### 各层设计

**Layer 1: 媒体后端**

不自己写编解码，封装 FFmpeg 作为子进程调用：

```python
class MediaBackend:
    def probe(self, path: str) -> MediaInfo:
        """用 ffprobe 获取视频元信息（时长、分辨率、编码、帧率）"""
        
    def extract_frame(self, path: str, timestamp: float) -> bytes:
        """提取指定时间点的帧用于预览（输出为图片字节流）"""
        
    def transcode(self, input_path: str, output_path: str, 
                   edits: list[EditOp]) -> subprocess.Popen:
        """将剪辑操作转化为 FFmpeg filter_complex 命令并执行"""
```

关键设计：
- 所有 FFmpeg 调用异步执行（`asyncio.subprocess`），不阻塞 TUI
- 进度通过解析 FFmpeg 的 stderr 输出（`frame=... time=...`）获取
- 预览帧可以缓存到 `/tmp`，LRU 淘汰

**Layer 2: 核心引擎**

以**时间线（Timeline）**为核心数据模型：

```python
@dataclass
class Clip:
    source: str          # 源文件路径
    start: float         # 源文件中的起始时间（秒）
    end: float           # 源文件中的结束时间（秒）
    position: float      # 在时间线上的位置（秒）
    
@dataclass  
class Timeline:
    clips: list[Clip]    # 有序的片段列表
    duration: float      # 总时长（自动计算）

class EditEngine:
    def cut(self, clip_id: int, at: float) -> tuple[Clip, Clip]:
        """在指定时间点将片段一分为二"""
        
    def trim(self, clip_id: int, new_start: float, new_end: float):
        """调整片段的入点/出点"""
        
    def move(self, clip_id: int, new_position: float):
        """移动片段在时间线上的位置"""
        
    def delete(self, clip_id: int):
        """删除片段"""
        
    def merge(self, clip_ids: list[int]) -> Clip:
        """合并相邻片段"""
```

**Layer 3: 命令层**

Command 模式实现 Undo/Redo：

```python
class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...
    @abstractmethod
    def undo(self) -> None: ...

class CutCommand(Command):
    def __init__(self, engine, clip_id, at):
        self.engine = engine
        self.clip_id = clip_id
        self.at = at
        self._original_clip = None  # 保存原始状态用于 undo
    
    def execute(self):
        self._original_clip = self.engine.get_clip(self.clip_id).copy()
        self.engine.cut(self.clip_id, self.at)
    
    def undo(self):
        self.engine.restore_clip(self._original_clip)

class CommandHistory:
    undo_stack: list[Command]
    redo_stack: list[Command]
```

**Layer 4: TUI 层**

推荐使用 **Textual**（Python 现代 TUI 框架）：

```
┌─ 视频剪辑工具 ──────────────────────────────────┐
│ [文件浏览器]              [预览区（ASCII Art）]    │
│  ├─ video1.mp4            ┌──────────────┐       │
│  ├─ video2.mp4            │  Frame @2:30 │       │
│  └─ video3.mp4            └──────────────┘       │
│                                                   │
│ [时间线] ═══════════════════════════════════════  │
│  |▓▓▓▓clip1▓▓▓▓|▓▓clip2▓▓|▓▓▓▓▓clip3▓▓▓▓▓|    │
│  0:00     1:30   2:45        5:00       7:30     │
│              ▲ 播放头                              │
│                                                   │
│ [状态栏] Cut(c) Trim(t) Delete(d) Undo(u) Export(e)│
└──────────────────────────────────────────────────┘
```

Textual 支持：
- 组件化布局（Grid/Dock）
- 异步事件处理（与 asyncio 原生集成）
- 富文本渲染（颜色、进度条、Unicode 字符画时间线）
- 键盘快捷键绑定

### MVP 功能范围

**Phase 1（核心剪辑）**：
- 导入视频文件
- 时间线显示（文本字符画）
- 剪切（Cut at playhead）
- 删除片段
- 导出（拼接所有片段输出为新文件）
- Undo/Redo

**Phase 2（增强）**：
- 片段拖拽/移动
- 入点/出点 Trim
- 帧级预览（ASCII art 或 sixel 终端协议）
- 多轨道

**不做的事（MVP 边界）**：
- 音频独立处理
- 转场/特效
- 实时播放预览
- GPU 加速渲染

### 技术选型

| 组件 | 选型 | 理由 |
|---|---|---|
| TUI 框架 | Textual | 现代 async、组件化、活跃社区 |
| 媒体处理 | FFmpeg（subprocess） | 行业标准、功能最全、避免绑定 |
| 语言 | Python | 开发速度快、Textual 生态、FFmpeg CLI 封装简单 |
| 项目格式 | JSON | 时间线序列化、可读可编辑 |
| 包管理 | uv / poetry | 现代 Python 项目管理 |

- 标签: `system-design`, `tui`, `video-editing`, `architecture`, `mvp`
- 记录于: 2026-06-19

## Q: Redis 在 AI Agent 系统中有哪些应用场景？如何设计缓存策略？

### 六大应用场景

**1. 会话状态存储（Session State）**

Agent 的多轮对话状态（当前意图、已填槽位、对话历史摘要）需要低延迟读写。Redis 的 Hash 结构天然适合：

```python
# 存储会话状态
redis.hset(f"session:{session_id}", mapping={
    "current_intent": "book_flight",
    "slots": json.dumps({"departure": "北京", "destination": None}),
    "turn_count": 3,
    "created_at": "2026-06-20T10:00:00",
})
redis.expire(f"session:{session_id}", 3600)  # 1 小时过期

# 读取状态（<1ms）
state = redis.hgetall(f"session:{session_id}")
```

**2. 语义缓存（Semantic Cache）**

相似的用户问题可以复用之前的 LLM 回答，节省推理成本：

```python
def semantic_cache_lookup(query: str, threshold=0.92):
    query_embedding = embed(query)
    
    # 方案 A：Redis + RediSearch 向量检索
    results = redis.ft("cache_idx").search(
        KNNQuery(query_embedding, k=1)
    )
    if results and results[0].score > threshold:
        return json.loads(results[0].response)  # 命中缓存
    
    # 方案 B：简单哈希缓存（精确匹配）
    cache_key = f"llm_cache:{hashlib.md5(query.encode()).hexdigest()}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    return None  # 未命中，需调用 LLM
```

**3. 速率限制（Rate Limiting）**

保护 LLM API 免受过量请求：

```python
def check_rate_limit(user_id: str, limit=60, window=60) -> bool:
    key = f"rate:{user_id}:{int(time.time()) // window}"
    current = redis.incr(key)
    if current == 1:
        redis.expire(key, window)
    return current <= limit
```

**4. 工具结果缓存**

对确定性工具（天气 API、汇率查询）的结果做短期缓存：

```python
def cached_tool_call(tool_name: str, params: dict, ttl=300):
    cache_key = f"tool:{tool_name}:{hash(frozenset(params.items()))}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    result = execute_tool(tool_name, params)
    redis.setex(cache_key, ttl, json.dumps(result))
    return result
```

**5. 分布式锁与任务调度**

多 Agent 并发执行时，防止重复操作：

```python
# 防止多个 Agent 同时处理同一用户的同一请求
lock = redis.lock(f"agent_lock:{user_id}:{task_id}", timeout=30)
if lock.acquire(blocking=False):
    try:
        result = agent.execute(task)
    finally:
        lock.release()
```

**6. Pub/Sub 实现 Agent 间通信**

多 Agent 架构中的事件驱动通信：

```python
# Agent A 发布事件
redis.publish("agent_events", json.dumps({
    "type": "intent_classified",
    "intent": "complaint",
    "session_id": "abc123",
}))

# Agent B 订阅并响应
pubsub = redis.pubsub()
pubsub.subscribe("agent_events")
for message in pubsub.listen():
    event = json.loads(message["data"])
    if event["type"] == "intent_classified":
        route_to_handler(event)
```

### 缓存策略设计

**分层缓存架构**：

```
请求 → [L1: 精确匹配缓存] → [L2: 语义相似缓存] → [L3: LLM 推理]
        Redis String           Redis Vector         API 调用
        命中率 ~20%             命中率 ~15%           兜底
        延迟 <1ms              延迟 ~5ms             延迟 200-2000ms
```

**TTL 策略**：

| 数据类型 | TTL | 理由 |
|---|---|---|
| 会话状态 | 30min-2h | 对话结束后可释放 |
| LLM 回答缓存 | 1h-24h | 平衡新鲜度与成本 |
| 工具结果（天气） | 5-30min | 数据有时效性 |
| 工具结果（汇率） | 1-5min | 高频变化 |
| 速率限制计数器 | 等于窗口大小 | 窗口结束自动重置 |
| 用户画像 | 24h-7d | 变化慢，缓存价值高 |

**淘汰策略**：设置 `maxmemory-policy allkeys-lru`，内存不足时自动淘汰最久未访问的 key。对关键数据（会话状态）设置 `PERSIST` 防止被淘汰。

- 标签: `redis`, `caching`, `session-state`, `semantic-cache`, `rate-limiting`, `agent-infrastructure`
- 记录于: 2026-06-20

## Q: 消息队列在 AI Agent 系统中的作用？为什么不直接通过数据库通信？

### 消息队列的核心作用

**1. 异步解耦——Agent 各模块独立扩缩容**

```
同步调用（紧耦合）:
  用户请求 → 意图识别 → 工具调用 → 结果生成 → 返回
  任何一步慢/挂掉，整个链路阻塞

异步队列（松耦合）:
  用户请求 → [意图队列] → 意图识别 Worker
                          ↓ 结果写入
                        [工具队列] → 工具执行 Worker
                                     ↓ 结果写入
                                   [生成队列] → 回答生成 Worker → 回调/推送
```

每个 Worker 独立部署、独立扩容。意图识别慢了加 Worker 实例即可，不影响其他模块。

**2. 削峰填谷——应对 LLM 推理的吞吐限制**

```
用户请求速率:   ████████████████████  (峰值 1000 QPS)
LLM 处理能力:   ████████              (最大 200 QPS)

没有队列: 800 请求直接丢失或超时
有队列:   请求暂存队列，Worker 按能力消费，用户等待但不丢失
```

**3. 重试与死信——工具调用失败的容错**

```python
# 工具调用失败 → 消息回到队列 → 自动重试
# 重试 N 次仍失败 → 进入死信队列 → 人工/告警处理

@consumer("tool_execution_queue")
def execute_tool(message):
    try:
        result = call_external_api(message.tool, message.params)
        publish("result_queue", result)
    except RetryableError:
        message.retry(delay=exponential_backoff(message.retry_count))
    except FatalError:
        message.send_to_dead_letter()  # 不可恢复的错误
```

**4. 多 Agent 编排——事件驱动的工作流**

```
Supervisor Agent 发布任务:
  → [research_queue]  → Research Agent  → 发布结果到 [result_queue]
  → [coding_queue]    → Coding Agent    → 发布结果到 [result_queue]
  → [review_queue]    → Review Agent    → 发布结果到 [result_queue]

Supervisor 从 result_queue 收集所有子 Agent 结果后汇总
```

**5. 优先级调度**

```python
# 不同意图的优先级不同
if intent == "emergency":
    publish("agent_queue", message, priority=HIGH)    # 紧急优先处理
elif intent == "complaint":
    publish("agent_queue", message, priority=MEDIUM)
else:
    publish("agent_queue", message, priority=LOW)
```

### 为什么不直接用数据库通信？

| 维度 | 数据库轮询 | 消息队列 |
|---|---|---|
| **实时性** | 轮询间隔决定延迟（通常 >1s） | 推送模式，毫秒级 |
| **数据库压力** | 频繁 `SELECT ... WHERE status='pending'` 造成大量无效查询 | 零数据库压力，队列独立承载 |
| **消费语义** | 需自己实现"恰好消费一次"（加锁、状态更新、竞争） | 队列原生支持 ACK/NACK/Exactly-once |
| **扩展性** | 多个消费者竞争同一张表，锁冲突严重 | 天然支持多消费者组、分区 |
| **顺序保证** | 需额外排序字段 + 间隙锁 | 分区内天然有序 |
| **重试机制** | 自己实现（重试计数、延迟重试、死信） | 内置支持 |
| **背压** | 无法感知消费者处理能力 | 队列积压可触发告警/限流 |

**数据库通信的核心问题**：用数据库做消息通道是在拿**存储系统**当**通信系统**用——职责错配。数据库为"存取数据"优化（B+Tree 索引、ACID 事务），不为"生产-消费"优化。结果是高频轮询浪费 CPU/IO，且随消费者增多，锁竞争使性能非线性下降。

**唯一例外**：如果系统规模很小（QPS <10）且不想引入额外基础设施，用数据库 + 轮询可以快速起步。但一旦规模增长，迁移到消息队列几乎是必然路径。

### AI Agent 场景的队列选型

| 队列 | 特点 | 适用场景 |
|---|---|---|
| **Redis Streams** | 轻量、已有 Redis 时零成本引入 | 中小规模、简单异步 |
| **RabbitMQ** | 灵活路由、优先级队列、成熟 | 复杂路由/优先级调度 |
| **Kafka** | 高吞吐、持久化、回放 | 大规模日志/事件流 |
| **Celery** | Python 生态、开箱即用 | Python Agent 快速集成 |

- 标签: `message-queue`, `async`, `decoupling`, `agent-infrastructure`, `database-vs-queue`
- 记录于: 2026-06-20

## Q: Kafka 在 AI Agent 系统中的使用场景？如何保证消息顺序性？

### Kafka 核心特性回顾

Kafka 与 RabbitMQ/Redis Streams 的本质区别在于它是**分布式提交日志**，而非传统消息队列：

| 特性 | Kafka | 传统消息队列 |
|---|---|---|
| **消息模型** | 持久化日志（追加写入） | 投递后可删除 |
| **消费模式** | Pull（消费者主动拉取） | Push（队列推给消费者） |
| **消息回放** | 支持（任意 offset 重读） | 不支持（消费即删除） |
| **吞吐** | 百万级/秒 | 万级/秒 |
| **消息顺序** | 分区内有序 | 队列内有序 |

### AI Agent 系统中的六大场景

**1. Agent 轨迹日志（最典型场景）**

```python
# 每个 Agent 步骤都写入 Kafka
producer.send("agent_traces", value={
    "session_id": "sess_123",
    "timestamp": "2026-06-20T10:30:00Z",
    "step": 3,
    "thought": "需要查询订单状态",
    "action": "query_order",
    "params": {"order_id": "ORD_456"},
    "observation": {"status": "shipped"},
    "latency_ms": 230,
    "token_count": 450,
})
```

**价值**：
- 全量轨迹可回放——复现任何 Badcase
- 下游消费者可独立分析（成本分析、质量审计、Badcase 挖掘）
- 天然支持多消费者组（同一份日志，质量团队和成本团队各消费一份）

**2. 事件驱动的 Agent 编排**

```python
# 多 Agent 通过 Kafka Topic 协作
TOPICS = {
    "task_assignments": "Supervisor → 工作 Agent 的任务分配",
    "agent_results": "工作 Agent → Supervisor 的结果上报",
    "agent_events": "Agent 间的事件通知（状态变更、依赖完成等）",
}

# Supervisor Agent 发布任务
producer.send("task_assignments", key=b"research", value={
    "task_id": "task_001",
    "type": "research",
    "query": "Redis 缓存策略最佳实践",
    "deadline": "2026-06-20T11:00:00Z",
})

# Research Agent 消费任务
consumer = KafkaConsumer("task_assignments", group_id="research_agents")
for msg in consumer:
    if msg.value["type"] == "research":
        result = research_agent.execute(msg.value)
        producer.send("agent_results", value=result)
```

**3. LLM 调用审计与合规**

```python
# 所有 LLM 调用都经过 Kafka 记录
def audited_llm_call(prompt, model, user_id):
    # 调用前记录
    producer.send("llm_audit", value={
        "event": "request",
        "user_id": user_id,
        "model": model,
        "prompt_hash": hash(prompt),  # 不存原文，保护隐私
        "timestamp": now(),
    })
    
    response = llm.generate(prompt, model=model)
    
    # 调用后记录
    producer.send("llm_audit", value={
        "event": "response",
        "user_id": user_id,
        "tokens_used": response.usage.total_tokens,
        "cost": calculate_cost(response.usage, model),
        "has_tool_calls": bool(response.tool_calls),
    })
    
    return response
```

**4. 实时监控与告警**

```python
# Kafka Streams / Flink 实时处理
# 消费 agent_traces topic，计算实时指标

def process_stream():
    # 滑动窗口统计
    window_stats = {
        "error_rate_5min": errors_last_5min / total_last_5min,
        "avg_latency_1min": sum(latencies_1min) / len(latencies_1min),
        "tool_failure_rate": tool_errors / tool_calls,
    }
    
    if window_stats["error_rate_5min"] > 0.05:
        alert("Agent 错误率超过 5%", window_stats)
    
    if window_stats["avg_latency_1min"] > 5000:
        alert("Agent 平均延迟超过 5 秒", window_stats)
```

**5. 异步工具调用解耦**

```python
# 长耗时工具（如数据分析、文件处理）通过 Kafka 异步执行
producer.send("tool_requests", key=session_id.encode(), value={
    "session_id": session_id,
    "tool": "generate_report",
    "params": {"data_range": "2026-Q1", "format": "pdf"},
})

# 工具 Worker 处理完成后回写
producer.send("tool_results", key=session_id.encode(), value={
    "session_id": session_id,
    "tool": "generate_report",
    "result": {"file_url": "https://..."},
})

# Agent 消费结果，继续对话
```

**6. 训练数据管道**

```python
# 线上 Agent 轨迹 → Kafka → 数据清洗 → 训练集
# 用于 SFT 数据生产

# Consumer: 过滤高质量轨迹用于训练
consumer = KafkaConsumer("agent_traces", group_id="training_pipeline")
for msg in consumer:
    trace = msg.value
    if trace["task_completed"] and trace["user_rating"] >= 4:
        # 高质量轨迹 → 写入训练数据集
        write_to_training_set(trace)
```

### 如何保证消息顺序性

**Kafka 的顺序保证：分区（Partition）内有序，跨分区无序。**

```
Topic: agent_events (3 个分区)

Partition 0: [msg1] [msg4] [msg7]   ← 内部严格有序
Partition 1: [msg2] [msg5] [msg8]   ← 内部严格有序
Partition 2: [msg3] [msg6] [msg9]   ← 内部严格有序

跨分区: msg1 和 msg2 的顺序不保证
```

**关键：用 message key 控制分区分配。**

```python
# 同一 session_id 的消息总是进同一分区 → 保证单会话内有序
producer.send(
    "agent_traces",
    key=session_id.encode(),      # 按 session_id 分区
    value=trace_data,
)

# Kafka 的分区算法: partition = hash(key) % num_partitions
# 同一 key → 同一 hash → 同一分区 → 顺序保证
```

**典型 key 选择**：

| 场景 | 推荐 Key | 保证什么有序 |
|---|---|---|
| Agent 轨迹 | session_id | 同一会话的步骤有序 |
| 用户操作 | user_id | 同一用户的操作有序 |
| 订单处理 | order_id | 同一订单的状态变更有序 |
| 工具调用 | tool_call_id | 同一调用的请求和响应有序 |

**常见顺序性陷阱**：

```python
# ❌ 陷阱 1：消费者组内多实例竞争同一分区
# 一个分区只能被同一消费者组中的一个实例消费
# 如果实例数 > 分区数，多出的实例闲置

# ❌ 陷阱 2：重试导致乱序
# 消息 A 发送失败重试，消息 B 先到达 → A 和 B 乱序
# 解决：设置 max.in.flight.requests.per.connection=1（牺牲吞吐换顺序）
# 或使用幂等生产者：enable.idempotence=true

# ❌ 陷阱 3：分区数变更导致 key 映射变化
# 原来 hash("sess_123") % 3 = 1
# 扩到 6 分区后 hash("sess_123") % 6 = 4
# 同一 session 的新旧消息在不同分区 → 历史数据顺序断裂
# 解决：提前规划足够的分区数，避免运行中扩分区
```

**严格全局有序（极少需要）**：

```python
# 只用 1 个分区 → 全局有序，但吞吐受限于单 Broker
# 仅适用于低吞吐但严格有序的场景（如审计日志）
admin.create_topic("strict_order_events", num_partitions=1)
```

### Kafka vs 其他方案的选择

| 场景 | 推荐 | 理由 |
|---|---|---|
| Agent 轨迹日志 | **Kafka** | 高吞吐 + 持久化 + 多消费者 |
| 简单任务队列 | **Redis Streams** | 低延迟，轻量 |
| 复杂路由/优先级 | **RabbitMQ** | 灵活的 exchange + 优先级队列 |
| Python 快速集成 | **Celery** | 开箱即用 |
| 实时告警 | **Kafka + Flink** | 流处理 + 窗口聚合 |

- 标签: `kafka`, `event-driven`, `message-ordering`, `partitioning`, `agent-infrastructure`, `audit`
- 记录于: 2026-06-20
