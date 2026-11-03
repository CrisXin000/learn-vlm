# Grouped-Query Attention 与 Multi-Query Attention

> 日期：2026-11-03
> 标签：`pytorch` · `gqa` · `mqa` · `kv-cache` · `attention`

[Scaled Dot-Product Attention](pytorch-math-scaled-dot-product-attention.md) 里每个头各有一套 \(Q_h, K_h, V_h\)，输出是 \(\mathrm{softmax}(Q_h K_h^{\mathsf T}/\sqrt{d_k})\,V_h\)。解码时已经算过的 \(K, V\) 留在 KV cache 里，缓存的体积随头数线性涨。Multi-Query Attention（MQA）让所有 query 头共用同一组 \(K, V\)。Grouped-Query Attention（GQA）把 query 头分成 \(G\) 组，组内共享、组间分开，用 \(G\) 把 MQA 和多头注意力（MHA）接成一条线。分组只决定第 \(h\) 个 query 头去点哪一组 key。点完之后仍是上一篇的缩放点积，分数仍是 [Softmax 数值稳定与 Log-Sum-Exp](pytorch-math-softmax-logsumexp.md) 里的 logits。位置旋转仍只作用在 \(Q\) 和 \(K\) 上，见 [旋转位置编码（RoPE）](pytorch-math-rope.md)（`notes/pytorch-math-rope.md` 还在未合并的分支上，当前 `main` 没有这个文件）。上一篇把 `enable_gqa` 留成一句话，这篇把分组、重复和 cache 写开。

## 动机

自回归解码每生成一个 token，都要把新的 key、value 追加进 cache，再让当前 query 去看整段历史。MHA 里第 \(h\) 个头的 cache 是两条长度 \(S\)、宽度 \(d_k\) 与 \(d_v\) 的序列。一层、一条序列、\(H\) 个头，元素个数是

\[
N_{\mathrm{MHA}} = S \cdot H \cdot (d_k + d_v).
\]

\(d_k = d_v = d\) 时就是 \(2HSd\)。层数、batch 和 dtype 的字节数再乘上去。序列一长，cache 比参数还占地方，解码的时间花在把这些 \(K, V\) 读出来。读的量跟着 \(H\) 走，是因为每个 query 头都有自己的 key 头。

MQA（Shazeer，2019）把 key 头和 value 头收成 1 个：所有 query 头去点同一组 \(K, V\)。一层一条序列的 cache 变成 \(S(d_k + d_v)\)，是 MHA 的 \(1/H\)。\(H\) 个 query 头仍然各自不同，它们只能在同一套 key 上挑权重。

GQA（Ainslie 等，2023）取中间。记 key / value 的头数为 \(G = \texttt{num\_kv\_heads}\)，query 头数为 \(H\)。GQA 取

\[
1 < G < H, \qquad H \bmod G = 0.
\]

\(H\) 个 query 头均分成 \(G\) 组，每组

\[
n_{\mathrm{rep}} = H / G
\]

个 query 头共享一组 \(K, V\)。三个端点连在同一个参数上：

| | \(G\) | \(n_{\mathrm{rep}}\) | 谁共用一组 \(K, V\) |
|---|---|---|---|
| MHA | \(H\) | \(1\) | 每个 query 头单独一组 |
| GQA | \(1 < G < H\) | \(H/G\) | 每组 \(n_{\mathrm{rep}}\) 个 query 头 |
| MQA | \(1\) | \(H\) | 全部 query 头 |

cache 的元素个数（一层、batch 为 \(B\)）是

\[
N_{\mathrm{cache}} = B \cdot S \cdot G \cdot (d_k + d_v).
\]

它与 \(G\) 成正比，不随 query 头数 \(H\) 再乘一次。新生成一个 token，这一层追加 \(B \cdot G \cdot (d_k + d_v)\) 个数。

