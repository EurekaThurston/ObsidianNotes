---
type: entity
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, open-source, typescript, convex, simulation, a16z]
sources: 1
aliases: [AI Town, a16z AI Town, ai-town, AI-town]
---

# a16z AI Town

> **[斯坦福小镇](Wiki/Entities/AIAgents/Stanford-smallville)的工业级开源重写**——a16z 投资团队 + Convex CTO 合作的 TypeScript + Convex + PixiJS starter kit,MIT 许可,默认跑本地 llama3。

## 一句话角色

把 Park 2023 的 Python 研究代码改造成"半小时 fork 就能改成自己的 AI 沙盘"的生产级模板。**门槛被一举压低**——催生了"猫猫小镇"等民间二创遍地开花。知乎 p/656007815 那篇源码解读讲的就是本项目。

## 核心事实

| | |
|---|---|
| 仓库 | [github.com/a16z-infra/ai-town](https://github.com/a16z-infra/ai-town) |
| 许可 | MIT |
| 团队 | a16z 的 Yoko Li + Martin Casado + Convex CTO James Cowling |
| 技术栈 | TypeScript(前后端统一)+ **[Convex](https://convex.dev/)**(响应式后端/数据库)+ **PixiJS** 前端 |
| 默认 LLM | **llama3**(本地 Ollama)+ **mxbai-embed-large** 嵌入 |
| 兼容 LLM | Together.ai / 任何 OpenAI 兼容 API(配置可切) |
| 游戏引擎 | 自研 Convex 里的 "simulation engine"(全局共享状态 + 事务支持) |
| 架构文档 | [ARCHITECTURE.md](https://github.com/a16z-infra/ai-town/blob/main/ARCHITECTURE.md) |

## 和斯坦福小镇的差异

| 维度 | Stanford Smallville | a16z AI Town |
|---|---|---|
| 代码定位 | 研究代码 | 生产级 starter kit |
| 语言 | Python + Phaser(JS) | 统一 TypeScript |
| 后端 | 自搭 Python | Convex(开箱共享状态 + 事务 + 多玩家) |
| LLM 默认 | GPT-3.5-turbo(付费) | llama3(本地免费) |
| 部署 | 论文作者实验环境 | 半小时 fork-改-deploy |
| 许可 | 论文研究代码 | MIT,商用可 |
| 目标受众 | 研究者 | 开发者 / 爱好者 / 初创 |

## 技术亮点

- Convex 的**响应式数据库 + 事务**天然适合多 agent + 多人类 input 并发
- 游戏逻辑(`convex/aiTown`)定义世界状态 + 演化规则 + reactive 处理
- LLM 调用抽象成"agent 输入",与人类玩家 input 同形统一
- 典型运行:一张小地图 + 6~10 agent,不追求 Smallville 论文级的 25 agent 规模

## 相关

- [[Wiki/Entities/AIAgents/Stanford-smallville]] — 谱系源头
- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]] — 简化版本继承了 Smallville 的 memory/conversation/plan 大致结构
- [[Wiki/Concepts/AIFoundations/Ai-agent]]、[[Wiki/Concepts/AIFoundations/Harness-engineering]] — 架构上属于"把 agent loop 包成游戏引擎 tick"

## 深入阅读

- 主题读本:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] § 4
- Source:[[Wiki/Sources/AIAgents/Generative-agents-discussion]]

## 开放问题

- 2026 年项目活跃度(2026-01 还有 podcast 讨论,但是否有 1.0 稳定版未知)
- Convex 作为必选后端的锁定代价:自建项目要接 Convex 云服务或 self-host
- 与 Python 原版的 behavioral fidelity 差距(简化版 memory/reflect 没完全复刻)
