# Self-Attention 机制 — 考题

---

## 题 1 ⭐ — 选择题

Self-Attention 中的 Q（Query）、K（Key）、V（Value）三个矩阵是通过什么方式得到的？

A. 直接使用输入 embedding，不做变换
B. 将输入 X 分别乘以三个可学习的权重矩阵 W^Q, W^K, W^V
C. Q 和 K 共享同一个权重矩阵，V 独立
D. 通过 CNN 卷积提取得到

---

## 题 2 ⭐ — 选择题

Scaled Dot-Product Attention 中，为什么要除以 sqrt(d_k)？

A. 为了让 attention weight 的和等于 1
B. 为了防止当 d_k 较大时点积值过大，导致 softmax 梯度消失
C. 为了减少计算量
D. 为了让 Q 和 K 的量纲保持一致

---

## 题 3 ⭐⭐ — 选择题

关于 Multi-Head Attention，以下哪项说法是正确的？

A. 多个 head 串行计算，最后取平均
B. 每个 head 使用相同的 Q、K、V 权重矩阵，共享参数
C. 不同 head 在各自的表示子空间中计算 attention，然后拼接结果
D. head 数量与模型参数量无关

---

## 题 4 ⭐⭐ — 选择题

在 Transformer Decoder 中，Masked Self-Attention 的 causal mask 的作用是：

A. 防止 attention 关注到 padding token
B. 防止当前位置关注到未来的位置（"看到答案"）
C. 加速计算，只计算对角线附近的 attention
D. 提高模型对长序列的泛化能力

---

## 题 5 ⭐⭐ — 简答题

请推导 Self-Attention 的计算复杂度，并解释为什么它是 O(n² · d)。具体说明 n 和 d 分别代表什么，以及三个主要计算步骤各自的复杂度。

---

## 题 6 ⭐⭐ — 简答题

为什么 Self-Attention 需要位置编码（Positional Encoding）？如果不加位置编码，Self-Attention 能区分 "A loves B" 和 "B loves A" 吗？请解释原因。

---

## 题 7 ⭐⭐⭐ — 场景题

你正在部署一个基于 GPT 架构的对话模型到生产环境。用户反馈当对话历史超过 8000 token 时，推理延迟急剧增加。请从 Self-Attention 的机制出发：

1. 分析延迟增加的根因。
2. 提出至少两种缓解方案，并说明各自原理和 trade-off。
3. 如果你的模型使用了 KV Cache，当 batch size 增大时，KV Cache 的内存占用会如何变化？
