---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [ai, agent, open-source, local-deployment]
sources: 1
aliases: [OpenClaw, Clawdbot, 小龙虾]
---

# OpenClaw（小龙虾）

> 开源的**个人 AI Agent 系统**，部署在你自己的电脑或服务器上——一个可以操作你电脑的专属 AI 秘书。

## 概览

原名 Clawdbot，因图标是只龙虾被中文社区昵称"小龙虾"。本质上是一个 **AI 助手网关（Gateway）**：

- 一头接 [[Wiki/Concepts/AIApps/Llm|大语言模型]]
- 另一头接各种输入渠道（网页、飞书、微信、Telegram、Discord……）
- 以及各种工具（浏览器、文件系统、命令行等）

要解决的问题：**现有 AI 助手都被困在"聊天框"里**——你得在一个对话窗口里跟它说话，它既不能主动干活，也不能操作你的电脑。OpenClaw 打破这个壁垒。

## 关键属性

| 维度 | 普通 AI 聊天 | OpenClaw |
|---|---|---|
| **运行位置** | 云端 | 你的本地电脑/服务器 |
| **数据隐私** | 发给第三方 | 本地，你完全掌控 |
| **能力范围** | 只能聊 | 操作电脑、浏览网页、读写文件、执行命令 |
| **主动性** | 被动等问 | **心跳机制**定时主动执行 |
| **记忆** | 每次对话独立 | 长期记忆，跨会话保留 |
| **接入渠道** | 单一网页/App | 飞书、微信、Telegram、Discord… |

## 核心三件套

**记忆 + 主动 + 行动**，是 OpenClaw 相对普通 AI 聊天的根本差异：

- **记忆**（知道该做什么）：它记得你上周提过的项目截止日期
- **主动性**（知道何时做）：心跳机制定时唤醒，主动提醒
- **行动力**（知道如何做）：直接操作飞书、邮件、日历完成任务

## 与飞书的结合

飞书（Lark）承载了大量工作信息。通过飞书官方插件，OpenClaw 可以以"你的身份"操作文档、日历、任务。**过去需要你手动复制粘贴给 AI 的信息，现在 AI 自己去取。**

## 相关

- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — OpenClaw 是个人本地化的 Agent 实现
- [[Wiki/Concepts/AIApps/Mcp|MCP]] — OpenClaw 挂外部工具的标准接口
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — OpenClaw 可挂的知识模块

## 引用来源

- [[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题 / 待补

- 具体的 GitHub 仓库、项目维护者、许可证信息尚未 ingest——v2 文章从扫盲视角描述，未到技术细节。
- 本地部署的硬件成本与推理方案（调云端 API vs 跑本地小模型）。
- 与其他"个人 AI Agent"项目（Open Interpreter、Claude Desktop + MCP 组合等）的对比。