省掉的是缓存里的头数，以及 \(W_K, W_V\) 的输出宽度：从 \(H d\) 收到 \(G d\)。把每组 \(K, V\) 复制成 \(H\) 头再做 \(QK^{\mathsf T}\) 时，乘法次数仍按 \(H\) 个 query 头来算，每个 query 头还是要和长度为 \(S\) 的 key 做点积。融合内核可以让同一组 query 头共用一次对 \(K_g\) 的读取，不必先在显存里放 \(n_{\mathrm{rep}}\) 份副本。两种做法算的是下一节同一个加权和。KV cache 里留下的是复制之前的 \(G\) 头；把复制后的张量存进去，体积回到 \(N_{\mathrm{MHA}}\)。

## 数学

### 头到组的映射

沿用上一篇的记号。query 头 \(h = 0, \ldots, H-1\)，kv 头 \(g = 0, \ldots, G-1\)。每个头的宽度仍是 \(d_k\)、\(d_v\)。缩放只用 \(d_k\)，不用 \(d_{\mathrm{model}}\)，也不用 \(G\)。

\[
Q_h \in \mathbb{R}^{L \times d_k}, \quad
K_g \in \mathbb{R}^{S \times d_k}, \quad
V_g \in \mathbb{R}^{S \times d_v}.
\]

MHA 是 \(G = H\)，\(K_h, V_h\) 只给第 \(h\) 个 query 头。GQA 和 MQA 把下标从 \(h\) 收到组号

\[
g(h) = \left\lfloor \frac{h}{n_{\mathrm{rep}}} \right\rfloor, \qquad n_{\mathrm{rep}} = \frac{H}{G}.
\]

第 \(h\) 个 query 头的输出是标准缩放点积，key 和 value 换成这一组：

\begin{align*}
s^{(h)} &= \frac{Q_h K_{g(h)}^{\mathsf T}}{\sqrt{d_k}}, \\
\alpha^{(h)} &= \mathrm{softmax}(s^{(h)}), \\
o^{(h)} &= \alpha^{(h)} V_{g(h)}.
\end{align*}

softmax 沿 key 的长度 \(S\) 做，每一行非负且和为 1，所以 \(o^{(h)}\) 是 \(\{v_j^{(g(h))}\}\) 的凸组合。同一组的 query 头用同一套 \(K_g, V_g\)。它们的 \(Q_h\) 仍然各写各的，\(s^{(h)}\) 一般互不相同，权重 \(\alpha^{(h)}\) 也各算各的。共享的是 key 和 value，权重还是这一行 query 自己的 softmax。

\(H = 4\)、\(G = 2\) 时 \(n_{\mathrm{rep}} = 2\)：

| query 头 \(h\) | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| kv 头 \(g(h)\) | 0 | 0 | 1 | 1 |

### 重复之后仍是同一条 SDPA

把每组 \(K_g\) 沿头维连续复制 \(n_{\mathrm{rep}}\) 次，得到和 query 一样多的头：

\[
K^{\mathrm{rep}}_{h} = K_{g(h)}, \qquad V^{\mathrm{rep}}_{h} = V_{g(h)}.
\]

头的顺序是每组占连续的 \(n_{\mathrm{rep}}\) 个槽：\(0,0,\ldots,1,1,\ldots\)。这就是 `repeat_interleave`。把整段 \(0, \ldots, G-1\) 再平铺 \(n_{\mathrm{rep}}\) 遍，得到的是 \(0,1,\ldots,0,1,\ldots\)，和 \(g(h)\) 对不上。

复制完以后，上一篇的矩阵形式可以原样用：

\[
\mathrm{Attention}(Q, K^{\mathrm{rep}}, V^{\mathrm{rep}})
= \mathrm{softmax}\!\left(\frac{Q (K^{\mathrm{rep}})^{\mathsf T}}{\sqrt{d_k}}\right) V^{\mathrm{rep}}.
\]

按头切开，第 \(h\) 块就是上面的 \(o^{(h)}\)。因此「第 \(h\) 个头只和 \(K_{g(h)}, V_{g(h)}\) 做 SDPA」与「先 `repeat_interleave`，再对全体做一次 SDPA」是同一个函数。

点积仍发生在 \(\mathbb{R}^{d_k}\) 里。分量独立、方差为 1 时，\(\mathrm{Var}(q \cdot k) = d_k\)，除以 \(\sqrt{d_k}\) 的理由和上一篇相同。\(n_{\mathrm{rep}}\) 不进这个分母。

