# Cline 运行时图示、社区问题与 Issue 研究资料

本目录是 **2026-11-18（Asia/Shanghai）** 研究资料包的原样归档。正文、限制说明、出处，以及 Markdown 中的相对图片路径都保持原样。本目录不补充新的研究结论。

## 文件索引

| 文件 | 内容 |
|---|---|
| [cline-runtime-diagrams-and-issues.html](cline-runtime-diagrams-and-issues.html) | 自包含 HTML。样式与图已写在文件内，不依赖外部脚本或样式表。 |
| [cline-runtime-diagrams-and-issues.md](cline-runtime-diagrams-and-issues.md) | 同一份研究正文。图片使用相对路径 `cline-sharing-assets/...`。 |
| [cline-sharing-assets/](cline-sharing-assets/) | SVG、PNG，以及 Mermaid 源码 `src/*.mmd`。 |
| [cline-sharing-assets/ATTRIBUTION.md](cline-sharing-assets/ATTRIBUTION.md) | 图示来源与授权说明。展示时请保留来源链接。 |

## 快照里要分开看的两件事

收集与核对时间是 **2026-11-18（Asia/Shanghai）**。Cline 静态事实钉在公开提交 `4cdfb19a5e897b9e75827cbb4f280f322e27e44a`（提交时间 2026-11-17 17:13:09 Asia/Shanghai）。

- **当前主图**是 AgentRuntime 分层（`shared → llms → agents → core → apps`，循环在 `AgentRuntime` / `ClineCore`），见正文第 2.1 节。
- **历史 Task 图**（`Task` / `Controller` / `WebviewProvider`，`initiateTaskLoop` / `recursivelyMakeClineRequests`）只作对照，见正文第 2.2 节。正文写明不要把它讲成当前实现。
- 文中 GitHub Issue 的开放、关闭以及 `completed`、`not_planned`、`stale` 等状态，是该日读到的历史快照，不是此后的实时状态。

## 原文中的 `/workspace` 路径

这些路径是当时研究机上的位置，不是本仓库路径：

- `/workspace/cline-runtime-diagrams-and-issues.md` 与 `/workspace/cline-runtime-diagrams-and-issues.html`：即本目录中的同名文件。
- `/workspace/cline-sharing-assets/`：即本目录的 `cline-sharing-assets/`。
- `/workspace/cline-tech-sharing.md`：配套的 40 分钟讲稿，未收入本目录。
- `/workspace/cline`：当时只读查看的 Cline 检出，`HEAD` 为上述钉选提交；未收入本目录。
