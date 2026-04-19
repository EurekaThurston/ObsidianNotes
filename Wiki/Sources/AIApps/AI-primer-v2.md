---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [ai, primer, methodology, agent, harness, skills, mcp]
source_type: article
---

# AI 应用技术发展脉络与核心概念扫盲手册 v2

- **原文**：[[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]
- **作者**：Eureka × Claude（Eureka 提供素材与框架，Claude 执笔）
- **日期**：2026-04-19
- **类型**：article（面向零基础读者的综合扫盲）
- **上游素材**：
  - B 站视频 BV1zSDMBUE5o（作者：堂吉诃德拉曼查的英豪）
  - 配套飞书文档（feishu.cn/wiki/WBMfwiNkfi6uNFkRtXdcavDzn0e）
- **v1 原稿**：已删除（2026-04-19）；v2 为本仓正式收录版

## 摘要

面向"对 AI 完全没概念"的读者（含美术/设计/策划等非开发角色），系统梳理 2022–2026 年 AI 应用生态的主线：从 LLM 基础（词语接龙本质、Transformer、token、上下文、三步训练）→ 使用 AI 的三大必知怪癖（幻觉、上下文腐烂、输出不稳定）→ 2024-2026 的新面孔（推理模型、多模态）→ 从聊天到 Agent 的能力跃迁 → AI 连接外部世界的三把钥匙（Function Calling / MCP / RAG）→ 工程方法论三段论演进（Prompt → Context → Harness Engineering）→ Agent Skills 的模块化知识打包 → OpenClaw 代表的个人本地化 Agent 系统 → Vibe/Spec/Harness Coding 三种编程范式 → 2026 AI 技术栈三层全景（Model / Harness / Skills）。核心主张：**模型层正在商品化，真正的价值差异在 Harness 和 Skills 两层**；对任何角色使用 AI 的第一纪律是"AI 给的具体事实必须验证"。

## 关键要点

- **LLM 本质是词语接龙**，不是"知道"与"不知道"的认知——理解这点可消解一大半困惑。
- **三步训练（Pretrain → SFT → RLHF）** 决定了模型的能力与"性格"；不同公司 AI 的风格差异主要源于 RLHF。
- **幻觉是结构性问题**，不会被下一代模型消灭。这是后续 RAG / Harness / HITL 等机制的根本动机。
- **Context Rot**：上下文窗口虽然变长，但塞满会让 AI 变笨——这是 Skills 渐进式披露的设计动机。
- **推理模型（o1、DeepSeek-R1、Extended Thinking）** 是 2024 末以来的大变化，"先打草稿再答题"显著提升数学/逻辑/编程正确率。
- **多模态**已经是旗舰模型标配——对美术/设计岗位意义重大。
- **Agent Loop = 思考→行动→观察→再思考**；Subagent 式分工是 2025 年的主流做法。
- **MCP 是 AI 世界的 USB-C**：Host / Client / Server 三角色，已成为事实标准。
- **RAG 的检索靠 Embedding**：把文本变成数字向量找"语义相似"而非关键词匹配。
- **三段论**：Prompt Engineering（怎么问）→ Context Engineering（让 AI 看什么，Karpathy 提出）→ Harness Engineering（在什么条件下跑，Hashimoto 2026-02 命名）。马的比喻：口令 → 地图 → 缰绳+马鞍+护栏+赛道。
- **Harness 四大支柱**：上下文管理、架构约束、反馈回路、熵管理（Fowler 归三类 + OpenAI 的反馈回路）。
- **LangChain Terminal Bench 实验**：底层模型不变，只改 Harness，排名从第 30 升至第 5——证明"环境 > 模型"。
- **Skills 渐进式披露三层**：元数据（始终加载）→ SKILL.md（相关时）→ 资源文件（执行中）；类比 Google Maps 导航。
- **OpenClaw（"小龙虾"）**：本地部署的个人 AI Agent 系统；核心三件套 = 记忆 + 主动 + 行动。
- **Vibe Coding 由 Andrej Karpathy 2025-02 命名**（v1 误归 YC，v2 已修正）。
- **2026 技术栈三层**：Model（正在商品化）/ Harness（差异化主战场）/ Skills（跨工具专业资产）。
- **用户第一纪律**：AI 给的具体事实（数字、链接、引用、API 名）必须验证一遍再用。

## 涉及实体 / 概念

**本次新建**：
- [[Wiki/Concepts/AIApps/Llm|LLM]]
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]]
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]]
- [[Wiki/Concepts/AIApps/Reasoning-model|推理模型]]
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]]
- [[Wiki/Concepts/AIApps/Mcp|MCP]]
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]]
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]]
- [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]]
- [[Wiki/Entities/AIApps/OpenClaw|OpenClaw]]

**更新的既有页**：
- [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] — 加 Vibe Coding 命名、Context Engineering 倡导
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 补 Embedding 简释 + 与 Agent/MCP 的交叉引用

**提及但未建页（按需再建）**：
Transformer、Token、Embedding、Function Calling、Multimodal、Subagent、HITL、CoT、Vibe/Spec Coding、Linter、CI/CD、Mitchell Hashimoto、Martin Fowler

## 与既有 wiki 的关系

- **印证**：[[Wiki/Concepts/Methodology/Rag|RAG]] 的定位——文章也把 RAG 视为"开卷考试"式机制，和既有页观点一致；同时补出"Embedding 是检索的底层机制"这一 v1 缺失点。
- **新增主题**：wiki 第 4 个主题 `AIApps`（AI 应用生态），与既有 Methodology / Niagara / AIArt 三主题并列独立。
- **对 [[Wiki/Entities/Methodology/Karpathy|Karpathy]] 的扩展**：v2 明确了他在 AI 工程方法论演进中的双重贡献——提出 Context Engineering 概念、命名 Vibe Coding 术语。
- **矛盾**：v1 原稿里有 3 处事实瑕疵（Vibe Coding 归属 YC、Fowler 三大类 vs 四大支柱自相矛盾、MCP 捐赠细节不准），v2 已修正。

## 开放问题 / 待追踪

- **MCP 治理细节**：MCP 是否已真正捐赠给基金会、哪个基金会——原始材料（飞书文档）有此陈述但 v2 采取了模糊表述，待官方公告核实。
- **`agentskills.io` 域名真伪**：v1 提到这个域名，v2 改为"Anthropic 工程博客/GitHub 规范仓库"。如需要精确引用，需人类核实。
- **LangChain Terminal Bench 实验具体数字**（第 30→5、52.8%→66.5%）：v2 保留了数字但注明"以原博客为准"。
- **推理模型细节**：推理模型"打草稿"具体机制（显式 CoT 训练 vs 强化学习 vs 混合），本文只做用户视角解释，未深入训练范式。
