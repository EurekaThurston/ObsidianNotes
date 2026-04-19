---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [ai, agent, protocol, tooling]
sources: 1
aliases: [MCP, Model Context Protocol, 模型上下文协议]
---

# MCP（Model Context Protocol）

> AI 世界的 **USB-C 接口**——让工具只实现一次，就能被任何 AI 应用调用。

## 概览

2024 年 11 月 Anthropic 推出的开放标准协议，用来解决 AI 与外部工具/数据源之间的"N × M 集成难题"：N 个 AI 模型 × M 个工具 = N × M 套定制集成，开发成本爆炸。

**类比**：USB-C 之前，鼠标 PS/2、打印机并口、手机各种充电线。USB-C 统一了所有。MCP 对 AI 做的是同样的事。

## 为什么需要 MCP（背景）

2023 年 OpenAI 推出 **Function Calling**，让 LLM 可以调外部工具。但**每家 AI 公司的接口格式都不一样**——给 GPT 写的工具用不了 Claude，反过来也一样。MCP 就是为了消除这种重复。

## 三个角色

- **MCP Host（宿主）**：发起请求的 AI 应用（Claude Desktop、Cursor、IDE 等）
- **MCP Client（客户端）**：宿主内部与 Server 保持连接的部分
- **MCP Server（服务器）**：对外提供工具和数据的那一端（文件系统、GitHub、Figma、数据库、自定义业务系统……）

工具商只需写一次 MCP Server，所有支持 MCP 的 AI 都能用。AI 应用只需实现一次 MCP Client，就能挂上全世界的 MCP Server。

## 行业采纳

- OpenAI、Google DeepMind、Cursor、GitHub、Vercel 等陆续支持
- 2025 年后事实上已是行业标准
- 具体治理（是否捐赠给某基金会等组织细节）以 Anthropic 官方公告为准

## 与相关概念的区别

| 概念 | 定位 | 对比 |
|---|---|---|
| **Function Calling** | 各家专有的函数调用 | MCP 之前的做法，各不兼容 |
| **MCP** | 跨厂商标准协议 | 统一接口 |
| **[[Wiki/Concepts/AIApps/Agent-skills|Skill]]** | 模块化专家知识包 | 告诉 AI"怎么做"；MCP 告诉 AI"能做什么" |
| **CLAUDE.md / AGENTS.md** | 项目级规则文件 | 针对当前项目的约定 |

简记：**MCP 是协议（管道），Skill 是知识（内容），CLAUDE.md 是章程（规则）。**

## 相关

- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — MCP 是 Agent 连外部世界的主要手段
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — 与 MCP 互补：Skills 给知识，MCP 给工具
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 把数据"塞"给 AI 的另一条路径（检索），MCP 则是让 AI"主动去取"

## 引用来源

- 主题读本(推荐通读):[[Wiki/Readers/AIApps/AI-primer-v2-读本]]
- 原子 source:[[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- MCP Server 的安全模型：本地 server 有文件系统访问权，远程 server 可能访问敏感 API——权限边界和审计如何做？
- 多 MCP Server 并存时的工具冲突/命名空间问题。
