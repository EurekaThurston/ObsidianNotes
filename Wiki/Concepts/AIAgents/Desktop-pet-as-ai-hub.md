---
type: concept
created: 2026-04-25
updated: 2026-04-25
tags: [aiagents, desktop-pet, mcp, agent-hub, architecture, reader]
sources: 1
aliases: [桌宠 AI 入口, Desktop Pet AI Hub, 桌宠 Hub]
---

# Desktop Pet as AI Hub(桌宠作为 AI 应用入口)

> 把"桌宠"这一拟人化常驻形象,作为承载所有 AI 应用的统一入口/总线。形态是壳,**真正的第一性原理是 MCP host**。

## 一句话角色 / 概览

桌宠 AI 入口 = **常驻形象层(壳)** + **LLM Hub 层(脑)** + **MCP host 层(总线)**。形象层只管召唤/反馈/陪伴感,Hub 层做模型路由 + 上下文 + 记忆,MCP 层把"未来所有 AI 应用"接成可热插拔的 server。三层独立、可替换前端、桌宠只是 Hub 的一个 view。

> [!abstract]
> 一句话:**桌宠是 MCP host 的人格化前端**。把它当 UI 层做,Hub/MCP 是真正吃业务复杂度的地方。

## 核心事实 / 字段速查

| 维度 | 说明 |
|---|---|
| **形态选择** | Live2D(2D,主流;素材多) / VRM(3D;表情驱动强) / Spine / Sprite。Live2D 是现在性价比最高的起点 |
| **壳技术栈** | **Tauri 2.0**(推荐,体积 10MB,Rust 后端做系统级 hook) > Electron > WPF(Windows-only,贴合度最高) |
| **Hub 必备能力** | LLM provider 抽象 + 流式 + 多轮上下文 + 工具调用桥接 + 长期记忆 |
| **MCP 必备能力** | server 进程管理 + tools 聚合 + tool call 路由 + 配置兼容 Claude Desktop 格式(复用生态) |
| **对话桌宠的关键交互** | 透明窗口 + 点透 + 系统托盘 + 全局快捷键召唤 + 聊天气泡 + (可选)语音 |
| **第一性原理** | "未来所有 AI 应用都是 MCP server,桌宠只是其中一个 host。"——这条决定整个数据流的形状 |

> [!warning] 反模式:把 MCP 当后期功能加
> 桌宠最容易踩的设计陷阱:先做"单体 chatbot + 形象",以为后面再加工具调用就行。**MCP 决定 Hub 的整个数据流形状**(tool 注册、function calling 桥接、上下文回灌),从 P0 就要按 MCP-host 设计,后期改造代价远大于一开始就走对。

> [!warning] 反模式:把 Hub 绑死在桌宠前端
> Hub 应作为独立 sidecar 进程跑(Node / Python / Rust 均可),桌宠通过 IPC 调它。这样以后 CLI / VS Code 插件 / 浏览器扩展可以共享同一个 Hub。**桌宠不是核心,Hub 才是**。

## 与现成项目的关系

| 项目 | 与本概念关系 |
|---|---|
| [[Wiki/Entities/AIAgents/AIRI]] | 最接近的开源参照。形态/Provider 抽象/Live2D 都做到了,但 **MCP 不是一级公民**(姊妹仓 mcp-launcher 独立,主仓 plugins/ 无 mcp client),且背了"Neuro-sama 复刻 + 玩游戏"的额外包袱 |
| Open-LLM-VTuber | 偏 VTuber 直播向,桌宠形态需自己包 |
| Fay | 数字人助理框架,工程偏重 |
| VPet | 传统桌宠,有插件系统但 AI 集成是社区做的,Windows-only |

完整选型矩阵见 [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]]。

## 三档自做路线

| 档 | 做什么 | 工作量(一人业余) | 适用 |
|---|---|---|---|
| **L1 完全从零** | 自写 Live2D 渲染 / LLM client / MCP 协议 | 6-12 月 | 没有理由 |
| **L2 从零做壳 + 装配成熟库** ⭐ | 自写窗口/交互/Hub 架构,渲染/LLM/MCP 全用现成库 | 2-3 月出 v0.1 | 推荐 |
| **L3 fork AIRI** | 接受其架构,定向改 | 1-2 月 | 喜欢其形态 |

L2 详细方案见 [[Readers/AIAgents/桌宠 AI 入口的从零方案]]。

## 相关

- [[Wiki/Concepts/AIFoundations/Mcp]] — MCP 协议本身;桌宠的"总线"标准
- [[Wiki/Concepts/AIFoundations/Ai-agent]] — Agent 概念;桌宠是有人格的 agent host
- [[Wiki/Concepts/AIFoundations/Agent-skills]] — Skills 与 MCP server 的能力打包对比
- [[Wiki/Entities/AIAgents/AIRI]] — 最接近的开源参考
- [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] — 桌宠侧的 MCP host 实施手册
- [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] — 选型矩阵

## 深入阅读

- 主题读本:[[Readers/AIAgents/桌宠 AI 入口的从零方案]]
- 选型综合:[[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]]
- 实施手册:[[Wiki/Syntheses/AIAgents/Mcp-host-implementation]]

## 开放问题 / 矛盾

- **形象与生产力的张力**:Live2D + 语音的"陪伴感"和"高频 AI 工具调用"的"专注效率",在 UX 上可能拉扯——同一只桌宠又当伴侣又当 Cursor 替身,可能两边都不极致。读本 P0-P3 后需重新评估。
- **Hub 跨前端复用的真实价值**:架构上把 Hub 拆 sidecar 是干净的,但"我真会做 CLI / VS Code 插件去复用它吗"取决于本人产能。如不复用,sidecar 是过度工程。
