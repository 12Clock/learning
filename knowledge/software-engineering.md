# 软件工程

> 涵盖：单元测试、代码覆盖率、Mock 技术、代码质量工具等。

## Q: 代码单测的分支覆盖率是怎么统计的？代码插桩的具体原理是什么？

### 覆盖率类型

| 类型 | 衡量什么 | 示例 |
|---|---|---|
| **行覆盖率（Line）** | 每行代码是否被执行 | 10/12 行被执行 = 83% |
| **分支覆盖率（Branch）** | 每个决策点的每个分支是否被走过 | if/else 的 true 和 false 分支都被测试 |
| **条件覆盖率（Condition）** | 复合条件中每个子条件的 true/false 是否都被覆盖 | `a && b` 中 a 和 b 分别取过 true 和 false |
| **路径覆盖率（Path）** | 所有可能的执行路径是否被覆盖 | 理论最强但路径数指数增长，实际不可行 |

**分支覆盖率 > 行覆盖率**：行覆盖率无法发现只测了 if-true 没测 if-false 的情况。

### 分支覆盖率的统计方法

每个"决策点"被识别并跟踪其所有分支的执行情况：

```python
# 源代码
def check(x, y):
    if x > 0:        # 决策点 1：2 个分支（true / false）
        result = "positive"
    else:
        result = "non-positive"
    
    if y != 0:        # 决策点 2：2 个分支
        result += " and non-zero"
    
    return result
```

```
分支覆盖率 = 已执行的分支数 / 总分支数
- 决策点 1：true ✓, false ✓  → 2/2
- 决策点 2：true ✓, false ✗  → 1/2
- 总覆盖率：3/4 = 75%
```

**决策点类型**：
- `if/else`、`elif`
- `switch/case`（每个 case 是一个分支）
- 三元运算符 `a ? b : c`
- 逻辑短路 `a && b`、`a || b`（短路本身产生隐式分支）
- 循环 `for/while`（进入循环 vs 跳过循环）
- 异常处理 `try/catch`（正常路径 vs 异常路径）

### 代码插桩原理

**核心思想**：在代码的关键位置插入"探针"（计数器），运行时记录哪些位置被执行了。

**源代码级插桩（Source Instrumentation）**

在编译/解释前，对源代码做自动变换，插入计数代码：

```python
# 原始代码
def check(x):
    if x > 0:
        return "positive"
    return "non-positive"

# 插桩后的代码（工具自动生成）
_branch_counter = {}

def check(x):
    _branch_counter['check:1'] = True            # 函数入口
    if x > 0:
        _branch_counter['check:1:true'] = True    # if-true 分支
        return "positive"
    else:
        _branch_counter['check:1:false'] = True   # if-false 分支
    return "non-positive"
```

测试运行结束后，读取 `_branch_counter` 即可知道哪些分支被执行。

**字节码/AST 级插桩**

不修改源代码，而是在 AST（抽象语法树）或字节码层面注入探针：

```
源代码 → AST 解析 → 遍历 AST 找到所有决策节点 
       → 在每个分支入口插入计数器节点 → 重新编译执行
```

Python 的 `coverage.py` 就是用这种方式——利用 `sys.settrace()` 在解释器层面追踪每行执行，或通过 AST 重写插入探针。

**二进制插桩**

直接修改编译后的二进制文件或在运行时动态插入探针：
- **静态二进制插桩**：修改 ELF/PE 文件，在目标位置插入跳转指令
- **动态二进制插桩**：运行时通过 PIN、DynamoRIO 等工具拦截指令执行
- 用于无源码场景（如第三方库、系统内核）

### 各语言工具

| 语言 | 工具 | 插桩方式 |
|---|---|---|
| Python | `coverage.py` | `sys.settrace` + AST |
| JavaScript | Istanbul / `nyc` | AST 源码变换 |
| Go | `go test -cover` | 编译时源码插桩 |
| Java | JaCoCo | 字节码插桩（agent 模式） |
| C/C++ | gcov / lcov | 编译时插桩（`-fprofile-arcs`） |
| Rust | `cargo-tarpaulin` | 编译时插桩 |

### 实践建议

- 分支覆盖率 **80%** 是合理的工程目标；追求 100% 往往投入产出比极低
- 关注**增量覆盖率**——新代码的覆盖率比全量覆盖率更有意义
- 覆盖率高不等于测试质量高——100% 覆盖但断言不够的测试比 70% 覆盖但断言充分的测试差

- 标签: `unit-test`, `code-coverage`, `branch-coverage`, `instrumentation`
- 记录于: 2026-06-19

## Q: 单元测试中的 Mock 是怎么实现的？AST 和 LSP 都难以生成单测的代码怎么处理？

### Mock 的实现原理

Mock 的本质是**用可控的替身对象替换真实依赖**，使测试仅验证被测代码的逻辑，不受外部系统影响。

**Python 中的 Mock 实现（`unittest.mock`）**

