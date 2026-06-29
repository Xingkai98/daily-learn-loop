# Self-Attention 机制 — 模拟面试参考答案

---

## 问题 1：Self-Attention 计算过程

**评分要点：**
- 能否清晰描述 Q/K/V 三个投影的来源和意义
- 能否准确解释缩放因子的数学动机
- 对 softmax 梯度的理解程度

**满分回答示范：**

Self-Attention 的计算分五步走——

首先，输入是一个序列 X ∈ R^{n×d}，n 个 token，每个 d 维。我们分别乘以三个可学习的权重矩阵 W^Q, W^K, W^V，投影得到 Query、Key、Value。Query 表示"我在找什么"，Key 表示"我有什么标签"，Value 表示"我实际携带的信息"。

第二步，用 Q 和 K 的转置做点积，得到一个 n×n 的注意力分数矩阵。Score[i][j] 表示第 i 个 token 对第 j 个 token 的原始关注度。

第三步，除以 sqrt(d_k) 做缩放。这里 d_k 是每个 head 的维度。为什么要缩放？因为点积的方差是 d_k——假设 Q 和 K 的每个分量独立，均值为 0 方差为 1。如果 d_k=64，方差就是 64，标准差 8。那点积值可能在 ±20 甚至更大的范围。这么大的值进 softmax 会产生近乎 one-hot 的输出，梯度几乎为 0。除以 sqrt(d_k) 把方差压回 1，让 softmax 输出平滑，梯度正常。

第四步，对缩放后的分数做 softmax，按行归一化成概率分布。

第五步，用这个概率分布对 V 做加权求和。每个 token 的输出是所有 token 的 Value 的加权组合，权重就是注意力概率。

**补充——softmax 梯度推导：**
softmax 的雅可比矩阵：∂y_i/∂z_j = y_i(δ_{ij} - y_j)。当某个 y_k ≈ 1 而其他 y ≈ 0 时，所有偏导数都趋近于 0，这就是梯度消失的发生条件。

**补充——hard attention：**
如果用 max 代替 softmax（hard attention），问题有两个：一是不可导，无法端到端训练；二是丢失了所有其他 token 的信息（稀疏梯度变为只有一个位置有信号）。实际上，hard attention 通常需要用 REINFORCE 等强化学习方法训练，收敛很慢。

**常见错误：**
- 说 softmax 是为了"归一化"但不提梯度——深度不够
- 把 softmax 说成 sigmoid——概念混淆
- 无法推导 softmax 梯度——数学基础薄弱

---

## 问题 2：不同层的 Attention 模式

**评分要点：**
- 是否了解不同层的关注模式差异
- 能否提出可操作的实验设计
- 对 head 剪枝有实际经验或深入理解

**满分回答示范：**

确实有研究证明不同层的 attention 模式不同。BERTology 的一系列工作（如 Clark et al. 2019, "What Does BERT Look At?"）发现：

- **底层（1-3 层）：** Attention 模式非常分散，接近均匀分布。主要在学习局部的词法特征——比如一个词的周围邻居。
- **中层（4-8 层）：** 开始出现语法结构相关的 attention。某些 head 专门关注依存关系——比如动词的 head 会关注它的主语和宾语。
- **高层（9-12 层）：** Attention 变得稀疏且专注，关注对任务最关键的几个 token。比如 [CLS] token 的 attention 可能集中在情感的极性词上。

**实验设计：**
我会做 probing 实验。用一个简单的线性分类器，以各层 attention 输出为输入，分别预测 POS tag（词性标注）、dependency relation（依存关系）、semantic role（语义角色）。如果底层对 POS tag 的 probing 准确率高，说明底层保留了词法信息；如果高层对 semantic role 准确率高，说明高层涌现了语义信息。

**Head 剪枝：**
Michel et al. (2019) 发现剪掉大部分 head 对性能影响很小。具体做法：对每个 head，计算保留它和去掉它之间的 loss 差异（importance score），去掉重要性最低的 head，然后 fine-tune 几步恢复性能。实验显示 BERT 的 144 个 head 中，剪掉 50% 几乎不影响 MNLI 分数，剪掉 80% 才开始显著下降。

**常见错误：**
- 说"底层关注全局、高层关注局部"——说反了
- 不知道 probing 或表示不知道如何验证——研究能力不足
- 认为剪枝后不需要 fine-tune——工程理解不深

---

