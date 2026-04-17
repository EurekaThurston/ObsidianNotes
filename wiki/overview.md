---
type: overview
created: 2026-04-17
updated: 2026-04-17
tags: [overview]
sources: 1
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新:2026-04-17

---

## 当前主题

**知识库方法论本身(meta / bootstrap 阶段)。**

本仓库还在启动期,唯一 ingest 过的源是 Karpathy 的 *LLM Wiki* idea file(见 [[wiki/sources/karpathy-llm-wiki]])。因此目前 wiki 的内容都是关于"**如何建这个 wiki**"——是一次自举(self-hosting)。

## 核心论点(目前综合)

1. **持续整合 > 查询时检索**。相比 [[wiki/concepts/rag|RAG]] 每次重拼碎片,[[wiki/concepts/llm-wiki-方法论|LLM Wiki 方法论]] 在 ingest 时就完成跨源整合,让知识持久化、复利化。
2. **三层架构**:raw(只读源)/ wiki(LLM 写的 md)/ schema(LLM 维护规则)。职责清晰是这套方法能落地的前提。
3. **人类负责策源与方向,LLM 负责全部 bookkeeping**。这是分工的核心——LLM 不会厌倦,不会忘记更新链接,一次可以触达十几个文件。
4. **思想溯源到 [[wiki/concepts/memex|Memex]]**(1945)。Bush 的愿景 80 年后由 LLM 补上"维护者"这最后一块。

## 开放问题 / 待观察

- **规模边界**:index.md 在 ~100 源、数百页规模下够用,超过后需引入搜索引擎(qmd 等)。本仓库何时需要?
- **多人协作**:原文提到团队 wiki 可行,但冲突解决与 review 流程未展开。
- **跨领域分库 vs. 单库**:未来如果内容扩到多个不相干主题,是否分仓?

## 主要分支(目前)

只有一条主线:**knowledge-management / llm-wiki / methodology**。
等你扔进来第一批实质性(非 meta)资料后,这里会长出新分支。

---

## 入口

- 所有页面目录:[[index]]
- 操作规程:[[CLAUDE]]
- 时间线:[[log]]
- 方法论原文:[[raw/notes/Karpathy Wiki 方法论]]
