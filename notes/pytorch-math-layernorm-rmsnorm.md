# LayerNorm 与 RMSNorm

> 日期：2026-10-22
> 标签：`pytorch` · `layernorm` · `rmsnorm` · `数值稳定` · `transformer`

Transformer 的每一块都在注意力或前馈外面套一条残差。残差把子层输出加回去之后，隐藏状态的尺度就不再是 [Scaled Dot-Product Attention](pytorch-math-scaled-dot-product-attention.md) 里「分量方差为 1」的那条假设：\(q\cdot k\) 的方差是 \(d_k\)，除以 \(\sqrt{d_k}\) 才把标准差拉回 1。方差随深度涨上去，这个除法拉不回来，softmax 重新堆成 one-hot；分数再大一点，就靠近 [Softmax 数值稳定与 Log-Sum-Exp](pytorch-math-softmax-logsumexp.md) 里 float16 的 \(\exp\) 大约在 11 溢出的那条线。最大值平移仍能给出有限概率，平移消不掉尖峰。LayerNorm 在被归一化的那些维上减均值、除标准差；RMSNorm 不减均值，只除均方根。PyTorch 2.14 的 `nn.LayerNorm`、`F.layer_norm`、`nn.RMSNorm`、`F.rms_norm` 就是这两个式子。这篇从定义写到接口。

## 动机

注意力的输出是 value 的凸组合，再乘输出投影。凸组合不会把尺度放大到超过 value 本身，输出投影会。前馈是把宽度拉开再投回来的线性层，尺度由权重和输入一起决定。残差把这个向量加回输入：

\[
h \leftarrow h + \mathrm{Sublayer}(h).
\]

上一篇的方差是按「\(q, k\) 的分量独立、均值 0、方差 1」算的，\(\mathrm{Var}(q\cdot k) = d_k\)。常见的线性层初始化把输出分量的方差保持在和输入同一量级：输入方差是 1 时，\(q, k\) 的分量方差约是 1，\(1/\sqrt{d_k}\) 才对准。残差每加一次，分量方差就多一份子层的贡献。设这时的分量标准差是 \(\sigma\) 而不是 1，均值仍是 0，则单项 \(q_t k_t\) 的方差是 \(\sigma^4\)，点积的标准差是 \(\sigma^2\sqrt{d_k}\)，除掉 \(\sqrt{d_k}\) 之后分数的标准差是 \(\sigma^2\)。\(\sigma = 4\)、\(d_k = 64\) 时，这个标准差约是 16，已经越过 float16 里 \(\exp\) 大约 11 的溢出点。`F.softmax` 先减行最大值，权重仍然有限，但质量堆在最大的 key 上，雅可比 \(\alpha_i(\delta_{ij} - \alpha_j)\) 回到上一篇说的接近 0 的区域。

归一化放在残差改写尺度的地方，让下一层的 Q、K 或前馈重新看到受控的二阶矩。两个位置：

- Post-LN。Vaswani 等人把子层写成 \(\mathrm{LayerNorm}(x + \mathrm{Sublayer}(x))\)：先加残差，再归一化。\(\gamma = 1\)、\(\beta = 0\) 时，块的输出均值是 0、方差约是 1，下一块直接吃这个尺度。
- Pre-LN。归一化放进支路：\(x + \mathrm{Sublayer}(\mathrm{Norm}(x))\)。子层看到的输入尺度是固定的，捷径本身是一次不加归一化的加法。很多块加起来，捷径的方差仍可以随深度累加。主路在块内没有被拉回固定尺度，堆叠的出口若还要一份这样的表示，就在外面再放一次 Norm。

放在哪个位置，和用哪一种统计量，是两件独立的事。LayerNorm 和 RMSNorm 都可以填进上面的 \(\mathrm{Norm}\)。语言模型的块里常把这一格换成 RMSNorm：少一次均值归约，也少一个 \(\beta\)。换掉的还有平移不变。进归一化的向量没有一份必须先减掉的大均值时，两个分母接近；有的话，这份均值进 RMS，把其余分量压小。

## 数学

### LayerNorm

文档里的运算来自论文 Layer Normalization。对一个向量 \(x \in \mathbb{R}^n\)（实现里 \(n\) 是 `normalized_shape` 那些维的元素个数，前面的维各自单独算）：

\[
\mu = \frac{1}{n}\sum_{i=1}^{n} x_i,
\qquad
\mathrm{Var}(x) = \frac{1}{n}\sum_{i=1}^{n} (x_i - \mu)^2,
\]

\[
\mathrm{LN}(x) = \frac{x - \mu}{\sqrt{\mathrm{Var}(x) + \epsilon}}.
\]

