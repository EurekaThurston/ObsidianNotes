---
type: concept
created: 2026-04-17
updated: 2026-04-17
tags: [methodology, llm, knowledge-management, core]
sources: 1
aliases: [LLM Wiki, Wiki 方法论]
---

# LLM Wiki 方法论

> 让 LLM 增量构建并持续维护一个持久化、互联的 markdown 知识库,而不是每次查询时从零检索——知识是**编译一次,持续保鲜**,而不是在每次提问时重新推导。

## 概览

这是一种"**让 LLM 做 bookkeeping,让人做思考**"的知识管理范式,由 [[wiki/entities/karpathy|Andrej Karpathy]] 在 2026 年提出(见 [[wiki/sources/karpathy-llm-wiki]])。

它和 [[wiki/concepts/rag|RAG]] 的区别在于:RAG 每次查询都要重新从原始文档里拼碎片;Wiki 则由 LLM 在每次 ingest 时就把新信息**整合进已有的 wiki**——更新实体页、修订综合、标注矛盾。查询时直接在已整合好的 wiki 上进行,答案也可以沉淀回 wiki,形成复利。

## 关键属性

### 三层架构

- **Raw sources**:只读的原始资料。LLM 从中读,永不改写。
- **Wiki**:LLM 生成的 markdown 页面(摘要、实体、概念、综合、总览)。LLM 完全拥有。
- **Schema**:如 `CLAUDE.md` / `AGENTS.md`,告诉 LLM "如何维护这个 wiki"。人与 LLM 共同演进。

### 三种操作

| 操作 | 做什么 |
|---|---|
| **Ingest** | 摄入新源:读 → 讨论 → 写源摘要 → 更新受影响的实体/概念 → 更新 index → 更新 overview → 追加 log |
| **Query** | 基于 wiki 回答;有价值的答案归档为 synthesis |
| **Lint** | 定期体检:找矛盾、孤儿页、缺失链接、可补的空白 |

### 两个特殊文件

- `index.md`:内容目录,按类别列出所有页面(content-oriented)。
- `log.md`:时间线,追加式,条目用一致前缀(chronological)。

### 哲学

- 人类职责:**策源、提问、指方向、思考意义**。
- LLM 职责:**其他所有事**——读、写、归档、交叉引用、保持一致。
- 理由:知识库的瓶颈从来不是读或想,而是 bookkeeping。人会厌倦、会忘;LLM 不会。

## 相关

- [[wiki/concepts/rag|RAG]] — 对比参照(查询时检索 vs. 持续整合)
- [[wiki/concepts/memex|Memex]] — 思想源流(Vannevar Bush 1945)
- [[wiki/entities/karpathy|Andrej Karpathy]] — 提出者
- [[CLAUDE]] — 本仓库对此方法论的**具体化实现**

## 引用来源

- [[wiki/sources/karpathy-llm-wiki]] (raw: [[raw/notes/Karpathy Wiki 方法论]])

## 开放问题 / 待观察

- **规模边界**:原文说 index.md 在"~100 源、数百页"规模下够用,超过后需搜索引擎(qmd 等)。本仓库规模到何时需要引入?
- **多人协作**:原文提到"团队内部 wiki"可行,但具体的冲突解决、review 流程未展开。
- **跨仓库引用**:不同主题是否该分开建 wiki,还是一个大 wiki 用 tag 区分?(本文件未明确回答)
