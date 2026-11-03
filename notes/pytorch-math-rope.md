# 旋转位置编码（RoPE）

> 日期：2026-11-01
> 标签：`pytorch` · `rope` · `位置编码` · `attention` · `transformer`

[Scaled Dot-Product Attention](pytorch-math-scaled-dot-product-attention.md) 的分数是 \(q_i \cdot k_j / \sqrt{d_k}\)。这个点积只看见两条向量，看不见它们在序列里相距多远。绝对位置编码把位置向量加进 token，再进 Q、K 的线性层，点积里会同时留下两个绝对位置，一般拆不成只含位移的一项。旋转位置编码（RoPE）改在投影之后做：把 \(q\)、\(k\) 的每两维当成一个平面，按位置转一个角度。转完之后的点积等于原向量夹着相对旋转 \(R_{n-m}\) 的点积，位移相同则分数相同。这些分数仍是 [Softmax 数值稳定与 Log-Sum-Exp](pytorch-math-softmax-logsumexp.md) 里的 logits，仍除以 \(\sqrt{d_k}\)。旋转是正交的，不改变 \(q\)、\(k\) 的范数，所以 [LayerNorm 与 RMSNorm](pytorch-math-layernorm-rmsnorm.md) 要管的尺度还在：RoPE 不代替归一化。这篇从这条旋转恒等式写到一份可运行的参考实现。

## 动机

一层自注意力要让位置 \(i\) 读到位置 \(j\)。相似度是这两处 query 和 key 的点积。如果 \(q\)、\(k\) 只由 token 内容经线性层得到，序列怎么排列都不进这个点积：把 token 置换一下，分数矩阵做同一个置换，模型分不出「挨着」和「隔了很远」。因果掩码只规定下标 \(j\) 能不能被下标 \(i\) 看见，不规定两个都看得见的位置相距多少。被挡住的位置在进 softmax 之前写成 \(-\infty\)，其余位置的分数仍只是点积，距离没有写进这个点积。

绝对位置编码把一份位置向量加到 token 上，再投影。记内容为 \(x_m\)、位置向量为 \(p_m\)，投影后的 query 是 \(W_q(x_m + p_m)\)。它和位置 \(n\) 的 key 做点积，展开之后是内容与内容、内容与位置、位置与位置这三类项的和。\(W_q\)、\(W_k\) 是学出来的，这个和一般写不成某个只含 \(m-n\) 的函数。即使先不投影、直接把位置向量加到已经算好的 \(q\)、\(k\) 上，

\[
(q + p_m)\cdot(k + p_n) = q\cdot k + q\cdot p_n + p_m\cdot k + p_m\cdot p_n,
\]

中间两项跟着绝对下标 \(m\) 和 \(n\) 走。把两个位置一起加上常数 \(s\)，\(q\cdot p_{n+s}\) 与 \(p_{m+s}\cdot k\) 一般不再是原来的数，点积跟着变。

RoPE 不加位置向量。Q、K 投影照常产生 \(q_m, k_n \in \mathbb{R}^{d}\)，\(d = d_k\) 是这个头的宽度，要求偶数。然后按位置左乘一个旋转 \(R_m\)、\(R_n\)，再送进点积。\(R_m\) 由一组只依赖位置 \(m\) 的平面旋转拼成。这些旋转满足 \(R_m^{\mathsf T} R_n = R_{n-m}\)，下一节把矩阵乘开。于是

\[
(R_m q)\cdot(R_n k) = q^{\mathsf T} R_{n-m} k.
\]

右边的位置信息只有差 \(n-m\)。同一对向量，只要相对位移不变，点积就不变；整段位置编号一起平移，点积矩阵逐项不变。value 不旋转，输出仍是原来那些 value 的凸组合，权重则看见了距离。旋转要乘在进入点积的那两条向量上。先把隐藏状态 \(x\) 旋转，再乘一个一般不与 \(R_m\) 交换的投影 \(W\)，\((W R_m x)\cdot(W R_n y)\) 写不成 \(x^{\mathsf T} R_{n-m} y\)。

