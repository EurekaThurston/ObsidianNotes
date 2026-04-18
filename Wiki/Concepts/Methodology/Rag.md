---
type: concept
created: 2026-04-17
updated: 2026-04-17
tags: [llm, knowledge-management, retrieval]
sources: 1
aliases: [Retrieval-Augmented Generation, 检索增强生成]
---

# RAG(Retrieval-Augmented Generation)

> 主流的"LLM + 文档"范式:把文档切块、建索引,查询时检索相关片段注入 prompt,由 LLM 生成答案。

## 概览

目前大多数"把资料丢给 LLM 问"的产品(NotebookLM、ChatGPT 文件上传、多数企业知识库)采用这种模式。它能工作,但在 [[Wiki/Sources/Methodology/Karpathy-llm-wiki|Karpathy]] 看来存在一个根本缺陷:**LLM 每次查询都在从零发现知识,没有任何积累**。

## 关键属性

- **查询时检索**:问题到来时才去原始文档里找相关片段。
- **无状态**:两次查询之间没有任何持久的"理解"被建立。
- **合成靠当场**:需要综合 5 篇材料的细微问题,每次都要现场拼碎片。
- **不处理矛盾**:不同源间的冲突不会被显式记录,只会在某次回答里碰巧浮现。

## 与 LLM Wiki 的对比

| 维度 | RAG | [[Wiki/Concepts/Methodology/Llm-wiki-方法论\|LLM Wiki]] |
|---|---|---|
| 知识积累 | 无 | 持续复利 |
| 交叉引用 | 查询时再找 | ingest 时已建立 |
| 矛盾标注 | 无 | 显式记录 |
| 合成 | 每次重做 | 一次编译,长期更新 |
| 维护成本 | 几乎为 0 | LLM 承担 bookkeeping |
| 查询延迟 | 较高(要检索+综合) | 较低(答案已在 wiki 里) |
| 适合 | 一次性、广而浅 | 长期深耕、需要累积 |

## 相关

- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — Karpathy 提出的替代方案
- [[Wiki/Concepts/Methodology/Memex|Memex]] — 更早的"关联式个人知识库"愿景

## 引用来源

- [[Wiki/Sources/Methodology/Karpathy-llm-wiki]] (raw: [[Raw/Notes/Karpathy Wiki 方法论]])

## 开放问题

- RAG 与 LLM Wiki 并非二选一——本仓库如果长到需要,可以在 wiki 之上再加 RAG 作为底层搜索(原文提到 qmd 就扮演这个角色)。
