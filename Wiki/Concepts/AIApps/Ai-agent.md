---
type: concept
created: 2026-04-19
updated: 2026-04-21
tags: [ai, agent, automation]
sources: 2
aliases: [Agent, AI Agent, 智能体]
---

# AI Agent（智能体）

> 给 [[Wiki/Concepts/AIApps/Llm|LLM]] "装上手脚"——让它不只能说话，还能**做事**。核心模式是一个循环：思考 → 行动 → 观察结果 → 再思考，直到任务完成。

## 概览

纯 LLM 只是个"会说话的大脑"。Agent 把它和工具（文件、API、浏览器、命令行……）连起来，让它可以在环境中循环地执行多步骤任务。

**对比**：

- **纯聊天**：你问"帮我查明天天气"，AI 说"我没有联网功能"。
- **Agent**：你说"帮我查明天天气，如果下雨就提醒我带伞并写进日历"——它自己调天气 API、判断下雨、调日历 API 创建事件、告诉你搞定了。

## Agent Loop（核心循环）

1. **接收任务**
2. **思考推理**、制定计划（可能用 [[Wiki/Concepts/AIApps/Reasoning-model|推理模型]]）
3. **调用工具**（搜索 / 读写文件 / 调 API / 执行命令……）
4. **观察结果**
5. **判断完成与否**——没完成回到 2；完成就输出

这个循环看似简单，实际上意味着 AI 从"客服"进化成了"能完成项目的实习生"。后面出现的 [[Wiki/Concepts/AIApps/Mcp|MCP]]、[[Wiki/Concepts/AIApps/Agent-skills|Skills]]、[[Wiki/Concepts/AIApps/Harness-engineering|Harness]]，都是为了让这个循环跑得又稳又好。

## 三代演进

| 时间 | 阶段 | 特征 |
|---|---|---|
| 2022–2023 | AI Chat（聊天） | AI 只给建议，人工复制粘贴执行 |
| 2024 | AI Composer（编排） | AI 能直接改代码/文档，每步需人确认 |
| 2025–2026 | AI Agent（智能体） | AI 自动读、改、测试、提交，人只做最终审核 |

## Subagent（多 Agent 协作）

2025 年起主流做法：**主 Agent 拆任务、派给多个 Subagent 各自专攻**，最后主 Agent 汇总。典型实现：Claude Code 的 subagent 机制、CrewAI、AutoGen。

**第一性原理是"上下文隔离"，不是分工并行**——子 agent 在一次性干净上下文里读 40 文件、生成 80K token 中间思考，最后只返回 300 字结论，主 agent 上下文只长 300 字。把"有效上下文"从单窗口 200K 放大到 N × 200K。

完整展开（包工头隐喻、链路拓扑、派活 prompt 工程、四个次级收益、Anthropic 观察的 agent 数与质量非线性关系）见独立概念页 → [[Wiki/Concepts/AIApps/Multi-agent|Multi-agent / Subagent 架构]]。

## 与相关概念的区别

| 概念 | 是什么 | 比喻 |
|---|---|---|
| **LLM** | 只会说话的大脑 | CPU |
| **Agent** | 装了手脚、会循环执行的 LLM | 一个能干活的实习生 |
| **[[Wiki/Concepts/AIApps/Harness-engineering|Harness]]** | 围绕 Agent 的约束和基础设施 | 操作系统 |
| **[[Wiki/Concepts/AIApps/Agent-skills|Skill]]** | 专业知识包 | 实习生手上的操作手册 |
| **[[Wiki/Concepts/AIApps/Mcp|MCP]]** | Agent 调工具的标准接口 | USB-C |

## 相关

- [[Wiki/Concepts/AIApps/Llm|LLM]] — Agent 的底层大脑
- [[Wiki/Concepts/AIApps/Multi-agent|Multi-agent / Subagent 架构]] — 主 Agent 派活给子 Agent 的完整模型
- [[Wiki/Concepts/AIApps/Mcp|MCP]] — Agent 调外部工具的协议
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] — 围绕 Agent 的工程方法论
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — 给 Agent 挂的专家模块
- [[Wiki/Entities/AIApps/OpenClaw|OpenClaw]] — 面向个人的本地化 Agent 系统
- [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|三段论演进]] — Agent 出现后 Prompt Engineering 不够用，催生了后两阶段

## 引用来源

- 主题读本(推荐通读):[[Readers/AIApps/AI 应用生态全景 2026]]
- 原子 source:[[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])
- 原子 source(subagent 深度):[[Wiki/Sources/AIApps/Multi-agent-conversation]] (raw: [[Raw/Notes/Multi-agent 对话]])
- 主题读本(subagent 专题):[[Readers/AIApps/为什么上下文有限反而必须切多 Agent]]

## 开放问题

- Agent 自主性边界：到什么程度必须有 HITL？不同风险等级任务的审核点怎么设计？
- 多 Agent 协作的失败模式研究还不成熟——主 Agent 错误分工、子 Agent 互相"甩锅"的实际案例。
