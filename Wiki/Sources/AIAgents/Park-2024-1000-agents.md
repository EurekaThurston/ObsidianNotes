---
type: source
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, generative-agents, simulation, social-science, paper, stanford]
source_type: paper
---

# Park et al. 2024 — Generative Agent Simulations of 1,000 People

- **原文**:本仓未托管 PDF,直链 arXiv [2411.10109](https://arxiv.org/abs/2411.10109)
- **作者**:Joon Sung Park 等(Stanford HAI + 合作者)
- **日期**:2024-11(arXiv 发布),2025-05 被 Stanford HAI 重点报道
- **URL**:<https://arxiv.org/abs/2411.10109>
- **类型**:paper(Park 的博士论文分章,Stanford Digital Repository [purl.stanford.edu/jm164ch6237](https://purl.stanford.edu/jm164ch6237))
- **Stanford HAI 报道**:[AI Agents Simulate 1052 Individuals' Personalities with Impressive Accuracy](https://hai.stanford.edu/news/ai-agents-simulate-1052-individuals-personalities-with-impressive-accuracy)
- **HAI 项目页**:[Generative Agent Simulations of 1,000 People](https://ai4pb.stanford.edu/projects/generative-agent-simulations-of-1,000-people)
- **获取方式**:本 wiki 未全文阅读 PDF。本页内容基于 **arXiv 摘要 + Stanford HAI 新闻 + 项目主页** 整合

## 摘要

2023 Smallville 的直接续作。从"25 个虚构 NPC"跃迁到 **1052 个真实美国人的 agent 化**:研究团队招募并**深度访谈 1052 位美国居民 2 小时**,把每人的访谈转录塞给一个 LLM 作为 in-depth 上下文,让它回答原受访者**同样的**问卷题、行为决策题、扩散实验题。

样本按美国人口特征分层(年龄 / 种族 / 性别 / 民族 / 教育 / 政治光谱),使结论具代表性。核心实证:**agent 在 General Social Survey 上复现对应真人答案的准确度 ≈ 85%**——**与真人两周后自己再答一遍的稳定度相当**。这是 agent-based social simulation 第一次有了**真人 ground truth 的定量对照**。

## 关键要点

### 架构转向

- **不再是 Smallville 那套 Memory/Reflection/Planning**——用 agent-based self-report 为主的架构:把**访谈转录 + 人口学属性**作为 agent 的 persistent context + retrieval 源
- Smallville 的三件套解决的是"agent 如何连贯活过多天",Park 2024 解决的是"agent 如何和特定真人的分布一致"——不同问题不同解
- 相关 arXiv [2411.10109](https://arxiv.org/abs/2411.10109) 副标题:*"LLM Agents Grounded in Self-Reports Enable General-Purpose Simulation of Individuals"*——**self-report grounding** 是关键词

### 样本规模与工程

- **1052 人**,每人 **2 小时深度访谈**,全部转录作为 grounding
- 样本按美国人口统计分层——不是方便样本
- 访谈 + 每人对应 agent,可 scale 到群体级实验

### 评测结论

- **General Social Survey 复现准确率 85%**——与"真人两周后再答一遍"的 test-retest reliability 同量级
- 跨**态度 / 行为 / 扩散**三类任务稳定
- 个体级保真度 + 群体级涌现现象都可评测

### 应用案例

论文和 HAI 项目页展示了数字孪生可为以下场景低成本跑"先行实验":

- **公共政策**:可用 agent 群先试不同的政策推行方式,看哪种受众反应最好
- **内容推荐 / 平台设计**:算法改动先在 agent 群跑,测对立、极化、参与度的预期影响
- **公共健康干预**:医疗信息传播 / 疫苗观念转变的 agent 级模拟
- **社会科学研究**:节省"找真人跑"的成本,代价是失真边界需明确

### 范式转向

从 2023 Smallville 的 HCI demo("能不能涌现像人的行为")→ 2024 1000 agents 的社会科学方法论("能不能替代一部分真人参与研究")。**评测标准从主观 believability → 客观真人 ground truth 对照**,这是这个议题能从新奇玩具进入严肃科学的关键跳板。

## 涉及实体 / 概念

- [[Wiki/Entities/AIAgents/Joon-sung-park]](博士论文分章)
- [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]](本页强化了这个范式)
- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]](对照面——已不是主架构)

## 与既有 wiki 的关系

- **承接**:[[Wiki/Sources/AIAgents/Park-2023-generative-agents]]——同一作者线下一阶段
- **印证**:[[Wiki/Concepts/AIFoundations/Hallucination]]——85% 匹配率≠100%,15% 的差异正是 agent 相对真人"发挥"的风险边界
- **新增**:AIAgents 下"数字孪生社会学"维度,与另一分支 [[Wiki/Entities/AIAgents/A16z-ai-town]](工程化游戏化)形成方向分叉

## 开放问题

- 本 source 未读 PDF,具体 prompt、retrieval 策略、access grouping 细节待核验
- 85% 复现率的**置信区间、问题类型差异**、失败案例的系统性分析不知
- 样本是美国人 → 推广到其他文化 / 社会的 agent 访谈是否需要重做?
