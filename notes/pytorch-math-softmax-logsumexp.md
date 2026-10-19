# Softmax 数值稳定与 Log-Sum-Exp

> 日期：2026-10-18
> 标签：`pytorch` · `softmax` · `log-sum-exp` · `数值稳定` · `cross-entropy`

分类和语言模型的训练损失几乎都写在 logits 上。logits 没有上界，直接套 \(\exp\) 会在 float32 里溢出、在 float16 里更早溢出，损失变成 NaN，梯度跟着报废。PyTorch 的 `F.softmax`、`F.log_softmax` 和 `nn.CrossEntropyLoss` 用的是同一条恒等式：减去最大值之后再做指数归一化；对数概率和交叉熵则写成 \(x - \mathrm{LSE}(x)\)，不先物化概率。这篇把这条恒等式从定义推到接口。

## 动机

Softmax 把一个实数向量变成概率：

\[
\mathrm{softmax}(x)_i = \frac{e^{x_i}}{\sum_j e^{x_j}}
\]

分子分母都要算指数。IEEE-754 的可表示范围是有限的：

| dtype | 最大有限值 | \(\ln(\text{最大值})\) | 朴素 \(\exp\) 的大致溢出点 |
|---|---|---|---|
| float32 | \(\approx 3.40\times 10^{38}\) | \(\approx 88.72\) | \(x \ge 89\) 变成 \(+\infty\) |
| float16 | \(65504\) | \(\approx 11.09\) | \(x \ge 12\) 变成 \(+\infty\) |

溢出之后分子分母都是 \(+\infty\)，\(\infty/\infty\) 得到 NaN。另一头，全体 logits 都很负时，每一项 \(\exp\) 下溢成 0，分母为 0，\(0/0\) 同样是 NaN。

训练里这不是边角情形。最后一层线性层的输出没有被约束到一个小区间；学习率偏大、初始化偏大、或者模型对某个类非常有把握时，logit 到几十、几百都很常见。float16 甚至在 logit 刚过 11 就会溢出。损失里出现一个 NaN，反向传播会把参数更新成 NaN，这一轮之后的训练没有意义。

还有一层更隐蔽：就算 softmax 本身因为先减了最大值而没有溢出，极小的概率在 float32 里仍然会变成精确的 0。再对这个 0 取 \(\log\)，损失是 \(+\infty\)，梯度是 NaN。交叉熵要的是 \(\log\mathrm{softmax}\)，应当直接用 \(x - \mathrm{LSE}(x)\) 计算，跳过「先得到概率、再取对数」这一步。

## 数学

### 平移不变

对任意常数 \(c\)，

\[
\mathrm{softmax}(x - c\mathbf{1})_i
= \frac{e^{x_i - c}}{\sum_j e^{x_j - c}}
= \frac{e^{x_i}\,e^{-c}}{e^{-c}\sum_j e^{x_j}}
= \mathrm{softmax}(x)_i.
\]

分子分母同时乘 \(e^{-c}\)，比值不变。因此可以自由选择 \(c\)，不改变 softmax 的值。

取 \(c = m = \max_j x_j\)。平移后的每个分量 \(x_i - m \le 0\)，其中至少有一个等于 0，于是

\[
e^{x_i - m} \in (0, 1], \qquad \sum_j e^{x_j - m} \in [1, C].
\]

最大的一项是 \(e^0 = 1\)，不会溢出；分母至少是 1，不会出现 \(0/0\)。比最大值低大约 20 以上的项，相对 float32 里一个 \(O(1)\) 的和已经小于一个 ulp；低过大约 104 则下溢成精确的 0。丢掉这些项不改变可表示的 softmax。

均值也能当 \(c\)，但均值不保证平移后的最大值 \(\le 0\)。只要还有分量比均值高出 89 以上，\(\exp\) 照样溢出。所以实现里用的是最大值，不是均值。

### LogSumExp

对数配分函数（log-sum-exp）

\[
\mathrm{LSE}(x) = \log\sum_j e^{x_j}.
\]

把同一个 \(m = \max_j x_j\) 提出来：

\begin{align*}
\mathrm{LSE}(x)
&= \log\sum_j e^{(x_j - m) + m} \\
&= \log\Bigl(e^{m}\sum_j e^{x_j - m}\Bigr) \\
&= m + \log\sum_j e^{x_j - m}.
\end{align*}

右边的 \(\log\) 里，和落在 \([1, C]\)，对数有定义而且不大。这就是 log-sum-exp 技巧。

### 从 LSE 到 log_softmax，再到交叉熵

\[
\log\mathrm{softmax}(x)_i
= x_i - \mathrm{LSE}(x)
= (x_i - m) - \log\sum_j e^{x_j - m}.
\]

两步形式 \(\log(\mathrm{softmax}(x)_i)\) 与上式在实数上相等，在浮点里不相等：softmax 把远小于最大值的分量舍入成 0 之后，对数不再有定义。第二行先减去 \(m\)，再减去一个落在 \([0, \log C]\) 里的对数，得到的是一个大的负数。第一行若先把 \(\mathrm{LSE}(x)\) 算成一个 \(O(|x|)\) 的数再从 \(x_i\) 里减去，尾数会被大数相消吃掉。

