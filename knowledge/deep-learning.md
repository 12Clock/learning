# 深度学习

> 涵盖：神经网络基础、Transformer 架构、注意力机制、训练技巧。

## Q: Self-Attention 的实现原理？为什么分 Q、K、V？同一个 token 在不同位置的向量一样吗？

### Self-Attention 的计算流程

**输入**：序列中每个 token 的嵌入向量 X（维度 d_model）

**Step 1：线性投影生成 Q、K、V**

```
Q = X · W_Q    (Query 矩阵)
K = X · W_K    (Key 矩阵)
V = X · W_V    (Value 矩阵)
```

其中 W_Q、W_K、W_V 是可学习的权重矩阵，维度为 `d_model × d_k`。

**Step 2：计算注意力分数**

```
Attention(Q, K, V) = softmax(Q · K^T / √d_k) · V
```

分解开：
1. `Q · K^T`：每个 query 与所有 key 做点积，得到相似度矩阵（n × n）
2. `/ √d_k`：缩放因子，防止点积值过大导致 softmax 进入饱和区（梯度接近 0）
3. `softmax`：将相似度转为概率分布（每行和为 1）
4. `· V`：按注意力权重对 value 加权求和，得到每个位置的输出

**数值示例**：

```
序列: ["I", "love", "cats"]
维度: d_model=4, d_k=3

X = [[0.1, 0.2, 0.3, 0.4],    # "I" 的嵌入
     [0.5, 0.6, 0.7, 0.8],    # "love" 的嵌入
     [0.9, 1.0, 1.1, 1.2]]    # "cats" 的嵌入

Q = X · W_Q  →  3×3 矩阵（每个 token 一个 query 向量）
K = X · W_K  →  3×3 矩阵（每个 token 一个 key 向量）
V = X · W_V  →  3×3 矩阵（每个 token 一个 value 向量）

# 注意力权重矩阵（3×3）
scores = softmax(Q · K^T / √3)
# scores[i][j] = token i 对 token j 的注意力权重
# 例如 scores[2][0] = "cats" 对 "I" 的关注度

output = scores · V  # 每个 token 的输出是所有 V 的加权和
```

### 为什么要分成 Q、K、V 三个向量？

**核心原因：让同一个 token 扮演不同角色。**

如果不分 Q、K、V（即直接用 X 做自注意力）：
```
Attention = softmax(X · X^T / √d) · X
```
这意味着每个 token 用**同一个表示**来"提问"和"被检索"——角色耦合了。

分成三个投影后：

| 向量 | 角色 | 类比 |
|---|---|---|
| **Q（Query）** | "我在找什么？" | 搜索引擎的搜索词 |
| **K（Key）** | "我能提供什么？" | 文档的标题/摘要 |
| **V（Value）** | "我的实际内容" | 文档的正文 |

**例子**："The cat sat on the mat"

当处理 "sat" 这个 token 时：
- `Q_sat`（Query）编码了"sat 需要关注什么？"——可能是主语和地点
- `K_cat`（Key）编码了"cat 作为被关注对象的标识"——名词、主语
- `V_cat`（Value）编码了"cat 的语义内容"——动物、主语

Q 和 K 的点积决定**关注度**，V 提供**关注后获取的内容**。这种分离让模型能学到"用一种方式匹配（Q·K），用另一种方式传递信息（V）"。

**如果只用一个矩阵**：匹配和信息传递用同一个表示，模型的表达能力受限。经验上，去掉 Q/K/V 的分离会导致性能显著下降。

### 同一个 token 在不同位置的向量一样吗？

**初始 Token Embedding 相同，但加入位置编码后不同。**

**Token Embedding**（词表查找）：同一个词不管出现在哪个位置，查表得到的嵌入向量相同。
```
"cat" 在位置 0 的 token embedding == "cat" 在位置 5 的 token embedding
```

**但 Transformer 需要位置信息**——"The cat ate the fish" 和 "The fish ate the cat" 的 token 集合相同，但语义完全不同。解决方案：

**位置编码（Positional Encoding）**：

```
最终输入 = Token Embedding + Positional Encoding
```

| 方案 | 原理 | 使用者 |
|---|---|---|
| **正弦位置编码** | 用不同频率的 sin/cos 函数编码位置 | 原始 Transformer |
| **可学习位置编码** | 位置嵌入作为可训练参数 | GPT、BERT |
| **RoPE（旋转位置编码）** | 通过旋转矩阵编码相对位置 | LLaMA、Qwen |
| **ALiBi** | 在注意力分数上加线性偏置 | BLOOM |

加入位置编码后，同一个 token 在不同位置的表示就不同了。

**更进一步**：经过多层 Transformer 后，同一个词在不同上下文中的向量差异会更大——这正是 Transformer 的核心能力（上下文化表示）。

```
"bank" 在 "river bank" 中   →  经过注意力后，向量偏向"岸边"语义
"bank" 在 "bank account" 中 →  经过注意力后，向量偏向"银行"语义
```

这与 Word2Vec 等静态嵌入形成鲜明对比——静态嵌入中 "bank" 只有一个固定向量。

### Multi-Head Attention

实际使用中，注意力被分成 h 个"头"并行计算：

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) · W_O

head_i = Attention(Q · W_Qi, K · W_Ki, V · W_Vi)
```

**为什么多头**：每个头可以关注不同类型的关系——有的头关注语法依赖，有的关注语义相似，有的关注位置关系。分头后每个头的维度降低（`d_k = d_model / h`），总计算量不变。

- 标签: `self-attention`, `transformer`, `positional-encoding`, `deep-learning`
- 记录于: 2026-06-19
