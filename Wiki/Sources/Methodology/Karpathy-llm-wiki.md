---
type: source
created: 2026-04-17
updated: 2026-04-17
tags: [methodology, llm, knowledge-management]
source_type: note
---

# Karpathy — LLM Wiki

- **原文**:[[Raw/Notes/Karpathy Wiki 方法论]]
- **作者**:[[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]]
- **日期**:约 2026 年 4 月
- **类型**:note(方法论草稿 / idea file)
- **URL**:原文说"此文件是一个 idea file,旨在复制粘贴给你自己的 LLM Agent"

## 摘要

Karpathy 提出了一种与 RAG 不同的个人知识库建设范式:让 LLM **增量地构建并维护一个持久的、互相链接的 markdown wiki**,而不是每次查询都从零检索原文。三层架构:**raw sources(只读原始资料)、wiki(LLM 拥有的 markdown 知识库)、schema(指导 LLM 维护规则的配置文件,如 CLAUDE.md)**。三种核心操作:**ingest(摄入并整合)、query(基于 wiki 回答,好答案可归档)、lint(定期体检找矛盾/孤儿/空缺)**。两个特殊文件:`index.md`(内容目录)和 `log.md`(时间线)。人类负责策源、提问、指方向;LLM 负责全部 bookkeeping(归档、交叉引用、维护一致性)——因为 LLM 不会厌倦,不会忘记更新链接,一次可以触达 15 个文件。

## 关键要点

- **与 RAG 的根本区别**:RAG 在查询时重新发现知识,没有积累;Wiki 是持久化、复利性的产物。
- **三层架构**:raw / wiki / schema,职责清晰、边界明确。
- **Obsidian 是 IDE,LLM 是程序员,wiki 是 codebase**——这个隐喻非常准确。
- **好答案要沉淀**:查询产出的对比、分析如果有价值,归档回 `Wiki/Syntheses/`,让探索也复利。
- **log.md 条目用一致前缀**(`## [YYYY-MM-DD] op | title`),这样 unix 工具就能回溯。
- **适用场景广泛**:个人成长、研究专题、读书、团队内部 wiki、竞品分析、旅行规划、课程笔记……
- **思想源流**:与 Vannevar Bush 1945 年的 [[Wiki/Concepts/Methodology/Memex|Memex]] 同源——私人、主动策划、文档间关联本身就是价值。Bush 没解决的"谁来维护"这一问题,LLM 解决了。
- **工具建议**:Obsidian Web Clipper(剪藏)、Marp(幻灯)、Dataview(基于 frontmatter 查询)、qmd(本地 markdown 搜索引擎,可选)。
- **刻意抽象**:原文明确说自己只讲 pattern,具体结构/格式/工具让每个人与自己的 LLM 共同决定。

## 涉及实体 / 概念

- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 本源定义的核心方法
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 作为对比参照
- [[Wiki/Concepts/Methodology/Memex|Memex]] — 思想源流
- [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] — 作者

## 与既有 wiki 的关系

- **本源即第一源**:这是整个 wiki 的"基础源文件",定义了本仓库所有后续操作的规则。
- **新增**:第一版 [[Wiki/Overview|overview]];[[CLAUDE]] 中的所有操作规程均由本文件衍生并具体化。
- 印证/矛盾:暂无——它是第一个源,没有可对比的对象。