这和 Softmax、缩放点积那两篇接在分数的尺度上，不接在「指数别溢出」上。旋转正交，\(\lVert R_m q\rVert = \lVert q\rVert\)。\(q\) 与 \(k\) 彼此独立、分量独立且标准正态时，点积的方差仍是 \(d_k\)，除以 \(\sqrt{d_k}\) 仍把标准差拉回 1。隐藏状态的分量标准差若是 \(\sigma\) 而不是 1，缩放后的分数标准差仍是 \(\sigma^2\)，和 LayerNorm 那篇里残差把尺度抬上去之后的计算相同。RoPE 不把 \(\sigma\) 拉回 1。分数很大时，仍靠最大值平移得到有限的 softmax；平移消不掉 one-hot。

## 数学

### 把两维看成一个平面

把宽度 \(d\) 拆成 \(d/2\) 对，第 \(i\) 对是下标 \((2i,\ 2i+1)\)，\(i = 0, \ldots, d/2 - 1\)。这一对的角频率是

\[
\theta_i = \mathrm{base}^{-2i/d}.
\]

参考实现里 \(\mathrm{base} = 10000\)。于是 \(\theta_0 = 1\)，越靠后的一对转得越慢。位置 \(m\) 在第 \(i\) 对上的角度是 \(m\theta_i\)。

平面上逆时针转 \(\phi\) 的矩阵是

\[
R(\phi)
= \begin{pmatrix} \cos\phi & -\sin\phi \\ \sin\phi & \cos\phi \end{pmatrix}.
\]

它作用在列向量 \((x_{2i},\ x_{2i+1})^{\mathsf T}\) 上。\(R_m\) 是 \(d\times d\) 的分块对角矩阵，第 \(i\) 块是 \(R(m\theta_i)\)，块与块之间不混合。多头时每个头各转自己的 \(d_k\) 维，频率表用的是 \(d_k\)，不是 \(d_{\mathrm{model}}\)。这和缩放点积那篇「缩放因子用 \(d_k\)」是同一处宽度。

角度相加就是矩阵相乘。把 \(R(\alpha)R(\beta)\) 乘开，用余弦、正弦的和角公式，得到 \(R(\alpha+\beta)\)。转置是反向旋转：\(R(\phi)^{\mathsf T} = R(-\phi)\)。因此每一块

\[
R(m\theta_i)^{\mathsf T}\, R(n\theta_i) = R(-m\theta_i)\, R(n\theta_i) = R((n-m)\theta_i).
\]

分块对角，所以整个矩阵同样满足 \(R_m^{\mathsf T} R_n = R_{n-m}\)。\(R_m\) 正交：\(R_m^{\mathsf T} R_m = I\)，因而 \(\lVert R_m q\rVert = \lVert q\rVert\)。

同一件事可以写成复数。把第 \(i\) 对看成 \(z = x_{2i} + \mathrm{i}\, x_{2i+1}\)，位置 \(m\) 的旋转就是乘 \(e^{\mathrm{i} m \theta_i} = \cos(m\theta_i) + \mathrm{i}\sin(m\theta_i)\)。实部、虚部拆开，就是上面的 \(R(m\theta_i)\)。

### 点积只留下相对位移

查询在位置 \(m\)、键在位置 \(n\)，旋转之后的点积是

\begin{align*}
(R_m q)\cdot(R_n k)
&= (R_m q)^{\mathsf T} (R_n k) \\
&= q^{\mathsf T} R_m^{\mathsf T} R_n k \\
&= q^{\mathsf T} R_{n-m} k.
\end{align*}

右边只通过 \(n-m\) 依赖两个位置。\(R_{n-m} = R_{-(m-n)}\)，所以它也只是差 \(m-n\) 的函数。两边同时加上整数 \(s\)：

\[
(R_{m+s} q)\cdot(R_{n+s} k)
= q^{\mathsf T} R_{(n+s)-(m+s)} k
= q^{\mathsf T} R_{n-m} k.
\]

\(q\)、\(k\) 任意，不必来自线性层。把一条序列的位置编号全体加上 \(s\)，每一对 \((q_i, k_j)\) 的位移都不变，整张点积矩阵不变。

写成第 \(i\) 对的坐标，设 \(\phi = (n-m)\theta_i\)，则这一对对点积的贡献是

\[
(q_{2i} k_{2i} + q_{2i+1} k_{2i+1})\cos\phi
+ (q_{2i+1} k_{2i} - q_{2i} k_{2i+1})\sin\phi.
\]

