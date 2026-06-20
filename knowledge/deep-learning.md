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

## Q: 多模态大模型的具体结构是什么样的？视觉编码器和语言模型如何衔接？

### 整体架构：三大组件

几乎所有主流多模态大模型（LLaVA、GPT-4V、Qwen-VL、InternVL 等）都遵循相同的三段式架构：

```
图像 → [视觉编码器] → 视觉特征序列 → [投影/适配模块] → 视觉 token → [大语言模型] → 文本输出
         (frozen)        (N×d_v)        (bridge)        (M×d_llm)     (LLM backbone)
```

| 组件 | 作用 | 典型选择 |
|---|---|---|
| **视觉编码器（Vision Encoder）** | 将图像编码为特征向量序列 | CLIP ViT-L/14, EVA-CLIP, SigLIP, InternViT |
| **投影/适配模块（Projector）** | 对齐视觉特征空间与语言特征空间 | Linear, MLP, Q-Former, Perceiver Resampler |
| **大语言模型（LLM Backbone）** | 联合理解视觉+文本 token，生成回答 | LLaMA, Vicuna, Qwen, InternLM |

### 组件一：视觉编码器（Vision Encoder）

#### ViT（Vision Transformer）的工作方式

ViT 是多模态模型最常用的视觉编码器，核心思想是**把图像当作 token 序列来处理**：

```
原始图像 (224×224×3)
    ↓ 切分为 patch
16×16 的 patch，共 (224/16)² = 196 个 patch
    ↓ 线性投影
每个 patch → 一个 d 维向量（如 1024 维）
    ↓ 加上位置编码
196 个带位置信息的向量 + 1 个 [CLS] token
    ↓ 送入 Transformer Encoder（多层 Self-Attention）
输出: 197 个 d 维向量
```

```python
# ViT 核心流程（伪代码）
class ViT:
    def forward(self, image):  # image: [B, 3, 224, 224]
        # 1. Patch Embedding：将图像切分为 patch 并投影
        patches = self.patch_embed(image)  # [B, 196, d_model]
        
        # 2. 加入 [CLS] token 和位置编码
        cls_token = self.cls_token.expand(B, 1, -1)
        x = torch.cat([cls_token, patches], dim=1)  # [B, 197, d_model]
        x = x + self.pos_embed  # [B, 197, d_model]
        
        # 3. 多层 Transformer Encoder
        for block in self.blocks:  # 通常 24 层（ViT-L）
            x = block(x)  # Self-Attention + FFN
        
        return x  # [B, 197, d_model]
```

#### 为什么用 CLIP 预训练的 ViT？

原始 ViT 在 ImageNet 上用分类任务训练，学到的是"分类特征"。CLIP（Contrastive Language-Image Pre-training）用**图文对比学习**训练 ViT，学到的是**图像-语言对齐的语义特征**：

```
CLIP 训练:
  图像 → ViT → 图像嵌入 ─┐
                          ├→ 对比损失（拉近配对图文，推远非配对）
  文本 → Text Encoder → 文本嵌入 ─┘
```

CLIP ViT 的优势：输出的特征本身就处于"图文共享语义空间"，为后续与 LLM 对齐降低了难度。

#### 特征提取位置的选择

```
ViT 层级:
  Layer 1  → 底层特征（边缘、纹理）
  ...
  Layer 12 → 中层特征（部件、局部结构）
  ...
  Layer 24 → 高层特征（全局语义）       ← 大多数方案取这里
  [CLS]    → 全局表示（一个向量）       ← 仅做分类时用
```

多模态模型通常**丢弃 [CLS]，保留所有 patch token**（如 196 个），因为需要空间细粒度信息来回答"图中左上角是什么"这类问题。部分方案取**倒数第二层**（-2 层），因为最后一层对 CLIP 的对比任务过拟合。

### 组件二：投影/适配模块（Projector）——核心衔接

视觉编码器和 LLM 的特征空间维度和语义不对齐——投影模块是**唯一的桥梁**。

#### 方式一：线性投影（Linear Projection）

最简单的方式，LLaVA v1 使用：

```python
# 一个线性层完成维度对齐
class LinearProjector(nn.Module):
    def __init__(self, d_vision, d_llm):
        self.proj = nn.Linear(d_vision, d_llm)
        # 例如 d_vision=1024 (CLIP ViT-L) → d_llm=4096 (LLaMA-7B)
    
    def forward(self, vision_features):
        # vision_features: [B, 196, 1024]
        return self.proj(vision_features)  # [B, 196, 4096]
```

**特点**：参数极少（~4M），训练快，但表达能力有限。视觉 token 数量不变（196 个全部保留），占用 LLM 上下文窗口较多。

#### 方式二：MLP 投影（LLaVA-1.5）

在线性层基础上加深网络：

```python
class MLPProjector(nn.Module):
    def __init__(self, d_vision, d_llm):
        self.mlp = nn.Sequential(
            nn.Linear(d_vision, d_llm),
            nn.GELU(),
            nn.Linear(d_llm, d_llm),
        )
    
    def forward(self, vision_features):
        return self.mlp(vision_features)  # [B, 196, 4096]
```

