---
type: entity
created: 2026-04-25
updated: 2026-04-25
tags: [aiagents, desktop-pet, opensource, live2d, vrm, vtuber]
sources: 1
aliases: [AIRI, Project AIRI, moeru-ai/airi]
---

# AIRI(Project AIRI)

> 开源 Neuro-sama 复刻;Tauri/Web + Live2D/VRM + 26+ LLM provider 的"AI waifu / 数字伴侣"平台。当前 v0.9.0(2026-04-10),组织 moeru-ai。

## 一句话角色 / 概览

**桌宠 AI 入口议题里最接近的开源参照**。强在 LLM provider 抽象(xsAI 统一封装 26+ 厂商)、形象层(Live2D + VRM 含自动眨眼/视线/idle)、跨平台(Web / 桌面 Electron / 移动 PWA)、agent 真跑通(Minecraft / Factorio)。**短在 MCP**——主仓 `plugins/` 无 mcp-client,姊妹仓 `mcp-launcher` 是独立的 Go 进程,二者尚未在端到端链路里缝起来。

## 核心事实 / 字段速查

| 维度 | 内容 |
|---|---|
| **GitHub** | github.com/moeru-ai/airi(主仓) + github.com/moeru-ai/mcp-launcher(姊妹) |
| **定位** | 开源 Neuro-sama 复刻;数字伴侣;能聊天/玩游戏/看视频 |
| **架构** | pnpm monorepo,`apps/` + `packages/` + `services/` + `plugins/` + `engines/` + `integrations/`(VSCode) |
| **桌面端** | `apps/stage-tamagotchi`(Electron),`pnpm dev:tamagotchi` |
| **Web 端** | `apps/stage-web`,`pnpm dev` |
| **移动端** | `apps/stage-pocket`(PWA) |
| **LLM provider 数** | 原生 26+:OpenAI / Anthropic / Gemini / DeepSeek / 通义 / 豆包 / Kimi / OpenRouter / Ollama / vLLM / SGLang / xAI / Groq / Mistral / 智谱 / 硅基流动 / 阶跃 / 百川 / minimax / 讯飞 / 腾讯混元 / Player2 / ... |
| **形象** | VRM(3D,@pixiv/three-vrm)+ Live2D 双轨,均含自动眨眼/视线/idle |
| **TTS / ASR** | ElevenLabs / Whisper(客户端);WebSpeech / VAD |
| **本地推理** | WebGPU 浏览器内 + Ollama / vLLM / SGLang;HuggingFace Candle |
| **存储** | DuckDB WASM / pglite(浏览器内嵌)+ Drizzle ORM |
| **已有 plugins**(`plugins/`) | airi-plugin-bilibili-laplace / airi-plugin-claude-code / airi-plugin-game-chess / airi-plugin-homeassistant / airi-plugin-web-extension |
| **agent 实现** | Minecraft agent(完整)/ Factorio agent(PoC)/ Telegram + Discord 聊天 |
| **MCP 现状** | ⚠️ **不是一级公民**。文档提"MCP Registry",姊妹仓 mcp-launcher 定位"MCP server 的 Ollama"(Go,99%),但主仓无 mcp-client 实现 |
| **plugin 系统** | 标 WIP;现有 plugins 走目录约定,未见稳定 SDK |
| **记忆层** | "Alaya 记忆层"标 WIP;`@proj-airi/memory-pgvector` 包存在 |
| **打包** | Win/macOS/Linux 可执行;Nix flake 一行 `nix run github:moeru-ai/airi` |
| **License** | 开源(LICENSE 文件,具体协议未在 README 抓到) |
| **release 频率** | 75 个 release,~每 2-3 周一版,API 不稳定 |

> [!warning] 架构包袱
> AIRI 的目标是"复刻 Neuro-sama"——VTuber 直播 + 玩游戏 + 多模态语音是核心戏路。如果你做桌宠是为了"AI 应用入口/生产力工具",这些躯壳对你是负重而非资产。

> [!warning] MCP 接入要自己写
> 当前没有 turnkey 路线。三条路:**(a)** 仿 `airi-plugin-claude-code` 写 `airi-plugin-mcp-client`(干净,可 PR 上游)/ **(b)** 用 mcp-launcher 作 daemon + AIRI 端轻量 client(快但多依赖)/ **(c)** prompt 层模拟(demo 用,不推荐落地)。详见 [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]]。

## 适配本议题(桌宠 AI 入口)的评估

| 维度 | 评 | 说明 |
|---|---|---|
| **LLM 灵活度** | ⭐⭐⭐⭐⭐ | xsAI 抽象做得最好,换模型只改 baseURL |
| **形象层** | ⭐⭐⭐⭐⭐ | Live2D + VRM 完整,自动微动作 |
| **MCP / 工具调用通用性** | ⭐⭐ | agent 是 per-domain 写死,无标准 tool registry |
| **桌面常驻形态** | ⭐⭐⭐ | Electron app,不是"贴桌面爬"的深度 OS 集成 |
| **架构契合度(对"AI 应用入口")** | ⭐⭐ | 多了 VTuber/游戏的负重 |
| **作为 fork 起点** | ⭐⭐⭐ | 可,但要剪掉很多;迭代快上游合并地狱 |
| **作为装配库参考** | ⭐⭐⭐⭐⭐ | 看它怎么做 provider 抽象 / Live2D 集成是最佳教科书 |

## 相关

- [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] — 议题概念,本项目是其最接近的开源实现
- [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] — 与 Open-LLM-VTuber / Fay / VPet 横向对比
- [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] — MCP 接入手册(含 AIRI 三路线)
- [[Wiki/Concepts/AIFoundations/Mcp]] — MCP 协议本身

## 深入阅读

- 主题读本:[[Readers/AIAgents/桌宠 AI 入口的从零方案]] § 4(AIRI 选型评估)与 § 5(为什么不 fork)

## 开放问题 / 矛盾

- AIRI 的 plugin 系统标 WIP——若深度 fork,API 会不稳定。是否能稳到能依赖,需观察后续 1-2 个 release。
- mcp-launcher(姊妹仓)的 MCP server 启动接口尚不清晰(GitHub 页面信息稀疏),不确定能否被外部 host 直接调用,还是只为 AIRI 主仓预留。需后续真实跑一遍验证。
