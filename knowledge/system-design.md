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
