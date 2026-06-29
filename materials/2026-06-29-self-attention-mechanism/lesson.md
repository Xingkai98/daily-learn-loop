# Self-Attention 机制 — 教案

## 1. 概念定义

**Self-Attention（自注意力）** 是 Transformer 架构的核心算子，它允许输入序列中的每个位置直接与序列中的所有其他位置交互，从而捕获全局依赖关系。

### 直觉类比

想象一个会议室里的人在讨论问题。每个人说一句话（一个 token），但每个人说话时会参考房间里所有人之前说的内容——有的人的话对你更重要（高权重），有的人不太相关（低权重）。Self-Attention 就是在数学上实现这个"每个词看所有词，并决定关注谁"的过程。

### 关键术语

| 术语 | 含义 |
|------|------|
| Query (Q) | "我在找什么"——当前 token 想从其他 token 获取信息的查询向量 |
| Key (K) | "我有什么"——每个 token 提供给其他 token 检索的键向量 |
| Value (V) | "我的内容"——每个 token 实际携带的信息向量 |
| Attention Score | Q 和 K 的点积，衡量两个 token 之间的相关性 |
| Attention Weight | 对 Score 做 softmax 归一化后的权重分布 |
| Scaled Dot-Product | 点积除以 sqrt(d_k) 防止梯度消失 |

## 2. 核心原理

### 2.1 数学定义

给定输入矩阵 X ∈ R^{n × d_model}（n 个 token，每个 d_model 维）：

```
Q = X · W^Q    (W^Q ∈ R^{d_model × d_k})
K = X · W^K    (W^K ∈ R^{d_model × d_k})
V = X · W^V    (W^V ∈ R^{d_model × d_v})

Attention(Q, K, V) = softmax(Q·K^T / sqrt(d_k)) · V
```

### 2.2 计算步骤

**Step 1: 线性投影。** 将输入 X 分别乘以三个可学习的权重矩阵 W^Q, W^K, W^V，得到 Q, K, V。

**Step 2: 计算注意力分数。** 对每个 Query 和所有 Key 做点积：
```
Scores = Q · K^T    # shape: (n, n)
```
Score[i][j] 表示第 i 个 token 对第 j 个 token 的"关注程度"（原始分数）。

**Step 3: 缩放。** 除以 sqrt(d_k)：
```
Scaled_Scores = Scores / sqrt(d_k)
```
为什么要缩放？当 d_k 较大时，点积的值会很大，导致 softmax 进入梯度极小的饱和区。除以 sqrt(d_k) 使方差保持在 1 附近。

**Step 4: Softmax 归一化。**
```
Weights = softmax(Scaled_Scores)    # 沿最后一维归一化
```
每行变成一个概率分布（和为 1），表示该 token 对各个 token 的注意力权重。

**Step 5: 加权求和。**
```
Output = Weights · V    # shape: (n, d_v)
```
每个 token 的输出 = 所有 token 的 Value 按注意力权重加权求和。

### 2.3 Masked Self-Attention

在 decoder 中，为了防止"看到未来"，使用因果遮罩（causal mask）：

```
Mask[i][j] = 0  if i >= j (可以看到)
Mask[i][j] = -∞ if i < j  (不能看到，softmax(-∞) → 0)
```

在 softmax 之前加上 mask：
```
Weights = softmax(Scaled_Scores + Mask)
```

## 3. 关键细节

### 3.1 为什么是 Scaled Dot-Product？

假设 Q 和 K 的每个分量独立，均值为 0，方差为 1。则点积 q·k = Σ q_i·k_i 的方差为 d_k。当 d_k = 64 时，点积可能达到 ±8 的量级（标准差 8），此时 softmax 输出接近 one-hot，梯度几乎为 0。除以 sqrt(d_k) 将方差拉回 1，训练更稳定。

### 3.2 Multi-Head Attention

单头注意力的表达能力有限。Multi-Head 并行运行 h 个独立的 Attention，每个关注不同的表示子空间：

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) · W^O

其中 head_i = Attention(Q·W_i^Q, K·W_i^K, V·W_i^V)
```

每个 head 的维度 d_k = d_v = d_model / h。例如 d_model=512, h=8，则每个 head 是 64 维。

**直觉：** 不同 head 学会关注不同模式——有的关注语法依存，有的关注指代关系，有的关注语义相似性。

### 3.3 计算复杂度

- 矩阵乘法 Q·K^T: O(n² · d_k)
- Softmax: O(n²)
- 加权求和 Weights·V: O(n² · d_v)
- 总体: O(n² · d)，其中 d = d_model

**平方复杂度是 Self-Attention 的主要瓶颈**，这也催生了 Flash Attention、Sparse Attention、Linformer 等优化方案。

### 3.4 位置编码

Self-Attention 本身是**置换等变**的——打乱输入顺序，输出也会同样的方式打乱。这意味着它天然没有序列顺序的概念。因此需要位置编码（Positional Encoding）注入位置信息：

- **Sinusoidal PE**: sin/cos 函数生成确定性编码
- **Learned PE**: 可学习的位置嵌入
- **RoPE**: 旋转位置编码，在 Q 和 K 上施加旋转变换，使注意力分数仅依赖于相对位置（目前主流方案，GPT-NeoX, LLaMA 等使用）

## 4. 实际应用

| 模型 | Self-Attention 使用方式 |
|------|-------------------------|
| BERT | Encoder-only，双向 attention（能看到所有位置） |
| GPT | Decoder-only，causal/masked attention（只能看左边） |
| T5 | Encoder-Decoder，encoder 用双向，decoder 用 causal |
| Vision Transformer (ViT) | 将图像切分成 patch，作为 token 序列输入 |
| AlphaFold | 用于蛋白质结构中氨基酸残基之间的相互作用建模 |

## 5. 常见误区

1. **"Attention 就是加权平均"**——不完全是。Q·K^T 捕获的是 token 间的**交互**，softmax 后的加权求和只是最后一步。真正的表达能力来自 Q、K、V 三个可学习的投影。

2. **"Self-Attention 代替了 RNN 所以更快"**——在训练时确实可并行（因为不依赖上一步的输出），但推理时每一步仍要计算所有历史 token 的 attention（除非用 KV Cache）。对于长序列，平方复杂度反而是劣势。

3. **"head 越多越好"**——不是。head 太多会使每个 head 的维度太小（d_k = d_model / h），表达能力反而下降。需要在 head 数量和各 head 维度间平衡。

4. **"Attention weights 就是可解释性"**——谨慎。Attention weights 只能说明模型"看了"哪里，不能说明模型"为什么"做某个预测。Jain & Wallace (2019) 已经证明了 attention weights 和 feature importance 之间的相关性很弱。

## 6. 延伸阅读

- **Attention Is All You Need** (Vaswani et al., 2017) — 原始论文
- **Flash Attention** (Dao et al., 2022) — IO-aware 精确 attention，O(n²) 内存 → O(n)
- **RoPE** (Su et al., 2023) — 旋转位置编码，相对位置建模
- **The Annotated Transformer** — Harvard NLP 的逐行注释实现
- **Formal Algorithms for Transformers** (Phuong & Hutter, 2022) — Transformer 数学形式化