\(d=2\) 时只有一对，\(\theta_0 = 1\)。取 \(q = (1, 0)\)、\(k = (0, 1)\)、\(m = 0\)、\(n = 1\)，则 \(\phi = 1\)，上式等于 \(-\sin 1\)。\(R_0 q = (1, 0)\)，\(R_1 k = (-\sin 1,\ \cos 1)\)，点积同样是 \(-\sin 1\)。

只旋转其中一边，恒等式用不上。\((R_m q)\cdot k = q^{\mathsf T} R_{-m} k\)，右边只剩绝对位置 \(m\)。\(m\) 换成 \(m+s\) 会得到 \(R_{-(m+s)}\)，一般不再等于原来的点积。

### 放进注意力分数

自注意力里，位置 \(m\) 的 query、位置 \(n\) 的 key 在投影之后、点积之前旋转。value 保持原样。分数和输出是

\begin{align*}
s_{mn}
&= \frac{(R_m q_m)\cdot(R_n k_n)}{\sqrt{d_k}}
= \frac{q_m^{\mathsf T} R_{n-m} k_n}{\sqrt{d_k}}, \\
\alpha_{m,:}
&= \mathrm{softmax}(s_{m,:}), \\
o_m
&= \sum_n \alpha_{mn}\, v_n.
\end{align*}

softmax 仍沿 key 那一维做，每一行权重非负且和为 1，所以 \(o_m\) 仍是 \(\{v_n\}\) 的凸组合。位置进的是分数，不是 value。

\(m = n\) 时 \(R_{n-m} = I\)，同一位置上的一对向量，旋转后的点积等于没转的点积。不同位置才把内容「拧」开一个相对角。

设 \(q\)、\(k\) 彼此独立，分量独立、均值 0、方差 1。正交变换不改变这个分布：\(R_m q\) 与 \(R_n k\) 仍是彼此独立的标准正态向量，点积方差仍是 \(d_k\)，和缩放点积那篇里没转之前相同。默认缩放仍是 \(1/\sqrt{d_k}\)。均值仍是 0、分量方差改成 \(\sigma^2\) 时，单项 \(q_t k_t\) 的方差是 \(\sigma^4\)，\(d_k\) 项相加，点积的标准差是 \(\sigma^2\sqrt{d_k}\)，除掉 \(\sqrt{d_k}\) 之后是 \(\sigma^2\)。旋转不改这个计算。

分数矩阵在位置平移下逐项不变，不必再靠 softmax 的平移不变去消一个每行共用的常数。那条平移不变仍然管另一件事：分数已经很大时，`softmax` 先减去行最大值，输出有限；权重可以仍然接近 one-hot。RoPE 不缩放 logits。

## PyTorch 里怎么做

PyTorch 2.15 的 `torch.nn` 和 `torch.nn.functional` 里没有 RoPE 模块，也没有 `rotary_embedding`。`F.scaled_dot_product_attention` 收到的 query、key 是已经旋转过的；它只做 \(QK^{\mathsf T}/\sqrt{E}\)、softmax、再乘 \(V\)，默认缩放仍是 \(1/\sqrt{E}\)。位置编码要在调用它之前完成。

包内能直接算 RoPE 的是 ONNX 算子 `torch.onnx.ops.rotary_embedding`，文档把它标成可在 `nn.Module` 里调用、再导出的原生实现，不是 `nn` 里的一层。2.15 的签名是

```text
torch.onnx.ops.rotary_embedding(
    X, cos_cache, sin_cache, position_ids=None, *,
    interleaved=False, num_heads=0, rotary_embedding_dim=0,
) -> Tensor
```

四维 `X` 的形状是 \((B, H, S, D)\)，\(D\) 为偶数，和 `scaled_dot_product_attention` 的头维布局一致。`position_ids` 形状 \((B, S)\) 时，`cos_cache`、`sin_cache` 是 \((M, D/2)\)，第 \(p\) 行是位置编号 \(p\) 的余弦和正弦，\(M\) 要盖住出现的编号。`interleaved=True` 按相邻两维 \((2i, 2i+1)\) 旋转，和下面的参考实现一致。默认 `False` 把最后一维切成前后两半，配对的是下标 \(i\) 和 \(i+D/2\)。两种都是分块旋转，相对位移的恒等式都成立，但轴的配法不同，同一条输入的坐标对不上。`rotary_embedding_dim=0` 表示整段 \(D\) 都旋转；改成更小的偶数时，只旋转前面那么多维，其余坐标原样拷贝，余弦表的宽度改成这个偶数的一半。

