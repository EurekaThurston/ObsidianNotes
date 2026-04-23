---
type: source
created: 2026-04-24
updated: 2026-04-24
tags: [ai, code-retrieval, agentic-grep, ue4, artist-tooling, landing-design]
source_type: note
---

# 代码检索与美术问答机器人对话 — Claudian 的 agentic grep 方法论 + 项目级落地设计

- **原文**:[[Raw/Notes/代码检索与美术问答机器人对话]]
- **作者**:Eureka × Claudian(对话式 Q&A)
- **日期**:2026-04-24
- **类型**:note(对话归纳,非逐字)

## 摘要

Eureka 了解到同事使用 "grep" 概念 + Claude Code + OpenClaw 向 UE 项目源码提问检索成功,引出三个递进问题:(1) Claudian 是怎么检索代码的?"grep" 是什么?(2) 能不能做一个给美术的项目代码问答机器人,解决他们对材质参数 / 宏开关 / Niagara 参数靠经验主义的问题?(3) 如果是项目自定义的 symbol,你怎么知道该搜索什么?Claudian 以"agentic grep"命名其检索方式(Grep + Glob + Read 三件套,临场 grep 不走 RAG),在 UE 4.26 stock 上用 `DitheredLODTransition` 做了端到端 demo,并设计了美术向代码问答机器人的完整落地方案(架构、persona、沉淀回路、术语桥梁、部署风险)。关键自省:Claudian 对 UE 原生 symbol 能 grep 得准是因为训练数据先验,对项目自定义 symbol 则完全失效——破解靠"锚点反推 + 约定穷举 + 种子爬行 + 停下追问"四路策略,以及把 wiki 本身作为"术语桥梁 + 长期记忆"。

## 关键要点

- **Agentic grep** = LLM 自己临场 grep 代码库(Grep / Glob / Read 三件套),无 embedding / 无预建索引 / 每次现查。**不是 RAG**
- **vs RAG 的核心差异**:字面匹配 vs 语义近似;零维护 vs 需要 reindex;可追溯 `path:line` vs 可能幻觉片段
- **适用域**:代码(符号化系统)比 RAG 准得多;自然语言文档 RAG 仍有优势
- **核心弱点**:依赖关键词命中。未知 symbol 时破解四策略:
  1. **锚点反推**:UI 可见字符串 → grep `LOCTEXT` / `DisplayName` / `ToolTip`
  2. **约定穷举**:UE 前缀(`b` / `F` / `U` / `A` / `E`)/ shader 宏大写下划线 / CVar `r.XX`
  3. **种子爬行**:命中一个相邻 `#include` / function / 注释都是新 grep 种子
  4. **停下追问**:宁可多问一轮,不能胡答
- **DitheredLODTransition demo**:从 `MaterialInstanceBasePropertyOverrides.h` UI 定义 → `MaterialTemplate.ush` shader 宏合成 → `ClipLODTransition` per-pixel discard → `LocalVertexFactory.ush` per-instance 植被 → `DepthRendering.cpp` stencil 优化路径 → `RendererSettings.h` mobile 性能警告,全链路 `path:line` 证据
- **美术向代码问答机器人**核心工作流:自然语言 → 跨仓 grep → Read 上下文 → 用"讲给美术听"的语气重写 + 证据附上
- **项目已有的弹药库**:CLAUDE.md §9 的三 code root + `Wiki/Sources/<repo>/` 源摘要 = 预消化文档。Karpathy Wiki 方法论的 compounding 效应在此兑现
- **两条腿并跑**:临场 grep(长尾)+ wiki 命中(高频,更精,有人类复核)
- **关键工程坑**:p4 权限 / 机密边界 / "未 grep 到证据不许作答"(禁幻觉)/ 美术词 ≠ 代码 symbol(术语桥梁)
- **POC 最小可验证**:先覆盖材质 static switch 一个子系统,人工整理术语对照表,跑 `/ask-material-artist` 给 3-5 个美术试用,看实际价值再扩
- **元反省**:训练数据先验是"作弊"的一部分,老实承认比装懂重要

## 涉及实体 / 概念

- [[Wiki/Concepts/AIApps/Agentic-grep|Agentic grep]](本次新建)
- [[Wiki/Syntheses/AIApps/Artist-code-qa-bot|给美术做代码问答机器人]](本次新建)
- [[Wiki/Concepts/Methodology/Rag|RAG]](被对比)
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]](compounding 效应被印证)
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]](禁幻觉是核心部署纪律)
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]](persona / 子命令实现路径之一)
- [[Wiki/Entities/AIApps/OpenClaw|OpenClaw]](Eureka 同事的机器人类似栈)
- [[Wiki/Entities/Claudian|Claudian]](自证案例)

## 与既有 wiki 的关系

- **扩展**:[[Wiki/Concepts/Methodology/Rag]] 原本只讲"语义检索"和"无积累"两大属性;本 source 补上"在代码域 agentic grep 更强"这条实战结论
- **印证**:[[Wiki/Concepts/Methodology/Llm-wiki-方法论]] 的 compounding 效应——wiki 越丰富,代码问答机器人越省 grep,越准
- **应用化**:[[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]] 三段论中"Harness 上下文管理"支柱在项目代码场景的具体实例
- **新概念**:[[Wiki/Concepts/AIApps/Agentic-grep]] 命名并沉淀这种检索范式
- **新综合**:[[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] 作为 AIApps 下第一份"项目级落地设计"
- **配套读本**:[[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] — 本 source 对应的线性叙事版,Dithered LOD Transition 作贯穿案例