方差是有偏的，分母是 \(n\) 不是 \(n - 1\)，和 `torch.var(x, correction=0)` 相同。文档把 \(\epsilon\) 加在方差上再开方，不放在开方之后。

这条方差就是二阶矩减掉均值的平方：

\[
\mathrm{Var}(x) = \mathbb{E}[x^2] - \mu^2,
\qquad
\mathrm{Var}(x) + \epsilon = \mathbb{E}[x^2] - \mu^2 + \epsilon.
\]

平移 \(x + c\mathbf{1}\)：均值变成 \(\mu + c\)，偏差 \(x_i - \mu\) 不变，方差不变，输出不变。沿归一化维加常数，LayerNorm 的结果不变。

全常数向量的方差是 0。分子是 0，分母是 \(\sqrt{\epsilon}\)，输出是 0。\(\epsilon\) 让全零向量不会除以 0；它并不把这个向量变成方差为 1 的结果。

正的缩放也不是严格不变：\(a > 0\) 时分母是 \(\sqrt{a^2\mathrm{Var}(x) + \epsilon}\)，只有 \(\epsilon\) 相对 \(a^2\mathrm{Var}(x)\) 可以忽略，才回到 \(\mathrm{LN}(x)\)。\(a < 0\) 时分子变号。RMSNorm 同样如此。

### RMSNorm

PyTorch 文档给的式子没有均值，也没有 \(\beta\)。它写明这是论文 Root Mean Square Layer Normalization 里的运算：

\[
\mathrm{RMS}(x) = \sqrt{\epsilon + \frac{1}{n}\sum_{i=1}^{n} x_i^2},
\qquad
y_i = \frac{x_i}{\mathrm{RMS}(x)}.
\]

根号里是 \(\epsilon\) 加上平方的平均。仿射 \(\gamma\) 放在下一小节，默认是 1。

和 LayerNorm 的分母比，差在 \(\mu^2\)：

\[
\mathrm{RMS}(x)^2 - \big(\mathrm{Var}(x) + \epsilon\big) = \mu^2.
\]

\(\mu = 0\) 且两边用同一个 \(\epsilon\) 时，两个分母相等，不带仿射的输出重合。均值不是 0 时，RMSNorm 的分母把 \(\mu^2\) 算进能量，分子又留着原始的 \(x_i\)，公共偏移不会被减掉。

加上常数 \(c\)：

\[
\mathrm{RMS}(x + c\mathbf{1})^2 = \mathrm{RMS}(x)^2 + 2c\mu + c^2.
\]

分子同时变成 \(x + c\mathbf{1}\)，输出变了。这就是丢掉减均值之后不再有的那条不变性。

把向量写成 \(x = z + c\mathbf{1}\)，\(z\) 的均值是 0。交叉项 \(2c\,\mathbb{E}[z]\) 是 0，所以 \(\mathbb{E}[x^2] = \mathbb{E}[z^2] + c^2\)。\(|c|\) 远大于 \(z\) 的幅度、且 \(\gamma = 1\) 时，\(\mathrm{RMS}(x) \approx |c|\)，

\[
y \approx \mathrm{sign}(c)\,\mathbf{1} + \frac{z}{|c|}.
\]

\(z\) 里留给下一层 Q、K 的起伏被 \(|c|\) 压小。LayerNorm 先减掉 \(c\)，输出只由 \(z\) 的形状决定。全常数向量在 RMSNorm 里约是 \(\mathrm{sign}(c)\,\mathbf{1}\)，不是 0：幅度被 RMS 吃掉，符号留下。

手写时少掉的是一次均值归约和一次广播减法。RMSNorm 只对平方做一次归约，再乘上平方根的倒数。`elementwise_affine=True` 时参数是一份 \(\gamma\)，形状为 `normalized_shape`；LayerNorm 默认还有一份同形状的 \(\beta\)。

### 仿射 γ、β 和 ε

归一化之后还有逐元素仿射。LayerNorm：

\[
y = \mathrm{LN}(x) \odot \gamma + \beta.
\]

RMSNorm 只有乘法：

\[
y_i = \frac{x_i}{\mathrm{RMS}(x)}\,\gamma_i.
\]

\(\gamma\) 初始化为 1，\(\beta\) 初始化为 0，一开始仿射是恒等。\(\gamma\) 让模型把各个分量的尺度重新拉开。\(\beta\) 只存在于 LayerNorm。全常数或全零的输入，归一化那一步是 0，LayerNorm 的输出就是 \(\beta\)。RMSNorm 没有这个加项。

