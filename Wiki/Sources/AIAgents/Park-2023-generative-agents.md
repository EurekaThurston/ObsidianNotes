---
type: source
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, generative-agents, smallville, paper, uist]
source_type: paper
---

# Park et al. 2023 — Generative Agents: Interactive Simulacra of Human Behavior

- **原文**:本仓未托管 PDF,直链 arXiv [2304.03442](https://arxiv.org/abs/2304.03442)(镜像 [3dvar.com/Park2023Generative.pdf](https://3dvar.com/Park2023Generative.pdf))
- **作者**:Joon Sung Park、Joseph C. O'Brien、Carrie J. Cai、Meredith Ringel Morris、Percy Liang、Michael S. Bernstein
- **日期**:2023-04(arXiv);2023-10 UIST Best Paper
- **URL**:<https://arxiv.org/abs/2304.03442>
- **类型**:paper(academic,ACL/UIST 2023)
- **开源代码**:[github.com/joonspk-research/generative_agents](https://github.com/joonspk-research/generative_agents)
- **获取方式**:本 wiki 未全文阅读 PDF。本页内容基于**论文摘要 + 公开报道 + 开源代码 README + Stanford HAI 二级解读**整合;深度字段级引用请回溯原文

## 摘要

提出 **Generative Agents**——用大语言模型(实验用 GPT-3.5-turbo / ChatGPT)驱动的计算 agent,在一个名为 **Smallville** 的沙盘里模拟 **25 个 NPC 的日常生活**:起床做饭、上班、聊天、形成观点、回忆过去、规划未来、自发组织派对。

论文贡献不是"接了个 LLM",而是一套 agent 架构——**Memory Stream + Reflection + Planning**——让 LLM 的单轮响应能力能跨天演化成连贯个性、社交网络和涌现行为。通过用户研究证明其"believability"(可信度)显著高于消融对照,且人类评估者在盲测中难以区分 agent 对话与真人。

## 关键要点

### 架构三件套

- **Memory Stream**(记忆流):所有观察以自然语言 + 时间戳追加式存储。检索时三维度打分:
  - `recency`:指数衰减
  - `importance`:LLM 给每条记忆 1-10 分("吃早饭"低分,"家人去世"高分)
  - `relevance`:当前情境 query 的 embedding 余弦
  - 三分加权 top-k 喂 prompt
- **Reflection**(反思):当最近记忆的 importance 累积过阈值,触发 LLM 从零散观察提炼高阶洞察("Klaus 在读 gentrification 论文"→"Klaus 是个认真的学者")。反思也入记忆流、可被更高阶反思复用 → 层级化
- **Planning**(规划):粗到细:先一天计划 → 小时级 → 分钟级递归。执行时遇事件 reactive 重排

### 实验

- 25 个 NPC,每个有名字、人设、关系网、职业
- 沙盘包含:咖啡馆、酒吧、学校、宿舍、商店、公园……
- 运行时间:论文主实验 2 个游戏日(完整论文 + 社会事件)
- 涌现行为:自发情人节派对、竞选议员、口耳相传信息扩散

### 评测

- "partial context" 消融:单独去掉 Memory/Reflection/Planning 任一都显著降分
- 人类盲评:25 个 agent 对话 vs 真人对话,人类识别率近随机

### 开源状态

- 代码 `joonspk-research/generative_agents`:Python 后端 + Phaser H5 前端,可本地跑
- 数据:25 agent 两天完整 replay 开源(可 replay 观看)
- **不包含**:自定义 Phaser map 编辑器(要改地图得自己来)

## 复现成本(公开报道)

- 论文实验用 GPT-3.5-turbo(不是 GPT-4)
- 每 agent 每 tick 都调 LLM → 25 agent × N tick × 多个 LLM 调用(perceive / retrieve / plan / converse)
- **广泛报道:两天完整实验数千美元 token**
- 国内复现者流传数字:**100 秒游戏时间 ≈ 6 元 RMB**
- 瓶颈不在代码而在钱;改用本地 llama3/Qwen 可显著降成本但 reflection 质量下降

## 涉及实体 / 概念

- [[Wiki/Entities/AIAgents/Joon-sung-park]]
- [[Wiki/Entities/AIAgents/Stanford-smallville]]
- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]](本页的架构抽象)
- [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]](本页开辟的研究范式)

## 与既有 wiki 的关系

- **新增**:AIAgents 主题下第一份学术 source;架构三件套作为独立概念页
- **印证 / 交叉引用**:
  - [[Wiki/Concepts/AIFoundations/Multi-agent]]——多 agent 最极端的"长周期持续运行"场景
  - [[Wiki/Concepts/AIFoundations/Context-window]]——memory stream 是对"上下文有限"的工程解答:把无限历史压到有限 prompt
- **后续工作**:[[Wiki/Sources/AIAgents/Park-2024-1000-agents]](Park 博士论文,从 25 虚构 → 1052 真人)
- **工程化版本**:[[Wiki/Entities/AIAgents/A16z-ai-town]](TypeScript + Convex 重写)
- **配套读本**:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]]

## 开放问题(从 source 角度)

- 本 source 基于摘要 + 二级报道,**没有逐字读过 PDF**。关键细节(具体 prompt、reflection 阈值、planning 递归深度)未核验
- 论文附录据称含完整 prompt 模板——是本议题最有价值的材料,后续若需深挖应下载 PDF 单独 ingest