本篇的参考实现走相邻两维，不依赖这个 ONNX 算子。`positions` 是位置编号，普通序列就是 `torch.arange(L)`。

```python
def rope_angles(positions, d, base=10000.0):
    # θ_i = base^{-2i/d}，i = 0 .. d/2-1。返回 cos、sin，形状是 positions 的形状再接 d/2。
    i = torch.arange(d // 2, dtype=torch.float32)
    theta = base ** (-2.0 * i / d)
    ang = positions.to(dtype=torch.float32).unsqueeze(-1) * theta
    return torch.cos(ang), torch.sin(ang)

def apply_rotary_emb(x, cos, sin):
    # (x_{2i}, x_{2i+1}) 左乘 R(φ_i)。cos、sin 广播到 x[..., 0::2]。
    x1 = x[..., 0::2]
    x2 = x[..., 1::2]
    y1 = x1 * cos - x2 * sin
    y2 = x1 * sin + x2 * cos
    return torch.stack((y1, y2), dim=-1).flatten(-2)
```

`x` 的最后一维是偶数 \(d\)。序列放在倒数第二维、形状 \((B, H, L, d)\) 时，`cos`、`sin` 用 `(L, d/2)` 就能从右侧广播到 `(B, H, L, d/2)`。不要把频率表变成 `(1, L, d/2)`：那一维会对到 head 上，\(H \neq L\) 时广播失败。

LLaMA 参考实现里的 `apply_rotary_emb` 是同一旋转的复数写法：最后一维两两 `view_as_complex`，乘 \(e^{\mathrm{i} m \theta_i}\)（`torch.polar` 或等价的 \(\cos + \mathrm{i}\sin\)），再 `view_as_real` 拆开。它的张量是 `(batch, seq, n_heads, head_dim)`，频率沿 seq 广播。上面的函数改成和缩放点积注意力相同的 `(..., seq, d)`，实数路径和这条复数路径在 float32 里对齐。

另一套常见写法是 `rotate_half`。把 \(x = [x^{(1)}, x^{(2)}]\) 切成前后各 \(d/2\) 维，

\[
y = \big(x^{(1)}\cos - x^{(2)}\sin,\ \ x^{(2)}\cos + x^{(1)}\sin\big),
\]

余弦、正弦先沿最后一维拉成 \(d\) 维（两半各用同一份角）。这是下标 \(i\) 与 \(i+d/2\) 的平面旋转，对应 `rotary_embedding(..., interleaved=False)`。名字和 LLaMA 那份 `apply_rotary_emb` 不是同一个配对。

长上下文里有人会改 \(\theta_i\) 的间距。本篇只用 \(\mathrm{base}^{-2i/d}\)，不引入那种重标度。

## 可运行示例

CPU、float32、PyTorch 2.15。注释里的张量是默认打印精度下的结果。