## 问题 3：32K 上下文 Attention 设计

**评分要点：**
- 理解 Flash Attention 的 IO 优化原理
- 能区分不同优化方案的适用场景
- 对长上下文的工程挑战有整体认知

**满分回答示范：**

我的设计决策是这样的：

**第一选择：Flash Attention 2/3。**
Flash Attention 的核心创新是 IO-aware 计算。标准 attention 的问题不是算力，是带宽——n×n 的 attention matrix 写入 HBM（显存）再读出来，读写开销远超计算。Flash Attention 用 tiling 把矩阵分块，每块在 SRAM（on-chip）里完成 QK^T、softmax、加权求和全流程，只把最终结果写回 HBM。Flash Attention 2 进一步优化了前向/反向的 work partitioning，把非矩阵乘法操作的 overhead 降到最低。

为什么比标准实现快？标准实现：QK^T → 写 n×n 到 HBM → 读回来做 softmax → 写回去 → 读回来乘 V。三次 HBM 读写 n×n 矩阵。Flash Attention：分块在 SRAM 内一次走完，HBM 只读 Q、K、V，只写 Output。

**局限：** Flash Attention 是 exact attention（不改变 math 结果），所以它不降低 O(n²) 的算法复杂度。32K 下大约是 1B 对 token 交互，还行；128K 下是 16B，即使有 tiling 也扛不住，因为总计算量摆在那里。

**对于 128K：** 我会考虑 Ring Attention（分布到多 GPU 上）或者 Sparse Attention（如 Longformer 的 local + global pattern）。也可以用层次化方案，先把长序列分成 chunk，chunk 内做 full attention，chunk 间用一个 summary token 交互。

**PagedAttention vs Flash Attention 的区别：**
Flash Attention 解决的是 attention **计算**的效率（更快地算 attention）。PagedAttention（vLLM）解决的是 KV Cache **内存管理**的效率。它把 KV Cache 从连续内存变为分页管理（类似操作系统的虚拟内存），避免 fragmentation，提升吞吐量。两者是正交的优化，可以同时使用。

**常见错误：**
- 说 Flash Attention 降低了时间复杂度——它不降低，只是降低常数
- 混淆 PagedAttention 和 Flash Attention——两个不同的优化方向
- 不提 bandwidth vs compute bound 的分析——缺少 profiling 意识

---

## 问题 4：MHA vs MQA vs GQA

**评分要点：**
- 准确描述三者的结构和数学差异
- 能计算 KV Cache 内存占用
- 理解 trade-off 曲线

**满分回答示范：**

三者的区别在于 Key 和 Value 的 head 数量：

- **MHA（Multi-Head Attention）：** 每个 head 有独立的 K 和 V。h 个 head 就有 h 组 K、V。
- **MQA（Multi-Query Attention）：** 所有 head 共享一组 K 和 V。只有 Q 有多头。
- **GQA（Grouped Query Attention）：** 折中方案。将 Q 的 head 分成 g 个组，每个组共享一组 K 和 V。

**KV Cache 内存（每个 token）：**
- MHA: 2 × h × d_head × dtype_bytes = 2 × d_model × dtype_bytes
- MQA: 2 × 1 × d_head × dtype_bytes = 2 × d_model/h × dtype_bytes
- GQA: 2 × g × d_head × dtype_bytes

以 LLaMA 2 70B 为例：h=64, d_model=8192, d_head=128, FP16。
- MHA: 2 × 64 × 128 × 2 = 32 KB/token
- MQA: 2 × 1 × 128 × 2 = 0.5 KB/token（缩减 64×）
- GQA(g=8): 2 × 8 × 128 × 2 = 4 KB/token（缩减 8×）

**Trade-off 曲线：**
g 从 1 到 h 时，KV Cache 线性增长，但模型质量不是线性的。LLaMA 2 的实验显示 g=8 时质量与 MHA 几乎持平，但 KV Cache 缩减了 8 倍。这暗示 attention 的多个 head 对 K-V 的信息需求有大量共享。

**训练/推理调整：**
- MHA 训练时如果需要切换成 GQA 推理，可以在 fine-tune 阶段引入 GQA 结构（或者从头训练）。直接将 MHA 的 K、V head 平均池化到 GQA 通常会导致质量显著下降。
- 最佳实践是从预训练阶段就用 GQA（如 LLaMA 2、Mistral）。