**效果**：比单线性层显著提升（LLaVA-1.5 论文中报告多个 benchmark 涨 5-10 点），几乎不增加推理延迟。目前最常用的方案。

#### 方式三：Q-Former（BLIP-2）

引入可学习的 query token，通过交叉注意力从视觉特征中提取固定数量的语义表示：

```python
class QFormer(nn.Module):
    def __init__(self, num_query_tokens=32, d_qformer=768, d_vision=1024):
        # 可学习的 query tokens
        self.query_tokens = nn.Parameter(torch.randn(num_query_tokens, d_qformer))
        # 交叉注意力层（query 关注 vision features）
        self.cross_attn_layers = nn.ModuleList([
            CrossAttentionBlock(d_qformer, d_vision) for _ in range(12)
        ])
        self.proj = nn.Linear(d_qformer, d_llm)
    
    def forward(self, vision_features):
        # vision_features: [B, 196, 1024]
        queries = self.query_tokens.expand(B, -1, -1)  # [B, 32, 768]
        
        for layer in self.cross_attn_layers:
            queries = layer(
                q=queries,        # query tokens 做 Q
                kv=vision_features  # 视觉特征做 K、V
            )
        # queries: [B, 32, 768] → 投影到 LLM 维度
        return self.proj(queries)  # [B, 32, 4096]
```

**关键创新**：196 个视觉 token → 32 个 query token（压缩 6 倍），大幅减少 LLM 的输入长度。但引入了额外的 ~100M 参数，训练成本更高。

#### 方式四：Perceiver Resampler（Flamingo）

与 Q-Former 思路相似，也是用交叉注意力压缩视觉 token 数量：

```python
class PerceiverResampler(nn.Module):
    def __init__(self, num_latents=64, d_model=1024):
        self.latent_tokens = nn.Parameter(torch.randn(num_latents, d_model))
        self.cross_attn = CrossAttention(d_model)
        self.self_attn = SelfAttention(d_model)
    
    def forward(self, vision_features):
        x = self.latent_tokens.expand(B, -1, -1)  # [B, 64, 1024]
        x = self.cross_attn(q=x, kv=vision_features)
        x = self.self_attn(x)
        return x  # [B, 64, 1024]
```

#### 方式对比

| 方式 | 视觉 token 数 | 额外参数 | 信息保留 | 代表模型 |
|---|---|---|---|---|
| **Linear** | 不压缩（196） | ~4M | 完整 | LLaVA v1 |
| **MLP** | 不压缩（196） | ~8-16M | 完整 | LLaVA-1.5, LLaVA-NeXT |
| **Q-Former** | 压缩（32-64） | ~100M | 有损 | BLIP-2, InstructBLIP |
| **Perceiver** | 压缩（64-128） | ~50M | 有损 | Flamingo, Otter |
| **C-Abstractor** | 压缩（可选） | ~20M | 可控 | Honeybee |

**趋势**：LLaVA-1.5 的 MLP 方案效果与复杂方案持平甚至更优（在多数 benchmark 上超过 Q-Former），成为当前主流。压缩方案在 token 数量受限或视频（帧数多）场景仍有价值。

### 组件三：LLM 中的视觉 token 注入

投影完成后，视觉 token 与文本 token **拼接**送入 LLM：

```
LLM 输入序列:
[BOS] [SYS_PROMPT...] [IMG_1] [IMG_2] ... [IMG_196] [USER_TEXT...] [ASSISTANT]
 ↑ 文本 token          ↑ 视觉 token（投影后）       ↑ 文本 token
```

```python
def build_multimodal_input(image, text_input, vision_encoder, projector, tokenizer):
    # 1. 视觉特征提取 + 投影
    vision_features = vision_encoder(image)     # [1, 196, 1024]
    vision_tokens = projector(vision_features)  # [1, 196, 4096]
    
    # 2. 文本 token 嵌入
    text_ids = tokenizer(text_input)
    text_embeds = llm.embed_tokens(text_ids)    # [1, L_text, 4096]
    
    # 3. 拼接：视觉 token 插入到 <image> 占位符的位置
    # 最终序列 = [sys_prompt] + [vision_tokens] + [user_text]
    input_embeds = torch.cat([
        sys_embeds,       # 系统提示的 embedding
        vision_tokens,    # 视觉 token（196 个）
        text_embeds       # 用户文本的 embedding
    ], dim=1)
    
    # 4. 送入 LLM 的 decoder（跳过 embedding 层，直接从第一个 Transformer block 开始）
    output = llm(inputs_embeds=input_embeds)
    return output
```

**关键点**：视觉 token 和文本 token 在 LLM 内部**共享同一组 Transformer 层**，通过 Self-Attention 实现跨模态交互。LLM 不区分它们的来源——在它看来，视觉 token 就是一些"特殊词"。

#### 注入位置的变体

```
方案 A（前缀式，LLaVA）:
  [IMG_1..IMG_196] [文本 token...]
  视觉 token 在前，文本在后

方案 B（交错式，Flamingo）:
  [文本...] <image> [IMG_1..IMG_64] </image> [文本...]
  视觉 token 通过 cross-attention 注入（不占用主序列位置）

方案 C（占位符替换式，Qwen-VL）:
  [文本...] <img_start> [IMG_tokens] <img_end> [文本...]
  用特殊 token 标记图像区域
```