因果掩码加在分数上，形状按 \((L, S)\) 广播到每个头。位置 \(i\) 看不到 \(j > i\) 时，那些分数写成 \(-\infty\)，权重是 0。掩码不看 \(g(h)\)，分组不改它。

这些分数仍是 logits。query 头 \(h\)、位置 \(i\) 那一行的 log-sum-exp 是

\[
\mathrm{LSE}(s^{(h)}_{i,:}) = \log\sum_{j=1}^{S} \exp(s^{(h)}_{ij}).
\]

在线算法里的 \((m, \ell, o)\) 也是按这一行、沿 key 更新。同一组里的另一个 query 头有自己的一行，不共用这个 LSE。FlashAttention 算的还是这个加权和。GQA 改变的是有多少份不同的 \(K, V\) 要读，不改变一行 softmax 的定义。

### 旋转只进 Q 和 K

RoPE 把位置 \(m\) 的旋转 \(R_m\) 乘在进入点积的 query 和 key 上，value 不转。GQA 里第 \(h\) 个头、查询位置 \(i\)、键位置 \(j\) 的分数是

\[
s^{(h)}_{ij}
= \frac{(R_i q^{(h)}_i) \cdot (R_j k^{(g(h))}_j)}{\sqrt{d_k}}.
\]

\(k^{(g)}\) 先按位置旋转、再按组复制，和先复制再旋转是同一个向量：副本和原头在同一个位置，乘的是同一个 \(R_j\)。\(V_{g(h)}\) 保持原样，输出仍是这些 value 的凸组合。旋转正交，不改变这一头的范数，缩放因子仍是 \(1/\sqrt{d_k}\)。`notes/pytorch-math-rope.md` 尚未并入 `main`，式子和这里用的是同一套 \(R_m\)。

## PyTorch 里怎么做

`torch.nn` 没有名为 `GQA` 或 `MQA` 的模块。`nn.MultiheadAttention` 的构造参数是 `embed_dim` 和 `num_heads`，每个头一套投影，没有 `num_key_value_heads`。

`F.scaled_dot_product_attention` 算的仍是缩放点积。PyTorch 2.15 的签名里有一个默认关闭的开关（文档标成 beta，参数以后可能变）：

```text
scaled_dot_product_attention(
    query, key, value,
    attn_mask=None, dropout_p=0.0, is_causal=False,
    scale=None, enable_gqa=False,
)
```

`query` 的形状是 \((N, \ldots, H_q, L, E)\)，`key` / `value` 是 \((N, \ldots, H_{kv}, S, E)\)（value 的最后一维可以是 \(E_v\)）。输出形状 \((N, \ldots, H_q, L, E_v)\)。默认 `enable_gqa=False`。\(H_q = H_{kv}\) 时就是 MHA。\(H_{kv} = 1\) 时头维长度为 1，矩阵乘可以广播到 \(H_q\)，MQA 不打开开关也能算。\(1 < H_{kv} < H_q\) 且两个头数对不上时，直接相乘会在头维报尺寸不一致。

文档给出的参考实现，在做 \(QK^{\mathsf T}\) 之前把 K、V 重复到与 query 一样多的头：

```python
if enable_gqa:
    key = key.repeat_interleave(query.size(-3) // key.size(-3), -3)
    value = value.repeat_interleave(query.size(-3) // value.size(-3), -3)
```

`-3` 是头所在的轴。重复次数是 \(H_q / H_{kv}\)，要求整除，并且 key 与 value 的头数相同。重复之后仍是

\[
\mathrm{softmax}\!\left(\frac{QK^{\mathsf T}}{\sqrt{E}}\right) V.
\]

`scale` 的默认值仍是 \(1/\sqrt{E}\)，只能写成关键字参数。`is_causal=True` 仍是下三角，和 `attn_mask` 不能同时给。函数不另写一套注意力。

文档把 GQA 写成实验特性：FlashAttention、cuDNN、CUDA 上的 math 内核，以及 NVIDIA CUDA 上的 memory-efficient attention 可以走这条路径；Nested tensor 不支持。参考实现里的 `repeat_interleave` 会把头数物化成 \(H_q\)。CPU 使用与该公式一致的 PyTorch 实现。本篇的例子在 CPU、float32、PyTorch 2.15 上跑过：`enable_gqa=True` 的输出，和先 `repeat_interleave`、再以默认 `enable_gqa=False` 调用，是对齐的。cache 保存重复之前的 \(H_{kv}\) 头。

