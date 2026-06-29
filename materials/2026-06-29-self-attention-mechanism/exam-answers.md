# Self-Attention 机制 — 考题答案

---

## 题 1 ⭐ — 选择题

**正确答案：B**

将输入 X 分别乘以三个可学习的权重矩阵 W^Q, W^K, W^V。

**解析：**
- A 错误：如果直接使用输入，attention 将失去灵活性，无法在不同子空间中捕获不同模式。
- B 正确：这是 Self-Attention 的标准做法。三个投影矩阵是模型的可学习参数。
- C 错误：Q 和 K 在某些实现中可能共享（如某些轻量级模型），但标准 Transformer 中是独立的。即使共享，V 也是独立的。
- D 错误：Self-Attention 全程使用线性投影和矩阵乘法，不涉及卷积操作。

---

## 题 2 ⭐ — 选择题

**正确答案：B**

为了防止当 d_k 较大时点积值过大，导致 softmax 梯度消失。

**解析：**
- 当 d_k 较大时（如 64 或 128），Q·K 的点积值可能很大（方差为 d_k）。大值输入 softmax 后输出接近 one-hot，此时 softmax 的雅可比矩阵接近零矩阵，梯度几乎消失。
- 除以 sqrt(d_k) 将方差缩放回 ~1，使 softmax 输出较为平滑，梯度流动正常。
- A 错误：softmax 本身就能保证和为 1，与缩放无关。
- C、D 均不是缩放的主要原因。

---

## 题 3 ⭐⭐ — 选择题

**正确答案：C**

不同 head 在各自的表示子空间中计算 attention，然后拼接结果。

**解析：**
- A 错误：多个 head 是**并行**计算的，不是串行。
- B 错误：每个 head 有自己独立的 W^Q, W^K, W^V 权重矩阵，不共享。
- C 正确：MultiHead(Q,K,V) = Concat(head_1, ..., head_h) · W^O。每个 head 在 d_model/h 维的子空间中独立计算 attention。
- D 错误：head 数量直接影响参数量。更多 head → 更多的 W^Q/W^K/W^V 矩阵（虽然每个更小，但总数不变）。

---

## 题 4 ⭐⭐ — 选择题

**正确答案：B**

防止当前位置关注到未来的位置（"看到答案"）。

**解析：**
- B 正确：Decoder 的 causal mask 是一个下三角矩阵（上三角为 -∞），确保 token i 只能 attend 到位置 ≤ i 的 token。这是自回归生成的必要条件。
- A 错误：Padding mask 是另一种 mask，用于忽略填充 token，不是 causal mask 的功能。
- C 错误：Causal mask 并未减少计算量，依然要计算所有位置的 attention score（只是部分被 mask 掉）。
- D 错误：Causal mask 与泛化能力无直接关系。

---

## 题 5 ⭐⭐ — 简答题

**满分答案要点：**

设 n = 序列长度，d = d_model（模型维度），d_k ≈ d_v ≈ d。

**Step 1: 线性投影** Q = X·W^Q, K = X·W^K, V = X·W^V
- X: (n, d), W: (d, d_k)
- 复杂度：O(n · d · d_k) × 3 ≈ O(n · d²)

**Step 2: 注意力分数** Scores = Q·K^T
- Q: (n, d_k), K^T: (d_k, n)
- 复杂度：O(n² · d_k) ≈ O(n² · d)

**Step 3: 加权求和** Output = softmax(Scores)·V
- softmax: O(n²)，对每行的 n 个元素
- Weights: (n, n)，V: (n, d_v)
- 复杂度：O(n² · d_v) ≈ O(n² · d)

**总体复杂度：O(n² · d)**（Step 2 和 Step 3 主导，因为 n >> d 时平方项压倒线性项）。

**关键洞察：** 当序列长度 n 远大于模型维度 d 时，平方项 O(n²) 是瓶颈。这就是长序列推理慢的根本原因。

---

## 题 6 ⭐⭐ — 简答题

**满分答案要点：**

**不能区分。** 因为 Self-Attention 是**置换等变**的。

具体证明：
如果对输入序列做任意排列 π，则：
- Q' = π(X)·W^Q = π(X·W^Q) = π(Q)
- 同理 K' = π(K), V' = π(V)
- Attention(Q', K', V') = π(Attention(Q, K, V))

即输出的排列与输入的排列一致。

对于 "A loves B" 和 "B loves A"：
- 如果不加位置编码，两个句子只是 token 排列不同。Self-Attention 会给 "loves" 对 "A" 和 "B" 的 attention 权重相同（取决于 embedding 的相似度），无法区分主语和宾语。
- 加上位置编码后，位置 0 的 "A" 和位置 2 的 "A" 有不同的表示，模型才能捕获词序信息。

**常见的位置编码方案：**
- Sinusoidal PE（原始 Transformer）
- Learned PE（BERT, GPT）
- RoPE（LLaMA, GPT-NeoX）— 当前主流，直接编码相对位置到 attention score 中

---

## 题 7 ⭐⭐⭐ — 场景题

**1. 延迟增加的根因：**

Self-Attention 的复杂度是 O(n² · d)，其中 n 为 token 数。当 n 从 1000 增长到 8000 时，Q·K^T 矩阵从 1000² = 1M 个元素增长到 8000² = 64M，增加了 **64 倍**。同时 softmax 和加权求和也在 n² 规模的数据上操作。GPT 架构每一层都有 Masked Self-Attention，12-96 层的模型会将这一开销放大。

**2. 缓解方案：**

**方案 A：KV Cache + 增量推理**
- 原理：推理时每次只生成一个 token，只需要对新增 token 计算 Q（其他 token 的 K、V 已缓存）。
- 效果：将每步 decoder attention 从 O(n² · d) 降为 O(n · d)。
- Trade-off：KV Cache 占用 O(n_layer · n · d_model · 2) 内存（FP16 下约 2 × layers × n × d bytes）。batch 推理时每个样本独立维护 KV Cache，内存压力大。

**方案 B：Flash Attention**
- 原理：利用 GPU 内存层次（HBM → SRAM），通过 tiling 和 recomputation 避免将整个 n×n 的 attention matrix 写入 HBM。
- 效果：降低内存占用从 O(n²) 到 O(n)，显著加速（实际 2-4x）。
- Trade-off：不改变算法复杂度 O(n²)，极端长序列（n > 100K）仍是瓶颈。且需要特定的硬件和 CUDA kernel。

**方案 C：Sliding Window / Sparse Attention**
- 原理：限制每个 token 只能关注周围固定窗口大小的 token（如 window=4096）。
- 效果：复杂度降为 O(n · w)，w 为窗口大小。
- Trade-off：丢失了长距离依赖，某些任务（如长文档理解）会受影响。代表：Mistral 的 Sliding Window Attention。

**3. KV Cache 内存分析：**

KV Cache 内存 = 2 × n_layers × batch_size × seq_len × d_model × dtype_bytes

例如：layers=32, batch=8, seq_len=8000, d_model=4096, FP16:
= 2 × 32 × 8 × 8000 × 4096 × 2 bytes
= 32 GB

当 batch size 从 1 增到 8，KV Cache 线性增长 8 倍。这是大模型推理的主要内存瓶颈之一，催生了 PagedAttention（vLLM）和 GQA（Grouped Query Attention）等优化。