类别下标为 \(y\) 时，负对数似然就是把这一项变号：

\[
\mathrm{NLL}(\log p,\ y) = -\log p_y.
\]

交叉熵直接写在 logits 上：

\[
\mathrm{CE}(x, y)
= -\log\mathrm{softmax}(x)_y
= -x_y + \mathrm{LSE}(x).
\]

模型把质量放在别的类上、正确答案的 logit 比最大值低 \(\Delta\) 时，\(\mathrm{CE} \approx \Delta\)。差距是 1000，损失就是大约 1000，有限，梯度也有限。

对 \(x_i\) 求导（\(y\) 是硬标签）：

\[
\frac{\partial \mathrm{LSE}}{\partial x_i} = \mathrm{softmax}(x)_i,
\qquad
\frac{\partial \mathrm{CE}}{\partial x_i} = \mathrm{softmax}(x)_i - \mathbf{1}[i = y].
\]

每个分量都落在 \([-1, 1]\) 里，全体分量之和为 0。损失可以很大，对 logits 的梯度不会跟着爆炸。在 logits 上做交叉熵，梯度在概率下溢之后仍然是这条有界的差；先 softmax 再对概率取对数时，\(\log 0\) 会让损失和梯度一起变成非有限值。

## PyTorch 里怎么做

公开接口和上面的式子一一对应。`dim` 要显式传入；省略 `dim` 的写法已经弃用。对形状 \((N, C)\) 的分类 logits，`dim=-1`，每一行单独减自己的最大值、单独归一化。

- `torch.nn.functional.softmax(input, dim, dtype=None)`  
  沿 `dim` 做 softmax。稳定算法就是上一节的平移：减去该维上的最大值，再 \(\exp\)，再除以和。`dtype` 会在计算前把输入转成指定类型，文档说明它用来避免低精度溢出。返回值在该维上非负且和为 1。它的输出不能直接送进 `NLLLoss`：`NLLLoss` 要的是对数概率。

- `torch.nn.functional.log_softmax(input, dim, dtype=None)`  
  沿 `dim` 计算 \(\log\mathrm{softmax}\)。文档写明：它和 \(\log(\mathrm{softmax}(x))\) 在数学上等价，但把两步拆开更慢，而且数值不稳定；这个函数用另一种写法同时把前向和梯度算对。那种写法就是 \(x - \mathrm{LSE}(x)\)，并且 LSE 用最大值平移。先减最大值再相减，比「先得到一个 \(O(|x|)\) 的 LSE、再从很大的 logit 里减去它」少掉一次大数抵消。

- `torch.logsumexp(input, dim, keepdim=False)`  
  文档写明这个计算做了数值稳定处理，用的就是 \(m + \log\sum_j\exp(x_j - m)\)。

- `torch.nn.CrossEntropyLoss`（以及 `F.cross_entropy`）  
  输入是**未归一化的 logits**，不要求非负，也不要求和为 1。目标是类别下标（`long`，范围 \([0, C)\)）时，文档给出的逐样本损失就是

  \[
  l_n = -\log\frac{\exp(x_{n,y_n})}{\sum_c \exp(x_{n,c})},
  \]

  并说明这一情形**等价于**先做 `LogSoftmax`，再做 `NLLLoss`。模块把这两步合成一次，调用方传入 logits 即可。无 batch 的输入形状可以是 \((C,)\)，配套的目标是标量下标。

logits 里已经出现 \(+\infty\) 时，平移救不回来：最大值是无穷，指数仍然溢出，`F.softmax` 得到 NaN。技巧假设 logits 有限。

## 可运行示例

下面这段在 CPU、默认 float32 上即可运行（PyTorch 2.x）。注释里的张量是默认打印精度下的结果。

