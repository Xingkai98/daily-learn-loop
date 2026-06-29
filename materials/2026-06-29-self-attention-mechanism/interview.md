# Self-Attention 机制 — 模拟面试

面试风格：对标 Google/ByteDance/OpenAI — 从基础深挖到系统设计，逐层追问。

---

## 问题 1：基础深挖 (L1→L2)

**主问题：** 请用你自己的话解释 Self-Attention 的计算过程，从输入 X 到最终输出。

**追问链：**
- 你说到 softmax，softmax 之前的除以 sqrt(d_k) 是做什么的？如果不除以 sqrt(d_k) 会怎样？
- 你能把 softmax 函数的梯度推导一下吗？在什么条件下梯度会趋近于 0？
- 如果用 max 函数代替 softmax（即 hard attention），会有什么问题？

---

## 问题 2：架构理解 (L2→L3)

**主问题：** 在一个标准的 6 层 Transformer Encoder 中，不同层的 Self-Attention 捕获的模式有何不同？有没有研究支持你的说法？

**追问链：**
- 如果让你设计一个实验来验证"底层关注局部语法、高层关注全局语义"，你会怎么设计？
- Multi-Head Attention 中，你如何判断一个模型需要多少个 head？head 太少或太多分别会导致什么问题？
- 有没有办法在训练后剪枝掉一些不重要的 head？具体怎么做？

---

## 问题 3：系统设计 (L3→L4)

**主问题：** 假设你需要为一个 LLM 推理引擎设计 Attention 模块，目标是在单张 A100 上支持 32K token 的上下文窗口。你的设计决策是什么？

**追问链：**
- Flash Attention 解决了什么问题？它为什么比标准实现快？
- 你提到 Flash Attention 是 exact attention（精确注意力），那为什么不所有地方都用它？有什么局限？
- 如果上下文进一步扩大到 128K，Flash Attention 还能 hold 住吗？如果不能，你会考虑其他什么方案？
- PagedAttention（vLLM） 和 Flash Attention 解决的是不同的问题吗？分别是什么？

---

## 问题 4：对比分析 (L2)

**主问题：** Multi-Head Attention (MHA)、Multi-Query Attention (MQA) 和 Grouped Query Attention (GQA) 有什么区别？

**追问链：**
- 各自的 KV Cache 内存占用是多少？算一下。
- GQA 中 group 的数量如何选择？设为 1 就是 MQA，设为 num_heads 就是 MHA——这个 trade-off 曲线是什么形状的？
- 如果一个模型从 MHA 切换到 GQA，在训练和推理上分别需要做什么调整？对模型质量有影响吗？

---

## 问题 5：交叉注意力 (L2→L3)

**主问题：** Self-Attention 和 Cross-Attention 有什么区别？在 Encoder-Decoder 架构中，Cross-Attention 的 Q、K、V 分别来自哪里？

**追问链：**
- 如果 Encoder 的输出序列长度是 m，Decoder 的序列长度是 n，Cross-Attention 的计算复杂度是多少？
- 这和 Self-Attention 的 O(n² · d) 有什么不同？在什么场景下 Cross-Attention 会成为瓶颈？
- 为什么 GPT 系列只用 Decoder，不需要 Cross-Attention？这种设计选择的原因是什么？

---

## 问题 6：位置编码 (L2)

**主问题：** RoPE（旋转位置编码）是目前最主流的位置编码方案。它是怎么工作的？相比 Sinusoidal PE 有什么优势？

**追问链：**
- RoPE 如何做到只依赖于相对位置？推导一下两个 token 经过 RoPE 后的 attention score 公式。
- RoPE 在长序列外推（extrapolation）上有什么问题？NTK-aware scaling 或 YaRN 是怎么解决这个问题的？
- 如果模型训练时最大长度是 4096，推理时需要支持 16384，你用 RoPE 的话会怎么做？

---

## 问题 7：工程实践 (L3→L4)

**主问题：** 你在项目中部署过一个基于 Transformer 的模型，推理延迟不达标。你从 Self-Attention 的角度做了哪些优化？

**追问链：**
- 你说的 KV Cache 是 prefix caching 还是 exact caching？你考虑了 prefix 匹配的哈希策略吗？
- 你用了什么 attention kernel？是 Flash Attention 还是 xformers 还是手写的 Triton kernel？选择依据是什么？
- Batch 推理时，不同请求的序列长度差异很大。你怎么做 batching？continuous batching 对 attention 计算有什么影响？
- 如果用户量增长 10 倍，你当前的 attention 实现还能撑住吗？瓶颈在哪里？

---

## 问题 8：前沿趋势 (L4)

**主问题：** Self-Attention 的 O(n²) 复杂度一直是瓶颈。除了 Flash Attention（不改变复杂度），有没有真正降低复杂度的替代方案？你怎么看这些方案的未来？

**追问链：**
- Mamba / State Space Models 声称用 O(n) 复杂度达到 Transformer 的效果。它们的核心思想是什么？和 Attention 有什么本质区别？
- Linear Attention 试图用核函数技巧将 O(n²) 降为 O(n)，但为什么一直没有大规模采用？
- 你认为 3-5 年内，Self-Attention 还是 LLM 的主流选择吗？为什么？
