---
type: synthesis
created: 2026-04-19
updated: 2026-04-19
tags: [ai, methodology, evolution, engineering]
sources: 1
aliases: [三段论演进, Prompt-Context-Harness, AI 工程三阶段]
---

# Prompt → Context → Harness：AI 工程方法论三阶段演进

> AI 应用开发的思路，三年走了三个大台阶。这是 2022–2026 年最关键的主线叙事。

## 总览

| 阶段 | 时间 | 核心问题 | 类比 |
|---|---|---|---|
| **Prompt Engineering** | 2022–2024 | 怎么问？ | 教马"左转""右转"的口令 |
| **Context Engineering** | 2025 | 让 AI 看到什么？ | 给马一张地图，让它自己看路 |
| **Harness Engineering** | 2026 | 在什么条件下运行？ | 给马装上缰绳、马鞍、护栏、赛道、裁判、计时器 |

---

## 第一阶段：Prompt Engineering（提示词工程）— 2022–2024

### 核心问题：怎么问？

当 [[Wiki/Concepts/AIFoundations/Llm|LLM]] 刚普及时，人们发现"同一个问题换个问法，AI 回答质量天差地别"，于是诞生了提示词工程——**通过精心设计提示词，让模型输出更好的结果**。

### 常见技巧

- **角色设定**："你是一位资深 UE4 技术美术"
- **思维链（Chain of Thought, CoT）**：让它"一步步思考"（Wei et al., Google Brain, 2022）
- **少样本示例（Few-shot）**：给几个输入-输出例子当示范
- **格式指定**：要求输出 JSON / Markdown 等结构化格式

### 局限

不管你多会措辞，AI 仍然**看不到你的代码库、业务文档、历史决策**。"一次问一次答"的模式有天花板。

**注**：CoT 后来被"内化到模型训练里"，催生了专门的 [[Wiki/Concepts/AIFoundations/Reasoning-model|推理模型]]——这是提示技巧升级为架构设计的典型例子。

---

## 第二阶段：Context Engineering（上下文工程）— 2025

### 核心问题：让 AI 看到什么？

2025 年，[[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] 提出：Context Engineering 比 Prompt Engineering 更关键。Shopify CEO Tobi Lütke 也大力推广。

关注点从"怎么措辞"转向**"如何给 AI 搭建一个完整的信息环境"**：

- 用 [[Wiki/Concepts/Methodology/Rag|RAG]] 把相关文档塞进去
- 用 [[Wiki/Concepts/AIFoundations/Mcp|MCP]] 接通外部工具和数据
- 管理对话历史、长期记忆
- 精心设计系统提示词（System Prompt）

### 局限

交互模式还没变——**依然是"人给信息，AI 生成内容"**。一旦 AI 要自主跑多步任务（即 [[Wiki/Concepts/AIFoundations/Ai-agent|Agent]] 时代），光有好上下文不够，还得有约束、有反馈、有护栏。

---

## 第三阶段：Harness Engineering（驾驭工程）— 2026

### 核心问题：在什么条件下运行？

2026 年 2 月，这个词从一篇博客出发，几周内变成整个 AI 工程圈的高频术语。

**起源**：
- **Mitchell Hashimoto**（HashiCorp 联合创始人、Terraform 作者）博客首次命名并定义："每当发现 Agent 犯错，就花时间设计一个机制，确保它永远不会再犯同样的错。"
- 几天后 **OpenAI** 发布报告：3 名工程师 5 个月不写一行代码，纯靠 Agent（Codex）生成约 100 万行代码，交付真实产品内测版。
- **Martin Fowler** 深度分析，将 Harness 归纳为三大类；叠加 OpenAI 强调的反馈回路，形成"四大支柱"。

### 四大支柱

1. **上下文管理**：精准注入，不多不少。对抗 [[Wiki/Concepts/AIFoundations/Context-window|Context Rot]]。
2. **架构约束**：把规则代码化，用机器守边界。
3. **反馈回路**：测试/Linter 失败信息成为下一轮上下文，实现自我修复。
4. **熵管理**：定期清理技术债。

详见 [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]]。

### 一个有力佐证

**LangChain 编码 Agent** 在 Terminal Bench 2.0 上，底层模型**一个参数都没改**，只优化 Harness，排名从第 30 升至第 5，得分 52.8% → 66.5%（具体数字以原博客为准）。

---

## 核心洞察

这条三段论的终极启示：

> **决定 AI 表现好坏的最大变量，往往不是模型本身多聪明，而是模型被放在什么样的环境里。**

这也是 2026 AI 技术栈三层全景的基础：

```
┌─────────────────────────────────────────────┐
│  第 3 层：Skills（技能层，跨工具可移植）    │
├─────────────────────────────────────────────┤
│  第 2 层：Agent Harness（驾驭层）           │
│  （差异化主战场）                           │
├─────────────────────────────────────────────┤
│  第 1 层：Model（模型层，正在商品化）       │
└─────────────────────────────────────────────┘
```

**模型层正在大宗商品化**，各家模型能力越来越接近。真正的竞争壁垒在上面两层——你的 Harness 有多稳、你的 [[Wiki/Concepts/AIFoundations/Agent-skills|Skills]] 有多专业。

两个产品可能用同一个底层 LLM，但拥有更好 Harness 和 Skills 的那个，会提供**远超对方**的用户体验。

---

## 相关

- [[Wiki/Concepts/AIFoundations/Llm|LLM]] — 三段论的服务对象
- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — 第 2→3 阶段跃迁的触发器
- [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]]
- [[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] — Harness 支柱一的核心工具
- [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] — Context Engineering 倡导者、Vibe Coding 命名者
- [[Wiki/Concepts/Methodology/Vibe-coding|Vibe Coding]] — 演进第一阶段的独立概念页
- [[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat|如何向 AI 提问]] — 第一阶段的实操指南

## 引用来源

- [[Wiki/Sources/AIFoundations/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- 第四阶段会是什么？候选：可观测性工程（Observability Engineering）？AI 生态工程（Ecosystem Engineering）？
- 三阶段是否真的"向后兼容"——有了 Harness 是否意味着前两阶段可以不学？实践看来不行，新手仍需补全三阶段基本功。