HuggingFace 的 LLaMA 用配置项 `num_attention_heads` 和 `num_key_value_heads`。组大小是 `num_attention_heads // num_key_value_heads`。其 `repeat_kv` 作用在形状 \((B, H_{kv}, S, d)\) 上：在头维插入一维，展开 \(n_{\mathrm{rep}}\) 倍再 reshape 回去。没有额外的 batch 维时，这个头维就是 SDPA 的 `-3`，数值等于 `repeat_interleave(..., dim=1)`。`Tensor.repeat` 沿头维平铺得到的是整段组号反复出现的那种顺序，和 \(g(h)\) 不一致。

## 可运行示例

CPU、默认 float32、PyTorch 2.x（这里用 2.15）即可。

```python
import math
import torch
import torch.nn.functional as F

torch.manual_seed(0)

# (B, heads, seq, d)。H 个 query 头，G 组 K/V，n_rep = H // G。
B, H, G, L, S, E = 1, 4, 2, 3, 3, 4
n_rep = H // G
Q = torch.randn(B, H, L, E)
K = torch.randn(B, G, S, E)
V = torch.randn(B, G, S, E)
K_mha = torch.randn(B, H, S, E)

def repeat_kv(x, n_rep):
    # (B, G, S, E) -> (B, G * n_rep, S, E)。与 LLaMA 的 repeat_kv 同序。
    # n_rep == 1 时就是 MHA，原样返回。
    if n_rep == 1:
        return x
    b, g, s, e = x.shape
    return x[:, :, None].expand(b, g, n_rep, s, e).reshape(b, g * n_rep, s, e)

K_rep = repeat_kv(K, n_rep)
V_rep = repeat_kv(V, n_rep)
# 期望：K_rep.shape == (1, 4, 3, 4)，与 Q 的头数对齐。
assert K_rep.shape == (B, H, S, E)
assert torch.equal(K_rep, K.repeat_interleave(n_rep, dim=1))
# g(h) = h // n_rep。头 0、1 看见组 0，头 2、3 看见组 1。
assert torch.equal(K_rep[:, 0], K[:, 0]) and torch.equal(K_rep[:, 1], K[:, 0])
assert torch.equal(K_rep[:, 2], K[:, 1]) and not torch.equal(K_rep[:, 0], K_rep[:, 2])
# 沿头维平铺得到的排列和 g(h) 不同。
assert not torch.equal(K.repeat(1, n_rep, 1, 1), K_rep)

scale = 1.0 / math.sqrt(E)
scores_mha = (Q @ K_mha.transpose(-2, -1)) * scale
scores_gqa = (Q @ K_rep.transpose(-2, -1)) * scale
# MHA 与重复后的 GQA，分数形状都是 (B, H, L, S) = (1, 4, 3, 3)。
assert scores_mha.shape == scores_gqa.shape == (B, H, L, S)

out = torch.softmax(scores_gqa, dim=-1) @ V_rep
fused = F.scaled_dot_product_attention(Q, K_rep, V_rep, dropout_p=0.0)
by_flag = F.scaled_dot_product_attention(Q, K, V, dropout_p=0.0, enable_gqa=True)
assert out.shape == (B, H, L, E)
assert torch.allclose(out, fused, atol=1e-5, rtol=1e-5)
assert torch.allclose(out, by_flag, atol=1e-5, rtol=1e-5)

# 按组做：每组 query 只看自己的那一份 K/V，拼起来等于整段 SDPA。
parts = []
for g in range(G):
    sl = slice(g * n_rep, (g + 1) * n_rep)
    k_g = K[:, g : g + 1].expand(B, n_rep, S, E)
    v_g = V[:, g : g + 1].expand(B, n_rep, S, E)
    parts.append(F.scaled_dot_product_attention(Q[:, sl], k_g, v_g, dropout_p=0.0))
assert torch.allclose(torch.cat(parts, dim=1), out, atol=1e-5, rtol=1e-5)

# MQA：唯一的 kv 头在矩阵乘里可以广播。重复到 H 头再做，结果相同。
K1, V1 = K[:, :1], V[:, :1]
mqa_broadcast = F.scaled_dot_product_attention(Q, K1, V1, dropout_p=0.0)
mqa_repeat = F.scaled_dot_product_attention(
    Q, K1.repeat_interleave(H, dim=1), V1.repeat_interleave(H, dim=1), dropout_p=0.0
)
assert mqa_broadcast.shape == (B, H, L, E)
assert torch.allclose(mqa_broadcast, mqa_repeat, atol=1e-5, rtol=1e-5)

# 因果掩码仍是 (L, S) 的下三角，与分组无关。
allow = torch.ones(L, S, dtype=torch.bool).tril()
manual_c = torch.softmax(scores_gqa.masked_fill(~allow, float("-inf")), dim=-1) @ V_rep
causal = F.scaled_dot_product_attention(Q, K_rep, V_rep, is_causal=True, dropout_p=0.0)
assert causal.shape == out.shape
assert torch.allclose(manual_c, causal, atol=1e-5, rtol=1e-5)
```

