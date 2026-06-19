# 评估与优化

> 涵盖：离线评估指标、线上 Badcase 发现、意图粒度平衡。

## Q: 如何离线评估 Agent 的意图识别模块？有哪些指标？

### 评测集构建

```
evaluation/
  intent_test_set.jsonl
  # 每条格式:
  # {"query": "帮我查一下快递到哪了", "expected_intent": "order_query", "tags": ["typical"]}
  # {"query": "这个质量有问题", "expected_intent": "complaint", "tags": ["edge_case"]}
```

**构建原则**：
- 每个意图至少 50 条（典型 30 + 边界 20）
- 包含各种表达变体（正式/口语/方言/错别字）
- 包含跨意图混淆样本（每对易混淆意图至少 10 条）
- 标注者至少 2 人，计算标注一致率（Kappa > 0.8）

### 核心指标

| 指标 | 公式 | 衡量什么 |
|---|---|---|
| **Overall Accuracy** | 正确数 / 总数 | 整体准确度 |
| **Per-Intent Precision** | TP / (TP+FP) per intent | 各意图的误触发率 |
| **Per-Intent Recall** | TP / (TP+FN) per intent | 各意图的漏识别率 |
| **Macro F1** | 各意图 F1 的平均 | 对长尾意图友好 |
| **Weighted F1** | 按样本量加权的 F1 | 反映实际分布下的表现 |

### 高级指标

**1. 混淆矩阵（Confusion Matrix）**

```
              预测
           A    B    C    unknown
真   A  [ 45    3    1    1 ]
实   B  [  2   48    0    0 ]
     C  [  0    1   46    3 ]
```

重点关注非对角线的大值——这些是系统性的混淆模式，优化 Prompt/示例时优先解决。

**2. 分桶评估**

按不同维度切分评测集，发现模型的弱点：
- 按查询长度（短 <5 字 / 中 5-20 字 / 长 >20 字）
- 按表达方式（正式 / 口语 / 含错别字）
- 按意图频率（高频 / 低频 / 新增）
- 按模糊程度（明确 / 模糊 / 多意图）

**3. 覆盖率（Coverage）**

```
coverage = 1 - (被分为 unknown/out_of_scope 的样本数 / 总样本数)
```

低覆盖率意味着模型"不敢分类"——可能置信度阈值设太高，或意图定义太窄。

### 评估流程

```python
def evaluate_intent_model(model, test_set):
    predictions = [model.classify(case.query) for case in test_set]
    
    report = classification_report(
        y_true=[c.expected_intent for c in test_set],
        y_pred=[p.intent for p in predictions],
        output_dict=True
    )
    
    # 混淆矩阵
    cm = confusion_matrix(y_true, y_pred)
    
    # 低置信度分析
    low_conf = [p for p in predictions if p.confidence < 0.7]
    
    return report, cm, low_conf
```

- 标签: `evaluation`, `metrics`, `confusion-matrix`, `intent-recognition`
- 记录于: 2026-06-19

## Q: 线上日志中如何发现意图识别的错误 Badcase？

### 发现方法

**1. 低置信度采样**

```python
# 每小时采样 confidence < 0.7 的 case
low_conf_cases = logs.filter(confidence__lt=0.7).sample(n=100)
# 人工标注后与模型预测对比
```

低置信度 case 是模型"犹豫"的信号，错误率通常是正常 case 的 3-5 倍。

**2. 用户行为反向信号**

即使没有显式反馈，用户行为本身包含丰富的信号：

| 行为信号 | 可能含义 |
|---|---|
| 用户立即重复发送（rephrased） | 上一轮的回答没有满足需求 → 可能意图识别错了 |
| 用户说"不是"、"我是说" | 显式纠正 → 意图确定错了 |
| 对话中途放弃（session drop-off） | 某个环节体验差 → 检查最后几轮 |
| 用户跳转到人工客服 | 自动化失败 → 高优先级 Badcase |
| 同一用户短时间内多次发起新会话 | 之前的会话没解决问题 |