`nn.LayerNorm(..., bias=False)` 去掉的是 \(\beta\)，均值照减。它和 RMSNorm 不是同一个函数：前者对平移不变，后者不对。

\(\epsilon\) 的默认值两边不一样。`nn.LayerNorm` 的 `eps` 默认 \(10^{-5}\)。`nn.RMSNorm` 的 `eps` 默认 `None`，用计算类型（opmath）的机器精度：fp16、bf16 和 fp32 取 `torch.finfo(torch.float32).eps`（约 \(1.19\times 10^{-7}\)），fp64 取 `torch.finfo(torch.float64).eps`。半精度输入的稳定项仍是 float32 的机器精度，不是 float16 那个大约 \(9.8\times 10^{-4}\) 的 eps。

BatchNorm 在每个通道上用一个标量缩放和偏移，统计量来自 batch，`eval()` 用跑出来的 running mean。这里的统计量来自每个样本自己的最后几维，`train()` 和 `eval()` 是同一个函数。文档也写明：LayerNorm 的 \(\gamma\)、\(\beta\) 与 `normalized_shape` 同形，是逐元素的，不是每个通道一个标量。

## PyTorch 里怎么做

统计量落在最后 \(D\) 维上，\(D\) 是 `normalized_shape` 的维数。`normalized_shape = (3, 5)` 时，均值或 RMS 在最后两维的 15 个元素上各算一次。单个整数在模块里收成一元组，只归一化最后一维。Transformer 里这就是隐藏维，batch 和序列位置各自独立。

`F.layer_norm` 和 `F.rms_norm` 的 `normalized_shape` 要传元组或列表，例如 `(D,)`。PyTorch 2.14 里传单个 int 会 `TypeError`。模块的构造函数接受 int。

- `torch.nn.LayerNorm(normalized_shape, eps=1e-5, elementwise_affine=True, bias=True, device=None, dtype=None)`  
  `elementwise_affine=True` 时有 `weight`（\(\gamma\)，初值 1）和 `bias`（\(\beta\)，初值 0）。`bias=False` 去掉 \(\beta\)，只在 `elementwise_affine=True` 时有意义。`elementwise_affine=False` 时两者都没有。前向就是减均值、除 \(\sqrt{\mathrm{Var}+\epsilon}\)、再仿射。方差对应 `torch.var(..., correction=0)`。

- `torch.nn.functional.layer_norm(input, normalized_shape, weight=None, bias=None, eps=1e-5)`  
  `weight` 和 `bias` 为 `None` 时不做仿射。传入时形状要和 `normalized_shape` 一致。

- `torch.nn.RMSNorm(normalized_shape, eps=None, elementwise_affine=True, device=None, dtype=None)`  
  除以 RMS，再乘 \(\gamma\)。没有 `bias` 参数。`elementwise_affine=True` 时只有 `weight`，初值 1。`eps=None` 时用上一节的机器精度。`forward` 调用 `F.rms_norm`。

- `torch.nn.functional.rms_norm(input, normalized_shape, weight=None, eps=None)`  
  `weight=None` 时不乘 \(\gamma\)。

`nn.LayerNorm(D)` 在默认初值下和 `F.layer_norm(x, (D,))` 数值相同：\(\gamma = 1\)、\(\beta = 0\)。`nn.RMSNorm(D)` 和 `F.rms_norm(x, (D,))` 相同。两边的默认 \(\epsilon\) 不同，一边是 \(10^{-5}\)，另一边约是 \(1.19\times 10^{-7}\)。同一条非零均值的向量，两种输出的差别里因此既有 \(\mu\)，也有这份 \(\epsilon\)。要比「均值是 0 时二者重合」，`eps` 要传同一个数。

## 可运行示例

下面这段在 CPU、float32 上即可运行（`F.rms_norm` 自 PyTorch 2.4 起提供；接口按 2.14 文档）。注释里的张量是默认打印精度下的结果。