## 要点

- MHA 是 \(G = H\)，每个 query 头一套 \(K, V\)。MQA 是 \(G = 1\)，全部 query 头共用一套。GQA 是 \(1 < G < H\) 且 \(H\) 能被 \(G\) 整除，第 \(h\) 个头用组 \(g(h) = \lfloor h / n_{\mathrm{rep}} \rfloor\)，\(n_{\mathrm{rep}} = H / G\)。
- 一层的 KV cache 有 \(B \cdot S \cdot G \cdot (d_k + d_v)\) 个元素，随 \(G\) 线性变。新 token 追加的也是 \(G\)，不是 \(H\)。cache 里留复制前的 \(G\) 头。
- 沿头维 `repeat_interleave` 得到 \(K^{\mathrm{rep}}_h = K_{g(h)}\)。之后仍是 \(\mathrm{softmax}(QK^{\mathsf T}/\sqrt{d_k})\,V\)。因果掩码和 \(\sqrt{d_k}\) 不随组大小改。同一组共享 \(K, V\)，不共享 softmax 权重，也不共享这一行的 LSE。
- RoPE 只乘在 \(Q\) 和 \(K\) 上。先旋转再按组复制，与先复制再旋转，在同一位置上是同一个向量。\(V\) 不转。
- PyTorch 2.15 没有 `nn.GQA`。`F.scaled_dot_product_attention` 的公式不变；`enable_gqa=True` 时，参考实现先对 K、V 在头维 `-3` 上做 `repeat_interleave`。LLaMA 风格的配置名是 `num_key_value_heads`。

## 练习

1. 设 batch \(B = 1\)，\(H = 32\)，\(G = 8\)，\(d_k = d_v = 128\)，已缓存长度 \(S = 4096\)，每个元素 2 字节，模型 32 层。分别写出一层里 MHA（\(G = H\)）、这一组 GQA、以及 MQA（\(G = 1\)）的 KV cache 元素个数和字节数。GQA 相对 MHA 的字节比是多少？再生成 1 个 token 时，32 层一共新增多少字节？若有人把 `repeat_interleave` 之后的 \(K, V\) 存进 cache，一层的字节数回到哪一种？
2. 取 \(H = 8\)、\(G = 4\)。列出 \(h = 0, \ldots, 7\) 的 \(g(h)\)。`repeat_interleave` 之后，下标为 5 的那个头槽来自哪个 kv 头？若改成把整段组号平铺 \(n_{\mathrm{rep}}\) 次，这个槽又来自哪个？说明为什么只有前一种排列，等于「每个 query 头只和 \(K_{g(h)}, V_{g(h)}\) 做 \(\mathrm{softmax}(QK^{\mathsf T}/\sqrt{d_k})\,V\)」。同一组里的两个 query 头，在 \(Q\) 不同的时候，注意力权重为什么一般不同？\(\sqrt{d_k}\) 和因果掩码要不要改成依赖 \(G\)？
