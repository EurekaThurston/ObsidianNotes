---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [ai, llm, pitfall, safety]
sources: 1
aliases: [幻觉, Hallucination, AI 胡说]
---

# 幻觉（Hallucination）

> AI 一本正经地胡说的结构性毛病——不是 bug，不会被下一代模型完全消灭。

## 概览

[[Wiki/Concepts/AIApps/Llm|LLM]] 的核心动作是"猜下一个词"，它**没有"知道"与"不知道"的概念**。当训练数据里没有准确答案时，它也会顺着语气编一个——而且编得**又流畅又自信**。这就是幻觉。

理解幻觉是使用 AI 的第一道基本功。不理解这个，你会被它的"笃定语气"骗到。

## 典型表现

- 编造不存在的论文、书名、作者、人物
- 编造不存在的 API 函数、库名、命令行参数
- 把 A 的观点安到 B 头上
- 给出错误的日期、数字、统计，但语气确信
- 在解释代码时编造源码里不存在的"设计理由"

## 为什么是结构性的

- 模型的目标函数是"预测下一个 token 的概率"，不是"说真话"。
- 训练语料本身就混杂真假，模型学到的是"看起来合理的表达"。
- RLHF 会让模型更倾向"讨好"用户（听起来像个好回答）而非"承认我不知道"。
- 下一代模型可以缓解幻觉（比如更好的后训练、让模型学会说"不知道"、加入在线检索），但**不能消除**。只要核心机制还是概率生成，就有幻觉。

## 对抗幻觉的机制（后面几章都有对应）

- **[[Wiki/Concepts/Methodology/Rag|RAG]]**：把可信资料塞进上下文——AI 有"开卷"可抄，胡编少了。
- **工具调用 / [[Wiki/Concepts/AIApps/Mcp|MCP]]**：让 AI 去真实系统查数据（查数据库、跑脚本、搜网页），而不是靠记忆。
- **[[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] 的反馈回路**：让 AI 的产出被测试/Linter/类型检查等自动验证，错了就强制改。
- **HITL（Human-in-the-Loop）**：关键决策点叫人审核。
- **[[Wiki/Concepts/AIApps/Reasoning-model|推理模型]]**：通过显式"打草稿"减少低级幻觉；但对"编造事实"这类幻觉效果有限。

## 用户第一纪律

> **AI 给出的任何具体事实（数字、日期、链接、引用、API 名称、函数签名）都要验证一遍再用。**

这是使用 AI 的底线，对任何岗位、任何场景都适用。美术引用参考图名、策划引用游戏案例、开发引用 API 文档——全都要核验。

## 相关

- [[Wiki/Concepts/AIApps/Llm|LLM]] — 幻觉的载体
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口]] — 上下文塞得过满也会诱发类似"幻觉"的失忆
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] — 用工程机制对抗幻觉
- [[Wiki/Concepts/Methodology/Rag|RAG]] — "开卷考试"降低幻觉

## 引用来源

- 主题读本(推荐通读):[[Readers/AIApps/AI 应用生态全景 2026]]
- 原子 source:[[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- 幻觉率的客观测量方法？现有 benchmark 都有偏差。
- "让模型说不知道"的训练技巧，边界在哪？过度谨慎会让模型变成"什么都答不了"。
