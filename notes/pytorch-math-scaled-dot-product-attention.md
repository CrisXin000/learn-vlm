# Scaled Dot-Product Attention

> 日期：2026-10-19
> 标签：`pytorch` · `attention` · `softmax` · `flashattention` · `transformer`

注意力把「当前位置」写成一组 query，把「被看的位置」写成 key 和 value。输出是 value 的加权和，权重来自 query 与 key 的点积，再过一次 softmax。点积会随维度变大，所以要先除以 \(\sqrt{d_k}\)。这些分数就是 [Softmax 数值稳定与 Log-Sum-Exp](pytorch-math-softmax-logsumexp.md) 里的 logits：上一篇解决「指数别溢出」，这一篇解决「进 softmax 之前分数本身别大到让分布变成 one-hot」。FlashAttention 算的是同一个加权和，只是用在线的 log-sum-exp 把 \(L \times S\) 的矩阵留在计算过程里，不写回显存。

## 动机

一层自注意力要让序列里每个位置都能读到其他位置。做法是：位置 \(i\) 拿自己的 query 去和每个 key 做相似度，变成一组非负且和为 1 的权重，再对 value 求加权平均。相似度用点积，因为点积是一次矩阵乘 \(QK^{\mathsf T}\)，可以整批算完。

点积有一个和维度绑在一起的尺度问题。设 \(q, k \in \mathbb{R}^{d_k}\) 的各分量独立、均值 0、方差 1，则

\[
q \cdot k = \sum_{t=1}^{d_k} q_t k_t, \qquad \mathrm{Var}(q \cdot k) = d_k.
\]

单项 \(q_t k_t\) 的方差是 \(\mathbb{E}[q_t^2]\mathbb{E}[k_t^2] = 1\)，\(d_k\) 个相互独立的项加起来，方差就是 \(d_k\)，标准差是 \(\sqrt{d_k}\)。\(d_k = 64\) 时标准差约是 8，\(d_k = 1024\) 时约是 32。常见的线性层初始化（把输出方差保持在 1 附近）管的是 \(q\)、\(k\) 本身的分量，管不住这个求和：维度越宽，点积越散。

softmax 吃的是这些点积。分数的标准差从 1 涨到 8 以后，最大的那一项会比其余项高出好几个单位，权重几乎全部堆到一个 key 上。softmax 对分数的雅可比是

\[
\frac{\partial \alpha_i}{\partial s_j} = \alpha_i(\delta_{ij} - \alpha_j).
\]

当某个 \(\alpha_k \to 1\)、其余 \(\alpha \to 0\) 时，上式的每一项都趋向 0。反向传到 \(Q\)、\(K\) 的梯度一起消失，这一层学不动「该看谁」。Vaswani 等人在 *Attention Is All You Need* 里把除以 \(\sqrt{d_k}\) 放进定义，就是为了把点积的方差拉回 1，让 softmax 停在梯度还在的区域。

这和上一篇是前后两截。上一篇的最大值平移保证：分数即使已经很大，`softmax` 仍然给出有限概率，而不是 NaN。平移消掉的是溢出，消不掉尖峰。不缩放的 \(d_k = 64\) 点积，尾部经常超过 float16 里 \(\exp\) 大约 11 的溢出点；`F.softmax` 先减行最大值，结果仍是有限的，但分布已经接近 one-hot，雅可比仍然接近 0。缩放是在构造 logits，数值稳定是在安全地计算 softmax。

## 数学

### 定义

一行序列的 query、key、value 写成矩阵（多头时每个头各有一套，头和 batch 都放在前面的维度里）：

\[
Q \in \mathbb{R}^{L \times d_k}, \quad
K \in \mathbb{R}^{S \times d_k}, \quad
V \in \mathbb{R}^{S \times d_v}.
\]

\(L\) 是 query 的长度，\(S\) 是 key / value 的长度，自注意力里通常 \(L = S\)。缩放点积注意力是

\[
\mathrm{Attention}(Q, K, V)
= \mathrm{softmax}\!\left(\frac{QK^{\mathsf T}}{\sqrt{d_k}}\right) V.
\]

拆成三个量。位置 \(i\) 对位置 \(j\) 的分数、权重和输出：

