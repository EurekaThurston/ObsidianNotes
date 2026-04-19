---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [ai, llm, foundation]
sources: 1
aliases: [大语言模型, Large Language Model, LLM]
---

# LLM（Large Language Model，大语言模型）

> 本质是一个"高度复杂的词语接龙引擎"——根据前面所有字猜下一个字最可能是什么。

## 概览

你用的 ChatGPT、Claude、DeepSeek、豆包、Kimi 本质上都是 LLM。它读过互联网上几乎所有公开文本，把人类的语言模式学成了庞大的概率分布。之所以看起来像在"思考"，是因为概率接龙本身已经能涌现出推理的表象。

**重要澄清**：LLM 没有"真的在想"，也没有"知道 vs 不知道"的自觉。这是理解它为什么会 [[Wiki/Concepts/AIApps/Hallucination|产生幻觉]]、为什么有[[Wiki/Concepts/AIApps/Context-window|上下文腐烂]]、为什么同一问题两次答不一样的钥匙。

## 关键事实 / 属性

- **底层架构**：Transformer（2017，Google《Attention is All You Need》）。自注意力机制让模型可以"一眼扫过整段话"，而不是从左到右逐字。
- **处理单元**：[[Wiki/Concepts/AIApps/Context-window|token]]——粗略地说英文 1 token ≈ 0.75 个单词，中文 1 汉字 ≈ 1-2 token。
- **上限**：[[Wiki/Concepts/AIApps/Context-window|上下文窗口]]——一次能看多少 token。当前旗舰已到数十万至百万级。

## 三步训练：一个 LLM 是怎么造出来的

1. **预训练（Pre-training）**：几万亿 token 的互联网语料上做"猜下一个字"。花掉成本 90%+，造就"通用语感"。训完只会续写，不听指令。
2. **指令微调（SFT，Supervised Fine-Tuning）**：人工写"问题—好回答"样本，教它学会对话模式。
3. **人类反馈强化学习（RLHF）**：人类对 AI 多个候选回答打分，用打分调教模型。这一步决定了模型的"性格"。

**为什么各家 AI 性格差那么多** → RLHF 偏好设置不同。
**为什么知识有"截止日期"** → 预训练语料有时间点。
**为什么有时它礼貌但答非所问** → 被训练成"讨人喜欢"有时胜过"答对"。

## 代表产品（2026）

- **闭源旗舰**：Claude（Anthropic）、GPT（OpenAI）、Gemini（Google）
- **开源主力**：Llama（Meta）、DeepSeek、Qwen（通义千问）、GLM（智谱）
- **国内消费端**：豆包、Kimi、文心一言

## 相关

- [[Wiki/Concepts/AIApps/Hallucination|幻觉]] — LLM 的结构性毛病
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]] — LLM 的"视力范围"及其陷阱
- [[Wiki/Concepts/AIApps/Reasoning-model|推理模型]] — 2024 末起的新变种，先"打草稿"再答
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — 把 LLM 装上手脚

## 引用来源

- 主题读本(推荐通读):[[Wiki/Readers/AIApps/AI-primer-v2-读本]]
- 原子 source:[[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- Transformer 架构本身是否会被下一代架构（Mamba、SSM 等）替换？目前主流仍是 Transformer 及其变体。
- RLHF 对"性格"的塑造边界在哪？同一底模用不同 RLHF 能走多远？