```python
import math
import torch
import torch.nn.functional as F

# 沿最后一维。方差用有偏估计（除以 n），eps 在根号里。
def naive_layer_norm(x, eps=1e-5):
    mean = x.mean(dim=-1, keepdim=True)
    var = ((x - mean) ** 2).mean(dim=-1, keepdim=True)
    return (x - mean) / torch.sqrt(var + eps)

def naive_rms_norm(x, eps):
    mean_sq = x.pow(2).mean(dim=-1, keepdim=True)
    return x / torch.sqrt(mean_sq + eps)

x = torch.tensor(
    [[1.0, 2.0, 3.0, 4.0],
     [10.0, 10.0, 10.0, 10.0]],
    dtype=torch.float32,
)
# 第一行均值 2.5、有偏方差 1.25。常数行减完均值是 0。
# 期望 F.layer_norm: tensor([[-1.3416, -0.4472,  0.4472,  1.3416],
#                            [ 0.0000,  0.0000,  0.0000,  0.0000]])
ln = naive_layer_norm(x)
ref_ln = F.layer_norm(x, (4,))
assert torch.allclose(ln, ref_ln)
assert torch.allclose(ref_ln[1], torch.zeros(4))
assert torch.allclose(torch.nn.LayerNorm(4)(x), ref_ln)

# eps=None 用 float32 的机器精度。10/sqrt(100+eps) 打印为 1。
# 期望 F.rms_norm: tensor([[0.3651, 0.7303, 1.0954, 1.4606],
#                          [1.0000, 1.0000, 1.0000, 1.0000]])
eps_rms = torch.finfo(torch.float32).eps
ref_rms = F.rms_norm(x, (4,))
assert torch.allclose(naive_rms_norm(x, eps_rms), ref_rms)
assert torch.allclose(ref_rms[1], torch.ones(4), atol=1e-6)
assert torch.allclose(torch.nn.RMSNorm(4)(x), ref_rms)

# 平移只留在 LayerNorm。x+7 的 RMS 第一行约为 [0.8363, 0.9409, 1.0454, 1.1500]。
shifted = x + 7
assert torch.allclose(F.layer_norm(shifted, (4,)), ref_ln)
assert not torch.allclose(F.rms_norm(shifted, (4,)), ref_rms, atol=1e-3)

# 均值是 0 且 eps 相同：两个分母都是 sqrt(E[x^2] + eps)。
centered = x - x.mean(dim=-1, keepdim=True)
assert torch.allclose(
    F.layer_norm(centered, (4,), eps=1e-5),
    F.rms_norm(centered, (4,), eps=1e-5),
)

# correction=1 除以 n-1。n=4 时输出相对 F.layer_norm 缩小 sqrt(3/4)。
var_unbiased = x.var(dim=-1, correction=1, keepdim=True)
bad = (x - x.mean(dim=-1, keepdim=True)) / torch.sqrt(var_unbiased + 1e-5)
assert torch.allclose(bad[0] / ref_ln[0], torch.full((4,), math.sqrt(0.75)), atol=1e-4)

# 1e4 上相差 1、2、3。LayerNorm 还原成对 (0,1,2,3) 的归一化；
# RMSNorm 的分母约是 1e4，输出离 1 不到 2e-4。
offset = torch.tensor([1.0e4, 1.0e4 + 1, 1.0e4 + 2, 1.0e4 + 3])
bare = torch.tensor([0.0, 1.0, 2.0, 3.0])
assert torch.allclose(F.layer_norm(offset, (4,)), F.layer_norm(bare, (4,)))
assert (F.rms_norm(offset, (4,)) - 1).abs().max() < 2e-4

# γ 初值 1，β 初值 0。常数行过 LayerNorm 得到 β。
# bias=False 仍减均值，默认初值下等于 ref_ln，不等于 RMSNorm。
gamma = torch.tensor([1.0, 0.5, 2.0, -1.0])
beta = torch.tensor([0.1, -0.2, 0.3, 0.0])
aff = F.layer_norm(x, (4,), weight=gamma, bias=beta)
# 期望: tensor([[-1.2416, -0.4236,  1.1944, -1.3416],
#               [ 0.1000, -0.2000,  0.3000,  0.0000]])
assert torch.allclose(aff, ref_ln * gamma + beta)
ln_m = torch.nn.LayerNorm(4)
assert torch.equal(ln_m.weight.detach(), torch.ones(4))
assert torch.equal(ln_m.bias.detach(), torch.zeros(4))
with torch.no_grad():
    ln_m.weight.copy_(gamma)
    ln_m.bias.copy_(beta)
ln_m.train()
y_train = ln_m(x).detach()
ln_m.eval()
assert torch.equal(y_train, ln_m(x).detach()) and torch.allclose(ln_m(x), aff)

ln_gamma = torch.nn.LayerNorm(4, bias=False)
assert ln_gamma.bias is None and torch.allclose(ln_gamma(x), ref_ln)
assert not torch.allclose(ln_gamma(x), ref_rms, atol=1e-2)

rms_m = torch.nn.RMSNorm(4)
assert rms_m.eps is None
assert [name for name, _ in rms_m.named_parameters()] == ["weight"]
with torch.no_grad():
    rms_m.weight.copy_(gamma)
assert torch.allclose(rms_m(x), ref_rms * gamma)

# 分量标准差 8、d=64：缩放分数标准差约 63.9，平均最大概率约 0.983。
# LayerNorm 之后标准差约 1.002，平均最大概率约 0.244。
torch.manual_seed(0)
N, S, d = 256, 16, 64
q = torch.randn(N, d) * 8
K = torch.randn(N, S, d) * 8
scaled = torch.einsum("nd,nsd->ns", q, K) / math.sqrt(d)
p = torch.softmax(scaled, dim=-1)
qn = F.layer_norm(q, (d,))
Kn = F.layer_norm(K, (d,))
scaled_n = torch.einsum("nd,nsd->ns", qn, Kn) / math.sqrt(d)
pn = torch.softmax(scaled_n, dim=-1)
assert scaled.std().item() > 50
assert p.amax(dim=-1).mean().item() > 0.95
assert abs(scaled_n.std().item() - 1.0) < 0.05
assert pn.amax(dim=-1).mean().item() < 0.35
```