```python
from unittest.mock import patch, MagicMock

# 方式 1：patch 装饰器——替换模块中的对象
@patch('myapp.services.external_api.call')
def test_process(mock_call):
    mock_call.return_value = {"status": "ok"}  # 控制返回值
    result = process_data()
    mock_call.assert_called_once_with(expected_args)  # 验证调用

# 方式 2：MagicMock——创建万能替身
mock_db = MagicMock()
mock_db.query.return_value = [{"id": 1, "name": "test"}]
mock_db.query.side_effect = Exception("timeout")  # 模拟异常
```

**Mock 的底层机制**：

1. **Monkey Patching**：Python 的 `patch` 本质上是运行时替换对象属性。`patch('module.ClassName')` 在测试期间将 `module.ClassName` 指向一个 Mock 对象，测试结束后恢复原值。

2. **`__getattr__` 魔法方法**：`MagicMock` 通过 `__getattr__` 拦截所有属性访问，自动返回新的 Mock 对象。这意味着 `mock.any_attr.any_method()` 不会报错——所有调用都被记录。

3. **调用记录**：每次调用 Mock 对象时，参数被记录到 `call_args_list`，可用于后续断言验证。

**其他语言的 Mock 实现**：

| 语言 | 框架 | 机制 |
|---|---|---|
| Java | Mockito | 运行时通过 CGLIB/ByteBuddy 生成子类代理 |
| Go | gomock | 接口（interface）+ 代码生成 |
| JavaScript | Jest | 模块系统劫持（`jest.mock()`） |
| C++ | Google Mock | 模板 + 虚函数重写 |

### AST/LSP 难以生成单测的代码类型

**1. 高度耦合的代码**

```python
def process_order(order_id):
    db = DatabaseConnection()                    # 直接实例化
    user = db.get_user(order_id)                 # 链式依赖
    payment = PaymentGateway.charge(user.card)   # 静态调用
    EmailService().send(user.email, payment)     # 又一个直接实例化
    return payment.status
```

问题：依赖全部硬编码在函数内部，Mock 需要 patch 多个不同模块的对象。AST 分析可以找到这些依赖，但无法自动判断 patch 路径。

**解决**：重构为依赖注入：
```python
def process_order(order_id, db, payment_gw, email_svc):
    user = db.get_user(order_id)
    payment = payment_gw.charge(user.card)
    email_svc.send(user.email, payment)
    return payment.status
```

**2. 复杂状态机/多步交互**

```python
class Workflow:
    def run(self):
        self.step1()
        if self.state == "retry":
            self.step1()      # 状态驱动的循环
        self.step2()
        if self.step2_result.needs_approval:
            self.wait_for_approval()  # 阻塞等待外部事件
            self.step3()
```

问题：测试需要模拟复杂的状态转换序列，AST 无法推断有效的状态组合。

**解决**：
- 每个 step 独立测试
- 用状态图驱动测试用例生成（明确列出所有状态转换）
- Property-based testing（如 Hypothesis）自动生成状态序列

**3. 动态调度和反射**

```python
handler = getattr(self, f"handle_{event_type}")  # 动态方法查找
handler(data)
```

问题：静态分析无法确定 `handler` 指向哪个方法。

**解决**：
- 建立 event_type → handler 的映射表，基于映射表生成测试
- 或用集成测试覆盖完整的事件处理流程

**4. 外部系统交互密集的代码**

数据库查询、文件系统操作、网络请求交织在业务逻辑中，且行为依赖外部状态。

**解决**：
- 使用 **Repository 模式** 抽象数据访问层
- 用 **testcontainers** 起真实的临时数据库（PostgreSQL/Redis 容器）
- 对文件系统操作用 `tmp_path` fixture 或内存文件系统

### 通用应对策略

1. **重构优先**：如果代码难以测试，先重构使其可测试，而非硬写复杂的 Mock
2. **集成测试兜底**：当单元测试成本过高时，用集成测试覆盖关键路径
3. **Contract Testing**：测试接口契约而非实现细节，减少 Mock 的脆弱性
4. **Property-based Testing**：用 Hypothesis/QuickCheck 自动发现边界情况

- 标签: `mock`, `unit-test`, `dependency-injection`, `testing-strategy`
- 记录于: 2026-06-19

## Q: MySQL 索引在什么情况下会失效？LIKE 模糊查询时索引什么情况下失效？

### 索引失效的常见场景

#### 1. 对索引列使用函数或表达式

```sql
-- ❌ 索引失效：对 created_at 使用了函数
SELECT * FROM orders WHERE YEAR(created_at) = 2026;

-- ✅ 改写：范围查询，索引生效
SELECT * FROM orders 
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
```

```sql
-- ❌ 索引失效：对列做运算
SELECT * FROM orders WHERE price + 10 > 100;

-- ✅ 改写：把运算移到值一侧
SELECT * FROM orders WHERE price > 90;
```

**原因**：B+Tree 索引存储的是列的原始值。对列施加函数后，MySQL 无法直接在索引中查找变换后的值，只能全表扫描。

#### 2. 隐式类型转换

