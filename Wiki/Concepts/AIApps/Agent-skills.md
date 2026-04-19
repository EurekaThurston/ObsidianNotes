---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [ai, agent, skills, modularity]
sources: 1
aliases: [Agent Skills, Skills, Skill, 智能体技能]
---

# Agent Skills（智能体技能）

> 把专业知识和工作流程**打包成一个可复用的模块**。挂上之后，AI 瞬间从"通才"变成"某个领域的专家"。

## 概览

每次和 AI 协作都要重复输入一大堆规范、术语、流程——效率低且不一致。Agent Skills 的思路：**一次写好，到处复用**。

本质上一个 Skill 就是：**一个带 `SKILL.md` 指令文件的文件夹**，可能还包含模板、脚本、参考资料。

**一句话**：Skill 不直接解决问题，而是通过注入合适的知识和流程，**把 AI 变成解决这个问题的专家**。

**对美术/设计读者的类比**：Skill 就像 Photoshop 的"动作面板（Actions）"或 After Effects 的预设——把一套繁琐流程一键调出来，换个素材照样跑。

## 核心设计：渐进式披露（Progressive Disclosure）

这是 Skills 最巧妙的地方，直接针对 [[Wiki/Concepts/AIApps/Context-window|Context Rot]] 这个问题设计。

类比 **Google Maps 导航**——不会一开始就把全程每个细节念出来，而是先给大方向，快到路口才报详细指令。

**三层加载**：

| 层次 | 何时加载 | 内容 | 资源占用 |
|---|---|---|---|
| **第 1 层：元数据** | Agent 启动时 | Skill 名字 + 一句话简介 | 极低 |
| **第 2 层：指令** | AI 判断任务相关时 | `SKILL.md` 完整内容 | 中等 |
| **第 3 层：资源** | AI 执行中按需读 | 脚本、模板、参考文档 | 按需 |

**好处**：**你可以挂几十上百个 Skill 也不影响性能**——平时只占极少 token 维护索引，真要用才展开。理论上一个 Skill 包可以打包无限量的知识。

## 行业标准化

2025 年 Anthropic 公布 Agent Skills 规范后，Claude Code、Cursor、OpenAI Codex、GitHub Copilot、Vercel 等主流 AI 编程工具陆续支持或正在支持这一格式，正在成为跨工具可移植的开放标准。

（具体官方资源以 Anthropic 工程博客及 GitHub 规范仓库为准。）

## Skills vs MCP vs CLAUDE.md

| 维度 | CLAUDE.md | Skills | [[Wiki/Concepts/AIApps/Mcp\|MCP]] |
|---|---|---|---|
| 本质 | 项目级规范文件 | 模块化专家知识包 | 工具接口协议 |
| 加载时机 | 启动时全量加载 | 按需加载 | 按需调用 |
| 解决什么 | 这个项目怎么做事 | 某个领域的专业知识复用 | AI 与外部系统的通信 |
| 可复用性 | 仅限当前项目 | 跨项目、跨工具 | 跨模型、跨应用 |

简记：**CLAUDE.md 是项目章程；Skills 是专业手册；MCP 是通信协议。**

## 相关

- [[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]] — 渐进式披露的动因
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] — Skills 是支柱一"上下文管理"的核心工具
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — Skills 是给 Agent 挂的知识模块
- [[Wiki/Concepts/AIApps/Mcp|MCP]] — 与 Skills 互补：MCP 给工具，Skills 给知识

## 引用来源

- [[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- Skills 的安全边界：第三方 Skill 里可以藏 prompt injection 吗？
- 跨工具可移植性的实际程度——各家对 SKILL.md 格式的"方言"差异。
- Skill 的发现 / 分发机制：类似 npm 的生态会不会成型？
