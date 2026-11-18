# 图示资源出处与授权说明

本目录中的图示用于 Xin 的 Cline 技术分享研究资料包，仅作学习与引用。请在展示时保留来源链接。本说明不是法律意见。

收集时间：2026-11-18（Asia/Shanghai）。Cline 源码与官方 mermaid 钉在提交 `4cdfb19a5e897b9e75827cbb4f280f322e27e44a`（2026-11-17）。

## 官方仓库 mermaid（只读提取后本地重绘）

- `official/layered-sdk.svg`：来自 `sdk/ARCHITECTURE.md` 中 “Layered Model” mermaid。源：https://github.com/cline/cline/blob/4cdfb19a5e897b9e75827cbb4f280f322e27e44a/sdk/ARCHITECTURE.md
- `official/agents-deps.svg`：来自 `sdk/AGENTS.md` “Dependency Direction” mermaid。源：https://github.com/cline/cline/blob/4cdfb19a5e897b9e75827cbb4f280f322e27e44a/sdk/AGENTS.md
- `official/historical-task-webview.svg`：来自 `.clinerules/cline-overview.md` “Architecture Overview” mermaid。这是历史 VS Code Task / Controller / Webview 地图，不是当前分层运行时的主图。源：https://github.com/cline/cline/blob/4cdfb19a5e897b9e75827cbb4f280f322e27e44a/.clinerules/cline-overview.md

## 据官方文档 / PR 文本重绘（非仓库里现成的 mermaid 文件）

- `official/docs-package-stack.svg`：据 https://docs.cline.bot/sdk/architecture/overview 的 Package Stack ASCII（2026-11-18 抓取 HTTP 200）重绘。
- `official/docs-agent-loop.svg`：据 https://docs.cline.bot/sdk/clinecore 中 Agent 循环 ASCII（2026-11-18 抓取 HTTP 200）重绘。
- `official/docs-hub-spoke.svg`：据 https://docs.cline.bot/sdk/architecture/hub-spoke 的三角色与通信流描述（2026-11-18 抓取 HTTP 200）重绘。
- `official/pr10651-compaction.svg`：据 https://github.com/cline/cline/pull/10651 中的 canonical history / sidecar 文本图（2026-11-18 抓取 HTTP 200）重绘。
- `official/docs-auto-compact-condensing.png`：官方文档配图，来自 https://docs.cline.bot/features/auto-compact ，文件 URL：https://storage.googleapis.com/cline_public_images/docs/assets/condensing.png

## 第三方：David P. S. Marques（Agent Harness Anatomy #3）

来源页：https://dpsm.github.io/agent-harness-anatomy/cline/
作者：David P. S. Marques（页面 meta author）
钉选：cline/cline tag v4.1.22 → f58bc118bdeef1bd2813cd08e00d98bdcda96475（页面写明 2026-09-29）
`dpsm/*.svg` 均从该页的 `figures/` 目录原样下载（2026-11-18 HTTP 200）。

## 第三方社区 mermaid（从公开页面提取源码后本地重绘）

- `community/aigccamp-*.svg`：https://aigccamp.com/source/cline （页面作者未标注；文中写“基于 2026 年中 cline/cline 主仓库”）
- `community/martianlee-*.svg`：https://martianlee.github.io/posts/2026-06-30-cline-architecture （SeongHwa Lee，页面标注 Jun 30, 2026）
- `community/gitdiagram-overview.svg`：https://gitdiagram.com/cline/cline （页面写明 AI 生成概览，last updated 2026-10-12；站点署名 Made by Ahmed Khaleel）
- `community/qingwave-plan-act.svg`：https://qingwave.github.io/cline-prompt/ （页面标注 Dec 2, 2025）；同类流程图亦见于 https://juejin.cn/post/7580180556982747186

`src/*.mmd` 为上述 mermaid 源码副本，便于核对。
