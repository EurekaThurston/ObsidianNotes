---
type: synthesis
created: 2026-04-25
updated: 2026-04-25
tags: [aiagents, desktop-pet, mcp, stack-selection, comparison]
sources: 1
aliases: [桌宠选型矩阵, Desktop Pet Stack Comparison]
---

# 桌宠 AI 入口的选型矩阵

> 把"做一个桌宠作为 AI 应用入口"按 **路线档(L1/L2/L3)** + **现成项目矩阵** 两个维度铺开。议题概念见 [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]]。

## 1. 路线档分级

| 档 | 做什么 | 工作量(一人业余) | 适用前提 | 评价 |
|---|---|---|---|---|
| **L1 完全从零** | 自写 Live2D 渲染 / LLM client / MCP 协议 | 6-12 月 | — | 没有理由,Live2D / Vercel AI SDK / @modelcontextprotocol/sdk 都很成熟 |
| **L2 从零做壳 + 装配** ⭐ | 自写窗口/交互/Hub 架构,渲染/LLM/MCP 全用现成库 | 2-3 月 v0.1,半年成熟 | 现成项目躯壳不合需求,要 MCP-first | 推荐,对"AI 应用入口"目标契合度最高 |
| **L3 fork AIRI** | 接受 AIRI 架构,定向改 | 1-2 月 | 喜欢 AIRI 的形态,愿意扛上游合并 | 起步快,但背 VTuber/游戏躯壳 |

**L2 装配清单**(你不用自己写的部分):

| 模块 | 选型 | 替代 |
|---|---|---|
| 桌面壳 | **Tauri 2.0**(Rust + WebView,体积 ~10MB) | Electron / WPF |
| 形象渲染 | **pixi-live2d-display** + Cubism SDK | @pixiv/three-vrm(VRM 3D)|
| LLM 抽象 | **Vercel AI SDK**(`ai` + `@ai-sdk/*`)流式/tool calling/structured output | LangChain.js / xsAI / 自写 |
| MCP | **`@modelcontextprotocol/sdk`** 官方 TS SDK | mcp-launcher(Go,前置 daemon) |
| 本地 DB / 记忆 | **SQLite**(better-sqlite3 / Tauri SQL plugin)+ `sqlite-vec` 或 `lancedb` | DuckDB / pglite |
| TTS / ASR | edge-tts(免费)/ Azure / 火山;whisper.cpp / WebSpeech | ElevenLabs(贵) |
| UI 框架 | Vue / React + shadcn / Element Plus | — |
| 配置文件 | **复用 Claude Desktop 的 `claude_desktop_config.json` 格式** | 自定义(不推荐,生态不通用) |

## 2. 现成项目矩阵

| 项目 | 技术栈 | 形态 | LLM provider | MCP | 角色 | 适配"AI 入口" |
|---|---|---|---|---|---|---|
| [[Wiki/Entities/AIAgents/AIRI]] | Tauri/Electron + Vue + WebGPU | 桌面/Web/PWA | ⭐⭐⭐⭐⭐ 26+ 厂商 | ⭐⭐ 无主仓 client | Live2D + VRM 完整 | ⭐⭐⭐ 有 VTuber 包袱 |
| Open-LLM-VTuber | Python + Web | VTuber 直播向 | ⭐⭐⭐⭐ 主流齐 | ⭐⭐ | Live2D | ⭐⭐ 直播向,桌宠形态需自包 |
| Fay | Python + Unity/Web | 数字人助理 | ⭐⭐⭐⭐ | ⭐⭐ | 多模型支持 | ⭐⭐⭐ 工程偏重,UI 杂 |
| VPet | C# WPF | Win 经典桌宠 | ⭐⭐ 社区接 | ❌ | 自家 sprite 动画 | ⭐⭐ Win-only,AI 自接 |
| 自做 L2 | Tauri + Vue + Live2D | 自定 | 由 Vercel AI SDK 决定 | ⭐⭐⭐⭐⭐ MCP-first | Live2D | ⭐⭐⭐⭐⭐ |

## 3. 决策矩阵(按你的目标收敛)

> [!question] 如果你的目标是"未来所有 AI 应用的统一入口",怎么选?

| 你的偏好 | 推荐路线 |
|---|---|
| 想最快看到东西、不在意躯壳里多 30% 不用的功能 | **L3 fork AIRI**,剪掉 Minecraft/Factorio/VTuber 部分 |
| 想要 MCP-first 架构、长期自主权、愿意扛 2-3 月起步期 | **L2 自做** ⭐ |
| 只是想体验"桌宠 + LLM"看看是不是真喜欢这种交互 | **直接装 AIRI v0.9.0 释出版**,试用 1 周再说 |
| 喜欢 Win 桌宠那种"贴桌面爬、点头摇尾"的拟物感 | **VPet + 接 ChatGPT 插件**(社区已有) |
| 想做语音+VTuber 角色直播 | **Open-LLM-VTuber 或 Fay** |

## 4. L2 推荐架构(详见手册)

```
┌──────────────────────────────────────────┐
│  Tauri 前端 (Vue/React + pixi-live2d)     │
│  ├─ 形象层:透明窗口 + 点透 + 托盘 + 快捷键 │
│  └─ 对话层:聊天气泡 / 设置面板             │
├──────────────────────────────────────────┤
│  Tauri Rust 后端                          │
│  └─ 系统级:全局快捷键 / 剪贴板 / 屏幕监听  │
├──────────────────────────────────────────┤
│  Node sidecar (Hub 大脑)                  │
│  ├─ LLM Router (Vercel AI SDK)           │
│  ├─ MCP Manager (官方 SDK) ←── 核心       │
│  ├─ Memory (SQLite + sqlite-vec)         │
│  └─ Event Bus (前端订阅)                  │
└──────────────────────────────────────────┘
        ↓ stdio / SSE / HTTP
┌──────────────────────────────────────────┐
│  N 个 MCP server                          │
│  filesystem / git / 你的特效贴图 / ...    │
└──────────────────────────────────────────┘
```

详细分阶段路线、避坑、实施顺序见 [[Readers/AIAgents/桌宠 AI 入口的从零方案]];MCP 部分单独成手册 [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]]。

## 5. 关键决策原则

1. **MCP-first 不可让步**——它决定 Hub 数据流的形状,不是后期功能
2. **Hub 单独抽 sidecar**——前端可换(桌宠/CLI/VS Code 插件共享)
3. **配置照抄 Claude Desktop 格式**——你装的 MCP server 在 Cursor / Claude / 你的桌宠之间通用,这是最便宜的兼容性投资
4. **形象层和 Hub 严格解耦**——以后想换 VRM / 3D / 像素皮肤都不影响 Hub
5. **不要先做语音和复杂动画**——P5 之前都不碰,核心价值不在那里

## 相关

- [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] — 议题概念
- [[Wiki/Entities/AIAgents/AIRI]] — 最接近的现成项目
- [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] — MCP 接入手册
- [[Wiki/Concepts/AIFoundations/Mcp]] — MCP 协议
- [[Readers/AIAgents/桌宠 AI 入口的从零方案]] — 主题读本
