---
type: overview
created: 2026-04-17
updated: 2026-04-18
tags: [overview]
sources: 1
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新:2026-04-18

---

## 当前主题

**Wiki 现在覆盖一个主题**:

### 1. 知识库方法论本身(meta / bootstrap)

自举阶段产出:基于 [[Raw/Notes/Karpathy Wiki 方法论]] 建的三层架构(Raw/Wiki/schema);详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]。

---

## 其他论点(来自方法论 ingest)

**关于 LLM Wiki 本身**(基于 Karpathy 那篇 idea file):

1. **持续整合 > 查询时检索**:相比 [[Wiki/Concepts/Methodology/Rag|RAG]] 每次重拼碎片,[[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki]] 在 ingest 时就完成跨源整合
2. **三层架构**:raw(只读源)/ wiki(LLM 写的 md)/ schema(规则)
3. **人类提供方向,LLM 负责 bookkeeping**:一次 ingest 触达 15 个文件不是 bug,是特性
4. **思想溯源到 [[Wiki/Concepts/Methodology/Memex|Memex]]**(1945)

---

## 开放问题 / 待观察

### 关于 Wiki 本身
- **规模边界**:当前 ~5 个页面,index.md 还够用;当超过 200 页时怎么办?
- **Concept 页跨仓共享**:未来 `stock` 仓 ingest 时,概念页能复用多少?

---

## 主要分支(目前)

```
当前 wiki 的知识图 ≈ 1 条线

methodology (meta)
    ├── karpathy.md
    ├── llm-wiki-方法论
    ├── memex
    └── rag
```

---

## 入口

- 所有页面目录:[[index]]
- 操作规程:[[CLAUDE]]
- 时间线:[[log]]
- **方法论原文**:[[Raw/Notes/Karpathy Wiki 方法论]]

---

## 下一步建议

- **纳入第一个真实技术 source**——验证 wiki 流程在代码领域的实际效果
- **纳入 `stock` UE 公版引擎**——从 Niagara/Rendering 等模块开始
