---
type: entity
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, generative-agents, simulation, sandbox, stanford]
sources: 2
aliases: [Smallville, 斯坦福 AI 小镇, 斯坦福小镇, Stanford AI Town]
---

# 斯坦福小镇(Smallville)

> Park et al. 2023 论文里的**像素风沙盘**——25 个 LLM 驱动的 NPC 在虚拟小镇里起床、上班、聊天、开派对,用于证明生成式智能体能涌现出可信的人类行为。

## 一句话角色

**世界第一个用 LLM 驱动 25 个 agent 持续过日子的沙盘实验**。2023-04 论文发布即火,UIST 2023 Best Paper。后续所有"AI 小镇"项目(a16z AI Town、AgentSims、猫猫小镇 ...)在谱系上都追溯到这里。

## 核心事实

| | |
|---|---|
| 出处 | [[Wiki/Sources/AIAgents/Park-2023-generative-agents\|Park et al. 2023]] |
| 代码仓 | [github.com/joonspk-research/generative_agents](https://github.com/joonspk-research/generative_agents) |
| 技术栈 | Python 后端 + **Phaser H5 游戏引擎**前端 |
| 主 LLM | GPT-3.5-turbo / ChatGPT(非 GPT-4) |
| Agent 数 | 25 |
| 架构 | [[Wiki/Concepts/AIAgents/Generative-agents-architecture\|Memory Stream + Reflection + Planning]] |
| 沙盘内容 | 咖啡馆、酒吧、学校、宿舍、商店、公园、普通住宅 |
| 涌现案例 | 自发情人节派对、竞选议员、口耳相传扩散 |

## 复现要点

- **代码**开源,**能跑**
- **真正瓶颈**是 token 成本:每 agent 每 tick 多次 LLM 调用 × 25 agent 并行 → 25 agent 两天游戏时间 ≈ 数千美元 GPT-3.5
- **国内复现者流传**:模拟 100 秒游戏时间 ≈ 6 元 RMB
- 改场景(换地图 / 人设 / 语言)需要动 Phaser + 空间语义树

## 相关

- [[Wiki/Entities/AIAgents/Joon-sung-park]] — 主作者
- [[Wiki/Entities/AIAgents/A16z-ai-town]] — 工业界 TS/Convex 重写版
- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]] — 三件套架构抽象
- [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]] — 范式定位

## 深入阅读

- 主题读本:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]]
- Source:[[Wiki/Sources/AIAgents/Park-2023-generative-agents]]

## 开放问题

- 前端 Phaser 地图编辑器未随代码开源,换地图成本未知
- 完整 prompt 模板据称在论文附录;本 wiki 未逐字核验
