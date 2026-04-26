---
type: concept
created: 2026-04-19
updated: 2026-04-27
tags: [ai, agent, methodology, engineering]
sources: 2
aliases: [Harness, Harness Engineering, 驾驭工程]
---

# Harness Engineering（驾驭工程）

> **如果 LLM 是 CPU，那 Harness 就是操作系统。** 裹在 AI 模型周围的整套软件基础设施——决定 AI 能看什么、能做什么、怎么拆任务、怎么存记忆、什么时候跑测试、什么时候叫人审批。

## 概览

2026 年 2 月这个词从一篇博客出发，几周内变成整个 AI 工程圈的高频术语。它是在 [[Wiki/Concepts/AIFoundations/Ai-agent|Agent]] 成熟之后自然浮现的问题：**光让 AI"能干活"不够，还得让它在护栏里干活**。

关键原则（Mitchell Hashimoto 的定义）：

> **每当发现 Agent 犯错，就花时间设计一个机制，确保它永远不会再犯同样的错。**

## 起源

- **Mitchell Hashimoto**（HashiCorp 联合创始人、Terraform 作者）2026-02 博客《My AI Adoption Journey》首次命名并定义。
- 几天后 **OpenAI** 发布报告：3 名工程师 5 个月不写一行代码，纯靠 Agent（Codex）生成约 100 万行代码，交付真实产品内测版。
- **Martin Fowler** 做了深度分析，归纳为**三大类**（Context Engineering、Architecture Constraints、Garbage Collection）。
- 叠加 OpenAI 强调的 **反馈回路**，业界逐渐形成"**四大支柱**"的说法。

## 四大支柱

### 支柱一：上下文管理（Context Management）

Agent 只要当前任务所需的信息，**不多不少**。

问题：把所有规则塞一个大文件里会失控（会被模型忽略、会污染其他任务、会触发 [[Wiki/Concepts/AIFoundations/Context-window|Context Rot]]）。

解法：**分层上下文 + 渐进式披露**。
- OpenAI 用 `AGENTS.md` 作为动态反馈循环文件
- Anthropic 用 `CLAUDE.md` + 大量分层 README / 频繁更新的进度文件
- 系统级解法：[[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] 的三层加载

### 支柱二：架构约束（Architecture Constraints）

把规则"代码化"——**用机器而不是人来守边界**。

人类盯不住 AI 写的大量代码；机器可以。典型做法：
- 用 Linter 规定"service 层不能反向依赖 controller 层"，违规代码合不进来
- 用类型系统、测试、schema 校验建立自动化边界
- 用 CI/CD 在提交时做最后防线

OpenAI 的经验：**让 AI 自己写 Linter 来约束 AI**——元层次的 Harness。

### 支柱三：反馈回路（Feedback Loops）

AI 写代码 → 跑测试 → 收到报错 → 自动修正 → 再跑测试 → ……

让 AI 从"一次性输出"进化为"自我修复的闭环"。失败信息本身就是下一轮最好的上下文——Linter 的错误消息不只是拦截违规，还在告诉 AI"为什么错了"和"如何修复"。

### 支柱四：熵管理 / 垃圾回收（Entropy Management）

AI 长时间运行会积累"技术债"：重复代码、废弃文件、命名不一致、过时注释。**不定期清理就越跑越烂**。

做法：
- 定期 refactor 作业
- 专门的 cleanup subagent
- CI 中的 dead code / unused import 检查
- 文档与代码的一致性校验

## 一个有力的佐证

**LangChain 的编码 Agent 在 Terminal Bench 2.0 上**，底层模型**一个参数都没改**，只优化 Harness（文档结构、验证回路、追踪系统），排名从全球第 30 跃升至第 5，得分从 52.8% 升到 66.5%（具体数字以原博客为准）。

这件事的核心洞察：

> **决定 Agent 表现好坏的最大变量，往往不是模型本身多聪明，而是模型被放在什么样的环境里。**

这也是 2026 AI 技术栈三层全景的基础——模型层正在商品化，竞争壁垒在 Harness 和 Skills 两层。

## HITL（Human-in-the-Loop）

Harness 的常见组件：在关键决策点（要删数据、要花钱、要对外发消息）**强制让人按确认键**。不是倒退回人工，而是**让人只在真正重要的地方出现**。这是成熟 Agent 系统的标配。

## 相关

- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — Harness 服务的对象
- [[Wiki/Concepts/AIFoundations/Llm|LLM]] — Agent 的大脑;Harness 不改 LLM 本身,只改它周围的环境
- [[Wiki/Concepts/AIFoundations/Multi-agent|Multi-agent / Subagent]] — 支柱一的最强工具(空间维度切上下文)
- [[Wiki/Concepts/AIFoundations/Ralph-pattern|Ralph 循环模式]] — 长任务 harness 的具名落地(时间维度切上下文 + 经验库就是 Hashimoto 定义的最朴素实例)
- [[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] — 支柱一（上下文管理）的核心工具
- [[Wiki/Concepts/AIFoundations/Context-window|Context Window & Context Rot]] — 支柱一的动因
- [[Wiki/Concepts/AIFoundations/Hallucination|幻觉]] — 支柱二和三的动因
- [[Wiki/Concepts/AIFoundations/Mcp|MCP]] — Agent 拿工具的标准协议,Harness 的上下文动态组装多半经过 MCP
- [[Wiki/Concepts/AIFoundations/Reasoning-model|推理模型]] — 推理模型的"隐式 CoT"加剧了 Context 消耗,放大 Harness 的必要性
- [[Wiki/Concepts/Methodology/Vibe-coding|Vibe Coding]] — 演进第一阶段;Harness 是它的反面答卷
- [[Wiki/Syntheses/AIFoundations/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]] — 历史脉络

## 引用来源

- 主题读本(推荐通读):[[Readers/AIFoundations/AI 应用生态全景 2026]]
- 原子 source:
  - [[Wiki/Sources/AIFoundations/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])
  - [[Wiki/Sources/AIFoundations/Ralph-multi-agent-video]] (raw: [[Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜]]) — Ralph 模式 + 经验库机制对 Hashimoto 定义的具体落地

## 开放问题

- Harness 的工程成本/收益边界：什么规模的团队/项目才值得投入？
- Harness 本身的可移植性：某家的 Harness 经验能多大程度迁移到另一家？
- 四大支柱之外是否还有缺漏？"可观测性 / Tracing"经常被讨论，有望成为第五柱。
