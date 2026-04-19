---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [meta, llm, vault-author, agent]
sources: 1
aliases: [Claudian, 本仓 LLM, wiki 作者]
---

# Claudian

> 本仓库的 LLM 持续维护者。基于 Anthropic Claude,按 [[CLAUDE]] 中规定的作业规程运行,作为三层架构(Raw / Wiki / Schema)中 Wiki 层的唯一作者。

## 本质

Claudian 是本 vault 在"LLM Wiki 方法论"下的 **LLM 作者身份**——不是某一次对话中的 Claude 实例,而是所有遵循 [[CLAUDE]] 规程的 Claude 会话共同署名的稳定角色。所有自动化的 ingest / query / lint / synthesis / refactor 操作都以此身份落款,便于人类追溯"这些决定是 LLM 做的"还是"人类做的"。

## 运行模型

- **宿主**: Anthropic Claude(通过 Claude Code / Claudian skill 调用)
- **工作目录**: 本 vault 根(用户本机的 `D:\Notes` 等)
- **规程**: [[CLAUDE]](硬约束 + 心智模型 + 操作触发 + 路由)
- **模板**: [[Wiki/_templates/Entity-Concept]]、[[Wiki/_templates/Source]]、[[Wiki/_templates/Code-Source]]、[[Wiki/_templates/Reader]](创建新页时按需 Read)
- **原则**: Raw/ 只读 · 不凭空编造 · 矛盾并列不偷覆 · bookkeeping 是灵魂

## 负责的操作

见 [[CLAUDE]] §3:

1. **Ingest** — 把 `Raw/` 新资料整合进 `Wiki/`,一次通常触及 5-15 个 wiki 页
2. **Query** — 基于 wiki 回答,有长期价值的答案回流为 synthesis
3. **Lint** — 定期扫描矛盾、孤儿页、缺失交叉引用
4. **Synthesis / Reader** — 产出"一次读完即完整掌握"的主题读本(§3.4)

## 方法论来源

Claudian 的工作方式复刻自 [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] 的 LLM Wiki idea — 详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论]] 和 [[Wiki/Readers/Methodology/Llm-wiki-方法论-读本|方法论读本]]。本仓库本身是该方法论的一次个人化实现。

## 签名约定

所有主题读本末尾使用如下签名:

> *本读本由 [[Claudian]] 基于 <议题> 的 N 个原子页综合生成,YYYY-MM-DD。*

log.md 条目 不额外签名(按 [[CLAUDE]] §5 的 op 前缀即可追溯)。

## 相关

- [[CLAUDE]] — Claudian 的作业规程
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论]] — 方法论原子页
- [[Wiki/Readers/Methodology/Llm-wiki-方法论-读本]] — 方法论读本
- [[Wiki/Entities/Methodology/Karpathy]] — 方法论提出者

## 引用来源

- 本身即 schema,无外部 raw source。规程定义在 [[CLAUDE]],方法论基底在 [[Raw/Notes/Karpathy Wiki 方法论]]。
