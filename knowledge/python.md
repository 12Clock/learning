# Python

> 涵盖：Python 语言特性、并发模型、性能优化、常用库。

## Q: Python 有没有真正的多线程？为什么要有 GIL？Lock 和 RLock 有什么区别？

### Python 的多线程现状

Python（CPython 实现）**有多线程**（`threading` 模块），但由于 **GIL（Global Interpreter Lock，全局解释器锁）** 的存在，**多个线程无法同时执行 Python 字节码**。

**表现**：
- **I/O 密集型任务**：多线程有效。线程在等待 I/O（网络请求、文件读写、数据库查询）时会释放 GIL，其他线程可以运行。
- **CPU 密集型任务**：多线程几乎无加速甚至更慢。所有线程争抢同一个 GIL，加上线程切换开销，性能可能比单线程还差。

```python
import threading, time

# I/O 密集型 → 多线程有效
def io_task():
    time.sleep(1)  # sleep 释放 GIL

# CPU 密集型 → 多线程无效
def cpu_task():
    sum(range(10**7))  # 纯 Python 计算不释放 GIL
```

### 为什么要有 GIL

**根本原因：CPython 的引用计数内存管理不是线程安全的。**

CPython 使用引用计数（reference counting）管理对象生命周期。每个对象都有一个 `ob_refcnt` 计数器：

```c
// CPython 内部
typedef struct {
    Py_ssize_t ob_refcnt;  // 引用计数
    PyTypeObject *ob_type;
} PyObject;
```

如果两个线程同时修改同一个对象的 `ob_refcnt`（一个加一个减），会发生竞态条件——计数错误会导致：
- 计数提前归零 → 对象被过早释放 → 悬挂指针 → 段错误
- 计数永不归零 → 内存泄漏

**GIL 的解决方案**：用一把全局锁保护所有 Python 对象的引用计数操作。简单粗暴但有效——保证任何时刻只有一个线程在执行 Python 字节码。

**为什么不去掉 GIL？**

去掉 GIL 需要给每个对象加细粒度锁，这会：
1. **单线程性能退化 10-30%**：每次对象操作都要加锁/解锁
2. **破坏 C 扩展兼容性**：大量 C 扩展（NumPy、pandas）假设了 GIL 的存在
3. **增加代码复杂度**：CPython 核心开发者需要重写大量内部代码

**Python 3.13+ 的 Free-threaded 模式（PEP 703）**：
- 实验性支持无 GIL 的 CPython（`--disable-gil` 编译选项）
- 使用 biased reference counting + 每对象锁替代 GIL
- 单线程性能损失约 5-10%
- 仍处于实验阶段，第三方库兼容性有限

### CPU 密集型的替代方案

| 方案 | 原理 | 适用场景 |
|---|---|---|
| `multiprocessing` | 多进程，每个进程有独立 GIL | 通用 CPU 密集 |
| `concurrent.futures` | 统一 API，支持进程池/线程池 | 简单的并行任务 |
| C 扩展（NumPy 等） | 在 C 层面释放 GIL 后做计算 | 数值计算 |
| `asyncio` | 单线程事件循环 | I/O 密集 + 高并发 |
| Cython / PyPy | 编译 Python 或 JIT | 性能关键路径 |

### Lock vs RLock

**Lock（互斥锁）**

```python
import threading

lock = threading.Lock()

def worker():
    lock.acquire()     # 获取锁
    # 临界区
    lock.release()     # 释放锁

# 或用 context manager
with lock:
    # 临界区
    pass
```

**特性**：
- 同一时刻只有一个线程能持有锁
- **不可重入**：如果持有锁的线程再次 `acquire()`，会**死锁**（自己等自己释放）

```python
lock = threading.Lock()
lock.acquire()
lock.acquire()  # 死锁！当前线程永远等待自己释放锁
```

**RLock（可重入锁，Reentrant Lock）**

```python
rlock = threading.RLock()

def outer():
    with rlock:        # 第一次获取
        inner()        # 内部再次获取同一把锁

def inner():
    with rlock:        # 第二次获取——不会死锁！
        pass
```

**特性**：
- **同一线程可以多次 acquire**，每次 acquire 必须对应一次 release
- 内部维护一个计数器（`_count`）和持有者线程 ID（`_owner`）
- 只有当计数器归零时，锁才真正释放
- 不同线程之间仍然互斥

**对比**：

| 维度 | Lock | RLock |
|---|---|---|
| 可重入 | 否（同线程二次 acquire 死锁） | 是（同线程可多次 acquire） |
| 性能 | 略快（更简单的内部实现） | 略慢（需维护 owner + count） |
| 适用场景 | 简单互斥、没有嵌套调用 | 递归调用、方法互相调用且共享锁 |
| 典型用途 | 保护简单的共享变量 | 保护可能被递归访问的资源 |

**选择建议**：默认用 `Lock`（更简单、更安全——容易发现死锁 bug）；只有当你确实需要在同一线程中嵌套获取锁时才用 `RLock`。如果发现需要 `RLock`，先考虑是否可以重构代码避免嵌套加锁。

- 标签: `python`, `threading`, `GIL`, `lock`, `concurrency`
- 记录于: 2026-06-19