Flamingo 的交错式特别值得注意——它不直接将视觉 token 拼入序列，而是在 LLM 的每一层**新增 cross-attention 模块**：

```python
# Flamingo 式：在 LLM 每一层增加 gated cross-attention
class FlamingoLayer(nn.Module):
    def __init__(self, original_layer, d_model):
        self.original_layer = original_layer  # 原始 LLM 层
        self.cross_attn = CrossAttention(d_model)
        self.gate = nn.Parameter(torch.tensor(0.0))  # 初始为 0，逐步学习
    
    def forward(self, x, vision_features=None):
        if vision_features is not None:
            x = x + self.gate.tanh() * self.cross_attn(q=x, kv=vision_features)
        x = self.original_layer(x)  # 原始 self-attention + FFN
        return x
```

**gate 初始化为 0** 保证训练初期 LLM 的行为不变（不会因为引入视觉信号而崩溃），随训练逐步开启跨模态交互。

### 训练策略：两阶段训练

几乎所有方案都采用**两阶段训练**：

```
阶段 1：预训练对齐（Pre-training Alignment）
  - 数据：大量图文对（如 CC3M、LAION 子集，~600K-1M 对）
  - 训练部分：只训练投影模块（Projector）
  - 冻结部分：视觉编码器 ✅ 冻结  |  LLM ✅ 冻结
  - 目标：学会将视觉特征映射到 LLM 的语义空间
  - 任务：图像描述生成（image captioning）
  - 典型时间：~5 小时（8×A100）

阶段 2：指令微调（Instruction Tuning）
  - 数据：多模态指令数据（如 LLaVA-Instruct-150K, ShareGPT4V）
  - 训练部分：投影模块 + LLM 全参数（或 LoRA）
  - 冻结部分：视觉编码器 ✅ 冻结
  - 目标：让模型理解多模态指令并生成高质量回答
  - 任务：VQA、图像推理、OCR、对话
  - 典型时间：~10 小时（8×A100）
```

```python
# 阶段 1：只训练 projector
for param in vision_encoder.parameters():
    param.requires_grad = False
for param in llm.parameters():
    param.requires_grad = False
# projector 参数保持 requires_grad=True

# 阶段 2：解冻 LLM，继续冻结视觉编码器
for param in llm.parameters():
    param.requires_grad = True  # 或用 LoRA
```

**为什么视觉编码器始终冻结**：CLIP ViT 已经有很好的视觉表示能力，微调可能导致灾难性遗忘，且冻结后训练成本大幅降低。

### 高分辨率处理：动态分辨率

早期模型固定 224×224 输入，导致细节丢失。现代方案（LLaVA-NeXT, InternVL2）支持动态分辨率：

```
高分辨率图像 (672×1008)
    ↓ 动态切分
┌────────┬────────┬────────┐
│ crop_1 │ crop_2 │ crop_3 │   每个 crop 336×336
├────────┼────────┼────────┤
│ crop_4 │ crop_5 │ crop_6 │
└────────┴────────┴────────┘
    + 一个全局缩略图 (336×336)
    ↓ 分别送入 ViT
7 × 576 tokens = 4032 个视觉 token
    ↓ 送入 LLM
```

这大幅提升了 OCR、文档理解等需要细粒度信息的任务，但也导致视觉 token 数量膨胀（4000+），对 LLM 上下文窗口压力很大。

### 代表性模型对比

| 模型 | 视觉编码器 | 投影方式 | LLM | 视觉 token 数 |
|---|---|---|---|---|
| **LLaVA-1.5** | CLIP ViT-L/14 @336 | 2 层 MLP | Vicuna-7B/13B | 576 |
| **BLIP-2** | EVA-CLIP ViT-G | Q-Former (32 queries) | FlanT5/OPT | 32 |
| **Flamingo** | NFNet-F6 | Perceiver (64) + Cross-Attn | Chinchilla-80B | 64 |
| **Qwen-VL** | OpenCLIP ViT-G @448 | 单层 Cross-Attn (256) | Qwen-7B | 256 |
| **InternVL2** | InternViT-6B | MLP (pixel shuffle下采样) | InternLM2 | 256-1024 |
| **GPT-4V** | 未公开 | 未公开 | GPT-4 | 未公开 |

### 本质总结

多模态大模型的核心思想其实很简洁：

```
把图像变成一种"外语"，通过翻译器（Projector）翻译成 LLM 能懂的"词"，
然后 LLM 就像处理普通文本一样处理它们——Self-Attention 天然支持跨模态交互。
```

视觉编码器提供"看"的能力，投影模块完成"翻译"，LLM 负责"理解和表达"。三者的设计空间各自独立——可以换更强的 ViT、换更好的投影方式、换更大的 LLM——这种模块化是当前多模态架构的核心优势。

- 标签: `multimodal`, `vision-encoder`, `projector`, `vit`, `clip`, `llava`, `deep-learning`
- 记录于: 2026-06-20
