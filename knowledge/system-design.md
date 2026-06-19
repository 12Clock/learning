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