**常见错误：**
- 认为 MQA 和 MHA 的模型参数量一样——其实 MQA 少了很多 K、V 参数
- 混淆 KV Cache 大小和计算量——内存和计算是两个维度

---

## 问题 5：Self-Attention vs Cross-Attention

**评分要点：**
- 清晰区分两种 attention 的数据流
- 能正确计算复杂度
- 理解 Decoder-only 架构的设计选择

**满分回答示范：**

**区别：**
- Self-Attention：Q、K、V 来自**同一个输入序列**。每个 token 关注同一序列中的所有 token。
- Cross-Attention：Q 来自**Decoder 的当前状态**，K 和 V 来自**Encoder 的输出**。Decoder 的每个位置去"查询"Encoder 编码好的源序列信息。

在 Encoder-Decoder 架构（如 T5）中，Cross-Attention 在每一层 decoder self-attention 之后，让 decoder 能够访问完整的输入序列信息。

**复杂度：**
设 Encoder 输出长度 m，Decoder 序列长度 n：
- Cross-Attention: Q(n×d) · K^T(d×m) → O(n·m·d)
- 而 Self-Attention 在 Decoder 中是 O(n²·d)
- 当 m >> n 时（如翻译任务中源语言很长但目标语言在逐步生成），Cross-Attention 可能成为瓶颈。

**为什么 GPT 只用 Decoder？**
这是设计哲学的选择——Decoder-only 模型把所有信息（输入+输出）都放在同一个序列中，通过 causal self-attention 统一处理。优势是简洁、扩展性好（scaling law 友好）。代价是：
- 输入需要和输出一起处理（没有单独的 encoder 做深度编码）
- 长输入时 self-attention 的 O(n²) 包含输入部分
- Prompt 越长，每个生成 step 的 attention 计算量越大

OpenAI 的选择是"大道至简"——用统一的 causal attention + 足够的 scale 来弥补没有 encoder 的损失。

**常见错误：**
- 说 Cross-Attention 的 Q 来自 Encoder——说反了，Q 来自 Decoder
- 不知道 Decoder-only 的历史原因——GPT 系列的 scaling law 经验表明这个设计更强

---

## 问题 6：RoPE 位置编码

**评分要点：**
- 理解 RoPE 的旋转操作原理
- 能推导相对位置性质
- 了解长序列外推技术

**满分回答示范：**

**RoPE 工作原理：**
RoPE 将位置信息编码为 Q 和 K 的旋转。对于位置 m 的 token，将其 d 维向量分成 d/2 对（每对看作复数），对第 i 对施加角度为 m·θ_i 的旋转，其中 θ_i = 10000^{-2i/d}。

**为什么是相对位置：**
两个 token 在位置 m 和 n 的 attention score 为 q_m^T k_n。RoPE 后，旋转的性质使得 (R_m q)^T (R_n k) = q^T R_{n-m} k，即 attention score 只依赖于相对位置 (n-m)，而不是绝对位置 m 或 n 本身。这源于旋转矩阵的性质 R_m^T R_n = R_{n-m}。

**长期外推问题：**
训练时模型只见过位置 0-4095 的旋转角度。推理时位置 8000 对应更大的角度，而模型从未学过这些角度的"行为"，性能骤降。

**解决方案——NTK-aware scaling / YaRN：**
核心思想是"压低高频、保持低频"。将 θ_i 替换为 θ_i / λ，相当于将高频分量按比例缩放，使得长位置对应回训练时见过的角度范围。YaRN 进一步区分了高频和低频分量，对不同频率分量使用不同的缩放策略。

**实际操作：**
如果训练 4096 要推理 16384，我直接在 RoPE 中设置 NTK-aware scaling factor = (16384/4096) 的某个次幂，或者用 Dynamic NTK 在推理时动态调整。

**常见错误：**
- 说 RoPE 是一种"位置编码向量"直接加到 embedding 上——它是施加在 Q 和 K 上的旋转操作，不改变向量的模长
- 不知道或说不清 NTK-aware scaling——这是当前最主流的 RoPE 外推方法

---

## 问题 7：Attention 工程优化实战

**评分要点：**
- 有实际部署经验
- 理解各种 attention kernel 的选型依据
- 了解 continuous batching 和 KV Cache 管理的工程细节

**满分回答示范：**

（这是一个经验型问题，答题应体现真实的工程决策思路）

**优化路径：**

