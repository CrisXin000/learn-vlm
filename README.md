# VLM Learn

系统学习视觉语言模型（Vision-Language Models）的个人学习项目。

## 学习路线

按论文发展脉络精读，从奠基工作到现代 VLM 组装范式：

| 论文 | 年份 | 主题 | 状态 |
|---|---|---|---|
| [CLIP](https://arxiv.org/abs/2103.00020)（Radford et al.） | 2021 | 对比图文预训练 + zero-shot 分类 | ✅ 完成 |
| [Flamingo](https://arxiv.org/abs/2204.14198)（Alayrac et al.） | 2022 | 冻结双塔 + 轻量桥接，视觉 in-context few-shot | ✅ 完成 |
| [LLaVA: Visual Instruction Tuning](https://arxiv.org/abs/2304.08485)（Liu et al.） | 2023 | 架构简化 + 视觉指令微调 | 🔄 进行中 |

待定方向：SigLIP / SigLIP2、LLaVA 后续版本、native unified multimodal（TUNA 等）。

## 目录结构

```
notes/
├── clip/
│   └── qa-log.md        # CLIP 学习问答记录
├── flamingo/
│   └── qa-log.md        # Flamingo 学习问答记录
├── llava/
│   └── qa-log.md        # LLaVA 学习问答记录
├── vlm/
│   └── rl-training.md   # VLM 的 RL 训练（专题笔记）
├── pytorch-math-softmax-logsumexp.md  # Softmax 数值稳定与 Log-Sum-Exp
└── pytorch-math-scaled-dot-product-attention.md  # 缩放点积注意力
```

## 学习笔记

论文问答之外的数学与框架笔记：

- [Softmax 数值稳定与 Log-Sum-Exp](notes/pytorch-math-softmax-logsumexp.md)：朴素 `exp` 为何溢出，以及 `F.softmax`、`F.log_softmax`、`nn.CrossEntropyLoss` 如何保持稳定（2026-10-18）
- [Scaled Dot-Product Attention](notes/pytorch-math-scaled-dot-product-attention.md)：从 `QK^T / √d_k` 到 softmax 再乘 `V`，以及它和 FlashAttention 在数学上相同、在访存上不同（2026-10-19）

## 学习方式

- **问答驱动**：逐段精读论文，提出问题 → 教学式解答 → Q&A 追加到对应论文的 `qa-log.md`
- **原文核对**：答案尽量以论文原文为准（关键数字、图表、附录细节直接从论文 PDF 全文核对）
- **跨论文串联**：每个新论文都接在前一篇的历史链条上（如 CLIP 的视觉塔 → Flamingo 的桥接范式 → LLaVA 的简化配方）

## 已覆盖的关键概念（截至 2026-08）

- **CLIP**：对比预训练对齐机制、文本空间传递的相似性形成、zero-shot 数据流、权重共享、三种下游用法（zero-shot / linear probe / fine-tuning）、评估与排查、有效鲁棒性、数据重叠分析、CLIP 在现代 VLM 中的角色
- **Flamingo**：GATED XATTN-DENSE（tanh 门控零初始化）、Perceiver Resampler（64 固定 token）、Q/K/V 方向选择、视觉 K/V 缓存、M3W 交错数据、few-shot 范式、历史意义
- **LLaVA**：GPT-4 合成指令数据（caption + boxes 符号化、三种 Response 设计、prompt 工程）、数据链路完整闭环