## 要点

- LayerNorm 是 \((x - \mu) / \sqrt{\mathrm{Var}(x) + \epsilon}\)。方差有偏，除以 \(n\)，和 `torch.var(..., correction=0)` 相同。沿归一化维加常数不改变输出。全常数向量的这一步是 0，再加上 \(\beta\)。
- RMSNorm 是 \(x_i / \sqrt{\epsilon + \mathbb{E}[x^2]}\)，不减均值，没有 \(\beta\)。两个分母的平方差在 \(\mu^2\)；\(\mu = 0\) 且 \(\epsilon\) 相同则输出重合。\(x = z + c\mathbf{1}\) 且 \(|c|\) 很大时，\(z\) 的起伏被压到大约 \(1/|c|\)。
- 少掉的是一次均值归约、一次减法和一份 \(\beta\)。交出去的是平移不变。均值已经接近 0 时两个分母接近；均值很大时，不减均值会把剩余变化吃进 RMS。`LayerNorm(..., bias=False)` 只去掉 \(\beta\)，均值仍减。
- 残差把子层的尺度加进隐藏状态。分量标准差变成 \(\sigma\) 之后，缩放点积的标准差是 \(\sigma^2\)，\(1/\sqrt{d_k}\) 不再把分数留在 softmax 梯度还在的区域。Post-LN 归一化残差和；Pre-LN 归一化子层的输入，捷径保持不归一化的求和。
- `F.layer_norm` 的 `eps` 默认 \(10^{-5}\)，`F.rms_norm` 的 `eps` 默认 `None`（fp32 / fp16 / bf16 用 float32 的机器精度）。函数接口的 `normalized_shape` 传元组。`train()` 和 `eval()` 都用当前输入自己的统计量。

## 练习

1. 设 \(x = (1,\ 2,\ 3,\ 4)\)，LayerNorm 的 \(\epsilon = 10^{-5}\)。手算 \(\mu\)、有偏方差和 \(\mathrm{LN}(x)\)（四位小数），再算 \(\mathbb{E}[x^2]\) 和 \(\mathrm{RMS}(x)\)（RMS 的 \(\epsilon\) 取 float32 的机器精度，先说明它会不会改变四位小数）。把每个分量加 10，哪一个输出不变？把 \(x\) 换成 \(x - \mu\)，并且两边用同一个 \(\epsilon\)，两个输出是否重合？用 \(\mathrm{Var}(x) = \mathbb{E}[x^2] - \mu^2\) 指出分母差在哪一项。若方差改成 `torch.var(..., correction=1)`，\(n = 4\) 时输出相对 `F.layer_norm` 差一个什么因子？
2. 接上两篇。隐藏状态的分量独立、均值 0、方差 \(\sigma^2\)，Q、K 投影把输出方差保持在同一量级。说明缩放后的分数 \((q\cdot k)/\sqrt{d_k}\) 的标准差是 \(\sigma^2\)。取 \(d_k = 64\)：\(\sigma = 1\)（刚经过 \(\gamma = 1\) 的 Norm）和 \(\sigma = 4\)（残差把标准差抬到 4）各是多少？\(\sigma = 4\) 有没有越过 float16 里 \(\exp\) 大约 11 的溢出点？`F.softmax` 还会不会给出有限权重，这些权重离 one-hot 还有多远？再设同一位置的向量是 \(z + c\mathbf{1}\)，\(z\) 均值 0、方差 1。LayerNorm 的输出为什么不依赖 \(c\)？写出 \(\mathrm{RMS}(z + c\mathbf{1})^2\)，说明 \(|c|\) 很大时 RMSNorm 怎样把 \(z\) 的起伏送进下一层的 Q、K。