```python
# 提取用户纠正信号
correction_cases = logs.filter(
    next_message__contains=["不是", "我说的是", "我是想", "不对"]
)
```

**3. 路由后行为异常**

```python
# 意图识别为 A，但后续工具调用结果与 A 不匹配
mismatches = logs.filter(
    intent="order_query",
    tool_result__status="no_order_found"
)
# 可能是意图识别对了但实体错了，也可能是意图本身就错了
```

**4. 定期 Canary 注入**

向线上系统定期注入已知答案的 case，监控准确率趋势。如果准确率下降，说明系统可能退化（模型版本更新、数据分布变化等）。

### 系统化流程

```
每日自动:
  1. 采样低置信度 case 100 条
  2. 提取用户纠正信号 case 全量
  3. 检查 Canary 准确率
  
每周人工:
  4. 标注上述 case（2 人标注 + 交叉验证）
  5. 更新 Badcase 库
  6. 分析 Badcase 模式 → 制定优化方案
```

- 标签: `badcase`, `monitoring`, `online-evaluation`, `user-signals`
- 记录于: 2026-06-19

## Q: 意图类别太细容易分错，太粗又不够用，如何平衡？

### 问题本质

这是意图粒度（Granularity）的经典权衡：

```
粗粒度: order_related          → 不够用（查订单、改订单、取消订单都混在一起）
细粒度: query_order_logistics   → 太细（和 query_order_status 很难区分）
```

### 解决方案：层级化意图体系

**两级意图结构**：

```
一级意图（粗，5-10 个）     二级意图（细，每级 3-8 个）
├─ order_manage              ├─ query_status
│                            ├─ modify_order
│                            └─ cancel_order
├─ payment                   ├─ refund
│                            ├─ invoice
│                            └─ payment_issue
├─ product_consult           ├─ spec_query
│                            ├─ compare
│                            └─ recommend
└─ complaint                 ├─ quality_issue
                             ├─ service_issue
                             └─ delivery_issue
```

**执行策略**：

```python
# 方案 A：两步分类
level1 = classify_level1(query)           # 先粗分（准确率高）
level2 = classify_level2(query, level1)   # 再细分（在限定范围内）

# 方案 B：一步分类 + 层级标签
result = classify(query)  # 直接输出 "order_manage.cancel_order"
level1, level2 = result.split(".")
```

方案 A 更稳定（每步搜索空间小），方案 B 更快（一次调用）。

### 粒度设计原则

1. **按后续处理路径决定粒度**：如果两个意图的后续处理逻辑完全不同（不同工具、不同 Agent），就必须区分；如果处理逻辑一样只是参数不同，用同一个意图 + 不同实体即可。

```
# 需要区分（不同处理逻辑）
query_order → 调查询 API
cancel_order → 调取消 API（需要确认）

# 不需要区分（同一处理逻辑，参数不同）
query_order_logistics 和 query_order_status → 都调同一个订单查询 API
```

2. **从粗到细迭代**：先用粗粒度上线，收集线上数据后，发现哪些粗意图内部有明显的分流需求，再拆分。不要一开始就设计 50 个细意图。

3. **可合并回退**：如果某个细意图的识别准确率持续低于 85%，考虑合并回上级意图。

### 数据驱动的调整

```python
# 用混淆矩阵发现应该合并的意图
for (intent_a, intent_b), confusion_rate in confusion_pairs:
    if confusion_rate > 0.15:  # 互相混淆率 >15%
        print(f"考虑合并 {intent_a} 和 {intent_b}")

# 用聚类发现应该拆分的意图
for intent in intents:
    cluster_variance = compute_variance(intent_cases[intent])
    if cluster_variance > threshold:
        print(f"考虑拆分 {intent}，内部差异大")
```

- 标签: `intent-granularity`, `hierarchical-intent`, `taxonomy`, `design`
- 记录于: 2026-06-19