```sql
-- phone 是 VARCHAR 类型
-- ❌ 索引失效：传入数字，MySQL 做隐式转换
SELECT * FROM users WHERE phone = 13800138000;

-- ✅ 索引生效：传入字符串
SELECT * FROM users WHERE phone = '13800138000';
```

**原因**：MySQL 将 VARCHAR 列的每个值转为数字来比较，等价于对列施加了 `CAST()` 函数。但反过来，如果列是 INT 而传入字符串，MySQL 会转换值而非列，索引仍然生效。

#### 3. 联合索引不满足最左前缀

```sql
-- 联合索引 idx(a, b, c)
SELECT * FROM t WHERE a = 1 AND b = 2 AND c = 3;  -- ✅ 全部命中
SELECT * FROM t WHERE a = 1 AND b = 2;              -- ✅ 命中 a, b
SELECT * FROM t WHERE a = 1;                         -- ✅ 命中 a
SELECT * FROM t WHERE b = 2 AND c = 3;              -- ❌ 跳过 a，索引失效
SELECT * FROM t WHERE a = 1 AND c = 3;              -- ⚠️ 只命中 a，c 无法用索引
```

**原因**：B+Tree 按 (a, b, c) 的顺序排列。不知道 a 的值就无法定位 b 的范围，就像字典中不知道首字母就无法翻到对应页。

#### 4. OR 条件中包含未索引列

```sql
-- a 有索引，b 无索引
-- ❌ 索引失效：OR 要求两侧都能用索引
SELECT * FROM t WHERE a = 1 OR b = 2;

-- ✅ 如果 a 和 b 都有索引 → MySQL 可能用 index_merge
```

#### 5. NOT、!=、NOT IN

```sql
-- ❌ 通常导致索引失效（优化器认为扫描成本更低）
SELECT * FROM users WHERE status != 'active';
SELECT * FROM users WHERE id NOT IN (1, 2, 3);

-- ✅ 如果排除的范围很小，改为 IN
SELECT * FROM users WHERE status IN ('inactive', 'banned', 'deleted');
```

**注意**：这不是绝对的。如果 `!=` 排除的数据量很小（如 99% 的数据都是 'active'），优化器可能仍然选择全表扫描，因为回表成本太高。

#### 6. 范围查询后的列不走索引

```sql
-- 联合索引 idx(a, b, c)
SELECT * FROM t WHERE a = 1 AND b > 10 AND c = 3;
-- a: 等值查询 ✅ 索引命中
-- b: 范围查询 ✅ 索引命中
-- c: b 之后的列 ❌ 索引不生效（B+Tree 在范围查询后无法继续有序查找）
```

#### 7. SELECT * 导致回表代价过高

```sql
-- 即使有索引，优化器可能因为回表成本太高而选择全表扫描
-- ❌ 需要回表取所有列
SELECT * FROM orders WHERE status = 'pending';

-- ✅ 覆盖索引（不需要回表）
SELECT id, status FROM orders WHERE status = 'pending';
```

### LIKE 模糊查询与索引

**核心规则：前缀匹配走索引，前导通配符不走索引。**

```sql
-- ✅ 索引生效：前缀确定，可以在 B+Tree 中定位范围
SELECT * FROM users WHERE name LIKE '张%';
-- 等价于 name >= '张' AND name < '张\xff'，B+Tree 范围扫描

-- ❌ 索引失效：前导通配符，不知道从哪里开始查
SELECT * FROM users WHERE name LIKE '%三';

-- ❌ 索引失效：两端通配符
SELECT * FROM users WHERE name LIKE '%张三%';
```

**原因**：B+Tree 按字典序排列。`LIKE '张%'` 能确定扫描的起止范围（所有以"张"开头的值连续存放），但 `LIKE '%三'` 无法确定任何范围——以"三"结尾的值散布在整棵树中。

**需要前导通配符时的替代方案**：

```sql
-- 方案 1：全文索引（FULLTEXT INDEX）
ALTER TABLE articles ADD FULLTEXT INDEX ft_content(content);
SELECT * FROM articles WHERE MATCH(content) AGAINST('关键词');

-- 方案 2：倒排索引（搜索引擎）
-- 将数据同步到 Elasticsearch，用倒排索引做全文检索

-- 方案 3：冗余反转列（适合后缀匹配）
ALTER TABLE users ADD COLUMN name_reversed VARCHAR(50);
UPDATE users SET name_reversed = REVERSE(name);
CREATE INDEX idx_name_rev ON users(name_reversed);
-- LIKE '%三' 变成 LIKE '三%' 的反转查询
SELECT * FROM users WHERE name_reversed LIKE REVERSE('%三');
```

### 如何验证索引是否生效

```sql
EXPLAIN SELECT * FROM users WHERE name LIKE '张%';
-- 关注以下字段:
-- type:   ref/range（用了索引）vs ALL（全表扫描）
-- key:    实际使用的索引名（NULL 表示没用索引）
-- rows:   预估扫描行数
-- Extra:  Using index（覆盖索引）/ Using where（需要回表过滤）
```

- 标签: `mysql`, `index`, `query-optimization`, `like`, `b-plus-tree`
- 记录于: 2026-06-20
