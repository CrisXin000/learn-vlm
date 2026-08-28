# LLaVA 学习问答记录

> 论文: [Visual Instruction Tuning](https://arxiv.org/abs/2304.08485) (Liu et al., 2023, NeurIPS 2023 Oral)

---

> 记录于 2026-08-27

## Q: LLaVA 的输入数据是人类标注的吗？

### 答案：指令数据是 GPT-4 合成的，且 GPT-4 没见过图片

```
COCO 图像 ──┬─ 人类写的 caption ──┐
            └─ 人类标的 boxes ────┴→ 拼成纯文本 → 喂给 GPT-4（纯文本版）→ 158K 指令数据
```

- 论文原文："we leverage language-only GPT-4 or ChatGPT as the strong teacher (both accept only text as input)"；Table 1 注："the visual image is not used to prompt GPT"。
- 三类数据：对话 58K + 详细描述 23K + 复杂推理 77K = 158K。
- GPT-4 = 文字推演者：基于 caption + boxes 的视觉摘要生成"像看着图回答"的指令。

### 人类标注只出现在两处

1. **原料层**：COCO caption 和 bounding boxes（人类视觉知识的来源）。
2. **种子层**：每种数据类型手工设计几个示例做 few-shot 种子——论文原话 "the only human annotations we have during data collection"。生成过程零人工。

### 为什么不用人工标注

人工标注慢且标准模糊（"time-consuming and less well-defined"）；LLM 当 teacher 便宜可扩展（当时已有研究显示 GPT 文本标注超过众包）。消融：GPT-4 稳定优于 ChatGPT，尤其空间推理。

### 关键局限

数据质量被 caption 天花板锁死：caption 没提到的视觉细节（遮挡、图中文字等）GPT-4 无从得知。LLaVA 视觉知识上界 = COCO caption 表达力；LLaVA-1.5 换更大规模描述数据正是抬天花板。

### 历史意义

"LLM-as-teacher + 合成数据"范式早期的标志性应用：人类只提供原料和种子。此后几乎所有 VLM 指令数据（ShareGPT4V、GPT-4V 蒸馏集）都走这条路。

---

> 记录于 2026-08-27

## Q: 为什么要设计两种 Context 和三种 Response 方式？

### 两种 Context：解决"怎么把图给纯文本 GPT-4 看"

约束：GPT-4 只接受文本 → 图像必须符号化（论文原话 "encode the image as an LLM-recognizable sequence"）。

| Context | 编码什么 | 缺失后果 |
|---|---|---|
| Captions（5 条） | 语义"是什么"（场景/动作/关系） | 只有 boxes 时不知画面里发生什么 |
| Boxes（归一化坐标） | 空间"在哪里"（类别+位置） | 只有 caption 时无法生成位置/相对关系问答 |

互补：caption 给场景语义，boxes 给空间结构（附录 prompt 明确要求生成 relative positions 类问题——没有 boxes 造不出来）。5 条 caption = 同一场景多视角。

### 三种 Response：按"缺什么补什么"补能力维度

被否掉的朴素方案：`Human: 问题 Assistant: caption`——原文批它 "lacks diversity and in-depth reasoning"。

| 类型（数量） | 能力维度 | 教什么 |
|---|---|---|
| Conversation（58K） | 交互性/多样性 | 多轮对话、助手语态；物体类型/计数/动作/位置/相对关系 + 背景知识类复杂问题 |
| Detailed description（23K） | 完整性 | 说全图里信息——zero-shot 详细描述能力的直接数据源 |
| Complex reasoning（77K） | 深度 | step-by-step CoT——服务于 ScienceQA（LLaVA 92.53% vs GPT-4 2-shot 82.69%） |

数量分布反映取舍：推理最重（看图→理解的关键一跃），对话居中（助手核心形态），描述最少（单向输出难度最低）。

### 共同逻辑：格式统一，内容多样

三种 Response 用同一种对话格式、要求三种截然不同的行为 → 模型学会"读指令 → 切换行为模式"，映射越泛化 zero-shot 越强。两设计合起来定义"视觉指令跟随"：Context 决定视觉信息上界，Response 决定行为模式覆盖度。

---

> 记录于 2026-08-27

## Q: caption 是从何而来的？

### 答案：COCO 数据集自带的人工标注，LLaVA 一条没标

- COCO（Lin et al., ECCV 2014）创建时由众包工人（MTurk）标注 caption：每图 5 个不同工人各写 1 条一句话描述（标注指令：描述图中重要部分）。
- "5 条 caption" = 5 个不同人的视角，非同一人换说法——论文 "describe the visual scene from various perspectives" 的来历。
- 附录实例（同一张装行李图）：5 条分别侧重物体/地点/动作——信息相同、表述不同。
- LLaVA 直接借用（论文原话 "we use COCO images [31]"）。

### 完整链条

```
2014 众包工人 → COCO caption(5/图) + boxes(80类) → 2023 LLaVA 借用 → GPT-4 转写 → 158K 指令数据
```

### "零人工标注"的相对性 + 天花板做实

- LLaVA 自己没有雇人标注，但站在 COCO 众包标注的肩膀上；真正零人工的只有 GPT-4 转写环。
- COCO caption 只有一句话 ×5 条，颜色/遮挡/图中文字等细节常缺失 → GPT-4 只能在 5 句话的信息量内转写。天花板不在 GPT-4，而在上游 caption 的信息密度——LLaVA-1.5 换更大规模详细数据正是为此。

---

> 记录于 2026-08-27

## Q: 根据 caption 和 bbox 生成 response 时，用什么样的 prompt？

### 结构（论文附录 F / Table 13 伪代码）

三部分：system 指令 + few-shot 种子示例（手工 (context, response) 对）+ 当前 query（新图 context）。

### system 指令各片段的设计意图

| 片段 | 意图 |
|---|---|
| "You are an AI visual assistant... seeing a single image" | 语态设定：回答像在看图而非文本转述，决定训练数据风格 → LLaVA 对话风格 |
| "provided with five sentences, describing the same image" | 输入格式声明：5 句话 = 同一张图的描述 |
| "Only include questions that have definite answers... Do not ask any question that cannot be answered confidently" | 确定性约束（最关键）：只准生成可验证的问答，防 GPT-4 自由发挥产生幻觉数据（= 训练噪声） |
| 枚举问题维度（types/counting/actions/locations/relative positions） | 多样性保证 |
| "Provide detailed answers... reasoning steps" | 深度保证：为 CoT 数据定调 |

### few-shot 与 query

- few-shot 种子 = 数据收集过程中"唯一的人工标注"，示范输入输出格式。
- query = 5 条 caption + boxes 列表拼接（如 `person: [0.681, 0.242, 0.774, 0.694], ...`）。
- Detailed description / Complex reasoning 的 prompt 在代码库，结构相同，仅任务措辞不同。

### 可迁移的 prompt 骨架

"角色设定 → 输入声明 → 质量约束 → 维度枚举 → few-shot 示范"——后来所有 LLM 蒸馏数据（含 GPT-4V 蒸馏集）的 prompt 模板骨架。

---

<!-- 新的 Q&A 会追加在下面 -->