```python
import math
import torch
import torch.nn.functional as F

def rope_angles(positions, d, base=10000.0):
    i = torch.arange(d // 2, dtype=torch.float32)
    theta = base ** (-2.0 * i / d)
    ang = positions.to(dtype=torch.float32).unsqueeze(-1) * theta
    return torch.cos(ang), torch.sin(ang)

def apply_rotary_emb(x, cos, sin):
    x1 = x[..., 0::2]
    x2 = x[..., 1::2]
    y1 = x1 * cos - x2 * sin
    y2 = x1 * sin + x2 * cos
    return torch.stack((y1, y2), dim=-1).flatten(-2)

def apply_rotary_emb_complex(x, positions, base=10000.0):
    # LLaMA apply_rotary_emb：相邻两维看成复数，乘 e^{i m θ_i}。positions 形状 (L,)。
    d = x.shape[-1]
    idx = torch.arange(0, d, 2, dtype=torch.float32)          # 2i
    theta = 1.0 / (base ** (idx / d))                         # base^{-2i/d}
    freqs = torch.outer(positions.to(torch.float32), theta)
    cis = torch.polar(torch.ones_like(freqs), freqs)
    xc = torch.view_as_complex(x.reshape(*x.shape[:-1], -1, 2).contiguous())
    return torch.view_as_real(xc * cis).flatten(-2)

# d = 2，θ_0 = 1。q 在位置 0 不动；k 在位置 1 转到 (-sin 1, cos 1)。
# 标量位置给出形状 (d/2,) 的 cos、sin。写成形状 (1,) 会多出一个前导维。
q = torch.tensor([1.0, 0.0])
k = torch.tensor([0.0, 1.0])
rq = apply_rotary_emb(q, *rope_angles(torch.tensor(0.0), 2))
rk = apply_rotary_emb(k, *rope_angles(torch.tensor(1.0), 2))
# 期望 rq: tensor([1., 0.])
# 期望 rk: tensor([-0.8415,  0.5403])
dot = torch.dot(rq, rk)                                        # tensor(-0.8415)
rel = torch.dot(q, rk)                                         # q^T R_{1-0} k
dot_shift = torch.dot(
    apply_rotary_emb(q, *rope_angles(torch.tensor(5.0), 2)),
    apply_rotary_emb(k, *rope_angles(torch.tensor(6.0), 2)),
)
assert rq.shape == q.shape and rk.shape == k.shape
assert torch.allclose(rq, torch.tensor([1.0, 0.0]))
assert torch.allclose(rk, torch.tensor([-math.sin(1.0), math.cos(1.0)]))
assert torch.allclose(dot, rel)
assert torch.allclose(dot, dot_shift)
assert torch.allclose(dot, torch.tensor(-math.sin(1.0)))

# 只旋转 q。点积跟着绝对位置走，平移不再保持。
bare = torch.dot(apply_rotary_emb(q, *rope_angles(torch.tensor(0.0), 2)), k)
moved = torch.dot(apply_rotary_emb(q, *rope_angles(torch.tensor(1.0), 2)), k)
assert torch.allclose(bare, torch.tensor(0.0))
assert not torch.allclose(bare, moved)

# 实数旋转、复数乘法、ONNX interleaved=True 三者对齐。默认 interleaved=False 是前后两半。
torch.manual_seed(0)
B, H, L, D = 2, 3, 5, 8
x = torch.randn(B, H, L, D)
pos = torch.arange(L)
cos, sin = rope_angles(pos, D)
y = apply_rotary_emb(x, cos, sin)
assert y.shape == x.shape
assert torch.allclose(y, apply_rotary_emb_complex(x, pos), atol=1e-6, rtol=1e-6)
pos_ids = pos.view(1, -1).expand(B, -1)
y_onnx = torch.onnx.ops.rotary_embedding(x, cos, sin, pos_ids, interleaved=True)
y_half = torch.onnx.ops.rotary_embedding(x, cos, sin, pos_ids, interleaved=False)
assert torch.allclose(y, y_onnx)
assert not torch.allclose(y, y_half, atol=1e-4)

# 位置编号全体 +100：分数矩阵不变，softmax 加权也不变。value 没有旋转。
# 未旋转的 QK^T/√d 是另一张表。F.scaled_dot_product_attention 吃旋转之后的 Q、K。
torch.manual_seed(1)
Q = torch.randn(B, H, L, D)
K = torch.randn(B, H, L, D)
V = torch.randn(B, H, L, D)

def scores(positions):
    c, s = rope_angles(positions, D)
    qr = apply_rotary_emb(Q, c, s)
    kr = apply_rotary_emb(K, c, s)
    return (qr @ kr.transpose(-2, -1)) / math.sqrt(D)

s0 = scores(pos)
s1 = scores(pos + 100)
plain = (Q @ K.transpose(-2, -1)) / math.sqrt(D)
out0 = torch.softmax(s0, dim=-1) @ V
out1 = torch.softmax(s1, dim=-1) @ V
fused = F.scaled_dot_product_attention(
    apply_rotary_emb(Q, cos, sin),
    apply_rotary_emb(K, cos, sin),
    V,
    dropout_p=0.0,
)
assert s0.shape == (B, H, L, L) and out0.shape == (B, H, L, D)
assert torch.allclose(s0, s1, atol=1e-5, rtol=1e-5)
assert torch.allclose(out0, out1, atol=1e-5, rtol=1e-5)
assert torch.allclose(out0, fused, atol=1e-5, rtol=1e-5)
assert not torch.allclose(s0, plain, atol=1e-3)

# 同一位置：R_m^T R_m = I，逐 token 的点积等于没转。
same = (apply_rotary_emb(Q, cos, sin) * apply_rotary_emb(K, cos, sin)).sum(-1)
raw = (Q * K).sum(-1)
assert torch.allclose(same, raw, atol=1e-5, rtol=1e-5)

# 相对位移不为 0 时范数仍保持，点积标准差仍约 sqrt(d_k)。
torch.manual_seed(0)
N, d = 100_000, 64
q = torch.randn(N, d)
k = torch.randn(N, d)
mpos = torch.randint(0, 2048, (N,))
qr = apply_rotary_emb(q, *rope_angles(mpos, d))
kr = apply_rotary_emb(k, *rope_angles(mpos + 17, d))
dots = (qr * kr).sum(-1)
# 期望：dots.std() ≈ 7.977，(dots / sqrt(64)).std() ≈ 0.997，sqrt(64) = 8
assert torch.allclose(qr.norm(dim=-1), q.norm(dim=-1), atol=1e-5, rtol=1e-5)
assert abs(dots.std().item() - math.sqrt(d)) < 0.05
assert abs((dots / math.sqrt(d)).std().item() - 1.0) < 0.02
```