```python
import torch
import torch.nn.functional as F

# float32: exp(89) 起溢出为 +inf。这组 logits 远超这个门槛。
logits = torch.tensor([1000.0, 1001.0, 1002.0])

naive = torch.exp(logits) / torch.exp(logits).sum()
# 分子分母都是 +inf。期望：tensor([nan, nan, nan])

shifted = logits - logits.max()                 # [-2, -1, 0]
stable = torch.exp(shifted) / torch.exp(shifted).sum()
ref = F.softmax(logits, dim=-1)
# 两者都是有限概率，最大质量在最后一类。
# 期望：tensor([0.0900, 0.2447, 0.6652])
assert torch.isnan(naive).all()
assert torch.allclose(stable, ref)

# 全体为大负数：朴素 exp 下溢成 0，0/0 → nan；减最大值后仍是普通 softmax。
neg = torch.tensor([-1000.0, -1001.0, -1002.0])
naive_neg = torch.exp(neg) / torch.exp(neg).sum()
# 期望 naive_neg: tensor([nan, nan, nan])
# 期望 F.softmax: tensor([0.6652, 0.2447, 0.0900])
assert torch.isnan(naive_neg).all()
assert torch.allclose(F.softmax(neg, dim=-1), stable.flip(0))

# 正确类的概率在 float32 里放不下。
# 稳定 softmax 会把该项变成精确 0，再 log 就是 inf，梯度是 nan。
# log_softmax / cross_entropy 保持在对数域，损失等于 logit 差距。
wide = torch.tensor([0.0, 1000.0])
probs = F.softmax(wide, dim=-1)                 # tensor([0., 1.])
broken = torch.log(probs)                        # tensor([-inf, 0.])
log_probs = F.log_softmax(wide, dim=-1)          # tensor([-1000., 0.])
loss = F.cross_entropy(wide, torch.tensor(0))    # tensor(1000.)
nll = F.nll_loss(log_probs, torch.tensor(0))
assert torch.isinf(broken[0])
assert torch.allclose(log_probs, torch.tensor([-1000.0, 0.0]))
assert torch.allclose(loss, nll) and torch.allclose(loss, torch.tensor(1000.0))

wide_g = wide.detach().requires_grad_(True)
F.cross_entropy(wide_g, torch.tensor(0)).backward()
# ∂CE/∂x = softmax - one_hot ≈ [-1, 1]，损失虽是 1000，梯度仍有界。
assert torch.allclose(wide_g.grad, torch.tensor([-1.0, 1.0]))

# 同一公式的两种浮点求值顺序。先减 max 的 F.log_softmax 更准。
ref64 = torch.log_softmax(logits.double(), dim=-1)
direct = torch.log_softmax(logits, dim=-1).double()
via_lse = (logits - torch.logsumexp(logits, dim=-1)).double()
# direct 相对 float64 的误差约 1e-7；via_lse 约 1e-5（大数相减吃掉尾数）。
assert (direct - ref64).abs().max() < 1e-6
assert (via_lse - ref64).abs().max() > 1e-6

# float16 的 exp 在 12 就已经是 inf。F.softmax 先平移，结果仍有限。
half = torch.tensor([10.0, 11.0, 12.0], dtype=torch.float16)
half_naive = torch.exp(half) / torch.exp(half).sum()
# 期望 torch.exp(half): tensor([22032., 59872., inf], dtype=torch.float16)
# 期望 half_naive:     tensor([0., 0., nan], dtype=torch.float16)
# 期望 F.softmax:      tensor([0.0900, 0.2448, 0.6650], dtype=torch.float16)
assert torch.isinf(torch.exp(half)).any()
assert not torch.isfinite(half_naive).all()
assert torch.isfinite(F.softmax(half, dim=-1)).all()
```

`logits = [1000, 1001, 1002]` 上，`F.cross_entropy(logits, torch.tensor(2))` 与 `F.nll_loss(F.log_softmax(logits, dim=-1), torch.tensor(2))` 都打印为 `tensor(0.4076)`，并且 `torch.allclose` 为真。这就是文档里「`CrossEntropyLoss` = `LogSoftmax` + `NLLLoss`」在数值上的样子。

## 要点

- 朴素 softmax 在 float32 中大约从 logit \(89\) 起得到 NaN，在 float16 中大约从 \(12\) 起得到 NaN；全体很大的负数则会 \(0/0\)。
- \(\mathrm{softmax}(x) = \mathrm{softmax}(x - m)\)，\(m = \max x\) 把指数的输入压到 \((-\infty, 0]\)，分母落在 \([1, C]\)。
- \(\mathrm{LSE}(x) = m + \log\sum_j e^{x_j - m}\)，\(\log\mathrm{softmax}(x)_i = x_i - \mathrm{LSE}(x)\)。交叉熵是 \(-\,x_y + \mathrm{LSE}(x)\)。
- `F.softmax` 做最大值平移；`F.log_softmax` 在对数域里完成同一件事，避免「概率下溢成 0 再取对数」。`nn.CrossEntropyLoss` 在类别下标目标下等价于 `LogSoftmax` 接 `NLLLoss`，输入保持为 logits。
- 硬标签交叉熵对 logits 的梯度是 \(\mathrm{softmax}(x) - \mathrm{onehot}(y)\)，每个分量都在 \([-1, 1]\) 里。损失可以很大，梯度不会因为 logit 的尺度一起爆炸。

## 练习

1. 设 \(x = (0,\ 200)\)，用 float32 心算两种平移：\(c = \mathrm{mean}(x)\) 和 \(c = \max(x)\)。哪一种的 \(\exp\) 仍然溢出？溢出的那一项大约是多少？
2. 取一批形状 \((N, C)\)、含有量级 \(10^3\) 的 logits，以及类别下标 `target`。比较这三个量：`F.cross_entropy(logits, target)`、`F.nll_loss(F.log_softmax(logits, dim=-1), target)`、`F.nll_loss(torch.log(F.softmax(logits, dim=-1)), target)`。哪两个一致，哪一个变成 inf 或 nan？再用同一批 logits 先做一次 `softmax`，然后送进 `F.cross_entropy`，说明损失为什么不再等于上面的第一个量。
