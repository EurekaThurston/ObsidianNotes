---
type: concept
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, simulation, social-science, research-paradigm]
sources: 2
aliases: [LLM-driven ABM, Agent-Based Social Simulation, 生成式社会模拟, LLM agent 社会模拟]
---

# Agent-Based Social Simulation(LLM 驱动的社会模拟)

> 用 LLM 驱动的 agent 群体模拟真实人类行为与社会现象的研究范式——从 2023 Smallville 的"25 虚构 NPC 能不能像人"起步,到 2024 Park 的"1052 真美国人能不能被 agent 复现"成熟为可对照真人 ground truth 的定量方法。

## 一句话定义

把**每个 agent = 一个 LLM + 持久化人格上下文 + 记忆/规划机制**,让多个 agent 互动,观察**群体级涌现现象**(扩散、极化、共识、协作);并**用真人数据校准** agent 的保真度。

## 为什么这个范式重要

经典 ABM(agent-based model)有两个长期痛点:

1. **agent 行为规则要手工写死**:"遇到 X 就 Y"、"阈值 Z 则 R"——真实人类决策的灵活度远超规则能捕获
2. **agent 和真人的对应关系模糊**:ABM 模拟的是"抽象消费者 / 抽象投票人",无法预测具体个体或可识别群体

**LLM driven 解的是什么**:

- agent 决策不再是硬编码规则——LLM 看 agent 的 persona + 记忆上下文,自然语言给出"该做什么"。灵活度来自语言模型
- 2024 Park 的升级:**每个 agent 直接 ground 到一个真人的 2 小时访谈**——agent↔真人的对应可验证

## 两条演进主线

### 主线 A:HCI demo → 涌现行为研究

[[Wiki/Entities/AIAgents/Stanford-smallville|Smallville]] (Park 2023) + [[Wiki/Entities/AIAgents/A16z-ai-town|a16z AI Town]] + AgentSims 等:

- 研究"LLM agent 能不能**涌现**可信人类行为"
- 评测靠**主观 believability**(人类盲测判断 agent 对话 vs 真人对话)
- 价值:证明 LLM 是可行的 agent 驱动引擎;方法学上为后续铺路

### 主线 B:数字孪生 → 真人对照社会科学

[[Wiki/Sources/AIAgents/Park-2024-1000-agents|Park 2024]] (1000 人论文):

- 研究"LLM agent 能不能**复现**特定真人或特定人口分布"
- 评测靠**真人问卷 ground truth 对照**(General Social Survey 85% 匹配)
- 价值:低成本预跑政策 / 内容推荐 / 公共健康干预,让 LLM simulation 进入严肃社会科学工具箱

## 典型下游应用方向

| 领域 | 用法 | 代表研究 |
|---|---|---|
| 公共政策 | 预跑政策推行,看群体反应 | Park 2024 + HAI 后续项目 |
| 经济学 | 拍卖 / 博弈 / 市场微结构模拟 | SmartSim, EconAgent 等 |
| 虚假信息扩散 | 谣言 / 阴谋论在 agent 社交网络里的传播动力学 | 陆续出现,尚无公认代表作 |
| 游戏 NPC | 商用化:会生活、会成长的 NPC | a16z AI Town 衍生、Inworld 等 |
| 心理 / 陪伴 | agent 作为长期社交对象 | Character.AI 同源,但架构不同 |

## 关键工程约束(为什么这范式贵)

- 每 agent 每 tick 都要调 LLM,乘法爆炸
- 100 agent 模拟一周游戏时间 → token 预算按万美元起
- 优化方向(社区当前在探索):
  - memory 压缩 / 分层(只让最重要的记忆走大模型)
  - 异步 tick(agent 不必每帧同步更新)
  - 小模型 + 大模型分工(routine 用 Haiku 级,reflection / key 决策用 Opus 级)
  - 本地开源模型(llama3 / Qwen)替代闭源 API,降单 token 成本

## 跟传统 agent 技术的区别

| 维度 | 经典 ABM(如 NetLogo) | LLM agent social simulation |
|---|---|---|
| 决策 | 手写规则 | LLM prompt 驱动 |
| 语言 | agent 不说话 / 用符号 | 自然语言全程 |
| 扩展 | 加规则=加代码 | 改 persona / 改 prompt |
| 成本 | 几乎为零 | token 乘法爆炸 |
| 保真度 | 低(抽象个体) | 可校准到真人级(2024 突破) |
| 可解释 | 规则明确 | **黑箱 + 部分可解释**(记忆流可读) |

## 和本 wiki 其他议题的关系

- **vs [[Wiki/Concepts/AIFoundations/Multi-agent|工具型 multi-agent]]**:工具型 multi-agent 是"主 agent 派活"解上下文瓶颈;social simulation 是"多 agent 共生演化"观察群体现象。**第一性原理 / 拓扑 / 评测标准全不同**
- **vs [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]]**:后者是"单一任务的垂直落地";social simulation 是"开放式群体涌现研究"。两者同在 AIAgents 主题下但服务对象迥异——本 wiki 的 AIAgents 主题**同时容纳这两种形态**(工业项目级落地 + 学术多 agent 模拟研究)

## 相关

- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]] — 这个范式目前最经典的架构模板
- [[Wiki/Entities/AIAgents/Joon-sung-park]] — 范式推动者
- [[Wiki/Entities/AIAgents/Stanford-smallville]]、[[Wiki/Entities/AIAgents/A16z-ai-town]] — 标志性项目
- [[Wiki/Concepts/AIFoundations/Multi-agent]] — 对照面(工具型多 agent)
- [[Wiki/Concepts/AIFoundations/Hallucination]] — 85% 匹配率的另一面:15% 失真是本范式的固有风险

## 深入阅读

- 主题读本:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]]
- Sources:[[Wiki/Sources/AIAgents/Park-2023-generative-agents]]、[[Wiki/Sources/AIAgents/Park-2024-1000-agents]]

## 开放问题

- **跨文化泛化**:Park 2024 基于美国人口,推广到其他社会需要重新做访谈吗?
- **伦理 / 法律**:用真人访谈作 grounding 的知情同意、数据存储、"数字复制体"权属问题
- **本范式的 reproducibility**:token 成本劝退复现,学界标准能不能让 replication 变可行?