## 要点

- 把第 \(i\) 对坐标 \((x_{2i}, x_{2i+1})\) 用 \(R(m\theta_i)\) 旋转，\(\theta_i = \mathrm{base}^{-2i/d}\)，\(\mathrm{base}\) 取 10000。\(R_m^{\mathsf T} R_n = R_{n-m}\)，所以 \((R_m q)\cdot(R_n k) = q^{\mathsf T} R_{n-m} k\)，只依赖位移 \(n-m\)。
- 注意力分数是这个点积除以 \(\sqrt{d_k}\)，再 softmax，再乘没有旋转的 \(V\)。位置平移不改变分数矩阵，输出仍是 value 的凸组合。
- \(R_m\) 正交，不改变范数。分量独立且方差为 1 时，点积方差仍是 \(d_k\)，缩放仍是 \(1/\sqrt{d_k}\)。分量标准差是 \(\sigma\) 时，缩放后的分数标准差仍是 \(\sigma^2\)。RoPE 不代替 LayerNorm / RMSNorm，也不代替 softmax 的最大值平移。
- PyTorch 2.15 的 `torch.nn` 没有 RoPE。`torch.onnx.ops.rotary_embedding(..., interleaved=True)` 是相邻两维；默认 `False` 是前后两半。LLaMA 的 `apply_rotary_emb` 用复数乘法实现前一种。`F.scaled_dot_product_attention` 不负责旋转。

## 练习

1. 设 \(d = 2\)，于是 \(\theta_0 = 1\)。取 \(q = (1,\ 0)\)，\(k = (0,\ 1)\)。写出 \(R(\phi)\)，计算 \(m = 0\)、\(n = 1\) 时的 \(R_m q\)、\(R_n k\) 和点积，再用 \(q^{\mathsf T} R_{n-m} k\) 算一遍。这个点积是否等于 \(-\sin 1\)？把位置改成 \(5\) 和 \(6\)，点积是多少？若只旋转 \(q\)、不旋转 \(k\)，比较 \((R_0 q)\cdot k\) 和 \((R_1 q)\cdot k\)，并说明点积为什么不再只依赖一个相对位移。
2. 接上两篇。说明 \(R_m\) 正交如何给出 \(\lVert R_m q\rVert = \lVert q\rVert\)。\(q\)、\(k\) 的分量独立、标准正态时，\((R_m q)\cdot(R_n k)\) 的方差为什么仍是 \(d_k\)？写出送进 softmax 的分数。\(m = n\) 时它和 \(q\cdot k\) 是什么关系？再取偶数 \(d_k\) 的 \(Q, K, V\)，比较位置 \(0,\ldots,L-1\) 与全体加一个整数 \(s\) 的两套 \(Q'K'^{\mathsf T}/\sqrt{d_k}\)，以及未旋转的 \(QK^{\mathsf T}/\sqrt{d_k}\)。哪两套相同，加权和 \(V\) 是否跟着相同？若这些分数里出现量级 \(10^3\) 的项，`F.softmax` 还会不会得到有限权重，权重离 one-hot 还有多远？旋转改得了哪一件，LayerNorm / RMSNorm 改的又是哪一件？