首先定位瓶颈——用 PyTorch Profiler + nvtx 标记，发现 attention 部分占推理延迟的 60%+。

1. **启用 KV Cache（如果还没做）：** 将 decode 阶段从每次重算所有 token 的 K、V，改为只算最新 token。这一步将 decode 延迟从 O(n²) 降到 O(n)。

2. **替换 Attention Kernel：** 评估了 Flash Attention 2、xformers memory_efficient_attention 和 PyTorch 2.0 的 scaled_dot_product_attention。在 A100 上 Flash Attention 2 最快，因为它的 backward 优化更好（写回 HBM 的数据量更少）。选型依据是：A100 的 SRAM 192KB，FA2 的 tile size 设计正好用满。

3. **Prefix Caching：** 多个请求可能有相同的 system prompt。使用 Radix Tree（Trie）匹配相同前缀，共享 KV Cache。这节省了 prefill 阶段对公共前缀的重复计算。但哈希匹配和树维护有 CPU overhead，需要在请求量较大时才有正收益。

4. **Continuous Batching：** 不同请求在不同 step 结束。与其等整个 batch 中最长的请求完成，不如每当一个请求结束就动态插入新请求。这增加了 GPU 利用率，但对 attention kernel 的要求更高——需要支持变长的序列（ragged batch）。

**10 倍流量下的瓶颈：**
KV Cache 内存是最大瓶颈。8 个请求 × 32K context × 32 layers × 8192 dim × 2 bytes ≈ 16 GB 的 KV Cache。流量 10 倍后面临两个选择：要么加 GPU（水平扩展），要么用 vLLM 的 PagedAttention + prefix caching（提升单 GPU 吞吐）。实际中两者结合——先用软件优化提高单卡利用率，到极限后再加卡。

**常见错误：**
- 只说理论方案没有实战经验——不具体
- 没提到 profiling——优化没有数据驱动
- 忽略 KV Cache 的内存问题——这是生产环境的真实瓶颈

---

## 问题 8：Attention 替代方案的未来

**评分要点：**
- 了解 State Space Models、Linear Attention 等前沿方向
- 有独立思考和技术判断力
- 不对任何方案有宗教式的信仰

**满分回答示范：**

**Mamba / SSM：**
核心思想是用一个结构化的状态空间模型代替 attention。SSM 将输入通过一个线性时不变系统，状态转移可表示为卷积形式。Mamba 的关键创新是 selective scan——让状态转移参数依赖于输入（类似 attention 的 data-dependence），同时保持高效的硬件感知实现（parallel scan + kernel fusion）。

本质区别：Attention 是显式的 token 间交互（每个 token 显式计算与所有其他 token 的 score），SSM 是隐式的状态压缩（历史信息压缩到固定大小的状态中）。

**Linear Attention：**
思路是用核函数 φ 将 softmax(QK^T) 近似为 φ(Q)φ(K)^T，利用矩阵乘法结合律改变计算顺序：先算 φ(K)^T·V（O(n)），再乘 φ(Q)（O(n)），总复杂度 O(n) 而非 O(n²)。但问题是：
- 需要保证 φ 的非负性，否则训练不稳定
- 实际效果在大多数任务上不如 softmax attention
- Performer (2020) 用了随机正交特征，但收敛速度慢
- 记忆压缩的方式丢失了精细的 token 级别信息

**我的判断：**
3-5 年内，Self-Attention 仍然是 LLM 的主流。原因：
1. **工程惯性：** Flash Attention 已经把 attention 的效率推到极致，学术界改变的动力不足。
2. **SSM 的局限：** Mamba 及其变体在长序列（>100K）上确有优势，但在常规长度（<8K）上质量仍不如同样参数量的 Transformer。而且 SSM 的 in-context learning 能力比 Transformer 弱——这暗示 attention 对于"从上下文中学习"有不可替代的作用。
3. **混合架构：** 更可能的是混合——如 Google 的 Griffin 用 attention + SSM 的混合架构。

但我不会押注任何一种。如果 SSM 在 scaling law 上追平 Transformer——那就是范式转换的时刻。保持关注但不冲动。

**常见错误：**
- 说 Mamba 是"RNN 的回归"——不完全对，Mamba 借鉴了 RNN 的递归结构但 key innovation 是 selective SSM
- 武断地说"Attention 不会被取代"——缺少分析依据
- 对 Linear Attention 只有表面了解——不知道为什么不 work