\begin{align*}
s_{ij} &= \frac{q_i \cdot k_j}{\sqrt{d_k}}, \\
\alpha_{ij} &= \frac{e^{s_{ij}}}{\sum_{j'=1}^{S} e^{s_{ij'}}}, \\
o_i &= \sum_{j=1}^{S} \alpha_{ij}\, v_j.
\end{align*}

softmax 沿 key 那一维做：每一行 \(\alpha_{i,:}\) 非负且和为 1，所以 \(o_i\) 是 \(\{v_j\}\) 的凸组合。\(d_v\) 可以和 \(d_k\) 不同；缩放只用 \(d_k\)，因为点积发生在 query 和 key 之间。

多头就是把这个式子重复 \(H\) 次。第 \(h\) 个头有自己的 \(Q_h, K_h, V_h\)，宽度仍是 \(d_k\)，缩放因子仍是 \(1/\sqrt{d_k}\)，不是 \(1/\sqrt{d_{\mathrm{model}}}\)。各头的输出沿宽度拼起来，再乘一个输出投影。`F.scaled_dot_product_attention` 不做这些投影，它收到的张量里，头已经是 batch 维。

### 在线算法：同一个 softmax，不存整张权重

把一行的计算顺序列出来。分数 \(x_1,\ldots,x_S\)，对应 value \(v_1,\ldots,v_S\)。维护三个运行量：当前最大值 \(m\)、未归一化的配分函数 \(\ell\)、未归一化的输出 \(o\)。初值 \(m = -\infty\)，\(\ell = 0\)，\(o = 0\)。看到第 \(j\) 个 key：

\begin{align*}
m' &= \max(m, x_j), \\
\alpha &= e^{m - m'}, \\
\ell' &= \alpha\,\ell + e^{x_j - m'}, \\
o' &= \alpha\, o + e^{x_j - m'}\, v_j.
\end{align*}

扫完后的 \(o / \ell\) 就是 \(\mathrm{softmax}(x)\,V\) 的那一行。\(\alpha = e^{m_{\mathrm{old}} - m_{\mathrm{new}}}\) 是上一篇的平移恒等式：最大值一旦变大，旧的指数和要乘上 \(e^{m_{\mathrm{old}} - m_{\mathrm{new}}}\) 才和「相对新最大值」的指数在同一个零点上。整行一次性做 \(\mathrm{softmax}\) 再乘 \(V\)，和按 key 扫完再除以 \(\ell\)，在实数上是同一个数。

一块 key 的合并用的是同一组式子。块内先算出自己的最大值 \(m_{\mathrm{blk}}\)、\(\ell_{\mathrm{blk}} = \sum e^{x - m_{\mathrm{blk}}}\) 和 \(o_{\mathrm{blk}} = \sum e^{x - m_{\mathrm{blk}}} v\)，再写

