---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [ai, llm, context, pitfall]
sources: 1
aliases: [上下文窗口, Context Window, Context Rot, 上下文腐烂]
---

# 上下文窗口（Context Window）与 Context Rot

> AI 一次能"看到"的文本长度上限，以及上下文塞太满导致模型变笨的现象。

## 概览

**Context Window** 是 [[Wiki/Concepts/AIApps/Llm|LLM]] 一次能同时读取和关注的 token 数量上限。类比人的短期记忆——你能记住一页纸，记不住一整本书。

- 早期模型：几千 token（一篇短文）
- 2024 年起：数十万 token（一本书）
- 旗舰模型：百万级 token（一整个代码库）

看起来"变大就变强"，但实际使用中藏着一个反直觉现象：**Context Rot（上下文腐烂）**。

## Context Rot：越满越笨

当上下文塞到 80%、90% 满时，AI 的表现会明显下滑：

- 忘记早先说过的约定
- 前后矛盾
- 忽略你刚刚塞进去的关键文档
- 回答开始"走神"

另一个等价说法是 **"Lost in the middle"**——AI 对最前面和最后面的内容记得清楚，中间段落容易失焦（和人类阅读长文的心理很像）。

**这件事的工程意义极大**：

- 不是给 AI 越多信息越好，而是要**精准地只给当下需要的那一份**。
- 这就是 [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] 采用"渐进式披露"的根本动机。
- 也是 [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] 四大支柱之一"上下文管理"的核心问题。
- 也是 [[Wiki/Concepts/AIApps/Ai-agent|Agent]] 采用 subagent 分工的动机——让每个子 Agent 只看自己那一小摊。

## Token：计量单位

- **英文**：1 token ≈ 0.75 个单词
- **中文**：1 个汉字 ≈ 1-2 token
- **代码**：常见变量名/关键字通常是 1 个 token

### 为什么要关心 token

- **API 按 token 计费**：输入 token 和输出 token 都收钱，输出通常更贵。
- **prompt caching 省钱**：相同前缀多次调用可以缓存，大幅降低成本——这也是 Skills 渐进式披露真金白银的价值。
- **上下文有限**：输入的 token + 生成的 token 不能超过模型窗口。

## 相关

- [[Wiki/Concepts/AIApps/Llm|LLM]] — 上下文窗口是 LLM 的"视力范围"
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — 渐进式披露直接对抗 Context Rot
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] — "上下文管理"是其四大支柱之一
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — Subagent 分工也是应对 Context Rot 的手段
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 通过检索把"最相关的部分"塞进上下文

## 引用来源

- 主题读本(推荐通读):[[Wiki/Readers/AIApps/AI-primer-v2-读本]]
- 原子 source:[[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- Context Rot 的具体规律（哪些位置最容易丢？和模型规模相关吗？）还没有统一结论。Dex Horthy 提出的 "Smart Zone / Dumb Zone" 是一种经验划分。
- 超长上下文（百万级）在实际项目中的真实利用率？很多案例反映即使模型宣称 1M，实际有效利用可能远小于此。