\begin{align*}
m' &= \max(m, m_{\mathrm{blk}}), \\
\ell' &= e^{m - m'}\,\ell + e^{m_{\mathrm{blk}} - m'}\,\ell_{\mathrm{blk}}, \\
o' &= e^{m - m'}\,o + e^{m_{\mathrm{blk}} - m'}\,o_{\mathrm{blk}}.
\end{align*}

这仍是「到目前为止的精确 softmax 加权和」，所以块的顺序不影响实数结果。

FlashAttention（Dao 等，2022）以及 FlashAttention-2 做的就是这件事：按块更新 \((m, \ell, o)\)。它和手写的 `matmul → softmax → matmul` 是同一个函数。差别在访存。手写版本要把 \(L \times S\) 的分数矩阵和概率矩阵都存下来；序列一长，这两张表比 \(Q, K, V\) 本身还大，后者的规模是 \(O(L d_k + S d_k + S d_v)\)。在线版本每行只留一个标量 \(m\)、一个标量 \(\ell\) 和一条长度 \(d_v\) 的输出，当前块的分数用完即弃，\(L \times S\) 的矩阵不进入全局显存。加法顺序不同，所以浮点结果不必逐位相同。

因果掩码在送进 softmax 之前，把不允许看的位置写成 \(-\infty\)。那些位置的权重是 0，输出仍是剩余 value 的凸组合。

## PyTorch 里怎么做

`torch.nn.functional.scaled_dot_product_attention` 的文档给出了与上面公式对应的参考实现（这里只留计算本身）：

```python
scale_factor = 1 / math.sqrt(query.size(-1)) if scale is None else scale
attn_weight = query @ key.transpose(-2, -1) * scale_factor
attn_weight = torch.softmax(attn_weight, dim=-1)   # 掩码已加进 attn_weight
return attn_weight @ value
```

`torch.softmax` 就是上一篇里的稳定 softmax：沿最后一维减最大值再指数归一化。

接口（PyTorch 2.14 的签名；文档标成 beta，参数以后可能变）：

```text
scaled_dot_product_attention(
    query, key, value,
    attn_mask=None, dropout_p=0.0, is_causal=False,
    scale=None, enable_gqa=False,
) -> Tensor
```

- `query` 的形状是 \((N, \ldots, H_q, L, E)\)，`key` 是 \((N, \ldots, H, S, E)\)，`value` 是 \((N, \ldots, H, S, E_v)\)。\(E\) 就是 \(d_k\)，\(E_v\) 就是 \(d_v\)。输出形状 \((N, \ldots, H_q, L, E_v)\)。没有 batch、没有头时，\((L, E)\)、\((S, E)\)、\((S, E_v)\) 也能算。
- `scale` 只能作为关键字参数。默认 \(1/\sqrt{E}\)。`enable_gqa=True` 时多个 query 头共享同一组 key / value；默认 `False`，本篇的公式按每个头自己的 \(K, V\) 来写。
- `is_causal=True` 且 \(L = S\) 时，掩码是下三角：位置 \(i\) 只看 \(j \le i\)。它和 `attn_mask` 不能同时给。实现上等价于 softmax 之前把上三角（不含对角线）加上 \(-\infty\)。
- `attn_mask` 可以广播到注意力权重的形状 \((N, \ldots, H_q, L, S)\)。布尔掩码里 `True` 表示这个位置**参与**注意力；浮点掩码则直接加到分数上。这和 `nn.MultiheadAttention` 的 `key_padding_mask` 相反，那边的 `True` 表示掩掉。
- `dropout_p > 0` 时函数总会做 dropout，不会自己看模块是不是在 `eval()`。默认 `0.0`。和手写 softmax 对数值时保持默认。

文档列出的实现有三种：与上面公式一致的 PyTorch C++ 版本、FlashAttention-2、Memory-Efficient Attention。CUDA 上还可能走到 cuDNN。CPU 使用 PyTorch 实现。融合内核改变浮点运算的结合顺序，同一组输入在不同后端上不必逐位相同；要更高精度时，文档指出 C++ 实现支持 `float64`。`torch.nn.attention.sdpa_kernel` 可以用 `SDPBackend.MATH`、`SDPBackend.FLASH_ATTENTION`、`SDPBackend.EFFICIENT_ATTENTION` 限定后端。本篇的例子在 CPU 上跑，走的是与公式一致的那条路径。

## 可运行示例

CPU、默认 float32、PyTorch 2.x 即可。注释里的数字来自这段代码的默认打印。

```python
import math
import torch
import torch.nn.functional as F

torch.manual_seed(0)

# 分量独立、标准正态时，点积的标准差约等于 sqrt(d_k)。
d = 64
q = torch.randn(100_000, d)
k = torch.randn(100_000, d)
dots = (q * k).sum(dim=-1)
# 期望：dots.std() ≈ 7.992，(dots / sqrt(64)).std() ≈ 0.999，sqrt(64) = 8
assert abs(dots.std().item() - math.sqrt(d)) < 0.05
assert abs((dots / math.sqrt(d)).std().item() - 1.0) < 0.02

# 不缩放时，softmax 把质量堆到最大的 key 上。均匀分布时每个 key 是 1/16 = 0.0625。
torch.manual_seed(0)
S, d, N = 16, 256, 128
q = torch.randn(N, d)
K = torch.randn(N, S, d)
raw = torch.einsum("nd,nsd->ns", q, K)
p_raw = torch.softmax(raw, dim=-1)
p_scaled = torch.softmax(raw / math.sqrt(d), dim=-1)
# 期望：平均最大概率，不缩放 ≈ 0.914，缩放后 ≈ 0.261
assert p_raw.amax(dim=-1).mean().item() > 0.85
assert p_scaled.amax(dim=-1).mean().item() < 0.35

# 手写 matmul + softmax 与 F.scaled_dot_product_attention 对齐。
# scale 的默认值就是 1/sqrt(E)，并且只能写成关键字参数。
torch.manual_seed(1)
B, H, L, E, S = 2, 3, 4, 8, 5
query = torch.randn(B, H, L, E)
key = torch.randn(B, H, S, E)
value = torch.randn(B, H, S, E)
scale = 1.0 / math.sqrt(E)
scores = query @ key.transpose(-2, -1) * scale
manual = torch.softmax(scores, dim=-1) @ value
fused = F.scaled_dot_product_attention(query, key, value, dropout_p=0.0)
fused_scale = F.scaled_dot_product_attention(
    query, key, value, dropout_p=0.0, scale=scale
)
assert torch.allclose(manual, fused, atol=1e-5, rtol=1e-5)
assert torch.allclose(manual, fused_scale, atol=1e-5, rtol=1e-5)

# 在线 softmax：一行 query 扫过 S 个 key，只保留 m、ℓ、o。
# 这是 FlashAttention 分块合并在「一块只有一个 key」时的特例。
x = scores[0, 0, 0]          # (S,)
v_row = value[0, 0]          # (S, E)
m = x.new_tensor(float("-inf"))
ell = x.new_zeros(())
acc = v_row.new_zeros(E)
for j in range(S):
    m_new = torch.maximum(m, x[j])
    alpha = torch.exp(m - m_new)             # 最大值变了就重缩放旧统计量
    p_j = torch.exp(x[j] - m_new)
    ell = ell * alpha + p_j
    acc = acc * alpha + p_j * v_row[j]
    m = m_new
online = acc / ell
# 与一次性 softmax(scores) @ V 的同一行一致
assert torch.allclose(online, manual[0, 0, 0], atol=1e-5, rtol=1e-5)
```

## 要点

- \(\mathrm{Attention}(Q,K,V) = \mathrm{softmax}(QK^{\mathsf T}/\sqrt{d_k})\,V\)。每一行输出是 value 的凸组合，权重来自这一行 query 和所有 key 的缩放点积。
- 分量独立且方差为 1 时，点积的方差是 \(d_k\)。除以 \(\sqrt{d_k}\) 把标准差拉回 1。不缩放时权重趋向 one-hot，softmax 的雅可比 \(\alpha_i(\delta_{ij}-\alpha_j)\) 趋向 0，\(Q\)、\(K\) 的梯度消失。
- 这些分数就是上一篇的 logits。最大值平移让 `softmax` 在大分数下仍给出有限值；缩放则让分数保持在梯度还在的尺度。两件事解决的是两个问题。
- 在线更新 \((m, \ell, o)\) 用的是同一条平移恒等式。FlashAttention 按块做这个更新，与手写 `matmul + softmax + matmul` 在实数上相同；手写版本存放 \(L \times S\) 的分数和概率，在线版本不把这张矩阵写回显存。
- `F.scaled_dot_product_attention` 的默认缩放是 \(1/\sqrt{E}\)，`dropout_p=0` 时与手写公式一致。CPU 走该公式的实现；CUDA 上的 FlashAttention 等融合内核改变求和顺序，结果不必逐位相同。

## 练习

1. 设 \(q, k\) 的 \(d_k\) 个分量独立、服从标准正态。推导 \(\mathrm{Var}(q\cdot k)\)。取 \(d_k = 1024\)，算不缩放与除以 \(\sqrt{d_k}\) 之后的标准差。再设两个分数恰好是 \((2\sqrt{d_k},\ 0)\)：不缩放时较大的那个是否越过 float16 里 \(\exp\) 约 11 的溢出点？把这两个分数先除以 \(\sqrt{d_k}\) 再做 softmax，较大项的权重大约是多少？不除、直接做 softmax 呢？
2. 取 \(L = S\) 的随机 \(Q, K, V\)。用 `is_causal=True` 跑一次 `F.scaled_dot_product_attention`，再用手写分数把上三角（不含对角线）填成 `-inf` 之后做 softmax，核对两个输出。然后在在线更新里送入一个 \(x_j = -\infty\)：这一项会不会改写运行中的最大值 \(m\)，它对 \(\ell\) 和 \(o\) 的增量是多少？
